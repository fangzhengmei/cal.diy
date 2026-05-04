# 预约创建后通知触发路径分析

## 概述

本文档分析 Cal.diy 系统中预约创建后触发通知的完整路径，以及提醒和工作流的串联机制。

## 重要发现：工作流（Workflow）系统已被删除

根据数据库迁移文件 `packages/prisma/migrations/20260319000000_drop_workflow_tables/migration.sql`，工作流相关的表和功能已经被完全删除：

### 已删除的工作流相关表

```sql
-- 1. Drop junction/child tables first
DROP TABLE IF EXISTS "WorkflowStepTranslation" CASCADE;
DROP TABLE IF EXISTS "WorkflowReminder" CASCADE;
DROP TABLE IF EXISTS "WorkflowsOnEventTypes" CASCADE;
DROP TABLE IF EXISTS "WorkflowsOnTeams" CASCADE;
DROP TABLE IF EXISTS "WorkflowOptOutContact" CASCADE;
DROP TABLE IF EXISTS "AIPhoneCallConfiguration" CASCADE;

-- 2. Drop WorkflowStep
DROP TABLE IF EXISTS "WorkflowStep" CASCADE;

-- 3. Drop Workflow
DROP TABLE IF EXISTS "Workflow" CASCADE;

-- 4. Drop enum types
DROP TYPE IF EXISTS "WorkflowTriggerEvents";
DROP TYPE IF EXISTS "WorkflowActions";
DROP TYPE IF EXISTS "WorkflowType";
DROP TYPE IF EXISTS "WorkflowTemplates";
DROP TYPE IF EXISTS "WorkflowMethods";
DROP TYPE IF EXISTS "WorkflowStepAutoTranslatedField";
DROP TYPE IF EXISTS "WorkflowContactType";
```

### 结论

**系统中不再存在工作流（Workflow）功能**。所有与工作流相关的代码和数据库表都已被移除。当前系统使用以下三种独立的通知机制：

1. **邮件/SMS 通知** - 直接发送邮件和短信
2. **Webhook 通知** - 向外部系统发送事件通知
3. **定时提醒** - 通过 Cron 任务触发的预订确认提醒

---

## 当前通知系统架构

### 1. 邮件/SMS 通知系统

#### 核心组件

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| `BookingEmailSmsHandler` | `packages/features/bookings/lib/BookingEmailSmsHandler.ts` | 处理预订创建后发送通知的核心逻辑 |
| `BookingEmailAndSmsTriggerTasker` | `packages/features/bookings/lib/tasker/BookingEmailAndSmsTriggerTasker.ts` | 预订通知的触发器（基于 Trigger.dev） |
| `email-manager` | `packages/emails/email-manager.ts` | 实际的邮件发送逻辑 |

#### 通知类型

`BookingEmailSmsHandler` 支持三种主要的通知操作：

```typescript
export const BookingActionMap = {
  confirmed: "BOOKING_CONFIRMED",
  rescheduled: "BOOKING_RESCHEDULED",
  requested: "BOOKING_REQUESTED",
} as const;
```

#### 处理方法

| 方法 | 触发场景 | 说明 |
|------|----------|------|
| `_handleConfirmed()` | 预订已确认 | 发送确认邮件给组织者和参会者 |
| `_handleRescheduled()` | 预订已重新安排 | 发送重新安排通知 |
| `_handleRoundRobinRescheduled()` | 轮询预订重新安排 | 处理轮询事件的重新安排 |
| `_handleRequested()` | 预订请求（需确认） | 发送请求确认邮件 |
| `handleAddAttendee()` | 添加单个参会者 | 发送邀请邮件 |
| `handleAddGuests()` | 添加多个访客 | 发送邀请邮件 |

### 2. Webhook 通知系统

#### 核心组件

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| `WebhookTaskerProducerService` | `packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts` | Webhook 任务生产者，轻量级队列服务 |
| `WebhookNotificationHandler` | `packages/features/webhooks/lib/service/WebhookNotificationHandler.ts` | Webhook 通知处理逻辑 |
| `WebhookFeature` | `packages/features/webhooks/lib/facade/WebhookFeature.ts` | Webhook 功能门面，统一 API |

#### 支持的 Webhook 事件类型

