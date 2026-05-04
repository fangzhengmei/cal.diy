# 预约创建后通知触发路径分析（Round 2）

## 核心修正声明

**第一轮分析中关于"工作流系统已完全删除"的结论过于绝对。**

实际情况需要更精细的区分：
- ✅ **数据库层面**：原 `Workflow`、`WorkflowStep` 等表已删除
- ⚠️ **代码层面**：存在三种不同状态的 "workflow" 相关代码
- ❌ **命名层面**：大量 "workflow" 只是命名残留，与工作流引擎无关

---

## 一、"Workflow" 相关代码的三重区分

### 1.1 运行时能力：Trigger.dev 异步任务系统

**这是实际可运行的代码，是当前系统的"工作流"实现方式。**

#### 配置文件

```typescript
// packages/features/trigger.config.ts
export default defineConfig({
  project: process.env.TRIGGER_DEV_PROJECT_REF ?? "",
  
  // 任务目录配置
  dirs: [
    "./bookings/lib/tasker/trigger/notifications",    // 预订通知任务
    "./calendars/lib/tasker/trigger",                 // 日历任务
    "./ee/billing/service/proration/tasker/trigger",  // 账单任务
    "./ee/organizations/lib/billing/tasker/trigger",  // 组织账单任务
    "./webhooks/lib/tasker/trigger",                   // Webhook 投递任务
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

#### Trigger.dev 任务定义

| 任务类型 | 文件路径 | 任务 ID | 说明 |
|----------|----------|---------|------|
| **预订确认通知** | `bookings/lib/tasker/trigger/notifications/confirm.ts` | `booking.send.confirm.notifications` | 预订确认后发送邮件/SMS |
| **预订重新安排通知** | `bookings/lib/tasker/trigger/notifications/reschedule.ts` | `booking.send.reschedule.notifications` | 预订重新安排后发送通知 |
| **轮询预订重新安排** | `bookings/lib/tasker/trigger/notifications/rr-reschedule.ts` | `booking.send.rr-reschedule.notifications` | 轮询事件重新安排通知 |
| **预订请求通知** | `bookings/lib/tasker/trigger/notifications/request.ts` | `booking.send.request.notifications` | 预订请求（需确认）通知 |
| **Webhook 投递** | `webhooks/lib/tasker/trigger/deliver-webhook.ts` | `webhook.deliver` | Webhook 异步投递 |
| **日历任务** | `calendars/lib/tasker/trigger/ensure-default-calendars.ts` | `calendars.ensure-default-calendars` | 确保默认日历存在 |

#### 任务执行示例

```typescript
// packages/features/bookings/lib/tasker/trigger/notifications/confirm.ts
export const confirm = schemaTask({
  id: "booking.send.confirm.notifications",
  ...bookingNotificationsTaskConfig,
  schema: bookingNotificationTaskSchema,
  run: async (payload) => {
    const { TriggerDevLogger } = await import("@calcom/lib/triggerDevLogger");
    const { BookingEmailSmsHandler } = await import("@calcom/features/bookings/lib/BookingEmailSmsHandler");
    const { BookingRepository } = await import("@calcom/features/bookings/repositories/BookingRepository");
    const { prisma } = await import("@calcom/prisma");
    const { BookingEmailAndSmsTaskService } = await import("../../BookingEmailAndSmsTaskService");

    const triggerDevLogger = new TriggerDevLogger();
    const emailsAndSmsHandler = new BookingEmailSmsHandler({ logger: triggerDevLogger });
    const bookingRepo = new BookingRepository(prisma);
    const bookingTaskService = new BookingEmailAndSmsTaskService({
      logger: triggerDevLogger,
      bookingRepository: bookingRepo,
      emailsAndSmsHandler: emailsAndSmsHandler,
    });
    await bookingTaskService.confirm(payload);
  },
});
```

#### 当前状态：部分禁用

```typescript
// packages/features/bookings/lib/service/RegularBookingService.ts:2215
const isBookingEmailSmsTaskerEnabled = false;  // 硬编码为 false

