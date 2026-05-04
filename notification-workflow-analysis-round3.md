# 预约通知触发路径分析（Round 3）

## 核心修正声明

**第一轮和第二轮关于"工作流"的结论存在以下问题需要修正：**

1. **代码定位错误**：Webhook 触发存在**两种独立路径**，之前混淆了 `handleWebhookTrigger` 和 `webhookProducer.queue*Webhook`
2. **条件判断不精细**：`isBookingEmailSmsTaskerEnabled = false` 只影响邮件通知，不影响 Webhook 任务
3. **边界不清晰**：没有明确区分"可验证证据"和"不确定边界"
4. **Trigger.dev vs 旧 tasker**：存在两套异步任务系统，之前混淆了

---

## 第一部分：三类"workflow"相关代码的复核

### 一、"仍在运行"的代码：Trigger.dev 异步任务系统

#### 1.1 可验证证据

**证据 A：配置文件完整存在**

```typescript
// packages/features/trigger.config.ts:1-75
export default defineConfig({
  project: process.env.TRIGGER_DEV_PROJECT_REF ?? "",
  
  // 任务目录配置 - 这些目录下的任务会被 Trigger.dev 扫描
  dirs: [
    "./bookings/lib/tasker/trigger/notifications",
    "./calendars/lib/tasker/trigger",
    "./ee/billing/service/proration/tasker/trigger",
    "./ee/organizations/lib/billing/tasker/trigger",
    "./webhooks/lib/tasker/trigger",
  ],
  
  // 重试配置
  retries: {
    enabledInDev: false,
    default: {
      maxAttempts: 3,
      minTimeoutInMs: 1000,
      maxTimeoutInMs: 10000,
      factor: 2,
      randomize: true,
    },
  },
  
  maxDuration: 600,
});
```

**证据 B：任务定义文件完整存在**

| 任务类型 | 文件路径 | 任务 ID |
|----------|----------|---------|
| 预订确认通知 | `bookings/lib/tasker/trigger/notifications/confirm.ts` | `booking.send.confirm.notifications` |
| 预订重新安排 | `bookings/lib/tasker/trigger/notifications/reschedule.ts` | `booking.send.reschedule.notifications` |
| 轮询预订重新安排 | `bookings/lib/tasker/trigger/notifications/rr-reschedule.ts` | `booking.send.rr-reschedule.notifications` |
| 预订请求通知 | `bookings/lib/tasker/trigger/notifications/request.ts` | `booking.send.request.notifications` |
| Webhook 投递 | `webhooks/lib/tasker/trigger/deliver-webhook.ts` | `webhook.deliver` |
| 日历任务 | `calendars/lib/tasker/trigger/ensure-default-calendars.ts` | `calendars.ensure-default-calendars` |

**证据 C：任务触发器存在**

```typescript
// packages/features/bookings/lib/tasker/BookingEmailAndSmsTriggerTasker.ts:1-31
export class BookingEmailAndSmsTriggerDevTasker implements IBookingEmailAndSmsTasker {
  async confirm(payload: ...) {
    const { confirm } = await import("./trigger/notifications/confirm");
    const handle = await confirm.trigger(payload);  // 实际的 Trigger.dev 触发
    return { runId: handle.id };
  }
  // 类似的 reschedule、request、rrReschedule 方法
}
```

```typescript
// packages/features/webhooks/lib/tasker/WebhookTriggerTasker.ts:1-24
export class WebhookTriggerTasker implements IWebhookTasker {
  async deliverWebhook(payload: WebhookTaskPayload): Promise<WebhookDeliveryResult> {
    const { deliverWebhook } = await import("./trigger/deliver-webhook");
    const handle = await deliverWebhook.trigger(payload);  // 实际的 Trigger.dev 触发
    return { taskId: handle.id };
  }
}
```

**证据 D：异步/同步选择逻辑**

```typescript
// packages/lib/tasker/Tasker.ts:9-10
const isAsyncTaskerEnabled =
  ENABLE_ASYNC_TASKER && process.env.TRIGGER_SECRET_KEY && process.env.TRIGGER_API_URL;

// packages/lib/constants.ts:280-281
export const ENABLE_ASYNC_TASKER =
  process.env.ENABLE_ASYNC_TASKER === "true" && !process.env.NEXT_PUBLIC_IS_E2E && !IS_API_V2_E2E;
```

