# Cal.diy 重复预约系列机制详解

本文档详细说明 cal.diy 中重复预约（Recurring Booking）的完整实现机制，包括前端配置、数据存储、批量生成、外部日历同步以及单条实例修改的决策逻辑。

---

## 一、前端配置到数据库存储

### 1.1 前端配置组件

重复预约规则通过 `EventRecurringTab` 组件进行配置：

- **包装组件**: `packages/platform/atoms/event-types/wrappers/EventRecurringTabPlatformWrapper.tsx`
- **核心组件**: `@calcom/features/eventtypes/components/tabs/recurring/EventRecurringTab`

### 1.2 重复规则数据结构

重复规则使用 RRule（iCalendar 重复规则）标准，定义在 `packages/prisma/zod-utils.ts:234-243`：

```typescript
export const recurringEventType = z
  .object({
    dtstart: z.date().optional(),      // 开始日期
    interval: z.number(),               // 间隔（如每2周=2）
    count: z.number(),                  // 重复次数
    freq: z.nativeEnum(Frequency),      // 频率类型
    until: z.date().optional(),         // 结束日期（可选）
    tzid: z.string().optional(),        // 时区
  })
  .nullable();
```

**频率类型（Frequency 枚举）**：
```typescript
export enum Frequency {
  YEARLY = 0,    // 每年
  MONTHLY = 1,   // 每月
  WEEKLY = 2,    // 每周（最常用）
  DAILY = 3,     // 每日
  HOURLY = 4,    // 每小时
  MINUTELY = 5,  // 每分钟
  SECONDLY = 6,  // 每秒
}
```

### 1.3 数据库存储模型

#### EventType 表（事件类型）

重复规则存储在 `EventType.recurringEvent` JSON 字段中：

```prisma
model EventType {
  // ... 其他字段
  recurringEvent  Json?  // 存储完整的重复规则
  // 示例值:
  // {
  //   "freq": 2,        // WEEKLY
  //   "interval": 1,    // 每周
  //   "count": 10,      // 共10次
  //   "dtstart": "2024-01-01T00:00:00.000Z",
  //   "tzid": "America/New_York"
  // }
}
```

#### Booking 表（预约实例）

每个重复预约实例通过 `recurringEventId` 关联到同一系列：

```prisma
model Booking {
  // ... 其他字段
  recurringEventId  String?  // 重复系列的唯一标识符
  // 同一系列的所有 Booking 共享相同的 recurringEventId
  
  @@index([recurringEventId])  // 索引，用于快速查询同一系列
}
```

#### BookingReference 表（外部日历引用）

存储与外部日历（Google/Apple/Office365）的关联：

```prisma
model BookingReference {
  // ... 其他字段
  thirdPartyRecurringEventId  String?  // 外部日历的重复事件 ID
  
  type          String   // 如 "google_calendar", "office365_calendar"
  uid           String   // 外部日历事件的唯一 ID
  externalCalendarId  String?  // 外部日历 ID
}
```

---

## 二、批量生成预约实例

### 2.1 创建流程概览

当用户预订一个重复事件时，系统会：
1. 根据重复规则计算所有日期
2. 为每个日期创建独立的 `Booking` 记录
3. 所有 Booking 共享同一个 `recurringEventId`

### 2.2 核心服务

#### RecurringBookingService

**位置**: `packages/features/bookings/lib/service/RecurringBookingService.ts`

核心方法 `handleNewRecurringBooking` 处理重复预约的创建：

```typescript
export const handleNewRecurringBooking = async function (
  this: RecurringBookingService,
  {
    input,
    deps,
    creationSource,
  }: {
    input: BookingHandlerInput;
    deps: IRecurringBookingServiceDependencies;
    creationSource: CreationSource;
  }
): Promise<BookingResponse[]> {
  const data = input.bookingData;  // 包含所有日期的数组
  const { regularBookingService } = deps;
  const createdBookings: BookingResponse[] = [];
  
  // 提取所有重复日期
  const allRecurringDates: { start: string; end: string | undefined }[] = 
    data.map((booking) => { return { start: booking.start, end: booking.end }; });
  
  let thirdPartyRecurringEventId = null;
  
  // 处理 Round Robin 场景（第一个 slot 决定 lucky user）
  const firstBooking = data[0];
  const isRoundRobin = firstBooking.schedulingType === SchedulingType.ROUND_ROBIN;
  
  let luckyUsers;
  
  // Round Robin 场景：第一个 slot 需要特殊处理
  if (isRoundRobin) {
    const recurringEventData = {
      ...firstBooking,
      allRecurringDates,
      isFirstRecurringSlot: true,
      // ...
    };
    const firstBookingResult = await regularBookingService.createBooking({
      bookingData: recurringEventData,
      // ...
    });
    luckyUsers = firstBookingResult.luckyUsers;
  }
  
  // 遍历所有日期，创建预约
  for (let key = isRoundRobin ? 1 : 0; key < data.length; key++) {
    const booking = data[key];
    
    const recurringEventData = {
      ...booking,
      allRecurringDates,
      isFirstRecurringSlot: key == 0,
      thirdPartyRecurringEventId,  // 传递外部日历的重复事件 ID
      currentRecurringIndex: key,   // 当前索引
      luckyUsers,                    // Round Robin 的幸运用户
      // ...
    };
    
    const eachRecurringBooking = await regularBookingService.createBooking({
      bookingData: recurringEventData,
      // ...
    });
    
    createdBookings.push(eachRecurringBooking);
    
    // 捕获第一个成功的外部日历重复事件 ID
    if (!thirdPartyRecurringEventId) {
      if (eachRecurringBooking.references && 
          eachRecurringBooking.references.length > 0) {
        for (const reference of eachRecurringBooking.references) {
          if (reference.thirdPartyRecurringEventId) {
            thirdPartyRecurringEventId = reference.thirdPartyRecurringEventId;
            break;
          }
        }
      }
    }
  }
  
  return createdBookings;
}
```

