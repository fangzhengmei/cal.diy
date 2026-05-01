# Cal.diy 幂等机制与补偿策略深度分析报告

## 0. 前言：核心发现摘要

### 0.1 关于"幂等防重复"的重要澄清

**之前的分析存在误导**。经过深入代码分析，发现：

| 链路 | `idempotencyKey` 状态 | 实际幂等机制 |
|------|----------------------|-------------|
| **主预约创建链路** | ❌ 值为 `null`，**未使用** | 基于 `uid` + `iCalUID` |
| **外部同步链路** | ✅ 生成并传递，但**无查询检查** | 基于 `hasStartTimeChanged()` + `iCalUID` 解析 |

**关键结论**：`IdempotencyKeyService.generate()` 生成的键**没有任何地方用于查询或检查重复**。它只是一个数据字段，被存储但从未被查询。

### 0.2 关于"补偿与重试"的重要澄清

| 场景 | 补偿机制 | 重试机制 |
|------|---------|---------|
| **建库成功但外部写入失败** | ❌ **无主动补偿** | ❌ **无主动重试** |
| **reschedule 场景** | ✅ 主动删除旧事件 | ❌ 无重试 |
| **cancel 场景** | ✅ 主动删除事件 | ❌ 无重试 |

**关键结论**：当数据库成功但外部日历写入失败时，系统**没有任何自动补偿或重试机制**。只有日志记录，依赖人工介入。

---

## 1. 幂等防重复机制分层分析

### 1.1 两条链路的幂等机制对比

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                           两条链路的幂等机制对比                                        │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌─────────────────────────────────────┐    ┌─────────────────────────────────────┐    │
│  │      主预约创建链路                   │    │      外部同步链路                     │    │
│  │   (访客提交预约表单)                  │    │   (CalendarSyncService)              │    │
│  └───────────────────┬─────────────────┘    └───────────────────┬─────────────────┘    │
│                      │                                            │                      │
│                      ▼                                            ▼                      │
│  ┌─────────────────────────────────────┐    ┌─────────────────────────────────────┐    │
│  │ idempotencyKey 状态                  │    │ idempotencyKey 状态                  │    │
│  │                                     │    │                                     │    │
│  │ 值: null                            │    │ 值: 生成 UUID v5                    │    │
│  │ 来源: buildDryRunBooking 硬编码     │    │ 来源: IdempotencyKeyService.generate │    │
│  │                                     │    │                                     │    │
│  │ ❌ 从未被查询用于幂等检查            │    │ ❌ 生成后传递，但无查询检查           │    │
│  │ ❌ 只是数据字段，无实际作用          │    │ ❌ 只是传递，无实际幂等逻辑           │    │
│  └─────────────────────────────────────┘    └─────────────────────────────────────┘    │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 主预约创建链路的实际幂等机制

#### 1.2.1 链路概览

**入口**：
- `POST /api/book/event` (普通预约)
- `POST /api/book/recurring-event` (重复预约)

**核心服务**：`RegularBookingService.createBooking()`

#### 1.2.2 `idempotencyKey` 的实际状态

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts:204`

```typescript
// buildDryRunBooking 函数中
idempotencyKey: null,  // 硬编码为 null，从未被赋值
```

**验证证据**：搜索整个代码库，**没有任何地方**使用 `idempotencyKey` 进行查询：

```
搜索结果：
- 无 `findFirst.*idempotencyKey`
- 无 `findMany.*idempotencyKey`
- 无 `WHERE.*idempotencyKey`
```

#### 1.2.3 实际生效的幂等机制（第一层：Booking UID）

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts:1267-1268`

```typescript
const seed = `${organizerUser.username}:${dayjs(reqBody.start).utc().format()}:${Date.now()}`;
const uid = translator.fromUUID(uuidv5(seed, uuidv5.URL));
```

**分析**：

| 组成部分 | 说明 | 幂等性影响 |
|---------|------|-----------|
| `organizerUser.username` | 组织者用户名 | 固定 |
| `dayjs(reqBody.start).utc().format()` | 预约开始时间 | 固定（相同预约时间） |
| `Date.now()` | 当前时间戳 | **每次不同** |

**关键问题**：由于包含 `Date.now()`，**每次调用都会生成不同的 UID**。这意味着：
- ✅ 数据库层面：UID 是主键，防止同一记录重复插入
- ❌ 业务层面：无法防止用户重复提交相同的预约请求

**实际效果**：如果用户快速点击"提交"两次，会创建两个独立的预约记录，具有不同的 UID。

#### 1.2.4 实际生效的幂等机制（第二层：iCalUID）

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts:1306-1309`

```typescript
const iCalUID = getICalUID({
  event: { iCalUID: originalRescheduledBooking?.iCalUID, uid: originalRescheduledBooking?.uid },
  uid,
});
```

**格式**：`<booking_uid>@cal.com`

**用途**：
1. **外部日历标识**：外部日历（Google/Outlook）通过此字段识别事件
2. **重新安排追踪**：`originalRescheduledBooking.iCalUID` 用于追踪重新安排的事件

**代码位置**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:82, 157`

```typescript
// 从外部日历事件的 iCalUID 解析出 bookingUid
const [bookingUid] = event.iCalUID?.split("@") ?? [undefined];
```

**实际幂等效果**：
- ✅ 外部日历 → Cal.diy 同步时，通过 `iCalUID` 解析找到对应的预约
- ❌ 主预约创建链路中，无任何基于 `iCalUID` 的重复检查

#### 1.2.5 实际生效的幂等机制（第三层：rescheduleUid + fromReschedule）

**代码位置**：`packages/features/bookings/repositories/BookingRepository.ts:577-586`