// 只有同时满足以下条件才使用异步任务
if (ENABLE_ASYNC_TASKER && !noEmail && isBookingEmailSmsTaskerEnabled) {
  try {
    await deps.bookingEmailAndSmsTasker.send({
      action: bookingEmailsAndSmsTaskerAction,
      schedulingType: evtWithMetadata.eventType.schedulingType,
      payload: {
        bookingId: booking.id,
        conferenceCredentialId,
        platformClientId,
        // ...
      },
    });
  } catch (err) {
    tracingLogger.error("bookingEmailAndSmsTasker error:", err);
  }
}
```

**关键发现**：
1. Trigger.dev 任务系统**代码完整存在**
2. 邮件通知的异步路径**当前被硬编码禁用**（`isBookingEmailSmsTaskerEnabled = false`）
3. Webhook 投递任务**可能仍在使用**（需要进一步验证）

---

### 1.2 历史遗留：已废弃但未完全删除的代码

#### 测试文件中的 Workflows 数据

```typescript
// packages/trpc/server/routers/viewer/bookings/confirm.handler.test.ts:51-59
// 测试数据中仍然包含 workflows 字段
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

```typescript
// 第 113 行：工作流触发测试被注释
// expectWorkflowToBeTriggered({ emailsToReceive: [organizer.email], emails });
```

**分析**：
- 测试数据结构保留了 workflows 字段（为了兼容旧测试）
- 实际触发测试代码已被注释
- 这是**历史遗留**，不是运行时能力

#### 数据库迁移历史

```sql
-- packages/prisma/migrations/20260319000000_drop_workflow_tables/migration.sql
-- 2026年3月删除的表
DROP TABLE IF EXISTS "WorkflowStepTranslation" CASCADE;
DROP TABLE IF EXISTS "WorkflowReminder" CASCADE;
DROP TABLE IF EXISTS "WorkflowsOnEventTypes" CASCADE;
DROP TABLE IF EXISTS "WorkflowsOnTeams" CASCADE;
DROP TABLE IF EXISTS "WorkflowOptOutContact" CASCADE;
DROP TABLE IF EXISTS "AIPhoneCallConfiguration" CASCADE;
DROP TABLE IF EXISTS "WorkflowStep" CASCADE;
DROP TABLE IF EXISTS "Workflow" CASCADE;

-- 删除的枚举
DROP TYPE IF EXISTS "WorkflowTriggerEvents";
DROP TYPE IF EXISTS "WorkflowActions";
DROP TYPE IF EXISTS "WorkflowType";
DROP TYPE IF EXISTS "WorkflowTemplates";
DROP TYPE IF EXISTS "WorkflowMethods";
DROP TYPE IF EXISTS "WorkflowStepAutoTranslatedField";
DROP TYPE IF EXISTS "WorkflowContactType";
```

**分析**：
- 原工作流引擎的数据库层**已完全删除**
- 这是明确的**功能删除**，不是临时禁用

---

### 1.3 命名残留：用了 "workflow" 名称，但不是工作流引擎

#### 案例 1：workflow-email-service.ts

```typescript
// packages/emails/workflow-email-service.ts
// 文件名包含 "workflow"，但实际只是邮件发送辅助函数

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

**分析**：
- 文件名包含 "workflow" 是**历史残留**
- 实际功能：发送反馈邮件、月度摘要、预订重定向通知
- 与工作流引擎**完全无关**，只是一个命名不当的工具类

#### 案例 2：Zapier 集成描述

```typescript
// packages/app-store/zapier/_metadata.ts
description:
  "Workflow automation for everyone. Use the Cal.diy Zapier app to trigger your workflows when a booking is created, rescheduled, or cancelled, or after a meeting ends.",
```

**分析**：
- 这里的 "workflows" 指的是 **Zapier 的工作流**（用户在 Zapier 平台创建的自动化）
- 不是 Cal.diy 内部的工作流引擎
- 通过 Webhook 机制与 Zapier 集成

#### 案例 3：类型定义中的残留

```typescript
// packages/platform/atoms/event-types/wrappers/types.ts
export type PlatformTabs = keyof Omit<TabMap, "workflows" | "webhooks" | "instant" | "ai" | "apps">;
```

**分析**：
- "workflows" 被从 `PlatformTabs` 中排除
- 这是**已删除功能的残留类型定义**
- 表明平台层不再支持 workflows 标签页

#### 案例 4：翻译文件中的残留

```typescript
// packages/lib/translationConstants.ts
/**
 * Supported locales for auto-translation features.
 * Used by both event type and workflow step translations.
 */
