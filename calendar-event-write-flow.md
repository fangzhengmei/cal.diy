# Cal.diy 日历事件写入时序分析报告

## 1. 整体架构概览

日历事件写入是一个**分布式事务场景**，涉及数据库操作和外部API调用。系统采用**最终一致性**策略，而非强一致性事务。

### 1.1 两种预约确认模式

| 模式 | 触发条件 | 状态流转 | 日历写入时机 |
|------|---------|---------|-------------|
| **即时确认** | `isConfirmedByDefault = true` | PENDING → ACCEPTED (直接) | 预约创建后立即写入 |
| **需要确认** | `isConfirmedByDefault = false` | PENDING → (等待确认) → ACCEPTED | 组织者确认后写入 |

### 1.2 关键组件职责

| 组件 | 文件位置 | 核心职责 |
|------|---------|---------|
| **RegularBookingService** | `packages/features/bookings/lib/service/RegularBookingService.ts` | 预约创建主流程、协调各组件 |
| **EventManager** | `packages/features/bookings/lib/EventManager.ts` | 外部日历/视频/CRM集成调用 |
| **handleConfirmation** | `packages/features/bookings/lib/handleConfirmation.ts` | 待确认预约的确认流程 |
| **createBooking** | `packages/features/bookings/lib/handleNewBooking/createBooking.ts` | 数据库持久化（事务内） |
| **IdempotencyKeyService** | `packages/lib/idempotencyKey/idempotencyKeyService.ts` | 幂等键生成 |

---

## 2. 即时确认模式时序（isConfirmedByDefault = true）

### 2.1 完整时序图

```
┌──────────┐     ┌────────────────┐     ┌──────────────────┐     ┌──────────────┐     ┌──────────────┐
│  前端    │     │ API Layer      │     │ RegularBooking   │     │  createBooking │     │ EventManager │
│ (Client) │     │ (/api/book/    │     │ Service          │     │  (DB事务)     │     │ (外部集成)   │
└────┬─────┘     └───────┬────────┘     └────────┬─────────┘     └──────┬───────┘     └──────┬───────┘
     │                    │                       │                      │                    │
     │ 1. POST /api/book/event                    │                      │                    │
     │───────────────────►│                       │                      │                    │
     │                    │                       │                      │                    │
     │                    │ 2. createBooking()    │                      │                    │
     │                    │──────────────────────►│                      │                    │
     │                    │                       │                      │                    │
     │                    │                       │ 3. 前置检查          │                      │
     │                    │                       │   - 可用性检查        │                      │
     │                    │                       │   - 用户选择          │                      │
     │                    │                       │   - 事件构建          │                      │
     │                    │                       │                      │                    │
     │                    │                       │ 4. saveBooking()     │                      │
     │                    │                       │─────────────────────►│                    │
     │                    │                       │                      │                    │
     │                    │                       │                      │ ┌────────────────┐ │
     │                    │                       │                      │ │ prisma.$trans- │ │
     │                    │                       │                      │ │ action()       │ │
     │                    │                       │                      │ │                │ │
     │                    │                       │                      │ │ 4a. 原预约更新  │ │
     │                    │                       │                      │ │ (reschedule)   │ │
     │                    │                       │                      │ │                │ │
     │                    │                       │                      │ │ 4b. 创建新预约  │ │
     │                    │                       │                      │ │ (Booking表)    │ │
     │                    │                       │                      │ │                │ │
     │                    │                       │                      │ │ 4c. 创建参与者  │ │
     │                    │                       │                      │ │ (Attendee表)   │ │
     │                    │                       │                      │ └────────────────┘ │
     │                    │                       │                      │                    │
     │                    │                       │◄─────────────────────│                    │
     │                    │                       │   返回 booking 对象   │                    │
     │                    │                       │                      │                    │
     │                    │                       │ 5. EventManager.create() │                 │
     │                    │                       │──────────────────────────────────────────►│
     │                    │                       │                      │                    │
     │                    │                       │                      │                    ├──────────┐
     │                    │                       │                      │                    │ 外部API调用
     │                    │                       │                      │                    │ (Google/
     │                    │                       │                      │                    │  Outlook等)
     │                    │                       │                      │                    │
     │                    │                       │                      │                    │◄─────────┘
     │                    │                       │                      │                    │
     │                    │                       │                      │                    │ 5a. 创建视频会议
     │                    │                       │                      │                    │ (Daily/Zoom等)
     │                    │                       │                      │                    │
     │                    │                       │                      │                    │ 5b. 创建日历事件
     │                    │                       │                      │                    │
     │                    │                       │                      │                    │ 5c. 创建CRM事件
     │                    │                       │                      │                    │
     │                    │                       │◄───────────────────────────────────────────│
     │                    │                       │   返回 results + referencesToCreate         │
     │                    │                       │                      │                    │
     │                    │                       │ 6. 检查结果状态       │                      │
     │                    │                       │                      │                    │
     │                    │                       │    ┌─────────────────────┐                │
     │                    │                       │    │ 全部失败?           │                │
     │                    │                       │    │ results.every(      │                │
     │                    │                       │    │   res => !res.success│                │
     │                    │                       │    │ )                   │                │
     │                    │                       │    └──────────┬──────────┘                │
     │                    │                       │         │               │                 │
     │                    │                       │    是    │               │    否           │
     │                    │                       │         ▼               ▼                 │
     │                    │                       │    记录错误        继续执行              │
     │                    │                       │    (不回滚)      (保存成功引用)          │
     │                    │                       │                      │                    │
     │                    │                       │ 7. 更新iCalUID(如需要)│                  │
     │                    │                       │    prisma.booking.update()               │
     │                    │                       │                      │                    │
     │                    │                       │ 8. 发送邮件/触发Webhook │                 │
     │                    │                       │                      │                    │
     │                    │◄──────────────────────│                      │                    │
     │◄───────────────────│   返回 booking 响应   │                      │                    │
```