```typescript
async findFirstBookingByReschedule({ originalBookingUid }: { originalBookingUid: string }) {
  return await this.prismaClient.booking.findFirst({
    where: {
      fromReschedule: originalBookingUid,  // 基于 fromReschedule 字段查询
    },
    select: {
      uid: true,
    },
  });
}
```

**用途**：用于追踪重新安排的预约关系

**数据模型**：
```
原预约 (uid: abc123)
    │
    └──► 被取消，status: CANCELLED
         fromReschedule: null
         rescheduled: true
         
新预约 (uid: def456)
    │
    └──► status: ACCEPTED
         fromReschedule: 'abc123'  // 指向原预约
```

**实际效果**：
- ✅ 可以查询某个预约是否已经被重新安排过
- ❌ 不是用于防止重复创建的幂等机制

#### 1.2.6 主预约创建链路幂等机制总结

| 机制 | 代码位置 | 实际用途 | 是否防止重复创建 |
|------|---------|---------|-----------------|
| `idempotencyKey` | `RegularBookingService.ts:204` | ❌ 值为 null，无作用 | ❌ |
| `uid` | `RegularBookingService.ts:1267-1268` | 预约唯一标识 | ⚠️ 数据库层面，非业务层面 |
| `iCalUID` | `RegularBookingService.ts:1306-1309` | 外部日历标识 | ⚠️ 外部同步时使用 |
| `fromReschedule` | `BookingRepository.ts:577` | 重新安排追踪 | ❌ 非幂等用途 |

**关键结论**：主预约创建链路**没有业务层面的幂等保护**。如果用户重复提交相同的预约请求，会创建多个独立的预约记录。

---

### 1.3 外部同步链路的实际幂等机制

#### 1.3.1 链路概览

**入口**：`CalendarSyncService.handleEvents()`

**触发时机**：外部日历（Google/Outlook）事件变更时

**核心场景**：
1. 外部日历中事件被取消 → Cal.diy 中取消对应预约
2. 外部日历中事件时间被修改 → Cal.diy 中重新安排对应预约

#### 1.3.2 `idempotencyKey` 的生成与传递

**代码位置**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:266-271`

```typescript
return {
  eventTypeId: booking.eventTypeId!,
  start,
  end,
  // ...
  rescheduleUid: booking.uid,
  idempotencyKey: IdempotencyKeyService.generate({
    startTime: new Date(start),
    endTime: new Date(end),
    userId: booking.userId ?? undefined,
    reassignedById: null,
  }),
  // ...
};
```

**`IdempotencyKeyService` 实现**：`packages/lib/idempotencyKey/idempotencyKeyService.ts`

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

**分析**：

| 组成部分 | 说明 | 相同请求时 |
|---------|------|-----------|
| `startTime.valueOf()` | 开始时间戳 | 相同 |
| `endTime.valueOf()` | 结束时间戳 | 相同 |
| `userId` | 用户ID | 相同 |
| `reassignedById` | 重新分配ID | 相同（通常为 null） |

**特点**：基于内容哈希，相同输入总是生成相同的 key。

#### 1.3.3 `idempotencyKey` 的实际使用情况

**关键发现**：搜索整个代码库，**没有任何地方**使用这个生成的 `idempotencyKey` 进行查询。

```
搜索结果：
- 无 `findFirst.*idempotencyKey`
- 无 `findMany.*idempotencyKey`
- 无 `WHERE.*idempotencyKey`
- 只有 `ManagedEventReassignment` 场景中显式设置并存储
```

**代码追踪**：

```typescript
// CalendarSyncService.ts:204-213
await regularBookingService.createBooking({
  bookingData: buildRescheduleBookingData(booking, event),  // 包含 idempotencyKey
  bookingMeta: {
    skipCalendarSyncTaskCreation: true,  // 关键标志
    skipAvailabilityCheck: true,
    skipEventLimitsCheck: true,
  },
});

// buildRescheduleBookingData 返回类型
type RescheduleBookingData = CreateRegularBookingData & {
  responses: Record<string, unknown>;
  idempotencyKey: string;  // 类型定义存在
};
```

**实际流向**：
1. `CalendarSyncService` 生成 `idempotencyKey`
2. 传递给 `RegularBookingService.createBooking()`
3. 存储到数据库的 `Booking.idempotencyKey` 字段
4. **从未被查询或用于任何幂等检查**

#### 1.3.4 实际生效的幂等机制（第一层：iCalUID 解析）

**代码位置**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:82, 157`

```typescript
// cancelBooking 和 rescheduleBooking 中
const [bookingUid] = event.iCalUID?.split("@") ?? [undefined];
if (!bookingUid) {
  log.debug("Unable to sync, booking UID not found in iCalUID");
  return;  // 无法解析则跳过
}
```

**逻辑**：
1. 外部日历事件的 `iCalUID` 格式为 `<booking_uid>@cal.com`
2. 解析出 `bookingUid`
3. 通过 `bookingUid` 查找 Cal.diy 中的预约记录

**幂等效果**：
- ✅ 确保只处理 Cal.diy 创建的事件（通过 `@cal.com` 后缀判断）
- ✅ 确保找到对应的预约记录

**代码位置**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:46-48`

```typescript
// 只处理 Cal.com 创建的日历事件
const calEvents = calendarSubscriptionEvents.filter((e) =>
  e.iCalUID?.toLowerCase()?.endsWith("@cal.com")
);
```

#### 1.3.5 实际生效的幂等机制（第二层：hasStartTimeChanged）

**代码位置**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:190-196`

```typescript
if (!hasStartTimeChanged(booking, event)) {
  log.debug("Skipping reschedule, start time has not changed", { bookingUid });
  metrics.count("calendar.sync.rescheduleBooking.calls", 1, {
    attributes: { status: "skipped", reason: "no_time_change" },
  });
  return;  // 时间未变化则跳过
}
```