export const TRANSLATION_SUPPORTED_LOCALES = [
  // ...
];
```

**分析**：
- 注释提到 "workflow step translations"
- 但原 `WorkflowStepTranslation` 表已被删除
- 这是**注释残留**，不影响运行时

---

## 二、Notification（通知）vs Reminder（提醒）的明确边界

### 2.1 核心定义

| 维度 | Notification（通知） | Reminder（提醒） |
|------|----------------------|------------------|
| **触发时机** | 事件发生时**立即**触发 | **预定的未来时间点**触发 |
| **时间关系** | 与事件**同时**发生 | 在事件**之前或之后**的特定时间 |
| **目的** | 告知用户某个事件**已经发生** | 提醒用户**即将发生**的事件或**需要采取**的行动 |
| **持久化** | 无（发送后即完成） | 有（需要记录已发送状态） |
| **取消机制** | 无（已发送不可取消） | 有（可删除预定任务） |
| **重复发送** | 一次 | 可多次（不同时间间隔） |

### 2.2 Notification（通知）的实现

#### 触发场景

1. **预订确认** (`BOOKING_CONFIRMED`)
2. **预订重新安排** (`BOOKING_RESCHEDULED`)
3. **预订请求** (`BOOKING_REQUESTED` - 需确认)
4. **预订创建** (`BOOKING_CREATED` - Webhook)
5. **预订取消** (`BOOKING_CANCELLED` - Webhook)

#### 代码实现

```typescript
// 同步邮件通知（当前使用）
// packages/features/bookings/lib/service/RegularBookingService.ts:2151-2168
if (!noEmail) {
  if (!isDryRun && !(eventType.seatsPerTimeSlot && rescheduleUid)) {
    await emailsAndSmsHandler.send({
      action: BookingActionMap.confirmed,
      data: {
        eventType: {
          metadata: eventType.metadata,
          schedulingType: eventType.schedulingType,
        },
        eventNameObject,
        evt,
        additionalInformation,
        additionalNotes,
        customInputs,
      },
    });
    bookingEmailsAndSmsTaskerAction = BookingActionMap.confirmed;
  }
}
```

```typescript
// 直接 Webhook 触发
// packages/features/bookings/lib/service/RegularBookingService.ts:2432-2439
if (isConfirmedByDefault) {
  // ...
  
  // Send Webhook call if hooked to BOOKING_CREATED & BOOKING_RESCHEDULED
  await handleWebhookTrigger({
    subscriberOptions,
    eventTrigger,  // BOOKING_CREATED 或 BOOKING_RESCHEDULED
    webhookData,
    isDryRun,
    traceContext,
  });
}
```

```typescript
// 异步 Webhook 排队（用于 BOOKING_REQUESTED）
// packages/features/bookings/lib/service/RegularBookingService.ts:2466-2484
if (booking && booking.status === BookingStatus.PENDING && !isDryRun) {
  try {
    await deps.webhookProducer.queueBookingRequestedWebhook({
      bookingUid: booking.uid,
      userId: subscriberOptions.userId ?? undefined,
      eventTypeId: subscriberOptions.eventTypeId ?? undefined,
      teamId: Array.isArray(subscriberOptions.teamId)
        ? subscriberOptions.teamId[0]
        : (subscriberOptions.teamId ?? undefined),
      orgId: subscriberOptions.orgId ?? undefined,
      oAuthClientId: platformClientId ?? undefined,
    });
  } catch (webhookError) {
    // 错误处理
  }
}
```

#### Notification 处理流程

```
事件发生（预订创建/确认/重新安排）
         ↓
┌─────────────────────────────────────────────────────┐
│              Notification 触发分支                    │
├─────────────────────────────────────────────────────┤
│  分支 1: 邮件/SMS 通知                               │
│     ├─ 同步: emailsAndSmsHandler.send()            │
│     └─ 异步（禁用）: bookingEmailAndSmsTasker.send()│
├─────────────────────────────────────────────────────┤
│  分支 2: Webhook 通知                               │
│     ├─ 直接触发: handleWebhookTrigger()             │
│     └─ 异步排队: webhookProducer.queue*Webhook()   │
└─────────────────────────────────────────────────────┘
         ↓
    立即执行/排队
         ↓
    通知发送完成（无持久化记录）