### 2.2 关键分支：成功与失败处理

**代码位置**：`RegularBookingService.ts:2064-2074`

```typescript
// 检查所有集成是否都失败
if (results.length > 0 && results.every((res) => !res.success)) {
  const error = {
    errorCode: "BookingCreatingMeetingFailed",
    message: "Booking failed",
  };

  tracingLogger.error(
    `EventManager.create failure in some of the integrations ${organizerUser.username}`,
    safeStringify({ error, results })
  );
  // 注意：仅记录错误，不回滚数据库操作！
} else {
  // 至少有一个集成成功，继续执行
  // 保存references、发送邮件等
}
```

**设计决策分析**：

| 场景 | 处理策略 | 原因 |
|------|---------|------|
| **全部失败** | 记录错误日志，预约仍为ACCEPTED状态 | 预约记录已保存，用户需要知道预约存在但日历写入失败 |
| **部分失败** | 保存成功的references，忽略失败的 | 最终一致性，失败的可后续重试 |
| **全部成功** | 正常流程，保存所有references | 理想情况 |

### 2.3 事务边界分析

**数据库事务范围**（`createBooking.ts:139-147`）：

```typescript
return prisma.$transaction(async (tx) => {
  // 1. 如果是重新安排，更新原预约为已取消
  if (originalBookingUpdateDataForCancellation) {
    await tx.booking.update(originalBookingUpdateDataForCancellation);
  }

  // 2. 创建新预约记录
  const booking = await tx.booking.create(createBookingObj);

  return { ...booking, userUuid: booking.user?.uuid ?? null };
});
```

**事务内操作**：
| 操作 | 表 | 说明 |
|------|-----|------|
| `tx.booking.update()` | `Booking` | 仅reschedule时，更新原预约状态为CANCELLED |
| `tx.booking.create()` | `Booking` | 创建新预约记录 |
| (隐含) | `Attendee` | 通过`createMany`创建参与者 |

**事务外操作**（无原子性保证）：
| 操作 | 组件 | 说明 |
|------|------|------|
| `EventManager.create()` | EventManager | 外部日历/视频/CRM API调用 |
| `prisma.booking.update()` (references) | 后续更新 | 保存外部引用 |
| `prisma.booking.update()` (iCalUID) | 后续更新 | 更新日历UID |
| 邮件发送 | `emailsAndSmsHandler` | 通知邮件 |
| Webhook触发 | `handleWebhookTrigger` | 事件通知 |

**关键问题**：数据库事务提交后，外部API调用可能失败，导致数据不一致。系统通过**日志记录**和**部分成功处理**来应对。

---

## 3. 需要确认模式时序（isConfirmedByDefault = false）

### 3.1 完整时序图