**`hasStartTimeChanged` 实现**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:351-357`

```typescript
export const hasStartTimeChanged = (
  booking: BookingWithEventType,
  event: CalendarSubscriptionEventItem
): boolean => {
  if (!event.start) return false;
  return event.start.getTime() !== booking.startTime.getTime();
};
```

**幂等效果**：
- ✅ 防止外部日历重复推送相同的时间变更
- ✅ 防止无限循环（如果 Cal.diy → 外部日历 → Cal.diy 同步链路上时间未真正变化）

#### 1.3.6 实际生效的幂等机制（第三层：skipCalendarSyncTaskCreation）

**代码位置**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:209`

```typescript
bookingMeta: {
  // Skip calendar event creation to avoid infinite loops
  // (Google/Office365 → Cal.diy → Google/Office365 → ...)
  skipCalendarSyncTaskCreation: true,  // 关键标志
  skipAvailabilityCheck: true,
  skipEventLimitsCheck: true,
}
```

**作用**：
1. 当从外部日历同步到 Cal.diy 时，**不创建外部日历事件**
2. 防止 `Google → Cal.diy → Google → Cal.diy...` 的无限循环

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts:1833-1836`

```typescript
const eventManager =
  !isDryRun && !skipCalendarSyncTaskCreation
    ? new EventManager({ ...organizerUser, credentials }, apps)
    : buildDryRunEventManager();  // 使用空操作的 EventManager