```

---

### 2.3 Reminder（提醒）的实现

#### 触发场景

| 提醒类型 | 触发时机 | 触发条件 |
|----------|----------|----------|
| **待确认预订提醒** | 预订创建后 48h/24h/3h | 预订状态仍为 `PENDING` |
| **会议开始 Webhook** | 预订开始时间 | 订阅了 `MEETING_STARTED` |
| **会议结束 Webhook** | 预订结束时间 | 订阅了 `MEETING_ENDED` |
| **No-Show 检查** | 会议开始后一段时间 | 订阅了 `AFTER_*_CAL_VIDEO_NO_SHOW` |

#### 代码实现

##### 类型 1：Cron 触发的待确认提醒

```typescript
// apps/web/app/api/cron/bookingReminder/route.ts
async function postHandler(request: NextRequest) {
  // 认证检查
  const apiKey = request.headers.get("authorization") || request.nextUrl.searchParams.get("apiKey");
  if (process.env.CRON_API_KEY !== apiKey) {
    return NextResponse.json({ message: "Not authenticated" }, { status: 401 });
  }

  const reminderIntervalMinutes = [48 * 60, 24 * 60, 3 * 60];  // 48小时、24小时、3小时
  let notificationsSent = 0;

  for (const interval of reminderIntervalMinutes) {
    // 查询符合条件的预订
    const bookings = await prisma.booking.findMany({
      where: {
        status: BookingStatus.PENDING,
        createdAt: {
          lte: dayjs().add(-interval, "minutes").toDate(),
        },
        endTime: { gte: new Date() },  // 预订未结束
        OR: [
          { payment: { none: {} } },      // 无支付需求
          { payment: { some: {} }, paid: true },  // 已支付
        ],
      },
      // ... select 字段
    });

    // 查询已发送的提醒
    const reminders = await prisma.reminderMail.findMany({
      where: {
        reminderType: ReminderType.PENDING_BOOKING_CONFIRMATION,
        referenceId: { in: bookingsToRemind.map((b) => b.id) },
        elapsedMinutes: { gte: interval },
      },
    });

    // 发送提醒（跳过已发送的）
    for (const booking of bookingsToRemind.filter((b) => !reminders.some((r) => r.referenceId == b.id))) {
      // ... 构建 CalendarEvent ...
      
      await sendOrganizerRequestReminderEmail(evt, booking?.eventType?.metadata as EventTypeMetadata);

      // 记录已发送
      await prisma.reminderMail.create({
        data: {
          referenceId: booking.id,
          reminderType: ReminderType.PENDING_BOOKING_CONFIRMATION,
          elapsedMinutes: interval,
        },
      });
      notificationsSent++;
    }
  }

  return NextResponse.json({ notificationsSent });
}
```

**数据模型**：
```prisma
// packages/prisma/schema.prisma:1080-1093
enum ReminderType {
  PENDING_BOOKING_CONFIRMATION
}

model ReminderMail {
  id             Int          @id @default(autoincrement())
  referenceId    Int
  reminderType   ReminderType
  elapsedMinutes Int
  createdAt      DateTime     @default(now())

  @@index([referenceId])
  @@index([reminderType])
}
```

##### 类型 2：预定 Webhook 触发

```typescript
// packages/features/webhooks/lib/scheduleTrigger.ts:282-320
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

    // 写入 webhookScheduledTriggers 表
    await prisma.webhookScheduledTriggers.create({
      data: {
        payload,
        appId: subscriber.appId,
        // 根据事件类型确定触发时间
        startAfter: triggerEvent === WebhookTriggerEvents.MEETING_ENDED 
          ? booking.endTime 
          : booking.startTime,
        subscriberUrl,
        webhook: {
          connect: { id: subscriber.id },
        },
        booking: {
          connect: { id: booking.id },
        },
      },
    });
  } catch (error) {
    console.error("Error cancelling scheduled jobs", error);
  }
}
```

**数据模型**：`webhookScheduledTriggers` 表存储预定的 Webhook 触发。

##### 类型 3：预定 No-Show 任务

```typescript
// packages/features/bookings/lib/handleNewBooking/scheduleNoShowTriggers.ts
const _scheduleNoShowTriggers = async (args: ScheduleNoShowTriggersArgs) => {
  const { booking, triggerForUser, organizerUser, eventTypeId, teamId, orgId, oAuthClientId, isDryRun = false } = args;

  const isCalVideoLocation = booking.location === DailyLocationType || booking.location?.trim() === "";

  if (isDryRun || !isCalVideoLocation) return;

  const noShowPromises: Promise<any>[] = [];

  // 查询订阅了 AFTER_HOSTS_CAL_VIDEO_NO_SHOW 的 Webhook
  const subscribersHostsNoShowStarted = await getWebhooks({
    userId: triggerForUser ? organizerUser.id : null,
    eventTypeId,
    triggerEvent: WebhookTriggerEvents.AFTER_HOSTS_CAL_VIDEO_NO_SHOW,
    teamId,
    orgId,
    oAuthClientId,
  });

  noShowPromises.push(
    ...subscribersHostsNoShowStarted.map((webhook) => {
      if (booking?.startTime && webhook.time && webhook.timeUnit) {
        // 计算触发时间（会议开始后 + webhook.time）
        const scheduledAt = dayjs(booking.startTime)
          .add(webhook.time, webhook.timeUnit.toLowerCase() as dayjs.ManipulateType)
          .toDate();
        
        // 通过 tasker 创建预定任务
        return tasker.create(
          "triggerHostNoShowWebhook",
          {
            triggerEvent: WebhookTriggerEvents.AFTER_HOSTS_CAL_VIDEO_NO_SHOW,
            bookingId: booking.id,
            webhook: { ...webhook, time: webhook.time, timeUnit: webhook.timeUnit },
          },
          { scheduledAt, referenceUid: booking.uid }
        );
      }
      return Promise.resolve();
    })
  );

  // 类似处理 AFTER_GUESTS_CAL_VIDEO_NO_SHOW
  // ...

  await Promise.all(noShowPromises);
};
```

#### Reminder 处理流程

```
预订创建/确认
         ↓