**触发条件（必须同时满足）**：
1. `ENABLE_ASYNC_TASKER === "true"`（且不是 E2E 测试）
2. `TRIGGER_SECRET_KEY` 环境变量存在
3. `TRIGGER_API_URL` 环境变量存在

#### 1.2 不确定边界

| 不确定项 | 说明 | 验证方法 |
|----------|------|----------|
| 环境变量配置 | `ENABLE_ASYNC_TASKER`、`TRIGGER_SECRET_KEY`、`TRIGGER_API_URL` 的实际值 | 检查 `.env` 或部署配置 |
| Trigger.dev 平台状态 | Trigger.dev 项目是否实际配置、任务是否已部署 | 检查 Trigger.dev dashboard |
| 不同部署环境差异 | 生产环境 vs 开发环境 vs 测试环境配置可能不同 | 检查各环境的环境变量 |

#### 1.3 关键修正：邮件通知被硬编码禁用，但 Webhook 不受影响

**之前的错误**：混淆了 `isBookingEmailSmsTaskerEnabled` 和 `ENABLE_ASYNC_TASKER` 的影响范围。

**实际情况**：

```typescript
// packages/features/bookings/lib/service/RegularBookingService.ts:2527-2554
const isBookingEmailSmsTaskerEnabled = false;  // 硬编码为 false - 只影响邮件

// 邮件通知的条件（额外限制）
if (ENABLE_ASYNC_TASKER && !noEmail && isBookingEmailSmsTaskerEnabled) {
  // 只有这里被禁用
  await deps.bookingEmailAndSmsTasker.send({ ... });
}

// 但 Webhook 通知不受此限制
// 详见下文"Webhook 两种触发路径"
```

**结论**：
- ❌ **邮件通知**：`isBookingEmailSmsTaskerEnabled = false` 硬编码禁用
- ⚠️ **Webhook 通知**：可能仍在运行，取决于 `ENABLE_ASYNC_TASKER` 和 Trigger.dev 配置

---

### 二、"历史遗留"的代码：原工作流引擎（已删除）

#### 2.1 可验证证据

**证据 A：数据库迁移明确删除**

```sql
-- packages/prisma/migrations/20260319000000_drop_workflow_tables/migration.sql:1-34
-- Drop workflow-related tables and columns (EE feature cleanup)

-- 1. Drop junction/child tables first
DROP TABLE IF EXISTS "WorkflowStepTranslation" CASCADE;
DROP TABLE IF EXISTS "WorkflowReminder" CASCADE;
DROP TABLE IF EXISTS "WorkflowsOnEventTypes" CASCADE;
DROP TABLE IF EXISTS "WorkflowsOnTeams" CASCADE;
DROP TABLE IF EXISTS "WorkflowOptOutContact" CASCADE;
DROP TABLE IF EXISTS "AIPhoneCallConfiguration" CASCADE;

-- 2. Drop WorkflowStep (references Workflow and Agent)
DROP TABLE IF EXISTS "WorkflowStep" CASCADE;

-- 3. Drop Workflow (references User and Team)
DROP TABLE IF EXISTS "Workflow" CASCADE;

-- 4. Remove relation fields from other tables
ALTER TABLE "users" DROP COLUMN IF EXISTS "whitelistWorkflows";

-- 5. Drop enum types
DROP TYPE IF EXISTS "WorkflowTriggerEvents";
DROP TYPE IF EXISTS "WorkflowActions";
DROP TYPE IF EXISTS "WorkflowType";
DROP TYPE IF EXISTS "WorkflowTemplates";
DROP TYPE IF EXISTS "WorkflowMethods";
DROP TYPE IF EXISTS "WorkflowStepAutoTranslatedField";
DROP TYPE IF EXISTS "WorkflowContactType";
```

**证据 B：当前 schema.prisma 无工作流模型**

```prisma
// packages/prisma/schema.prisma
// 搜索 "Workflow"、"workflow" 无结果
// 搜索 "Reminder" 只有：
enum ReminderType {
  PENDING_BOOKING_CONFIRMATION  // 唯一的提醒类型，与工作流无关
}
```

**证据 C：测试代码被注释**

```typescript
// packages/trpc/server/routers/viewer/bookings/confirm.handler.test.ts:113
// 原工作流触发测试被注释
// expectWorkflowToBeTriggered({ emailsToReceive: [organizer.email], emails });
```

#### 2.2 不确定边界

