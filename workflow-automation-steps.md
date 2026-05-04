# Cal.diy Workflow 自动化机制详解

本文档详细描述了 cal.diy 系统中 Workflow 自动化（事件触发多步动作）的实现机制，包括内部预约事件与外部渠道的串联方式，以及跨不同外部服务的 step 失败恢复路径。

---

## 一、系统架构概览

### 1.1 历史演进

**重要说明**：旧的 Workflow 系统（包含 `Workflow`、`WorkflowStep`、`WorkflowReminder` 等表）已在 2026 年 3 月的迁移中被删除（`20260319000000_drop_workflow_tables`）。当前系统使用更现代化的任务调度和 Webhook 机制。

### 1.2 当前架构

当前系统主要通过以下组件实现自动化：

```
┌─────────────────────────────────────────────────────────────────┐
│                        事件触发层                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────────────────┐  │
│  │ 预约事件    │  │ Webhook事件 │  │ 时间触发事件           │  │
│  │ (Booking)   │  │             │  │ (MEETING_STARTED等)   │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬───────────┘  │
│         │                  │                      │               │
│         └──────────────────┼──────────────────────┘               │
│                            │                                       │
└────────────────────────────┼───────────────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────────────┐
│                        任务调度层 (Tasker)                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Tasker 基类                                │   │
│  │  - AsyncTasker (Trigger.dev): 生产环境异步执行               │   │
│  │  - SyncTasker: E2E测试/回退场景同步执行                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                          │
│        ┌───────────────────┼───────────────────┐                    │
│        │                   │                   │                    │
┌────────▼───────────┬───────▼───────┬─────────▼─────────────┐      │
│ WebhookTasker       │ BookingEmail  │ 其他任务调度器          │      │
│ (Webhook 发送)       │ AndSmsTasker │ (CRM, Analytics等)     │      │
│                     │ (邮件/SMS)    │                        │      │
└─────────────────────┴───────────────┴────────────────────────┘      │
└───────────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────▼───────────────────────────────────────┐
│                        任务存储层                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Task 表 (PostgreSQL)                       │   │
│  │  - payload: 任务数据                                           │   │
│  │  - type: 任务类型                                              │   │
│  │  - scheduledAt: 调度时间                                       │   │
│  │  - maxAttempts: 最大重试次数                                   │   │
│  │  - attempts: 当前尝试次数                                      │   │
│  │  - succeededAt: 成功时间                                       │   │
│  │  - lastError: 最后错误信息                                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 二、事件触发机制

### 2.1 触发事件类型

系统定义了丰富的触发事件类型（`WebhookTriggerEvents` 枚举），位于 `packages/prisma/schema.prisma:1120`：

| 事件类型 | 描述 | 触发时机 |
|---------|------|---------|
| `BOOKING_CREATED` | 预约创建 | 新预约创建时 |
| `BOOKING_PAYMENT_INITIATED` | 支付开始 | 支付流程启动时 |
| `BOOKING_PAID` | 支付完成 | 支付成功时 |
| `BOOKING_RESCHEDULED` | 预约改期 | 预约时间变更时 |
| `BOOKING_REQUESTED` | 预约请求 | 需要确认的预约提交时 |
| `BOOKING_CANCELLED` | 预约取消 | 预约被取消时 |
| `BOOKING_REJECTED` | 预约被拒 | 预约请求被拒绝时 |
| `BOOKING_NO_SHOW_UPDATED` | 未出席状态更新 | 未出席状态变更时 |
| `FORM_SUBMITTED` | 表单提交 | 表单填写完成时 |
| `MEETING_STARTED` | 会议开始 | 会议启动时（时间触发） |
| `MEETING_ENDED` | 会议结束 | 会议结束时（时间触发） |
| `RECORDING_READY` | 录制就绪 | 会议录制文件准备好时 |
| `RECORDING_TRANSCRIPTION_GENERATED` | 转录生成 | 会议转录完成时 |
| `OOO_CREATED` | 休假创建 | 外出办公条目创建时 |
| `AFTER_HOSTS_CAL_VIDEO_NO_SHOW` | 主持人未出席 | Cal Video 会议主持人未出席时 |
| `AFTER_GUESTS_CAL_VIDEO_NO_SHOW` | 嘉宾未出席 | Cal Video 会议嘉宾未出席时 |
| `FORM_SUBMITTED_NO_EVENT` | 无事件表单提交 | 表单提交但无关联事件时 |
| `DELEGATION_CREDENTIAL_ERROR` | 委托凭证错误 | 委托凭证出现问题时 |
| `WRONG_ASSIGNMENT_REPORT` | 错误分配报告 | 分配错误被报告时 |

### 2.2 触发方式分类

#### 即时触发（Immediate Triggers）

这类事件在发生时立即触发：
- `BOOKING_CREATED`、`BOOKING_CANCELLED`、`BOOKING_RESCHEDULED` 等预约生命周期事件
- `FORM_SUBMITTED`、`OOO_CREATED` 等操作事件

**实现位置**：
- `packages/features/webhooks/lib/service/WebhookService.ts:135` - `processWebhooks()` 方法

```typescript
async processWebhooks(
  trigger: WebhookTriggerEvents,
  payload: WebhookPayload,
  subscribers: WebhookSubscriber[]
): Promise<void> {
  if (subscribers.length === 0) {
    this.log.debug("No subscribers to process for trigger:", { trigger });
    return;
  }

  const promises = subscribers.map(async (subscriber) => {
    // 为每个订阅者发送 Webhook
    const result = await this.sendWebhook(trigger, payload, subscriber);
    // ...
  });

  // 使用 Promise.allSettled 确保单个 Webhook 失败不会影响其他 Webhook
  const results = await Promise.allSettled(promises);
  // ...
}
```

#### 时间触发（Scheduled Triggers）

这类事件基于预约时间在特定时刻触发：
- `MEETING_STARTED` - 会议开始时
- `MEETING_ENDED` - 会议结束时
- `AFTER_HOSTS_CAL_VIDEO_NO_SHOW` / `AFTER_GUESTS_CAL_VIDEO_NO_SHOW` - 会议开始后一段时间检测未出席

**实现位置**：
- `packages/features/webhooks/lib/scheduleTrigger.ts:282` - `scheduleTrigger()` 函数

```typescript
export async function scheduleTrigger({
  booking,
  subscriberUrl,
  subscriber,
  triggerEvent,
  isDryRun = false,
}: {
  booking: { id: number; endTime: Date; startTime: Date };
  subscriberUrl: string;
  subscriber: { id: string; appId: string | null };
  triggerEvent: WebhookTriggerEvents;
  isDryRun?: boolean;
}) {
  if (isDryRun) return;
  try {
    const payload = JSON.stringify({ triggerEvent, ...booking });

    await prisma.webhookScheduledTriggers.create({
      data: {
        payload,
        appId: subscriber.appId,
        // 根据事件类型选择开始时间
        startAfter: triggerEvent === WebhookTriggerEvents.MEETING_ENDED 
          ? booking.endTime 
          : booking.startTime,
        subscriberUrl,
        webhook: { connect: { id: subscriber.id } },
        booking: { connect: { id: booking.id } },
      },
    });
  } catch (error) {
    console.error("Error cancelling scheduled jobs", error);
  }
}
```

**关键数据结构**：`WebhookScheduledTriggers` 表存储预定触发的任务，包含：
- `payload`: 触发时的负载数据
- `startAfter`: 触发时间（会议开始/结束时间）
- `subscriberUrl`: Webhook 订阅者 URL
- `webhookId`: 关联的 Webhook 配置

---

## 三、外部渠道集成

### 3.1 Webhook 渠道

Webhook 是系统中最主要的外部集成渠道，支持将事件推送到任意 HTTP 端点。

#### 核心组件

| 组件 | 文件位置 | 职责 |
|------|---------|------|
| `WebhookService` | `packages/features/webhooks/lib/service/WebhookService.ts` | Webhook 发送逻辑、订阅者管理 |
| `WebhookTasker` | `packages/features/webhooks/lib/tasker/WebhookTasker.ts` | Webhook 任务调度器（Async/Sync 双模式） |
| `WebhookTriggerTasker` | `packages/features/webhooks/lib/tasker/WebhookTriggerTasker.ts` | Trigger.dev 异步任务实现 |
| `deliverWebhook` | `packages/features/webhooks/lib/tasker/trigger/deliver-webhook.ts` | Trigger.dev 任务定义 |
| `sendWebhook` | `packages/features/tasker/tasks/sendWebook.ts` | 内部任务处理器 |

#### Webhook 发送流程

```
预约事件发生
      │
      ▼