┌─────────────────────────────────────────────────────────────┐
│                   Reminder 调度阶段                          │
├─────────────────────────────────────────────────────────────┤
│  类型 1: 待确认预订提醒                                      │
│     ├─ 无需预调度                                            │
│     └─ 依赖外部 Cron 定期调用 /api/cron/bookingReminder     │
├─────────────────────────────────────────────────────────────┤
│  类型 2: 会议开始/结束 Webhook                               │
│     └─ scheduleTrigger() → 写入 webhookScheduledTriggers 表 │
├─────────────────────────────────────────────────────────────┤
│  类型 3: No-Show 检查                                       │
│     └─ tasker.create() → 写入 task 表（scheduledAt 字段）   │
└─────────────────────────────────────────────────────────────┘
         ↓
         等待触发时间到达
         ↓
┌─────────────────────────────────────────────────────────────┐
│                   Reminder 触发阶段                          │
├─────────────────────────────────────────────────────────────┤
│  类型 1: Cron 触发                                          │
│     ├─ 检查 ReminderMail 表（避免重复发送）                  │
│     ├─ 发送提醒邮件                                          │
│     └─ 记录到 ReminderMail 表                                │
├─────────────────────────────────────────────────────────────┤
│  类型 2: 预定 Webhook                                        │
│     └─ 外部调度器检查 webhookScheduledTriggers.startAfter    │
├─────────────────────────────────────────────────────────────┤
│  类型 3: 预定任务                                            │
│     └─ 外部调度器检查 task.scheduledAt                        │
└─────────────────────────────────────────────────────────────┘
```

---

### 2.4 边界案例分析

#### 案例：预订请求 (BOOKING_REQUESTED)

```typescript
// 预订请求时，同时触发 Notification 和 调度 Reminder

// 1. Notification（立即）
// 发送请求确认邮件给组织者和参会者
await emailsAndSmsHandler.send({
  action: BookingActionMap.requested,
  data: { evt, attendees: attendeesList, eventType, additionalNotes },
});

// 2. 排队 Webhook Notification（异步）
await deps.webhookProducer.queueBookingRequestedWebhook({
  bookingUid: booking.uid,
  // ...
});

// 3. Reminder（预定未来触发）
// 不需要显式调度，依赖外部 Cron 调用 /api/cron/bookingReminder
// 该 API 会检查：
//   - 预订状态为 PENDING
//   - 创建时间超过 48h/24h/3h
//   - 未发送过该间隔的提醒
```

#### 案例：预订确认 (BOOKING_CONFIRMED)

```typescript
// 预订确认时：

// 1. Notification（立即）
// 发送确认邮件
await emailsAndSmsHandler.send({
  action: BookingActionMap.confirmed,
  data: { ... },
});

// 触发 Webhook
await handleWebhookTrigger({
  eventTrigger: WebhookTriggerEvents.BOOKING_CREATED, // 或 BOOKING_RESCHEDULED
  // ...
});

// 2. Reminder（预定未来触发）
// 调度会议开始/结束 Webhook
for (const subscriber of subscribersMeetingEnded) {
  scheduleTrigger({
    booking,
    subscriberUrl: subscriber.subscriberUrl,
    subscriber,
    triggerEvent: WebhookTriggerEvents.MEETING_ENDED,
    isDryRun,
  });
}