| 不确定项 | 说明 | 验证方法 |
|----------|------|----------|
| 测试数据残留用途 | 测试数据中仍有 `workflows` 字段，是单纯残留还是有其他用途？ | 检查测试运行时是否实际使用 |
| 迁移完整性 | 是否所有工作流相关代码都已清理？ | 搜索 "workflow" 关键词 |

---

### 三、"仅命名残留"的代码：用了"workflow"名称，但与工作流引擎无关

#### 3.1 案例 A：`packages/emails/workflow-email-service.ts`

**可验证证据**：

```typescript
// packages/emails/workflow-email-service.ts:1-31
// 文件名包含 "workflow"，但实际功能是：

const sendEmail = (prepare: () => BaseEmail) => {
  return new Promise((resolve, reject) => {
    try {
      const email = prepare();
      resolve(email.sendEmail());
    } catch (e) {
      reject(console.error(`${prepare.constructor.name}.sendEmail failed`, e));
    }
  });
};

// 导出的三个函数，都与工作流引擎无关：

export const sendFeedbackEmail = async (feedback: Feedback) => {
  await sendEmail(() => new FeedbackEmail(feedback));
};

export const sendMonthlyDigestEmail = async (eventData: MonthlyDigestEmailData) => {
  await sendEmail(() => new MonthlyDigestEmail(eventData));
};

export const sendBookingRedirectNotification = async (bookingRedirect: IBookingRedirect) => {
  await sendEmail(() => new BookingRedirectEmailNotification(bookingRedirect));
};
```

**实际调用者**：
- `packages/trpc/server/routers/viewer/ooo/outOfOfficeEntryDelete.handler.ts`
- `packages/trpc/server/routers/viewer/ooo/outOfOfficeCreateOrUpdate.handler.ts`

**用途**：发送预订重定向通知（当用户设置了 OOO 自动重定向时）

**结论**：**纯命名残留**，与工作流引擎完全无关。

#### 3.2 案例 B：Zapier 集成描述中的 "workflows"

**可验证证据**：

```typescript
// packages/app-store/zapier/_metadata.ts:4-8
description:
  "Workflow automation for everyone. Use the Cal.diy Zapier app to trigger your workflows when a booking is created, rescheduled, or cancelled, or after a meeting ends.",
```

**分析**：
- 这里的 "workflows" 指的是 **Zapier 平台的工作流**（用户在 Zapier 网站创建的自动化）
- 不是 Cal.diy 内部的工作流引擎
- 集成方式是通过 **Webhook**（Cal.diy 发送事件给 Zapier）

**结论**：**外部概念**，不是 Cal.diy 内部功能。

#### 3.3 案例 C：测试数据中的 `workflows` 字段

**可验证证据**：

```typescript
// packages/trpc/server/routers/viewer/bookings/confirm.handler.test.ts:51-59
// 测试数据中包含 workflows 字段
workflows: [
  {
    userId: organizer.id,
    trigger: "NEW_EVENT",
    action: "EMAIL_HOST",
    template: "REMINDER",
    activeOn: [1],
  },
],
```

**但实际触发测试被注释**：
```typescript
// 第 113 行
// expectWorkflowToBeTriggered({ emailsToReceive: [organizer.email], emails });
```

**不确定边界**：
- 这是单纯的测试数据残留？
- 还是测试框架需要这个字段但不实际使用？

**结论**：**大概率是测试数据残留**，但需要进一步验证测试运行时行为。

---

## 第二部分：Webhook 触发的两种独立路径（关键修正）

### 之前的错误

之前混淆了两种完全不同的 Webhook 触发机制：
1. `handleWebhookTrigger` → `sendOrSchedulePayload`（旧机制）
2. `webhookProducer.queue*Webhook` → `WebhookTasker`（新机制）

### 路径 A：`handleWebhookTrigger`（旧机制）

#### 代码位置

```typescript
// 触发点：packages/features/bookings/lib/service/RegularBookingService.ts
// 用于：BOOKING_CREATED、BOOKING_RESCHEDULED、BOOKING_PAYMENT_INITIATED、BOOKING_PAID
```

#### 调用链

```
handleWebhookTrigger()
    ↓
sendOrSchedulePayload()  ← 检查 TASKER_ENABLE_WEBHOOKS
    ↓
    ├─ 如果 TASKER_ENABLE_WEBHOOKS === "1"
    │       ↓
    │   schedulePayload() → tasker.create("sendWebhook", ...)  ← 旧 tasker
    │
    └─ 否则
            ↓
        sendPayload() → 直接 fetch() 发送 HTTP  ← 同步
```

#### 关键代码