┌─────────────────┐
│  查找订阅者      │
│  getSubscribers │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  processWebhooks│
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│  TASKER_ENABLE_WEBHOOKS │
│         = 1?            │
└───────────┬─────────────┘
     ┌──────┴──────┐
     │ 是           │ 否
     ▼              ▼
┌──────────┐   ┌─────────────────┐
│schedule- │   │sendWebhook-     │
│Webhook   │   │Directly         │
│(异步队列) │   │(同步发送)        │
└────┬─────┘   └────────┬────────┘
     │                   │
     └─────────┬─────────┘
               ▼
        ┌──────────────┐
        │ 实际 HTTP 请求│
        │ (fetch)      │
        └──────────────┘
```

#### 直接发送实现（`sendWebhookDirectly`）

位于 `packages/features/webhooks/lib/service/WebhookService.ts:75`：

```typescript
private async sendWebhookDirectly(
  trigger: WebhookTriggerEvents,
  payload: WebhookPayload,
  subscriber: WebhookSubscriber
): Promise<WebhookDeliveryResult> {
  const { subscriberUrl, payloadTemplate } = subscriber;
  if (!subscriberUrl) throw new Error("Missing subscriber URL");

  // 确定 Content-Type
  const contentType =
    !payloadTemplate || this.isJsonTemplate(payloadTemplate)
      ? "application/json"
      : "application/x-www-form-urlencoded";

  // 构建请求体
  const body = JSON.stringify({
    triggerEvent: trigger,
    createdAt: payload.createdAt,
    payload: payload.payload,
  });

  // 生成签名（HMAC-SHA256）
  const signature = subscriber.secret
    ? createHmac("sha256", subscriber.secret).update(body).digest("hex")
    : "no-secret-provided";

  // 发送 HTTP POST 请求
  const response = await fetch(subscriberUrl, {
    method: "POST",
    headers: {
      "Content-Type": contentType,
      "X-Cal-Signature-256": signature,      // 用于接收方验证
      "X-Cal-Webhook-Version": subscriber.version,
    },
    redirect: "manual",
    body,
  });

  const responseText = await response.text();

  return {
    ok: response.ok,
    status: response.status,
    message: responseText || undefined,
    duration: 0,
    subscriberUrl: subscriberUrl,
    webhookId: subscriber.id,
  };
}
```

#### Webhook 安全机制

1. **签名验证**：使用 HMAC-SHA256 对请求体签名，接收方可以通过 `X-Cal-Signature-256` 头验证请求来源
2. **版本控制**：`X-Cal-Webhook-Version` 头标识 API 版本
3. **订阅者隔离**：每个 Webhook 配置独立的 `secret` 密钥

### 3.2 邮件渠道

邮件渠道主要用于发送预约相关的通知（确认邮件、改期通知、取消通知等）。

#### 核心组件

| 组件 | 文件位置 | 职责 |
|------|---------|------|
| `BookingEmailAndSmsTasker` | `packages/features/bookings/lib/tasker/BookingEmailAndSmsTasker.ts` | 邮件/SMS 任务调度器 |
| `BookingEmailAndSmsTaskService` | `packages/features/bookings/lib/tasker/BookingEmailAndSmsTaskService.ts` | 邮件发送业务逻辑 |
| `BookingEmailSmsHandler` | `packages/features/bookings/lib/BookingEmailSmsHandler.ts` | 邮件模板和发送器 |
| `workflow-email-service` | `packages/emails/workflow-email-service.ts` | 工作流邮件服务（反馈、月报等） |

#### 邮件触发流程

```
预约操作（确认/改期/取消）
           │
           ▼
┌──────────────────────────┐
│ BookingEmailAndSmsTasker │
│ .send()                  │
└───────────┬──────────────┘
            │
            ▼
    ┌───────────────┐
    │ 选择执行模式   │
    └───────┬───────┘
     ┌──────┴──────┐
     │ Async       │ Sync
     ▼              ▼
┌──────────┐   ┌─────────────────┐
│Trigger.  │   │BookingEmail-    │
│dev 任务  │   │AndSmsSyncTasker │
└────┬─────┘   └────────┬────────┘
     │                   │
     └─────────┬─────────┘
               ▼
    ┌──────────────────┐
    │BookingEmailAndSms│
    │TaskService       │
    │.confirm/reschedule│
    │/request          │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │BookingEmailSms   │
    │Handler.send()    │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ 实际邮件发送      │
    │ (nodemailer)     │
    └──────────────────┘
```

#### 邮件任务调度器实现

`BookingEmailAndSmsTasker` 位于 `packages/features/bookings/lib/tasker/BookingEmailAndSmsTasker.ts:16`：

```typescript
export class BookingEmailAndSmsTasker extends Tasker<IBookingEmailAndSmsTasker> {
  constructor(public readonly dependencies: IBookingEmailAndSmsTaskerDependencies) {
    super(dependencies);
  }