### 2.3 前端创建 Hook

**位置**: `packages/platform/atoms/hooks/bookings/useCreateRecurringBooking.ts`

```typescript
export const useCreateRecurringBooking = (
  { onSuccess, onError }: IUseCreateRecurringBooking = { ... }
) => {
  const createRecurringBooking = useMutation<
    ApiResponse<BookingResponse[]>,
    Error,
    RecurringBookingCreateBody[]
  >({
    mutationFn: (data) => {
      return http.post<ApiResponse<BookingResponse[]>>(
        "/bookings/recurring", data
      ).then((res) => res.data);
    },
    // ...
  });
  return createRecurringBooking;
};
```

---

## 三、外部日历双向同步协作

### 3.1 同步架构

Cal.diy 通过 `EventManager` 管理与外部日历的双向同步：

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│   Cal.diy       │         │   EventManager  │         │  外部日历        │
│  Booking        │◄───────►│  (创建/更新/    │◄───────►│  Google/        │
│  BookingRef     │         │   删除)         │         │  Apple/         │
│                │         │                 │         │  Office365      │
└─────────────────┘         └─────────────────┘         └─────────────────┘
```

### 3.2 EventManager 核心功能

**位置**: `packages/features/bookings/lib/EventManager.ts`

#### 创建日历事件

```typescript
public async create(
  event: CalendarEvent,
  options?: { skipCalendarEvent?: boolean }
): Promise<CreateUpdateResult> {
  // 1. 处理视频会议集成（如 Google Meet, Zoom）
  const isDedicated = evt.location ? isDedicatedIntegration(evt.location) : null;
  if (isDedicated) {
    const result = await this.createVideoEvent(evt);
    // ...
  }
  
  // 2. 创建日历事件
  const clonedCalEvent = cloneDeep(event);
  if (!skipCalendarEvent) {
    results.push(...(await this.createAllCalendarEvents(clonedCalEvent)));
  }
  
  // 3. 创建 CRM 事件
  const createdCRMEvents = skipCalendarEvent ? [] : await this.createAllCRMEvents(evt);
  
  // 4. 构建引用（包含 thirdPartyRecurringEventId）
  const referencesToCreate = results.map((result) => {
    let thirdPartyRecurringEventId;
    // ...
    const isCalendarType = isCalendarResult(result);
    if (isCalendarType) {
      evt.iCalUID = result.iCalUID || event.iCalUID || undefined;
      thirdPartyRecurringEventId = result.createdEvent?.thirdPartyRecurringEventId;
    }
    
    return {
      type: result.type,
      uid: createdEventObj ? createdEventObj.id : (result.createdEvent?.id?.toString() ?? ""),
      thirdPartyRecurringEventId: isCalendarType ? thirdPartyRecurringEventId : undefined,
      externalCalendarId: isCalendarType ? result.externalId : undefined,
      // ...
    };
  });
  
  return { results, referencesToCreate };
}
```

#### 删除日历事件（处理重复系列）

```typescript
private async deleteCalendarEventForBookingReference({
  reference,
  event,
  isBookingInRecurringSeries,
}: {
  reference: PartialReference;
  event: CalendarEvent;
  isBookingInRecurringSeries?: boolean;
}) {
  // 关键决策：如果是重复系列且有 thirdPartyRecurringEventId，
  // 使用该 ID 删除整个系列，否则只删除单个实例
  const bookingRefUid =
    isBookingInRecurringSeries && reference?.thirdPartyRecurringEventId
      ? reference.thirdPartyRecurringEventId
      : reference.uid;
  
  const calendarCredential = await this.getCalendarCredential(
    credentialId, credentialType, delegationCredentialId
  );
  
  if (calendarCredential) {
    await deleteEvent({
      credential: calendarCredential,
      bookingRefUid,  // 根据 isBookingInRecurringSeries 决定是单个还是系列
      event,
      externalCalendarId: bookingExternalCalendarId,
    });
  }
}
```

#### 取消事件入口

```typescript
public async cancelEvent(
  event: CalendarEvent,
  bookingReferences: Pick<
    BookingReference,
    "uid" | "type" | "externalCalendarId" | "credentialId" | "thirdPartyRecurringEventId"
  >[],
  isBookingInRecurringSeries?: boolean
) {
  await this.deleteEventsAndMeetings({
    event,
    bookingReferences,
    isBookingInRecurringSeries,  // 传递此参数决定删除范围
  });
}
```

### 3.3 外部日历回写机制（外部日历 → Cal.diy）

当用户在外部日历（Google/Apple/Office365）中直接修改或取消事件时，系统通过 `CalendarSyncService` 处理这些回写操作。

**位置**: `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts`

#### 事件筛选与状态判断

系统只处理 Cal.diy 创建的事件，通过 `iCalUID` 的后缀判断：

```typescript
async handleEvents(
  selectedCalendar: SelectedCalendar,
  calendarSubscriptionEvents: CalendarSubscriptionEventItem[]
) {
  // 只处理 Cal.com 创建的日历事件
  const calEvents = calendarSubscriptionEvents.filter((e) =>
    e.iCalUID?.toLowerCase()?.endsWith("@cal.com")
  );

  if (calEvents.length === 0) {
    log.debug("handleEvents: no calendar events to process");
    return;
  }

  await Promise.all(
    calEvents.map((e) => {
      // 关键判断：根据外部事件状态决定操作类型
      if (e.status === "cancelled") {
        // 状态为 cancelled → 触发取消操作
        return this.cancelBooking(e, selectedCalendar.userId);
      } else {
        // 其他状态 → 检查是否需要改期
        return this.rescheduleBooking(e, selectedCalendar.userId);
      }
    })
  );
}
```

#### 触发取消的条件

当外部日历事件的 `status === "cancelled"` 时，系统执行取消操作：

```typescript
async cancelBooking(event: CalendarSubscriptionEventItem, calendarUserId: number) {
  // 从 iCalUID 中提取 booking UID
  // 格式: {booking-uid}@cal.com
  const [bookingUid] = event.iCalUID?.split("@") ?? [undefined];
  if (!bookingUid) {
    log.debug("Unable to sync, booking UID not found in iCalUID");
    return;
  }

  // 查找对应的 Booking
  const booking = await this.deps.bookingRepository.findBookingByUidWithEventType({ bookingUid });
  if (!booking) {
    log.debug("Unable to sync, booking not found in database", { bookingUid });
    return;
  }

  // 权限检查：确保日历所有者是预约的组织者
  if (booking.userId !== calendarUserId) {
    log.debug("Skipping sync, calendar owner is not the booking host", {
      bookingUid,
      calendarUserId,
      bookingUserId: booking.userId,
    });
    return;
  }

  // 执行取消
  await handleCancelBooking({
    userId: booking.userId,
    bookingData: {
      uid: booking.uid,
      cancellationReason: "Cancelled on user's calendar",
      cancelledBy: booking.userPrimaryEmail,
      // 关键：跳过日历同步，防止回环
      skipCalendarSyncTaskCancellation: true,
    },
  });
}
```

#### 触发改期的条件

当外部日历事件状态不为 `cancelled` 时，系统检查时间是否变化，以决定是否执行改期：

```typescript
async rescheduleBooking(event: CalendarSubscriptionEventItem, calendarUserId: number) {
  const [bookingUid] = event.iCalUID?.split("@") ?? [undefined];
  if (!bookingUid) {
    log.debug("Unable to sync, booking UID not found in iCalUID");
    return;
  }

  const booking = await this.deps.bookingRepository.findBookingByUidWithEventType({ bookingUid });
  if (!booking) {
    log.debug("Unable to sync, booking not found in database", { bookingUid });
    return;
  }

  // 权限检查
  if (booking.userId !== calendarUserId) {
    log.debug("Skipping sync, calendar owner is not the booking host", {
      bookingUid,
      calendarUserId,
      bookingUserId: booking.userId,
    });
    return;
  }

  // 关键判断：只有当开始时间变化时才执行改期
  if (!hasStartTimeChanged(booking, event)) {
    log.debug("Skipping reschedule, start time has not changed", { bookingUid });
    return;
  }

  // 构建改期数据
  const rescheduleBookingData = buildRescheduleBookingData(booking, event);
  
  // 执行改期
  const { getRegularBookingService } = await import(
    "@calcom/features/bookings/di/RegularBookingService.container"
  );
  const regularBookingService = getRegularBookingService();
  await regularBookingService.createBooking({
    bookingData: rescheduleBookingData,
    bookingMeta: {
      // 关键：跳过日历同步，防止回环
      skipCalendarSyncTaskCreation: true,
      skipAvailabilityCheck: true,
      skipEventLimitsCheck: true,
    },
  });
}