```typescript
// packages/features/webhooks/lib/constants.ts
export const WEBHOOK_TRIGGER_EVENTS_GROUPED_BY_APP = {
  core: [
    WebhookTriggerEvents.BOOKING_CANCELLED,
    WebhookTriggerEvents.BOOKING_CREATED,
    WebhookTriggerEvents.BOOKING_RESCHEDULED,
    WebhookTriggerEvents.BOOKING_PAID,
    WebhookTriggerEvents.BOOKING_PAYMENT_INITIATED,
    WebhookTriggerEvents.MEETING_ENDED,
    WebhookTriggerEvents.MEETING_STARTED,
    WebhookTriggerEvents.BOOKING_REQUESTED,
    WebhookTriggerEvents.BOOKING_REJECTED,
    WebhookTriggerEvents.RECORDING_READY,
    WebhookTriggerEvents.RECORDING_TRANSCRIPTION_GENERATED,
    WebhookTriggerEvents.BOOKING_NO_SHOW_UPDATED,
    WebhookTriggerEvents.OOO_CREATED,
    WebhookTriggerEvents.AFTER_HOSTS_CAL_VIDEO_NO_SHOW,
    WebhookTriggerEvents.AFTER_GUESTS_CAL_VIDEO_NO_SHOW,
    WebhookTriggerEvents.DELEGATION_CREDENTIAL_ERROR,
    WebhookTriggerEvents.WRONG_ASSIGNMENT_REPORT,
  ] as const,
};
```

#### Webhook 触发方式

Webhook 有两种触发方式：

1. **直接触发** - 通过 `handleWebhookTrigger()` 函数
2. **异步队列** - 通过 `WebhookTaskerProducerService` 排队处理

### 3. 定时提醒系统

#### 核心组件

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| `bookingReminder` API | `apps/web/app/api/cron/bookingReminder/route.ts` | 预订提醒的核心实现（Cron 任务） |
| `ReminderMail` 模型 | `packages/prisma/schema.prisma:1084` | 数据库中的提醒邮件记录 |

#### 提醒类型

当前系统只有一种提醒类型：

```prisma
enum ReminderType {
  PENDING_BOOKING_CONFIRMATION
}
```

#### 提醒触发逻辑

提醒通过 Cron 任务定期触发，检查以下时间间隔：

```typescript
// apps/web/app/api/cron/bookingReminder/route.ts:23
const reminderIntervalMinutes = [48 * 60, 24 * 60, 3 * 60]; // 48小时、24小时、3小时
```

提醒触发条件：
1. 预订状态为 `PENDING`（待确认）
2. 预订创建时间超过指定间隔
3. 预订尚未结束
4. 该间隔的提醒尚未发送过

---

## 预约创建后的完整通知触发路径

### 主入口：RegularBookingService

预约创建的核心逻辑位于 `packages/features/bookings/lib/service/RegularBookingService.ts`。

### 触发流程概览

```
用户提交预约请求
        ↓
RegularBookingService.createBooking()
        ↓
┌─────────────────────────────────────────────────────────────┐
│                      预约创建流程                              │
├─────────────────────────────────────────────────────────────┤
│  1. 验证预订数据 (validateBookingTime, validateEventLength)  │
│  2. 检查可用性 (ensureAvailableUsers)                         │
│  3. 创建预订记录 (createBooking)                              │
│  4. 日历同步 (EventManager.create/reschedule)                 │
│  5. 触发通知 (邮件 + Webhook + 事件)                          │
└─────────────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────────────────┐
│                      通知触发分支                              │
├─────────────────────────────────────────────────────────────┤
│  分支1: 邮件/SMS 通知  ← 直接触发                            │
│  分支2: Webhook 通知    ← 直接或异步触发                      │
│  分支3: 预订事件处理    ← bookingEventHandler                │
│  分支4: 定时提醒调度    ← 安排未来触发                        │
└─────────────────────────────────────────────────────────────┘
```

### 详细触发路径

#### 路径 1：邮件/SMS 通知触发

**触发位置**：`RegularBookingService.ts` 中的 `handler` 函数

**根据预订状态的不同处理**：

##### 情况 A：预订已重新安排 (Rescheduled)