```
┌──────────┐     ┌────────────────┐     ┌──────────────────┐     ┌──────────────┐     ┌──────────────────┐
│ 访客前端  │     │ API Layer      │     │ RegularBooking   │     │  createBooking │     │ handleConfirmation │
│ (Client) │     │ (/api/book/    │     │ Service          │     │  (DB事务)     │     │ (确认流程)        │
└────┬─────┘     └───────┬────────┘     └────────┬─────────┘     └──────┬───────┘     └─────────┬────────┘
     │                    │                       │                      │                       │
     │ 1. POST /api/book/event                    │                      │                       │
     │───────────────────►│                       │                      │                       │
     │                    │                       │                      │                       │
     │                    │ 2. createBooking()    │                      │                       │
     │                    │──────────────────────►│                      │                       │
     │                    │                       │                      │                       │
     │                    │                       │ 3. 前置检查(同即时确认)│                      │
     │                    │                       │                      │                       │
     │                    │                       │ 4. saveBooking()     │                      │
     │                    │                       │─────────────────────►│                       │
     │                    │                       │                      │                       │
     │                    │                       │                      │ ┌────────────────┐    │
     │                    │                       │                      │ │ prisma.$trans- │    │
     │                    │                       │                      │ │ action()       │    │
     │                    │                       │                      │ │                │    │
     │                    │                       │                      │ │ 创建预约记录   │    │
     │                    │                       │                      │ │ status: PENDING│    │
     │                    │                       │                      │ │                │    │
     │                    │                       │                      │ │ 创建参与者记录 │    │
     │                    │                       │                      │ └────────────────┘    │
     │                    │                       │                      │                       │
     │                    │                       │◄─────────────────────│                       │
     │                    │                       │   返回 booking (PENDING)                     │
     │                    │                       │                      │                       │
     │                    │                       │ 5. 发送"请求确认"邮件 │                      │
     │                    │                       │                      │                       │
     │                    │◄──────────────────────│                      │                       │
     │◄───────────────────│   返回 PENDING 状态   │                      │                       │
     │                    │                       │                      │                       │
     │                    │                       │                      │                       │
     │  (等待组织者确认)   │                       │                      │                       │
     │                    │                       │                      │                       │
     │                    │                       │                      │                       │
┌────┴─────┐     ┌────────┴────────┐     ┌──────┴──────────┐     ┌──────┴───────┐     ┌─────────┴────────┐
│组织者前端 │     │ tRPC Router     │     │ confirm.handler  │     │               │     │                   │
│ (Organizer)│    │ (bookings.confirm)│    │                  │     │               │     │                   │
└────┬─────┘     └───────┬────────┘     └────────┬─────────┘     └───────────────┘     └─────────┬────────┘
     │                    │                       │                                            │
     │ 6. 确认预约         │                       │                                            │
     │ (调用 tRPC)         │                       │                                            │
     │───────────────────►│                       │                                            │
     │                    │                       │                                            │
     │                    │ 7. 确认处理           │                                            │
     │                    │──────────────────────►│                                            │
     │                    │                       │                                            │
     │                    │                       │ 8. handleConfirmation()                   │
     │                    │                       │────────────────────────────────────────────►│
     │                    │                       │                                            │
     │                    │                       │                                            ├──────────┐
     │                    │                       │                                            │ EventManager
     │                    │                       │                                            │ .create()
     │                    │                       │                                            │
     │                    │                       │                                            │ 8a. 创建视频会议
     │                    │                       │                                            │
     │                    │                       │                                            │ 8b. 创建日历事件
     │                    │                       │                                            │
     │                    │                       │                                            │ 8c. 创建CRM事件
     │                    │                       │                                            │
     │                    │                       │                                            │◄─────────┘
     │                    │                       │                                            │
     │                    │                       │ 9. 检查结果状态(同时序2.2)               │
     │                    │                       │                                            │
     │                    │                       │ 10. 更新预约状态和保存references           │
     │                    │                       │    prisma.booking.update({                │
     │                    │                       │      status: ACCEPTED,                     │
     │                    │                       │      references: { create: referencesToCreate },
     │                    │                       │      metadata: { videoCallUrl, ... }      │
     │                    │                       │    })                                       │
     │                    │                       │                                            │
     │                    │                       │ 11. 发送确认邮件/触发Webhook              │
     │                    │                       │                                            │
     │                    │◄──────────────────────│                                            │
     │◄───────────────────│                       │                                            │
```

### 3.2 确认流程核心代码

**handleConfirmation 关键逻辑** (`handleConfirmation.ts:245-308`)：