```typescript
// packages/features/webhooks/lib/sendOrSchedulePayload.ts:1-10
const sendOrSchedulePayload: SendOrSchedulePayload = async (...args) => {
  // 检查的是 TASKER_ENABLE_WEBHOOKS，不是 ENABLE_ASYNC_TASKER
  if (process.env.TASKER_ENABLE_WEBHOOKS === "1") return schedulePayload(...args);
  return sendPayload(...args);
};

// packages/features/webhooks/lib/schedulePayload.ts:1-14
const schedulePayload: SchedulePayload = async (secretKey, triggerEvent, createdAt, webhook, data) => {
  // 调用的是旧 tasker.create，不是 Trigger.dev 的 .trigger()
  await tasker.create("sendWebhook", JSON.stringify({ secretKey, triggerEvent, createdAt, webhook, data }));
  return { ok: true, status: 200, message: "Webhook scheduled successfully" };
};
```

#### 直接发送的实现

```typescript
// packages/features/webhooks/lib/sendPayload.ts:301-327
const _sendPayload = async (
  secretKey: string | null,
  webhook: WebhookForPayload,
  body: string,
  contentType: "application/json" | "application/x-www-form-urlencoded"
) => {
  const { subscriberUrl, version } = webhook;
  
  // 直接发送 HTTP POST
  const response = await fetch(subscriberUrl, {
    method: "POST",
    headers: {
      "Content-Type": contentType,
      "X-Cal-Signature-256": createWebhookSignature({ secret: secretKey, body }),
      "X-Cal-Webhook-Version": version,
    },
    redirect: "manual",
    body,
  });

  return { ok: response.ok, status: response.status };
};
```

---

### 路径 B：`webhookProducer.queue*Webhook`（新机制）

#### 代码位置

```typescript
// 触发点：packages/features/bookings/lib/service/RegularBookingService.ts
// 用于：BOOKING_REQUESTED（预订状态为 PENDING 时）
```

#### 调用链

```
webhookProducer.queueBookingRequestedWebhook()
    ↓
WebhookTaskerProducerService.queueTask()
    ↓
webhookTasker.deliverWebhook()
    ↓
Tasker.dispatch()  ← 检查 ENABLE_ASYNC_TASKER && TRIGGER_* 环境变量
    ↓
    ├─ 如果满足条件
    │       ↓
    │   WebhookTriggerTasker → deliverWebhook.trigger(payload)  ← Trigger.dev
    │
    └─ 否则
            ↓
        WebhookSyncTasker → 同步执行
```

#### 关键代码

```typescript
// packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts:222-233
private async queueTask(operationId: string, taskPayload: WebhookTaskPayload): Promise<void> {
  try {
    const result = await this.deps.webhookTasker.deliverWebhook(taskPayload);
    this.log.debug("Webhook delivery task queued", { operationId, taskId: result.taskId });
  } catch (error) {
    // ...
  }
}

// packages/features/webhooks/lib/tasker/WebhookTasker.ts:36-43
export class WebhookTasker extends Tasker<IWebhookTasker> {
  async deliverWebhook(payload: WebhookTaskPayload): Promise<WebhookDeliveryResult> {
    return await this.dispatch("deliverWebhook", payload);
  }
}

// packages/lib/tasker/Tasker.ts:9-10
const isAsyncTaskerEnabled =
  ENABLE_ASYNC_TASKER && process.env.TRIGGER_SECRET_KEY && process.env.TRIGGER_API_URL;
```

---

### 两条路径对比表

| 维度 | 路径 A：`handleWebhookTrigger` | 路径 B：`webhookProducer.queue*Webhook` |
|------|--------------------------------|-------------------------------------------|
| **触发场景** | BOOKING_CREATED、BOOKING_RESCHEDULED、BOOKING_PAYMENT_* | BOOKING_REQUESTED（预订状态为 PENDING） |
| **环境变量** | `TASKER_ENABLE_WEBHOOKS === "1"` | `ENABLE_ASYNC_TASKER=true` + `TRIGGER_SECRET_KEY` + `TRIGGER_API_URL` |
| **异步机制** | 旧 `tasker.create("sendWebhook", ...)` | Trigger.dev `.trigger()` |
| **默认行为** | 同步发送（除非 `TASKER_ENABLE_WEBHOOKS=1`） | 取决于 Trigger.dev 配置 |
| **代码位置** | `packages/features/bookings/lib/handleWebhookTrigger.ts` | `packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts` |

---

