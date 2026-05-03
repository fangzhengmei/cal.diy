# 预约系统时间槽引擎分析报告

## 1. 整体架构概览

### 1.1 核心模块层次结构

时间槽生成系统采用分层架构，从底层的日期范围处理到顶层的可用时间槽计算，形成一个清晰的数据流管道：

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           AvailableSlotsService (顶层)                            │
│  - 整合所有数据源（工作时间、忙碌时间、限制等）                                      │
│  - 多主机聚合可用性计算                                                             │
│  - 预约限制和座位管理                                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      UserAvailabilityService (中间层)                             │
│  - 单用户可用性计算                                                                 │
│  - 工作时间 + 日期覆盖处理                                                         │
│  - 日历忙碌时间获取                                                                 │
│  - OOO（外出）和假期处理                                                           │
│  - 预订限制（bookingLimits/durationLimits）处理                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      基础工具层 (date-ranges.ts, slots.ts)                       │
│  - buildDateRanges(): 从工作时间/日期覆盖构建日期范围                             │
│  - intersect(): 多用户日期范围交集计算（团队预约）                                 │
│  - subtract(): 从可用范围减去忙碌范围                                             │
│  - getSlots(): 从日期范围生成具体时间槽                                           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键数据结构

#### DateRange（日期范围）
```typescript
type DateRange = {
  start: Dayjs;  // 开始时间
  end: Dayjs;    // 结束时间
};
```

这是系统中最基础的数据结构，所有时间计算都围绕此展开。

#### WorkingHours（工作时间规则）
```typescript
type WorkingHours = {
  days: number[];           // 星期几 (0-6, 0=周日)
  startTime: Date;          // 开始时间（时分部分）
  endTime: Date;            // 结束时间（时分部分）
};
```

周期性工作时间规则，适用于每周重复的模式。

#### DateOverride（日期覆盖）
```typescript
type DateOverride = {
  date: Date;               // 特定日期
  startTime: Date;          // 该日期的开始时间
  endTime: Date;            // 该日期的结束时间
};
```

用于覆盖特定日期的工作时间，可以是：
- **特殊可用性**：某天提前或延后工作
- **完全不可用**：`startTime === endTime` 表示该天完全不可用

---

## 2. 数据流程详解

### 2.1 完整的时间槽生成流程

```
输入参数
    │
    ▼
┌─────────────────┐
│ 1. 获取主机列表  │  动态事件类型 / 团队事件 / 循环赛
└────────┬────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│ 2. 为每个主机计算可用性 (getUsersAvailability)                │
│    ┌──────────────────────────────────────────────────────┐   │
│    │ 2.1 构建基础日期范围 (buildDateRanges)                │   │
│    │     - 解析 WorkingHours -> 日期范围                   │   │
│    │     - 应用 DateOverrides（优先级更高）                │   │
│    │     - 合并重叠范围                                     │   │
│    └──────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼                                    │
│    ┌──────────────────────────────────────────────────────┐   │
│    │ 2.2 收集忙碌时间                                       │   │
│    │     - 日历集成（Google/Outlook等）freebusy 数据       │   │
│    │     - 现有预约（Bookings）                            │   │
│    │     - 事件前后缓冲时间                                │   │
│    │     - 预订限制导致的"忙碌"                            │   │
│    └──────────────────────────────────────────────────────┘   │
│                           │                                    │
│                           ▼                                    │
│    ┌──────────────────────────────────────────────────────┐   │
│    │ 2.3 计算净可用范围                                     │   │
│    │     subtract(dateRanges, busyTimes)                   │   │
│    │     从可用范围减去所有忙碌范围                          │   │
│    └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│ 3. 多主机聚合可用性 (getAggregatedAvailability)               │
│    根据调度类型（SchedulingType）选择策略：                   │
│                                                               │
│    - COLLECTIVE（集体）: intersect(allUserRanges)            │
│      找到所有主机都可用的时间范围                              │
│                                                               │
│    - ROUND_ROBIN（循环赛）: 保留各主机独立范围               │
│      后续在预约时选择可用主机                                 │
│                                                               │
│    - MANAGED（托管）: 类似于循环赛，但由管理员控制           │
└──────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│ 4. 生成时间槽 (getSlots)                                      │
│    - 根据频率（frequency）/ 事件时长（eventLength）          │
│    - 计算槽位开始时间                                         │
│    - 应用最小预订通知时间                                     │
│    - 标记 OOO 时间槽为 "away"                                 │
└──────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│ 5. 后处理过滤                                                 │
│    - 检查与已预留槽位的冲突                                   │
│    - 应用座位预约逻辑（seatsPerTimeSlot）                    │
│    - 过滤超出请求日期范围的槽位                               │
│    - 检查未来预订限制                                         │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 规则优先级系统

### 3.1 优先级层次结构

时间槽引擎采用**严格的优先级层次**，从最高到最低依次为：

| 优先级 | 规则类型 | 说明 | 处理位置 |
|--------|----------|------|----------|
| **1 (最高)** | 日期覆盖 (DateOverride) | 特定日期的可用性覆盖，可完全取消某天可用性 | `buildDateRanges()` |
| **2** | 外出 (OutOfOffice) | 用户标记的外出日期，完全不可用 | `buildDateRanges()`, `getSlots()` |
| **3** | 假期 (Holidays) | 根据国家/地区设置的公共假期 | `calculateHolidayBlockedDates()` |
| **4** | 日历忙碌时间 | 从 Google/Outlook 等日历获取的事件 | `getBusyTimes()` |
| **5** | 现有预约 | 已确认的预订 | `getBusyTimes()` |
| **6** | 前后缓冲时间 | 事件前后需要的缓冲时间 | `getBusyTimes()` |
| **7** | 预订限制 | 每天/每周/每月最多预订数 | `getBusyTimesFromLimits()` |
| **8** | 时长限制 | 每天/每周/每月最多预订时长 | `getBusyTimesFromLimits()` |
| **9 (最低)** | 常规工作时间 | 周期性的工作时间规则 | `buildDateRanges()` |

### 3.2 日期覆盖 vs 工作时间的交互

这是最关键的优先级关系，在 `buildDateRanges()` 中实现：

```typescript
// packages/features/schedules/lib/date-ranges.ts:312-330
const dateRanges = Object.values({
  ...groupedWorkingHours,
  ...groupedDateOverrides,  // 日期覆盖展开在工作时间之后
  // 对象展开时，后面的属性会覆盖前面的同名键
}).map(
  (ranges) => ranges.filter((range) => range.start.valueOf() !== range.end.valueOf())
);
```

**关键点**：
1. 使用对象展开（spread）语法，`groupedDateOverrides` 在 `groupedWorkingHours` 之后
2. 这意味着对于同一天，日期覆盖会**完全替换**工作时间
3. 空范围（start === end）表示该天**完全不可用**，在过滤阶段被移除

**示例场景**：
```
常规工作时间：周一至周五 9:00-17:00
日期覆盖：2023-06-13 10:00-15:00（特殊时间）
日期覆盖：2023-06-14 00:00-00:00（完全不可用）