```

**`buildDryRunEventManager` 实现**：

```typescript
const buildDryRunEventManager = () => {
  return {
    create: async () => ({ results: [], referencesToCreate: [] }),
    reschedule: async () => ({ results: [], referencesToCreate: [] }),
  };
};
```

**幂等效果**：
- ✅ 防止无限循环同步
- ✅ 确保外部日历的变更只被 Cal.diy 消费一次

#### 1.3.7 外部同步链路幂等机制总结

| 机制 | 代码位置 | 实际用途 | 是否防止重复操作 |
|------|---------|---------|-----------------|
| `idempotencyKey` | `CalendarSyncService.ts:266-271` | ❌ 生成后存储，无查询 | ❌ |
| `iCalUID` 解析 | `CalendarSyncService.ts:82, 157` | 找到对应预约 | ✅ |
| `hasStartTimeChanged()` | `CalendarSyncService.ts:190-196` | 检查时间是否真正变化 | ✅ |
| `skipCalendarSyncTaskCreation` | `CalendarSyncService.ts:209` | 防止无限循环 | ✅ |

**关键结论**：外部同步链路的**实际幂等机制**是：
1. `iCalUID` 解析找到对应预约
2. `hasStartTimeChanged()` 检查时间是否真正变化
3. `skipCalendarSyncTaskCreation` 防止无限循环

**`idempotencyKey` 只是一个被存储但从未被查询的数据字段**。

---

### 1.4 幂等机制生效层级总览

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                        幂等机制生效层级总览                                              │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                        主预约创建链路                                              │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  层级 1: 数据库主键 (uid)                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 机制: uid 是 Booking 表的唯一键                                           │   │  │
│  │  │ 效果: 防止同一记录重复插入                                                 │   │  │
│  │  │ 限制: 每次请求生成不同 uid（因 Date.now()），无法防止业务层面重复提交      │   │  │
│  │  │ 代码: RegularBookingService.ts:1267-1268                                 │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                  │  │
│  │  层级 2: iCalUID（仅用于外部标识）                                              │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 机制: 格式为 <uid>@cal.com                                                │   │  │
│  │  │ 效果: 外部日历通过此字段识别事件                                           │   │  │
│  │  │ 限制: 主链路中无任何基于 iCalUID 的重复检查                               │   │  │
│  │  │ 代码: RegularBookingService.ts:1306-1309                                 │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                  │  │
│  │  层级 3: idempotencyKey（❌ 无实际作用）                                      │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 机制: 硬编码为 null                                                       │   │  │
│  │  │ 效果: 无任何作用                                                          │   │  │
│  │  │ 代码: RegularBookingService.ts:204                                        │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                        外部同步链路                                                │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  层级 1: iCalUID 解析（✅ 实际生效）                                            │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 机制: 从外部事件 iCalUID 解析出 bookingUid                               │   │  │
│  │  │ 效果: 找到 Cal.diy 中对应的预约记录                                      │   │  │
│  │  │ 代码: CalendarSyncService.ts:82, 157                                     │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                  │  │
│  │  层级 2: hasStartTimeChanged()（✅ 实际生效）                                  │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 机制: 比较外部事件时间与预约时间                                          │   │  │
│  │  │ 效果: 时间未变化则跳过操作，防止重复处理                                  │   │  │
│  │  │ 代码: CalendarSyncService.ts:190-196                                     │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                  │  │
│  │  层级 3: skipCalendarSyncTaskCreation（✅ 实际生效）                           │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 机制: 使用空操作的 EventManager                                           │   │  │
│  │  │ 效果: 防止 Google → Cal.diy → Google 的无限循环                         │   │  │
│  │  │ 代码: CalendarSyncService.ts:209                                          │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                  │  │
│  │  层级 4: idempotencyKey（❌ 无实际作用）                                      │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 机制: 生成 UUID v5 并传递                                                 │   │  │
│  │  │ 效果: 存储到数据库，但从未被查询                                          │   │  │
│  │  │ 代码: CalendarSyncService.ts:266-271                                      │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 建库成功但外部写入失败后的补偿与重试机制

### 2.1 事务边界与失败场景分析

#### 2.1.1 时序图：成功路径

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    建库成功 + 外部写入成功（理想路径）                                   │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  RegularBookingService.createBooking()                                                  │
│           │                                                                              │
│           ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 步骤 1: 数据库事务（原子性）                                                       │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  prisma.$transaction(async (tx) => {                                           │  │
│  │     // 如果是 reschedule，更新原预约为 CANCELLED                                │  │
│  │     await tx.booking.update(originalBookingUpdateDataForCancellation);         │  │
│  │                                                                                  │  │
│  │     // 创建新预约记录                                                            │  │
│  │     const booking = await tx.booking.create(createBookingObj);                 │  │
│  │     // ├── Booking 表                                                           │  │
│  │     // ├── Attendee 表（createMany）                                            │  │
│  │     // └── 状态: ACCEPTED 或 PENDING                                           │  │
│  │                                                                                  │  │
│  │     return booking;                                                             │  │
│  │  });                                                                            │  │
│  │                                                                                  │  │
│  │  ✅ 事务提交成功                                                                 │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│           │                                                                              │
│           ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 步骤 2: 外部日历写入（非事务性）                                                   │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  eventManager.create(evt)                                                       │  │
│  │           │                                                                     │  │
│  │           ├──► 8a. 创建视频会议（Daily/Zoom/Teams）                            │  │
│  │           │                                                                     │  │
│  │           ├──► 8b. 创建日历事件（Google/Outlook/Apple）                        │  │
│  │           │                                                                     │  │
│  │           └──► 8c. 创建 CRM 事件（Salesforce/HubSpot）                         │  │
│  │                                                                                  │  │
│  │  返回: { results, referencesToCreate }                                          │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│           │                                                                              │
│           ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 步骤 3: 保存外部引用（非事务性）                                                   │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  // 即时确认模式: 后续单独更新                                                    │  │
│  │  prisma.booking.update({                                                        │  │
│  │     data: {                                                                     │  │
│  │        references: { create: referencesToCreate },  // BookingReference 表      │  │
│  │        iCalUID: evt.iCalUID || booking.iCalUID,                               │  │
│  │     }                                                                           │  │
│  │  });                                                                            │  │
│  │                                                                                  │  │
│  │  // 需要确认模式: 与状态更新一起                                                 │  │
│  │  prisma.booking.update({                                                        │  │
│  │     data: {                                                                     │  │
│  │        status: ACCEPTED,                                                        │  │
│  │        references: { create: referencesToCreate },                             │  │
│  │     }                                                                           │  │
│  │  });                                                                            │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
│  ✅ 全部成功                                                                              │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.1.2 时序图：失败路径（建库成功，外部写入失败）

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    建库成功 + 外部写入失败（问题路径）                                   │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  RegularBookingService.createBooking()                                                  │
│           │                                                                              │
│           ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 步骤 1: 数据库事务（✅ 成功提交）                                                   │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  prisma.$transaction(...) → 成功                                                │  │
│  │                                                                                  │  │
│  │  Booking 表: 记录已创建，status = ACCEPTED                                       │  │
│  │  Attendee 表: 参与者已创建                                                        │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│           │                                                                              │
│           ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 步骤 2: 外部日历写入（❌ 失败）                                                    │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  eventManager.create(evt)                                                       │  │
│  │           │                                                                     │  │
│  │           ├──► 8a. 创建视频会议 → ❌ 失败（如 API 超时）                        │  │
│  │           │                                                                     │  │
│  │           ├──► 8b. 创建日历事件 → ❌ 失败（如 Google Calendar 不可用）          │  │
│  │           │                                                                     │  │
│  │           └──► 8c. 创建 CRM 事件 → 可能成功或失败                               │  │
│  │                                                                                  │  │
│  │  返回: {                                                                         │  │
│  │    results: [                                                                   │  │
│  │      { type: "daily_video", success: false, error: {...} },                    │  │
│  │      { type: "google_calendar", success: false, error: {...} },               │  │
│  │      { type: "hubspot_crm", success: true, ... },                              │  │
│  │    ],                                                                           │  │
│  │    referencesToCreate: [  // 只包含成功的                                       │  │
│  │      { type: "hubspot_crm", uid: "abc123", ... }                              │  │
│  │    ]                                                                             │  │
│  │  }                                                                              │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│           │                                                                              │
│           ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 步骤 3: 结果检查（⚠️ 关键决策点）                                                 │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  代码位置: RegularBookingService.ts:2064-2074                                   │  │
│  │                                                                                  │  │
│  │  if (results.length > 0 && results.every((res) => !res.success)) {              │  │
│  │     // 全部失败                                                                  │  │
│  │     const error = {                                                              │  │
│  │       errorCode: "BookingCreatingMeetingFailed",                                 │  │
│  │       message: "Booking failed",                                                │  │
│  │     };                                                                           │  │
│  │                                                                                  │  │
│  │     tracingLogger.error(                                                         │  │
│  │       `EventManager.create failure in some of the integrations`,                │  │
│  │       safeStringify({ error, results })                                         │  │
│  │     );                                                                           │  │
│  │     // ⚠️ 注意：只记录日志，不回滚，不补偿                                       │  │
│  │  } else {                                                                        │  │
│  │     // 部分成功或全部成功                                                        │  │
│  │     // 保存成功的 referencesToCreate                                             │  │
│  │     // 继续执行后续流程（邮件、Webhook等）                                        │  │
│  │  }                                                                              │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│           │                                                                              │
│           ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 步骤 4: 保存成功的引用（如果有）                                                   │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  // 如果有成功的 CRM 事件，保存其引用                                            │  │
│  │  prisma.booking.update({                                                        │  │
│  │     data: {                                                                     │  │
│  │        references: { create: referencesToCreate },  // 只包含 hubspot_crm      │  │
│  │     }                                                                           │  │
│  │  });                                                                            │  │
│  │                                                                                  │  │
│  │  // 视频会议和日历事件的引用丢失                                                 │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│           │                                                                              │
│           ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 最终状态                                                                           │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  Booking 表: ✅ 存在，status = ACCEPTED                                          │  │
│  │  Attendee 表: ✅ 存在                                                             │  │
│  │                                                                                  │  │
│  │  外部服务状态:                                                                    │  │
│  │    - Google Calendar: ❌ 无事件                                                   │  │
│  │    - Daily.co: ❌ 无会议                                                         │  │
│  │    - HubSpot: ✅ 有事件                                                          │  │
│  │                                                                                  │  │
│  │  BookingReference 表:                                                            │  │
│  │    - ✅ hubspot_crm 引用                                                         │  │
│  │    - ❌ google_calendar 引用（缺失）                                             │  │
│  │    - ❌ daily_video 引用（缺失）                                                 │  │
│  │                                                                                  │  │
│  │  ⚠️ 数据不一致！                                                                 │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 关键代码分析：外部写入失败后的处理

#### 2.2.1 全部失败场景

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts:2064-2074`

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
  
  // ⚠️ 关键：这里只有日志记录
  // - 没有回滚数据库操作
  // - 没有补偿机制
  // - 没有重试逻辑
}
```

**分析**：

| 预期行为 | 实际行为 |
|---------|---------|
| 回滚数据库事务 | ❌ 无 |
| 删除已创建的成功引用 | ❌ 无（但此时 referencesToCreate 为空） |
| 重试外部调用 | ❌ 无 |
| 标记预约为异常状态 | ❌ 无（保持 ACCEPTED） |
| 通知用户/组织者 | ❌ 无（只有日志） |

#### 2.2.2 部分失败场景

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts:2074-2149`