```typescript
// RegularBookingService.ts:2030-2047
if (!eventType.seatsPerTimeSlot && originalRescheduledBooking?.uid) {
  // ... 日历同步逻辑 ...
  
  if (!noEmail && isConfirmedByDefault && !isDryRun) {
    await emailsAndSmsHandler.send({
      action: BookingActionMap.rescheduled,
      data: {
        evt,
        eventType,
        additionalInformation: metadata,
        additionalNotes,
        iCalUID,
        originalRescheduledBooking,
        rescheduleReason,
        isRescheduledByBooker: reqBody.rescheduledBy === bookerEmail,
        users,
        changedOrganizer,
      },
    });
    bookingEmailsAndSmsTaskerAction = BookingActionMap.rescheduled;
  }
}
```

##### 情况 B：预订已确认 (Confirmed)

```typescript
// RegularBookingService.ts:2050-2170
} else if (isConfirmedByDefault) {
  // ... 日历同步逻辑 ...
  
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
}
```

##### 情况 C：预订请求 (Requested - 需确认)

```typescript
// RegularBookingService.ts:2189-2203
if (!isConfirmedByDefault && noEmail !== true && !bookingRequiresPayment) {
  if (!isDryRun) {
    await emailsAndSmsHandler.send({
      action: BookingActionMap.requested,
      data: { evt, attendees: attendeesList, eventType, additionalNotes },
    });
    bookingEmailsAndSmsTaskerAction = BookingActionMap.requested;
  }
}
```

#### 路径 2：Webhook 通知触发

Webhook 通知有多个触发点：

##### 触发点 A：预订创建/重新安排

```typescript
// RegularBookingService.ts:2432-2439
if (isConfirmedByDefault) {
  // ... 其他逻辑 ...
  
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

##### 触发点 B：预订请求

```typescript
// RegularBookingService.ts:2466-2484
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

##### 触发点 C：支付相关

```typescript
// RegularBookingService.ts:2311-2328
const subscriberOptionsPaymentInitiated: GetSubscriberOptions = {
  userId: organizerUser.id,
  eventTypeId,
  triggerEvent: WebhookTriggerEvents.BOOKING_PAYMENT_INITIATED,
  teamId: null,
  orgId: null,
  oAuthClientId: platformClientId,
};
await handleWebhookTrigger({
  subscriberOptions: subscriberOptionsPaymentInitiated,
  eventTrigger: WebhookTriggerEvents.BOOKING_PAYMENT_INITIATED,
  webhookData: {
    ...webhookData,
    paymentId: payment?.id,
  },
  isDryRun,
  traceContext,
});
```

##### 触发点 D：会议开始/结束（定时）

```typescript
// RegularBookingService.ts:2365-2415
if (isConfirmedByDefault) {
  const subscribersMeetingEnded = await getWebhooks(subscriberOptionsMeetingEnded);
  const subscribersMeetingStarted = await getWebhooks(subscriberOptionsMeetingStarted);

  if (booking && booking.status === BookingStatus.ACCEPTED) {
    const bookingWithCalEventResponses = {
      ...booking,
      responses: reqBody.calEventResponses,
    };
    for (const subscriber of subscribersMeetingEnded) {
      scheduleTriggerPromises.push(
        scheduleTrigger({
          booking: bookingWithCalEventResponses,
          subscriberUrl: subscriber.subscriberUrl,
          subscriber,
          triggerEvent: WebhookTriggerEvents.MEETING_ENDED,
          isDryRun,
        })
      );
    }

    for (const subscriber of subscribersMeetingStarted) {
      scheduleTriggerPromises.push(
        scheduleTrigger({
          booking: bookingWithCalEventResponses,
          subscriberUrl: subscriber.subscriberUrl,
          subscriber,
          triggerEvent: WebhookTriggerEvents.MEETING_STARTED,
          isDryRun,
        })
      );
    }
  }
}
```

#### 路径 3：预订事件处理

```typescript
// RegularBookingService.ts:2217-2230
await this.fireBookingEvents({
  booking: {
    ...booking,
    userEmail: booking.user?.email ?? null,
  },
  organizerUser,
  hashedLink: hasHashedBookingLink ? (reqBody.hashedLink ?? null) : null,
  isDryRun,
  bookerEmail,
  bookerName: fullName,
  originalRescheduledBooking,
  isRecurringBooking: !!input.bookingData.allRecurringDates,
  tracingLogger,
});
```