// 辅助函数：检查开始时间是否变化
export const hasStartTimeChanged = (
  booking: BookingWithEventType,
  event: CalendarSubscriptionEventItem
): boolean => {
  if (!event.start) return false;
  return event.start.getTime() !== booking.startTime.getTime();
};
```

#### 回写触发条件汇总表

| 外部日历事件状态 | 系统操作 | 触发条件 |
|-----------------|---------|---------|
| `status === "cancelled"` | 取消预约 | 状态明确标记为取消 |
| `status !== "cancelled"` | 改期预约 | 开始时间发生变化（`startTime` 不同） |
| `status !== "cancelled"` | 无操作 | 开始时间未变化 |

---

### 3.4 同步回环防护机制

**问题描述**：如果不加以防护，会发生无限循环：
```
Cal.diy 创建/取消事件 → 同步到外部日历
    ↑                                         ↓
    ←────────── 外部日历 Webhook/轮询通知 ←──
           (再次触发 Cal.diy 操作)
```

#### 防护机制 1：`skipCalendarSyncTaskCancellation`

在外部日历回写取消时使用，阻止 `EventManager` 再次向外部日历发送取消请求：

**位置**: `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:119-129`

```typescript
await handleCancelBooking({
  userId: booking.userId,
  bookingData: {
    uid: booking.uid,
    cancellationReason: "Cancelled on user's calendar",
    cancelledBy: booking.userPrimaryEmail,
    // 关键：设置为 true，跳过向外部日历的同步
    skipCalendarSyncTaskCancellation: true,
  },
});
```

**在 handleCancelBooking 中的处理**：

**位置**: `packages/features/bookings/lib/handleCancelBooking.ts:425-463`

```typescript
// 只有当 skipCalendarSyncTaskCancellation 为 false 时才同步
if (!skipCalendarSyncTaskCancellation) {
  try {
    const eventManager = new EventManager({ ...bookingToDelete.user, credentials }, ...);
    await eventManager.cancelEvent(
      evt, 
      bookingToDelete.references, 
      isBookingInRecurringSeries
    );
  } catch (error) {
    log.error(`Error deleting integrations`, safeStringify({ error }));
  }
}
```

#### 防护机制 2：`skipCalendarSyncTaskCreation`

在外部日历回写改期时使用，阻止 `EventManager` 再次向外部日历发送创建请求：

**位置**: `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:204-213`

```typescript
await regularBookingService.createBooking({
  bookingData: buildRescheduleBookingData(booking, event),
  bookingMeta: {
    // 关键：设置为 true，跳过向外部日历的同步
    skipCalendarSyncTaskCreation: true,
    skipAvailabilityCheck: true,
    skipEventLimitsCheck: true,
  },
});
```

**在 RegularBookingService 中的处理**：

该参数会传递给 `EventManager`，阻止创建外部日历事件。

#### 完整的同步流程图（含回环防护）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         双向同步与回环防护                                     │
└─────────────────────────────────────────────────────────────────────────────┘

【方向 1：Cal.diy → 外部日历】

用户在 Cal.diy 操作
        │
        ▼
┌───────────────┐
│  createBooking │ 或 │ handleCancelBooking │
│               │
│  skipCalendarSync │
│  = false (默认)   │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ EventManager  │
│  .create()    │ 或 │ .cancelEvent() │
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  外部日历 API  │
│ (Google/Apple/│
│  Office365)   │
└───────────────┘


【方向 2：外部日历 → Cal.diy（含回环防护）】

用户在外部日历操作
        │
        ▼
┌───────────────────┐
│ Webhook 或轮询通知 │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ CalendarSyncService│
│ .handleEvents()   │
│                   │
│ 过滤条件：         │
│ - iCalUID 以      │
│   @cal.com 结尾   │
└─────────┬─────────┘
          │
    ┌─────┴─────┐
    │           │
    ▼           ▼
 status=   status!=
 cancelled  cancelled
    │           │
    ▼           ▼
 cancelBooking  检查时间
    │         是否变化
    │           │
    │      ┌────┴────┐
    │      │         │
    │    时间变化   时间不变
    │      │         │
    │      ▼         │
    │  reschedule    │
    │    Booking     │
    │      │         │
    ▼      ▼         │
┌─────────────────────────┐         │
│ 关键：设置跳过同步        │         │
│                         │         │
│ cancelBooking:          │         │
│   skipCalendarSyncTask  │         │
│   Cancellation = true   │         │
│                         │         │
│ rescheduleBooking:      │         │
│   skipCalendarSyncTask  │         │
│   Creation = true       │         │
└────────────┬────────────┘         │
             │                        │
             ▼                        │
    ┌─────────────────┐               │
    │ handleCancel    │               │
    │ Booking /       │               │
    │ createBooking   │               │
    │                 │               │
    │ 检测到 skip=    │               │
    │ true，跳过      │               │
    │ EventManager    │               │
    │ 外部同步        │               │
    └────────┬────────┘               │
             │                        │
             │                        │
             ▼                        ▼
        ┌─────────────────────────────────┐
        │   防止回环：不再向外部日历发送    │
        │   操作，循环在此终止             │
        └─────────────────────────────────┘
```