```typescript
} else {
  // 部分成功或全部成功
  const additionalInformation: AdditionalInformation = {};

  if (results.length) {
    // 处理 Google Meet 结果
    // ...
    
    // 保存成功的引用
    additionalInformation.hangoutLink = results[0].createdEvent?.hangoutLink;
    additionalInformation.conferenceData = results[0].createdEvent?.conferenceData;
    additionalInformation.entryPoints = results[0].createdEvent?.entryPoints;
    evt.appsStatus = handleAppsStatus(results, booking, reqAppsStatus);
    
    // 更新 iCalUID（如果外部日历返回了不同的）
    if (!isDryRun && evt.iCalUID !== booking.iCalUID) {
      await deps.prismaClient.booking.update({
        where: { id: booking.id },
        data: {
          iCalUID: evt.iCalUID || booking.iCalUID,
        },
      });
    }
  }
  
  // 发送邮件
  // 触发 Webhook
  // ...
}
```

**分析**：

| 项目 | 行为 |
|------|------|
| 成功的引用 | ✅ 保存到 BookingReference 表 |
| 失败的引用 | ❌ 不保存，只记录日志 |
| 重试失败的 | ❌ 无重试 |
| 标记部分失败 | ❌ 无特殊标记 |
| `appsStatus` | ✅ 记录各集成的成功/失败状态 |

**`handleAppsStatus` 作用**：将各集成的成功/失败状态记录到 `evt.appsStatus`，用于邮件通知和日志。

#### 2.2.3 EventManager 内部的部分失败处理

**代码位置**：`packages/features/bookings/lib/EventManager.ts:792-850`（deleteEventsAndMeetings）

```typescript
public async deleteEventsAndMeetings({/*...*/}) {
  // ...
  
  // 使用 Promise.allSettled 确保部分失败不影响整体
  (await Promise.allSettled(allPromises)).some((result) => {
    if (result.status === "rejected") {
      // 只记录警告日志
      log.warn(
        "Error deleting calendar event or video meeting for booking",
        safeStringify({ error: result.reason })
      );
    }
  });
  
  // ⚠️ 没有重试，没有补偿
}
```

**设计意图**：
- `deleteEventsAndMeetings` 用于 reschedule/cancel 场景
- 使用 `allSettled` 确保即使某些外部服务不可用，也不阻塞整体流程
- 但这也意味着失败的删除操作没有任何补偿

### 2.3 补偿机制分析

#### 2.3.1 只有在特定场景下才有主动补偿

**补偿机制触发条件**：

| 场景 | 是否有补偿 | 补偿动作 |
|------|-----------|---------|
| **建库成功，外部写入失败** | ❌ 无 | 无 |
| **reschedule 场景** | ✅ 有 | 删除旧的外部事件 |
| **cancel 场景** | ✅ 有 | 删除外部事件 |

#### 2.3.2 reschedule 场景的补偿机制

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts:1868-1902`

```typescript
// 重新安排时，先删除旧的外部事件
if (!skipCalendarSyncTaskCreation) {
  await originalHostEventManager.deleteEventsAndMeetings({
    event: deletionEvent,
    bookingReferences: originalRescheduledBooking.references,
  });
}
```

**流程**：

```
reschedule 请求
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ 步骤 1: 删除旧的外部事件（补偿动作）                            │
├──────────────────────────────────────────────────────────────┤
│  originalHostEventManager.deleteEventsAndMeetings()          │
│           │                                                   │
│           ├──► 删除日历事件（基于 bookingReferences）          │
│           ├──► 删除视频会议（基于 bookingReferences）          │
│           └──► 删除 CRM 事件（基于 bookingReferences）        │
│                                                               │
│ 使用 Promise.allSettled，部分失败不影响整体                    │
└──────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│ 步骤 2: 创建新的预约记录和外部事件                              │
├──────────────────────────────────────────────────────────────┤
│  // 数据库事务                                                 │
│  prisma.$transaction({                                        │
│    // 标记旧预约为 CANCELLED                                   │
│    // 创建新预约                                               │
│  })                                                           │
│                                                               │
│  // 外部日历写入                                               │
│  eventManager.create()                                        │
└──────────────────────────────────────────────────────────────┘
```

**关键问题**：
- 如果步骤 1（删除旧事件）部分失败，失败的事件不会被重试
- 如果步骤 2（创建新事件）失败，旧事件已经被删除（或部分删除），没有回滚机制

#### 2.3.3 cancel 场景的补偿机制

类似地，取消预约时会调用 `deleteEventsAndMeetings` 删除外部事件，但同样：
- 使用 `allSettled`
- 部分失败只记录日志
- 无重试

### 2.4 重试机制分析

#### 2.4.1 没有主动重试机制

**搜索整个代码库**：

```
搜索结果：
- 无 `retry(` 函数调用
- 无 `for (let i = 0; i < maxRetries; i++)` 模式
- 无 `while (!success && retries < maxRetries)` 模式
- 无任何重试库（如 p-retry）的使用
```

**唯一的"重试"迹象**：凭证刷新机制

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts:1830-1831`