## 第三部分：Notification 与 Reminder 的结构化对比

### 核心定义

| 概念 | 定义 | 关键特征 |
|------|------|----------|
| **Notification（通知）** | 事件发生时**立即**发送的信息 | 与事件同时发生，告知用户"某事已发生" |
| **Reminder（提醒）** | **预定的未来时间点**发送的信息 | 在事件之前/之后的特定时间发送，提醒用户"需要做某事"或"某事即将发生" |

---

### 对比表（一页总结）

| 维度 | Notification（通知） | Reminder（提醒） |
|------|----------------------|------------------|
| **触发时机** | 事件发生时**立即** | **预定的未来时间点** |
| **时间关系** | 与事件**同时**发生 | 在事件**之前或之后**的特定时间 |
| **目的** | 告知用户某个事件**已经发生** | 提醒用户**即将发生**的事件或**需要采取**的行动 |
| **持久化** | ❌ 无（发送后即完成，不记录状态） | ✅ 有（需要记录已发送状态，避免重复） |
| **取消机制** | ❌ 无（已发送不可取消） | ✅ 有（可删除预定任务） |
| **重复发送** | 一次 | 可多次（不同时间间隔） |

---

### Notification 详细分类

#### 类型 A：邮件/SMS 通知

| 项目 | 说明 |
|------|------|
| **触发条件** | 预订创建/确认/重新安排时 |
| **触发位置** | `RegularBookingService.ts` 中 `emailsAndSmsHandler.send()` |
| **执行方式** | 同步执行（`isBookingEmailSmsTaskerEnabled = false` 硬编码禁用异步） |
| **持久化** | 无 |
| **相关文件** | `packages/features/bookings/lib/BookingEmailSmsHandler.ts` |

#### 类型 B：Webhook 通知（路径 A）

| 项目 | 说明 |
|------|------|
| **触发条件** | 预订创建/重新安排/支付时 |
| **触发位置** | `RegularBookingService.ts` 中 `handleWebhookTrigger()` |
| **执行方式** | 默认同步 `fetch()`，或 `TASKER_ENABLE_WEBHOOKS=1` 时用旧 tasker |
| **持久化** | 无 |
| **相关文件** | `packages/features/bookings/lib/handleWebhookTrigger.ts` |

#### 类型 C：Webhook 通知（路径 B）

| 项目 | 说明 |
|------|------|
| **触发条件** | 预订请求（状态 PENDING）时 |
| **触发位置** | `RegularBookingService.ts` 中 `deps.webhookProducer.queueBookingRequestedWebhook()` |
| **执行方式** | 取决于 `ENABLE_ASYNC_TASKER` 和 Trigger.dev 配置 |
| **持久化** | 无 |
| **相关文件** | `packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts` |

---

### Reminder 详细分类

#### 类型 1：待确认预订提醒

| 项目 | 说明 |
|------|------|
| **触发条件** | 预订状态为 `PENDING`，且创建时间超过 48h/24h/3h |
| **调度方式** | 外部 Cron 定期调用 `/api/cron/bookingReminder` |
| **持久化表** | `ReminderMail`（记录已发送的提醒） |
| **取消机制** | 预订确认/取消后，Cron 不再匹配 |
| **相关文件** | `apps/web/app/api/cron/bookingReminder/route.ts` |

#### 类型 2：会议开始/结束 Webhook

| 项目 | 说明 |
|------|------|
| **触发条件** | 预订开始时间 / 预订结束时间 |
| **调度方式** | `scheduleTrigger()` 写入 `webhookScheduledTriggers` 表 |
| **持久化表** | `webhookScheduledTriggers` |
| **取消机制** | `deleteWebhookScheduledTriggers()` |
| **相关文件** | `packages/features/webhooks/lib/scheduleTrigger.ts` |

#### 类型 3：No-Show 检查

| 项目 | 说明 |
|------|------|
| **触发条件** | 会议开始后一段时间（仅 Cal Video） |
| **调度方式** | `scheduleNoShowTriggers()` 调用 `tasker.create()` 写入 `task` 表 |
| **持久化表** | `task`（`scheduledAt` 字段指定触发时间） |
| **取消机制** | `cancelNoShowTasksForBooking()` |
| **相关文件** | `packages/features/bookings/lib/handleNewBooking/scheduleNoShowTriggers.ts` |

---

## 第四部分：预约创建后的完整触发路径（修正版）

### 流程图