```typescript
// 步骤1：调用EventManager创建外部事件
const eventManager = new EventManager(user, apps);
const scheduleResult = await eventManager.create(evt, { 
  skipCalendarEvent: !areCalendarEventsEnabled 
});

// 步骤2：检查所有集成是否都失败
if (results.length > 0 && results.every((res) => !res.success)) {
  const error = {
    errorCode: "BookingCreatingMeetingFailed",
    message: "Booking failed",
  };
  tracingLogger.error(`Booking ${user.username} failed`, safeStringify({ error, results }));
  // 注意：即使全部失败，仍会继续执行，不会回滚！
}

// 步骤3：更新预约状态并保存references
const updatedBooking = await prisma.booking.update({
  where: { id: bookingId },
  data: {
    status: BookingStatus.ACCEPTED,  // PENDING → ACCEPTED
    references: {
      create: scheduleResult.referencesToCreate,  // 保存成功的外部引用
    },
    metadata: {
      ...(typeof booking.metadata === "object" ? booking.metadata : {}),
      videoCallUrl: meetingUrl,
    },
  },
  // ...select
});
```

### 3.3 两种模式对比

| 维度 | 即时确认模式 | 需要确认模式 |
|------|------------|-------------|
| **触发时机** | 访客提交表单后立即 | 组织者点击"确认"后 |
| **状态流转** | 直接创建为ACCEPTED | PENDING → (等待) → ACCEPTED |
| **日历写入时机** | 预约创建后立即 | 确认时才写入 |
| **References保存** | 后续单独update | 确认时与状态update一起 |
| **失败影响** | 预约已确认但日历可能失败 | 预约保持PENDING状态 |

---

## 4. 幂等防重复机制

### 4.1 多层次幂等设计

系统采用**多层次幂等机制**防止重复创建：