// 调度 No-Show 检查（如果是 Cal Video）
await scheduleNoShowTriggers({
  booking,
  triggerForUser: true,
  organizerUser,
  eventTypeId,
  // ...
});
```

---

## 三、预约创建后的完整通知触发路径（修订版）

### 3.1 入口点：RegularBookingService

```typescript
// packages/features/bookings/lib/service/RegularBookingService.ts
// handler 函数是预订创建的核心处理函数
```

### 3.2 完整流程图

```
用户提交预约请求
         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    预订创建核心流程                                        │
├─────────────────────────────────────────────────────────────────────────┤
│  1. 验证预订数据                                                          │
│     - validateBookingTimeIsNotOutOfBounds()                              │
│     - validateEventLength()                                               │
│                                                                           │
│  2. 检查用户可用性                                                        │
│     - ensureAvailableUsers()                                              │
│     - getLuckyUser() (轮询事件)                                          │
│                                                                           │
│  3. 创建预订记录                                                          │
│     - createBooking() → 写入 booking 表                                  │
│                                                                           │
│  4. 日历同步                                                              │
│     - EventManager.create() / reschedule()                               │
│     - 创建/更新日历事件                                                    │
│     - 生成视频会议链接（如适用）                                           │
└─────────────────────────────────────────────────────────────────────────┘
         ↓
         ┌─────────────────────────────────────────────────────────────┐
         │                    Notification（立即触发）                    │
         ├─────────────────────────────────────────────────────────────┤
         │  分支 A: 邮件/SMS 通知（同步）                                │
         │  ┌─────────────────────────────────────────────────────┐    │
         │  │ 根据预订状态选择处理方法：                             │    │
         │  │                                                      │    │
         │  │ 状态: ACCEPTED（已确认）                             │    │
         │  │   → BookingActionMap.confirmed                      │    │
         │  │   → _handleConfirmed()                               │    │
         │  │   → 发送确认邮件给组织者和参会者                      │    │
         │  │                                                      │    │
         │  │ 状态: RESCHEDULED（已重新安排）                      │    │
         │  │   → BookingActionMap.rescheduled                    │    │
         │  │   → _handleRescheduled() 或 _handleRoundRobinRescheduled() │
         │  │   → 发送重新安排通知                                 │    │
         │  │                                                      │    │
         │  │ 状态: PENDING（待确认）                              │    │
         │  │   → BookingActionMap.requested                      │    │
         │  │   → _handleRequested()                               │    │
         │  │   → 发送请求确认邮件                                 │    │
         │  └─────────────────────────────────────────────────────┘    │
         │                                                                 │
         │  分支 B: Webhook 通知                                          │
         │  ┌─────────────────────────────────────────────────────┐    │
         │  │ 方式 1: 直接触发（用于 BOOKING_CREATED/RESCHEDULED） │    │
         │  │   → handleWebhookTrigger()                           │    │
         │  │   → 立即查询订阅者 → 构建 payload → 发送 HTTP 请求   │    │
         │  │                                                      │    │
         │  │ 方式 2: 异步排队（用于 BOOKING_REQUESTED）           │    │
         │  │   → webhookProducer.queueBookingRequestedWebhook()  │    │
         │  │   → 写入任务队列（Trigger.dev）→ 异步处理            │    │
         │  └─────────────────────────────────────────────────────┘    │
         │                                                                 │
         │  分支 C: 预订事件处理                                          │
         │  ┌─────────────────────────────────────────────────────┐    │
         │  │   → fireBookingEvents()                              │    │
         │  │   → bookingEventHandler.onBookingCreated() 或        │    │
         │  │     bookingEventHandler.onBookingRescheduled()       │    │
         │  │   → 平台层事件处理（如 hashedLink 更新等）            │    │
         │  └─────────────────────────────────────────────────────┘    │
         └─────────────────────────────────────────────────────────────┘
         ↓
         ┌─────────────────────────────────────────────────────────────┐
         │                    Reminder（预定未来触发）                    │
         ├─────────────────────────────────────────────────────────────┤
         │  类型 1: 待确认预订提醒                                      │
         │  ┌─────────────────────────────────────────────────────┐    │
         │  │  触发条件:                                            │    │
         │  │    - 预订状态为 PENDING                               │    │
         │  │    - 创建时间超过 48h/24h/3h                         │    │
         │  │    - 预订尚未结束                                    │    │
         │  │                                                      │    │
         │  │  触发方式:                                            │    │
         │  │    - 外部 Cron 定期调用 /api/cron/bookingReminder   │    │
         │  │    - 检查 ReminderMail 表避免重复发送                │    │
         │  │    - 发送后记录到 ReminderMail 表                    │    │
         │  └─────────────────────────────────────────────────────┘    │
         │                                                                 │
         │  类型 2: 会议开始/结束 Webhook                                 │
         │  ┌─────────────────────────────────────────────────────┐    │
         │  │  调度函数: scheduleTrigger()                         │    │
         │  │                                                      │    │
         │  │  触发时间:                                            │    │
         │  │    - MEETING_STARTED → booking.startTime            │    │
         │  │    - MEETING_ENDED → booking.endTime                │    │
         │  │                                                      │    │
         │  │  存储位置: webhookScheduledTriggers 表               │    │
         │  │    - payload: 事件数据                               │    │
         │  │    - startAfter: 触发时间                            │    │
         │  │    - subscriberUrl: 回调地址                         │    │
         │  └─────────────────────────────────────────────────────┘    │
         │                                                                 │
         │  类型 3: No-Show 检查（仅 Cal Video）                         │
         │  ┌─────────────────────────────────────────────────────┐    │
         │  │  调度函数: scheduleNoShowTriggers()                  │    │
         │  │                                                      │    │
         │  │  触发条件:                                            │    │
         │  │    - 会议地点是 Cal Video (Daily.co)                 │    │
         │  │    - 用户订阅了相应的 Webhook 事件                    │    │
         │  │                                                      │    │
         │  │  触发时间:                                            │    │
         │  │    - meeting.startTime + webhook.time (小时/分钟)   │    │
         │  │                                                      │    │
         │  │  存储位置: task 表                                   │    │
         │  │    - type: "triggerHostNoShowWebhook" 或            │    │
         │  │            "triggerGuestNoShowWebhook"              │    │
         │  │    - scheduledAt: 预定触发时间                       │    │
         │  │    - referenceUid: 预订 UID（用于取消）              │    │
         │  └─────────────────────────────────────────────────────┘    │
         └─────────────────────────────────────────────────────────────┘
         ↓
    预订创建流程结束
         ↓
    等待 Reminder 触发时间到达