```
用户提交预约请求
         ↓
┌─────────────────────────────────────────────────────────────────┐
│                    RegularBookingService.handler()                │
│                    预订创建核心流程                                │
├─────────────────────────────────────────────────────────────────┤
│  1. 验证预订数据                                                   │
│  2. 检查用户可用性                                                 │
│  3. 创建预订记录（booking 表）                                    │
│  4. 日历同步                                                       │
└─────────────────────────────────────────────────────────────────┘
         ↓
         ┌─────────────────────────────────────────────────────────┐
         │              Notification（立即触发）                    │
         │              无持久化，发送后即完成                       │
         ├─────────────────────────────────────────────────────────┤
         │  分支 1：邮件/SMS 通知（同步）                           │
         │  ┌─────────────────────────────────────────────────┐    │
         │  │ 判断预订状态：                                    │    │
         │  │                                                   │    │
         │  │ ACCEPTED（已确认）                               │    │
         │  │   → BookingActionMap.confirmed                   │    │
         │  │   → emailsAndSmsHandler._handleConfirmed()      │    │
         │  │                                                   │    │
         │  │ RESCHEDULED（已重新安排）                        │    │
         │  │   → BookingActionMap.rescheduled                 │    │
         │  │   → emailsAndSmsHandler._handleRescheduled()    │    │
         │  │                                                   │    │
         │  │ PENDING（待确认）                                 │    │
         │  │   → BookingActionMap.requested                   │    │
         │  │   → emailsAndSmsHandler._handleRequested()      │    │
         │  │                                                   │    │
         │  │ ⚠️ 注意：异步路径被硬编码禁用                     │    │
         │  │ isBookingEmailSmsTaskerEnabled = false          │    │
         │  └─────────────────────────────────────────────────┘    │
         │                                                             │
         │  分支 2：Webhook 通知（路径 A - 旧机制）                  │
         │  ┌─────────────────────────────────────────────────┐    │
         │  │ 触发点：BOOKING_CREATED / BOOKING_RESCHEDULED   │    │
         │  │                  / BOOKING_PAYMENT_*             │    │
         │  │                                                   │    │
         │  │ handleWebhookTrigger()                           │    │
         │  │   → sendOrSchedulePayload()                      │    │
         │  │                                                   │    │
         │  │ 判断：TASKER_ENABLE_WEBHOOKS === "1" ?          │    │
         │  │                                                   │    │
         │  │ 是 → schedulePayload()                           │    │
         │  │      → tasker.create("sendWebhook", ...)        │    │
         │  │      → 旧异步任务系统                             │    │
         │  │                                                   │    │
         │  │ 否 → sendPayload()                               │    │
         │  │      → 直接 fetch() 发送 HTTP                    │    │
         │  │      → 同步执行                                   │    │
         │  └─────────────────────────────────────────────────┘    │
         │                                                             │
         │  分支 3：Webhook 通知（路径 B - 新机制）                  │
         │  ┌─────────────────────────────────────────────────┐    │
         │  │ 触发点：BOOKING_REQUESTED（状态 PENDING）        │    │
         │  │                                                   │    │
         │  │ webhookProducer.queueBookingRequestedWebhook()   │    │
         │  │   → WebhookTasker.deliverWebhook()              │    │
         │  │                                                   │    │
         │  │ 判断：ENABLE_ASYNC_TASKER &&                     │    │
         │  │       TRIGGER_SECRET_KEY &&                      │    │
         │  │       TRIGGER_API_URL ?                          │    │
         │  │                                                   │    │
         │  │ 是 → WebhookTriggerTasker                        │    │
         │  │      → deliverWebhook.trigger(payload)          │    │
         │  │      → Trigger.dev 异步任务                      │    │
         │  │                                                   │    │
         │  │ 否 → WebhookSyncTasker                           │    │
         │  │      → 同步执行                                   │    │
         │  └─────────────────────────────────────────────────┘    │
         └─────────────────────────────────────────────────────────┘
         ↓
         ┌─────────────────────────────────────────────────────────┐
         │              Reminder（预定触发）                        │
         │              有持久化，需记录发送状态                       │
         ├─────────────────────────────────────────────────────────┤
         │  类型 1：待确认预订提醒                                   │
         │  ┌─────────────────────────────────────────────────┐    │
         │  │ 无需预调度                                         │    │
         │  │                                                   │    │
         │  │ 触发方式：外部 Cron 定期调用                       │    │
         │  │         POST /api/cron/bookingReminder           │    │
         │  │                                                   │    │
         │  │ 触发条件检查：                                     │    │
         │  │   - booking.status === PENDING                    │    │
         │  │   - booking.createdAt <= now - 48h/24h/3h        │    │
         │  │   - booking.endTime >= now（预订未结束）          │    │
         │  │                                                   │    │
         │  │ 去重机制：查询 ReminderMail 表                     │    │
         │  │         避免重复发送同一间隔的提醒                  │    │
         │  │                                                   │    │
         │  │ 发送后：写入 ReminderMail 表                       │    │
         │  └─────────────────────────────────────────────────┘    │
         │                                                             │
         │  类型 2：会议开始/结束 Webhook                            │
         │  ┌─────────────────────────────────────────────────┐    │
         │  │ 预调度函数：scheduleTrigger()                      │    │
         │  │                                                   │    │
         │  │ 触发事件：                                         │    │
         │  │   - MEETING_STARTED → booking.startTime          │    │
         │  │   - MEETING_ENDED   → booking.endTime            │    │
         │  │                                                   │    │
         │  │ 持久化：写入 webhookScheduledTriggers 表           │    │
         │  │   - payload: 事件数据                             │    │
         │  │   - startAfter: 触发时间                          │    │
         │  │   - subscriberUrl: 回调地址                       │    │
         │  │                                                   │    │
         │  │ 取消：deleteWebhookScheduledTriggers()            │    │
         │  └─────────────────────────────────────────────────┘    │
         │                                                             │
         │  类型 3：No-Show 检查（仅 Cal Video）                     │
         │  ┌─────────────────────────────────────────────────┐    │
         │  │ 预调度函数：scheduleNoShowTriggers()              │    │
         │  │                                                   │    │
         │  │ 触发条件：                                         │    │
         │  │   - booking.location === DailyLocationType        │    │
         │  │   - 用户订阅了 AFTER_*_CAL_VIDEO_NO_SHOW         │    │
         │  │                                                   │    │
         │  │ 触发时间：                                         │    │
         │  │   booking.startTime + webhook.time（小时/分钟）   │    │
         │  │                                                   │    │
         │  │ 持久化：写入 task 表                               │    │
         │  │   - type: "triggerHostNoShowWebhook" 或          │    │
         │  │           "triggerGuestNoShowWebhook"             │    │
         │  │   - scheduledAt: 预定触发时间                      │    │
         │  │   - referenceUid: 预订 UID（用于取消）             │    │
         │  │                                                   │    │
         │  │ 取消：cancelNoShowTasksForBooking()               │    │
         │  └─────────────────────────────────────────────────┘    │
         └─────────────────────────────────────────────────────────┘
         ↓
    预订创建流程结束
         ↓
    等待 Reminder 触发时间到达
```

