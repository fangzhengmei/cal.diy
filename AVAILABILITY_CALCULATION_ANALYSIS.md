# Cal.diy 可用时间段计算分析报告

## 1. 概述

本文档详细分析了 Cal.diy 系统中如何计算主机的可用时间段，从工作时间设置、忙碌事件排除到最终向访客展示可选时间槽的完整生成流程。

### 核心文件结构

| 文件路径 | 功能描述 |
|---------|---------|
| `packages/features/schedules/lib/slots.ts` | 时间槽生成核心逻辑 |
| `packages/features/schedules/lib/date-ranges.ts` | 日期范围处理（相交、相减、合并） |
| `packages/features/availability/lib/getUserAvailability.ts` | 用户可用性计算核心 |
| `packages/lib/availability.ts` | 工作时间处理与转换 |
| `packages/features/busyTimes/services/getBusyTimes.ts` | 忙碌时间获取服务 |
| `packages/trpc/server/routers/viewer/slots/util.ts` | 可用时间槽服务主入口 |

---

## 2. 工作时间设置机制

### 2.1 数据模型

工作时间存储在 `Availability` 模型中，核心字段包括：

- **`days`**: 星期几数组（0=周日, 1-6=周一到周六）
- **`startTime`**: 开始时间（UTC 时间）
- **`endTime`**: 结束时间（UTC 时间）
- **`date`**: 可选，特定日期覆盖（用于日期覆盖功能）

### 2.2 时区转换

工作时间在数据库中以 UTC 存储，在计算可用性时需要转换为用户的本地时区。