`fireBookingEvents` 方法的实现：

```typescript
// RegularBookingService.ts:2617-2651
async fireBookingEvents({
  booking,
  organizerUser,
  hashedLink,
  isDryRun,
  bookerEmail,
  bookerName,
  originalRescheduledBooking,
  isRecurringBooking,
  tracingLogger,
}) {
  try {
    const bookingCreatedPayload = buildBookingCreatedPayload({
      booking,
      organizerUserId: organizerUser.id,
      organizerUserUuid: organizerUser.uuid,
      hashedLink,
      isDryRun,
    });

    const bookingEventHandler = this.deps.bookingEventHandler;

    if (!isRecurringBooking) {
      if (originalRescheduledBooking) {
        const bookingRescheduledPayload: BookingRescheduledPayload = {
          ...bookingCreatedPayload,
          oldBooking: {
            uid: originalRescheduledBooking.uid,
            startTime: originalRescheduledBooking.startTime,
            endTime: originalRescheduledBooking.endTime,
          },
        };
        await bookingEventHandler.onBookingRescheduled({
          payload: bookingRescheduledPayload,
        });
      } else {
        await bookingEventHandler.onBookingCreated({
          payload: bookingCreatedPayload,
        });
      }
    }
  } catch (error) {
    tracingLogger.error("Error while firing booking events", safeStringify(error));
  }
}
```

#### 路径 4：定时提醒调度

##### No-Show 触发器

```typescript
// RegularBookingService.ts:2495-2510
try {
  if (isConfirmedByDefault) {
    await scheduleNoShowTriggers({
      booking: {
        startTime: booking.startTime,
        id: booking.id,
        location: booking.location,
        uid: booking.uid,
      },
      triggerForUser: true,
      organizerUser: { id: organizerUser.id },
      eventTypeId,
      teamId: null,
      orgId: null,
      isDryRun,
    });
  }
} catch (error) {
  tracingLogger.error("Error while scheduling no show triggers", JSON.stringify({ error }));
}
```

##### 异步邮件任务（未启用）

```typescript
// RegularBookingService.ts:2515-2548
if (!isDryRun) {
  // ... analytics 事件 ...

  // Unused until we deploy to trigger.dev production
  // for now we only enable for cal.com org and we keep our current email system
  // cal.com org members will see emails in double while we test
  if (ENABLE_ASYNC_TASKER && !noEmail && isBookingEmailSmsTaskerEnabled) {
    try {
      await deps.bookingEmailAndSmsTasker.send({
        action: bookingEmailsAndSmsTaskerAction,
        schedulingType: evtWithMetadata.eventType.schedulingType,
        payload: {
          bookingId: booking.id,
          conferenceCredentialId,
          platformClientId,
          platformRescheduleUrl,
          platformCancelUrl,
          platformBookingUrl,
          isRescheduledByBooker: reqBody.rescheduledBy === bookerEmail,
        },
      });
    } catch (err) {
      tracingLogger.error("bookingEmailAndSmsTasker error:", err);
    }
  }
}
```

**注意**：`isBookingEmailSmsTaskerEnabled` 被硬编码为 `false`，所以这个异步邮件任务路径当前未启用。

---

## 提醒机制详解

### 当前提醒系统的限制

当前系统的提醒功能非常有限，仅支持：

1. **单一提醒类型**：`PENDING_BOOKING_CONFIRMATION`（待确认预订提醒）
2. **固定时间间隔**：48小时、24小时、3小时
3. **仅发送给组织者**：提醒组织者确认待定预订

### 提醒触发流程

```
Cron 调度器 (如 Vercel Cron, GitHub Actions, 或其他)
        ↓
POST /api/cron/bookingReminder (需带 CRON_API_KEY 认证)
        ↓
┌─────────────────────────────────────────────────────────────┐
│  1. 遍历提醒间隔 [48h, 24h, 3h]                              │
│  2. 查询符合条件的预订:                                         │
│     - status = PENDING                                        │
│     - createdAt <= now - interval                             │
│     - endTime >= now (预订未结束)                             │
│     - 无支付需求 或 已支付                                     │
│  3. 检查 ReminderMail 表是否已发送过该间隔的提醒               │
│  4. 发送提醒邮件 (sendOrganizerRequestReminderEmail)          │
│  5. 记录到 ReminderMail 表                                    │
└─────────────────────────────────────────────────────────────┘
```