  public async send(data: {
    action: BookingActionType;
    schedulingType: SchedulingType | null;
    payload: BookingEmailAndSmsTaskPayload;
  }): Promise<{ runId: string }> {
    const { action, schedulingType, payload } = data;
    let taskResponse: { runId: string } = { runId: "task-not-found" };

    try {
      // 根据动作类型分发到不同的任务
      if (action === BookingActionMap.rescheduled) {
        if (schedulingType === "ROUND_ROBIN") {
          taskResponse = await this.dispatch("rrReschedule", payload);
        } else {
          taskResponse = await this.dispatch("reschedule", payload);
        }
      }
      if (action === BookingActionMap.confirmed) {
        taskResponse = await this.dispatch("confirm", payload);
      }
      if (action === BookingActionMap.requested) {
        taskResponse = await this.dispatch("request", payload);
      }

      this.logger.info(`BookingEmailAndSmsTasker send ${action} success:`, taskResponse, {
        bookingId: payload.bookingId,
      });
    } catch {
      taskResponse = { runId: "task-failed" };
      this.logger.error(`BookingEmailAndSmsTasker send ${action} failed`, taskResponse, {
        bookingId: payload.bookingId,
      });
    }

    return taskResponse;
  }
}
```

#### 邮件动作类型

| 动作类型 | `BookingActionMap` | 触发场景 |
|---------|-------------------|---------|
| `confirm` | `confirmed` | 预约确认时 |
| `request` | `requested` | 需要确认的预约提交时 |
| `reschedule` | `rescheduled` | 普通预约改期时 |
| `rrReschedule` | `rescheduled` | Round Robin 预约改期时 |

#### Trigger.dev 邮件任务定义

以 `confirm` 任务为例，位于 `packages/features/bookings/lib/tasker/trigger/notifications/confirm.ts:6`：

```typescript
export const confirm = schemaTask({
  id: "booking.send.confirm.notifications",
  ...bookingNotificationsTaskConfig,
  schema: bookingNotificationTaskSchema,
  run: async (payload) => {
    // 动态导入依赖（避免循环依赖）
    const { TriggerDevLogger } = await import("@calcom/lib/triggerDevLogger");
    const { BookingEmailSmsHandler } = await import("@calcom/features/bookings/lib/BookingEmailSmsHandler");
    const { BookingRepository } = await import("@calcom/features/bookings/repositories/BookingRepository");
    const { prisma } = await import("@calcom/prisma");
    const { BookingEmailAndSmsTaskService } = await import("../../BookingEmailAndSmsTaskService");

    // 初始化服务
    const triggerDevLogger = new TriggerDevLogger();
    const emailsAndSmsHandler = new BookingEmailSmsHandler({ logger: triggerDevLogger });
    const bookingRepo = new BookingRepository(prisma);
    const bookingTaskService = new BookingEmailAndSmsTaskService({
      logger: triggerDevLogger,
      bookingRepository: bookingRepo,
      emailsAndSmsHandler: emailsAndSmsHandler,
    });

    // 执行发送逻辑
    await bookingTaskService.confirm(payload);
  },
});
```

### 3.3 SMS 渠道

**当前状态**：SMS 功能在代码中已定义框架但未实现。

位于 `packages/features/tasker/tasks/index.ts:15`：

```typescript
const tasks: Record<TaskTypes, () => Promise<TaskHandler>> = {
  // ...
  sendSms: () => Promise.resolve(() => Promise.reject(new Error("Not implemented"))),
  // ...
};
```

**数据库支持**：旧的 `Workflow` 系统中曾有 `SMS_ATTENDEE`、`SMS_NUMBER` 等动作类型，但随着 Workflow 表的删除，这些功能也已废弃。

### 3.4 Slack 渠道

**当前状态**：代码库中没有找到直接的 Slack 集成实现。

- `packages/app-store/zapier/DESCRIPTION.md` 中提到了 Slack，但这是通过 Zapier 间接集成的
- 系统通过 Webhook 机制支持与 Slack 的集成（用户可以配置 Slack 的 Incoming Webhook URL）

**推荐集成方式**：
1. 使用 Webhook 渠道，配置 Slack 的 Incoming Webhook URL
2. 通过 Zapier/Make 等自动化平台中转

### 3.5 CRM 渠道

CRM 集成通过 `createCRMEvent` 任务实现，位于 `packages/features/tasker/tasks/crm/createCRMEvent.ts`。

**任务配置**（`packages/features/tasker/tasks/index.ts:28`）：

```typescript
export const tasksConfig = {
  createCRMEvent: {
    minRetryIntervalMins: IS_PRODUCTION ? 10 : 1,  // 重试间隔
    maxAttempts: 10,                                    // 最大重试 10 次
  },
  // ...
};
```

---

## 四、Tasker 任务调度系统

### 4.1 架构设计

Tasker 系统采用了**双模式设计**，支持异步（Trigger.dev）和同步（直接执行）两种执行模式：

```
                    ┌──────────────────┐
                    │    Tasker<T>     │
                    │   (抽象基类)      │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
       ┌──────────┐   ┌──────────┐   ┌──────────┐
       │Webhook-  │   │Booking-  │   │ 其他...  │
       │Tasker    │   │EmailAnd- │   │          │
       │          │   │SmsTasker │   │          │
       └────┬─────┘   └────┬─────┘   └────┬─────┘
            │               │               │
      ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
      │AsyncTasker│   │AsyncTasker│   │AsyncTasker│
      │(Trigger.  │   │(Trigger.  │   │(Trigger.  │
      │ dev)      │   │ dev)      │   │ dev)      │
      └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
            │               │               │
      ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
      │SyncTasker │   │SyncTasker │   │SyncTasker │
      │(直接执行)  │   │(直接执行)  │   │(直接执行)  │
      └───────────┘   └───────────┘   └───────────┘
```

### 4.2 模式选择逻辑

`Tasker` 基类在构造函数中决定使用哪种模式，位于 `packages/lib/tasker/Tasker.ts:9`：

```typescript
const isAsyncTaskerEnabled =
  ENABLE_ASYNC_TASKER && process.env.TRIGGER_SECRET_KEY && process.env.TRIGGER_API_URL;

export abstract class Tasker<T> {
  protected readonly asyncTasker: T;
  protected readonly syncTasker: T;
  protected readonly logger: ILogger;