```typescript
// After polling videoBusyTimes, credentials might have been changed due to refreshment, 
// so query them again.
const credentials = await refreshCredentials(allCredentials);
```

**分析**：
- 这是**凭证刷新**，不是操作重试
- 如果 OAuth token 过期，会刷新后重新获取凭证
- 但如果外部 API 调用本身失败（如超时、限流），不会重试

#### 2.4.2 凭证刷新 vs 操作重试

| 机制 | 触发条件 | 效果 |
|------|---------|------|
| **凭证刷新** | OAuth token 可能过期 | 获取新凭证 |
| **操作重试** | API 调用失败（超时、限流） | 重新调用 API |

**当前状态**：
- ✅ 有凭证刷新机制
- ❌ 无操作重试机制

### 2.5 责任边界分析

#### 2.5.1 各组件的责任边界

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         责任边界分析                                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 数据库层 (Prisma)                                                                 │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  责任范围:                                                                        │  │
│  │  - Booking/Attendee/BookingReference 表的 CRUD 操作                              │  │
│  │  - 本地事务的原子性保证                                                          │  │
│  │                                                                                  │  │
│  │  失败处理:                                                                        │  │
│  │  - 事务内操作失败 → 自动回滚 ✅                                                   │  │
│  │                                                                                  │  │
│  │  责任边界: 不关心外部服务，只关心本地数据一致性                                   │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 业务服务层 (RegularBookingService)                                                │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  责任范围:                                                                        │  │
│  │  - 协调数据库操作和外部服务调用                                                   │  │
│  │  - 业务流程编排                                                                  │  │
│  │                                                                                  │  │
│  │  失败处理:                                                                        │  │
│  │  - 检查 EventManager 结果                                                        │  │
│  │  - 只记录日志，不回滚，不补偿 ❌                                                  │  │
│  │                                                                                  │  │
│  │  责任边界: 实际上没有承担"最终一致性"责任                                         │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 外部集成层 (EventManager)                                                         │  │
│  ├──────────────────────────────────────────────────────────────────────────────────┤  │
│  │                                                                                  │  │
│  │  责任范围:                                                                        │  │
│  │  - 调用外部日历/视频/CRM API                                                      │  │
│  │  - 返回各集成的成功/失败状态                                                      │  │
│  │                                                                                  │  │
│  │  失败处理:                                                                        │  │
│  │  - 使用 Promise.allSettled，部分失败不影响整体                                   │  │
│  │  - 不重试，不补偿，只返回结果 ❌                                                  │  │
│  │                                                                                  │  │
│  │  责任边界: 只负责调用和返回结果，不负责一致性                                     │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 2.5.2 失败场景的责任归属

| 失败场景 | 负责组件 | 实际处理 | 理想处理 |
|---------|---------|---------|---------|
| 数据库事务内失败 | Prisma | ✅ 自动回滚 | 已满足 |
| 数据库事务提交后，外部写入失败 | RegularBookingService | ❌ 只记录日志 | 应补偿或标记异常 |
| 外部调用部分失败 | EventManager | ❌ 只返回失败状态 | 应重试或补偿 |
| reschedule 时删除旧事件失败 | EventManager | ❌ 只记录日志 | 应重试或标记 |

### 2.6 数据不一致的检测与修复

#### 2.6.1 检测不一致的可能方式

**当前系统没有主动检测机制**，但理论上可以：

| 检测方式 | 可行性 | 说明 |
|---------|-------|------|
| 比较 Booking.status 和 BookingReference | ⚠️ 部分可行 | 但部分成功是允许的 |
| 检查 appsStatus 字段 | ✅ 可行 | `evt.appsStatus` 记录了各集成状态 |
| 定期与外部日历同步对比 | ⚠️ 复杂 | 需要遍历所有预约 |

**`appsStatus` 数据结构**（来自 `handleAppsStatus` 函数）：

```typescript
{
  google_calendar: {
    success: boolean,
    error?: string,
    type: "calendar",
    label: "Google Calendar"
  },
  daily_video: {
    success: boolean,
    type: "video",
    label: "Daily.co"
  },
  hubspot_crm: {
    success: boolean,
    type: "crm",
    label: "HubSpot"
  }
}
```

#### 2.6.2 当前系统的"修复"方式