---

### 3.5 双向同步的关键机制（Cal.diy → 外部日历）

#### 创建时的同步

1. **Cal.diy → 外部日历**：
   - 创建 Booking 后，调用 `EventManager.create()`
   - 外部日历返回 `thirdPartyRecurringEventId`（对于重复事件）
   - 存储到 `BookingReference.thirdPartyRecurringEventId`

#### 取消时的同步

**位置**: `packages/features/bookings/lib/handleCancelBooking.ts:425-463`

```typescript
// 判断是否删除整个重复系列
const isBookingInRecurringSeries = !!(
  bookingToDelete.eventType?.recurringEvent &&
  bookingToDelete.recurringEventId &&
  allRemainingBookings  // 用户选择"取消所有剩余"
);

// 调用 EventManager 取消
if (!skipCalendarSyncTaskCancellation) {
  try {
    const eventManager = new EventManager({ ...bookingToDelete.user, credentials }, ...);
    await eventManager.cancelEvent(
      evt, 
      bookingToDelete.references, 
      isBookingInRecurringSeries  // 关键参数
    );
  } catch (error) {
    log.error(`Error deleting integrations`, safeStringify({ error }));
  }
}
```

---

## 四、单条实例修改决策逻辑

### 4.1 取消操作的决策参数

当用户取消一个重复预约时，系统根据以下参数决定操作范围：

| 参数 | 类型 | 作用 |
|------|------|------|
| `cancelSubsequentBookings` | boolean | 取消当前及**后续所有**预约 |
| `allRemainingBookings` | boolean | 取消**所有剩余**预约（包括当前之前的？需看具体实现） |

### 4.2 输入类型定义

**位置**: `packages/platform/types/bookings/2024-08-13/inputs/cancel-booking.input.ts`

```typescript
export class CancelBookingInput_2024_08_13 {
  @IsString()
  @IsOptional()
  @ApiPropertyOptional({ example: "User requested cancellation" })
  cancellationReason?: string;

  @IsBoolean()
  @IsOptional()
  @ApiPropertyOptional({
    description:
      "For recurring non-seated booking only - if true, cancel booking with the bookingUid of the individual recurrence and all recurrences that come after it.",
  })
  cancelSubsequentBookings?: boolean;
}
```

### 4.3 核心决策逻辑

**位置**: `packages/features/bookings/lib/handleCancelBooking.ts:346-415`

```typescript
// 解析输入参数
const {
  id,
  uid,
  allRemainingBookings,
  cancellationReason,
  seatReferenceUid,
  cancelledBy,
  cancelSubsequentBookings,
  // ...
} = bookingCancelInput.parse(body);

// 关键判断：是否批量取消
if (
  bookingToDelete.eventType?.recurringEvent &&
  bookingToDelete.recurringEventId &&
  (allRemainingBookings || cancelSubsequentBookings)
) {
  const recurringEventId = bookingToDelete.recurringEventId;
  
  // 确定时间范围：
  // - cancelSubsequentBookings: 从当前 booking 的 startTime 开始
  // - allRemainingBookings: 从现在开始
  const gte = cancelSubsequentBookings 
    ? bookingToDelete.startTime 
    : new Date();
  
  // 批量更新：标记所有符合条件的 booking 为 CANCELLED
  await bookingRepository.updateMany({
    where: {
      recurringEventId,
      startTime: {
        gte,  // 大于等于指定时间
      },
    },
    data: {
      status: BookingStatus.CANCELLED,
      cancellationReason: cancellationReason,
      cancelledBy: cancelledBy,
    },
  });
  
  // 获取所有被取消的 booking 用于后续处理（如同步外部日历）
  const allUpdatedBookings = await bookingRepository.findManyIncludeReferences({
    where: {
      recurringEventId: bookingToDelete.recurringEventId,
      startTime: {
        gte: new Date(),
      },
    },
  });
  updatedBookings = updatedBookings.concat(allUpdatedBookings);
  
} else {
  // 只取消当前单个 booking
  const updatedBooking = await bookingRepository.updateIncludeReferences({
    where: {
      uid: bookingToDelete.uid,
    },
    data: {
      status: BookingStatus.CANCELLED,
      cancellationReason: cancellationReason,
      cancelledBy: cancelledBy,
      iCalSequence: evt.iCalSequence || 100,
    },
  });
  updatedBookings.push(updatedBooking);
}
```