  constructor(dependencies: {
    asyncTasker: T;
    syncTasker: T;
    logger: ILogger;
  }) {
    this.logger = dependencies.logger;

    // 检查环境变量配置
    if (!isAsyncTaskerEnabled) {
      if (ENABLE_ASYNC_TASKER && (!process.env.TRIGGER_SECRET_KEY || !process.env.TRIGGER_API_URL)) {
        this.logger.info(
          "Missing env variables TRIGGER_SECRET_KEY or TRIGGER_API_URL, falling back to Sync tasker."
        );
      }
    }

    // 配置 Trigger.dev（如果启用）
    if (isAsyncTaskerEnabled) {
      configure({
        accessToken: process.env.TRIGGER_SECRET_KEY,
        baseURL: process.env.TRIGGER_API_URL,
      });
    }

    // 选择实际使用的 tasker
    this.asyncTasker = isAsyncTaskerEnabled ? dependencies.asyncTasker : dependencies.syncTasker;
    this.syncTasker = dependencies.syncTasker;
  }
  // ...
}
```

**环境变量说明**：

| 变量 | 作用 |
|------|------|
| `ENABLE_ASYNC_TASKER` | 功能开关（E2E 测试时自动为 false） |
| `TRIGGER_SECRET_KEY` | Trigger.dev 认证密钥 |
| `TRIGGER_API_URL` | Trigger.dev API 地址 |
| `TASKER_ENABLE_WEBHOOKS` | Webhook 任务开关（控制是直接发送还是入队） |

### 4.3 任务分发与安全执行

`dispatch` 方法是任务执行的入口，位于 `packages/lib/tasker/Tasker.ts:43`：

```typescript
public async dispatch<K extends keyof T>(
  taskName: K,
  ...args: T[K] extends (...args: any[]) => any ? Parameters<T[K]> : never
): Promise<T[K] extends (...args: any[]) => any ? Awaited<ReturnType<T[K]>> : never> {
  this.logger.info(`Safely Dispatching task '${String(taskName)}'`, { args });
  return this._safeDispatch(taskName, ...args);
}

private async _safeDispatch<K extends keyof T>(
  taskName: K,
  ...args: T[K] extends (...args: any[]) => any ? Parameters<T[K]> : never
): Promise<T[K] extends (...args: any[]) => any ? Awaited<ReturnType<T[K]>> : never> {
  try {
    this.logger.info(
      `${isAsyncTaskerEnabled ? "AsyncTasker" : "SyncTasker"} '${String(taskName)}' dispatched.`
    );
    const method = this.asyncTasker[taskName] as (...args: any[]) => any;
    return await method.apply(this.asyncTasker, args);
  } catch (err) {
    // 记录错误
    const taskerLabel = isAsyncTaskerEnabled ? "AsyncTasker" : "SyncTasker";
    const baseUrlInfo = isAsyncTaskerEnabled
      ? ` (baseURL: ${process.env.TRIGGER_API_URL ?? "unknown"})`
      : "";
    this.logger.error(
      `${taskerLabel} failed for '${String(taskName)}'.${baseUrlInfo}`,
      this.getErrorDetails(err)
    );

    // 关键：如果 AsyncTasker 失败，尝试回退到 SyncTasker
    if (this.asyncTasker === this.syncTasker) {
      // 已经是 SyncTasker 了，无法再回退
      throw err;
    }

    this.logger.warn(`Trying again with SyncTasker for '${String(taskName)}'.`);

    try {
      const fallbackMethod = this.syncTasker[taskName] as (...args: any[]) => any;
      return await fallbackMethod.apply(this.syncTasker, args);
    } catch (err) {
      this.logger.error(`SyncTasker failed for '${String(taskName)}'.`, this.getErrorDetails(err));
      throw err;
    }
  }
}
```

**执行流程**：

```
dispatch() 调用
      │
      ▼
┌─────────────────┐
│ _safeDispatch() │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  尝试 Async-    │
│  Tasker 执行    │
└────────┬────────┘
         │
    ┌────┴────┐
    │ 成功？   │
    └────┬────┘
   是     │     否
         │
         ▼
┌─────────────────┐
│AsyncTasker ===  │
│SyncTasker?      │
└────────┬────────┘
   是     │     否
         │
         ▼
┌─────────────────┐
│  回退到 Sync-   │
│  Tasker 重试    │
└────────┬────────┘
         │
    ┌────┴────┐
    │ 成功？   │
    └────┬────┘
   是     │     否
         │       │
         ▼       ▼
      返回结果  抛出错误
```

### 4.4 Trigger.dev 全局重试配置

位于 `packages/features/trigger.config.ts:28`：

```typescript
export default defineConfig({
  // ...
  retries: {
    enabledInDev: false,        // 开发环境禁用重试（便于调试）
    default: {
      maxAttempts: 3,           // 默认最大重试 3 次
      minTimeoutInMs: 1000,     // 最小重试间隔 1 秒
      maxTimeoutInMs: 10000,    // 最大重试间隔 10 秒
      factor: 2,                 // 指数退避因子
      randomize: true,           // 随机化（避免惊群效应）
    },
  },
  // ...
  maxDuration: 600,             // 单个任务最大执行时间 10 分钟
});
```

**指数退避算法**：
- 第 1 次重试：~1 秒后
- 第 2 次重试：~2 秒后
- 第 3 次重试：~4 秒后
- 以此类推，直到 `maxTimeoutInMs`

---

## 五、失败恢复机制

### 5.1 多层次恢复策略

系统实现了**四层失败恢复机制**：

```
┌─────────────────────────────────────────────────────────────────┐
│                        第 1 层：即时回退                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Tasker._safeDispatch()                                       │ │
│  │ - AsyncTasker 失败 → 立即回退到 SyncTasker                   │ │
│  │ - 适用于 Trigger.dev 服务不可用的场景                         │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        第 2 层：Trigger.dev 重试                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ trigger.config.ts 中的 retries 配置                          │ │
│  │ - 指数退避：1s → 2s → 4s → ...                               │ │
│  │ - 默认最大 3 次尝试                                           │ │
│  │ - 适用于临时性网络故障、第三方服务限流等场景                    │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        第 3 层：内部任务重试                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ TaskRepository.retry() + TaskProcessor                       │ │
│  │ - 任务持久化到 PostgreSQL 的 Task 表                          │ │
│  │ - 可配置的重试间隔和最大尝试次数                               │ │
│  │ - 适用于需要持久化、可手动干预的场景                           │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        第 4 层：隔离性保障                         │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │ Promise.allSettled() + 独立错误日志                          │ │
│  │ - 单个 Webhook 失败不影响其他订阅者                           │ │
│  │ - 详细的错误记录便于排查                                       │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 第 1 层：即时回退（Async → Sync）

**场景**：Trigger.dev 服务不可用、API 密钥错误、网络分区等。

**实现位置**：`packages/lib/tasker/Tasker.ts:51`（`_safeDispatch` 方法）

**关键代码**：

```typescript
try {
  // 首先尝试 AsyncTasker (Trigger.dev)
  const method = this.asyncTasker[taskName] as (...args: any[]) => any;
  return await method.apply(this.asyncTasker, args);
} catch (err) {
  // 记录错误
  this.logger.error(
    `${taskerLabel} failed for '${String(taskName)}'.${baseUrlInfo}`,
    this.getErrorDetails(err)
  );

  // 检查是否可以回退
  if (this.asyncTasker === this.syncTasker) {
    // 已经是 SyncTasker，无法回退
    throw err;
  }

  // 回退到 SyncTasker 重新执行
  this.logger.warn(`Trying again with SyncTasker for '${String(taskName)}'.`);

  try {
    const fallbackMethod = this.syncTasker[taskName] as (...args: any[]) => any;
    return await fallbackMethod.apply(this.syncTasker, args);
  } catch (err) {
    // SyncTasker 也失败了
    this.logger.error(`SyncTasker failed for '${String(taskName)}'.`, this.getErrorDetails(err));
    throw err;
  }
}
```