**唯一的修复途径：人工介入**

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         不一致检测与修复流程（理论）                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  1. 日志检测                                                                             │
│     ┌──────────────────────────────────────────────────────────────────────────────┐  │
│     │ tracingLogger.error(                                                         │  │
│     │   `EventManager.create failure in some of the integrations`,                │  │
│     │   { errorCode: "BookingCreatingMeetingFailed", results: [...] }            │  │
│     │ );                                                                           │  │
│     └──────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                          │
│                              ▼                                                          │
│  2. 人工发现                                                                             │
│     - 运维人员查看监控/日志                                                               │
│     - 用户反馈"日历中没有事件"                                                            │
│                              │                                                          │
│                              ▼                                                          │
│  3. 人工修复                                                                             │
│     ┌──────────────────────────────────────────────────────────────────────────────┐  │
│     │ 方式 A: 重新安排预约                                                           │  │
│     │         - 用户/组织者取消并重新创建                                            │  │
│     │         - 这会触发 deleteEventsAndMeetings + create                           │  │
│     │                                                                               │  │
│     │ 方式 B: 管理后台操作（如果有）                                                 │  │
│     │         - 手动触发日历同步                                                     │  │
│     │                                                                               │  │
│     │ 方式 C: 数据库直接操作（不推荐）                                               │  │
│     │         - 手动修改 booking.references                                          │  │
│     │         - 但无法修复外部日历中的缺失事件                                        │  │
│     └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.7 补偿与重试机制总结

#### 2.7.1 关键结论

| 问题 | 答案 |
|------|------|
| 建库成功但外部写入失败后，有补偿吗？ | ❌ **无主动补偿** |
| 有重试机制吗？ | ❌ **无主动重试**（只有凭证刷新） |
| 失败会回滚吗？ | ❌ **不会回滚**（数据库事务已提交） |
| 预约状态会变化吗？ | ❌ **保持 ACCEPTED**（只有日志） |
| 用户会被通知吗？ | ❌ **不会**（只有日志） |

#### 2.7.2 失败场景的最终状态

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    建库成功 + 外部写入失败的最终状态                                     │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  Cal.diy 数据库状态:                                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ Booking 表:                                                                       │  │
│  │   - status: ACCEPTED ✅                                                           │  │
│  │   - idempotencyKey: 存储但无用                                                    │  │
│  │   - metadata: 可能包含 appsStatus（各集成状态）                                   │  │
│  │                                                                                  │  │
│  │ BookingReference 表:                                                              │  │
│  │   - 只包含成功的外部服务引用                                                       │  │
│  │   - 失败的服务无引用                                                              │  │
│  │                                                                                  │  │
│  │ 日志:                                                                             │  │
│  │   - tracingLogger.error 记录了失败详情                                           │  │
│  │   - 包含 errorCode 和 results 数组                                               │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
│  外部服务状态:                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ Google Calendar: ❌ 无事件                                                        │  │
│  │ Daily.co: ❌ 无会议                                                               │  │
│  │ HubSpot: ✅ 有事件（如果成功）                                                    │  │
│  │                                                                                  │  │
│  │ ⚠️ 组织者和访客的日历中都没有这个预约                                             │  │
│  │ ⚠️ 但 Cal.diy 中显示为"已接受"                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
│  用户体验:                                                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 访客: 收到确认邮件，但日历中没有事件                                               │  │
│  │ 组织者: 收到确认邮件，但日历中没有事件                                             │  │
│  │                                                                                  │  │
│  │ 可能的困惑:                                                                       │  │
│  │ - "邮件说预约成功了，为什么日历里没有？"                                          │  │
│  │ - "我需要手动添加到日历吗？"                                                       │  │
│  │ - "这个预约到底有没有确认？"                                                       │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 前端触发入口与链路关联

### 3.1 两条链路的前端触发入口

#### 3.1.1 主预约创建链路

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         主预约创建链路的触发入口                                         │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  前端组件层                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                                  │  │
│  │  入口 1: Booker 组件（预约表单）                                                  │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 位置: apps/web/modules/bookings/components/Booker.tsx                   │   │  │
│  │  │ 职责: 收集用户输入（时间、姓名、邮箱、自定义字段）                            │   │  │
│  │  │ 触发: 用户点击"确认预约"按钮                                               │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                              │                                                   │  │
│  │                              ▼                                                   │  │
│  │  入口 2: createBooking 函数                                                     │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 位置: packages/features/bookings/lib/create-booking.ts                   │   │  │
│  │  │ 职责: 封装 API 调用                                                        │   │  │
│  │  │ 代码:                                                                     │   │  │
│  │  │ export const createBooking = async (data: BookingCreateBody) => {       │   │  │
│  │  │   const response = await post</*...*/>("/api/book/event", data);         │   │  │
│  │  │   return response;                                                        │   │  │
│  │  │ };                                                                         │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                              │                                                   │  │
│  │                              ▼                                                   │  │
│  │  入口 3: API 端点                                                               │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 普通预约: POST /api/book/event                                            │   │  │
│  │  │ 位置: apps/web/pages/api/book/event.ts                                    │   │  │
│  │  │                                                                           │   │  │
│  │  │ 重复预约: POST /api/book/recurring-event                                  │   │  │
│  │  │ 位置: apps/web/pages/api/book/recurring-event.ts                          │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                              │                                                   │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                          │
│                              ▼                                                          │
│  后端服务层                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                                  │  │
│  │  RegularBookingService.createBooking()                                           │  │
│  │  位置: packages/features/bookings/lib/service/RegularBookingService.ts          │  │
│  │                                                                                  │  │
│  │  RecurringBookingService.createBooking()                                         │  │
│  │  位置: packages/features/bookings/lib/service/RecurringBookingService.ts        │  │
│  │  职责: 循环调用 RegularBookingService 创建每个重复预约                           │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 3.1.2 确认流程入口（需要确认模式）

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         确认流程的触发入口                                               │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  前端组件层                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                                  │  │
│  │  入口 1: AcceptBookingButton 组件                                                │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 位置: apps/web/components/booking/AcceptBookingButton.tsx                │   │  │
│  │  │ 职责: 组织者点击"接受"按钮                                                 │   │  │
│  │  │ 触发: 组织者在预约详情页点击确认                                            │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                              │                                                   │  │
│  │                              ▼                                                   │  │
│  │  入口 2: useBookingConfirmation Hook                                            │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 位置: apps/web/components/booking/hooks/useBookingConfirmation.tsx        │   │  │
│  │  │ 职责: 封装 tRPC 调用逻辑                                                   │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                              │                                                   │  │
│  │                              ▼                                                   │  │
│  │  入口 3: tRPC Mutation                                                          │  │
│  │  ┌─────────────────────────────────────────────────────────────────────────┐   │  │
│  │  │ 路由: bookings.confirm                                                    │   │  │
│  │  │ 位置: packages/trpc/server/routers/viewer/bookings/_router.ts             │   │  │
│  │  │ 处理: packages/trpc/server/routers/viewer/bookings/confirm.handler.ts     │   │  │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │  │
│  │                              │                                                   │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                          │
│                              ▼                                                          │
│  后端服务层                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                                                                  │  │
│  │  handleConfirmation()                                                            │  │
│  │  位置: packages/features/bookings/lib/handleConfirmation.ts                     │  │
│  │                                                                                  │  │
│  │  职责:                                                                           │  │
│  │  - 调用 EventManager.create() 创建外部日历事件                                   │  │
│  │  - 更新 Booking.status 为 ACCEPTED                                              │  │
│  │  - 保存 BookingReference                                                         │  │
│  │  - 发送确认邮件                                                                 │  │
│  │                                                                                  │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 两条链路的幂等机制对比