### 4.4 决策流程图

```
                    ┌─────────────────────────────┐
                    │   用户发起取消操作           │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │ 检查是否为重复预约？         │
                    │ (eventType.recurringEvent   │
                    │  && recurringEventId)       │
                    └──────────────┬──────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           ▼                       ▼                       ▼
    ┌──────────────┐      ┌──────────────────┐    ┌──────────────────┐
    │   否         │      │ cancelSubsequent │    │ allRemaining     │
    │ (普通预约)    │      │ Bookings=true    │    │ Bookings=true    │
    └──────┬───────┘      └────────┬─────────┘    └────────┬─────────┘
           │                       │                       │
           ▼                       ▼                       ▼
    ┌──────────────┐      ┌──────────────────┐    ┌──────────────────┐
    │ 仅取消当前    │      │ 取消当前及之后   │    │ 取消所有剩余     │
    │ Booking      │      │ 所有 Booking     │    │ (从现在开始)     │
    └──────────────┘      │ startTime >=     │    │ startTime >=     │
                          │ 当前 booking     │    │ new Date()       │
                          │ .startTime       │    └──────────────────┘
                          └──────────────────┘
```

### 4.5 脱离原系列（Detach from Series）

在某些场景下，用户可能希望修改单条实例但保持其他实例不变。这通过以下方式实现：

1. **修改单条实例**：
   - 不传递 `cancelSubsequentBookings` 或 `allRemainingBookings`
   - 只修改当前 `Booking` 记录
   - 该实例仍保留 `recurringEventId`，但数据已独立

2. **重新安排（Reschedule）**：
   - 创建新的 `Booking` 记录
   - 原 `Booking` 标记为 `rescheduled=true`
   - 新 `Booking` 可以选择是否继承 `recurringEventId`

**位置**: `packages/features/bookings/lib/handleCancelBooking.ts:1319-1338`（EventManager.reschedule）

```typescript
public async reschedule(
  event: CalendarEvent,
  rescheduleUid: string,
  newBookingId?: number,
  changedOrganizer?: boolean,
  previousHostDestinationCalendar?: DestinationCalendar[] | null,
  isBookingRequestedReschedule?: boolean,
  skipDeleteEventsAndMeetings?: boolean
): Promise<CreateUpdateResult> {
  // ...
  // 获取原有 booking
  const booking = await prisma.booking.findUnique({
    where: { uid: rescheduleUid },
    select: {
      // ...
      references: {
        where: { deleted: null },
        select: {
          type: true,
          uid: true,
          // ...
        },
      },
    },
  });
  
  // 决策：是更新原有事件还是创建新事件？
  if (changedOrganizer) {
    // 组织者变更：删除旧事件，创建新事件
    await this.deleteEventsAndMeetings({
      event: { ...event, destinationCalendar: previousHostDestinationCalendar },
      bookingReferences: booking.references,
    });
    const createdEvent = await this.create(originalEvt);
    results.push(...createdEvent.results);
    updatedBookingReferences.push(...createdEvent.referencesToCreate);
  } else {
    // 时间/地点变更：更新原有事件
    // ...
  }
}
```

---

### 4.6 改期后系列标识继承机制（深度解析）

当用户改期（reschedule）一个预约时，`recurringEventId` 的继承逻辑是理解系列归属的核心。本节将沿 `createBooking` 数据流详细解析继承优先级、触发条件，以及站内改期与外部日历回写改期两条路径的最终落库结果。

---

#### 4.6.1 核心数据流：从请求到落库

改期操作的完整数据流如下：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    createBooking 数据流与 recurringEventId 继承              │
└─────────────────────────────────────────────────────────────────────────────┘

【入口：RegularBookingService.createBooking()】

输入参数：
┌─────────────────────────────────────────────────────────────────────────────┐
│ bookingData: {                                                               │
│   rescheduleUid: "abc123",      // 关键：原预约 UID，存在则表示是改期       │
│   recurringEventId: "recur_xyz", // 可选：显式传递的系列 ID（优先级 1）      │
│   start: "2024-01-16T14:00:00Z",                                            │
│   end: "2024-01-16T14:30:00Z",                                              │
│   // ... 其他参数                                                             │
│ }                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
【步骤 1：判断是否为改期】

if (bookingData.rescheduleUid) {
  // 是改期操作
  const originalRescheduledBooking = 
    await getOriginalRescheduledBooking(rescheduleUid, seatsEventType);
  // 获取原预约的完整信息，包括 recurringEventId
} else {
  // 是新建预约
  originalRescheduledBooking = null;
}
        │
        ▼
【步骤 2：buildNewBookingData() - 核心继承逻辑】

