# Cal.diy 可用时段计算链路分析报告

## 1. 概述

本文档按照**实际代码执行顺序**详细分析 Cal.diy 系统中可用时段的计算链路，重点区分：
- **主机侧**：工作时间、日期覆盖、请假、忙碌事件的裁剪顺序
- **访客侧**：时间槽生成、冲突占位、最小提前预约、过期时间过滤的实际处理顺序

---

## 2. 主机可用时段计算链路

### 2.1 整体流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      主机可用时段计算链路（按执行顺序）                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  输入：用户工作时间规则（Availability 数组）                                    │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段1：buildDateRanges() - 构建基础日期范围                              │  │
│  │                                                                       │  │
│  │  1.1 处理常规工作时间（Working Hours）                                   │  │
│  │      ├── 遍历 dateFrom 到 dateTo 的每一天                               │  │
│  │      ├── 检查星期几是否匹配工作时间规则                                   │  │
│  │      ├── 转换时区（考虑夏令时 DST）                                      │  │
│  │      └── 生成 DateRange: { start, end }                                │  │
│  │                                                                       │  │
│  │  1.2 处理日期覆盖（Date Overrides）                                      │  │
│  │      ├── 从 Availability 中筛选有 date 字段的条目                       │  │
│  │      ├── 如果某一天有日期覆盖，**完全覆盖**当天的常规工作时间              │  │
│  │      └── 日期覆盖可以是 0-length（表示当天不可用）                        │  │
│  │                                                                       │  │
│  │  1.3 处理请假（Out of Office）                                           │  │
│  │      ├── 为每个请假日期生成 0-length 的 DateRange                       │  │
│  │      │   { start: 日期, end: 日期 } （start === end）                  │  │
│  │      └── 通过"覆盖+过滤"机制排除当天                                      │  │
│  │                                                                       │  │
│  │  输出：                                                                 │  │
│  │  ├── dateRanges: 工作时间 + 日期覆盖（过滤 0-length）                  │  │
│  │  └── oooExcludedDateRanges: 工作时间 + 日期覆盖 + 请假（过滤 0-length） │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段2：获取忙碌时间集合                                                   │  │
│  │                                                                       │  │
│  │  2.1 日历忙碌时间（Calendar Busy Times）                                │  │
│  │      ├── 从连接的外部日历（Google、Outlook 等）获取                      │  │
│  │      └── 应用事件前后的缓冲时间                                          │  │
│  │                                                                       │  │
│  │  2.2 已接受预订（Accepted Bookings）                                    │  │
│  │      ├── 从数据库查询状态为 ACCEPTED 的预订                             │  │
│  │      ├── 应用预订前后的缓冲时间                                          │  │
│  │      └── 座位事件特殊处理：座位未满时只阻塞缓冲时间                       │  │
│  │                                                                       │  │
│  │  2.3 预订限制（Booking Limits）                                          │  │
│  │      ├── 检查每天/每周/每月/每年的预订数量限制                            │  │
│  │      └── 达到限制时标记该时间段为忙碌                                     │  │
│  │                                                                       │  │
│  │  2.4 时长限制（Duration Limits）                                         │  │
│  │      ├── 检查每天/每周/每月/每年的总时长限制                              │  │
│  │      └── 达到限制时标记该时间段为忙碌                                     │  │
│  │                                                                       │  │
│  │  2.5 团队预订限制（Team Booking Limits）                                │  │
│  │      └── 团队级别的预订数量限制                                          │  │
│  │                                                                       │  │
│  │  输出：                                                                 │  │
│  │  └── detailedBusyTimes: 合并所有忙碌时间的数组                         │  │
│  │      [                                                                 │  │
│  │        { start: Dayjs, end: Dayjs, title?, source? },                │  │
│  │        ...                                                             │  │
│  │      ]                                                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段3：统一裁剪 - subtract()                                            │  │
│  │                                                                       │  │
│  │  从 dateRanges 中减去所有 busyTimes，得到最终可用时间：                  │  │
│  │                                                                       │  │
│  │  dateRangesInWhichUserIsAvailable =                                   │  │
│  │      subtract(dateRanges, formattedBusyTimes)                        │  │
│  │                                                                       │  │
│  │  subtract 算法：                                                       │  │
│  │  ├── 对每个 sourceRange，按顺序检查 excludedRanges                    │  │
│  │  ├── 如果 excludedRange 完全在 sourceRange 之前：跳过                  │  │
│  │  ├── 如果 excludedRange 完全在 sourceRange 之后：跳过                  │  │
│  │  ├── 如果有重叠：                                                       │  │
│  │  │   ├── 重叠前的部分：保留（如果存在）                                 │  │
│  │  │   └── currentStart 移动到 excludedRange.end                        │  │
│  │  └── 最后检查：如果 currentStart < sourceEnd，保留剩余部分              │  │
│  │                                                                       │  │
│  │  输出：                                                                 │  │
│  │  └── 最终可用时间范围数组                                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 阶段1详解：buildDateRanges() 的实际处理顺序