**回退条件**：
1. `isAsyncTaskerEnabled` 为 true（配置了 Trigger.dev）
2. `asyncTasker` 和 `syncTasker` 是不同的实例
3. AsyncTasker 执行抛出异常

### 5.3 第 2 层：Trigger.dev 重试

**场景**：临时性网络故障、第三方服务限流、短暂的服务不可用。

**配置位置**：`packages/features/trigger.config.ts:28`

**配置参数**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `maxAttempts` | 3 | 最大尝试次数（包括首次） |
| `minTimeoutInMs` | 1000 | 最小重试间隔（毫秒） |
| `maxTimeoutInMs` | 10000 | 最大重试间隔（毫秒） |
| `factor` | 2 | 指数退避因子 |
| `randomize` | true | 是否随机化间隔（避免惊群） |
| `enabledInDev` | false | 开发环境是否启用 |

**重试时间线示例**（默认配置）：

```
T=0s:  首次执行失败
T=1s:  第 1 次重试 (minTimeout)
T=3s:  第 2 次重试 (1s * 2)
T=7s:  第 3 次重试 (2s * 2) → 达到 maxAttempts，停止
```

### 5.4 第 3 层：内部任务重试（Task 表）

**场景**：需要持久化的任务、可手动干预的失败、跨服务调用失败。

#### 数据模型（Task 表）

关键字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | 任务唯一标识 |
| `type` | String | 任务类型（如 `sendWebhook`、`createCRMEvent`） |
| `payload` | String | 任务数据（JSON 序列化） |
| `scheduledAt` | DateTime | 调度执行时间 |
| `maxAttempts` | Int | 最大尝试次数 |
| `attempts` | Int | 已尝试次数 |
| `succeededAt` | DateTime | 成功时间（null 表示未成功） |
| `lastError` | String | 最后一次错误信息 |
| `lastFailedAttemptAt` | DateTime | 最后失败时间 |
| `referenceUid` | String | 关联业务 ID（用于取消任务） |

#### 任务处理器（TaskProcessor）

位于 `packages/features/tasker/task-processor.ts:9`：

```typescript
export class TaskProcessor {
  async processQueue(): Promise<void> {
    // 获取下一批待执行任务
    const tasks = await Task.getNextBatch();
    console.info(`Processing ${tasks.length} tasks`, tasks);

    const tasksPromises = tasks.map(async (task) => {
      console.info(
        `Processing task ${task.id}, attempt:${task.attempts} maxAttempts:${task.maxAttempts}`,
        task
      );

      // 获取任务处理器
      const taskHandlerGetter = tasksMap[task.type as keyof typeof tasksMap];
      if (!taskHandlerGetter) throw new Error(`Task handler not found for type ${task.type}`);
      
      const taskConfig = tasksConfig[task.type as keyof typeof tasksConfig];
      const taskHandler = await taskHandlerGetter();

      // 执行任务
      return taskHandler(task.payload, task.id)
        .then(async () => {
          // 成功：标记任务完成
          await Task.succeed(task.id);
        })
        .catch(async (error) => {
          // 失败：记录并重试
          console.info(`Retrying task ${task.id}: ${error}`);
          await Task.retry({
            taskId: task.id,
            lastError: error instanceof Error ? error.message : "Unknown error",
            minRetryIntervalMins:
              taskConfig && "minRetryIntervalMins" in taskConfig 
                ? taskConfig.minRetryIntervalMins 
                : null,
          });
        });
    });

    // 等待所有任务处理完成（隔离失败）
    const settled = await Promise.allSettled(tasksPromises);
    const failed = settled.filter((result) => result.status === "rejected");
    const succeeded = settled.filter((result) => result.status === "fulfilled");
    console.info({ failed, succeeded });
  }
}
```

#### 任务查询逻辑（`getNextBatch`）

位于 `packages/features/tasker/repository.ts:22`：

```typescript
const makeWhereUpcomingTasks = (): Prisma.TaskWhereInput => ({
  // 1. 尚未成功
  succeededAt: null,
  // 2. 已到调度时间
  scheduledAt: {
    lt: new Date(),
  },
  // 3. 未达到最大尝试次数
  attempts: {
    lt: {
      _ref: "maxAttempts",  // 引用同一行的 maxAttempts 字段
      _container: "Task",
    },
  },
});

async getNextBatch() {
  return this.deps.prismaClient.task.findMany({
    where: makeWhereUpcomingTasks(),
    orderBy: {
      scheduledAt: "asc",  // 按调度时间排序
    },
    take: 1000,           // 每批最多 1000 个任务
  });
}
```

#### 任务重试方法（`retry`）

位于 `packages/features/tasker/repository.ts:110`：

```typescript
async retry({
  taskId,
  lastError,
  minRetryIntervalMins,
}: {
  taskId: string;
  lastError?: string;
  minRetryIntervalMins?: number | null;
}) {
  const failedAttemptTime = new Date();
  
  // 计算下一次调度时间
  const updatedScheduledAt = minRetryIntervalMins
    ? new Date(failedAttemptTime.getTime() + 1000 * 60 * minRetryIntervalMins)
    : undefined;  // 如果没有配置间隔，保持原调度时间（立即重试）

  return this.deps.prismaClient.task.update({
    where: { id: taskId },
    data: {
      attempts: { increment: 1 },                    // 尝试次数 +1
      lastError,                                      // 记录错误信息
      lastFailedAttemptAt: failedAttemptTime,        // 记录失败时间
      ...(updatedScheduledAt && {
        scheduledAt: updatedScheduledAt,             // 更新下次调度时间
      }),
    },
  });
}
```

#### 任务成功方法（`succeed`）

位于 `packages/features/tasker/repository.ts:139`：

```typescript
async succeed(taskId: string) {
  return this.deps.prismaClient.task.update({
    where: { id: taskId },
    data: {
      attempts: { increment: 1 },      // 尝试次数 +1
      succeededAt: new Date(),          // 标记成功时间
    },
  });
}
```

#### 任务类型配置

位于 `packages/features/tasker/tasks/index.ts:27`：

```typescript
export const tasksConfig = {
  createCRMEvent: {
    minRetryIntervalMins: IS_PRODUCTION ? 10 : 1,   // 生产环境 10 分钟，开发 1 分钟
    maxAttempts: 10,                                    // 最多重试 10 次
  },
  webhookDelivery: {
    minRetryIntervalMins: IS_PRODUCTION ? 5 : 1,    // 生产环境 5 分钟
    maxAttempts: 3,                                     // 最多重试 3 次
  },
};
```