---

## 第五部分：关键修正总结

### 修正 1：代码定位错误

**之前的错误**：
- 混淆了 `handleWebhookTrigger` 和 `webhookProducer.queue*Webhook` 两条独立路径
- 误认为 `TASKER_ENABLE_WEBHOOKS` 和 `ENABLE_ASYNC_TASKER` 是同一回事

**修正后的理解**：

| 路径 | 环境变量 | 异步机制 | 触发场景 |
|------|----------|----------|----------|
| `handleWebhookTrigger` | `TASKER_ENABLE_WEBHOOKS` | 旧 `tasker.create()` | BOOKING_CREATED 等 |
| `webhookProducer.queue*Webhook` | `ENABLE_ASYNC_TASKER` + `TRIGGER_*` | Trigger.dev `.trigger()` | BOOKING_REQUESTED |

---

### 修正 2：条件判断不精细

**之前的错误**：
- 误认为 `isBookingEmailSmsTaskerEnabled = false` 禁用了所有异步任务

**修正后的理解**：

| 任务类型 | 受 `isBookingEmailSmsTaskerEnabled` 影响 | 受 `ENABLE_ASYNC_TASKER` 影响 |
|----------|-------------------------------------------|-------------------------------|
| 邮件通知 | ✅ 是（硬编码禁用） | 是（但被前者覆盖） |
| Webhook 路径 A | ❌ 否 | ❌ 否（用 `TASKER_ENABLE_WEBHOOKS`） |
| Webhook 路径 B | ❌ 否 | ✅ 是 |

**结论**：Webhook 异步任务可能仍在运行，取决于环境变量配置。

---

### 修正 3：工作流概念混淆

**之前的错误**：
- 没有区分"原工作流引擎"、"Trigger.dev 异步任务"、"命名残留"