**核心函数**: `packages/features/schedules/lib/date-ranges.ts:226`

#### 2.2.1 处理逻辑的关键洞察

`buildDateRanges` 不是按"优先级"处理，而是通过**对象展开运算符的覆盖机制**和**0-length 过滤**实现的：

```typescript
// 1. 按日期分组处理
const groupedWorkingHours = groupByDate(/* 常规工作时间展开后的范围 */);
const groupedDateOverrides = groupByDate(/* 日期覆盖的范围 */);
const groupedOOO = groupByDate(/* 请假日期的 0-length 范围 */);

// 2. 合并时的覆盖机制
const dateRanges = Object.values({
  ...groupedWorkingHours,    // 常规工作时间
  ...groupedDateOverrides,   // 日期覆盖（会覆盖同一天的工作时间！）
}).map(
  // 过滤掉 0-length 的范围
  (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
);

// 3. 包含请假排除的版本
const oooExcludedDateRanges = Object.values({
  ...groupedWorkingHours,
  ...groupedDateOverrides,
  ...groupedOOO,              // 请假会覆盖同一天的所有时间！
}).map(
  (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
);
```
[packages/features/schedules/lib/date-ranges.ts:312-327](packages/features/schedules/lib/date-ranges.ts#L312-L327)

#### 2.2.2 实际例子说明

**场景**：
- 常规工作时间：周一至周五 9:00-18:00
- 日期覆盖：2026-05-05（周一）10:00-16:00
- 请假：2026-05-06（周二）

**处理过程**：

| 日期 | groupedWorkingHours | groupedDateOverrides | groupedOOO | 合并后（dateRanges） | 合并后（oooExcludedDateRanges） |
|-----|---------------------|----------------------|-----------|---------------------|--------------------------------|
| 2026-05-05 | [9:00-18:00] | [10:00-16:00] | - | [10:00-16:00]（日期覆盖优先） | [10:00-16:00] |
| 2026-05-06 | [9:00-18:00] | - | [0-length] | [9:00-18:00] | []（请假被过滤） |

**关键机制**：
1. **日期覆盖**：通过对象展开顺序 `...workingHours, ...dateOverrides`，后者覆盖前者
2. **请假**：通过生成 0-length 范围，然后在 `filter` 步骤中被移除，实现"排除"效果

#### 2.2.3 日期覆盖的特殊用法：0-length 表示"当天不可用"

用户可以通过设置日期覆盖的 `startTime === endTime` 来表示某一天完全不可用：

```typescript
// 如果日期覆盖是 0-length，过滤后当天就没有可用时间
(ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
```
[packages/features/schedules/lib/date-ranges.ts:317](packages/features/schedules/lib/date-ranges.ts#L317)

### 2.3 阶段2详解：忙碌时间的获取与合并

**核心函数**: `packages/features/availability/lib/getUserAvailability.ts:584-630`

#### 2.3.1 忙碌时间来源与获取顺序

```typescript
// 1. 从日历和预订获取忙碌时间（包含缓冲时间）
let busyTimes = await busyTimesService.getBusyTimes({
  credentials: user.credentials,      // 用于获取日历忙碌时间
  selectedCalendars,                   // 用户选择的日历
  beforeEventBuffer,                   // 事件前缓冲
  afterEventBuffer,                    // 事件后缓冲
  seatedEvent: !!eventType?.seatsPerTimeSlot,  // 座位事件标志
  // ...
});

// 2. 获取个人限制导致的忙碌时间
let busyTimesFromLimits = await getBusyTimesFromLimits(
  bookingLimits,    // 预订数量限制
  durationLimits,   // 时长限制
  // ...
);

// 3. 获取团队限制导致的忙碌时间
let busyTimesFromTeamLimits = await getBusyTimesFromTeamLimits(
  user,
  teamBookingLimits,
  // ...
);

// 4. 合并所有忙碌时间
const detailedBusyTimesWithSource: EventBusyDetails[] = [
  ...busyTimes.map((a) => ({...})),     // 日历 + 预订 + 缓冲
  ...busyTimesFromLimits,               // 个人限制
  ...busyTimesFromTeamLimits,           // 团队限制
];
```
[packages/features/availability/lib/getUserAvailability.ts:584-630](packages/features/availability/lib/getUserAvailability.ts#L584-L630)

#### 2.3.2 座位事件的特殊处理

对于支持座位的事件类型（`seatsPerTimeSlot > 0`），忙碌时间的计算逻辑不同：

```typescript
// 在 getBusyTimes 中
if (
  bookingSeatCountMap[bookedAt] < (eventType?.seatsPerTimeSlot || 1) &&
  eventTypeId === eventType?.id
) {
  // 座位未满：只添加前后缓冲时间作为忙碌时间
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
  return aggregate;  // 主时间段不阻塞！
}
```
[packages/features/busyTimes/services/getBusyTimes.ts:143-167](packages/features/busyTimes/services/getBusyTimes.ts#L143-L167)

### 2.4 阶段3详解：统一裁剪 subtract()

**核心函数**: `packages/features/schedules/lib/date-ranges.ts:423`

#### 2.4.1 算法逻辑

```typescript
export function subtract<TSourceRange extends DateRange, TExcludedRange extends DateRange>(
  sourceRanges: TSourceRange[],      // 可用时间范围
  excludedRanges: TExcludedRange[]    // 要排除的忙碌时间
): SubtractedRange<TSourceRange>[] {
  const result: SubtractedRange<TSourceRange>[] = [];
  
  // 先排序排除范围，便于顺序处理
  const sortedExcludedRanges = [...excludedRanges].sort(
    (a, b) => a.start.valueOf() - b.start.valueOf()
  );

  for (const { start: sourceStart, end: sourceEnd, ...passThrough } of sourceRanges) {
    let currentStart = sourceStart;

    for (const excludedRange of sortedExcludedRanges) {
      // 排除范围完全在可用范围之后：跳过
      if (excludedRange.start.valueOf() >= sourceEnd.valueOf()) break;
      
      // 排除范围完全在可用范围之前：跳过
      if (excludedRange.end.valueOf() <= currentStart.valueOf()) continue;

      // 有重叠
      if (excludedRange.start.valueOf() > currentStart.valueOf()) {
        // 重叠前的部分：保留
        result.push({ start: currentStart, end: excludedRange.start, ...passThrough });
      }

      // 移动当前开始位置到排除范围之后
      if (excludedRange.end.valueOf() > currentStart.valueOf()) {
        currentStart = excludedRange.end;
      }
    }

    // 最后检查：如果还有剩余时间，保留
    if (sourceEnd.valueOf() > currentStart.valueOf()) {
      result.push({ start: currentStart, end: sourceEnd, ...passThrough });
    }
  }

  return result;
}
```
[packages/features/schedules/lib/date-ranges.ts:423-452](packages/features/schedules/lib/date-ranges.ts#L423-L452)

#### 2.4.2 实际例子

**场景**：
- 可用时间：[9:00-18:00]
- 忙碌时间：[10:00-11:00, 14:00-15:00]

**处理过程**：

```
初始: currentStart = 9:00

处理第一个 excludedRange [10:00-11:00]:
  - 10:00 > 9:00，保留 [9:00-10:00]
  - currentStart = 11:00

处理第二个 excludedRange [14:00-15:00]:
  - 14:00 > 11:00，保留 [11:00-14:00]
  - currentStart = 15:00

最后检查:
  - 18:00 > 15:00，保留 [15:00-18:00]

结果: [9:00-10:00, 11:00-14:00, 15:00-18:00]
```

---

## 3. 访客可见时段处理链路

### 3.1 整体流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     访客可见时段处理链路（按执行顺序）                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  输入：聚合后的可用时间范围（多主机时已处理交集/并集）                          │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段1：时间槽生成 - getSlots()                                          │  │
│  │                                                                       │  │
│  │  输入参数：                                                           │  │
│  │  ├── dateRanges: 聚合后的可用时间范围                                  │  │
│  │  ├── frequency: 槽位间隔（slotInterval 或事件时长）                   │  │
│  │  ├── eventLength: 事件时长                                            │  │
│  │  ├── minimumBookingNotice: 最小预订通知时间                            │  │
│  │  ├── offsetStart: 开始偏移量                                           │  │
│  │  ├── showOptimizedSlots: 是否优化槽位显示                              │  │
│  │  └── datesOutOfOffice: 请假日期（用于标记 away 状态）                  │  │
│  │                                                                       │  │
│  │  处理逻辑：                                                           │  │
│  │  1.1 按开始时间排序 dateRanges                                        │  │
│  │  1.2 对每个 range：                                                   │  │
│  │      ├── 计算有效开始时间：max(range.start, 当前时间 + minimumBookingNotice) │  │
│  │      ├── 时区转换：转换到访客时区后检查分钟对齐                        │  │
│  │      ├── 槽位起始时间校正：                                           │  │
│  │      │   ├── 优化模式：尽量让槽位从整点/15分/5分开始                 │  │
│  │      │   └── 普通模式：从小时开始，向上取整到间隔                     │  │
│  │      ├── 应用 offsetStart                                             │  │
│  │      └── 循环生成槽位：                                                │  │
│  │          while (slotStartTime + eventLength <= range.end) {          │  │
│  │              检查是否是请假日期 → 标记 away = true                    │  │
│  │              添加到 slots Map                                          │  │
│  │              slotStartTime += frequency + offsetStart                 │  │
│  │          }                                                            │  │
│  │                                                                       │  │
│  │  输出：                                                               │  │
│  │  └── timeSlots: 时间槽数组                                           │  │
│  │      [                                                                 │  │
│  │        { time: Dayjs, away?: boolean, fromUser?, toUser?, ... },    │  │
│  │        ...                                                             │  │
│  │      ]                                                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段2：限制计划过滤（Restriction Schedule）                            │  │
│  │                                                                       │  │
│  │  仅当 eventType.restrictionScheduleId 存在时执行：                    │  │
│  │                                                                       │  │
│  │  2.1 获取限制计划的可用时间范围                                        │  │
│  │      const { dateRanges: restrictionRanges } = buildDateRanges({   │  │
│  │          availability: restrictionSchedule.availability,            │  │
│  │          timeZone: restrictionTimezone,                              │  │
│  │          dateFrom: startTime,                                         │  │
│  │          dateTo: endTime,                                             │  │
│  │      });                                                              │  │
│  │                                                                       │  │
│  │  2.2 过滤时间槽：只保留在 restrictionRanges 内的槽位                  │  │
│  │      availableTimeSlots = timeSlots.filter((slot) => {              │  │
│  │          const slotStart = slot.time;                                │  │
│  │          const slotEnd = slot.time.add(eventLength, "minute");      │  │
│  │                                                                       │  │
│  │          return restrictionRanges.some(                             │  │
│  │              (range) =>                                              │  │
│  │                  (slotStart.isAfter(range.start) ||                 │  │
│  │                   slotStart.isSame(range.start)) &&                 │  │
│  │                  (slotEnd.isBefore(range.end) ||                     │  │
│  │                   slotEnd.isSame(range.end))                         │  │
│  │          );                                                           │  │
│  │      });                                                              │  │
│  │                                                                       │  │
│  │  输出：过滤后的 availableTimeSlots                                    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段3：预留槽位冲突检查（Reserved Slots）                             │  │
│  │                                                                       │  │
│  │  预留槽位：其他访客临时"锁定"的槽位（默认 5 分钟）                    │  │
│  │                                                                       │  │
│  │  3.1 获取预留槽位                                                    │  │
│  │      const reservedSlots = await this._getReservedSlotsAndCleanupExpired({ │  │
│  │          bookerClientUid,       // 当前访客的 UID（排除自己的预留）   │  │
│  │          eventTypeId,                                                  │  │
│  │          usersWithCredentials,                                         │  │
│  │      });                                                              │  │
│  │                                                                       │  │
│  │  3.2 区分座位事件和普通事件                                          │  │
│  │      - 座位事件：只处理 isSeat = true 的预留，更新座位计数           │  │
│  │      - 普通事件：所有预留都视为冲突                                   │  │
│  │                                                                       │  │
│  │  3.3 冲突检查                                                        │  │
│  │      const busySlotsFromReservedSlots = reservedSlots.reduce(      │  │
│  │          (r, c) => {                                                 │  │
│  │              if (!c.isSeat) {  // 非座位预留，视为忙碌              │  │
│  │                  r.push({                                            │  │
│  │                      start: c.slotUtcStartDate,                      │  │
│  │                      end: c.slotUtcEndDate,                          │  │
│  │                  });                                                  │  │
│  │              }                                                        │  │
│  │              return r;                                                │  │
│  │          },                                                           │  │
│  │          []                                                           │  │
│  │      );                                                               │  │
│  │                                                                       │  │
│  │  3.4 过滤冲突槽位                                                    │  │
│  │      availableTimeSlots = availableTimeSlots                         │  │
│  │          .map((slot) => {                                            │  │
│  │              if (!checkForConflicts({                                │  │
│  │                      time: slot.time,                                 │  │
│  │                      busy: busySlotsFromReservedSlots,               │  │
│  │                      eventLength,                                     │  │
│  │                      currentSeats,    // 座位事件的当前座位数        │  │
│  │                  })) {                                                │  │
│  │                      return slot;  // 无冲突，保留                   │  │
│  │              }                                                        │  │
│  │              return undefined;  // 有冲突，过滤                       │  │
│  │          })                                                           │  │
│  │          .filter((item): item is {...} => !!item);                  │  │
│  │                                                                       │  │
│  │  输出：过滤冲突后的 availableTimeSlots                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段4：期限检查（Bounds Check）                                       │  │
│  │                                                                       │  │
│  │  4.1 计算期限限制                                                    │  │
│  │      const periodLimits = calculatePeriodLimits({                    │  │
│  │          periodType: eventType.periodType,     // ROLLING, ROLLING_WINDOW, etc. │  │
│  │          periodDays: eventType.periodDays,       // 滚动天数         │  │
│  │          periodStartDate: eventType.periodStartDate,                │  │
│  │          periodEndDate: eventType.periodEndDate,                    │  │
│  │          eventUtcOffset,                                              │  │
│  │          bookerUtcOffset,                                             │  │
│  │      });                                                              │  │
│  │                                                                       │  │
│  │  4.2 逐个槽位检查                                                    │  │
│  │      for (const [date, slots] of Object.entries(slotsMappedToDate)) { │  │
│  │          const filteredSlots = slots.filter((slot) => {             │  │
│  │                                                                       │  │
│  │              // 检查4.2.1：未来限制（滚动窗口等）                     │  │
│  │              const isFutureLimitViolation = isTimeViolatingFutureLimit({ │  │
│  │                  time: slot.time,                                     │  │
│  │                  periodLimits,                                         │  │
│  │              });                                                       │  │
│  │                                                                       │  │
│  │              // 检查4.2.2：过期时间和最小提前预约                     │  │
│  │              let isOutOfBounds = false;                               │  │
│  │              try {                                                    │  │
│  │                  isOutOfBounds = isTimeOutOfBounds({                 │  │
│  │                      time: slot.time,                                 │  │
│  │                      minimumBookingNotice: eventType.minimumBookingNotice, │  │
│  │                  });                                                   │  │
│  │              } catch (error) {                                        │  │
│  │                  if (error instanceof BookingDateInPastError) {      │  │
│  │                      throw new TRPCError({  // 明确的过去日期错误    │  │
│  │                          code: "BAD_REQUEST",                         │  │
│  │                          message: error.message,                      │  │
│  │                      });                                               │  │
│  │                  }                                                     │  │
│  │                  throw error;                                          │  │
│  │              }                                                         │  │
│  │                                                                       │  │
│  │              // 滚动窗口模式：找到第一个违规后，后续所有日期都跳过     │  │
│  │              if (isFutureLimitViolation && doesStartFromToday) {     │  │
│  │                  foundAFutureLimitViolation = true;                   │  │
│  │              }                                                         │  │
│  │                                                                       │  │
│  │              return (                                                  │  │
│  │                  !isFutureLimitViolation &&                           │  │
│  │                  !isOutOfBounds                                        │  │
│  │              );                                                        │  │
│  │          });                                                           │  │
│  │      }                                                                 │  │
│  │                                                                       │  │
│  │  输出：过滤后的 withinBoundsSlotsMappedToDate                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段5：请求范围过滤                                                    │  │
│  │                                                                       │  │
│  │  只保留访客请求的时间范围内的槽位：                                    │  │
│  │                                                                       │  │
│  │  const filteredSlotsMappedToDate = this.filterSlotsByRequestedDateRange({ │  │
│  │      slotsMappedToDate: withinBoundsSlotsMappedToDate,               │  │
│  │      startTime: input.startTime,                                       │  │
│  │      endTime: input.endTime,                                           │  │
│  │      timeZone: input.timeZone,                                         │  │
│  │  });                                                                   │  │
│  │                                                                       │  │
│  │  原因：buildDateRanges 在处理时区边界时使用了 ±1 天的缓冲区，         │  │
│  │        可能导致相邻日期的槽位泄漏到响应中                              │  │
│  │                                                                       │  │
│  │  输出：最终可见的时间槽                                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  最终返回：                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  {                                                                     │  │
│  │    slots: {                                                           │  │
│  │      "2026-05-05": [                                                 │  │
│  │          { time: "2026-05-05T10:00:00Z", ... },                     │  │
│  │          { time: "2026-05-05T11:00:00Z", ... },                     │  │
│  │          ...                                                           │  │
│  │      ],                                                                │  │
│  │      "2026-05-06": [...],                                             │  │
│  │    }                                                                   │  │
│  │  }                                                                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 阶段1详解：时间槽生成 getSlots()

**核心函数**: `packages/features/schedules/lib/slots.ts:232`

#### 3.2.1 最小预订通知的处理

最小预订通知在两个地方被处理：

**处理1：在时间槽生成时**
```typescript
const startTimeWithMinNotice = dayjs.utc().add(minimumBookingNotice, "minute");

orderedDateRanges.forEach((range) => {
  // 有效开始时间 = max(范围开始时间, 当前时间 + 最小通知时间)
  let slotStartTime = range.start.utc().isAfter(startTimeWithMinNotice)
    ? range.start
    : startTimeWithMinNotice;
  // ...
});
```
[packages/features/schedules/lib/slots.ts:123-130](packages/features/schedules/lib/slots.ts#L123-L130)

**处理2：在最终过滤时（双重保险）**
```typescript
isOutOfBounds = isTimeOutOfBounds({
  time: slot.time,
  minimumBookingNotice: eventType.minimumBookingNotice,
});
```
[packages/trpc/server/routers/viewer/slots/util.ts:1343](packages/trpc/server/routers/viewer/slots/util.ts#L1343)

#### 3.2.2 槽位起始时间校正

**优化模式**（`showOptimizedSlots = true`）：
```typescript
if (showOptimizedSlots) {
  // 计算需要多少分钟才能推到下一个"整齐"的开始时间
  const minutesRequiredToMoveToNextSlot = interval - (slotStartTime.minute() % interval);
  const minutesRequiredToMoveTo15MinSlot = 15 - (slotStartTime.minute() % 15);
  const minutesRequiredToMoveTo5MinSlot = 5 - (slotStartTime.minute() % 5);
  
  // 检查范围结束前是否有足够的"额外时间"来推
  const extraMinutesAvailable = range.end.diff(slotStartTime, "minutes") % interval;

  if (extraMinutesAvailable >= minutesRequiredToMoveToNextSlot) {
    // 推到下一个完整间隔（比如 60 分钟事件从 9:05 推到 10:00）
    correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveToNextSlot, "minute");
  } else if (extraMinutesAvailable >= minutesRequiredToMoveTo15MinSlot) {
    // 推到下一个 15 分钟刻度
    correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveTo15MinSlot, "minute");
  } else if (extraMinutesAvailable >= minutesRequiredToMoveTo5MinSlot) {
    // 推到下一个 5 分钟刻度
    correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveTo5MinSlot, "minute");
  }
  return correctedSlotStartTime;
}
```
[packages/features/schedules/lib/slots.ts:38-66](packages/features/schedules/lib/slots.ts#L38-L66)

**普通模式**：
```typescript
// 从小时开始，向上取整到间隔
return slotStartTime.startOf("hour").add(Math.ceil(slotStartTime.minute() / interval) * interval, "minute");
```
[packages/features/schedules/lib/slots.ts:68](packages/features/schedules/lib/slots.ts#L68)

### 3.3 阶段4详解：期限检查

#### 3.3.1 isTimeOutOfBounds - 过期时间和最小提前预约

**核心函数**: `packages/lib/isOutOfBounds.ts`

```typescript
// 检查逻辑（推测，基于调用方式）
function isTimeOutOfBounds({
  time: Dayjs,
  minimumBookingNotice: number,
}): boolean {
  const now = dayjs.utc();
  const earliestAllowed = now.add(minimumBookingNotice, "minute");
  
  // 如果时间在最早允许时间之前，返回 true（表示越界）
  return time.isBefore(earliestAllowed);
}
```

#### 3.3.2 滚动窗口模式的特殊处理

```typescript
// ROLLING 或 ROLLING_WINDOW 模式
const doesStartFromToday = this.doesRangeStartFromToday(eventType.periodType);

if (foundAFutureLimitViolation && doesStartFromToday) {
  break;  // 找到第一个违规后，后续所有日期都跳过
}
```
[packages/trpc/server/routers/viewer/slots/util.ts:1331-1333](packages/trpc/server/routers/viewer/slots/util.ts#L1331-L1333)

---

## 4. 边界情况与实际例子

### 4.1 例子1：日期覆盖 vs 常规工作时间

**场景**：
- 常规工作时间：周一至周五 9:00-18:00
- 日期覆盖：2026-05-05（周一）设置为 10:00-16:00

**处理流程**：
1. `groupedWorkingHours["2026-05-05"]` = `[9:00-18:00]`
2. `groupedDateOverrides["2026-05-05"]` = `[10:00-16:00]`
3. 合并时：`{ ...workingHours, ...dateOverrides }` → 后者覆盖前者
4. 结果：`dateRanges["2026-05-05"]` = `[10:00-16:00]`

**关键点**：日期覆盖是**完全替换**当天的工作时间，不是叠加。

### 4.2 例子2：请假的排除机制

**场景**：
- 常规工作时间：2026-05-06（周二）9:00-18:00
- 请假：2026-05-06

**处理流程**：
1. `groupedWorkingHours["2026-05-06"]` = `[9:00-18:00]`
2. `groupedOOO["2026-05-06"]` = `[{ start: 2026-05-06, end: 2026-05-06 }]`（0-length）
3. 合并时：`{ ...workingHours, ...ooo }` → ooo 覆盖
4. 过滤时：`filter((range) => range.start !== range.end)` → 0-length 被过滤
5. 结果：`oooExcludedDateRanges["2026-05-06"]` = `[]`

**关键点**：请假是通过**0-length 覆盖 + 过滤**实现的，不是通过 subtract。

### 4.3 例子3：忙碌时间统一裁剪

**场景**：
- 可用时间：2026-05-05 9:00-18:00
- 日历忙碌：10:00-11:00（外部会议）
- 已接受预订：14:00-15:00
- 缓冲时间：每个事件前后各 15 分钟

**处理流程**：
1. `busyTimes` 包含：
   - 日历事件 + 缓冲：9:45-11:15
   - 预订 + 缓冲：13:45-15:15
2. `subtract([9:00-18:00], [9:45-11:15, 13:45-15:15])`
3. 结果：
   - 9:00-9:45（第一个忙碌前的部分）
   - 11:15-13:45（两个忙碌之间的部分）
   - 15:15-18:00（最后一个忙碌后的部分）

### 4.4 例子4：座位事件的特殊处理

**场景**：
- 事件类型：支持座位，`seatsPerTimeSlot = 5`
- 当前预订：同一时间槽已有 3 个预订
- 新访客尝试预订

**处理流程**：
1. 检查座位数：3 < 5 → 座位未满
2. 忙碌时间只包含：
   - 预订前缓冲：13:45-14:00
   - 预订后缓冲：15:00-15:15
3. 主时间段 14:00-15:00 **不阻塞**
4. 结果：访客可以预订 14:00-15:00，座位数变为 4

### 4.5 例子5：最小预订通知

**场景**：
- 当前时间：2026-05-05 9:30
- 最小预订通知：60 分钟
- 可用时间：9:00-18:00，60 分钟槽位

**处理流程**：
1. `startTimeWithMinNotice` = 9:30 + 60 分钟 = 10:30
2. 第一个有效槽位开始时间 = max(9:00, 10:30) = 10:30
3. 槽位起始校正（假设 60 分钟间隔）：
   - 10:30 分钟部分是 30，不是 60 的倍数
   - 向上取整到 11:00
4. 结果：第一个可见槽位是 11:00-12:00

### 4.6 例子6：预留槽位冲突

**场景**：
- 访客 A 在 9:00 打开预订页面，选择了 14:00 的槽位
- 访客 B 在 9:01 打开同一事件的预订页面

**处理流程**（访客 B 的视角）：
1. 时间槽生成：14:00 是可用的
2. 获取预留槽位：发现访客 A 在 9:00 预留了 14:00-15:00
3. `busySlotsFromReservedSlots` = `[14:00-15:00]`
4. `checkForConflicts` 检查 14:00 是否冲突 → 是
5. 过滤：14:00 槽位被移除
6. 结果：访客 B 看不到 14:00 的槽位

**关键点**：预留槽位的有效期默认是 5 分钟（`DEFAULT_RESERVATION_DURATION = 5`）。

### 4.7 例子7：跨天工作时间

**场景**：
- 工作时间设置：22:00-02:00（跨越午夜）
- UTC 存储：startTime=22:00 UTC, endTime=02:00 UTC

**处理流程**（在 `getWorkingHours` 中）：
```typescript
// 转换为分钟数
const startTime = 22 * 60 + 0 = 1320;  // 22:00
const endTime = 2 * 60 + 0 = 120;       // 02:00

// 检查是否跨天
if (startTime > MINUTES_DAY_END || endTime > MINUTES_IN_DAY) {
  // 溢出到第二天
  const newWorkingHours: WorkingHours = {
    days: schedule.days.map((day) => (day + 1) % 7),  // 星期几 +1
    startTime: Math.max(startTime - MINUTES_IN_DAY, MINUTES_DAY_START),  // 1320-1440 = -120 → max(-120, 0) = 0
    endTime: endTime - MINUTES_IN_DAY,  // 120-1440 = -1320
  };
  // ...
}
```
[packages/lib/availability.ts:112-120](packages/lib/availability.ts#L112-L120)

**结果**：22:00-02:00 被拆分为：
- 当天 22:00-24:00
- 第二天 00:00-02:00

### 4.8 例子8：时区边界（半小时偏移时区）

**场景**：
- 主机时区：Asia/Kolkata（GMT+5:30）
- 工作时间：9:00-18:00（当地时间）
- UTC 存储：startTime=03:30 UTC, endTime=12:30 UTC
- 访客时区：UTC

**处理流程**：
1. 在 `buildDateRanges` 中转换时区
2. 在 `getSlots` 中再次转换到访客时区检查分钟对齐：
   ```typescript
   // 转换到目标时区 BEFORE 检查是否需要取整
   slotStartTime = slotStartTime.tz(timeZone);
   
   if (slotStartTime.minute() % interval !== 0) {
     slotStartTime = getCorrectedSlotStartTime({...});
   }
   ```
   [packages/features/schedules/lib/slots.ts:138-147](packages/features/schedules/lib/slots.ts#L138-L147)

**关键点**：分钟对齐检查是在**访客时区**中进行的，避免了像 Asia/Kolkata 这样的半小时偏移时区的问题。

---

## 5. 关键函数与文件引用

### 5.1 主机侧核心函数

| 函数名 | 文件路径 | 功能 |
|-------|---------|------|
| `buildDateRanges` | `packages/features/schedules/lib/date-ranges.ts:226` | 构建基础日期范围，处理工作时间、日期覆盖、请假 |
| `getWorkingHours` | `packages/lib/availability.ts:61` | 将 UTC 工作时间转换为目标时区 |
| `subtract` | `packages/features/schedules/lib/date-ranges.ts:423` | 从源范围中减去排除范围 |
| `getUserAvailability` | `packages/features/availability/lib/getUserAvailability.ts:361` | 获取用户完整可用性（工作时间 - 忙碌时间） |
| `getBusyTimes` | `packages/features/busyTimes/services/getBusyTimes.ts:29` | 获取日历和预订的忙碌时间 |

### 5.2 访客侧核心函数

| 函数名 | 文件路径 | 功能 |
|-------|---------|------|
| `getSlots` | `packages/features/schedules/lib/slots.ts:232` | 从可用范围生成时间槽 |
| `_getAvailableSlots` | `packages/trpc/server/routers/viewer/slots/util.ts:897` | 主入口，完整的时间槽计算流程 |
| `isTimeOutOfBounds` | `packages/lib/isOutOfBounds.ts` | 检查过期时间和最小提前预约 |
| `isTimeViolatingFutureLimit` | `packages/lib/isOutOfBounds.ts` | 检查未来限制（滚动窗口等） |
| `filterSlotsByRequestedDateRange` | `packages/trpc/server/routers/viewer/slots/util.ts:274` | 按请求范围过滤槽位 |

### 5.3 关键数据结构

```typescript
// 日期范围
type DateRange = {
  start: Dayjs;
  end: Dayjs;
};

// 时间槽
type TimeSlot = {
  time: Dayjs;
  userIds?: number[];
  away?: boolean;           // 请假标记
  fromUser?: IFromUser;     // 请假发起者
  toUser?: IToUser;         // 请假接收者
  reason?: string;          // 请假原因
  emoji?: string;           // 请假表情
};

// 忙碌时间详情
type EventBusyDetails = {
  start: Date | string;
  end: Date | string;
  title?: string;
  source?: string;          // 来源标识
  userId?: number;
};
```

---

*报告生成日期: 2026-05-02*

*基于代码分析，按实际执行顺序整理*