```
┌─────────────────────────────────────────────────────────────────┐
│                        幂等防护层次                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Layer 1: 业务层面 (IdempotencyKey)                            │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ IdempotencyKeyService.generate({                        │  │
│  │   startTime, endTime, userId, reassignedById           │  │
│  │ }) → UUID v5 (基于内容哈希)                              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  Layer 2: 预约层面 (Booking UID)                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ short.fromUUID(uuidv5(seed, uuidv5.URL))               │  │
│  │ seed = `${organizerUser.username}:${startTime}:${Date.now()}` │
│  └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  Layer 3: 日历层面 (iCalUID)                                  │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ getICalUID({ event, uid })                               │  │
│  │ 格式: <booking_uid>@cal.com                              │  │
│  │ 示例: abc123@cal.com                                     │  │
│  └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│  Layer 4: 外部引用层面 (BookingReference)                      │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ BookingReference 表存储:                                │  │
│  │ - uid: 外部服务返回的事件ID                              │  │
│  │ - type: 服务类型 (google_calendar, daily_video等)      │  │
│  │ - externalCalendarId: 外部日历ID                        │  │
│  │ - credentialId: 关联的凭证ID                            │  │
│  │ - deleted: 软删除标记                                    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 各层幂等机制详解

#### Layer 1: IdempotencyKeyService

**代码位置**：`packages/lib/idempotencyKey/idempotencyKeyService.ts`

```typescript
export class IdempotencyKeyService {
  static generate({
    startTime,
    endTime,
    userId,
    reassignedById,
  }: {
    startTime: Date | string;
    endTime: Date | string;
    userId?: number;
    reassignedById?: number | null;
  }) {
    return uuidv5(
      `${startTime.valueOf()}.${endTime.valueOf()}.${userId}${reassignedById ? `.${reassignedById}` : ""}`,
      uuidv5.URL
    );
  }
}
```

**使用场景**：
- 日历订阅同步（`CalendarSyncService`）
- 防止外部日历变更触发重复操作

#### Layer 2: Booking UID 生成

**代码位置**：`RegularBookingService.ts:1267-1268`

```typescript
const seed = `${organizerUser.username}:${dayjs(reqBody.start).utc().format()}:${Date.now()}`;
const uid = translator.fromUUID(uuidv5(seed, uuidv5.URL));
```

**特点**：
- 包含时间戳（`Date.now()`），每次调用生成不同的UID
- 用于Booking表的唯一标识
- **注意**：由于包含时间戳，不适合用于防止重复提交

#### Layer 3: iCalUID 生成

**代码位置**：`RegularBookingService.ts:1306-1309`

```typescript
const iCalUID = getICalUID({
  event: { iCalUID: originalRescheduledBooking?.iCalUID, uid: originalRescheduledBooking?.uid },
  uid,
});
```

**格式**：`<booking_uid>@cal.com`

**用途**：
- 日历事件的全局唯一标识
- 外部日历通过此字段识别事件
- 用于日历订阅同步时的事件匹配

#### Layer 4: BookingReference 外部引用

**数据结构**（Prisma Schema）：

```prisma
model BookingReference {
  id             Int       @id @default(autoincrement())
  type           String    // google_calendar, daily_video, etc.
  uid            String    // 外部服务返回的事件ID
  meetingId      String?   // 视频会议ID
  meetingPassword String? // 视频会议密码
  meetingUrl     String?   // 视频会议链接
  externalCalendarId String? // 外部日历ID
  credentialId   Int?      // 关联的凭证
  deleted        DateTime? // 软删除标记
  
  booking        Booking   @relation(fields: [bookingId], references: [id])
  bookingId      Int
  
  // 索引用于快速查找
  @@index([type, uid])
  @@index([bookingId])
}
```

**幂等保障**：
- 每次 `EventManager.create()` 返回 `referencesToCreate`
- 后续通过 `references: { create: referencesToCreate }` 保存
- 如果同一事件被多次处理，可能产生重复引用（需业务层判断）

### 4.3 日历订阅同步的幂等设计

**代码位置**：`CalendarSyncService.ts`

```typescript
async function rescheduleBooking(event: CalendarSubscriptionEventItem, calendarUserId: number) {
  // 1. 从 iCalUID 解析 booking UID
  const [bookingUid] = event.iCalUID?.split("@") ?? [undefined];
  
  // 2. 查找预约
  const booking = await this.deps.bookingRepository.findBookingByUidWithEventType({ bookingUid });
  
  // 3. 检查时间是否真正改变
  if (!hasStartTimeChanged(booking, event)) {
    log.debug("Skipping reschedule, start time has not changed", { bookingUid });
    return; // 幂等：时间未变化则跳过
  }
  
  // 4. 生成幂等键
  const idempotencyKey = IdempotencyKeyService.generate({
    startTime: new Date(start),
    endTime: new Date(end),
    userId: booking.userId ?? undefined,
    reassignedById: null,
  });
  
  // 5. 调用预约服务（传入幂等键）
  await regularBookingService.createBooking({
    bookingData: {
      ...,
      idempotencyKey,
      rescheduleUid: booking.uid,
    },
    bookingMeta: {
      skipCalendarSyncTaskCreation: true, // 关键：防止循环同步
      skipAvailabilityCheck: true,
      skipEventLimitsCheck: true,
    },
  });
}
```

**关键防循环机制**：

| 参数 | 值 | 目的 |
|------|-----|------|
| `skipCalendarSyncTaskCreation` | `true` | 防止外部日历→Cal.diy→外部日历的无限循环 |
| `skipAvailabilityCheck` | `true` | 跳过可用性检查（外部已验证） |
| `skipEventLimitsCheck` | `true` | 跳过事件限制检查 |

---

## 5. 补偿与重试逻辑

### 5.1 补偿策略概览

系统采用**软补偿**策略，没有分布式事务管理器（如Saga）：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         补偿策略层次                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Level 1: 部分失败容忍 (EventManager内部)                                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ 使用 Promise.allSettled 而非 Promise.all                           │  │
│  │ 一个集成失败不影响其他集成                                           │  │
│  │ 失败仅记录日志，不抛出异常                                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Level 2: 操作反向补偿 (Reschedule/Cancel场景)                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ deleteEventsAndMeetings()                                          │  │
│  │ - 删除日历事件                                                       │  │
│  │ - 删除视频会议                                                       │  │
│  │ - 删除CRM事件                                                        │  │
│  │                                                                      │  │
│  │ 使用场景：                                                            │  │
│  │ - 重新安排预约时，先删除旧事件                                       │  │
│  │ - 取消预约时，删除外部事件                                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Level 3: 日志记录与人工介入                                             │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ tracingLogger.error() / warn()                                     │  │
│  │ - 记录失败的集成类型                                                 │  │
│  │ - 记录错误详情                                                       │  │
│  │ - 记录相关上下文（用户、事件类型等）                                  │  │
│  │                                                                      │  │
│  │ 人工处理：                                                            │  │
│  │ - 通过管理后台查看失败日志                                           │  │
│  │ - 手动重新同步                                                       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Level 1: 部分失败容忍

**EventManager.deleteEventsAndMeetings 实现** (`EventManager.ts:792-850`)：

```typescript
public async deleteEventsAndMeetings({
  event,
  bookingReferences,
  isBookingInRecurringSeries,
}: {
  event: CalendarEvent;
  bookingReferences: PartialReference[];
  isBookingInRecurringSeries?: boolean;
}) {
  const calendarReferences = [],
    videoReferences = [],
    crmReferences = [],
    allPromises = [];

  // 分类不同类型的引用
  for (const reference of bookingReferences) {
    if (reference.type.includes("_calendar") && !reference.type.includes("other_calendar")) {
      calendarReferences.push(reference);
      allPromises.push(
        this.deleteCalendarEventForBookingReference({
          reference,
          event,
          isBookingInRecurringSeries,
        })
      );
    }

    if (reference.type.includes("_video")) {
      videoReferences.push(reference);
      allPromises.push(
        this.deleteVideoEventForBookingReference({ reference })
      );
    }

    if (reference.type.includes("_crm") || reference.type.includes("other_calendar")) {
      crmReferences.push(reference);
      allPromises.push(this.deleteCRMEvent({ reference, event }));
    }
  }

  // 关键：使用 allSettled 确保部分失败不影响整体
  (await Promise.allSettled(allPromises)).some((result) => {
    if (result.status === "rejected") {
      // 仅记录警告日志，不抛出异常
      log.warn(
        "Error deleting calendar event or video meeting for booking",
        safeStringify({ error: result.reason })
      );
    }
  });

  if (!allPromises.length) {
    log.warn("No calendar or video references found for booking - Couldn't delete events or meetings");
  }
}
```

**设计意图**：
- **软删除**：即使某些外部服务不可用，也不阻塞整体流程
- **日志追踪**：失败情况被详细记录，便于后续排查
- **最终一致性**：依赖人工或定时任务进行后续修复

### 5.3 Level 2: 反向补偿场景

#### 场景A：重新安排预约

**代码位置**：`RegularBookingService.ts:1868-1902`

```typescript
// 重新安排时，先删除旧的外部事件
if (!skipCalendarSyncTaskCreation) {
  await originalHostEventManager.deleteEventsAndMeetings({
    event: deletionEvent,
    bookingReferences: originalRescheduledBooking.references,
  });
}
```

**补偿流程**：

```
┌──────────────────────────────────────────────────────────────────────┐
│                      重新安排预约补偿流程                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. 调用 originalHostEventManager.deleteEventsAndMeetings()          │
│                           │                                           │
│                           ▼                                           │
│  2. 遍历 originalRescheduledBooking.references                       │
│                           │                                           │
│           ┌───────────────┼───────────────┐                         │
│           ▼               ▼               ▼                         │
│    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                 │
│    │ 日历引用     │ │ 视频引用     │ │ CRM引用      │                 │
│    │ (_calendar) │ │ (_video)    │ │ (_crm)       │                 │
│    └──────┬──────┘ └──────┬──────┘ └──────┬──────┘                 │
│           │               │               │                         │
│           ▼               ▼               ▼                         │
│    deleteCalendarEvent  deleteVideoEvent  deleteCRMEvent            │
│                           │                                           │
│                           ▼                                           │
│  3. 使用 Promise.allSettled 并行执行，忽略个别失败                    │
│                           │                                           │
│                           ▼                                           │
│  4. 继续创建新的预约和事件                                              │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