结果：
- 2023-06-12: 9:00-17:00（常规工作时间）
- 2023-06-13: 10:00-15:00（日期覆盖）
- 2023-06-14: 无可用时间（日期覆盖为空范围）
- 2023-06-15: 9:00-17:00（常规工作时间）
```

### 3.3 工作时间范围的合并逻辑

在 `processWorkingHours()` 中，存在智能的范围合并机制：

#### 情况1：同一起点的范围合并
```typescript
// packages/features/schedules/lib/date-ranges.ts:128-156
if (results[startResult.valueOf()]) {
  // 如果一个结果已存在，合并结束时间
  const oldKey = startResult.valueOf();
  const newKey = endResult.valueOf();
  
  results[newKey] = {
    start: results[oldKey].start,
    end: dayjs.max(results[oldKey].end, endResult),
  };
  delete results[oldKey]; // 删除之前的结束时间
}
```

**示例**：
```
范围1: 周一 9:00-12:00
范围2: 周一 9:00-17:00
结果:  周一 9:00-17:00（合并）
```

#### 情况2：同一终点的范围合并
```typescript
// packages/features/schedules/lib/date-ranges.ts:104-126
// 使用 endTimeToKeyMap 进行 O(1) 查找相同结束时间的范围
const keysWithSameEndTime = endTimeToKeyMap.get(endTimeKey) || [];
for (const key of keysWithSameEndTime) {
  const existingRange = results[key];
  if (startResult.valueOf() <= existingRange.end.valueOf() &&
      endResult.valueOf() >= existingRange.start.valueOf()) {
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
范围1: 周一 6:00-10:00
范围2: 周一 8:00-10:00
结果:  周一 6:00-10:00（合并为更宽的范围）
```

---

## 4. 多用户可用性聚合

### 4.1 调度类型与聚合策略

在 `getAggregatedAvailability()` 中根据 `SchedulingType` 采用不同策略：

#### COLLECTIVE（集体预约）
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

**示例**：
```
用户A: 9:00-12:00, 14:00-17:00
用户B: 10:00-15:00
用户C: 11:00-16:00

交集: 11:00-12:00, 14:00-15:00
```

#### ROUND_ROBIN（循环赛）
**策略**：保留各用户的独立范围，预约时选择可用主机

```typescript
// 不做交集计算，保留所有用户的范围
// 在实际预约时根据负载、优先级等选择主机
```

#### MANAGED（托管）
**策略**：类似于循环赛，但由管理员控制主机选择

### 4.2 intersect() 算法详解

`intersect()` 函数是团队预约的核心，采用**双指针归并**算法：

```typescript
// packages/features/schedules/lib/date-ranges.ts:354-416
export function intersect(ranges: DateRange[][]): DateRange[] {
  if (!ranges.length) return [];
  
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
  
  let commonAvailability = sortedRanges[0];
  
  for (let i = 1; i < sortedRanges.length; i++) {
    if (commonAvailability.length === 0) return []; // 早期退出
    
    const userRanges = sortedRanges[i];
    const intersectedRanges: ProcessedDateRange[] = [];
    
    let commonIndex = 0;
    let userIndex = 0;
    
    while (commonIndex < commonAvailability.length && userIndex < userRanges.length) {
      const commonRange = commonAvailability[commonIndex];
      const userRange = userRanges[userIndex];
      
      // 计算交集
      const intersectStartValue = Math.max(commonRange.startValue, userRange.startValue);
      const intersectEndValue = Math.min(commonRange.endValue, userRange.endValue);
      
      if (intersectStartValue < intersectEndValue) {
        // 存在有效交集
        intersectedRanges.push({
          start: commonRange.startValue > userRange.startValue ? commonRange.start : userRange.start,
          end: commonRange.endValue < userRange.endValue ? commonRange.end : userRange.end,
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

### 4.3 团队预约场景示例

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

## 5. 忙碌时间处理与 subtract() 算法

### 5.1 忙碌时间的来源

忙碌时间从多个渠道收集，在 `getBusyTimes()` 中整合：

| 来源 | 说明 | 处理位置 |
|------|------|----------|
| **日历集成** | Google Calendar、Outlook 等的 freebusy 数据 | `getCalendar().getAvailability()` |
| **现有预约** | 已确认的 Booking 记录 | `bookingRepo.findAllExistingBookings()` |
| **缓冲时间** | `beforeEventBuffer` / `afterEventBuffer` | 计算忙碌时间时扩展 |
| **预订限制** | 每天/每周最多 N 个预约 | `getBusyTimesFromLimits()` |
| **时长限制** | 每天/每周最多 N 分钟 | `getBusyTimesFromLimits()` |
| **预留槽位** | 其他用户临时预留的槽位 | `checkForConflicts()` |

### 5.2 缓冲时间的应用

在获取忙碌时间时，会自动应用事件前后缓冲：

```typescript
// packages/features/busyTimes/services/getBusyTimes.ts (示意)
busyTimes = busyTimes.map((busy) => ({
  start: dayjs(busy.start).subtract(beforeEventBuffer, 'minute'),
  end: dayjs(busy.end).add(afterEventBuffer, 'minute'),
}));
```

**示例**：
```
事件配置：
  - 会议时长：60 分钟
  - beforeEventBuffer：15 分钟
  - afterEventBuffer：30 分钟

现有预约：
  - 10:00-11:00（团队会议）

实际忙碌范围：
  - 09:45-11:30（扩展了缓冲时间）

可用槽位影响：
  - 09:00-10:00：不可用（被前缓冲阻挡）
  - 11:00-12:00：不可用（被后缓冲阻挡）
  - 11:30之后：可用
```

### 5.3 subtract() 算法详解

`subtract()` 函数是从可用范围中减去忙碌范围的核心算法：

```typescript
// packages/features/schedules/lib/date-ranges.ts:423-452
export function subtract<TSourceRange extends DateRange, TExcludedRange extends DateRange>(
  sourceRanges: TSourceRange[],
  excludedRanges: TExcludedRange[]
): SubtractedRange<TSourceRange>[] {
  const result: SubtractedRange<TSourceRange>[] = [];
  
  // 首先对排除范围进行排序
  const sortedExcludedRanges = [...excludedRanges].sort(
    (a, b) => a.start.valueOf() - b.start.valueOf()
  );
  
  for (const { start: sourceStart, end: sourceEnd, ...passThrough } of sourceRanges) {
    let currentStart = sourceStart;
    
    for (const excludedRange of sortedExcludedRanges) {
      // 优化1：如果排除范围在源范围之后，提前退出
      if (excludedRange.start.valueOf() >= sourceEnd.valueOf()) break;
      
      // 优化2：如果排除范围在当前起始点之前，跳过
      if (excludedRange.end.valueOf() <= currentStart.valueOf()) continue;
      
      // 情况A：排除范围的开始在当前起始点之后
      // 这意味着中间有一段可用时间
      if (excludedRange.start.valueOf() > currentStart.valueOf()) {
        result.push({ start: currentStart, end: excludedRange.start, ...passThrough });
      }
      
      // 情况B：排除范围的结束在当前起始点之后
      // 更新当前起始点到排除范围结束之后
      if (excludedRange.end.valueOf() > currentStart.valueOf()) {
        currentStart = excludedRange.end;
      }
    }
    
    // 最后：检查源范围结束之前是否还有剩余可用时间
    if (sourceEnd.valueOf() > currentStart.valueOf()) {
      result.push({ start: currentStart, end: sourceEnd, ...passThrough });
    }
  }
  
  return result;
}
```

### 5.4 subtract() 场景示例

#### 场景1：忙碌范围完全在可用范围中间
```
可用范围:  09:00 ───────────────────────────── 17:00
忙碌范围:           11:00 ───── 13:00
结果:      09:00 ── 11:00    13:00 ───── 17:00
           (可用)     (忙碌)      (可用)
```

#### 场景2：忙碌范围覆盖可用范围开头
```
可用范围:  09:00 ───────────────────────────── 17:00
忙碌范围:  08:00 ───── 10:00
结果:                10:00 ─────────────────── 17:00
                      (可用)
```

#### 场景3：忙碌范围覆盖可用范围结尾
```
可用范围:  09:00 ───────────────────────────── 17:00
忙碌范围:                15:00 ───── 18:00
结果:      09:00 ─────────────────── 15:00
           (可用)
```

#### 场景4：忙碌范围完全覆盖可用范围
```
可用范围:  09:00 ───────────────────────────── 17:00
忙碌范围:  08:00 ───────────────────────────── 18:00
结果:      (无可用时间)
```

#### 场景5：多个部分重叠的忙碌范围
```
可用范围:  09:00 ───────────────────────────────────── 18:00
忙碌范围1:         10:00 ───── 12:00
忙碌范围2:                  11:30 ───── 14:00
忙碌范围3:                           15:00 ───── 17:00

排序后排除范围: [10:00-12:00, 11:30-14:00, 15:00-17:00]

计算过程:
currentStart = 09:00

处理 10:00-12:00:
  - 10:00 > 09:00 → 结果添加 09:00-10:00
  - currentStart = 12:00

处理 11:30-14:00:
  - 14:00 > 12:00 → currentStart = 14:00

处理 15:00-17:00:
  - 15:00 > 14:00 → 结果添加 14:00-15:00
  - currentStart = 17:00

循环结束:
  - 18:00 > 17:00 → 结果添加 17:00-18:00

最终结果:
  09:00-10:00, 14:00-15:00, 17:00-18:00
```

### 5.5 预订限制的特殊处理

预订限制（bookingLimits 和 durationLimits）是一种"虚拟"的忙碌时间，在 `getBusyTimesFromLimits()` 中计算：

```typescript
// packages/features/busyTimes/lib/getBusyTimesFromLimits.ts (示意)
export async function getBusyTimesFromLimits(
  bookingLimits,
  durationLimits,
  dateFrom,
  dateTo,
  duration,
  eventType,
  existingBookings,
  timeZone
) {
  const busyTimes = [];
  const limitManager = new LimitManager();
  
  // 处理预订数量限制
  if (bookingLimits) {
    for (const period in bookingLimits) {
      const limit = bookingLimits[period];
      // 计算该时间段内的已有预订数
      const existingCount = countBookingsInPeriod(existingBookings, period);
      
      if (existingCount >= limit) {
        // 已达上限，标记整个时间段为忙碌
        limitManager.addBusyTime({
          start: periodStart,
          unit: period,
          timeZone,
          title: `已达到 ${limit} 个预订限制`,
          source: LimitSources.eventBookingLimit({ limit, unit: period }),
        });
      }
    }
  }
  
  // 处理时长限制（类似逻辑）
  if (durationLimits) {
    // ... 计算已用时长，超过则标记为忙碌
  }
  
  return limitManager.getBusyTimes();
}
```

**示例场景**：
```
事件配置：
  - 每天最多 2 个预订（bookingLimits: { PER_DAY: 2 }）
  - 工作时间：09:00-17:00

现有预订：
  - 10:00-11:00（已确认）
  - 14:00-15:00（已确认）

结果：
  - 当天剩余时间全部标记为"忙碌"
  - 原因：已达到每天 2 个预订的上限
```

---

## 6. 时间槽生成算法 (getSlots)

### 6.1 核心参数解析

`getSlots()` 函数接收以下关键参数：

```typescript
type GetSlots = {
  inviteeDate: Dayjs;           // 被邀请者视角的日期（用于时区）
  frequency: number;            // 时间槽频率（分钟），如 30, 60
  dateRanges: DateRange[];      // 可用日期范围（已减去忙碌时间）
  minimumBookingNotice: number; // 最小预订通知时间（分钟）
  eventLength: number;          // 事件时长（分钟）
  offsetStart?: number;         // 槽位开始偏移（分钟）
  datesOutOfOffice?: IOutOfOfficeData; // 外出日期数据
  showOptimizedSlots?: boolean; // 是否显示优化槽位（对齐到整点）
  datesOutOfOfficeTimeZone?: string; // 外出日期的时区
};
```

### 6.2 时间槽频率 vs 事件时长

这两个参数有重要区别：

| 参数 | 说明 | 示例 |
|------|------|------|
| `frequency` | 槽位之间的间隔 | 30 分钟：每 30 分钟显示一个槽位 |
| `eventLength` | 每个槽位的实际时长 | 60 分钟：每个槽位代表 1 小时的会议 |

**组合示例**：
```
配置：
  frequency = 30（每 30 分钟一个槽位）
  eventLength = 60（每个会议 60 分钟）
  可用范围 = 09:00-11:00

生成的槽位：
  09:00（实际占用 09:00-10:00）
  09:30（实际占用 09:30-10:30）
  10:00（实际占用 10:00-11:00）

注意：虽然 frequency 是 30 分钟，但每个槽位实际占用 60 分钟，
      因此相邻槽位会有时间重叠。用户选择后会阻塞相应的时间段。
```

### 6.3 槽位开始时间对齐算法

槽位开始时间的对齐是一个复杂的逻辑，在 `getCorrectedSlotStartTime()` 中处理：

```typescript
// packages/features/schedules/lib/slots.ts:27-69
function getCorrectedSlotStartTime({
  slotStartTime,
  range,
  showOptimizedSlots,
  interval,
}: {
  showOptimizedSlots: boolean | null | undefined;
  interval: number;
  slotStartTime: Dayjs;
  range: DateRange;
}) {
  if (showOptimizedSlots) {
    let correctedSlotStartTime = slotStartTime;
    
    // 计算需要移动到下一个槽位的分钟数
    const minutesRequiredToMoveToNextSlot = interval - (slotStartTime.minute() % interval);
    const minutesRequiredToMoveTo15MinSlot = 15 - (slotStartTime.minute() % 15);
    const minutesRequiredToMoveTo5MinSlot = 5 - (slotStartTime.minute() % 5);
    
    // 计算范围中在"最大可能槽位"后剩余的时间
    const extraMinutesAvailable = range.end.diff(slotStartTime, "minutes") % interval;
    
    // 策略1：如果有足够时间推到下一个完整间隔
    if (extraMinutesAvailable >= minutesRequiredToMoveToNextSlot) {
      // 例如：可用时间 09:05-12:00，60分钟事件
      // 总可用 175 分钟，最多 2 个 60 分钟槽位
      // 剩余 175-120=55 分钟，足够推到 10:00
      // 结果：槽位显示为 10:00, 11:00 而不是 09:05, 10:05
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveToNextSlot, "minute");
    }
    // 策略2：次之，推到下一个 15 分钟边界
    else if (extraMinutesAvailable >= minutesRequiredToMoveTo15MinSlot) {
      // 例如：09:05-11:55，中间有 10 分钟忙碌
      // 推到 09:15
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveTo15MinSlot, "minute");
    }
    // 策略3：最次之，推到下一个 5 分钟边界
    else if (extraMinutesAvailable >= minutesRequiredToMoveTo5MinSlot) {
      // 例如：11:22 推到 11:25
      correctedSlotStartTime = slotStartTime.add(minutesRequiredToMoveTo5MinSlot, "minute");
    }
    
    return correctedSlotStartTime;
  }
  
  // 非优化模式：简单对齐到 interval 边界
  return slotStartTime.startOf("hour").add(
    Math.ceil(slotStartTime.minute() / interval) * interval,
    "minute"
  );
}
```

### 6.4 优化槽位 vs 非优化槽位对比

通过测试用例可以清晰看到差异：

#### 测试用例：可用时间 09:30-17:30，60 分钟事件

**showOptimizedSlots = true**：
```
生成的槽位（8个）：
  09:30, 10:30, 11:30, 12:30, 13:30, 14:30, 15:30, 16:30

逻辑：范围正好是 8 小时，可以容纳 8 个完整的 60 分钟槽位
     从 09:30 开始正好能放满，不需要调整
```

**showOptimizedSlots = false**：
```
生成的槽位（7个）：
  10:00, 11:00, 12:00, 13:00, 14:00, 15:00, 16:00

逻辑：简单对齐到整点
     09:30 不是整点，向上取整到 10:00
     最后一个 16:00 开始的槽位结束于 17:00，在 17:30 之前
     相比优化模式少了一个槽位
```

### 6.5 间隔选择逻辑

系统会根据 `frequency` 自动选择合适的间隔：

```typescript
// packages/features/schedules/lib/slots.ts:113-121
let interval = Number(process.env.NEXT_PUBLIC_AVAILABILITY_SCHEDULE_INTERVAL) || 1;
const intervalsWithDefinedStartTimes = [60, 30, 20, 15, 10, 5];

for (let i = 0; i < intervalsWithDefinedStartTimes.length; i++) {
  if (frequency % intervalsWithDefinedStartTimes[i] === 0) {
    interval = intervalsWithDefinedStartTimes[i];
    break;
  }
}
```

**优先级**：60 → 30 → 20 → 15 → 10 → 5 → 1（默认）

**示例**：
```
frequency = 45 分钟
45 % 60 ≠ 0
45 % 30 ≠ 0
45 % 20 ≠ 0
45 % 15 = 0 ✓  → 选择 interval = 15

这意味着槽位会按 15 分钟边界对齐
```

### 6.6 重叠日期范围的槽位去重

当存在重叠的日期范围时，系统会智能处理避免重复槽位：

```typescript
// packages/features/schedules/lib/slots.ts:127-176
orderedDateRanges.forEach((range) => {
  // ... 计算 slotStartTime ...
  
  // 检查是否与已生成的槽位边界重叠
  const slotBoundariesValueArray = Array.from(slotBoundaries.keys());
  if (slotBoundariesValueArray.length > 0) {
    slotBoundariesValueArray.sort((a, b) => a - b);
    
    let prevBoundary = null;
    for (let i = slotBoundariesValueArray.length - 1; i >= 0; i--) {
      if (slotBoundariesValueArray[i] < slotStartTime.valueOf()) {
        prevBoundary = slotBoundariesValueArray[i];
        break;
      }
    }
    
    if (prevBoundary) {
      const prevBoundaryEnd = dayjs(prevBoundary).add(frequency + (offsetStart ?? 0), "minutes");
      if (prevBoundaryEnd.isAfter(slotStartTime)) {
        // 当前范围与前一个槽位的时间重叠
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
  
  // ... 生成槽位 ...
});
```

**场景示例**：
```
两个重叠的可用范围：
  范围1: 11:00-12:00
  范围2: 11:20-12:40

frequency = 45 分钟

处理范围1：
  生成槽位: 11:00（占用 11:00-11:45）
  下一个槽位会在 11:45，但范围1在 12:00 结束
  11:45 + 45 分钟 = 12:30，超出 12:00
  范围1只生成 1 个槽位

处理范围2：
  初始 slotStartTime = 11:20
  检查到前一个边界 11:00，其结束时间 = 11:00 + 45 = 11:45
  11:45 在 11:20 之后 → 需要调整
  slotStartTime = 11:45
  
  从 11:45 开始：
    11:45 + 45 = 12:30，在 12:40 之前
    生成槽位: 11:45

最终结果：
  11:00（来自范围1）
  11:45（来自范围2，跳过了重叠部分）
  
注意：没有生成重复或重叠的槽位
```

### 6.7 最小预订通知时间的应用

```typescript
// packages/features/schedules/lib/slots.ts:123-133
const startTimeWithMinNotice = dayjs.utc().add(minimumBookingNotice, "minute");

orderedDateRanges.forEach((range) => {
  let slotStartTime = range.start.utc().isAfter(startTimeWithMinNotice)
    ? range.start
    : startTimeWithMinNotice;
  
  // 对于当天预订，将秒数归零以避免时间计算问题
  slotStartTime = slotStartTime.set("second", 0).set("millisecond", 0);
  // ...
});
```

**示例**：
```
当前时间：12:00 UTC
minimumBookingNotice = 120 分钟（2 小时）

可用范围：今天 09:00-18:00

计算：
  startTimeWithMinNotice = 12:00 + 2 小时 = 14:00
  范围开始 09:00 < 14:00 → 实际从 14:00 开始

生成的槽位（假设 60 分钟事件）：
  14:00, 15:00, 16:00, 17:00
  
注意：14:00 之前的时间不显示，因为预订通知时间不足
```

### 6.8 外出时间槽的标记

在生成槽位时，会检查是否为外出日期并相应标记：

```typescript
// packages/features/schedules/lib/slots.ts:187-222
let dateOutOfOfficeExists = undefined;
if (datesOutOfOffice) {
  const slotDateYYYYMMDD = datesOutOfOfficeTimeZone
    ? slotStartTime.tz(datesOutOfOfficeTimeZone).format("YYYY-MM-DD")
    : slotStartTime.utc().format("YYYY-MM-DD");
  dateOutOfOfficeExists = datesOutOfOffice?.[slotDateYYYYMMDD];
}

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
    // ...
  };
}
```

**关键逻辑**：
1. 考虑 `datesOutOfOfficeTimeZone` 时区进行日期转换
2. 这对于跨时区场景很重要（如主机在洛杉矶，预订者在加尔各答）

---

## 7. 冲突检测与处理

### 7.1 冲突检测核心算法

`checkForConflicts()` 函数用于检测时间槽是否与忙碌时间冲突：

```typescript
// packages/features/bookings/lib/conflictChecker/checkForConflicts.ts:10-50
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
  // 早期返回：无忙碌时间则无冲突
  if (!Array.isArray(busy) || busy.length < 1) {
    return false;
  }
  
  // 特殊处理：座位事件 - 如果当前时间槽有座位数据，则无冲突
  // （座位事件允许多个预订，直到座位满）
  if (currentSeats?.some((booking) => booking.startTime.toISOString() === time.toISOString())) {
    return false;
  }
  
  // 计算槽位的时间范围
  const slotStart = time.valueOf();
  const slotEnd = slotStart + eventLength * 60 * 1000;
  
  // 预处理：排序忙碌时间（按开始时间）
  const sortedBusyTimes = busy
    .map((busyTime) => ({
      start: dayjs.utc(busyTime.start).valueOf(),
      end: dayjs.utc(busyTime.end).valueOf(),
    }))
    .sort((a, b) => a.start - b.start);
  
  // 遍历检查冲突
  for (const busyTime of sortedBusyTimes) {
    // 优化：如果忙碌时间在槽位结束之后，可提前退出
    if (busyTime.start >= slotEnd) {
      break;
    }
    // 优化：如果忙碌时间在槽位开始之前，跳过
    if (busyTime.end <= slotStart) {
      continue;
    }
    // 到这里说明存在时间重叠 → 冲突
    return true;
  }
  
  return false;
}
```

### 7.2 冲突判定逻辑

时间重叠的判定采用**开放区间**模型：

```
时间范围比较：
  槽位：  [slotStart, slotEnd)
  忙碌：  [busy.start, busy.end)

冲突条件（满足任一即可）：
  1. busy.start < slotEnd AND busy.end > slotStart
  
等价于（德摩根定律）：
  NOT (busy.start >= slotEnd OR busy.end <= slotStart)
```

**冲突场景图示**：

```
场景1：部分重叠（最常见）
  槽位：    10:00 ───────────── 11:00
  忙碌：         10:30 ───────────── 11:30
  结果：冲突 ✓

场景2：完全包含
  槽位：    10:00 ───────────────────── 12:00
  忙碌：         10:30 ───────── 11:30
  结果：冲突 ✓

场景3：被包含
  槽位：         10:30 ───────── 11:30
  忙碌：    10:00 ───────────────────── 12:00
  结果：冲突 ✓

场景4：刚好接触（边界相连）
  槽位：    10:00 ───────────── 11:00
  忙碌：                        11:00 ───────────── 12:00
  结果：无冲突 ✗
  说明：11:00 是槽位的结束（开放区间不包含），也是忙碌的开始
        所以 11:00 的会议和 11:00 开始的忙碌不冲突

场景5：完全分离
  槽位：    10:00 ───────────── 11:00
  忙碌：                              13:00 ───────────── 14:00
  结果：无冲突 ✗
```

### 7.3 座位事件的特殊处理

座位事件（seatsPerTimeSlot）是一种特殊的事件类型，允许多个预订：

```typescript
// checkForConflicts 中的座位处理
if (currentSeats?.some((booking) => booking.startTime.toISOString() === time.toISOString())) {
  return false; // 有座位数据 → 不认为是冲突
}
```

**座位事件逻辑**：
```
普通事件（无座位）：
  - 一个时间槽只能有一个预订
  - 任何重叠都视为冲突

座位事件（seatsPerTimeSlot = 5）：
  - 一个时间槽可以有多个预订，最多 5 个参与者
  - currentSeats 跟踪每个槽位的已预订人数
  - 只有当实际人数 >= 座位数时才不可用
  - 冲突检测在更高层处理（比较 _count.attendees 和 seatsPerTimeSlot）
```

### 7.4 预留槽位的冲突检测

除了常规忙碌时间，还需要检查其他用户临时预留的槽位：

```typescript
// packages/trpc/server/routers/viewer/slots/util.ts:1166-1244
// 获取其他用户预留的槽位
const reservedSlots = await this._getReservedSlotsAndCleanupExpired({
  bookerClientUid,
  eventTypeId: eventType.id,
  usersWithCredentials,
});

// ...

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

### 7.5 预订限制检测（座位事件）

在 `SlotsService_2024_09_04.reserveSlot()` 中：

```typescript
// apps/api/v2/src/modules/slots/slots-2024-09-04/services/slots.service.ts:139-149
if (eventType.seatsPerTimeSlot) {
  const attendeesCount = booking?.attendees?.length;
  if (attendeesCount) {
    const seatsLeft = eventType.seatsPerTimeSlot - attendeesCount;
    if (seatsLeft < 1) {
      throw new UnprocessableEntityException(
        `Booking with id=${input.eventTypeId} at ${input.slotStart} has no more seats left.`
      );
    }
  }
}
```

---

## 8. 时区处理与边界情况

### 8.1 时区转换的关键位置

时区处理是系统中最复杂的部分之一，在多个关键点进行转换：

#### 1. 槽位开始时间计算中的时区转换
```typescript
// packages/features/schedules/lib/slots.ts:135-147
// 在检查是否需要取整之前，转换到目标时区
// 这确保我们在本地时区检查分钟对齐，而不是 UTC
// 这防止了像 Asia/Kolkata (GMT+5:30) 这样的半小时偏移时区的问题
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

**问题场景**：
```
假设：
  - 主机在加尔各答（Asia/Kolkata, +05:30）
  - 工作时间：09:00-17:00（加尔各答时间）
  - 事件时长：60 分钟

如果在 UTC 时间计算：
  加尔各答 09:00 = UTC 03:30
  slotStartTime.minute() = 30
  interval = 60
  30 % 60 ≠ 0 → 会错误地调整到下一个边界

但实际上在加尔各答时区：
  slotStartTime.minute() = 0
  0 % 60 = 0 → 不需要调整
```

#### 2. 外出日期的时区处理
```typescript
// packages/features/schedules/lib/slots.ts:188-193
if (datesOutOfOffice) {
  const slotDateYYYYMMDD = datesOutOfOfficeTimeZone
    ? slotStartTime.tz(datesOutOfOfficeTimeZone).format("YYYY-MM-DD")
    : slotStartTime.utc().format("YYYY-MM-DD");
  dateOutOfOfficeExists = datesOutOfOffice?.[slotDateYYYYMMDD];
}
```

**跨时区场景**（来自测试用例）：
```
主机：
  - 时区：America/Los_Angeles（UTC-8 或 UTC-7）
  - 外出日期：2026-01-16（洛杉矶本地日期）

预订者：
  - 时区：Asia/Kolkata（UTC+5:30）
  - 查看时间：2026-01-17 05:30（加尔各答时间）

时间转换：
  洛杉矶 2026-01-16 16:00 = 加尔各答 2026-01-17 05:30
  
  虽然预订者看到的是 1月17日，但主机实际上是在 1月16日（洛杉矶时间）
  如果只看 UTC 或预订者时区，会错误地认为不是外出日期

正确处理：
  使用 datesOutOfOfficeTimeZone = "America/Los_Angeles" 进行转换
  slotStartTime.tz("America/Los_Angeles").format("YYYY-MM-DD") = "2026-01-16"
  正确识别为外出日期
```

### 8.2 半小时偏移时区的特殊处理

测试用例专门验证了这一点：

```typescript
// packages/features/schedules/lib/slots.test.ts:269-285
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

### 8.3 夏令时（DST）变更处理

在 `processWorkingHours()` 中处理 DST 变更：

```typescript
// packages/features/schedules/lib/date-ranges.ts:70-74
const offsetBeginningOfDay = dayjs(start.format("YYYY-MM-DD hh:mm")).tz(adjustedTimezone).utcOffset();
const offsetDiff = start.utcOffset() - offsetBeginningOfDay; 
// 在 DST 变更的那天会有 60 分钟的偏移差

start = start.add(offsetDiff, "minute");
end = end.add(offsetDiff, "minute");
```

**问题场景**：
```
假设：
  - 时区：America/New_York
  - 工作时间：09:00-17:00
  - DST 变更：3 月第二个周日 02:00 → 03:00（春天向前拨）
              11 月第一个周日 02:00 → 01:00（秋天向后拨）

春天 DST 变更日（向前拨 1 小时）：
  - 当天实际上只有 23 小时
  - 02:00-03:00 这个时间段不存在
  - 如果不处理，09:00 的工作时间可能会错误地偏移

秋天 DST 变更日（向后拨 1 小时）：
  - 当天有 25 小时
  - 01:00-02:00 出现两次
  - 需要确保时间计算的一致性
```

### 8.4 跨午夜可用性

系统支持跨越午夜的可用性，通过合并相邻日期范围实现：

```typescript
// 测试用例示例：packages/features/schedules/lib/date-ranges.test.ts:583-652
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

### 8.5 23:59 结束时间的特殊处理

用户通常设置可用性到 23:59，但实际上应该允许到午夜：

```typescript
// packages/features/schedules/lib/date-ranges.ts:81-83
// INFO: 我们只允许用户设置到晚上 11:59 的可用性，
// 这实际上导致他们不能使用到午夜。
if (endResult.hour() === 23 && endResult.minute() === 59) {
  endResult = endResult.add(1, "minute");
}
```

**示例**：
```
用户设置：
  周一 09:00-23:59

实际处理：
  转换为 09:00-24:00（即下一天 00:00）

影响：
  - 23:30 开始的 60 分钟会议可以被接受
  - 否则会因为 23:30 + 60 = 00:30 > 23:59 而被拒绝
```

### 8.6 日期覆盖的 ±1 天缓冲

在 `buildDateRanges()` 中处理日期覆盖时使用了缓冲：

```typescript
// packages/features/schedules/lib/date-ranges.ts:277-290
// TODO: 移除 .subtract(1, "day") 和 .add(1, "day") 部分
// 并重构以实际使用正确的日期。
// 截至 2024-02-20，本地和 UTC 日期在覆盖方面存在不匹配
// 以及 dateFrom 和 dateTo 字段，导致如果不返回 true，
// 就会出现"没有找到可用用户"的错误。
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

**问题背景**：
- 时区差异可能导致日期计算偏移一天
- 例如：主机在 UTC+12，预订者在 UTC-12
- 同一时刻在不同时区可能是不同的日期
- 缓冲处理确保不会因为时区边界问题漏掉日期覆盖

---

## 9. 预订限制系统详解

### 9.1 限制类型

系统支持两种类型的预订限制：

| 限制类型 | 配置字段 | 说明 |
|----------|----------|------|
| **预订数量限制** | `bookingLimits` | 每个时间段内最多预订数量 |
| **时长限制** | `durationLimits` | 每个时间段内最多预订时长（分钟） |

### 9.2 时间周期单位

```typescript
// packages/lib/intervalLimits/intervalLimit.ts
type IntervalLimit = {
  PER_YEAR?: number;
  PER_QUARTER?: number;
  PER_MONTH?: number;
  PER_WEEK?: number;
  PER_DAY?: number;
  PER_HOUR?: number;
};
```

### 9.3 限制处理流程

在 `getBusyTimesFromLimitsForUsers()` 中处理：

```typescript
// packages/trpc/server/routers/viewer/slots/util.ts:379-449
// 1. 遍历所有限制周期（优先级从大到小）
for (const key of descendingLimitKeys) {
  const limit = bookingLimits?.[key];
  if (!limit) continue;
  
  const unit = intervalLimitKeyToUnit(key);
  
  // 2. 获取该周期内的所有开始日期
  const periodStartDates = this.dependencies.userAvailabilityService.getPeriodStartDatesBetween(
    dateFrom,
    dateTo,
    unit,
    timeZone
  );
  
  // 3. 检查每个周期
  for (const periodStart of periodStartDates) {
    if (globalLimitManager.isAlreadyBusy(periodStart, unit, timeZone)) continue;
    
    const periodEnd = periodStart.endOf(unit);
    let totalBookings = 0;
    
    // 4. 统计该周期内的已有预订数
    for (const booking of busyTimesFromLimitsBookings) {
      if (!isBookingWithinPeriod(booking, periodStart, periodEnd, timeZone)) {
        continue;
      }
      totalBookings++;
      
      // 5. 如果达到限制，标记该周期为"忙碌"
      if (totalBookings >= limit) {
        globalLimitManager.addBusyTime({
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
}
```

### 9.4 时长限制的特殊计算

时长限制需要累计已预订时长：

```typescript
// packages/trpc/server/routers/viewer/slots/util.ts:554-637
if (durationLimits) {
  for (const key of descendingLimitKeys) {
    const limit = durationLimits?.[key];
    if (!limit) continue;
    
    const unit = intervalLimitKeyToUnit(key);
    const periodStartDates = this.dependencies.userAvailabilityService.getPeriodStartDatesBetween(
      dateFrom, dateTo, unit, timeZone
    );
    
    for (const periodStart of periodStartDates) {
      // ...
      const selectedDuration = (duration || eventType.length) ?? 0;
      
      // 首先检查：如果单次时长超过限制，直接标记为忙碌
      if (selectedDuration > limit) {
        limitManager.addBusyTime({...});
        continue;
      }
      
      // 累计该周期内的已预订时长
      let totalDuration = selectedDuration; // 加上当前请求的时长
      for (const booking of userBookings) {
        if (!isBookingWithinPeriod(booking, periodStart, periodEnd, timeZone)) {
          continue;
        }
        totalDuration += dayjs(booking.end).diff(dayjs(booking.start), "minute");
        
        if (totalDuration > limit) {
          limitManager.addBusyTime({...});
          break;
        }
      }
    }
  }
}
```

### 9.5 限制优先级

`descendingLimitKeys` 定义了限制检查的优先级：

```typescript
// packages/lib/intervalLimits/intervalLimit.ts
export const descendingLimitKeys: IntervalLimitKey[] = [
  "PER_HOUR",
  "PER_DAY",
  "PER_WEEK",
  "PER_MONTH",
  "PER_QUARTER",
  "PER_YEAR",
];
```

**逻辑**：
- 从最具体（小时）到最宽泛（年）
- 如果任何一个周期的限制已达到，标记为忙碌
- 使用 `globalLimitManager.isAlreadyBusy()` 避免重复检查

### 9.6 团队限制 vs 个人限制

系统支持两级限制：
1. **个人限制**：`eventType.bookingLimits` / `eventType.durationLimits`
2. **团队限制**：`team.bookingLimits`（如果是团队事件）

```typescript
// packages/features/availability/lib/getUserAvailability.ts:559-582
// 检查团队级别的限制
const teamForBookingLimits =
  initialData?.teamForBookingLimits ??
  eventType?.team ??
  (eventType?.parent?.team?.includeManagedEventsInLimits ? eventType?.parent?.team : null);

const teamBookingLimits = parseBookingLimit(teamForBookingLimits?.bookingLimits);

let busyTimesFromTeamLimits: EventBusyDetails[] = [];

if (initialData?.teamBookingLimits && teamForBookingLimits) {
  busyTimesFromTeamLimits = initialData.teamBookingLimits.get(user.id) || [];
} else if (teamForBookingLimits && teamBookingLimits) {
  // 回退到单独查询
  busyTimesFromTeamLimits = await getBusyTimesFromTeamLimits(
    user,
    teamBookingLimits,
    dateFrom.tz(finalTimezone),
    dateTo.tz(finalTimezone),
    teamForBookingLimits.id,
    teamForBookingLimits.includeManagedEventsInLimits,
    finalTimezone,
    initialData?.rescheduleUid ?? undefined
  );
}
```

---

## 10. 性能优化策略

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
- `dayjs.valueOf()`