#### 任务取消机制

系统支持通过 `referenceUid` 取消已调度的任务，位于 `packages/features/tasker/repository.ts:176`：

```typescript
async cancelWithReference(referenceUid: string, type: TaskTypes): Promise<{ id: string } | null> {
  try {
    return await this.deps.prismaClient.task.delete({
      where: {
        referenceUid_type: {    // 复合唯一索引
          referenceUid,
          type,
        },
      },
      select: { id: true },
    });
  } catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError && error.code === "P2025") {
      // P2025 = "Record to delete does not exist"
      console.warn(`Task with reference ${referenceUid} and type ${type} does not exist.`);
      return null;
    }
    throw error;
  }
}
```

**使用场景**：预约取消时，取消相关的 `MEETING_STARTED`、`MEETING_ENDED` 等预定任务。

示例（`packages/features/webhooks/lib/scheduleTrigger.ts:597`）：

```typescript
export async function cancelNoShowTasksForBooking({
  bookingUid,
  triggerEvent,
}: {
  bookingUid?: string;
  triggerEvent?: WebhookTriggerEvents;
}) {
  if (bookingUid) {
    if (triggerEvent === WebhookTriggerEvents.AFTER_HOSTS_CAL_VIDEO_NO_SHOW) {
      await tasker.cancelWithReference(bookingUid, "triggerHostNoShowWebhook");
    } else if (triggerEvent === WebhookTriggerEvents.AFTER_GUESTS_CAL_VIDEO_NO_SHOW) {
      await tasker.cancelWithReference(bookingUid, "triggerGuestNoShowWebhook");
    }
    // ...
  }
}
```

### 5.5 第 4 层：隔离性保障

**场景**：多个订阅者中部分失败，或同一工作流中多个步骤部分失败。

#### Webhook 处理的隔离性

位于 `packages/features/webhooks/lib/service/WebhookService.ts:135`：

```typescript
async processWebhooks(
  trigger: WebhookTriggerEvents,
  payload: WebhookPayload,
  subscribers: WebhookSubscriber[]
): Promise<void> {
  // ...

  const promises = subscribers.map(async (subscriber) => {
    try {
      // 每个订阅者独立处理
      const result = await this.sendWebhook(trigger, payload, subscriber);
      
      if (result.ok) {
        this.log.debug(`Webhook sent successfully`, {
          trigger, webhookId: subscriber.id, statusCode: result.status,
        });
      } else {
        this.log.error(`Webhook failed`, {
          error: result.message, trigger, webhookId: subscriber.id,
        });
      }
    } catch (err) {
      this.log.error("Error sending webhook", {
        error: err instanceof Error ? err.message : String(err),
        trigger, webhookId: subscriber.id,
      });
      // 重新抛出以确保 Promise.allSettled 捕获失败
      throw err;
    }
  });

  // 关键：使用 Promise.allSettled 而非 Promise.all
  // 这样单个 Webhook 失败不会影响整个处理流程
  const results = await Promise.allSettled(promises);

  // 统计和日志
  const successCount = results.filter((r) => r.status === "fulfilled").length;
  const failureCount = results.filter((r) => r.status === "rejected").length;

  this.log.info(`Webhook processing completed for ${trigger}`, {
    totalSubscribers: subscribers.length,
    successful: successCount,
    failed: failureCount,
  });

  // 记录每个失败的详情
  results.forEach((result, index) => {
    if (result.status === "rejected") {
      const subscriber = subscribers[index];
      this.log.error(`Webhook processing failed for subscriber`, {
        trigger, webhookId: subscriber?.id,
        subscriberUrl: subscriber?.subscriberUrl,
        error: result.reason,
      });
    }
  });
}
```

**为什么使用 `Promise.allSettled`**：

| 方法 | 行为 | 适用场景 |
|------|------|---------|
| `Promise.all` | 任一 Promise 拒绝则立即拒绝 | 需要所有操作都成功的场景 |
| `Promise.allSettled` | 等待所有 Promise 完成（无论成功或失败） | 需要隔离各个操作的场景 |

**示例**：

假设有 3 个 Webhook 订阅者：
- 订阅者 A：成功
- 订阅者 B：失败（网络超时）
- 订阅者 C：成功

使用 `Promise.allSettled`：
- 订阅者 A 和 C 正常发送
- 订阅者 B 的失败被记录，但不影响其他订阅者
- 处理流程完整完成

### 5.6 失败恢复路径总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        任务执行失败恢复路径                                    │
└─────────────────────────────────────────────────────────────────────────────┘

  任务开始
     │
     ▼
┌─────────────────┐
│  尝试执行任务    │
│  (AsyncTasker)  │
└────────┬────────┘
         │
    ┌────┴────┐
    │ 成功？   │
    └────┬────┘
   是     │     否
         │
         ▼
┌─────────────────────────┐
│ 第 1 层：Async → Sync   │
│ 检查是否可以回退         │
└───────────┬─────────────┘
            │
      ┌─────┴─────┐
      │ 可回退？   │
      └─────┬─────┘
     是      │      否
            │
            ▼
┌─────────────────────────┐
│ 使用 SyncTasker 重试    │
└───────────┬─────────────┘
            │
      ┌─────┴─────┐
      │ 成功？     │
      └─────┬─────┘
     是      │      否
            │
            ▼
┌─────────────────────────┐
│ 第 2 层：Trigger.dev    │
│ 指数退避重试             │
│ (如果使用 Trigger.dev)  │
└───────────┬─────────────┘
            │
      ┌─────┴─────┐
      │ 成功？     │
      └─────┬─────┘
     是      │      否
            │
            ▼
┌─────────────────────────┐
│ 第 3 层：Task 表持久化  │
│ 记录错误，调度下次重试   │
│ 检查 attempts < max-    │
│ Attempts                │
└───────────┬─────────────┘
            │
      ┌─────┴─────┐
      │ 还有重试？  │
      └─────┬─────┘
     是      │      否
            │
            ▼              ▼
    ┌─────────────┐  ┌──────────────────┐
    │ 等待下次调度 │  │ 第 4 层：最终失败 │
    │ 继续重试     │  │ 记录到 lastError  │
    └─────────────┘  │ 人工介入/告警     │
                     └──────────────────┘
```

---

## 六、典型场景分析

### 6.1 场景 1：新预约创建 → 发送确认邮件 + 触发 Webhook

**流程**：

```
用户完成预约
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│ RegularBookingService.create()                               │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ 邮件通知     │  │ Webhook通知  │  │ 其他...     │
    └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
           │                  │                  │
           ▼                  ▼                  ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│BookingEmailAndSms│ │  WebhookService  │ │  ...             │