**核心转换函数**: `getWorkingHours` ([packages/lib/availability.ts:61](packages/lib/availability.ts#L61))

```typescript
export function getWorkingHours(
  relativeTimeUnit: { timeZone?: string; utcOffset?: number },
  availability: { userId?: number | null; days: number[]; startTime: ConfigType; endTime: ConfigType }[]
)
```

**转换逻辑**:
1. 获取目标时区的 UTC 偏移量
2. 将 UTC 时间转换为分钟数并调整偏移
3. 处理跨天边界（0-1439 分钟范围）
4. 处理夏令时（DST）切换

### 2.3 日期范围构建

**核心函数**: `buildDateRanges` ([packages/features/schedules/lib/date-ranges.ts:226](packages/features/schedules/lib/date-ranges.ts#L226))

该函数将工作时间规则转换为具体的日期范围，处理以下元素：

1. **常规工作时间**（Working Hours）: 每周重复的时间规则
2. **日期覆盖**（Date Overrides）: 特定日期的可用时间设置，优先级高于常规工作时间
3. **请假**（Out of Office）: 标记为不可用的日期
4. **旅行日程**（Travel Schedules）: 旅行期间的时区切换

**处理流程**:
```
工作时间规则 → 按日期展开 → 合并重叠范围 → 应用日期覆盖 → 排除请假日期
```

### 2.4 旅行日程支持

系统支持用户设置旅行日程，在旅行期间自动切换时区：

```typescript
function getAdjustedTimezone(date: Dayjs, timeZone: string, travelSchedules: TravelSchedule[]) {
  let adjustedTimezone = timeZone;
  for (const travelSchedule of travelSchedules) {
    if (
      !date.isBefore(travelSchedule.startDate) &&
      (!travelSchedule.endDate || !date.isAfter(travelSchedule.endDate))
    ) {
      adjustedTimezone = travelSchedule.timeZone;
      break;
    }
  }
  return adjustedTimezone;
}
```
[packages/features/schedules/lib/date-ranges.ts:16](packages/features/schedules/lib/date-ranges.ts#L16)

---

## 3. 忙碌事件排除逻辑

### 3.1 忙碌时间来源

系统从多个来源获取忙碌时间，按优先级从高到低排列：

| 来源 | 优先级 | 描述 |
|-----|-------|------|
| 日历忙碌时间 | 1 | 从连接的外部日历（Google、Outlook 等）获取 |
| 已接受预订 | 2 | 数据库中已确认的预订 |
| 缓冲时间 | 3 | 预订前后的缓冲时间 |
| 预订限制 | 4 | 每天/每周/每月/每年的预订数量限制 |
| 时长限制 | 5 | 每天/每周/每月/每年的总时长限制 |
| 团队预订限制 | 6 | 团队级别的限制 |
| 请假（OOO） | 7 | 用户设置的请假 |
| 节假日 | 8 | 用户设置的国家节假日 |
| 预留槽位 | 9 | 其他用户临时预留的槽位 |

### 3.2 核心获取逻辑

**核心函数**: `BusyTimesService._getBusyTimes` ([packages/features/busyTimes/services/getBusyTimes.ts:29](packages/features/busyTimes/services/getBusyTimes.ts#L29))

**处理流程**:

```typescript
// 1. 获取数据库中的预订
const bookings = await bookingRepo.findAllExistingBookingsForEventTypeBetween({...});

// 2. 处理预订的缓冲时间
const minutesToBlockBeforeEvent = (eventType?.beforeEventBuffer || 0) + (afterEventBuffer || 0);
const minutesToBlockAfterEvent = (eventType?.afterEventBuffer || 0) + (beforeEventBuffer || 0);

// 3. 获取外部日历的忙碌时间
if (credentials?.length > 0 && !bypassBusyCalendarTimes) {
  const calendarBusyTimesQuery = await getBusyCalendarTimes(
    credentials, startTime, endTime, selectedCalendars, mode
  );
  // 应用缓冲时间到日历忙碌时间
  busyTimes.push(...result.map((busyTime) => ({
    start: busyTime.start.subtract(afterEventBuffer || 0, "minute").toDate(),
    end: busyTime.end.add(beforeEventBuffer || 0, "minute").toDate(),
  })));
}
```
[packages/features/busyTimes/services/getBusyTimes.ts:29](packages/features/busyTimes/services/getBusyTimes.ts#L29)

### 3.3 座位事件特殊处理

对于支持座位的事件类型（`seatsPerTimeSlot > 0`），系统有特殊的处理逻辑：

```typescript
// 座位事件：座位未满时不阻塞，只阻塞缓冲时间
if (
  bookingSeatCountMap[bookedAt] < (eventType?.seatsPerTimeSlot || 1) &&
  eventTypeId === eventType?.id
) {
  // 只添加前后缓冲时间作为忙碌时间
  if (minutesToBlockBeforeEvent) {
    aggregate.push({
      start: dayjs(startTime).subtract(minutesToBlockBeforeEvent, "minute").toDate(),
      end: dayjs(startTime).toDate(),
      title: "busy_time.buffer_time",
      source: "Buffer Time for seated event (before)",
    });
  }
  if (minutesToBlockAfterEvent) {
    aggregate.push({
      start: dayjs(endTime).toDate(),
      end: dayjs(endTime).add(minutesToBlockAfterEvent, "minute").toDate(),
      title: "busy_time.buffer_time",
      source: "Buffer Time for seated event (after)",
    });
  }
  return aggregate;
}
```
[packages/features/busyTimes/services/getBusyTimes.ts:143](packages/features/busyTimes/services/getBusyTimes.ts#L143)

### 3.4 重新安排例外

当用户重新安排预订时，原预订时间在相同时间不被视为忙碌：

```typescript
// 重新安排的预订在相同时间应该可以预订
if (rest.uid === rescheduleUid) {
  return aggregate;
}
```
[packages/features/busyTimes/services/getBusyTimes.ts:173](packages/features/busyTimes/services/getBusyTimes.ts#L173)

### 3.5 限制检查逻辑

**预订限制**（Booking Limits）和**时长限制**（Duration Limits）通过 `getBusyTimesFromLimits` 函数处理：

```typescript
// 检查是否达到限制
for (const periodStart of periodStartDates) {
  let totalBookings = 0;
  for (const booking of userBookings) {
    if (!isBookingWithinPeriod(booking, periodStart, periodEnd, timeZone)) {
      continue;
    }
    totalBookings++;
    if (totalBookings >= limit) {
      // 达到限制，标记为忙碌
      limitManager.addBusyTime({
        start: periodStart,
        unit,
        timeZone,
        title,
        source,
      });
      break;
    }
  }
}
```
[packages/trpc/server/routers/viewer/slots/util.ts:433](packages/trpc/server/routers/viewer/slots/util.ts#L433)

---

## 4. 时间槽生成完整流程

### 4.1 主入口

**核心服务**: `AvailableSlotsService._getAvailableSlots` ([packages/trpc/server/routers/viewer/slots/util.ts:897](packages/trpc/server/routers/viewer/slots/util.ts#L897))

### 4.2 完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        时间槽生成完整流程                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 获取事件类型                                                              │
│     ├── 动态事件类型：多个用户组合的动态预订                                    │
│     └── 常规事件类型：标准事件类型                                             │
│                                                                              │
│  2. 获取合格主机                                                              │
│     ├── 过滤被阻止的主机（Watchlist）                                         │
│     ├── 轮询事件：选择可用主机                                                 │
│     └── 集体事件：所有主机必须同时可用                                         │
│                                                                              │
│  3. 计算主机可用性                                                            │
│     ├── 3.1 获取用户工作时间                                                 │
│     │       └── buildDateRanges()                                           │
│     │                                                                       │
│     ├── 3.2 获取忙碌时间                                                     │
│     │       ├── 日历忙碌时间 (getBusyCalendarTimes)                        │
│     │       ├── 已接受预订                                                   │
│     │       ├── 缓冲时间                                                     │
│     │       ├── 预订限制/时长限制                                            │
│     │       └── 请假/节假日                                                  │
│     │                                                                       │
│     └── 3.3 计算可用日期范围                                                 │
│             └── subtract(dateRanges, busyTimes)  // 从工作时间减去忙碌时间  │
│                                                                              │
│  4. 聚合可用性                                                                │
│     ├── 集体事件（COLLECTIVE）: 所有主机的交集                                │
│     │       └── intersect(allUsersDateRanges)                              │
│     │                                                                       │
│     ├── 轮询事件（ROUND_ROBIN）: 任一主机的可用时间                           │
│     │       └── union(allUsersDateRanges)                                  │
│     │                                                                       │
│     └── 管理事件（MANAGED）: 类似轮询，但有额外逻辑                           │
│                                                                              │
│  5. 生成时间槽                                                                │
│     └── getSlots({                                                          │
│             dateRanges: aggregatedAvailability,                            │
│             frequency: eventType.slotInterval,                             │
│             eventLength: input.duration,                                   │
│             minimumBookingNotice: eventType.minimumBookingNotice,         │
│             showOptimizedSlots: eventType.showOptimizedSlots,             │
│         })                                                                  │
│                                                                              │
│  6. 应用额外限制                                                              │
│     ├── 限制计划（Restriction Schedule）: 额外的可用性限制                    │
│     ├── 预留槽位检查：其他用户临时预留的槽位                                   │
│     ├── 期限限制：                                                           │
│     │       ├── 过去时间不可用                                                │
│     │       ├── 最小预订通知                                                  │
│     │       └── 未来限制（滚动窗口等）                                        │
│     └── 座位事件：检查剩余座位                                                 │
│                                                                              │
│  7. 格式化输出                                                                │
│     ├── 按日期分组                                                           │
│     ├── 转换为 ISO 格式                                                      │
│     └── 添加座位信息（如果适用）                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 时间槽生成核心逻辑

**核心函数**: `getSlots` ([packages/features/schedules/lib/slots.ts:232](packages/features/schedules/lib/slots.ts#L232))

**关键参数**:
- `inviteeDate`: 访客日期
- `frequency`: 槽位间隔（分钟）
- `dateRanges`: 可用日期范围
- `minimumBookingNotice`: 最小预订通知时间（分钟）
- `eventLength`: 事件时长（分钟）
- `offsetStart`: 开始偏移量
- `datesOutOfOffice`: 请假日期
- `showOptimizedSlots`: 是否显示优化槽位

**槽位起始时间校正**:

```typescript
function getCorrectedSlotStartTime({
  slotStartTime,
  range,
  showOptimizedSlots,
  interval,
}: {...}) {
  if (showOptimizedSlots) {
    // 优化槽位显示：尽量让槽位从整点、15分、5分开始
    const minutesRequiredToMoveToNextSlot = interval - (slotStartTime.minute() % interval);
    const minutesRequiredToMoveTo15MinSlot = 15 - (slotStartTime.minute() % 15);
    const minutesRequiredToMoveTo5MinSlot = 5 - (slotStartTime.minute() % 5);
    const extraMinutesAvailable = range.end.diff(slotStartTime, "minutes") % interval;

    // 根据可用时间决定如何调整
    if (extraMinutesAvailable >= minutesRequiredToMoveToNextSlot) {
      // 推到下一个完整间隔
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveToNextSlot, "minute");
    } else if (extraMinutesAvailable >= minutesRequiredToMoveTo15MinSlot) {
      // 推到下一个15分刻度
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveTo15MinSlot, "minute");
    } else if (extraMinutesAvailable >= minutesRequiredToMoveTo5MinSlot) {
      // 推到下一个5分刻度
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveTo5MinSlot, "minute");
    }
    return correctedSlotStartTime;
  }

  // 非优化模式：从小时开始，向上取整到间隔
  return slotStartTime.startOf("hour").add(Math.ceil(slotStartTime.minute() / interval) * interval, "minute");
}
```
[packages/features/schedules/lib/slots.ts:27](packages/features/schedules/lib/slots.ts#L27)

### 4.4 日期范围运算

系统提供了多个日期范围运算函数：

**1. 相减运算**（subtract）: 从源范围中排除指定范围

```typescript
export function subtract<TSourceRange extends DateRange, TExcludedRange extends DateRange>(
  sourceRanges: TSourceRange[],
  excludedRanges: TExcludedRange[]
): SubtractedRange<TSourceRange>[] {
  const result: SubtractedRange<TSourceRange>[] = [];
  const sortedExcludedRanges = [...excludedRanges].sort((a, b) => a.start.valueOf() - b.start.valueOf());

  for (const { start: sourceStart, end: sourceEnd, ...passThrough } of sourceRanges) {
    let currentStart = sourceStart;

    for (const excludedRange of sortedExcludedRanges) {
      if (excludedRange.start.valueOf() >= sourceEnd.valueOf()) break;
      if (excludedRange.end.valueOf() <= currentStart.valueOf()) continue;

      // 有重叠，拆分源范围
      if (excludedRange.start.valueOf() > currentStart.valueOf()) {
        result.push({ start: currentStart, end: excludedRange.start, ...passThrough });
      }

      if (excludedRange.end.valueOf() > currentStart.valueOf()) {
        currentStart = excludedRange.end;
      }
    }

    // 添加剩余部分
    if (sourceEnd.valueOf() > currentStart.valueOf()) {
      result.push({ start: currentStart, end: sourceEnd, ...passThrough });
    }
  }

  return result;
}
```
[packages/features/schedules/lib/date-ranges.ts:423](packages/features/schedules/lib/date-ranges.ts#L423)

**2. 相交运算**（intersect）: 找出多个范围的共同部分（用于集体事件）

```typescript
export function intersect(ranges: DateRange[][]): DateRange[] {
  if (!ranges.length) return [];

  let commonAvailability = sortedRanges[0];

  for (let i = 1; i < sortedRanges.length; i++) {
    if (commonAvailability.length === 0) return [];

    const userRanges = sortedRanges[i];
    const intersectedRanges: ProcessedDateRange[] = [];

    let commonIndex = 0;
    let userIndex = 0;

    while (commonIndex < commonAvailability.length && userIndex < userRanges.length) {
      const commonRange = commonAvailability[commonIndex];
      const userRange = userRanges[userIndex];

      const intersectStartValue = Math.max(commonRange.startValue, userRange.startValue);
      const intersectEndValue = Math.min(commonRange.endValue, userRange.endValue);

      if (intersectStartValue < intersectEndValue) {
        // 有重叠，添加交集
        intersectedRanges.push({
          start: commonRange.startValue > userRange.startValue ? commonRange.start : userRange.start,
          end: commonRange.endValue < userRange.endValue ? commonRange.end : userRange.end,
          startValue: intersectStartValue,
          endValue: intersectEndValue,
        });
      }

      // 移动指针
      if (commonRange.endValue <= userRange.endValue) {
        commonIndex++;
      } else {
        userIndex++;
      }
    }
    commonAvailability = intersectedRanges;
  }

  return commonAvailability.map(({ start, end }) => ({ start, end }));
}
```
[packages/features/schedules/lib/date-ranges.ts:354](packages/features/schedules/lib/date-ranges.ts#L354)

---

## 5. 排除规则优先级详解

### 5.1 优先级列表

从代码分析来看，排除规则的优先级如下（从高到低）：

| 优先级 | 规则类型 | 处理位置 | 说明 |
|-------|---------|---------|------|
| 1 | 日历忙碌时间 | `getBusyCalendarTimes` | 外部日历的忙碌事件，最高优先级 |
| 2 | 已接受预订 | `findAllExistingBookingsForEventTypeBetween` | 数据库中已确认的预订 |
| 3 | 缓冲时间 | `_getBusyTimes` 中的计算 | 预订前后的缓冲 |
| 4 | 预订限制 | `getBusyTimesFromLimits` | 数量限制（每天/周/月/年） |
| 5 | 时长限制 | `getBusyTimesFromLimits` | 时长限制（每天/周/月/年） |
| 6 | 团队预订限制 | `getBusyTimesFromTeamLimits` | 团队级别的限制 |
| 7 | 请假（OOO） | `buildDateRanges` | 用户设置的请假 |
| 8 | 节假日 | `calculateHolidayBlockedDates` | 用户设置的国家节假日 |
| 9 | 预留槽位 | `checkForConflicts` | 其他用户临时预留 |

### 5.2 优先级处理机制

**在 `getUserAvailability` 中的处理顺序**:

```typescript
// 1. 首先构建工作时间范围（包含请假和节假日排除）
const { dateRanges, oooExcludedDateRanges } = buildDateRanges({
  dateFrom,
  dateTo,
  availability,
  timeZone: finalTimezone,
  travelSchedules,
  outOfOffice: datesOutOfOffice,  // 请假和节假日在这里排除
});

// 2. 然后获取所有忙碌时间
const detailedBusyTimesWithSource: EventBusyDetails[] = [
  ...busyTimes,                    // 日历忙碌时间 + 预订 + 缓冲
  ...busyTimesFromLimits,          // 预订限制 + 时长限制
  ...busyTimesFromTeamLimits,      // 团队限制
];

// 3. 最后从工作时间中减去所有忙碌时间
const dateRangesInWhichUserIsAvailable = subtract(dateRanges, formattedBusyTimes);
```
[packages/features/availability/lib/getUserAvailability.ts:511-648](packages/features/availability/lib/getUserAvailability.ts#L511-L648)

**关键点**:
- 请假和节假日在 `buildDateRanges` 阶段就已经从工作时间中排除
- 其他忙碌时间通过 `subtract` 函数从剩余可用时间中排除
- 同一优先级的忙碌时间通过 `mergeOverlappingRanges` 合并

---

## 6. 边界情况处理

### 6.1 时区边界

**问题**: 不同时区的日期边界不同，特别是半小时偏移时区（如 Asia/Kolkata GMT+5:30）

**解决方案**:

```typescript
// 在计算时区边界时，确保在目标时区中检查分钟对齐
slotStartTime = slotStartTime.tz(timeZone);

if (slotStartTime.minute() % interval !== 0) {
  slotStartTime = getCorrectedSlotStartTime({
    showOptimizedSlots,
    interval,
    slotStartTime,
    range,
  });
}
```
[packages/features/schedules/lib/slots.ts:138-147](packages/features/schedules/lib/slots.ts#L138-L147)

**夏令时处理**:

```typescript
// 在 processWorkingHours 中处理夏令时切换
const offsetBeginningOfDay = dayjs(start.format("YYYY-MM-DD hh:mm")).tz(adjustedTimezone).utcOffset();
const offsetDiff = start.utcOffset() - offsetBeginningOfDay; // DST 变化时会有 60 分钟偏移

start = start.add(offsetDiff, "minute");
end = end.add(offsetDiff, "minute");
```
[packages/features/schedules/lib/date-ranges.ts:70-74](packages/features/schedules/lib/date-ranges.ts#L70-L74)

### 6.2 日期覆盖边界

**问题**: UTC 和本地日期的差异可能导致日期覆盖不生效

**解决方案**:

```typescript
// 使用 ±1 天的缓冲区来处理日期边界
if (
  itemDateAsUtc.isBetween(
    dateFrom.subtract(1, "day").startOf("day"),
    dateTo.add(1, "day").endOf("day"),
    null,
    "[]"
  )
) {
  // 处理日期覆盖
}
```
[packages/features/schedules/lib/date-ranges.ts:282-289](packages/features/schedules/lib/date-ranges.ts#L282-L289)

### 6.3 最小预订通知

**问题**: 确保槽位时间在当前时间 + 最小通知时间之后

**解决方案**:

```typescript
// 在 getSlots 中处理
const startTimeWithMinNotice = dayjs.utc().add(minimumBookingNotice, "minute");

orderedDateRanges.forEach((range) => {
  let slotStartTime = range.start.utc().isAfter(startTimeWithMinNotice)
    ? range.start
    : startTimeWithMinNotice;
  // ... 后续处理
});
```
[packages/features/schedules/lib/slots.ts:123-130](packages/features/schedules/lib/slots.ts#L123-L130)

**额外检查**:

```typescript
// 在最后过滤阶段再次检查
try {
  isOutOfBounds = isTimeOutOfBounds({
    time: slot.time,
    minimumBookingNotice: eventType.minimumBookingNotice,
  });
} catch (error) {
  if (error instanceof BookingDateInPastError) {
    throw new TRPCError({
      code: "BAD_REQUEST",
      message: error.message,
    });
  }
  throw error;
}
```
[packages/trpc/server/routers/viewer/slots/util.ts:1343-1355](packages/trpc/server/routers/viewer/slots/util.ts#L1343-L1355)

### 6.4 23:59 边界

**问题**: 用户设置工作时间到 23:59，但实际上应该到午夜

**解决方案**:

```typescript
// INFO: 我们只允许用户设置可用性到 11:59PM，这实际上不会让他们可用到午夜
if (endResult.hour() === 23 && endResult.minute() === 59) {
  endResult = endResult.add(1, "minute");
}
```
[packages/features/schedules/lib/date-ranges.ts:81-83](packages/features/schedules/lib/date-ranges.ts#L81-L83)

### 6.5 跨天工作时间

**问题**: 工作时间跨越午夜（如 22:00 - 02:00）

**解决方案**:

```typescript
// 在 getWorkingHours 中处理跨天
// 检查是否溢出到前一天
if (startTime < MINUTES_DAY_START || endTime < MINUTES_DAY_START) {
  const newWorkingHours: WorkingHours = {
    days: schedule.days.map((day) => (day - 1 >= 0 ? day - 1 : 6)),
    startTime: startTime + MINUTES_IN_DAY,
    endTime: Math.min(endTime + MINUTES_IN_DAY, MINUTES_DAY_END),
  };
  currentWorkingHours.push(newWorkingHours);
}
// 检查是否溢出到后一天
else if (startTime > MINUTES_DAY_END || endTime > MINUTES_IN_DAY) {
  const newWorkingHours: WorkingHours = {
    days: schedule.days.map((day) => (day + 1) % 7),
    startTime: Math.max(startTime - MINUTES_IN_DAY, MINUTES_DAY_START),
    endTime: endTime - MINUTES_IN_DAY,
  };
  currentWorkingHours.push(newWorkingHours);
}
```
[packages/lib/availability.ts:102-120](packages/lib/availability.ts#L102-L120)

### 6.6 座位事件边界

**问题**: 座位事件在座位满之前应该可用

**解决方案**:

```typescript
// 座位引用在当前事件完全预订之前是非阻塞的
if (
  bookingSeatCountMap[bookedAt] < (eventType?.seatsPerTimeSlot || 1) &&
  eventTypeId === eventType?.id
) {
  // 只添加前后缓冲时间作为忙碌时间
  if (minutesToBlockBeforeEvent) {
    aggregate.push({
      start: dayjs(startTime).subtract(minutesToBlockBeforeEvent, "minute").toDate(),
      end: dayjs(startTime).toDate(),
      title: "busy_time.buffer_time",
      source: "Buffer Time for seated event (before)",
    });
  }
  if (minutesToBlockAfterEvent) {
    aggregate.push({
      start: dayjs(endTime).toDate(),
      end: dayjs(endTime).add(minutesToBlockAfterEvent, "minute").toDate(),
      title: "busy_time.buffer_time",
      source: "Buffer Time for seated event (after)",
    });
  }
  return aggregate;
}
```
[packages/features/busyTimes/services/getBusyTimes.ts:143-167](packages/features/busyTimes/services/getBusyTimes.ts#L143-L167)

### 6.7 重新安排边界

**问题**: 用户重新安排预订时，原时间应该可用

**解决方案**:

```typescript
// 重新安排相同预订到相同时间应该是可能的
if (rest.uid === rescheduleUid) {
  return aggregate;
}
```
[packages/features/busyTimes/services/getBusyTimes.ts:173-175](packages/features/busyTimes/services/getBusyTimes.ts#L173-L175)

**额外处理**:

```typescript
// 从日历忙碌时间中排除重新安排的预订
if (rescheduleUid) {
  const originalRescheduleBooking = bookings.find((booking) => booking.uid === rescheduleUid);
  if (originalRescheduleBooking) {
    openSeatsDateRanges.push({
      start: dayjs(originalRescheduleBooking.startTime),
      end: dayjs(originalRescheduleBooking.endTime),
    });
  }
}

// 从日历忙碌时间中减去这些范围
const result = subtract(
  calendarBusyTimes.map((value) => ({...})),
  openSeatsDateRanges
);
```
[packages/features/busyTimes/services/getBusyTimes.ts:243-261](packages/features/busyTimes/services/getBusyTimes.ts#L243-L261)

---

## 7. 关键数据结构

### 7.1 DateRange

```typescript
export type DateRange = {
  start: Dayjs;
  end: Dayjs;
};
```
[packages/features/schedules/lib/date-ranges.ts:6](packages/features/schedules/lib/date-ranges.ts#L6)

### 7.2 WorkingHours

```typescript
export type WorkingHours = Pick<Availability, "days" | "startTime" | "endTime">;
```
[packages/features/schedules/lib/date-ranges.ts:12](packages/features/schedules/lib/date-ranges.ts#L12)

### 7.3 EventBusyDetails

```typescript
type EventBusyDetails = {
  start: Date | string;
  end: Date | string;
  title?: string;
  source?: string;
  userId?: number;
};
```

### 7.4 GetUserAvailabilityResult

```typescript
export type GetUserAvailabilityResult = {
  busy: UserAvailabilityBusyDetails[];
  timeZone: string;
  dateRanges: { start: dayjs.Dayjs; end: dayjs.Dayjs }[];
  oooExcludedDateRanges: { start: dayjs.Dayjs; end: dayjs.Dayjs }[];
  workingHours: WorkingHoursWithUserId[];
  dateOverrides: TimeRange[];
  currentSeats: {...}[] | null;
  datesOutOfOffice?: IOutOfOfficeData;
};
```
[packages/features/availability/lib/getUserAvailability.ts:173](packages/features/availability/lib/getUserAvailability.ts#L173)

---

## 8. 性能优化

### 8.1 缓存机制

```typescript
// 在 AvailableSlotsService 中使用 Redis 缓存
function withSlotsCache(
  redisClient: IRedisService,
  func: (args: GetScheduleOptions) => Promise<IGetAvailableSlots>
) {
  return async (args: GetScheduleOptions): Promise<IGetAvailableSlots> => {
    const cacheKey = `${JSON.stringify(args.input)}`;
    let cachedResult: IGetAvailableSlots | null = null;
    
    try {
      cachedResult = await redisClient.get(cacheKey);
    } catch (err) {
      // 缓存失败时直接调用函数
    }

    if (cachedResult) {
      log.info("[CACHE HIT] Available slots", { cacheKey });
      return cachedResult;
    }
    
    const result = await func(args);
    // 异步设置缓存，不阻塞响应
    redisClient.set(cacheKey, result, { ttl });
    return result;
  };
}
```
[packages/trpc/server/routers/viewer/slots/util.ts:104](packages/trpc/server/routers/viewer/slots/util.ts#L104)

### 8.2 批量查询

```typescript
// 限制检查使用批量查询
const BATCH_SIZE_FOR_LIMIT_CHECKS = 50;
const MAX_CONCURRENT_LIMIT_CHECK_BATCHES = 5;

private async fetchBookingsForLimitChecksBatched(params: {...}) {
  // 分成批次
  const batches: number[][] = [];
  for (let i = 0; i < userIds.length; i += BATCH_SIZE_FOR_LIMIT_CHECKS) {
    batches.push(userIds.slice(i, i + BATCH_SIZE_FOR_LIMIT_CHECKS));
  }

  // 并行执行批次，但限制并发数
  for (let i = 0; i < batches.length; i += MAX_CONCURRENT_LIMIT_CHECK_BATCHES) {
    const currentBatch = batches.slice(i, i + MAX_CONCURRENT_LIMIT_CHECK_BATCHES);
    const batchResults = await Promise.all(
      currentBatch.map((batchUserIds) =>
        this.fetchBookingsForLimitChecksBatch({...})
      )
    );
    results.push(...batchResults.flat());
  }
}
```
[packages/features/busyTimes/services/getBusyTimes.ts:389-442](packages/features/busyTimes/services/getBusyTimes.ts#L389-L442)

---

## 9. 总结

Cal.diy 的可用时间计算系统是一个复杂但设计合理的系统，具有以下特点：

### 9.1 核心设计原则

1. **分层设计**: 从工作时间 → 忙碌事件排除 → 时间槽生成 → 输出格式化，每一层职责清晰
2. **优先级明确**: 不同来源的忙碌时间有明确的优先级，确保正确的冲突处理
3. **边界处理完善**: 时区、夏令时、跨天、座位事件、重新安排等边界情况都有专门处理
4. **性能优化**: 使用缓存、批量查询、提前退出等优化手段

### 9.2 关键流程

1. **工作时间设置**: 支持常规工作时间、日期覆盖、旅行日程时区切换
2. **忙碌事件排除**: 从多个来源获取忙碌时间，按优先级处理
3. **时间槽生成**: 基于可用范围、频率、时长生成槽位，支持优化显示
4. **额外限制**: 应用限制计划、预留槽位、期限限制等

### 9.3 技术亮点

1. **日期范围运算**: 实现了相交、相减、合并等基础运算，是可用性计算的核心
2. **时区处理**: 完善的时区转换和夏令时处理，支持全球用户
3. **座位事件**: 创新的座位预订模式，支持部分可用
4. **限制系统**: 灵活的预订限制和时长限制，支持多种时间粒度

---

*报告生成日期: 2026-05-02*