### 核心代码

```typescript
// apps/web/app/api/cron/bookingReminder/route.ts
async function postHandler(request: NextRequest) {
  // 认证检查
  const apiKey = request.headers.get("authorization") || request.nextUrl.searchParams.get("apiKey");
  if (process.env.CRON_API_KEY !== apiKey) {
    return NextResponse.json({ message: "Not authenticated" }, { status: 401 });
  }

  const reminderIntervalMinutes = [48 * 60, 24 * 60, 3 * 60];
  let notificationsSent = 0;

  for (const interval of reminderIntervalMinutes) {
    // 查询符合条件的预订
    const bookings = await prisma.booking.findMany({
      where: {
        status: BookingStatus.PENDING,
        createdAt: {
          lte: dayjs().add(-interval, "minutes").toDate(),
        },
        endTime: { gte: new Date() },
        OR: [
          { payment: { none: {} } },
          { payment: { some: {} }, paid: true },
        ],
      },
      // ... select 字段
    });

    // 过滤掉平台管理且禁用邮件的预订
    const bookingsToRemind = bookings.filter(
      (booking) =>
        !booking.user ||
        !booking.user.isPlatformManaged ||
        (booking.user.isPlatformManaged && Boolean(booking.user.platformOAuthClients?.[0]?.areEmailsEnabled))
    );

    // 查询已发送的提醒
    const reminders = await prisma.reminderMail.findMany({
      where: {
        reminderType: ReminderType.PENDING_BOOKING_CONFIRMATION,
        referenceId: { in: bookingsToRemind.map((b) => b.id) },
        elapsedMinutes: { gte: interval },
      },
    });

    // 发送提醒
    for (const booking of bookingsToRemind.filter((b) => !reminders.some((r) => r.referenceId == b.id))) {
      // 构建 CalendarEvent ...
      
      await sendOrganizerRequestReminderEmail(evt, booking?.eventType?.metadata as EventTypeMetadata);

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

---

## 各通知系统的对比

| 特性 | 邮件/SMS 通知 | Webhook 通知 | 定时提醒 |
|------|--------------|--------------|----------|
| **触发时机** | 预订创建/更新时立即 | 预订创建/更新时立即或定时 | Cron 定期检查 |
| **通知对象** | 组织者、参会者 | 外部系统订阅者 | 仅组织者 |
| **通知内容** | 邮件模板、短信 | 标准化 JSON 数据 | 提醒确认邮件 |
| **可配置性** | 部分（通过 eventType.metadata） | 高（可订阅不同事件） | 低（固定间隔） |
| **异步处理** | 同步执行（异步任务已禁用） | 支持异步队列 | 异步（Cron） |
| **重试机制** | 无 | 有（Trigger.dev） | 无（通过 ReminderMail 去重） |

---

## 关键代码位置索引

### 核心服务

| 文件路径 | 说明 |
|----------|------|
| `packages/features/bookings/lib/service/RegularBookingService.ts` | 预订创建的核心服务，所有通知的主入口 |
| `packages/features/bookings/lib/BookingEmailSmsHandler.ts` | 邮件/SMS 通知处理 |
| `packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts` | Webhook 任务生产者 |
| `packages/features/webhooks/lib/service/WebhookNotificationHandler.ts` | Webhook 通知处理 |
| `apps/web/app/api/cron/bookingReminder/route.ts` | 预订提醒 Cron API |

### 数据库模型

| 文件路径 | 说明 |
|----------|------|
| `packages/prisma/schema.prisma:1080-1093` | ReminderType 枚举和 ReminderMail 模型 |
| `packages/prisma/migrations/20260319000000_drop_workflow_tables/migration.sql` | 工作流表删除迁移（关键历史记录） |

### 触发器

| 文件路径 | 说明 |
|----------|------|
| `packages/features/bookings/lib/tasker/BookingEmailAndSmsTriggerTasker.ts` | 邮件/SMS 触发器（Trigger.dev） |
| `packages/features/bookings/lib/tasker/trigger/notifications/confirm.ts` | 确认通知触发器 |
| `packages/features/bookings/lib/tasker/trigger/notifications/reschedule.ts` | 重新安排通知触发器 |
| `packages/features/bookings/lib/tasker/trigger/notifications/request.ts` | 请求通知触发器 |

---

## 流程图总结

### 预约创建后的完整通知流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        用户提交预约请求                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│              RegularBookingService.createBooking()                            │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  1. 验证预订数据                                                       │    │
│  │  2. 检查用户可用性                                                     │    │
│  │  3. 创建数据库预订记录 (createBooking)                                 │    │
│  │  4. 日历同步 (EventManager.create/reschedule)                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
                    ┌─────────────────┼─────────────────┐
                    ↓                 ↓                 ↓
        ┌───────────────────┐ ┌───────────────┐ ┌─────────────────┐
        │   邮件/SMS 通知    │ │  Webhook 通知  │ │  预订事件处理    │
        │  (直接同步执行)    │ │ (直接/异步)    │ │ (bookingEvents) │
        └───────────────────┘ └───────────────┘ └─────────────────┘
                    ↓                 ↓                 ↓
        ┌───────────────────┐ ┌───────────────┐
        │ BookingEmailSms-  │ │ handleWebhook │
        │ Handler.send()    │ │ Trigger()     │
        └───────────────────┘ └───────────────┘
                    ↓                 ↓
        ┌─────────────────────────────────────┐
        │         email-manager.ts             │
        │  sendScheduledEmailsAndSMS()        │
        │  sendRescheduledEmailsAndSMS()       │
        │  sendOrganizerRequestEmail()         │
        └─────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        未来触发的定时任务                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. scheduleNoShowTriggers()     → No-Show 检查                            │
│  2. scheduleTrigger(MEETING_*)  → 会议开始/结束 Webhook                     │
│  3. 外部 Cron 调用              → /api/cron/bookingReminder (待确认提醒)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 结论

### 主要发现

1. **工作流系统已删除**：`Workflow`、`WorkflowStep`、`WorkflowReminder` 等表已在 2026 年 3 月的迁移中被完全删除。系统不再支持工作流功能。

2. **三种独立的通知机制**：
   - **邮件/SMS 通知**：直接同步发送，用于预订确认、重新安排、请求等场景
   - **Webhook 通知**：可直接触发或异步排队，用于通知外部系统
   - **定时提醒**：通过 Cron 任务触发，仅用于提醒组织者确认待定预订

3. **异步任务系统未完全启用**：
   - `BookingEmailAndSmsTriggerTasker`（基于 Trigger.dev）已实现但 `isBookingEmailSmsTaskerEnabled` 硬编码为 `false`
   - 邮件通知当前是同步执行的

4. **提醒系统功能有限**：
   - 仅支持 `PENDING_BOOKING_CONFIRMATION` 一种提醒类型
   - 固定时间间隔：48小时、24小时、3小时
   - 仅发送给组织者，用于提醒确认待定预订

### 与原工作流系统的差异

在工作流系统被删除前，可能存在以下能力（根据迁移文件推断）：

| 已删除的能力 | 说明 |
|-------------|------|
| 自定义工作流 | 用户可创建自定义工作流，定义触发条件和动作 |
| 工作流步骤 | 工作流可包含多个步骤（如发送邮件、调用 API 等） |
| 工作流提醒 | `WorkflowReminder` 表用于存储工作流触发的提醒 |
| 事件类型关联 | `WorkflowsOnEventTypes` 表将工作流关联到事件类型 |
| 团队工作流 | `WorkflowsOnTeams` 表支持团队级别的工作流 |

当前系统没有这些能力，通知机制是硬编码在 `RegularBookingService` 中的固定流程。

### 建议

如果需要恢复或扩展工作流能力，需要：

1. 重新设计工作流数据模型
2. 实现工作流引擎（触发条件评估、步骤执行）
3. 集成到预订创建流程中
4. 提供 UI 供用户配置工作流

当前系统的通知机制虽然简单，但对于基本使用场景已经足够。如果需要更灵活的自动化能力，可能需要重新引入工作流系统或使用 Webhook 结合外部自动化工具（如 Zapier、Make.com 等）。