│Tasker.send()     │ │.processWebhooks()│ │                  │
└────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
         │                     │                     │
         ▼                     ▼                     ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│  入队到 Trigger.  │ │  入队到 Task 表  │ │  ...             │
│  dev 或直接执行   │ │  或直接发送       │ │                  │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

**关键代码**：

预约服务中触发邮件和 Webhook（示例逻辑）：

```typescript
// 1. 发送邮件通知
await bookingEmailAndSmsTasker.send({
  action: BookingActionMap.confirmed,
  schedulingType: eventType.schedulingType ?? null,
  payload: { bookingId: newBooking.id, ... },
});

// 2. 触发 Webhook
const webhookPayload = buildWebhookPayload(newBooking);
await webhookService.processWebhooks(
  WebhookTriggerEvents.BOOKING_CREATED,
  webhookPayload,
  subscribers
);
```

### 6.2 场景 2：会议开始时触发 Webhook（时间触发）

**流程**：

```
预约创建时
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│ scheduleTrigger()                                            │
│ - 计算 startAfter = booking.startTime                        │
│ - 写入 WebhookScheduledTriggers 表                           │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ 等待...
                            │
                    会议开始时间到达
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 定时任务（Cron）扫描 WebhookScheduledTriggers                │
│ 查找 startAfter < NOW() 的记录                                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ 触发 MEETING_STARTED 事件                                     │
│ 调用 WebhookService.processWebhooks()                        │
└─────────────────────────────────────────────────────────────┘
```

**关键代码**（`scheduleTrigger.ts:282`）：

```typescript
export async function scheduleTrigger({
  booking,
  subscriberUrl,
  subscriber,
  triggerEvent,
}: {
  booking: { id: number; endTime: Date; startTime: Date };
  subscriberUrl: string;
  subscriber: { id: string; appId: string | null };
  triggerEvent: WebhookTriggerEvents;
}) {
  const payload = JSON.stringify({ triggerEvent, ...booking });

  await prisma.webhookScheduledTriggers.create({
    data: {
      payload,
      startAfter: triggerEvent === WebhookTriggerEvents.MEETING_ENDED 
        ? booking.endTime   // 会议结束触发
        : booking.startTime, // 会议开始触发
      subscriberUrl,
      webhook: { connect: { id: subscriber.id } },
      booking: { connect: { id: booking.id } },
    },
  });
}
```

### 6.3 场景 3：预约取消时清理预定任务

**流程**：

```
用户取消预约
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 触发 BOOKING_CANCELLED Webhook                            │
│ 2. 调用 cancelNoShowTasksForBooking()                        │
│ 3. 调用 cancelScheduledWebhooks()                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            │               │               │
            ▼               ▼               ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ 取消 NoShow 任务  │ │ 取消预定 Webhook  │ │ 删除 Webhook-    │
│ (Task 表)        │ │ (Task 表)        │ │ ScheduledTriggers │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

**关键代码**（`WebhookService.ts:256`）：

```typescript
async cancelScheduledWebhooks(
  bookingId: number,
  triggers: WebhookTriggerEvents[] = [
    WebhookTriggerEvents.MEETING_STARTED,
    WebhookTriggerEvents.MEETING_ENDED,
  ],
  isDryRun = false
): Promise<void> {
  if (isDryRun) return;

  try {
    const cancellationPromises = triggers.map(async (trigger) => {
      const referenceUid = `booking-${bookingId}-${trigger}`;
      try {
        // 通过 referenceUid 取消任务
        await this.tasker.cancelWithReference(referenceUid, "sendWebhook");
        return { trigger, success: true };
      } catch (error) {
        this.log.warn(`Failed to cancel webhook for trigger ${trigger}:`, {
          error: error instanceof Error ? error.message : String(error),
        });
        return { trigger, success: false, error };
      }
    });

    // 隔离各取消操作
    await Promise.allSettled(cancellationPromises);
  } catch (error) {
    this.log.error("Failed to cancel scheduled webhooks", {
      bookingId, triggers,
      error: error instanceof Error ? error.message : String(error),
    });
  }
}
```

### 6.4 场景 4：Webhook 发送失败的完整恢复路径

**假设场景**：
- 用户配置了一个 Webhook 订阅 `BOOKING_CREATED` 事件
- 新预约创建时，目标服务暂时不可用（503 Service Unavailable）

**恢复路径**：

```
步骤 1：新预约创建
         │
         ▼
步骤 2：WebhookService.processWebhooks()
         │
         ▼
步骤 3：检查 TASKER_ENABLE_WEBHOOKS
         │
    ┌────┴────┐
    │ = 1?    │
    └────┬────┘
   是     │     否
         │
         ▼              ▼
步骤 4a: 入队任务    步骤 4b: 直接发送
         │              │
         │              ▼
         │         发送失败
         │         记录日志
         │         结束（无重试）
         │
         ▼
步骤 5：TaskProcessor 处理任务
         │
         ▼
步骤 6：调用 sendWebhook()
         │
         ▼
步骤 7：fetch() 请求失败（503）
         │
         ▼
步骤 8：Task.retry()
         - attempts += 1
         - 记录 lastError = "503 Service Unavailable"
         - scheduledAt = NOW + 5 分钟（生产环境）
         │
         ▼
步骤 9：等待 5 分钟...
         │
         ▼
步骤 10：TaskProcessor 再次拾取任务
          │
          ▼
步骤 11：检查 attempts < maxAttempts（3）
          │
     ┌────┴────┐
     │ 是？    │
     └────┬────┘
    是     │     否
          │
          ▼              ▼
步骤 12a: 重试发送   步骤 12b: 超过最大次数
          │              │
          ▼              ▼
    ┌─────┴─────┐     记录错误
    │ 成功？     │     停止重试
    └─────┴─────┘
   是      │      否
          │
          ▼              ▼
    Task.succeed()    再次 Task.retry()
    标记成功            attempts = 3
                      下次 scheduledAt
                      但不会再被处理
                      （因为 attempts == maxAttempts）
```

---

## 七、配置参考

### 7.1 环境变量

| 变量 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ENABLE_ASYNC_TASKER` | boolean | false | 是否启用 Trigger.dev 异步任务 |
| `TRIGGER_SECRET_KEY` | string | - | Trigger.dev API 密钥 |
| `TRIGGER_API_URL` | string | - | Trigger.dev API 地址 |
| `TASKER_ENABLE_WEBHOOKS` | string (0/1) | - | Webhook 是否使用任务队列 |
| `TRIGGER_DEV_VERCEL_ACCESS_TOKEN` | string | - | Vercel 访问令牌（用于同步环境变量） |
| `TRIGGER_DEV_VERCEL_PROJECT_ID` | string | - | Vercel 项目 ID |
| `TRIGGER_DEV_VERCEL_TEAM_ID` | string | - | Vercel 团队 ID |
| `TRIGGER_DEV_PROJECT_REF` | string | - | Trigger.dev 项目引用 |