#### 场景B：取消预约

**类似机制**：取消预约时也会调用 `deleteEventsAndMeetings` 删除外部事件。

### 5.4 Level 3: 日志记录策略

**日志级别使用**：

| 级别 | 使用场景 | 示例 |
|------|---------|------|
| `error` | 重要操作失败 | EventManager.create 全部失败 |
| `warn` | 非关键失败、配置问题 | 某个集成删除失败、凭证过期警告 |
| `debug` | 详细流程信息 | 操作开始/结束、参数详情 |
| `info` | 重要操作成功 | 预约创建成功、同步完成 |

**关键错误日志示例** (`RegularBookingService.ts:2070-2073`)：

```typescript
if (results.length > 0 && results.every((res) => !res.success)) {
  const error = {
    errorCode: "BookingCreatingMeetingFailed",
    message: "Booking failed",
  };

  tracingLogger.error(
    `EventManager.create failure in some of the integrations ${organizerUser.username}`,
    safeStringify({ error, results })
  );
}
```

**日志内容包含**：
- 错误码（`errorCode`）
- 用户名（便于追踪）
- 完整的 results 数组（每个集成的成功/失败状态）

---

## 6. 前端触发入口关系

### 6.1 完整调用链

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           前端触发入口关系                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  场景1: 访客提交预约（即时确认）                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                           │  │
│  │  前端组件                                                                   │  │
│  │  ┌─────────────┐                                                          │  │
│  │  │ Booker      │  收集表单数据                                             │  │
│  │  │ (预约表单)   │                                                          │  │
│  │  └──────┬──────┘                                                          │  │
│  │         │                                                                  │  │
│  │         ▼                                                                  │  │
│  │  ┌─────────────────────────┐                                              │  │
│  │  │ createBooking()          │                                              │  │
│  │  │ (features/bookings/lib/  │                                              │  │
│  │  │  create-booking.ts)      │                                              │  │
│  │  └───────────┬─────────────┘                                              │  │
│  │              │                                                              │  │
│  │              ▼                                                              │  │
│  │  ┌─────────────────────────┐                                              │  │
│  │  │ POST /api/book/event    │                                              │  │
│  │  │ (pages/api/book/event.ts)│                                              │  │
│  │  └───────────┬─────────────┘                                              │  │
│  │              │                                                              │  │
│  │              ▼                                                              │  │
│  │  ┌─────────────────────────────────┐                                      │  │
│  │  │ RegularBookingService.createBooking() │                                │  │
│  │  │ (即时确认模式)                       │                                │  │
│  │  └───────────────────┬─────────────────┘                                      │  │
│  │                      │                                                        │  │
│  │                      ├──► 数据库事务 (saveBooking)                           │  │
│  │                      │                                                        │  │
│  │                      └──► EventManager.create()                              │  │
│  │                                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  场景2: 组织者确认预约（需要确认模式）                                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                           │  │
│  │  前端组件                                                                   │  │
│  │  ┌─────────────────────┐                                                  │  │
│  │  │ AcceptBookingButton │  点击确认按钮                                     │  │
│  │  │ 或 useBookingConfirm │                                                 │  │
│  │  └──────────┬──────────┘                                                  │  │
│  │             │                                                               │  │
│  │             ▼                                                               │  │
│  │  ┌─────────────────────────┐                                              │  │
│  │  │ tRPC Mutation           │                                              │  │
│  │  │ bookings.confirm        │                                              │  │
│  │  └───────────┬─────────────┘                                              │  │
│  │              │                                                              │  │
│  │              ▼                                                              │  │
│  │  ┌─────────────────────────┐                                              │  │
│  │  │ confirm.handler.ts      │                                              │  │
│  │  │ (trpc/server/routers/   │                                              │  │
│  │  │  viewer/bookings/)       │                                              │  │
│  │  └───────────┬─────────────┘                                              │  │
│  │              │                                                              │  │
│  │              ▼                                                              │  │
│  │  ┌─────────────────────────────┐                                          │  │
│  │  │ handleConfirmation()         │                                          │  │
│  │  │ (features/bookings/lib/     │                                          │  │
│  │  │  handleConfirmation.ts)      │                                          │  │
│  │  └───────────────┬─────────────┘                                          │  │
│  │                  │                                                          │  │
│  │                  ├──► EventManager.create()                                │  │
│  │                  │                                                          │  │
│  │                  └──► prisma.booking.update()                              │  │
│  │                       (status: ACCEPTED + references)                      │  │
│  │                                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键入口文件

