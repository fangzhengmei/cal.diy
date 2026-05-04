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

### 3.3 双向同步的关键机制

#### 创建时的同步

1. **Cal.diy → 外部日历**：
   - 创建 Booking 后，调用 `EventManager.create()`
   - 外部日历返回 `thirdPartyRecurringEventId`（对于重复事件）
   - 存储到 `BookingReference.thirdPartyRecurringEventId`

2. **外部日历 → Cal.diy**：
   - 通过 Webhook 或轮询监听外部日历变更
   - 使用 `iCalUID` 或 `externalCalendarId` 匹配 Cal.diy 中的记录

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
| 外部日历同步管理 | `packages/features/bookings/lib/EventManager.ts` |
| 前端创建 Hook | `packages/platform/atoms/hooks/bookings/useCreateRecurringBooking.ts` |
| 前端取消 Hook | `packages/platform/atoms/hooks/bookings/useCancelBooking.ts` |
| 取消输入类型 | `packages/platform/types/bookings/2024-08-13/inputs/cancel-booking.input.ts` |

---

## 七、注意事项

1. **Round Robin 重复预约**：第一个 slot 决定幸运用户（lucky user），后续 slot 复用该用户

2. **第三方重复事件 ID**：`thirdPartyRecurringEventId` 仅在第一个成功创建的外部日历事件中获取，后续 slot 复用此 ID

3. **日历同步循环防护**：取消操作中有 `skipCalendarSyncTaskCancellation` 参数，用于防止外部日历 Webhook 触发的取消操作再次同步回外部日历（无限循环）

4. **iCalSequence**：每次修改/取消预约时递增，用于 iCalendar 协议的版本控制

5. **座位预约与重复预约**：代码中有限制 `eventType.seatsPerTimeSlot && eventType.recurringEvent` 会抛出错误，即座位预约不支持重复