function buildNewBookingData(params) {
  const { reqBody, originalRescheduledBooking, ... } = params;
  
  // 默认值
  let recurringEventId = null;
  
  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │ 优先级 1：请求参数显式传递 (reqBody.recurringEventId)                    │
  // └─────────────────────────────────────────────────────────────────────────┘
  // 触发条件：
  // - 重复预约批量创建时，RecurringBookingService 显式传递相同的 ID
  // - 所有 slot 共享同一个 recurringEventId
  
  if (reqBody.recurringEventId) {
    recurringEventId = reqBody.recurringEventId;
    console.log("优先级 1 生效：使用请求传递的 recurringEventId");
  }
  
  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │ 优先级 2：从原预约继承 (originalRescheduledBooking.recurringEventId)      │
  // └─────────────────────────────────────────────────────────────────────────┘
  // 触发条件：
  // - 存在 rescheduleUid（是改期操作）
  // - getOriginalRescheduledBooking() 返回的原预约有 recurringEventId
  // - 注意：这会覆盖优先级 1 的值（因为代码顺序在后面）
  
  if (originalRescheduledBooking) {
    // ... 其他属性继承（metadata, paid, fromReschedule 等）
    
    if (originalRescheduledBooking.recurringEventId) {
      recurringEventId = originalRescheduledBooking.recurringEventId;
      console.log("优先级 2 生效：继承原预约的 recurringEventId");
    }
  }
  
  // ┌─────────────────────────────────────────────────────────────────────────┐
  // │ 优先级 3：默认值 (null)                                                   │
  // └─────────────────────────────────────────────────────────────────────────┘
  // 触发条件：
  // - 不是改期（没有 rescheduleUid）
  // - 或者原预约没有 recurringEventId
  
  return {
    ...newBookingData,
    recurringEventId,  // 最终值
  };
}
        │
        ▼
【步骤 3：落库（Prisma 事务）】

prisma.$transaction(async (tx) => {
  // 3a：更新原预约（如果是改期）
  if (originalRescheduledBooking) {
    await tx.booking.update({
      where: { uid: originalRescheduledBooking.uid },
      data: {
        rescheduled: true,                    // 标记为已改期
        status: BookingStatus.CANCELLED,      // 状态改为已取消
        // 注意：recurringEventId 保持不变！
      },
    });
  }
  
  // 3b：创建新预约
  const newBooking = await tx.booking.create({
    data: {
      uid: newUid,
      recurringEventId: calculatedValue,  // 继承逻辑计算出的最终值
      fromReschedule: originalRescheduledBooking?.uid,
      // ... 其他字段
    },
  });
  
  return newBooking;
});
```

---

#### 4.6.2 继承优先级与触发条件详解

| 优先级 | 来源 | 触发条件 | 代码位置 | 覆盖关系 |
|-------|------|---------|---------|---------|
| **1** | `reqBody.recurringEventId` | 请求中显式传递 | `createBooking.ts:222-224` | 被优先级 2 覆盖 |
| **2** | `originalRescheduledBooking.recurringEventId` | 改期且原预约有系列 ID | `createBooking.ts:249-251` | 覆盖优先级 1 |
| **3** | 默认值 `null` | 以上都不满足 | `createBooking.ts:206` | 不覆盖 |

**关键代码顺序分析**：

```typescript
// 优先级 1：先检查请求参数
if (reqBody.recurringEventId) {
  newBookingData.recurringEventId = reqBody.recurringEventId;
}

// ... 中间代码 ...

// 优先级 2：后检查原预约（可能覆盖优先级 1 的值）
if (originalRescheduledBooking) {
  // ...
  
  if (originalRescheduledBooking.recurringEventId) {
    // 这里会覆盖之前设置的值！
    newBookingData.recurringEventId = originalRescheduledBooking.recurringEventId;
  }
}
```

**这意味着**：如果改期的原预约有 `recurringEventId`，即使请求中显式传递了不同的 `recurringEventId`，最终也会使用原预约的值。

---

#### 4.6.3 原预约信息的获取机制

`getOriginalRescheduledBooking` 函数负责查询原预约的完整信息：

**位置**: `packages/features/bookings/repositories/BookingRepository.ts:1170-1217`

```typescript
async findOriginalRescheduledBooking(uid: string, seatsEventType?: boolean) {
  return await this.prismaClient.booking.findFirst({
    where: {
      uid: uid,
      status: {
        in: [ACCEPTED, CANCELLED, PENDING],  // 可以改期的状态
      },
    },
    // 关键：没有使用 select 限制字段
    // 这意味着会返回 Booking 表的所有字段，包括：
    // - recurringEventId
    // - status, paid, metadata 等
    include: {
      attendees: { select: { ... } },
      user: { select: { ... } },
      eventType: { select: { ... } },
      destinationCalendar: true,
      payment: true,
      references: true,
    },
  });
}
```

**重要**：`findFirst` 没有使用 `select` 限制字段，所以 `recurringEventId` 会被自动包含在返回结果中。

---

#### 4.6.4 两条路径对比：站内改期 vs 外部日历回写改期

改期有两条不同的触发路径，但最终落库逻辑是相同的。

##### 路径 A：站内改期（用户在 Cal.diy UI 中操作）

**数据流**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         站内改期数据流                                        │
└─────────────────────────────────────────────────────────────────────────────┘

用户在 UI 中点击"改期"
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 前端构建请求参数                                                              │
│ {                                                                             │
│   rescheduleUid: "abc123",      // 原预约 UID                                │
│   start: "2024-01-16T14:00:00Z",                                            │
│   end: "2024-01-16T14:30:00Z",                                              │
│   eventTypeId: 123,                                                           │
│   // 注意：不传递 recurringEventId！让后端决定                                │
│ }                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ RegularBookingService.createBooking()                                        │
│                                                                               │
│ 1. 检测到 rescheduleUid 存在 → 是改期操作                                    │
│                                                                               │
│ 2. 调用 getOriginalRescheduledBooking("abc123")                             │
│    返回：                                                                      │
│    {                                                                          │
│      uid: "abc123",                                                           │
│      recurringEventId: "recur_xyz789",  // 原预约的系列 ID                   │
│      status: "ACCEPTED",                                                      │
│      // ... 其他字段                                                          │
│    }                                                                          │
│                                                                               │
│ 3. buildNewBookingData()                                                      │
│    - reqBody.recurringEventId: undefined（未传递）                           │
│    - originalRescheduledBooking.recurringEventId: "recur_xyz789"（存在）    │
│    → 最终 recurringEventId = "recur_xyz789"（继承）                          │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
【最终落库结果】

原预约（uid: "abc123"）：
┌─────────────────┬─────────────────┬─────────────────┐
│     字段        │    原值         │    改期后       │
├─────────────────┼─────────────────┼─────────────────┤
│ uid             │ "abc123"        │ "abc123"        │
│ recurringEventId│ "recur_xyz789"  │ "recur_xyz789"  │ ← 保持不变！
│ status          │ "ACCEPTED"      │ "CANCELLED"     │
│ rescheduled     │ false           │ true            │
└─────────────────┴─────────────────┴─────────────────┘

新预约（uid: "def456"）：
┌─────────────────┬─────────────────┐
│     字段        │      值         │
├─────────────────┼─────────────────┤
│ uid             │ "def456"        │
│ recurringEventId│ "recur_xyz789"  │ ← 继承自原预约！
│ fromReschedule  │ "abc123"        │ ← 关联原预约
│ status          │ "ACCEPTED"      │
│ rescheduled     │ false           │
└─────────────────┴─────────────────┘
```