#### 入口1: 前端预约创建

**文件**：`packages/features/bookings/lib/create-booking.ts`

```typescript
export const createBooking = async (data: BookingCreateBody) => {
  const response = await post<
    BookingCreateBody,
    Omit<BookingResponse, "startTime" | "endTime"> & {
      startTime: string;
      endTime: string;
    }
  >("/api/book/event", data);
  return response;
};
```

**调用方**：预约表单组件（Booker）

#### 入口2: API端点

**文件**：`apps/web/pages/api/book/event.ts`

```typescript
async function handler(req: NextApiRequest & { userId?: number; traceContext: TraceContext }) {
  // ... 安全检查（人机验证、机器人检测、速率限制）
  
  const regularBookingService = getRegularBookingService();
  const booking = await regularBookingService.createBooking({
    bookingData: req.body,
    bookingMeta: {
      userId: session?.user?.id || -1,
      hostname: req.headers.host || "",
      // ...
    },
  });

  return booking;
}
```

#### 入口3: tRPC确认路由

**文件**：`packages/trpc/server/routers/viewer/bookings/confirm.handler.ts`

```typescript
// 处理组织者确认预约
export const confirmHandler = async ({ ctx, input }: { ctx: TrpcContext; input: Z.infer<typeof confirmSchema> }) => {
  // ... 获取预约信息
  
  await handleConfirmation({
    user: { ...organizerUser, credentials, destinationCalendar },
    evt: calEvent,
    prisma: ctx.prisma,
    bookingId: booking.id,
    booking,
    // ...
  });
  
  // ...
};
```