```

---

## 四、关键代码位置索引

### 4.1 Trigger.dev 任务系统

| 文件路径 | 说明 |
|----------|------|
| `packages/features/trigger.config.ts` | Trigger.dev 主配置 |
| `packages/features/bookings/lib/tasker/trigger/notifications/confirm.ts` | 预订确认通知任务 |
| `packages/features/bookings/lib/tasker/trigger/notifications/reschedule.ts` | 预订重新安排通知任务 |
| `packages/features/bookings/lib/tasker/trigger/notifications/request.ts` | 预订请求通知任务 |
| `packages/features/webhooks/lib/tasker/trigger/deliver-webhook.ts` | Webhook 投递任务 |
| `packages/features/bookings/lib/tasker/BookingEmailAndSmsTriggerTasker.ts` | 邮件通知任务触发器 |

### 4.2 Notification（通知）

| 文件路径 | 说明 |
|----------|------|
| `packages/features/bookings/lib/service/RegularBookingService.ts` | 预订服务主入口 |
| `packages/features/bookings/lib/BookingEmailSmsHandler.ts` | 邮件/SMS 通知处理 |
| `packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts` | Webhook 任务生产者 |
| `packages/features/webhooks/lib/handleWebhookTrigger.ts` | Webhook 直接触发处理 |

### 4.3 Reminder（提醒）

| 文件路径 | 说明 |
|----------|------|
| `apps/web/app/api/cron/bookingReminder/route.ts` | 待确认预订提醒 API |
| `packages/features/webhooks/lib/scheduleTrigger.ts` | 预定 Webhook 调度 |
| `packages/features/bookings/lib/handleNewBooking/scheduleNoShowTriggers.ts` | No-Show 检查调度 |
| `packages/prisma/schema.prisma:1080-1093` | ReminderType 枚举和 ReminderMail 模型 |

### 4.4 命名残留文件

| 文件路径 | 实际功能 | 命名问题 |
|----------|----------|----------|
| `packages/emails/workflow-email-service.ts` | 发送反馈邮件、月度摘要、预订重定向通知 | 文件名包含 "workflow"，但与工作流引擎无关 |
| `packages/app-store/zapier/_metadata.ts` | 描述 Zapier 集成 | "workflows" 指 Zapier 的工作流，不是 Cal.diy 内部功能 |

---

## 五、修订总结

### 5.1 关于"工作流"的准确结论

| 层面 | 状态 | 说明 |
|------|------|------|
| **数据库层** | ✅ 已删除 | `Workflow`、`WorkflowStep` 等表已在 2026 年 3 月迁移中删除 |
| **原工作流引擎** | ✅ 已删除 | 用户可配置的工作流引擎（自定义触发条件、步骤）已不存在 |
| **Trigger.dev 任务系统** | ⚠️ 部分可用 | 异步任务调度框架存在，但邮件通知路径被硬编码禁用 |
| **命名残留** | ⚠️ 存在 | 部分文件名、注释、类型定义仍包含 "workflow"，但与工作流引擎无关 |
| **测试数据残留** | ⚠️ 存在 | 测试文件中仍有 workflows 字段，但触发测试已被注释 |

### 5.2 关于 Notification vs Reminder 的准确结论

**Notification（通知）**：
- **时间性**：事件发生时立即触发
- **目的**：告知用户某个事件已经发生
- **持久化**：无（发送后即完成，不记录发送状态）
- **触发方式**：同步调用 或 异步排队

**Reminder（提醒）**：
- **时间性**：预定的未来时间点触发
- **目的**：提醒用户即将发生的事件或需要采取的行动
- **持久化**：有（需要记录已发送状态，避免重复发送）
- **触发方式**：预调度 → 等待时间到达 → 外部调度器触发

### 5.3 关键修正

1. **原结论**："工作流系统已完全删除"
   - **修正**：原用户可配置的工作流引擎已删除，但 Trigger.dev 异步任务系统仍存在（作为运行时能力），同时存在命名残留和测试数据残留

2. **原结论**：未明确区分 Notification 和 Reminder
   - **修正**：明确了两者在触发时机、目的、持久化、触发方式等维度的边界

3. **原结论**：未区分不同状态的 "workflow" 代码
   - **修正**：明确区分了运行时能力、历史遗留、命名残留三种状态

---

## 六、扩展思考

### 6.1 Trigger.dev 任务系统的定位

Trigger.dev 任务系统是一个**通用的异步任务调度框架**，不是传统意义上的"工作流引擎"：

| 特性 | 工作流引擎（原） | Trigger.dev 任务（现） |
|------|-----------------|-----------------------|
| 用户可配置 | ✅ 是 | ❌ 否 |
| 可视化编辑器 | ✅ 是 | ❌ 否 |
| 自定义触发条件 | ✅ 是 | ❌ 否（硬编码在代码中） |
| 多步骤编排 | ✅ 是 | ⚠️ 可实现，但需编码 |
| 异步执行 | ✅ 是 | ✅ 是 |
| 重试机制 | ✅ 是 | ✅ 是 |

**结论**：Trigger.dev 提供了**技术层面的异步任务能力**，但不再提供**业务层面的用户可配置工作流**。

### 6.2 系统架构的演变

```
旧架构（已删除）：
┌─────────────────────────────────────────────────────┐
│              用户可配置工作流引擎                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ Workflow    │  │ WorkflowStep│  │WorkflowRem- │ │
│  │ (定义)      │  │ (步骤)      │  │ inder(调度) │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│           ↓                ↓                ↓         │
│  ┌─────────────────────────────────────────────┐    │
│  │           工作流执行引擎（运行时）             │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘

新架构（当前）：
┌─────────────────────────────────────────────────────┐
│              Trigger.dev 异步任务系统                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ 预订通知任务 │  │ Webhook投递 │  │  日历任务   │ │
│  │ (硬编码)    │  │ (硬编码)    │  │ (硬编码)    │ │
│  └─────────────┘  └─────────────┘  └─────────────┘ │
│           ↓                ↓                ↓         │
│  ┌─────────────────────────────────────────────┐    │
│  │         Trigger.dev 平台（调度和执行）        │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### 6.3 未来可能的扩展点

1. **启用 Trigger.dev 邮件任务**：将 `isBookingEmailSmsTaskerEnabled` 改为 `true`，实现邮件通知的异步化
2. **基于 Webhook 的外部工作流**：通过 Webhook 与 Zapier、n8n 等外部自动化平台集成，实现用户可配置的工作流
3. **重新引入简化版工作流**：在新的架构基础上，重新设计用户可配置的工作流功能

---

*文档版本：Round 2*
*分析日期：2026-05-04*