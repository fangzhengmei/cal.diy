# 预约系统时间槽引擎技术分析报告（完整评审版）

> 版本：v2.0  
> 日期：2026-05-03  
> 状态：可评审

---

## 目录

1. [执行摘要](#1-执行摘要)
2. [整体架构与数据流](#2-整体架构与数据流)
3. [可用时间构建链路详解](#3-可用时间构建链路详解)
4. [排除规则叠加机制](#4-排除规则叠加机制)
5. [时间槽生成算法](#5-时间槽生成算法)
6. [冲突检测与收束机制](#6-冲突检测与收束机制)
7. [边界场景分析](#7-边界场景分析)
8. [规则优先级判定依据](#8-规则优先级判定依据)
9. [多用户聚合策略](#9-多用户聚合策略)
10. [性能优化与缓存策略](#10-性能优化与缓存策略)
11. [测试覆盖分析](#11-测试覆盖分析)
12. [常见问题排查](#12-常见问题排查)
13. [附录](#13-附录)

---

## 1. 执行摘要

### 1.1 核心问题

本报告旨在回答以下关键问题：

1. **可用时间如何构建？** 工作时间、日期覆盖、外出等规则如何协作生成基础可用范围？
2. **排除规则如何叠加？** 日历忙碌、现有预约、缓冲时间、预订限制等如何影响最终可用性？
3. **冲突如何收束？** 从输入到输出，冲突检测的完整链路是什么？
4. **边界场景如何处理？** 时区差异、DST变更、跨午夜可用性等特殊情况的判定依据是什么？

### 1.2 关键发现

| 发现项 | 关键结论 | 代码位置 |
|--------|----------|----------|
| **规则优先级** | DateOverride 通过对象展开完全覆盖 WorkingHours | `date-ranges.ts:312-318` |
| **冲突模型** | 采用开放区间 `[start, end)`，边界接触不视为冲突 | `checkForConflicts.ts:39-46` |
| **OOO处理** | OOO 在 `getSlots()` 中标记为 `away: true`，但不影响可用范围生成 | `slots.ts:187-222` |
| **时区对齐** | 分钟对齐在主机时区计算，而非 UTC | `slots.ts:135-147` |
| **预订限制** | 标记整个周期为"忙碌"，而非精确控制每个时间槽 | `getBusyTimesFromLimits` |

### 1.3 数据流简图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                     输入参数                                          │
│  eventTypeId, startTime, endTime, timeZone, duration, before/afterBuffer           │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          阶段一：可用时间构建                                         │
│                                                                                       │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────────────────┐   │
│  │ WorkingHours    │ ──▶ │ processWorking  │ ──▶ │ groupedWorkingHours         │   │
│  │ (周期性规则)     │     │ Hours()         │     │ (按日期分组的工作时间范围)  │   │
│  └─────────────────┘     └─────────────────┘     └─────────────────────────────┘   │
│                                                                                       │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────────────────┐   │
│  │ DateOverride    │ ──▶ │ processDate     │ ──▶ │ groupedDateOverrides         │   │
│  │ (特定日期覆盖)   │     │ Override()      │     │ (按日期分组的覆盖范围)       │   │
│  └─────────────────┘     └─────────────────┘     └─────────────────────────────┘   │
│                                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │ 对象展开合并：                                                                 │   │
│  │   dateRanges = { ...groupedWorkingHours, ...groupedDateOverrides }          │   │
│  │   注意：后者覆盖前者！                                                          │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          阶段二：排除规则叠加                                         │
│                                                                                       │
│  收集所有忙碌时间来源：                                                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐│
│  │ 日历集成        │  │ 现有预约        │  │ 前后缓冲        │  │ 预订限制        ││
│  │ freebusy        │  │ Bookings        │  │ bufferTimes     │  │ bookingLimits   ││
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘│
│           │                    │                    │                    │          │
│           └────────────────────┼────────────────────┼────────────────────┘          │
│                                │                    │                               │
│                                ▼                    ▼                               │
│                    ┌─────────────────────────────────────────┐                     │
│                    │         mergedBusyTimes                 │                     │
│                    │      (所有忙碌时间的并集)                │                     │
│                    └────────────────────┬────────────────────┘                     │
│                                         │                                             │
│                                         ▼                                             │
│                    ┌─────────────────────────────────────────┐                     │
│                    │         subtract()                      │                     │
│                    │    可用范围 - 忙碌范围 = 净可用范围      │                     │
│                    └─────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          阶段三：多用户聚合（如适用）                                 │
│                                                                                       │
│  根据 schedulingType 选择策略：                                                       │
│                                                                                       │
│  ┌─────────────────┐          ┌─────────────────────────────────────────┐          │
│  │ COLLECTIVE      │ ───────▶ │         intersect()                     │          │
│  │ (集体预约)       │          │    所有用户范围的交集                   │          │
│  └─────────────────┘          └─────────────────────────────────────────┘          │
│                                                                                       │
│  ┌─────────────────┐          ┌─────────────────────────────────────────┐          │
│  │ ROUND_ROBIN     │ ───────▶ │     保留各用户独立范围                  │          │
│  │ MANAGED         │          │     预约时选择可用主机                  │          │
│  └─────────────────┘          └─────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          阶段四：时间槽生成                                           │
│                                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │ getSlots() 核心逻辑：                                                          │   │
│  │                                                                                │   │
│  │ 1. 频率对齐：根据 frequency 选择 interval (60/30/20/15/10/5/1)             │   │
│  │ 2. 时区转换：转换到目标时区后计算分钟对齐                                      │   │
│  │ 3. 最小预订通知：从当前时间 + minimumBookingNotice 开始                      │   │
│  │ 4. 重叠范围处理：避免生成重复槽位                                             │   │
│  │ 5. OOO标记：检查 datesOutOfOffice，标记 away: true                          │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          阶段五：后处理与冲突收束                                     │
│                                                                                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐│
│  │ 预留槽位检查    │  │ 座位事件处理    │  │ 日期边界过滤    │  │ 未来预订限制    ││
│  │ reservedSlots   │  │ currentSeats    │  │ dateFrom/dateTo │  │ futureBookings  ││
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘│
│           │                    │                    │                    │          │
│           └────────────────────┼────────────────────┼────────────────────┘          │
│                                │                    │                               │
│                                ▼                    ▼                               │
│                    ┌─────────────────────────────────────────┐                     │
│                    │         最终可用时间槽                   │                     │
│                    │      (按日期分组的 slots 数组)           │                     │
│                    └─────────────────────────────────────────┘                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 整体架构与数据流

### 2.1 模块层次结构

时间槽引擎采用清晰的四层架构：

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Layer 4: 服务编排层                                      │
│  AvailableSlotsService (packages/trpc/server/routers/viewer/slots/util.ts)      │
│  - 整合所有数据源                                                                 │
│  - 多主机聚合与路由                                                                │
│  - 预留槽位与座位管理                                                              │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Layer 3: 用户可用性层                                    │
│  UserAvailabilityService (packages/features/availability/lib/getUserAvailability.ts) │
│  - 单用户完整可用性计算                                                            │
│  - 工作时间 + 日期覆盖处理                                                         │
│  - 日历忙碌时间获取                                                                 │
│  - 预订限制与时长限制处理                                                           │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Layer 2: 日期范围计算层                                   │
│  date-ranges.ts (packages/features/schedules/lib/)                                │
│  - buildDateRanges(): 构建基础日期范围                                             │
│  - intersect(): 多范围交集计算                                                     │
│  - subtract(): 范围差集计算                                                       │
│  - processWorkingHours/processDateOverride: 单规则处理                            │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Layer 1: 时间槽生成层                                     │
│  slots.ts (packages/features/schedules/lib/)                                      │
│  - getSlots(): 从日期范围生成具体时间槽                                           │
│  - getCorrectedSlotStartTime(): 槽位开始时间对齐                                  │
│  - buildSlotsWithDateRanges(): 实际槽位构建                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 核心入口函数

时间槽生成的完整入口在 `AvailableSlotsService._getAvailableSlots()`：

```typescript
// packages/trpc/server/routers/viewer/slots/util.ts (示意)
async _getAvailableSlots({
  eventType,
  startTime,
  endTime,
  timeZone,
  duration,
  rescheduleUid,
  bookingUid,
  isInstantMeeting,
  reqUid,
  bookerClientUid,
}: GetScheduleOptions) {
  
  // 步骤1: 获取主机列表
  const { usersWithCredentials, dynamicUserNames } = await this._calculateHostsAndAvailabilities({...});
  
  // 步骤2: 收集忙碌时间（所有主机）
  const currentSeats = await this._getCurrentSeats({...});
  const reservedSlots = await this._getReservedSlotsAndCleanupExpired({...});
  
  // 步骤3: 为每个用户计算可用性
  const userAvailabilityPromises = usersWithCredentials.map(async (user) => {
    return this.dependencies.userAvailabilityService._getUserAvailability({...});
  });
  
  // 步骤4: 聚合多用户可用性
  const aggregatedAvailability = await this._getAggregatedAvailability({
    userAvailabilities,
    eventType,
    currentSeats,
  });
  
  // 步骤5: 生成时间槽
  const slots = getSlots({
    inviteeDate: dayjs.tz(startTime, timeZone),
    frequency: eventType.slotInterval ?? eventType.length,
    dateRanges: aggregatedAvailability.dateRanges,
    minimumBookingNotice,
    eventLength: duration || eventType.length,
    datesOutOfOffice: aggregatedAvailability.datesOutOfOffice,
    showOptimizedSlots: eventType.optimizeBookingsLayout,
    datesOutOfOfficeTimeZone: usersTimeZone,
  });
  
  // 步骤6: 后处理过滤
  const availableTimeSlots = await this._applyPostProcessingFilters({
    slots,
    reservedSlots,
    eventType,
    usersWithCredentials,
    // ...
  });
  
  return { slots: availableTimeSlots };
}
```

### 2.3 数据流状态转换

从输入到输出，数据经历以下关键转换：

| 阶段 | 输入 | 输出 | 关键操作 |
|------|------|------|----------|
| **P0: 原始配置** | `eventType`, `user.availability` | `WorkingHours[]`, `DateOverride[]` | 解析数据库配置 |
| **P1: 规则展开** | `WorkingHours[]`, `DateOverride[]` | `DateRange[]` | `buildDateRanges()` |
| **P2: 忙碌收集** | `calendar.freebusy`, `bookings`, `limits` | `EventBusyDetails[]` | `getBusyTimes()` |
| **P3: 净可用** | `DateRange[]` - `EventBusyDetails[]` | `DateRange[]` | `subtract()` |
| **P4: 多用户** | `DateRange[][]` | `DateRange[]` | `intersect()` 或保留独立 |
| **P5: 槽位化** | `DateRange[]` + `frequency` | `Slot[]` | `getSlots()` |
| **P6: 后过滤** | `Slot[]` | `Slot[]` | 冲突检查、座位验证 |

---

## 3. 可用时间构建链路详解

### 3.1 buildDateRanges 核心机制

`buildDateRanges()` 是可用时间构建的核心函数，其关键设计是**对象展开覆盖**：

```typescript
// packages/features/schedules/lib/date-ranges.ts:312-330
export function buildDateRanges({
  availability,
  timeZone,
  dateFrom,
  dateTo,
  travelSchedules,
  outOfOffice,
}): { 
  dateRanges: DateRange[]; 
  oooExcludedDateRanges: DateRange[] 
} {
  
  // 步骤1: 处理工作时间
  const groupedWorkingHours = groupByDate(
    Object.values(
      availability.reduce((processed: Record<number, DateRange>, item) => {
        if (!("days" in item)) return processed;  // 跳过 DateOverride
        return processWorkingHours(processed, {...});
      }, {})
    )
  );
  
  // 步骤2: 处理 OOO（用于 oooExcludedDateRanges）
  const groupedOOO = groupByDate(
    outOfOffice
      ? Object.keys(outOfOffice).map((ooo) => processOOO(dayjs.utc(ooo), timeZone))
      : []
  );
  
  // 步骤3: 处理日期覆盖
  const groupedDateOverrides = groupByDate(
    Object.values(
      availability.reduce((processed: Record<number, DateRange>, item) => {
        if (!("date" in item && !!item.date)) return processed;  // 跳过 WorkingHours
        
        // 关键：±1 天缓冲处理时区边界
        if (itemDateAsUtc.isBetween(
          dateFrom.subtract(1, "day").startOf("day"),
          dateTo.add(1, "day").endOf("day"),
          null,
          "[]"
        )) {
          const newRange = processDateOverride({...});
          processed[newRange.end.valueOf()] = newRange;
        }
        return processed;
      }, {})
    )
  );
  
  // 步骤4: 关键合并逻辑 - 对象展开覆盖
  const dateRanges = Object.values({
    ...groupedWorkingHours,    // 先展开工作时间
    ...groupedDateOverrides,    // 后展开日期覆盖（会覆盖同日期的工作时间）
  }).map(
    // 过滤掉空范围（start === end 表示该天不可用）
    (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
  );
  
  // 步骤5: 构建包含 OOO 的范围（用于某些特殊场景）
  const oooExcludedDateRanges = Object.values({
    ...groupedWorkingHours,
    ...groupedDateOverrides,
    ...groupedOOO,  // OOO 也会覆盖
  }).map(
    (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
  );
  
  return { 
    dateRanges: dateRanges.flat(), 
    oooExcludedDateRanges: oooExcludedDateRanges.flat() 
  };
}
```

### 3.2 规则覆盖的精确机制

**核心设计：JavaScript 对象展开的属性覆盖**

```typescript
const dateRanges = Object.values({
  ...groupedWorkingHours,    // 键：日期字符串 "2026-05-03"
  ...groupedDateOverrides,    // 键：日期字符串 "2026-05-03"
});
```

**覆盖原理**：
1. `groupedWorkingHours` 和 `groupedDateOverrides` 都是以**日期字符串**为键的对象
2. 当两者有相同的键（同一天）时，后展开的 `groupedDateOverrides` 会**完全覆盖**前者
3. 这意味着：**对于同一天，DateOverride 会完全替换 WorkingHours**

### 3.3 覆盖场景示例

#### 场景 A：完全覆盖（正常工作时间 vs 特殊时间）

```
配置：
  WorkingHours: 周一至周五 09:00-17:00
  DateOverride: 2026-05-03 (周一) 10:00-15:00

构建过程：
  groupedWorkingHours["2026-05-03"] = [{ start: 09:00, end: 17:00 }]
  groupedDateOverrides["2026-05-03"] = [{ start: 10:00, end: 15:00 }]
  
  对象展开后：
    { "2026-05-03": [{ start: 10:00, end: 15:00 }] }  // 日期覆盖胜出

结果：
  2026-05-03 可用时间：10:00-15:00（不是 09:00-17:00）
```

#### 场景 B：完全不可用（空范围覆盖）

```
配置：
  WorkingHours: 周一至周五 09:00-17:00
  DateOverride: 2026-05-03 (周一) 00:00-00:00（start === end）

构建过程：
  groupedWorkingHours["2026-05-03"] = [{ start: 09:00, end: 17:00 }]
  groupedDateOverrides["2026-05-03"] = [{ start: 00:00, end: 00:00 }]
  
  对象展开后：
    { "2026-05-03": [{ start: 00:00, end: 00:00 }] }
  
  过滤阶段（start !== end）：
    [{ start: 00:00, end: 00:00 }].filter(r => r.start !== r.end) = []

结果：
  2026-05-03 完全不可用（无时间槽）
```

#### 场景 C：无覆盖（日期覆盖在请求范围外）

```
配置：
  WorkingHours: 周一至周五 09:00-17:00
  DateOverride: 2026-05-10 (下周一) 10:00-15:00

请求范围：2026-05-03 至 2026-05-07

构建过程：
  groupedWorkingHours: {
    "2026-05-03": [...],
    "2026-05-04": [...],
    "2026-05-05": [...],
    "2026-05-06": [...],
    "2026-05-07": [...],
  }
  groupedDateOverrides: {}  // 2026-05-10 不在请求范围内
  
  对象展开后：
    只有工作时间的键

结果：
  所有日期都使用常规工作时间
```

### 3.4 日期覆盖的 ±1 天缓冲

在 `buildDateRanges()` 中，日期覆盖的匹配使用了 ±1 天缓冲：

```typescript
// packages/features/schedules/lib/date-ranges.ts:282-290
// TODO: Remove the .subtract(1, "day") and .add(1, "day") part
// As of 2024-02-20, there are mismatches between local and UTC dates for overrides
if (
  itemDateAsUtc.isBetween(
    dateFrom.subtract(1, "day").startOf("day"),
    dateTo.add(1, "day").endOf("day"),
    null,
    "[]"
  )
) {
  // 处理该日期覆盖
}
```

**问题背景**：
- 时区差异可能导致日期计算偏移一天
- 例如：主机在 UTC+12（斐济），预订者在 UTC-12（贝克岛）
- 同一 UTC 时刻在不同时区可能是不同的日期

**判定依据**：
- 实际查询范围：`[dateFrom-1d, dateTo+1d]`
- 这是一个**防御性设计**，确保不会因为时区边界问题漏掉日期覆盖
- 后续在 `processDateOverride()` 中会精确计算实际时间范围

### 3.5 processWorkingHours 详解

`processWorkingHours()` 负责将周期性工作时间规则展开为具体的日期范围：

```typescript
// packages/features/schedules/lib/date-ranges.ts:32-173
export function processWorkingHours(
  results: Record<number, DateRange>,
  { item, timeZone, dateFrom, dateTo, travelSchedules }
) {
  const utcDateTo = dateTo.utc();
  let endTimeToKeyMap: Map<number, number[]> | undefined;

  // 遍历日期范围内的每一天
  for (let date = dateFrom.startOf("day"); utcDateTo.isAfter(date); date = date.add(1, "day")) {
    
    // 步骤1: 计算当天的时区（考虑旅行日程）
    const adjustedTimezone = getAdjustedTimezone(date, timeZone, travelSchedules);
    
    // 步骤2: 计算日期在目标时区的偏移
    const fromOffset = dateFrom.startOf("day").utcOffset();
    const offset = date.tz(adjustedTimezone).utcOffset();
    
    // 步骤3: 调整日期到时区
    const dateInTz = date.add(fromOffset - offset, "minutes").tz(adjustedTimezone);
    
    // 步骤4: 检查星期几是否匹配
    if (!item.days.includes(dateInTz.day())) {
      continue;
    }
    
    // 步骤5: 计算开始/结束时间
    let start = dateInTz
      .add(item.startTime.getUTCHours(), "hours")
      .add(item.startTime.getUTCMinutes(), "minutes");
    
    let end = dateInTz
      .add(item.endTime.getUTCHours(), "hours")
      .add(item.endTime.getUTCMinutes(), "minutes");
    
    // 步骤6: DST 调整
    const offsetBeginningOfDay = dayjs(start.format("YYYY-MM-DD hh:mm"))
      .tz(adjustedTimezone).utcOffset();
    const offsetDiff = start.utcOffset() - offsetBeginningOfDay;
    
    start = start.add(offsetDiff, "minute");
    end = end.add(offsetDiff, "minute");
    
    // 步骤7: 裁剪到请求范围
    const startResult = dayjs.max(start, dateFrom);
    let endResult = dayjs.min(end, dateTo.tz(adjustedTimezone));
    
    // 步骤8: 23:59 特殊处理（自动 +1 分钟）
    if (endResult.hour() === 23 && endResult.minute() === 59) {
      endResult = endResult.add(1, "minute");
    }
    
    // 步骤9: 无效范围跳过（结束早于开始）
    if (endResult.isBefore(startResult)) {
      continue;
    }
    
    // 步骤10: 范围合并（重叠或相邻范围）
    // ... 见 3.6 节
    
  }
  
  return results;
}
```

### 3.6 工作时间范围合并策略

`processWorkingHours()` 支持两种合并场景：

#### 场景 1：同一开始时间的范围合并

```typescript
// packages/features/schedules/lib/date-ranges.ts:128-156
if (results[startResult.valueOf()]) {
  // 如果已存在相同开始时间的范围，合并结束时间
  const oldKey = startResult.valueOf();
  const newKey = endResult.valueOf();
  
  results[newKey] = {
    start: results[oldKey].start,
    end: dayjs.max(results[oldKey].end, endResult),  // 取较大的结束时间
  };
  
  delete results[oldKey];
  continue;
}
```

**示例**：
```
范围1: 09:00-12:00
范围2: 09:00-17:00
合并后: 09:00-17:00（取最大结束时间）
```

#### 场景 2：同一结束时间的范围合并

```typescript
// packages/features/schedules/lib/date-ranges.ts:104-126
// 使用 endTimeToKeyMap 进行 O(1) 查找
const keysWithSameEndTime = endTimeToKeyMap.get(endTimeKey) || [];

for (const key of keysWithSameEndTime) {
  const existingRange = results[key];
  if (
    startResult.valueOf() <= existingRange.end.valueOf() &&
    endResult.valueOf() >= existingRange.start.valueOf()
  ) {
    // 合并：取最早开始时间，保持相同结束时间
    results[key] = {
      start: dayjs.min(existingRange.start, startResult),
      end: endResult,
    };
    foundOverlapping = true;
    break;
  }
}
```

**示例**：
```
范围1: 06:00-10:00
范围2: 08:00-10:00
合并后: 06:00-10:00（取最早开始时间）
```

---

## 4. 排除规则叠加机制

### 4.1 忙碌时间来源总览

忙碌时间从多个来源收集，最终合并为单一的忙碌时间数组：

| 来源类型 | 优先级 | 数据结构 | 处理位置 |
|----------|--------|----------|----------|
| **日历集成** | 高 | `EventBusyDetails[]` | `getCalendar().getAvailability()` |
| **现有预约** | 高 | `EventBusyDetails[]` | `bookingRepo.findAllExistingBookings()` |
| **前后缓冲** | 高 | 扩展现有忙碌时间 | `getBusyTimes()` 中计算 |
| **预订限制** | 中 | `EventBusyDetails[]` | `getBusyTimesFromLimits()` |
| **时长限制** | 中 | `EventBusyDetails[]` | `getBusyTimesFromLimits()` |
| **预留槽位** | 高 | `EventBusyDate[]` | `_getReservedSlotsAndCleanupExpired()` |

### 4.2 缓冲时间的应用

缓冲时间通过**扩展**现有忙碌时间来实现：

```typescript
// 示意逻辑
function applyBufferTimes(busyTimes, beforeBuffer, afterBuffer) {
  return busyTimes.map((busy) => ({
    ...busy,
    start: dayjs(busy.start).subtract(beforeBuffer, 'minute'),
    end: dayjs(busy.end).add(afterBuffer, 'minute'),
  }));
}
```

**判定依据**：
- 缓冲时间是**预防性的**，用于避免会议之间过于紧凑
- 缓冲时间应用于**所有来源**的忙碌时间（日历、预约等）
- 缓冲时间的效果是**扩展**忙碌范围，而非创建新的忙碌范围

### 4.3 subtract() 算法详解

`subtract()` 是从可用范围中减去忙碌范围的核心算法：

```typescript
// packages/features/schedules/lib/date-ranges.ts:423-452
export function subtract<TSourceRange extends DateRange, TExcludedRange extends DateRange>(
  sourceRanges: TSourceRange[],
  excludedRanges: TExcludedRange[]
): SubtractedRange<TSourceRange>[] {
  
  const result: SubtractedRange<TSourceRange>[] = [];
  
  // 步骤1: 排序排除范围（按开始时间）
  const sortedExcludedRanges = [...excludedRanges].sort(
    (a, b) => a.start.valueOf() - b.start.valueOf()
  );
  
  // 步骤2: 遍历每个源范围
  for (const { start: sourceStart, end: sourceEnd, ...passThrough } of sourceRanges) {
    let currentStart = sourceStart;  // 追踪当前可用起点
    
    // 步骤3: 遍历每个排除范围
    for (const excludedRange of sortedExcludedRanges) {
      
      // 优化1: 排除范围在源范围之后 → 后续都不会重叠
      if (excludedRange.start.valueOf() >= sourceEnd.valueOf()) break;
      
      // 优化2: 排除范围在当前起点之前 → 跳过
      if (excludedRange.end.valueOf() <= currentStart.valueOf()) continue;
      
      // 情况A: 排除范围的开始在当前起点之后
      // 这意味着 [currentStart, excludedRange.start) 是可用的
      if (excludedRange.start.valueOf() > currentStart.valueOf()) {
        result.push({ 
          start: currentStart, 
          end: excludedRange.start, 
          ...passThrough 
        });
      }
      
      // 情况B: 排除范围的结束在当前起点之后
      // 更新当前起点到排除范围结束
      if (excludedRange.end.valueOf() > currentStart.valueOf()) {
        currentStart = excludedRange.end;
      }
    }
    
    // 步骤4: 检查源范围结束之前是否还有剩余可用时间
    if (sourceEnd.valueOf() > currentStart.valueOf()) {
      result.push({ 
        start: currentStart, 
        end: sourceEnd, 
        ...passThrough 
      });
    }
  }
  
  return result;
}
```

### 4.4 subtract() 场景示例

#### 场景 1：单个忙碌范围在中间

```
源范围:    [09:00 ───────────────────────────── 17:00]
排除范围:          [11:00 ───── 13:00]

处理过程:
  currentStart = 09:00
  
  处理 [11:00-13:00]:
    11:00 > 09:00 → 添加 [09:00, 11:00)
    currentStart = 13:00
  
  循环结束:
    17:00 > 13:00 → 添加 [13:00, 17:00)

结果:
  [09:00-11:00], [13:00-17:00]
```

#### 场景 2：多个部分重叠的忙碌范围

```
源范围:    [09:00 ───────────────────────────────────── 18:00]
排除范围1:         [10:00 ───── 12:00]
排除范围2:                  [11:30 ───── 14:00]
排除范围3:                           [15:00 ───── 17:00]

排序后排除范围: [10:00-12:00, 11:30-14:00, 15:00-17:00]

处理过程:
  currentStart = 09:00
  
  处理 [10:00-12:00]:
    10:00 > 09:00 → 添加 [09:00, 10:00)
    currentStart = 12:00
  
  处理 [11:30-14:00]:
    14:00 > 12:00 → currentStart = 14:00
  
  处理 [15:00-17:00]:
    15:00 > 14:00 → 添加 [14:00, 15:00)
    currentStart = 17:00
  
  循环结束:
    18:00 > 17:00 → 添加 [17:00, 18:00)

结果:
  [09:00-10:00], [14:00-15:00], [17:00-18:00]
```

#### 场景 3：忙碌范围完全覆盖源范围

```
源范围:    [09:00 ───────────────────────────── 17:00]
排除范围: [08:00 ───────────────────────────────────── 18:00]

处理过程:
  currentStart = 09:00
  
  处理 [08:00-18:00]:
    08:00 <= 09:00 → 不添加
    currentStart = 18:00
  
  循环结束:
    17:00 <= 18:00 → 不添加

结果:
  []（无可用时间）
```

#### 场景 4：忙碌范围在源范围之前或之后

```
源范围:    [09:00 ───────────────────────────── 17:00]
排除范围1: [07:00 ───── 08:30]
排除范围2:                           [16:30 ───── 19:00]

处理过程:
  currentStart = 09:00
  
  处理 [07:00-08:30]:
    08:30 <= 09:00 → 跳过
  
  处理 [16:30-19:00]:
    16:30 > 09:00 → 添加 [09:00, 16:30)
    currentStart = 19:00
  
  循环结束:
    17:00 <= 19:00 → 不添加

结果:
  [09:00-16:30]
```

### 4.5 预订限制的特殊处理

预订限制（bookingLimits 和 durationLimits）是一种"虚拟"的忙碌时间：

```typescript
// 示意：packages/features/busyTimes/lib/getBusyTimesFromLimits.ts
async function getBusyTimesFromLimits({
  bookingLimits,
  durationLimits,
  dateFrom,
  dateTo,
  existingBookings,
  timeZone,
}) {
  const limitManager = new LimitManager();
  
  // 处理预订数量限制
  if (bookingLimits) {
    for (const unit of ['PER_HOUR', 'PER_DAY', 'PER_WEEK', 'PER_MONTH', 'PER_QUARTER', 'PER_YEAR']) {
      const limit = bookingLimits[unit];
      if (!limit) continue;
      
      // 获取该单位的所有周期开始日期
      const periodStarts = getPeriodStartDatesBetween(dateFrom, dateTo, unit, timeZone);
      
      for (const periodStart of periodStarts) {
        // 统计该周期内的已有预订数
        const count = countBookingsInPeriod(existingBookings, periodStart, unit);
        
        if (count >= limit) {
          // 已达上限 → 标记整个周期为"忙碌"
          limitManager.addBusyTime({
            start: periodStart,
            end: periodStart.endOf(unit),
            title: `已达到 ${limit} 个${unit}预订限制`,
            source: 'bookingLimit',
          });
        }
      }
    }
  }
  
  // 处理时长限制（类似逻辑）
  if (durationLimits) {
    // ... 累计已用时长，超过则标记为忙碌
  }
  
  return limitManager.getBusyTimes();
}
```

**关键设计**：
- 预订限制标记**整个周期**为忙碌，而非精确控制每个时间槽
- 这是一种**保守策略**：只要周期内已达上限，整个周期都不可用
- 周期单位优先级：`PER_HOUR` > `PER_DAY` > `PER_WEEK` > `PER_MONTH` > `PER_QUARTER` > `PER_YEAR`

### 4.6 排除规则叠加的优先级

虽然所有忙碌时间最终都通过 `subtract()` 处理，但它们的**收集方式**不同：

```
高优先级（直接影响可用范围）
    │
    ├── 1. 日期覆盖 (DateOverride)
    │       └── 在 buildDateRanges 阶段就已覆盖 WorkingHours
    │
    ├── 2. 日历忙碌时间
    │       └── 来自 freebusy API，实时获取
    │
    ├── 3. 现有预约 (Bookings)
    │       └── 数据库中已确认的预订
    │
    ├── 4. 前后缓冲时间
    │       └── 扩展现有忙碌时间的范围
    │
    ├── 5. 预留槽位 (ReservedSlots)
    │       └── 其他用户临时锁定的槽位（后处理阶段）
    │
    └── 6. 预订限制/时长限制
            └── 整个周期标记为忙碌（较保守）

低优先级
```

**关键区别**：
- **日期覆盖**：在 `buildDateRanges` 阶段就已生效，不进入 `subtract()`
- **其他排除规则**：在 `subtract()` 阶段从可用范围中减去
- **预留槽位**：在 `getSlots()` 之后的后处理阶段过滤

---

## 5. 时间槽生成算法

### 5.1 getSlots() 核心流程

`getSlots()` 将日期范围转换为具体的可预约时间槽：

```typescript
// packages/features/schedules/lib/slots.ts:232-262
const getSlots = ({
  inviteeDate,
  frequency,
  minimumBookingNotice,
  dateRanges,
  eventLength,
  offsetStart = 0,
  datesOutOfOffice,
  showOptimizedSlots,
  datesOutOfOfficeTimeZone,
}: GetSlots) => {
  return buildSlotsWithDateRanges({
    dateRanges,
    frequency,
    eventLength,
    timeZone: getTimeZone(inviteeDate),
    minimumBookingNotice,
    offsetStart,
    datesOutOfOffice,
    showOptimizedSlots,
    datesOutOfOfficeTimeZone,
  });
};
```

### 5.2 buildSlotsWithDateRanges 详解

```typescript
// packages/features/schedules/lib/slots.ts:71-230
function buildSlotsWithDateRanges({
  dateRanges,
  frequency,
  eventLength,
  timeZone,
  minimumBookingNotice,
  offsetStart,
  datesOutOfOffice,
  showOptimizedSlots,
  datesOutOfOfficeTimeZone,
}) {
  
  // 参数归一化
  frequency = minimumOfOne(frequency);
  eventLength = minimumOfOne(eventLength);
  offsetStart = offsetStart ? minimumOfOne(offsetStart) : 0;
  
  // 排序日期范围
  const orderedDateRanges = dateRanges.sort((a, b) => a.start.valueOf() - b.start.valueOf());
  
  // 存储生成的槽位（键：ISO 字符串，值：槽位数据）
  const slots = new Map<string, { time: Dayjs; userIds?: number[]; away?: boolean; ... }>();
  
  // 步骤1: 选择间隔单位
  let interval = Number(process.env.NEXT_PUBLIC_AVAILABILITY_SCHEDULE_INTERVAL) || 1;
  const intervalsWithDefinedStartTimes = [60, 30, 20, 15, 10, 5];
  
  for (let i = 0; i < intervalsWithDefinedStartTimes.length; i++) {
    if (frequency % intervalsWithDefinedStartTimes[i] === 0) {
      interval = intervalsWithDefinedStartTimes[i];
      break;
    }
  }
  
  // 步骤2: 计算最小预订通知时间边界
  const startTimeWithMinNotice = dayjs.utc().add(minimumBookingNotice, "minute");
  
  // 追踪槽位边界（用于避免重复生成）
  const slotBoundaries = new Map<number, true>();
  
  // 步骤3: 遍历每个日期范围
  orderedDateRanges.forEach((range) => {
    
    // 步骤3.1: 确定实际开始时间（考虑最小预订通知）
    let slotStartTime = range.start.utc().isAfter(startTimeWithMinNotice)
      ? range.start
      : startTimeWithMinNotice;
    
    // 归一化秒和毫秒
    slotStartTime = slotStartTime.set("second", 0).set("millisecond", 0);
    
    // 步骤3.2: 关键：时区转换后再计算分钟对齐
    // 这防止了半小时偏移时区的问题
    slotStartTime = slotStartTime.tz(timeZone);
    
    // 步骤3.3: 分钟对齐检查
    if (slotStartTime.minute() % interval !== 0) {
      slotStartTime = getCorrectedSlotStartTime({
        showOptimizedSlots,
        interval,
        slotStartTime,
        range,
      });
    }
    
    // 应用偏移
    slotStartTime = slotStartTime.add(offsetStart ?? 0, "minutes");
    
    // 步骤3.4: 重叠范围处理（避免重复槽位）
    const slotBoundariesValueArray = Array.from(slotBoundaries.keys());
    if (slotBoundariesValueArray.length > 0) {
      slotBoundariesValueArray.sort((a, b) => a - b);
      
      // 找到最近的前一个边界
      let prevBoundary = null;
      for (let i = slotBoundariesValueArray.length - 1; i >= 0; i--) {
        if (slotBoundariesValueArray[i] < slotStartTime.valueOf()) {
          prevBoundary = slotBoundariesValueArray[i];
          break;
        }
      }
      
      // 如果前一个边界的结束时间在当前开始时间之后
      if (prevBoundary) {
        const prevBoundaryEnd = dayjs(prevBoundary).add(frequency + (offsetStart ?? 0), "minutes");
        if (prevBoundaryEnd.isAfter(slotStartTime)) {
          const dayjsPrevBoundary = dayjs(prevBoundary);
          if (!dayjsPrevBoundary.isBefore(range.start)) {
            // 前一个边界在当前范围内 → 使用前一个边界
            slotStartTime = dayjsPrevBoundary;
          } else {
            // 前一个边界在当前范围之前 → 跳到前一个边界结束之后
            slotStartTime = prevBoundaryEnd;
          }
          slotStartTime = slotStartTime.tz(timeZone);
        }
      }
    }
    
    // 步骤3.5: 生成槽位循环
    while (!slotStartTime.add(eventLength, "minutes").subtract(1, "second").utc().isAfter(range.end)) {
      const slotKey = slotStartTime.toISOString();
      
      // 避免重复槽位
      if (slots.has(slotKey)) {
        slotStartTime = slotStartTime.add(frequency + (offsetStart ?? 0), "minutes");
        continue;
      }
      
      // 记录边界
      slotBoundaries.set(slotStartTime.valueOf(), true);
      
      // 步骤3.6: OOO 标记检查
      let dateOutOfOfficeExists = undefined;
      if (datesOutOfOffice) {
        const slotDateYYYYMMDD = datesOutOfOfficeTimeZone
          ? slotStartTime.tz(datesOutOfOfficeTimeZone).format("YYYY-MM-DD")
          : slotStartTime.utc().format("YYYY-MM-DD");
        dateOutOfOfficeExists = datesOutOfOffice?.[slotDateYYYYMMDD];
      }
      
      // 构建槽位数据
      let slotData = { time: slotStartTime };
      
      if (dateOutOfOfficeExists) {
        const { toUser, fromUser, reason, emoji, notes, showNotePublicly } = dateOutOfOfficeExists;
        slotData = {
          time: slotStartTime,
          away: true,  // 标记为"外出"
          ...(fromUser && { fromUser }),
          ...(toUser && { toUser }),
          ...(reason && { reason }),
          ...(emoji && { emoji }),
          ...(notes && showNotePublicly && { notes }),
          ...(showNotePublicly !== undefined && { showNotePublicly }),
        };
      }
      
      slots.set(slotKey, slotData);
      
      // 移动到下一个槽位
      slotStartTime = slotStartTime.add(frequency + (offsetStart ?? 0), "minutes");
    }
  });
  
  return Array.from(slots.values());
}
```

### 5.3 间隔选择逻辑

```typescript
// 优先级：60 → 30 → 20 → 15 → 10 → 5 → 1（默认）
const intervalsWithDefinedStartTimes = [60, 30, 20, 15, 10, 5];

for (let i = 0; i < intervalsWithDefinedStartTimes.length; i++) {
  if (frequency % intervalsWithDefinedStartTimes[i] === 0) {
    interval = intervalsWithDefinedStartTimes[i];
    break;
  }
}
```

**示例**：
| frequency | 整除检查 | interval | 说明 |
|-----------|----------|----------|------|
| 60 | 60 % 60 = 0 ✓ | 60 | 整点对齐 |
| 45 | 45 % 60 ≠ 0, 45 % 30 ≠ 0, 45 % 20 ≠ 0, **45 % 15 = 0** ✓ | 15 | 15分钟边界对齐 |
| 25 | 25 % 60 ≠ 0, ..., **25 % 5 = 0** ✓ | 5 | 5分钟边界对齐 |
| 7 | 所有都不能整除 | 1 | 无对齐 |

### 5.4 时区对齐的关键设计

**问题场景**：半小时偏移时区（如 Asia/Kolkata GMT+5:30）

```typescript
// packages/features/schedules/lib/slots.ts:135-147
// 关键：先转换到目标时区，再计算分钟对齐
slotStartTime = slotStartTime.tz(timeZone);

if (slotStartTime.minute() % interval !== 0) {
  slotStartTime = getCorrectedSlotStartTime({...});
}
```

**判定依据**：
- 如果在 UTC 时间计算分钟对齐，半小时偏移时区分出现问题
- **正确做法**：在主机时区计算分钟对齐

**示例**：
```
配置：
  - 主机时区：Asia/Kolkata (+05:30)
  - 工作时间：09:00-17:00（加尔各答时间）
  - 事件时长：60 分钟
  - interval：60（整点对齐）

加尔各答 09:00 = UTC 03:30

错误做法（在 UTC 计算）：
  UTC 03:30.minute() = 30
  30 % 60 ≠ 0 → 错误地调整到 04:00 UTC = 09:30 加尔各答

正确做法（在加尔各答时区计算）：
  slotStartTime.tz("Asia/Kolkata").minute() = 0
  0 % 60 = 0 → 不需要调整
```

### 5.5 优化槽位对齐算法

`getCorrectedSlotStartTime()` 处理槽位开始时间的对齐：

```typescript
// packages/features/schedules/lib/slots.ts:27-69
function getCorrectedSlotStartTime({
  slotStartTime,
  range,
  showOptimizedSlots,
  interval,
}) {
  if (showOptimizedSlots) {
    let correctedSlotStartTime = slotStartTime;
    
    // 计算需要移动的分钟数
    const minutesRequiredToMoveToNextSlot = interval - (slotStartTime.minute() % interval);
    const minutesRequiredToMoveTo15MinSlot = 15 - (slotStartTime.minute() % 15);
    const minutesRequiredToMoveTo5MinSlot = 5 - (slotStartTime.minute() % 5);
    
    // 计算范围中剩余的"额外"分钟（完整槽位后剩余的时间）
    const extraMinutesAvailable = range.end.diff(slotStartTime, "minutes") % interval;
    
    // 策略1: 如果有足够时间推到下一个完整间隔
    if (extraMinutesAvailable >= minutesRequiredToMoveToNextSlot) {
      // 示例：09:05-12:00，60分钟事件
      // 总可用 175 分钟 = 2*60 + 55
      // 剩余 55 分钟 >= 需要移动的 55 分钟（60-5）
      // 推到 10:00
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveToNextSlot, "minute");
    }
    // 策略2: 次之，推到下一个 15 分钟边界
    else if (extraMinutesAvailable >= minutesRequiredToMoveTo15MinSlot) {
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveTo15MinSlot, "minute");
    }
    // 策略3: 最次之，推到下一个 5 分钟边界
    else if (extraMinutesAvailable >= minutesRequiredToMoveTo5MinSlot) {
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveTo5MinSlot, "minute");
    }
    
    return correctedSlotStartTime;
  }
  
  // 非优化模式：简单向上取整
  return slotStartTime.startOf("hour").add(
    Math.ceil(slotStartTime.minute() / interval) * interval,
    "minute"
  );
}
```

### 5.6 优化 vs 非优化槽位对比

| 配置 | 优化模式 | 非优化模式 |
|------|----------|------------|
| 可用范围 | 09:30-17:30 | 09:30-17:30 |
| 事件时长 | 60分钟 | 60分钟 |
| 生成槽位 | 09:30, 10:30, 11:30, 12:30, 13:30, 14:30, 15:30, 16:30 (8个) | 10:00, 11:00, 12:00, 13:00, 14:00, 15:00, 16:00 (7个) |
| 逻辑 | 从范围开始时间，每次 +60 分钟 | 对齐到整点，09:30 向上取整到 10:00 |

### 5.7 重叠范围的槽位去重

当多个日期范围有重叠时，系统会智能避免生成重复槽位：

```typescript
// packages/features/schedules/lib/slots.ts:151-176
const slotBoundariesValueArray = Array.from(slotBoundaries.keys());
if (slotBoundariesValueArray.length > 0) {
  // 找到最近的前一个边界
  let prevBoundary = null;
  for (let i = slotBoundariesValueArray.length - 1; i >= 0; i--) {
    if (slotBoundariesValueArray[i] < slotStartTime.valueOf()) {
      prevBoundary = slotBoundariesValueArray[i];
      break;
    }
  }
  
  // 检查是否需要调整
  if (prevBoundary) {
    const prevBoundaryEnd = dayjs(prevBoundary).add(frequency + (offsetStart ?? 0), "minutes");
    if (prevBoundaryEnd.isAfter(slotStartTime)) {
      const dayjsPrevBoundary = dayjs(prevBoundary);
      if (!dayjsPrevBoundary.isBefore(range.start)) {
        // 前一个边界在当前范围内 → 使用前一个边界
        slotStartTime = dayjsPrevBoundary;
      } else {
        // 前一个边界在当前范围之前 → 跳到前一个边界结束之后
        slotStartTime = prevBoundaryEnd;
      }
      slotStartTime = slotStartTime.tz(timeZone);
    }
  }
}
```

**场景示例**：
```
两个重叠范围：
  范围1: 11:00-12:00
  范围2: 11:20-12:40

frequency = 45 分钟

处理范围1:
  生成槽位: 11:00（占用 11:00-11:45）
  下一个槽位会在 11:45，但范围1在 12:00 结束
  11:45 + 45 = 12:30，超出 12:00
  范围1只生成 1 个槽位

处理范围2:
  初始 slotStartTime = 11:20
  检查到前一个边界 11:00，其结束时间 = 11:00 + 45 = 11:45
  11:45 在 11:20 之后 → 需要调整
  
  前一个边界 11:00 在当前范围 11:20 之前
  → 跳到前一个边界结束之后: slotStartTime = 11:45
  
  从 11:45 开始：
    11:45 + 45 = 12:30，在 12:40 之前
    生成槽位: 11:45

最终结果：
  11:00（来自范围1）
  11:45（来自范围2，跳过了重叠部分）
```

### 5.8 外出标记的特殊处理

**关键发现**：OOO（外出）的处理方式取决于事件类型（单用户 vs 团队）：

#### 1. 单用户事件

在单用户事件中，OOO 只影响 UI 层：

- **`buildDateRanges` 返回值**：
  - `dateRanges`：不包含 OOO，可用范围不受影响
  - `oooExcludedDateRanges`：包含 OOO，但单用户事件不使用这个
- **`getSlots` 中标记**：
  - 通过 `datesOutOfOffice` 参数传递
  - 检查槽位日期是否在 OOO 中
  - 标记为 `away: true`

```typescript
// packages/features/schedules/lib/slots.ts:187-222
let dateOutOfOfficeExists = undefined;
if (datesOutOfOffice) {
  const slotDateYYYYMMDD = datesOutOfOfficeTimeZone
    ? slotStartTime.tz(datesOutOfOfficeTimeZone).format("YYYY-MM-DD")
    : slotStartTime.utc().format("YYYY-MM-DD");
  dateOutOfOfficeExists = datesOutOfOffice?.[slotDateYYYYMMDD];
}

if (dateOutOfOfficeExists) {
  slotData = {
    time: slotStartTime,
    away: true,  // 标记为"外出"
    // ... 其他 OOO 信息
  };
}
```

#### 2. 团队事件（COLLECTIVE / ROUND_ROBIN / 多用户）

**重要修正**：在团队事件中，OOO 会**直接影响可用范围的生成**！

从 `buildDateRanges` 可以看到：

```typescript
// packages/features/schedules/lib/date-ranges.ts:312-328
const dateRanges = Object.values({
  ...groupedWorkingHours,
  ...groupedDateOverrides,
}).map(
  (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
);

const oooExcludedDateRanges = Object.values({
  ...groupedWorkingHours,
  ...groupedDateOverrides,
  ...groupedOOO,  // 关键：这里加入了 OOO！
}).map(
  (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
);
```

然后在 `getAggregatedAvailability` 中：

```typescript
// packages/features/availability/lib/getAggregatedAvailability/getAggregatedAvailability.ts:25-44
const isTeamEvent =
  schedulingType === SchedulingType.COLLECTIVE ||
  schedulingType === SchedulingType.ROUND_ROBIN ||
  userAvailability.length > 1;

const fixedDateRanges = mergeOverlappingDateRanges(
  intersect(fixedHosts.map((s) => (!isTeamEvent ? s.dateRanges : s.oooExcludedDateRanges)))
);
```

**测试用例验证**（`date-ranges.test.ts:548-582`）：

```typescript
const outOfOffice = {
  "2023-06-13": {
    fromUser: { id: 1, displayName: "Team Free Example" },
  },
};

const { dateRanges, oooExcludedDateRanges } = buildDateRanges({...});

// dateRanges 包含 6月13日和6月14日（OOO 不影响）
expect(dateRanges[0]).toEqual({
  start: dayjs.utc("2023-06-13T14:00:00Z").tz(timeZone),
  end: dayjs.utc("2023-06-13T19:00:00Z").tz(timeZone),
});
expect(dateRanges[1]).toEqual({
  start: dayjs.utc("2023-06-14T12:00:00Z").tz(timeZone),
  end: dayjs.utc("2023-06-14T21:00:00Z").tz(timeZone),
});

// oooExcludedDateRanges 只包含 6月14日（6月13日被 OOO 排除了！）
expect(oooExcludedDateRanges.length).toBe(1);
expect(oooExcludedDateRanges[0]).toEqual({
  start: dayjs("2023-06-14T12:00:00Z").tz(timeZone),
  end: dayjs("2023-06-14T21:00:00Z").tz(timeZone),
});
```

#### 判定依据总结

| 事件类型 | 使用的范围 | OOO 影响 |
|----------|------------|----------|
| **单用户事件** | `dateRanges` | 只在 `getSlots` 中标记 `away: true`，不影响可用范围 |
| **团队事件**（COLLECTIVE/ROUND_ROBIN/多用户） | `oooExcludedDateRanges` | **直接影响可用范围**，OOO 日期的可用范围被完全移除 |

**为什么 OOO 在团队事件中会排除可用范围？**

因为 OOO 在 `buildDateRanges` 中被处理为 `{ start: date, end: date }`（空范围），然后通过对象展开覆盖机制：

```typescript
const oooExcludedDateRanges = Object.values({
  ...groupedWorkingHours,    // 先展开工作时间
  ...groupedDateOverrides,    // 然后是日期覆盖
  ...groupedOOO,               // 最后是 OOO（覆盖同日期的所有内容）
}).map(
  // 过滤掉空范围
  (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
);
```

由于 OOO 的范围是 `{ start: date, end: date }`（start === end），在过滤阶段会被移除，所以该日期的可用范围变为空数组。

---

## 6. 冲突检测与收束机制

### 6.1 冲突检测的多层架构

冲突检测发生在多个阶段：

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                          冲突检测层级                                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│  Level 1: 日期范围冲突（subtract）                                                    │
│  ─────────────────────────────────────                                               │
│  目的：从可用范围中减去忙碌时间                                                       │
│  位置：UserAvailabilityService._getUserAvailability()                                │
│  算法：subtract(availableRanges, busyRanges)                                         │
│  模型：开放区间 [start, end)                                                          │
│                                                                                       │
│  Level 2: 预留槽位冲突（checkForConflicts）                                           │
│  ─────────────────────────────────────────                                            │
│  目的：检查与其他用户临时锁定的槽位冲突                                               │
│  位置：AvailableSlotsService._getAvailableSlots() 后处理阶段                         │
│  算法：checkForConflicts(slot, reservedSlots)                                        │
│                                                                                       │
│  Level 3: 座位限制冲突                                                                │
│  ──────────────────────────                                                          │
│  目的：检查座位事件是否已满                                                           │
│  位置：多个位置（getCurrentSeats, reserveSlot 验证）                                  │
│  判定：attendeesCount >= seatsPerTimeSlot                                             │
│                                                                                       │
│  Level 4: 预订限制冲突                                                                │
│  ──────────────────────────                                                          │
│  目的：检查是否达到预订数量/时长限制                                                   │
│  位置：getBusyTimesFromLimits()                                                       │
│  判定：count >= limit 或 totalDuration > limit                                        │
│                                                                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 开放区间模型详解

系统所有时间比较都采用**开放区间**模型：

```
数学定义：
  [start, end)  包含 start，不包含 end

判定逻辑：
  范围 A = [A_start, A_end)
  范围 B = [B_start, B_end)
  
  重叠条件：
    A_start < B_end AND B_start < A_end
  
  不重叠条件（德摩根定律）：
    A_start >= B_end OR B_start >= A_end
```

### 6.3 边界场景判定

以下是 `checkForConflicts()` 的核心逻辑：

```typescript
// packages/features/bookings/lib/conflictChecker/checkForConflicts.ts:9-50
export function checkForConflicts({
  busy,
  time,
  eventLength,
  currentSeats,
}: {
  busy: BufferedBusyTimes;
  time: Dayjs;
  eventLength: number;
  currentSeats?: CurrentSeats;
}) {
  
  // 早期返回 1: 无忙碌时间则无冲突
  if (!Array.isArray(busy) || busy.length < 1) {
    return false;
  }
  
  // 早期返回 2: 座位事件特殊处理
  // 如果当前时间槽有座位数据，则不视为冲突
  // （座位事件允许多个预订，直到座位满）
  if (currentSeats?.some((booking) => booking.startTime.toISOString() === time.toISOString())) {
    return false;
  }
  
  // 计算槽位范围
  const slotStart = time.valueOf();
  const slotEnd = slotStart + eventLength * 60 * 1000;
  
  // 排序忙碌时间（按开始时间）
  const sortedBusyTimes = busy
    .map((busyTime) => ({
      start: dayjs.utc(busyTime.start).valueOf(),
      end: dayjs.utc(busyTime.end).valueOf(),
    }))
    .sort((a, b) => a.start - b.start);
  
  // 遍历检查冲突
  for (const busyTime of sortedBusyTimes) {
    // 优化 1: 忙碌时间在槽位结束之后 → 提前退出
    if (busyTime.start >= slotEnd) {
      break;
    }
    // 优化 2: 忙碌时间在槽位开始之前 → 跳过
    if (busyTime.end <= slotStart) {
      continue;
    }
    // 到这里说明存在时间重叠 → 冲突
    return true;
  }
  
  return false;
}
```

### 6.4 冲突场景判定表

| 场景 | 槽位范围 | 忙碌范围 | 是否冲突 | 判定依据 |
|------|----------|----------|----------|----------|
| **部分重叠** | [10:00, 11:00) | [10:30, 11:30) | ✅ 冲突 | 10:00 < 11:30 AND 10:30 < 11:00 |
| **完全包含** | [10:00, 12:00) | [10:30, 11:30) | ✅ 冲突 | 10:00 < 11:30 AND 10:30 < 12:00 |
| **被包含** | [10:30, 11:30) | [10:00, 12:00) | ✅ 冲突 | 10:30 < 12:00 AND 10:00 < 11:30 |
| **刚好接触** | [10:00, 11:00) | [11:00, 12:00) | ❌ 无冲突 | 10:00 < 12:00 ✓ 但 11:00 < 11:00 ✗ |
| **完全分离（前）** | [10:00, 11:00) | [08:00, 09:00) | ❌ 无冲突 | 10:00 < 09:00 ✗ |
| **完全分离（后）** | [10:00, 11:00) | [13:00, 14:00) | ❌ 无冲突 | 13:00 < 11:00 ✗ |

### 6.5 "刚好接触"场景的设计意图

**场景**：
```
槽位 A: [10:00, 11:00)  → 10:00 开始，11:00 结束（不包含 11:00）
槽位 B: [11:00, 12:00)  → 11:00 开始，12:00 结束（不包含 12:00）

这两个槽位在 11:00 处"接触"，但不重叠。
```

**设计意图**：
1. **用户体验**：11:00 结束的会议和 11:00 开始的会议之间没有时间冲突
2. **数学精确性**：开放区间模型在边界处是连续的
3. **缓冲时间**：如果需要会议之间有间隔，应该使用 `beforeEventBuffer` 和 `afterEventBuffer`

**示例**：
```
用户配置：
  - 会议 A: 10:00-11:00
  - 会议 B: 11:00-12:00
  
没有缓冲时间：
  - 这是允许的（两个会议"背靠背"）
  
配置 15 分钟缓冲：
  - 会议 A 的忙碌范围变为：[09:45, 11:15)
  - 会议 B 的忙碌范围变为：[10:45, 12:15)
  - 这两个范围会冲突（重叠 10:45-11:15）
```

### 6.6 座位事件的冲突处理

座位事件（`seatsPerTimeSlot`）是特殊的事件类型：

```typescript
// checkForConflicts 中的座位处理
if (currentSeats?.some((booking) => booking.startTime.toISOString() === time.toISOString())) {
  return false;  // 有座位数据 → 不认为是冲突
}
```

**判定依据**：

| 事件类型 | 冲突模型 | 可用条件 |
|----------|----------|----------|
| **普通事件** | 开放区间重叠 | 无任何重叠 |
| **座位事件** | 座位计数 | `attendeesCount < seatsPerTimeSlot` |

**座位事件流程**：
1. `getCurrentSeats()` 获取已有座位预订
2. 在 `checkForConflicts()` 中跳过冲突检测
3. 在 `reserveSlot()` 中验证座位是否已满：`attendeesCount >= seatsPerTimeSlot`

### 6.7 预留槽位的冲突检测

预留槽位是其他用户临时锁定的时间槽，需要在后处理阶段过滤：

```typescript
// packages/trpc/server/routers/viewer/slots/util.ts:1166-1244
const reservedSlots = await this._getReservedSlotsAndCleanupExpired({
  bookerClientUid,
  eventTypeId: eventType.id,
  usersWithCredentials,
});

if (reservedSlots?.length > 0) {
  // 处理座位预留
  let occupiedSeats = reservedSlots.filter(
    (item) => item.isSeat && item.eventTypeId === eventType.id
  );
  
  // 处理非座位预留（转换为忙碌时间）
  const busySlotsFromReservedSlots = reservedSlots.reduce<EventBusyDate[]>((r, c) => {
    if (!c.isSeat) {
      r.push({ start: c.slotUtcStartDate, end: c.slotUtcEndDate });
    }
    return r;
  }, []);
  
  // 过滤掉与预留槽位冲突的可用时间槽
  availableTimeSlots = availableTimeSlots
    .map((slot) => {
      if (
        !checkForConflicts({
          time: slot.time,
          busy: busySlotsFromReservedSlots,
          ...availabilityCheckProps,
        })
      ) {
        return slot; // 无冲突 → 保留
      }
      return undefined; // 有冲突 → 过滤掉
    })
    .filter((item): item is {...} => !!item);
}
```

**关键设计**：
- 预留槽位分为 `isSeat: true`（座位事件）和 `isSeat: false`（普通事件）
- 普通事件的预留槽位直接转换为忙碌时间，通过 `checkForConflicts()` 过滤
- 座位事件的预留槽位通过 `occupiedSeats` 单独处理

---

## 7. 边界场景分析

### 7.1 时区边界场景

#### 场景 A：半小时偏移时区

**问题**：Asia/Kolkata (GMT+5:30) 等时区的分钟数不是整数

**解决方案**：在主机时区计算分钟对齐，而非 UTC

```typescript
// 关键代码
slotStartTime = slotStartTime.tz(timeZone);  // 先转换时区

if (slotStartTime.minute() % interval !== 0) {  // 再检查分钟对齐
  slotStartTime = getCorrectedSlotStartTime({...});
}
```

**判定依据**：
| 时区 | 工作时间 | UTC 时间 | 分钟对齐结果 |
|------|----------|----------|--------------|
| Asia/Kolkata (+5:30) | 09:00-17:00 | 03:30-11:30 | 在 Kolkata 时区计算：分钟 = 0，无需调整 |
| **错误做法** | - | 在 UTC 计算：分钟 = 30 | 会错误地调整到下一个边界 |

#### 场景 B：跨时区 OOO 标记

**问题**：主机在洛杉矶，预订者在加尔各答，OOO 日期应该按哪个时区计算？

**解决方案**：使用 `datesOutOfOfficeTimeZone` 进行转换

```typescript
// packages/features/schedules/lib/slots.ts:188-193
const slotDateYYYYMMDD = datesOutOfOfficeTimeZone
  ? slotStartTime.tz(datesOutOfOfficeTimeZone).format("YYYY-MM-DD")
  : slotStartTime.utc().format("YYYY-MM-DD");
dateOutOfOfficeExists = datesOutOfOffice?.[slotDateYYYYMMDD];
```

**实际场景**：
```
主机：
  - 时区：America/Los_Angeles (UTC-8)
  - OOO 日期：2026-01-16（洛杉矶本地日期）

预订者：
  - 时区：Asia/Kolkata (UTC+5:30)
  - 查看时间：2026-01-17 05:30（加尔各答时间）

时间转换：
  洛杉矶 2026-01-16 16:00 = 加尔各答 2026-01-17 05:30

关键问题：
  - 预订者看到的是 1月17日，但主机实际上是在 1月16日（洛杉矶时间）
  - 如果只看 UTC 或预订者时区，会错误地认为不是 OOO 日期

正确处理：
  - 使用 datesOutOfOfficeTimeZone = "America/Los_Angeles" 进行转换
  - slotStartTime.tz("America/Los_Angeles").format("YYYY-MM-DD") = "2026-01-16"
  - 正确识别为 OOO 日期
```

### 7.2 DST（夏令时）变更场景

**问题**：DST 变更日的时间偏移会导致工作时间计算错误

**解决方案**：计算偏移差并调整

```typescript
// packages/features/schedules/lib/date-ranges.ts:70-74
const offsetBeginningOfDay = dayjs(start.format("YYYY-MM-DD hh:mm"))
  .tz(adjustedTimezone).utcOffset();
const offsetDiff = start.utcOffset() - offsetBeginningOfDay; 
// 在 DST 变更的那天会有 60 分钟的偏移差

start = start.add(offsetDiff, "minute");
end = end.add(offsetDiff, "minute");
```

**场景示例**：
```
时区：America/New_York
DST 变更：
  - 春天：3 月第二个周日 02:00 → 03:00（向前拨 1 小时）
  - 秋天：11 月第一个周日 02:00 → 01:00（向后拨 1 小时）

工作时间：09:00-17:00

春天 DST 变更日：
  - 当天实际上只有 23 小时
  - 02:00-03:00 这个时间段不存在
  - 如果不处理，09:00 的工作时间可能会错误地偏移

秋天 DST 变更日：
  - 当天有 25 小时
  - 01:00-02:00 出现两次
  - 需要确保时间计算的一致性

解决方案：
  - 计算当天开始时的偏移和当前时间的偏移差
  - 根据偏移差调整时间
```

### 7.3 跨午夜可用性场景

**问题**：用户可能需要跨越午夜的可用性（如 23:00-00:30）

**解决方案**：通过合并相邻日期范围实现

```typescript
// 测试用例示例
it("supports availability past midnight through merging adjacent date ranges", () => {
  const items = [
    {
      days: [1, 2, 3, 4, 5],
      startTime: new Date(Date.UTC(0, 0, 0, 23, 0)), // 23:00
      endTime: new Date(Date.UTC(0, 0, 0, 23, 59)),  // 23:59
    },
    {
      days: [2, 3, 4, 5, 6],
      startTime: new Date(Date.UTC(0, 0, 0, 0, 0)),  // 00:00
      endTime: new Date(Date.UTC(0, 0, 0, 0, 30)),     // 00:30
    },
  ];
  
  // 结果应该是合并后的范围：23:00-00:30（跨午夜）
});
```

**23:59 特殊处理**：
```typescript
// packages/features/schedules/lib/date-ranges.ts:81-83
// INFO: 我们只允许用户设置到晚上 11:59 的可用性，
// 这实际上导致他们不能使用到午夜。
if (endResult.hour() === 23 && endResult.minute() === 59) {
  endResult = endResult.add(1, "minute");
}
```

**场景**：
```
用户设置：
  周一 09:00-23:59

实际处理：
  转换为 09:00-24:00（即下一天 00:00）

影响：
  - 23:30 开始的 60 分钟会议可以被接受
  - 否则会因为 23:30 + 60 = 00:30 > 23:59 而被拒绝
```

### 7.4 日期覆盖的 ±1 天缓冲

**问题**：时区差异可能导致日期计算偏移一天

**解决方案**：使用 ±1 天缓冲

```typescript
// packages/features/schedules/lib/date-ranges.ts:282-290
// TODO: 移除 .subtract(1, "day") 和 .add(1, "day") 部分
// 截至 2024-02-20，本地和 UTC 日期在覆盖方面存在不匹配
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

**极端场景**：
```
主机时区：UTC+12（斐济）
预订者时区：UTC-12（贝克岛）

同一 UTC 时刻：
  - 主机时区：2026-05-04 00:00
  - 预订者时区：2026-05-02 00:00

问题：
  - 日期覆盖可能会因为时区边界问题而被漏掉
  - 例如：主机设置 2026-05-03 的日期覆盖
  - 从预订者视角看，可能跨越多个日期

解决方案：
  - 使用 ±1 天缓冲
  - 实际查询范围：[dateFrom-1d, dateTo+1d]
  - 后续在 processDateOverride() 中精确计算实际时间范围
```

### 7.5 空范围覆盖场景

**问题**：如何表示某一天完全不可用？

**解决方案**：使用 `start === end` 的空范围

```typescript
// packages/features/schedules/lib/date-ranges.ts:316-318
.map(
  // 过滤掉空范围（start === end 表示该天不可用）
  (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
);
```

**场景**：
```
配置：
  WorkingHours: 周一至周五 09:00-17:00
  DateOverride: 2026-05-03 00:00-00:00（start === end）

构建过程：
  groupedWorkingHours["2026-05-03"] = [{ start: 09:00, end: 17:00 }]
  groupedDateOverrides["2026-05-03"] = [{ start: 00:00, end: 00:00 }]
  
  对象展开后：
    { "2026-05-03": [{ start: 00:00, end: 00:00 }] }
  
  过滤阶段（start !== end）：
    [{ start: 00:00, end: 00:00 }].filter(r => r.start !== r.end) = []

结果：
  2026-05-03 完全不可用（无时间槽）
```

---

## 8. 规则优先级判定依据

### 8.1 优先级层次结构

时间槽引擎采用**严格的优先级层次**，从最高到最低依次为：

```
高优先级
    │
    ├── 1. 日期覆盖 (DateOverride)
    │       ├── 处理位置：buildDateRanges()
    │       ├── 机制：对象展开覆盖（后展开的覆盖先展开的）
    │       ├── 代码：{ ...groupedWorkingHours, ...groupedDateOverrides }
    │       └── 特殊情况：start === end 表示该天完全不可用
    │
    ├── 2. 外出 (OutOfOffice)
    │       ├── 处理位置：getSlots()
    │       ├── 机制：标记为 away: true（不影响可用范围生成）
    │       ├── 代码：slotData = { ..., away: true, ... }
    │       └── 注意：UI 层决定是否显示或过滤
    │
    ├── 3. 假期 (Holidays)
    │       ├── 处理位置：calculateHolidayBlockedDates()
    │       ├── 机制：根据 countryCode 自动计算
    │       └── 通常转换为忙碌时间处理
    │
    ├── 4. 日历忙碌时间
    │       ├── 处理位置：getCalendar().getAvailability()
    │       ├── 机制：freebusy API 获取
    │       └── 通过 subtract() 从可用范围减去
    │
    ├── 5. 现有预约 (Bookings)
    │       ├── 处理位置：bookingRepo.findAllExistingBookings()
    │       ├── 机制：数据库查询
    │       └── 通过 subtract() 从可用范围减去
    │
    ├── 6. 前后缓冲时间
    │       ├── 处理位置：getBusyTimes()
    │       ├── 机制：扩展现有忙碌时间范围
    │       ├── 代码：start.subtract(beforeBuffer), end.add(afterBuffer)
    │       └── 应用于所有来源的忙碌时间
    │
    ├── 7. 预订数量限制
    │       ├── 处理位置：getBusyTimesFromLimits()
    │       ├── 机制：标记整个周期为忙碌
    │       ├── 优先级：PER_HOUR > PER_DAY > PER_WEEK > PER_MONTH > PER_QUARTER > PER_YEAR
    │       └── 保守策略：只要周期内已达上限，整个周期都不可用
    │
    ├── 8. 预订时长限制
    │       ├── 处理位置：getBusyTimesFromLimits()
    │       ├── 机制：累计已用时长
    │       └── 类似数量限制，但计算的是时长而非数量
    │
    └── 9. 常规工作时间 (WorkingHours)
            ├── 处理位置：processWorkingHours()
            ├── 机制：周期性展开
            └── 优先级最低，被以上所有规则覆盖

低优先级
```

### 8.2 优先级判定代码位置

| 优先级 | 规则类型 | 判定代码位置 | 关键代码 |
|--------|----------|--------------|----------|
| **1 (最高)** | 日期覆盖 | `date-ranges.ts:312-318` | `{ ...groupedWorkingHours, ...groupedDateOverrides }` |
| **2** | 外出 | `slots.ts:187-222` | `if (dateOutOfOfficeExists) { slotData.away = true }` |
| **4-5** | 日历/预约 | `date-ranges.ts:423-452` | `subtract(sourceRanges, excludedRanges)` |
| **6** | 缓冲时间 | `getBusyTimes()` | `start.subtract(beforeBuffer)` |
| **7-8** | 预订限制 | `getBusyTimesFromLimits()` | `if (count >= limit) { addBusyTime(...) }` |
| **9 (最低)** | 工作时间 | `date-ranges.ts:32-173` | `processWorkingHours()` |

### 8.3 优先级冲突场景示例

#### 场景 A：日期覆盖 vs 工作时间

```
配置：
  WorkingHours: 周一至周五 09:00-17:00
  DateOverride: 2026-05-03 (周一) 10:00-15:00

关键代码：
  const dateRanges = Object.values({
    ...groupedWorkingHours,    // 先展开
    ...groupedDateOverrides,    // 后展开（覆盖）
  });

结果：
  2026-05-03 可用时间：10:00-15:00（日期覆盖胜出）
  其他工作日：09:00-17:00（工作时间）
```

#### 场景 B：日期覆盖（空范围）vs 工作时间

```
配置：
  WorkingHours: 周一至周五 09:00-17:00
  DateOverride: 2026-05-03 (周一) 00:00-00:00（start === end）

关键代码：
  .map(
    (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
  );

结果：
  2026-05-03 完全不可用（空范围被过滤）
  其他工作日：09:00-17:00
```

#### 场景 C：日历忙碌 vs 现有预约

```
配置：
  工作时间：09:00-17:00
  日历事件：10:00-12:00（Google Calendar）
  现有预约：14:00-15:00（已确认 Booking）

处理流程：
  1. buildDateRanges(): 生成 [09:00-17:00]
  2. getBusyTimes(): 收集 [10:00-12:00, 14:00-15:00]
  3. subtract(): [09:00-17:00] - [10:00-12:00, 14:00-15:00]
     = [09:00-10:00, 12:00-14:00, 15:00-17:00]

结果：
  可用时间：09:00-10:00, 12:00-14:00, 15:00-17:00
```

#### 场景 D：缓冲时间扩展

```
配置：
  工作时间：09:00-17:00
  现有预约：10:00-11:00
  beforeEventBuffer: 15 分钟
  afterEventBuffer: 30 分钟

处理流程：
  1. 原始忙碌时间：[10:00-11:00]
  2. 应用缓冲：
     start = 10:00 - 15 分钟 = 09:45
     end = 11:00 + 30 分钟 = 11:30
  3. 扩展后忙碌时间：[09:45-11:30]
  4. subtract(): [09:00-17:00] - [09:45-11:30]
     = [09:00-09:45, 11:30-17:00]

结果：
  可用时间：09:00-09:45, 11:30-17:00
  注意：09:00-10:00 的槽位不可用（被前缓冲阻挡）
        11:00-12:00 的槽位不可用（被后缓冲阻挡）
```

#### 场景 E：预订限制（保守策略）

```
配置：
  工作时间：09:00-17:00
  bookingLimits: { PER_DAY: 2 }（每天最多 2 个预订）
  现有预订：10:00-11:00, 14:00-15:00（已达上限）

处理流程：
  1. 统计：当天已有 2 个预订
  2. 比较：count (2) >= limit (2)
  3. 标记：整个周期（当天）为忙碌
  4. subtract(): [09:00-17:00] - [00:00-24:00] = []

结果：
  当天剩余时间全部不可用
  注意：这是保守策略，即使 11:00-14:00 实际上有空
```

### 8.4 优先级判定总结

| 规则类型 | 影响方式 | 判定依据 |
|----------|----------|----------|
| **DateOverride** | 完全替换同日期的 WorkingHours | 对象展开顺序 |
| **OutOfOffice** | 标记为 `away: true`（UI 层处理） | 日期匹配（考虑时区） |
| **日历/预约** | 通过 `subtract()` 从可用范围减去 | 开放区间重叠 |
| **缓冲时间** | 扩展现有忙碌时间范围 | `start.subtract()`, `end.add()` |
| **预订限制** | 标记整个周期为忙碌 | `count >= limit` 或 `duration > limit` |
| **WorkingHours** | 基础可用范围 | 周期性展开 |

---

## 9. 多用户聚合策略

### 9.1 调度类型与聚合策略

根据 `SchedulingType` 采用不同的聚合策略：

| 调度类型 | 聚合策略 | 代码位置 | 关键操作 |
|----------|----------|----------|----------|
| **COLLECTIVE** | 交集计算 | `intersect()` | 所有用户同时可用 |
| **ROUND_ROBIN** | 独立范围保留 | 无 | 预约时选择可用主机 |
| **MANAGED** | 独立范围保留 | 无 | 管理员控制主机选择 |

### 9.2 COLLECTIVE（集体预约）详解

**策略**：取所有用户的日期范围**交集**

```typescript
// 只有当所有主机都可用时才显示为可用
const result = intersect([user1Ranges, user2Ranges, user3Ranges]);
```

**数学定义**：
```
可用性 = R1 ∩ R2 ∩ ... ∩ Rn
其中 Ri 是第 i 个用户的可用范围
```

**`intersect()` 算法详解**：

```typescript
// packages/features/schedules/lib/date-ranges.ts:354-416
export function intersect(ranges: DateRange[][]): DateRange[] {
  if (!ranges.length) return [];
  
  type ProcessedDateRange = DateRange & { startValue: number; endValue: number };
  
  // 预处理：排序所有用户的范围并缓存时间戳
  const sortedRanges: ProcessedDateRange[][] = ranges.map((userRanges) =>
    userRanges
      .map((r) => ({
        ...r,
        startValue: r.start.valueOf(),
        endValue: r.end.valueOf(),
      }))
      .sort((a, b) => a.startValue - b.startValue)
  );
  
  let commonAvailability = sortedRanges[0];  // 从第一个用户的范围开始
  
  for (let i = 1; i < sortedRanges.length; i++) {
    if (commonAvailability.length === 0) return []; // 早期退出：无交集
    
    const userRanges = sortedRanges[i];
    const intersectedRanges: ProcessedDateRange[] = [];
    
    let commonIndex = 0;
    let userIndex = 0;
    
    // 双指针归并算法
    while (commonIndex < commonAvailability.length && userIndex < userRanges.length) {
      const commonRange = commonAvailability[commonIndex];
      const userRange = userRanges[userIndex];
      
      // 计算交集
      const intersectStartValue = Math.max(commonRange.startValue, userRange.startValue);
      const intersectEndValue = Math.min(commonRange.endValue, userRange.endValue);
      
      if (intersectStartValue < intersectEndValue) {
        // 存在有效交集
        const intersectStart =
          commonRange.startValue > userRange.startValue ? commonRange.start : userRange.start;
        const intersectEnd = commonRange.endValue < userRange.endValue ? commonRange.end : userRange.end;
        intersectedRanges.push({
          start: intersectStart,
          end: intersectEnd,
          startValue: intersectStartValue,
          endValue: intersectEndValue,
        });
      }
      
      // 移动指针：哪个范围先结束就移动哪个
      if (commonRange.endValue <= userRange.endValue) {
        commonIndex++;
      } else {
        userIndex++;
      }
    }
    
    commonAvailability = intersectedRanges;
  }
  
  // 去掉缓存值
  return commonAvailability.map(({ start, end }) => ({ start, end }));
}
```

**算法复杂度**：
- 时间：O(n * m)，其中 n 是用户数，m 是每个用户的平均范围数
- 空间：O(k)，其中 k 是交集范围数

**关键特性**：
1. **早期退出**：如果任何一步交集为空，立即返回空数组
2. **双指针策略**：两个指针独立前进，不需要嵌套循环
3. **缓存时间戳**：预计算 `valueOf()` 避免重复计算

### 9.3 团队预约场景示例

假设有一个 3 人团队的集体预约事件：

```
用户A（销售）：
  工作时间：周一至周五 9:00-18:00
  忙碌时间：周二 10:00-12:00（团队会议）
           周四 14:00-16:00（客户拜访）

用户B（技术）：
  工作时间：周一至周五 10:00-19:00
  忙碌时间：周三 9:00-17:00（集中开发日）
           周五 10:00-12:00（代码评审）

用户C（产品）：
  工作时间：周一至周五 9:30-17:30
  忙碌时间：周一下午 14:00-18:00（产品规划）
           周三上午 9:00-12:00（需求评审）

请求：找一个所有人都可用的 1 小时时间段
```

**计算过程**：

1. **计算各用户净可用范围**：
   ```
   用户A净可用：
     周一: 9:00-18:00
     周二: 9:00-10:00, 12:00-18:00
     周三: 9:00-18:00
     周四: 9:00-14:00, 16:00-18:00
     周五: 9:00-18:00
   
   用户B净可用：
     周一: 10:00-19:00
     周二: 10:00-19:00
     周三: 无可用
     周四: 10:00-19:00
     周五: 12:00-19:00
   
   用户C净可用：
     周一: 9:30-14:00
     周二: 9:30-17:30
     周三: 12:00-17:30
     周四: 9:30-17:30
     周五: 9:30-17:30
   ```

2. **计算三者交集**：
   ```
   周一:
     A: 9:00-18:00
     B: 10:00-19:00
     C: 9:30-14:00
     交集: 10:00-14:00
   
   周二:
     A: 9:00-10:00, 12:00-18:00
     B: 10:00-19:00
     C: 9:30-17:30
     交集: 12:00-17:30
   
   周三:
     B 完全不可用 → 无交集
   
   周四:
     A: 9:00-14:00, 16:00-18:00
     B: 10:00-19:00
     C: 9:30-17:30
     交集: 10:00-14:00, 16:00-17:30
   
   周五:
     A: 9:00-18:00
     B: 12:00-19:00
     C: 9:30-17:30
     交集: 12:00-17:30
   ```

3. **最终可用时间槽（假设 60 分钟频率）**：
   ```
   周一: 10:00, 11:00, 12:00, 13:00
   周二: 12:00, 13:00, 14:00, 15:00, 16:00, 17:00
   周四: 10:00, 11:00, 12:00, 13:00, 16:00, 17:00
   周五: 12:00, 13:00, 14:00, 15:00, 16:00, 17:00
   ```

---

## 10. 性能优化与缓存策略

### 10.1 时间戳缓存

在 `intersect()` 和 `subtract()` 中使用时间戳缓存：

```typescript
// intersect 中的预处理
type ProcessedDateRange = DateRange & { startValue: number; endValue: number };

const sortedRanges: ProcessedDateRange[][] = ranges.map((userRanges) =>
  userRanges
    .map((r) => ({
      ...r,
      startValue: r.start.valueOf(),
      endValue: r.end.valueOf(),
    }))
    .sort((a, b) => a.startValue - b.startValue)
);

// 后续比较使用 startValue/endValue，避免重复调用 valueOf()
```

**优化效果**：
- `dayjs.valueOf()` 是一个相对昂贵的操作
- 通过预计算并缓存，在循环比较中避免重复调用
- 测试用例验证：2000 个日期范围的处理在 3 秒内完成

### 10.2 早期退出策略

在多处使用早期退出优化：

```typescript
// intersect 中的早期退出
for (let i = 1; i < sortedRanges.length; i++) {
  if (commonAvailability.length === 0) {
    return [];  // 如果没有交集，提前返回
  }
  // ...
}

// checkForConflicts 中的早期退出
if (!Array.isArray(busy) || busy.length < 1) {
  return false;  // 无忙碌时间，直接返回无冲突
}

// 循环中的早期退出
for (const busyTime of sortedBusyTimes) {
  if (busyTime.start >= slotEnd) {
    break;  // 忙碌时间在槽位之后，后续都不会冲突
  }
  // ...
}
```

### 10.3 结束时间到范围键的映射

在 `processWorkingHours()` 中使用 O(1) 查找：

```typescript
// 创建 endTime 到 range keys 的映射，用于 O(1) 查找
if (!endTimeToKeyMap) {
  endTimeToKeyMap = new Map<number, number[]>();
  for (const [key, range] of Object.entries(results)) {
    const endTime = range.end.valueOf();
    if (!endTimeToKeyMap.has(endTime)) {
      endTimeToKeyMap.set(endTime, []);
    }
    endTimeToKeyMap.get(endTime)!.push(Number(key));
  }
}

// 使用映射进行 O(1) 查找
const keysWithSameEndTime = endTimeToKeyMap.get(endTimeKey) || [];
```

### 10.4 Redis 缓存

时间槽计算使用 Redis 缓存：

```typescript
// packages/trpc/server/routers/viewer/slots/util.ts:104-141
function withSlotsCache(
  redisClient: IRedisService,
  func: (args: GetScheduleOptions) => Promise<IGetAvailableSlots>
) {
  return async (args: GetScheduleOptions): Promise<IGetAvailableSlots> => {
    const cacheKey = `${JSON.stringify(args.input)}`;
    
    // 尝试从缓存获取
    let cachedResult: IGetAvailableSlots | null = null;
    try {
      cachedResult = await redisClient.get(cacheKey);
    } catch (err: unknown) {
      // 缓存失败时直接执行函数
    }
    
    if (cachedResult) {
      log.info("[CACHE HIT] Available slots", { cacheKey });
      return cachedResult;
    }
    
    // 缓存未命中，执行计算
    const result = await func(args);
    
    // 异步写入缓存（不等待）
    const ttl = parseInt(process.env.SLOTS_CACHE_TTL ?? "", 10) || DEFAULT_SLOTS_CACHE_TTL;
    redisClient.set(cacheKey, result, { ttl });
    
    return result;
  };
}
```

---

## 11. 测试覆盖分析

### 11.1 核心测试文件

| 测试文件 | 覆盖范围 | 关键测试场景 |
|----------|----------|--------------|
| `slots.test.ts` | 时间槽生成逻辑 | 优化槽位、OOO、时区处理、边界情况 |
| `date-ranges.test.ts` | 日期范围处理 | `buildDateRanges`、`intersect`、`subtract`、DST 处理 |

### 11.2 关键测试场景

#### 场景 1：半小时偏移时区
```typescript
it("tests slots for half hour timezones", async () => {
  const slots = getSlots({
    inviteeDate: dayjs.tz("2023-07-13T00:00:00.000", "Asia/Kolkata"),
    frequency: 60,
    minimumBookingNotice: 0,
    eventLength: 60,
    dateRanges: [
      {
        start: dayjs.tz("2023-07-13T07:30:00.000", "Asia/Kolkata"),
        end: dayjs.tz("2023-07-13T09:30:00.000", "Asia/Kolkata"),
      },
    ],
  });

  expect(slots).toHaveLength(1);
  // 注意：07:30 开始，但对齐到 08:00（因为 30 % 60 ≠ 0，在 Kolkata 时区计算）
  expect(slots[0].time.format()).toBe("2023-07-13T08:00:00+05:30");
});
```

#### 场景 2：跨时区 OOO 处理
```typescript
it("should mark slots as away for cross-timezone OOO (Berlin OOO viewed from Kolkata)", async () => {
  // 验证时区转换正确，OOO 标记正确应用
});
```

#### 场景 3：DST 变更处理
```typescript
it("It has the correct working hours on date of DST change (- tz)", () => {
  vi.useFakeTimers().setSystemTime(new Date("2023-11-05T13:26:14.000Z"));
  
  const item = {
    days: [1, 2, 3, 4, 5],
    startTime: new Date(Date.UTC(2023, 5, 12, 9, 0)),
    endTime: new Date(Date.UTC(2023, 5, 12, 17, 0)),
  };
  
  const timeZone = "America/New_York";
  // ... 验证 DST 变更日的工作时间计算正确
});
```

#### 场景 4：intersect 综合测试
```typescript
// 团队预约场景：3 用户，找共同可用时间
it("should handle team scheduling scenario", () => {
  const user1Availability = [...];
  const user2Availability = [...];
  const user3Availability = [...];
  
  const result = intersect([user1Availability, user2Availability, user3Availability]);
  expect(result).toHaveLength(2);  // 两个共同可用时间段
});
```

---

## 12. 常见问题排查

### 12.1 时间槽数量少于预期

**可能原因**：
1. **日期覆盖优先级**：检查是否有 DateOverride 完全替换了工作日
2. **忙碌时间未考虑**：检查日历事件、现有预约、缓冲时间
3. **预订限制已达上限**：检查 bookingLimits/durationLimits
4. **最小预订通知时间**：当前时间 + minimumBookingNotice 之后才显示
5. **时区差异**：主机和预订者时区不同导致日期计算偏移

**排查步骤**：
1. 在 `getUserAvailability()` 后检查 `dateRanges`（基础可用范围）
2. 检查 `busy` 数组（忙碌时间来源）
3. 检查 `subtract()` 后的净可用范围
4. 验证时区转换是否正确

### 12.2 时间槽开始时间不符合预期

**可能原因**：
1. **showOptimizedSlots 配置**：true/false 有不同的对齐策略
2. **interval 选择**：根据 frequency 自动选择 60/30/20/15/10/5/1
3. **重叠范围调整**：前一个范围的槽位会影响后续范围
4. **时区分钟对齐**：在主机时区而非 UTC 进行对齐计算

### 12.3 团队预约没有可用时间

**可能原因**：
1. **调度类型错误**：COLLECTIVE 需要所有主机同时可用
2. **缺少交集**：检查各主机的可用范围是否有重叠
3. **日期覆盖冲突**：某主机在关键日期有 DateOverride

**排查步骤**：
1. 确认 `eventType.schedulingType`
2. 分别检查每个主机的 `dateRanges`
3. 手动计算 `intersect()` 预期结果

### 12.4 跨时区 OOO 标记不正确

**可能原因**：
1. **`datesOutOfOfficeTimeZone` 未正确传递**
2. **UTC 日期 vs 本地日期** 混淆

**关键代码检查**：
```typescript
// 正确：使用指定时区转换
const slotDateYYYYMMDD = datesOutOfOfficeTimeZone
  ? slotStartTime.tz(datesOutOfOfficeTimeZone).format("YYYY-MM-DD")
  : slotStartTime.utc().format("YYYY-MM-DD");
```

---

## 13. 附录

### 13.1 核心函数速查表

| 函数 | 位置 | 主要职责 |
|------|------|----------|
| `getSlots()` | `packages/features/schedules/lib/slots.ts` | 从日期范围生成具体时间槽 |
| `buildDateRanges()` | `packages/features/schedules/lib/date-ranges.ts` | 从工作时间/日期覆盖构建日期范围 |
| `subtract()` | `packages/features/schedules/lib/date-ranges.ts` | 从源范围减去排除范围 |
| `intersect()` | `packages/features/schedules/lib/date-ranges.ts` | 计算多个范围的交集 |
| `checkForConflicts()` | `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts` | 检测时间槽与忙碌时间的冲突 |
| `getUsersAvailability()` | `packages/features/availability/lib/getUserAvailability.ts` | 单用户完整可用性计算 |
| `getAvailableSlots()` | `packages/trpc/server/routers/viewer/slots/util.ts` | 多主机聚合+时间槽生成 |

### 13.2 数据结构参考

#### DateRange
```typescript
type DateRange = {
  start: Dayjs;  // 开始时间（含，开放区间）
  end: Dayjs;    // 结束时间（不含，开放区间）
};
```

#### WorkingHours
```typescript
type WorkingHours = {
  days: number[];           // 0=周日, 1=周一...6=周六
  startTime: Date;          // 开始时间（时分秒，日期部分忽略）
  endTime: Date;            // 结束时间
};
```

#### DateOverride
```typescript
type DateOverride = {
  date: Date;               // 特定日期
  startTime: Date;          // 该日期的开始时间
  endTime: Date;            // 该日期的结束时间
  // 特殊情况：startTime === endTime 表示该天完全不可用
};
```

#### GetSlots 参数
```typescript
type GetSlots = {
  inviteeDate: Dayjs;           // 被邀请者视角日期（时区参考）
  frequency: number;            // 槽位频率（分钟）
  dateRanges: DateRange[];      // 可用日期范围
  minimumBookingNotice: number; // 最小预订通知时间（分钟）
  eventLength: number;          // 事件时长（分钟）
  offsetStart?: number;         // 槽位开始偏移
  datesOutOfOffice?: IOutOfOfficeData;  // 外出数据
  showOptimizedSlots?: boolean; // 是否优化对齐
  datesOutOfOfficeTimeZone?: string;  // 外出数据时区
};
```

### 13.3 关键文件索引

| 文件路径 | 主要职责 |
|----------|----------|
| `packages/features/schedules/lib/slots.ts` | 时间槽生成核心 |
| `packages/features/schedules/lib/date-ranges.ts` | 日期范围处理（buildDateRanges, intersect, subtract） |
| `packages/features/availability/lib/getUserAvailability.ts` | 单用户可用性计算 |
| `packages/trpc/server/routers/viewer/slots/util.ts` | 可用时间槽服务（多主机聚合） |
| `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts` | 冲突检测 |
| `apps/api/v2/src/modules/slots/slots-2024-09-04/services/slots.service.ts` | API v2 时间槽服务 |
| `packages/features/schedules/lib/slots.test.ts` | 时间槽测试 |
| `packages/features/schedules/lib/date-ranges.test.ts` | 日期范围测试 |

---

*报告版本：v2.0*
*报告生成时间：2026-05-03*
*基于代码提交版本：当前工作区*