| 维度 | 主预约创建链路 | 外部同步链路 |
|------|--------------|-------------|
| **触发源** | 访客主动提交 | 外部日历事件变更 |
| **入口组件** | Booker → createBooking → API | CalendarSyncService.handleEvents |
| **idempotencyKey** | ❌ 值为 null，无作用 | ✅ 生成并传递，但无查询 |
| **实际幂等机制** | uid（数据库主键） | iCalUID 解析 + hasStartTimeChanged |
| **循环防护** | 无（因 Date.now()） | skipCalendarSyncTaskCreation |

### 3.3 两条链路的外部写入失败处理对比

| 维度 | 主预约创建链路 | 外部同步链路 |
|------|--------------|-------------|
| **失败检查** | `results.every(res => !res.success)` | 无（因 skipCalendarSyncTaskCreation） |
| **日志记录** | ✅ tracingLogger.error | 无（EventManager 为空实现） |
| **补偿机制** | ❌ 无 | ❌ 无（不调用外部服务） |
| **预约状态** | ❌ 保持 ACCEPTED | 不涉及（只修改时间） |
| **用户通知** | ❌ 无 | 无 |

---

## 4. 精确结论与关键代码位置

### 4.1 关于幂等防重复的精确结论

#### 结论 1：`idempotencyKey` 字段**没有实际幂等作用**

| 链路 | 生成状态 | 查询状态 | 实际作用 |
|------|---------|---------|---------|
| 主预约创建链路 | ❌ 硬编码为 `null` | ❌ 无查询 | **无任何作用** |
| 外部同步链路 | ✅ 生成 UUID v5 | ❌ 无查询 | **只是数据字段** |

**证据**：
- 搜索整个代码库，**无任何** `findFirst.*idempotencyKey` 或 `WHERE.*idempotencyKey`
- 只有 `ManagedEventReassignment` 场景显式设置并存储，但也不用于查询

**代码位置**：
- 主链路硬编码 null: `RegularBookingService.ts:204`
- 外部链路生成但不查询: `CalendarSyncService.ts:266-271`

#### 结论 2：主预约创建链路的**实际幂等机制**是数据库主键

| 机制 | 代码位置 | 效果 | 限制 |
|------|---------|------|------|
| `uid` | `RegularBookingService.ts:1267-1268` | 数据库唯一键 | 因 `Date.now()`，每次请求生成不同值 |
| `iCalUID` | `RegularBookingService.ts:1306-1309` | 外部日历标识 | 主链路中无重复检查 |

**关键问题**：如果用户快速点击"提交"两次，会创建**两个独立的预约记录**。

#### 结论 3：外部同步链路的**实际幂等机制**是三层防护

| 层级 | 机制 | 代码位置 | 效果 |
|------|------|---------|------|
| Layer 1 | `iCalUID` 解析 | `CalendarSyncService.ts:82, 157` | 找到对应预约 |
| Layer 2 | `hasStartTimeChanged()` | `CalendarSyncService.ts:190-196` | 时间未变化则跳过 |
| Layer 3 | `skipCalendarSyncTaskCreation` | `CalendarSyncService.ts:209` | 防止无限循环 |

**证据**：
- 只有 `@cal.com` 后缀的事件才会被处理: `CalendarSyncService.ts:46-48`
- 时间未变化则直接返回: `CalendarSyncService.ts:190-196`
- 使用空操作的 EventManager: `RegularBookingService.ts:1833-1836`

### 4.2 关于补偿与重试的精确结论

#### 结论 1：建库成功但外部写入失败后，**无任何主动补偿或重试**

| 预期行为 | 实际行为 | 代码位置 |
|---------|---------|---------|
| 回滚数据库事务 | ❌ 不回滚 | - |
| 重试外部调用 | ❌ 不重试 | - |
| 删除已创建的成功引用 | ❌ 不删除 | - |
| 标记预约为异常 | ❌ 保持 ACCEPTED | - |
| 通知用户/组织者 | ❌ 不通知 | - |
| 记录错误日志 | ✅ 记录 | `RegularBookingService.ts:2070-2073` |

**证据**：
```typescript
// RegularBookingService.ts:2064-2074
if (results.length > 0 && results.every((res) => !res.success)) {
  const error = {
    errorCode: "BookingCreatingMeetingFailed",
    message: "Booking failed",
  };
  tracingLogger.error(/*...*/);
  // ⚠️ 只有日志，没有其他操作
}
```

#### 结论 2：只有 **reschedule/cancel 场景**有主动补偿

| 场景 | 补偿动作 | 代码位置 |