**修正后的理解**：

| 概念 | 状态 | 说明 |
|------|------|------|
| **原工作流引擎** | ❌ 已删除 | 用户可配置的工作流（Workflow 表等），2026 年 3 月迁移删除 |
| **Trigger.dev 异步任务** | ⚠️ 可能运行 | 代码存在，运行取决于 `ENABLE_ASYNC_TASKER` 和 Trigger.dev 配置 |
| **workflow-email-service.ts** | ✅ 运行 | 纯命名残留，实际是邮件发送工具类 |
| **Zapier "workflows"** | ✅ 运行 | 外部概念，指 Zapier 平台的自动化 |

---

## 第六部分：不确定边界与验证建议

### 待验证事项

| 事项 | 验证方法 | 预期结果 |
|------|----------|----------|
| `ENABLE_ASYNC_TASKER` 实际值 | 检查 `.env` 或部署配置 | 决定 Webhook 路径 B 是否异步 |
| `TRIGGER_SECRET_KEY` 存在性 | 检查环境变量 | 决定 Trigger.dev 是否可用 |
| `TASKER_ENABLE_WEBHOOKS` 实际值 | 检查环境变量 | 决定 Webhook 路径 A 是否异步 |
| 测试数据 `workflows` 字段用途 | 运行测试并观察 | 确认是否为单纯残留 |
| Trigger.dev 任务部署状态 | 检查 Trigger.dev dashboard | 确认任务是否已部署 |

### 建议的验证命令

```bash
# 检查环境变量（示例）
echo $ENABLE_ASYNC_TASKER
echo $TRIGGER_SECRET_KEY
echo $TRIGGER_API_URL
echo $TASKER_ENABLE_WEBHOOKS

# 搜索工作流相关代码
rg "workflow" --type ts packages/features/

# 运行相关测试
yarn test packages/features/webhooks/lib/service/__tests__/WebhookTaskerProducerService.test.ts
```

---

## 附录：关键代码位置索引

### Notification 相关

| 文件路径 | 说明 |
|----------|------|
| `packages/features/bookings/lib/service/RegularBookingService.ts` | 预订服务主入口，所有通知的触发点 |
| `packages/features/bookings/lib/BookingEmailSmsHandler.ts` | 邮件/SMS 通知处理 |
| `packages/features/bookings/lib/handleWebhookTrigger.ts` | Webhook 路径 A（旧机制） |
| `packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts` | Webhook 路径 B（新机制） |
| `packages/features/webhooks/lib/sendOrSchedulePayload.ts` | Webhook 发送/调度选择 |
| `packages/features/webhooks/lib/sendPayload.ts` | Webhook 直接发送 |
| `packages/features/webhooks/lib/schedulePayload.ts` | Webhook 旧异步调度 |

### Reminder 相关

| 文件路径 | 说明 |
|----------|------|
| `apps/web/app/api/cron/bookingReminder/route.ts` | 待确认预订提醒 API |
| `packages/features/webhooks/lib/scheduleTrigger.ts` | 会议开始/结束 Webhook 调度 |
| `packages/features/bookings/lib/handleNewBooking/scheduleNoShowTriggers.ts` | No-Show 检查调度 |
| `packages/prisma/schema.prisma:1080-1093` | ReminderType 枚举和 ReminderMail 模型 |

### Trigger.dev 任务相关

| 文件路径 | 说明 |
|----------|------|
| `packages/features/trigger.config.ts` | Trigger.dev 主配置 |
| `packages/lib/tasker/Tasker.ts` | 异步/同步任务选择基类 |
| `packages/lib/constants.ts:280-281` | `ENABLE_ASYNC_TASKER` 定义 |
| `packages/features/webhooks/lib/tasker/WebhookTriggerTasker.ts` | Webhook Trigger.dev 触发器 |
| `packages/features/bookings/lib/tasker/BookingEmailAndSmsTriggerTasker.ts` | 邮件 Trigger.dev 触发器 |

### 历史残留相关

| 文件路径 | 说明 |
|----------|------|
| `packages/prisma/migrations/20260319000000_drop_workflow_tables/migration.sql` | 工作流表删除迁移 |
| `packages/emails/workflow-email-service.ts` | 命名残留的邮件服务 |
| `packages/trpc/server/routers/viewer/bookings/confirm.handler.test.ts` | 包含 workflows 测试数据 |

---

*文档版本：Round 3*
*复核日期：2026-05-04*