##### 路径 B：外部日历回写改期（用户在 Google/Apple 日历中修改）

**数据流**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      外部日历回写改期数据流                                   │
└─────────────────────────────────────────────────────────────────────────────┘

用户在外部日历中修改事件时间
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ CalendarSyncService.handleEvents()                                           │
│                                                                               │
│ 1. 过滤条件：只处理 iCalUID 以 "@cal.com" 结尾的事件                         │
│    例如：iCalUID = "abc123@cal.com"                                          │
│                                                                               │
│ 2. 状态判断：                                                                  │
│    - e.status === "cancelled" → 取消                                         │
│    - 其他状态 && startTime 变化 → 改期                                        │
│                                                                               │
│ 3. 调用 this.rescheduleBooking(event, userId)                                 │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ CalendarSyncService.rescheduleBooking()                                      │
│                                                                               │
│ 1. 从 iCalUID 提取 bookingUid：                                              │
│    const [bookingUid] = event.iCalUID?.split("@") ?? [undefined];           │
│    → bookingUid = "abc123"                                                   │
│                                                                               │
│ 2. 构建改期数据：调用 buildRescheduleBookingData()                            │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ buildRescheduleBookingData(booking, event)                                   │
│                                                                               │
│ 返回：                                                                         │
│ {                                                                             │
│   eventTypeId: booking.eventTypeId,                                           │
│   start: event.start?.toISOString(),                                          │
│   end: calculatedEnd,                                                          │
│   rescheduleUid: booking.uid,           // "abc123" ← 关键！                │
│   // 注意：不传递 recurringEventId！                                           │
│   // 让后续的 createBooking 逻辑决定继承                                       │
│   idempotencyKey: "...",                                                      │
│   responses: { ... },                                                          │
│ }                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ regularBookingService.createBooking()                                         │
│                                                                               │
│ 关键参数：                                                                     │
│ bookingMeta: {                                                                │
│   skipCalendarSyncTaskCreation: true,  // 防止回环！                          │
│   skipAvailabilityCheck: true,                                                │
│   skipEventLimitsCheck: true,                                                 │
│ }                                                                             │
│                                                                               │
│ 后续流程与站内改期完全相同：                                                   │
│ 1. 检测到 rescheduleUid → 是改期                                             │
│ 2. getOriginalRescheduledBooking() → 获取原预约信息                           │
│ 3. buildNewBookingData() → 继承 recurringEventId                             │
└─────────────────────────────────────────────────────────────────────────────┘
        │
        ▼
【最终落库结果】

与站内改期**完全相同**！

原预约（uid: "abc123"）：
┌─────────────────┬─────────────────┐
│ recurringEventId│ "recur_xyz789"  │ ← 保持不变
│ status          │ "CANCELLED"     │
│ rescheduled     │ true            │
└─────────────────┴─────────────────┘

新预约（uid: "def456"）：
┌─────────────────┬─────────────────┐
│ recurringEventId│ "recur_xyz789"  │ ← 继承自原预约！
│ fromReschedule  │ "abc123"        │
│ status          │ "ACCEPTED"      │
└─────────────────┴─────────────────┘
```

---

#### 4.6.5 两条路径的关键差异与相同点

| 对比项 | 站内改期 | 外部日历回写改期 |
|-------|---------|-----------------|
| **触发入口** | UI → API → RegularBookingService | 外部日历 Webhook/轮询 → CalendarSyncService |
| **rescheduleUid 来源** | 前端显式传递 | 从 iCalUID 解析（`{uid}@cal.com`） |
| **skipCalendarSyncTaskCreation** | `false`（默认） | `true`（防止回环） |
| **getOriginalRescheduledBooking** | 调用 | 调用 |
| **buildNewBookingData 继承逻辑** | 相同 | 相同 |
| **最终落库结果** | 相同 | 相同 |
| **原预约 recurringEventId** | 保持不变 | 保持不变 |
| **新预约 recurringEventId** | 继承原预约 | 继承原预约 |

---

#### 4.6.6 特殊场景：改期的改期（链式改期）

如果一个已经改期过的预约再次被改期，会发生什么？

```
【场景】

原始预约 A：
- uid: "aaa111"
- recurringEventId: "recur_xyz789"
- status: ACCEPTED

第一次改期 → 预约 B：
- uid: "bbb222"
- recurringEventId: "recur_xyz789" （继承自 A）
- fromReschedule: "aaa111"
- status: ACCEPTED

预约 A 更新为：
- status: CANCELLED
- rescheduled: true

第二次改期（改期预约 B）→ 预约 C：
- uid: "ccc333"
- recurringEventId: ???
```

**数据流**：

```
1. 用户改期预约 B（uid: "bbb222"）

2. getOriginalRescheduledBooking("bbb222") 返回：
   {
     uid: "bbb222",
     recurringEventId: "recur_xyz789",  // 这个值还在！
     fromReschedule: "aaa111",
     status: "ACCEPTED",
     // ...
   }

3. buildNewBookingData()：
   - originalRescheduledBooking.recurringEventId = "recur_xyz789"（存在）
   → 新预约 C 的 recurringEventId = "recur_xyz789"
```

**最终结果**：

```
预约 A：
- status: CANCELLED
- rescheduled: true
- recurringEventId: "recur_xyz789"