### 7.2 Trigger.dev 重试配置（全局）

位于 `packages/features/trigger.config.ts`：

```typescript
retries: {
  enabledInDev: false,
  default: {
    maxAttempts: 3,
    minTimeoutInMs: 1000,
    maxTimeoutInMs: 10000,
    factor: 2,
    randomize: true,
  },
}
```

### 7.3 任务类型特定配置

位于 `packages/features/tasker/tasks/index.ts`：

| 任务类型 | minRetryIntervalMins | maxAttempts |
|---------|---------------------|-------------|
| `createCRMEvent` | 10（生产）/ 1（开发） | 10 |
| `webhookDelivery` | 5（生产）/ 1（开发） | 3 |

### 7.4 WebhookTriggerEvents 完整列表

```typescript
enum WebhookTriggerEvents {
  // 预约生命周期
  BOOKING_CREATED,
  BOOKING_PAYMENT_INITIATED,
  BOOKING_PAID,
  BOOKING_RESCHEDULED,
  BOOKING_REQUESTED,
  BOOKING_CANCELLED,
  BOOKING_REJECTED,
  BOOKING_NO_SHOW_UPDATED,
  
  // 表单
  FORM_SUBMITTED,
  FORM_SUBMITTED_NO_EVENT,
  
  // 会议时间触发
  MEETING_STARTED,
  MEETING_ENDED,
  
  // 录制
  RECORDING_READY,
  RECORDING_TRANSCRIPTION_GENERATED,
  
  // 休假
  OOO_CREATED,
  
  // 未出席检测
  AFTER_HOSTS_CAL_VIDEO_NO_SHOW,
  AFTER_GUESTS_CAL_VIDEO_NO_SHOW,
  
  // 错误/报告
  DELEGATION_CREDENTIAL_ERROR,
  WRONG_ASSIGNMENT_REPORT,
}
```

---

## 八、文件索引

### 8.1 核心文件

| 文件路径 | 说明 |
|---------|------|
| `packages/lib/tasker/Tasker.ts` | Tasker 基类，包含 Async→Sync 回退逻辑 |
| `packages/features/tasker/task-processor.ts` | 任务队列处理器 |
| `packages/features/tasker/repository.ts` | Task 表数据访问层 |
| `packages/features/tasker/tasks/index.ts` | 任务处理器映射和配置 |
| `packages/features/webhooks/lib/service/WebhookService.ts` | Webhook 服务核心逻辑 |
| `packages/features/webhooks/lib/tasker/WebhookTasker.ts` | Webhook 任务调度器 |
| `packages/features/bookings/lib/tasker/BookingEmailAndSmsTasker.ts` | 邮件/SMS 任务调度器 |
| `packages/features/bookings/lib/tasker/BookingEmailAndSmsTaskService.ts` | 邮件发送业务逻辑 |
| `packages/features/trigger.config.ts` | Trigger.dev 全局配置 |
| `packages/prisma/schema.prisma` | 数据库模型定义 |

### 8.2 任务定义文件

| 文件路径 | 任务类型 | 说明 |
|---------|---------|------|
| `packages/features/tasker/tasks/sendWebook.ts` | `sendWebhook` | Webhook 发送（内部任务） |
| `packages/features/tasker/tasks/webhookDelivery.ts` | `webhookDelivery` | Webhook 发送（新版） |
| `packages/features/tasker/tasks/crm/createCRMEvent.ts` | `createCRMEvent` | CRM 事件创建 |
| `packages/features/webhooks/lib/tasker/trigger/deliver-webhook.ts` | - | Trigger.dev Webhook 任务 |
| `packages/features/bookings/lib/tasker/trigger/notifications/confirm.ts` | - | Trigger.dev 确认邮件任务 |
| `packages/features/bookings/lib/tasker/trigger/notifications/reschedule.ts` | - | Trigger.dev 改期邮件任务 |
| `packages/features/bookings/lib/tasker/trigger/notifications/request.ts` | - | Trigger.dev 请求邮件任务 |
| `packages/features/bookings/lib/tasker/trigger/notifications/rr-reschedule.ts` | - | Trigger.dev Round Robin 改期任务 |

---

## 九、注意事项与最佳实践

### 9.1 注意事项

1. **旧 Workflow 系统已废弃**：`Workflow`、`WorkflowStep`、`WorkflowReminder` 等表已在 2026 年 3 月删除，相关功能已迁移到 Task 和 Webhook 系统。

2. **SMS 功能未实现**：`sendSms` 任务处理器返回 `Promise.reject(new Error("Not implemented"))`，使用前需要实现。

3. **Slack 无直接集成**：系统通过 Webhook 间接支持 Slack，需要用户配置 Slack Incoming Webhook URL。

4. **Trigger.dev 依赖**：异步任务功能依赖外部 Trigger.dev 服务，需要确保环境变量配置正确。

### 9.2 最佳实践

1. **监控失败任务**：
   ```typescript
   // 获取所有失败的任务（达到最大重试次数）
   const failedTasks = await Task.getFailed();
   ```

2. **配置合理的重试策略**：
   - 对于外部 API 调用（Webhook、CRM），设置较长的重试间隔
   - 对于即时通知（邮件），可以设置较短的间隔

3. **使用 `referenceUid` 进行关联**：
   - 任务创建时设置 `referenceUid`（如 `booking-{id}`）
   - 业务取消时通过 `cancelWithReference` 清理相关任务

4. **隔离性设计**：
   - 使用 `Promise.allSettled` 处理多个独立操作
   - 单个订阅者/步骤失败不影响整体流程

5. **日志记录**：
   - 系统已内置详细的日志记录
   - 失败时记录 `webhookId`、`subscriberUrl`、`error` 等信息便于排查

---

## 十、总结

Cal.diy 的 Workflow 自动化机制采用了**分层设计**和**多模式执行**策略：

### 核心设计理念

1. **双模式执行**：
   - **Async 模式**：使用 Trigger.dev 实现异步、可重试的任务调度
   - **Sync 模式**：直接执行，用于 E2E 测试和回退场景
   - **自动回退**：Async 失败时自动尝试 Sync

2. **四层失败恢复**：
   - **L1 即时回退**：Async → Sync 模式切换
   - **L2 Trigger.dev 重试**：指数退避，最多 3 次
   - **L3 Task 表持久化重试**：可配置间隔，最多 10 次
   - **L4 隔离性保障**：`Promise.allSettled` 隔离失败

3. **事件类型丰富**：
   - 即时触发事件（预约创建、取消等）
   - 时间触发事件（会议开始、结束等）
   - 异步检测事件（未出席检测等）

4. **外部渠道**：
   - **Webhook**：完整支持，含签名验证
   - **邮件**：完整支持，Trigger.dev 集成