### 6.3 两种模式的触发条件

**isConfirmedByDefault 计算逻辑** (`RegularBookingService.ts`)：

```typescript
const { userReschedulingIsOwner, isConfirmedByDefault } = await getRequiresConfirmationFlags({
  eventType,
  bookingStartTime: reqBody.start,
  userId,
  originalRescheduledBookingOrganizerId: originalRescheduledBooking?.user?.id,
  paymentAppData,
  bookerEmail,
});
```

**影响因素**：

| 因素 | 说明 |
|------|------|
| `eventType.requiresConfirmation` | 事件类型是否需要组织者确认 |
| `paymentAppData` | 是否需要支付（未支付时可能需要确认） |
| `userReschedulingIsOwner` | 重新安排时是否是组织者本人操作 |
| 其他业务规则 | 如工作时间外的预约等 |

---

## 7. 关键数据结构

### 7.1 EventResult（外部集成结果）

```typescript
// 来自 @calcom/types/EventManager
interface EventResult<T> {
  type: string;           // 集成类型: google_calendar, daily_video, etc.
  success: boolean;       // 是否成功
  uid?: string;           // 外部事件ID
  createdEvent?: T;       // 创建的事件详情
  updatedEvent?: T;       // 更新的事件详情
  originalEvent?: CalendarEvent; // 原始事件
  credentialId?: number;  // 使用的凭证ID
  delegatedToId?: string; // 委托凭证ID
  externalId?: string;    // 外部日历ID
  iCalUID?: string;       // 日历事件UID
  calWarnings?: string[]; // 警告信息
  error?: Error;          // 错误详情（失败时）
}
```

### 7.2 PartialReference（待保存的引用）

```typescript
interface PartialReference {
  type: string;                    // 集成类型
  uid: string;                     // 外部事件ID
  meetingId?: string;              // 视频会议ID
  meetingPassword?: string;        // 视频会议密码
  meetingUrl?: string;             // 视频会议链接
  externalCalendarId?: string;      // 外部日历ID
  thirdPartyRecurringEventId?: string; // 第三方重复事件ID
  credentialId?: number;           // 凭证ID
  delegationCredentialId?: string;  // 委托凭证ID
}
```

### 7.3 BookingReference（数据库存储结构）

```prisma
model BookingReference {
  id             Int       @id @default(autoincrement())
  type           String
  uid            String
  meetingId      String?
  meetingPassword String?
  meetingUrl     String?
  externalCalendarId String?
  credentialId   Int?
  deleted        DateTime?
  
  booking        Booking   @relation(fields: [bookingId], references: [id])
  bookingId      Int
  
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt
  
  @@index([bookingId])
  @@index([type, uid])
}
```

---

## 8. 总结与设计评价

### 8.1 设计优点

1. **最终一致性**：在分布式场景下（数据库 + 多个外部API），采用最终一致性是务实的选择
2. **多层次幂等**：通过UID、iCalUID、IdempotencyKey等多层机制防止重复
3. **部分失败容忍**：使用 `Promise.allSettled` 确保一个集成失败不影响其他
4. **清晰的状态机**：PENDING → ACCEPTED 状态流转明确

### 8.2 潜在风险与注意事项

| 风险 | 场景 | 缓解措施 |
|------|------|---------|
| **数据不一致** | 数据库事务提交后，外部API全部失败 | 详细日志记录 + 人工介入 |
| **部分成功** | 日历创建成功但视频会议失败 | 保存成功的references，失败的需手动处理 |
| **重复创建** | 网络超时导致前端重试 | 业务层需实现幂等检查 |
| **循环同步** | 外部日历→Cal.diy→外部日历 | `skipCalendarSyncTaskCreation` 标志 |

### 8.3 关键代码位置速查

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 数据库事务 | `handleNewBooking/createBooking.ts` | 139-147 |
| 全部失败检查 | `service/RegularBookingService.ts` | 2064-2074 |
| 部分失败容忍 | `EventManager.ts` | 792-850 (allSettled) |
| 确认流程 | `handleConfirmation.ts` | 245-308 |
| 幂等键生成 | `lib/idempotencyKey/idempotencyKeyService.ts` | 3-19 |
| 日历同步防循环 | `calendar-subscription/CalendarSyncService.ts` | 204-213 |
| 前端创建入口 | `features/bookings/lib/create-booking.ts` | 5-14 |