预约 B：
- status: CANCELLED
- rescheduled: true
- recurringEventId: "recur_xyz789"

预约 C（最新）：
- status: ACCEPTED
- fromReschedule: "bbb222"
- recurringEventId: "recur_xyz789"  ← 始终保持相同的系列 ID！
```

**结论**：无论改期多少次，所有预约（包括已取消的历史预约）都保持相同的 `recurringEventId`，这确保了系列的完整性。

---

#### 4.6.7 系列归属查询示例

通过 `recurringEventId` 可以查询同一系列的所有预约，包括历史改期记录：

```typescript
// 查询同一系列的所有预约（包括已改期/已取消的）
const allBookingsInSeries = await prisma.booking.findMany({
  where: {
    recurringEventId: "recur_xyz789",
  },
  orderBy: { startTime: 'asc' },
});

// 结果示例：
[
  { uid: "aaa111", status: "CANCELLED", rescheduled: true, startTime: "2024-01-15T10:00:00Z" },
  { uid: "bbb222", status: "CANCELLED", rescheduled: true, startTime: "2024-01-16T14:00:00Z" },
  { uid: "ccc333", status: "ACCEPTED",  rescheduled: false, startTime: "2024-01-17T16:00:00Z" },
  // ... 同一系列的其他预约
]

// 只查询当前有效的预约
const activeBookings = allBookingsInSeries.filter(b => 
  b.status === "ACCEPTED" && !b.rescheduled
);
```

---

#### 4.6.8 代码位置索引（改期相关）

| 功能 | 文件位置 |
|------|----------|
| 改期参数校验 | `packages/features/bookings/lib/service/RegularBookingService.ts:435-465` |
| 获取原预约信息 | `packages/features/bookings/lib/handleNewBooking/originalRescheduledBookingUtils.ts:7-21` |
| 原预约数据库查询 | `packages/features/bookings/repositories/BookingRepository.ts:1170-1217` |
| 系列标识继承逻辑 | `packages/features/bookings/lib/handleNewBooking/createBooking.ts:168-271` |
| 外部日历回写改期 | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:154-235` |
| 构建回写改期数据 | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:247-274` |

---

## 五、关键数据关联图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据模型关系                                     │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────┐
  │    EventType     │
  ├──────────────────┤
  │ id               │
  │ recurringEvent   │◄────── 存储重复规则（JSON）
  │  - freq          │        { freq, interval, count, ... }
  │  - interval      │
  │  - count         │
  │  - ...           │
  └────────┬─────────┘
           │
           │ 1:N (事件类型有多个预约)
           ▼
  ┌──────────────────┐
  │     Booking      │
  ├──────────────────┤
  │ id               │
  │ uid              │
  │ recurringEventId │◄────── 同一系列共享此 ID
  │ startTime        │
  │ endTime          │
  │ status           │
  └────────┬─────────┘
           │
           │ 1:N (一个预约有多个外部引用)
           ▼
  ┌──────────────────────────────┐
  │      BookingReference        │
  ├──────────────────────────────┤
  │ id                           │
  │ type                         │  google_calendar / office365_calendar
  │ uid                          │  外部日历事件 ID
  │ thirdPartyRecurringEventId   │◄────── 外部日历的重复事件 ID（关键！）
  │ externalCalendarId           │
  │ credentialId                 │
  └──────────────────────────────┘
```

---

## 六、代码位置索引

| 功能 | 文件位置 |
|------|----------|
| 重复规则类型定义 | `packages/prisma/zod-utils.ts:234-243` |
| 频率枚举定义 | `packages/prisma/zod-utils.ts:81-89` |
| 数据库模型 | `packages/prisma/schema.prisma` |
| 重复预约创建服务 | `packages/features/bookings/lib/service/RecurringBookingService.ts` |
| 普通预约创建服务 | `packages/features/bookings/lib/service/RegularBookingService.ts` |
| 取消预约逻辑 | `packages/features/bookings/lib/handleCancelBooking.ts` |
| 创建/改期预约逻辑 | `packages/features/bookings/lib/handleNewBooking/createBooking.ts` |
| 外部日历同步管理 | `packages/features/bookings/lib/EventManager.ts` |
| 外部日历回写同步 | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts` |
| 前端创建 Hook | `packages/platform/atoms/hooks/bookings/useCreateRecurringBooking.ts` |
| 前端取消 Hook | `packages/platform/atoms/hooks/bookings/useCancelBooking.ts` |
| 取消输入类型 | `packages/platform/types/bookings/2024-08-13/inputs/cancel-booking.input.ts` |

---

## 七、注意事项

1. **Round Robin 重复预约**：第一个 slot 决定幸运用户（lucky user），后续 slot 复用该用户

2. **第三方重复事件 ID**：`thirdPartyRecurringEventId` 仅在第一个成功创建的外部日历事件中获取，后续 slot 复用此 ID

3. **日历同步循环防护**：
   - `skipCalendarSyncTaskCancellation`：外部日历回写取消时使用，防止无限循环
   - `skipCalendarSyncTaskCreation`：外部日历回写改期时使用，防止无限循环

4. **iCalUID 格式**：Cal.diy 创建的事件使用 `{booking-uid}@cal.com` 格式，外部日历回写时通过此后缀识别

5. **iCalSequence**：每次修改/取消预约时递增，用于 iCalendar 协议的版本控制

6. **座位预约与重复预约**：代码中有限制 `eventType.seatsPerTimeSlot && eventType.recurringEvent` 会抛出错误，即座位预约不支持重复

7. **改期后系列标识继承**：默认情况下，改期会自动继承原预约的 `recurringEventId`，保持系列关联

8. **外部日历改期触发条件**：只有当 `startTime` 发生变化时才会触发改期，仅修改标题/描述等不会触发

9. **权限检查**：外部日历回写时会验证 `booking.userId === calendarUserId`，确保日历所有者是预约组织者
