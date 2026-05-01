# Cal.diy 可用时段计算链路分析报告（V2 - 事实修正版）

## 1. 概述

本文档基于**实际代码逻辑**修正了两个关键事实问题：
1. **跨天工作时间示例**：按 `getWorkingHours` 的分钟边界和溢出分支重算
2. **最小提前预约与过期时间过滤**：直接按 `isTimeOutOfBounds` 的实际判断条件和异常分支说明

最后提供**端到端顺序复盘**，标明各判断在访客可见时段生成链路中的位置。

---

## 2. 关键事实修正

### 2.1 修正一：getWorkingHours 的分钟边界与溢出分支

#### 2.1.1 核心常量与公式

**关键常量**（`packages/lib/availability.ts:54-56`）：
```typescript
export const MINUTES_IN_DAY = 60 * 24;        // = 1440
export const MINUTES_DAY_END = MINUTES_IN_DAY - 1;  // = 1439
export const MINUTES_DAY_START = 0;
```

**计算公式**（`packages/lib/availability.ts:79-84`）：
```typescript
const startTime =
  dayjs.utc(schedule.startTime).get("hour") * 60 +
  dayjs.utc(schedule.startTime).get("minute") -
  utcOffset;

const endTime =
  dayjs.utc(schedule.endTime).get("hour") * 60 +
  dayjs.utc(schedule.endTime).get("minute") - utcOffset;
```

**UTC 偏移获取**：
```typescript
const utcOffset =
  relativeTimeUnit.utcOffset ??
  (relativeTimeUnit.timeZone ? dayjs().tz(relativeTimeUnit.timeZone).utcOffset() : 0);
```

- 对于 **UTC+8**（如 Asia/Shanghai）：`utcOffset = 480` 分钟
- 对于 **UTC-5**（如 America/New_York）：`utcOffset = -300` 分钟
- 对于 **UTC**：`utcOffset = 0` 分钟

#### 2.1.2 三个分支的判断条件

**分支1：同一天处理**（`packages/lib/availability.ts:86-99`）
```typescript
const sameDayStartTime = Math.max(MINUTES_DAY_START, Math.min(MINUTES_DAY_END, startTime));
const sameDayEndTime = Math.max(MINUTES_DAY_START, Math.min(MINUTES_DAY_END, endTime));

if (sameDayEndTime < sameDayStartTime) {
  return currentWorkingHours;  // 同一天结束早于开始，直接返回（不添加）
}

if (sameDayStartTime !== sameDayEndTime) {
  // 添加同一天的工作时间
  const newWorkingHours: WorkingHours = {
    days: schedule.days,
    startTime: sameDayStartTime,
    endTime: sameDayEndTime,
  };
  currentWorkingHours.push(newWorkingHours);
}
```

**条件**：`sameDayEndTime >= sameDayStartTime` 且 `sameDayStartTime !== sameDayEndTime`

---

**分支2：溢出到前一天**（`packages/lib/availability.ts:100-110`）
```typescript
// check for overflow to the previous day
// overflowing days constraint to 0-6 day range (Sunday-Saturday)
if (startTime < MINUTES_DAY_START || endTime < MINUTES_DAY_START) {
  const newWorkingHours: WorkingHours = {
    days: schedule.days.map((day) => (day - 1 >= 0 ? day - 1 : 6)),
    startTime: startTime + MINUTES_IN_DAY,
    endTime: Math.min(endTime + MINUTES_IN_DAY, MINUTES_DAY_END),
  };
  if (schedule.userId) newWorkingHours.userId = schedule.userId;
  currentWorkingHours.push(newWorkingHours);
}
```

**条件**：`startTime < 0 || endTime < 0`

**处理**：
- 星期几减 1（前一天），周日（0）减 1 变为周六（6）
- 时间加 1440 分钟（一天）

---

**分支3：溢出到后一天**（`packages/lib/availability.ts:111-120`）
```typescript
// else, check for overflow in the next day
else if (startTime > MINUTES_DAY_END || endTime > MINUTES_IN_DAY) {
  const newWorkingHours: WorkingHours = {
    days: schedule.days.map((day) => (day + 1) % 7),
    startTime: Math.max(startTime - MINUTES_IN_DAY, MINUTES_DAY_START),
    endTime: endTime - MINUTES_IN_DAY,
  };
  if (schedule.userId) newWorkingHours.userId = schedule.userId;
  currentWorkingHours.push(newWorkingHours);
}
```

**条件**：`startTime > 1439 || endTime > 1440`

**处理**：
- 星期几加 1（后一天），周六（6）加 1 变为周日（0）
- 时间减 1440 分钟（一天）

#### 2.1.3 实际示例（按时区转换场景）

**关键理解**：`getWorkingHours` 的三个分支处理的是**时区转换后的溢出**，不是工作时间本身的"跨天"。

工作时间的"跨天"（如设置 22:00-02:00）需要特殊的存储格式，或者在 `processWorkingHours` 中处理。

---

**示例场景 A：UTC+8 时区，UTC 时间较早**

假设：
- 主机设置工作时间：**UTC 01:00-10:00**（对应 UTC+8 的 09:00-18:00）
- 目标时区：**UTC+8**（Asia/Shanghai）
- `utcOffset = 480` 分钟

**计算过程**：
```
startTime = 1*60 + 0 - 480 = 60 - 480 = -420 分钟
endTime = 10*60 + 0 - 480 = 600 - 480 = 120 分钟
```

**分支1 - 同一天检查**：
```
sameDayStartTime = max(0, min(1439, -420)) = 0
sameDayEndTime = max(0, min(1439, 120)) = 120

sameDayEndTime < sameDayStartTime  →  120 < 0  →  false
sameDayStartTime !== sameDayEndTime  →  0 !== 120  →  true
→ 添加同一天工作时间：days = schedule.days, startTime = 0, endTime = 120
→ 即 00:00-02:00（UTC+8 当地时间）
```

**分支2 - 溢出到前一天检查**：
```
startTime < 0 || endTime < 0  →  -420 < 0 || 120 < 0  →  true || false  →  true
→ 溢出到前一天
```

**溢出处理**：
```
days = schedule.days.map((day) => (day - 1 >= 0 ? day - 1 : 6))
     → 例如 [1,2,3,4,5] 变为 [0,1,2,3,4]（前一天）

startTime = startTime + 1440 = -420 + 1440 = 1020 分钟（17:00）
endTime = min(endTime + 1440, 1439) = min(120 + 1440, 1439) = min(1560, 1439) = 1439
→ 添加前一天工作时间：days = [0,1,2,3,4], startTime = 1020, endTime = 1439
→ 即前一天 17:00-23:59（UTC+8 当地时间）
```

**分支3 - 溢出到后一天检查**：
```
else if (startTime > 1439 || endTime > 1440)
→ 已经进入分支2，跳过分支3
```

**最终结果**：
1. **同一天**（schedule.days）：00:00-02:00
2. **前一天**（schedule.days - 1）：17:00-23:59

这对应 UTC+8 的 17:00-02:00（+1 天），即 UTC 01:00-10:00 的完整覆盖。

---

**示例场景 B：UTC-5 时区，UTC 时间较晚**

假设：
- 主机设置工作时间：**UTC 22:00-05:00（次日）**（对应 UTC-5 的 17:00-00:00）
- 目标时区：**UTC-5**（America/New_York）
- `utcOffset = -300` 分钟

**计算过程**：
```
startTime = 22*60 + 0 - (-300) = 1320 + 300 = 1620 分钟
endTime = 5*60 + 0 - (-300) = 300 + 300 = 600 分钟
```

**分支1 - 同一天检查**：
```
sameDayStartTime = max(0, min(1439, 1620)) = 1439
sameDayEndTime = max(0, min(1439, 600)) = 600

sameDayEndTime < sameDayStartTime  →  600 < 1439  →  true
→ 直接返回，不添加同一天工作时间
```

**分支2 - 溢出到前一天检查**：
```
startTime < 0 || endTime < 0  →  1620 < 0 || 600 < 0  →  false
→ 不进入分支2
```

**分支3 - 溢出到后一天检查**：
```
startTime > 1439 || endTime > 1440  →  1620 > 1439 || 600 > 1440  →  true || false  →  true
→ 溢出到后一天
```

**溢出处理**：
```
days = schedule.days.map((day) => (day + 1) % 7)
     → 例如 [1,2,3,4,5] 变为 [2,3,4,5,6]（后一天）

startTime = max(startTime - 1440, 0) = max(1620 - 1440, 0) = max(180, 0) = 180 分钟（03:00）
endTime = endTime - 1440 = 600 - 1440 = -840 分钟
→ 添加后一天工作时间：days = [2,3,4,5,6], startTime = 180, endTime = -840
```

**注意**：这里 `endTime = -840` 可能有问题，实际中应该需要特殊处理。

---

**示例场景 C：纯 UTC 时区，工作时间 22:00-02:00**

假设：
- 主机设置工作时间：**22:00-02:00**（跨越午夜）
- 存储格式：
  - `startTime` = 22:00 UTC（小时=22，分钟=0）
  - `endTime` = 02:00 UTC（小时=2，分钟=0）
- 目标时区：**UTC**
- `utcOffset = 0` 分钟

**计算过程**：
```
startTime = 22*60 + 0 - 0 = 1320 分钟（22:00）
endTime = 2*60 + 0 - 0 = 120 分钟（02:00）
```

**分支1 - 同一天检查**：
```
sameDayStartTime = max(0, min(1439, 1320)) = 1320
sameDayEndTime = max(0, min(1439, 120)) = 120

sameDayEndTime < sameDayStartTime  →  120 < 1320  →  true
→ 直接返回，不添加同一天工作时间
```

**分支2 - 溢出到前一天检查**：
```
startTime < 0 || endTime < 0  →  1320 < 0 || 120 < 0  →  false
→ 不进入分支2
```

**分支3 - 溢出到后一天检查**：
```
startTime > 1439 || endTime > 1440  →  1320 > 1439 || 120 > 1440  →  false
→ 不进入分支3
```

**最终结果**：`getWorkingHours` **不返回任何工作时间**！

**关键结论**：如果工作时间设置为 22:00-02:00（跨越午夜），且 `endTime` 存储为 02:00（不是 26:00），`getWorkingHours` 不会返回任何工作时间。

这说明：
1. 跨天工作时间可能需要特殊的存储格式（如 `endTime` 存储为 26:00）
2. 或者跨天工作时间的处理在其他地方（如 `processWorkingHours`）
3. 或者系统**根本不支持**跨天工作时间的直接设置

---

### 2.2 修正二：isTimeOutOfBounds 的实际判断条件与异常分支

#### 2.2.1 核心函数实现

**函数定义**（`packages/lib/isOutOfBounds.tsx:243-262`）：
```typescript
/**
 * To be used when we work on Timeslots(and not Dates) to check boundaries
 * It ensures that the time isn't in the past and also checks if the time is within the minimum booking notice.
 * Note: It throws error that needs to be caught by caller.
 */
export function isTimeOutOfBounds({
  time,
  minimumBookingNotice,
}: {
  time: dayjs.ConfigType;
  minimumBookingNotice?: number;
}) {
  const date = dayjs(time);

  // 第一步：检查是否在过去
  guardAgainstBookingInThePast(date.toDate());

  // 第二步：检查最小预订通知
  if (minimumBookingNotice) {
    const minimumBookingStartDate = dayjs().add(minimumBookingNotice, "minutes");
    if (date.isBefore(minimumBookingStartDate)) {
      return true;
    }
  }

  return false;
}
```

#### 2.2.2 过去时间检查

**函数定义**（`packages/lib/isOutOfBounds.tsx:15-21`）：
```typescript
function guardAgainstBookingInThePast(date: Date) {
  if (date >= new Date()) {
    // Date is in the future.
    return;
  }
  throw new BookingDateInPastError();
}
```

**判断条件**：
- 时间在过去：`date < new Date()` → **抛出 `BookingDateInPastError` 异常**
- 时间在未来或等于当前时间：`date >= new Date()` → **正常返回**

**异常类定义**（`packages/lib/isOutOfBounds.tsx:9-13`）：
```typescript
export class BookingDateInPastError extends Error {
  constructor(message = "Attempting to book a meeting in the past.") {
    super(message);
  }
}
```

#### 2.2.3 最小预订通知检查

**判断条件**：
```typescript
if (minimumBookingNotice) {
  const minimumBookingStartDate = dayjs().add(minimumBookingNotice, "minutes");
  if (date.isBefore(minimumBookingStartDate)) {
    return true;
  }
}
return false;
```

**逻辑**：
1. 如果 `minimumBookingNotice` 为 0 或 undefined，跳过检查，返回 `false`
2. 计算 `minimumBookingStartDate = 当前时间 + minimumBookingNotice 分钟`
3. 如果目标时间 **早于** `minimumBookingStartDate`，返回 `true`（越界）
4. 否则返回 `false`（正常）

#### 2.2.4 异常分支处理

**在时间槽服务中的处理**（`packages/trpc/server/routers/viewer/slots/util.ts:1343-1355`）：
```typescript
let isOutOfBounds = false;
try {
  isOutOfBounds = isTimeOutOfBounds({
    time: slot.time,
    minimumBookingNotice: eventType.minimumBookingNotice,
  });
} catch (error) {
  if (error instanceof BookingDateInPastError) {
    // 明确的过去日期错误 → 抛出 TRPCError
    throw new TRPCError({
      code: "BAD_REQUEST",
      message: error.message,
    });
  }
  throw error;  // 其他异常继续向上抛出
}
```

**处理流程**：
```
调用 isTimeOutOfBounds(slot.time, minimumBookingNotice)
        ↓
    [try 块]
        ↓
    guardAgainstBookingInThePast(date)
        ↓
    date >= new Date()?
   /              \
  否               是
  ↓                ↓
抛出              继续检查
BookingDateInPastError  minimumBookingNotice
        ↓                      ↓
   [catch 块]          date.isBefore(now + notice)?
        ↓                    /              \
   error instanceof        否               是
   BookingDateInPastError?  ↓                ↓
        ↓                返回 false       返回 true
   true          false     (正常)         (越界)
   ↓             ↓
抛出           抛出
TRPCError      error
(BAD_REQUEST)
```

#### 2.2.5 实际示例

**示例 A：时间在过去**

假设：
- 当前时间：2026-05-05 10:00:00
- 槽位时间：2026-05-05 09:00:00
- 最小预订通知：30 分钟

**执行流程**：
```typescript
isTimeOutOfBounds({
  time: "2026-05-05T09:00:00Z",
  minimumBookingNotice: 30,
})

// 第一步：检查过去
guardAgainstBookingInThePast(new Date("2026-05-05T09:00:00Z"))
// 09:00 < 10:00 → 抛出 BookingDateInPastError

// 在调用方捕获
try {
  isTimeOutOfBounds(...)
} catch (error) {
  if (error instanceof BookingDateInPastError) {
    throw new TRPCError({ code: "BAD_REQUEST", message: "Attempting to book a meeting in the past." })
  }
}
```

**结果**：抛出 `TRPCError`，HTTP 400 错误。

---

**示例 B：时间在未来，但小于最小提前通知**

假设：
- 当前时间：2026-05-05 10:00:00
- 槽位时间：2026-05-05 10:15:00（15 分钟后）
- 最小预订通知：30 分钟

**执行流程**：
```typescript
isTimeOutOfBounds({
  time: "2026-05-05T10:15:00Z",
  minimumBookingNotice: 30,
})

// 第一步：检查过去
guardAgainstBookingInThePast(new Date("2026-05-05T10:15:00Z"))
// 10:15 >= 10:00 → 正常返回

// 第二步：检查最小预订通知
const minimumBookingStartDate = dayjs().add(30, "minutes")  // 10:30
const date.isBefore(minimumBookingStartDate)  // 10:15 < 10:30 → true

return true  // 越界
```

**结果**：返回 `true`，槽位被过滤。

---

**示例 C：时间在未来，且大于最小提前通知**

假设：
- 当前时间：2026-05-05 10:00:00
- 槽位时间：2026-05-05 10:45:00（45 分钟后）
- 最小预订通知：30 分钟

**执行流程**：
```typescript
isTimeOutOfBounds({
  time: "2026-05-05T10:45:00Z",
  minimumBookingNotice: 30,
})

// 第一步：检查过去
// 10:45 >= 10:00 → 正常返回

// 第二步：检查最小预订通知
const minimumBookingStartDate = 10:30
const date.isBefore(minimumBookingStartDate)  // 10:45 < 10:30 → false

return false  // 正常
```

**结果**：返回 `false`，槽位保留。

---

**示例 D：最小预订通知为 0**

假设：
- 当前时间：2026-05-05 10:00:00
- 槽位时间：2026-05-05 10:00:00（当前时间）
- 最小预订通知：0 分钟

**执行流程**：
```typescript
isTimeOutOfBounds({
  time: "2026-05-05T10:00:00Z",
  minimumBookingNotice: 0,
})

// 第一步：检查过去
// 10:00 >= 10:00 → 正常返回（注意：是 >= 不是 >）

// 第二步：检查最小预订通知
if (minimumBookingNotice)  // 0 是 falsy，跳过

return false  // 正常
```

**结果**：返回 `false`，槽位保留。

**关键边界**：`date >= new Date()` 是**大于等于**，所以当前时间是允许的。

---

## 3. 端到端顺序复盘

### 3.1 访客可见时段生成链路全景

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                    访客可见时段生成链路（端到端顺序）                                      │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  输入：                                                                                   │
│  ├── eventTypeId: 事件类型 ID                                                            │
│  ├── startTime/endTime: 访客请求的时间范围                                               │
│  ├── timeZone: 访客时区                                                                  │
│  ├── duration: 预约时长（可选）                                                          │
│  └── rescheduleUid/bookingUid: 重新安排相关（可选）                                      │
│                                                                                          │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段1：获取事件类型与合格主机                                                       │  │
│  │                                                                                  │  │
│  │  1.1 查询 EventType                                                                 │  │
│  │      └── 获取 slotInterval（槽位间隔）、minimumBookingNotice（最小提前通知）等    │  │
│  │                                                                                  │  │
│  │  1.2 获取合格主机                                                                   │  │
│  │      ├── 过滤被阻止的主机（Watchlist）                                             │  │
│  │      ├── 动态事件类型：从配置的主机列表中筛选                                       │  │
│  │      └── 考虑调度算法（轮询、集体、管理等）                                         │  │
│  │                                                                                  │  │
│  │  输出：usersWithCredentials（合格主机列表）                                        │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段2：计算主机可用性（每个主机）                                                   │  │
│  │                                                                                  │  │
│  │  调用 getUserAvailability(user, ...)                                              │  │
│  │                                                                                  │  │
│  │  2.1 获取用户的工作时间设置                                                       │  │
│  │      └── availability = user.availability || defaultSchedule                     │  │
│  │                                                                                  │  │
│  │  2.2 构建基础日期范围（buildDateRanges）                                          │  │
│  │      ├── 处理常规工作时间：按 days 展开到具体日期                                  │  │
│  │      ├── 处理日期覆盖：有 date 字段的覆盖同一天的工作时间                          │  │
│  │      └── 处理请假：生成 0-length 范围，过滤时被移除                               │  │
│  │                                                                                  │  │
│  │  2.3 获取忙碌时间                                                                │  │
│  │      ├── 日历忙碌时间：从连接的日历获取（getBusyCalendarTimes）                  │  │
│  │      ├── 已接受预订：从数据库查询状态为 ACCEPTED 的预订                          │  │
│  │      ├── 缓冲时间：应用到日历事件和预订前后                                       │  │
│  │      ├── 预订限制：每天/每周/每月/每年的数量限制                                  │  │
│  │      ├── 时长限制：每天/每周/每月/每年的总时长限制                                │  │
│  │      └── 团队预订限制：团队级别的限制                                             │  │
│  │                                                                                  │  │
│  │  2.4 统一裁剪（subtract）                                                         │  │
│  │      └── dateRangesInWhichUserIsAvailable = subtract(dateRanges, busyTimes)    │  │
│  │                                                                                  │  │
│  │  输出：                                                                           │  │
│  │  ├── dateRanges: 最终可用时间范围                                                │  │
│  │  └── workingHours: 工作时间规则                                                  │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段3：多主机聚合                                                                 │  │
│  │                                                                                  │  │
│  │  根据事件类型的分配方式聚合：                                                      │  │
│  │                                                                                  │  │
│  │  3.1 集体事件（COLLECTIVE）                                                      │  │
│  │      └── aggregatedDateRanges = intersect(allUsersDateRanges)                  │  │
│  │      所有主机必须同时可用                                                         │  │
│  │                                                                                  │  │
│  │  3.2 轮询事件（ROUND_ROBIN）                                                     │  │
│  │      └── aggregatedDateRanges = union(allUsersDateRanges)                      │  │
│  │      任一主机可用即可                                                             │  │
│  │                                                                                  │  │
│  │  3.3 管理事件（MANAGED）                                                         │  │
│  │      └── 类似轮询，但有额外逻辑                                                   │  │
│  │                                                                                  │  │
│  │  输出：aggregatedDateRanges（聚合后的可用范围）                                  │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段4：时间槽生成（getSlots）                                                      │  │
│  │                                                                                  │  │
│  │  4.1 参数准备                                                                     │  │
│  │      ├── dateRanges: aggregatedDateRanges                                        │  │
│  │      ├── frequency: slotInterval 或 duration                                      │  │
│  │      ├── eventLength: duration 或事件时长                                         │  │
│  │      ├── minimumBookingNotice: 最小提前通知                                       │  │
│  │      └── showOptimizedSlots: 是否优化显示                                         │  │
│  │                                                                                  │  │
│  │  4.2 计算有效开始时间                                                             │  │
│  │      └── slotStartTime = max(range.start, now + minimumBookingNotice)          │  │
│  │      【第一次最小提前通知检查】                                                    │  │
│  │                                                                                  │  │
│  │  4.3 时区转换与分钟对齐                                                           │  │
│  │      ├── slotStartTime = slotStartTime.tz(timeZone)                             │  │
│  │      └── 如果 slotStartTime.minute() % interval !== 0，进行校正                 │  │
│  │                                                                                  │  │
│  │  4.4 槽位起始校正                                                                 │  │
│  │      ├── 优化模式：尽量从整点/15分/5分开始                                       │  │
│  │      └── 普通模式：从小时开始，向上取整到间隔                                     │  │
│  │                                                                                  │  │
│  │  4.5 循环生成槽位                                                                 │  │
│  │      while (slotStartTime + eventLength <= range.end) {                         │  │
│  │          检查是否是请假日期 → 标记 away = true                                   │  │
│  │          添加到 slots Map                                                         │  │
│  │          slotStartTime += frequency                                               │  │
│  │      }                                                                           │  │
│  │                                                                                  │  │
│  │  输出：timeSlots（按日期分组的时间槽数组）                                        │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段5：限制计划过滤（可选）                                                         │  │
│  │                                                                                  │  │
│  │  仅当 eventType.restrictionScheduleId 存在时执行：                                │  │
│  │                                                                                  │  │
│  │  5.1 获取限制计划的可用范围                                                       │  │
│  │      const { dateRanges: restrictionRanges } = buildDateRanges({               │  │
│  │          availability: restrictionSchedule.availability,                        │  │
│  │          ...                                                                     │  │
│  │      });                                                                         │  │
│  │                                                                                  │  │
│  │  5.2 过滤时间槽                                                                   │  │
│  │      availableTimeSlots = timeSlots.filter((slot) => {                          │  │
│  │          检查 slot.time 是否在 restrictionRanges 内                              │  │
│  │      });                                                                         │  │
│  │                                                                                  │  │
│  │  输出：过滤后的 availableTimeSlots                                                │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段6：预留槽位冲突检查                                                           │  │
│  │                                                                                  │  │
│  │  6.1 获取预留槽位                                                                 │  │
│  │      const reservedSlots = await this._getReservedSlotsAndCleanupExpired({     │  │
│  │          bookerClientUid,       // 当前访客的 UID（排除自己的预留）               │  │
│  │          eventTypeId,                                                             │  │
│  │          usersWithCredentials,                                                    │  │
│  │      });                                                                         │  │
│  │                                                                                  │  │
│  │  6.2 区分座位事件和普通事件                                                       │  │
│  │      ├── 座位事件：只处理 isSeat = true 的预留，更新座位计数                      │  │
│  │      └── 普通事件：所有预留都视为冲突                                             │  │
│  │                                                                                  │  │
│  │  6.3 冲突检查（checkForConflicts）                                               │  │
│  │      const busySlotsFromReservedSlots = reservedSlots.reduce(                 │  │
│  │          (r, c) => {                                                             │  │
│  │              if (!c.isSeat) {  // 非座位预留，视为忙碌                          │  │
│  │                  r.push({                                                        │  │
│  │                      start: c.slotUtcStartDate,                                  │  │
│  │                      end: c.slotUtcEndDate,                                      │  │
│  │                  });                                                              │  │
│  │              }                                                                    │  │
│  │              return r;                                                            │  │
│  │          },                                                                       │  │
│  │          []                                                                       │  │
│  │      );                                                                          │  │
│  │                                                                                  │  │
│  │  6.4 过滤冲突槽位                                                                 │  │
│  │      availableTimeSlots = availableTimeSlots                                     │  │
│  │          .map((slot) => {                                                        │  │
│  │              if (!checkForConflicts({...})) {                                   │  │
│  │                  return slot;  // 无冲突，保留                                   │  │
│  │              }                                                                    │  │
│  │              return undefined;  // 有冲突，过滤                                   │  │
│  │          })                                                                      │  │
│  │          .filter((item) => !!item);                                             │  │
│  │                                                                                  │  │
│  │  输出：过滤冲突后的 availableTimeSlots                                            │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段7：期限检查（核心：过去时间 + 最小提前通知 + 未来限制）                       │  │
│  │                                                                                  │  │
│  │  7.1 计算期限限制（calculatePeriodLimits）                                        │  │
│  │      ├── ROLLING：当前时间 + periodDays 天                                       │  │
│  │      ├── ROLLING_WINDOW：找到 N 个可预订日期                                     │  │
│  │      └── RANGE：指定的日期范围                                                    │  │
│  │                                                                                  │  │
│  │  7.2 逐个槽位检查                                                                 │  │
│  │      for (const [date, slots] of Object.entries(slotsMappedToDate)) {          │  │
│  │          const filteredSlots = slots.filter((slot) => {                         │  │
│  │                                                                                  │  │
│  │              // 7.2.1 未来限制检查                                                │  │
│  │              const isFutureLimitViolation = isTimeViolatingFutureLimit({       │  │
│  │                  time: slot.time,                                                 │  │
│  │                  periodLimits,                                                     │  │
│  │              });                                                                  │  │
│  │              // 检查是否超过滚动窗口或日期范围                                     │  │
│  │                                                                                  │  │
│  │              // 7.2.2 过去时间 + 最小提前通知检查【第二次】                      │  │
│  │              let isOutOfBounds = false;                                          │  │
│  │              try {                                                               │  │
│  │                  isOutOfBounds = isTimeOutOfBounds({                            │  │
│  │                      time: slot.time,                                             │  │
│  │                      minimumBookingNotice: eventType.minimumBookingNotice,       │  │
│  │                  });                                                              │  │
│  │              } catch (error) {                                                   │  │
│  │                  if (error instanceof BookingDateInPastError) {                 │  │
│  │                      // 明确的过去日期错误 → 抛出 TRPCError                      │  │
│  │                      throw new TRPCError({                                       │  │
│  │                          code: "BAD_REQUEST",                                     │  │
│  │                          message: error.message,                                  │  │
│  │                      });                                                          │  │
│  │                  }                                                                │  │
│  │                  throw error;                                                     │  │
│  │              }                                                                    │  │
│  │                                                                                  │  │
│  │              // 7.2.3 滚动窗口特殊处理                                            │  │
│  │              if (isFutureLimitViolation && doesStartFromToday) {                │  │
│  │                  foundAFutureLimitViolation = true;                              │  │
│  │                  // 找到第一个违规后，后续所有日期都跳过                          │  │
│  │              }                                                                    │  │
│  │                                                                                  │  │
│  │              return (                                                             │  │
│  │                  !isFutureLimitViolation &&                                      │  │
│  │                  !isOutOfBounds                                                   │  │
│  │              );                                                                   │  │
│  │          });                                                                      │  │
│  │      }                                                                           │  │
│  │                                                                                  │  │
│  │  【关键：isTimeOutOfBounds 的实际逻辑】                                           │  │
│  │  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  │  isTimeOutOfBounds({ time, minimumBookingNotice })                        │  │
│  │  │                                                                           │  │
│  │  │  第一步：guardAgainstBookingInThePast(date)                               │  │
│  │  │  ├── if (date >= new Date()) → return（正常）                             │  │
│  │  │  └── else → throw BookingDateInPastError                                 │  │
│  │  │                                                                           │  │
│  │  │  第二步：最小提前通知检查                                                   │  │
│  │  │  ├── if (!minimumBookingNotice) → return false（跳过）                   │  │
│  │  │  ├── minimumBookingStartDate = now + minimumBookingNotice                │  │
│  │  │  ├── if (date.isBefore(minimumBookingStartDate)) → return true（越界）  │  │
│  │  │  └── else → return false（正常）                                          │  │
│  │  └──────────────────────────────────────────────────────────────────────────┘  │
│  │                                                                                  │  │
│  │  输出：withinBoundsSlotsMappedToDate（过滤后的时间槽）                          │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段8：请求范围过滤                                                               │  │
│  │                                                                                  │  │
│  │  只保留访客请求的时间范围内的槽位：                                              │  │
│  │                                                                                  │  │
│  │  const filteredSlotsMappedToDate = this.filterSlotsByRequestedDateRange({     │  │
│  │      slotsMappedToDate: withinBoundsSlotsMappedToDate,                          │  │
│  │      startTime: input.startTime,                                                 │  │
│  │      endTime: input.endTime,                                                     │  │
│  │      timeZone: input.timeZone,                                                   │  │
│  │  });                                                                             │  │
│  │                                                                                  │  │
│  │  原因：buildDateRanges 在处理时区边界时使用了 ±1 天的缓冲区，                   │  │
│  │        可能导致相邻日期的槽位泄漏到响应中                                        │  │
│  │                                                                                  │  │
│  │  输出：filteredSlotsMappedToDate（最终可见的时间槽）                            │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                    ↓                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段9：格式化输出                                                                 │  │
│  │                                                                                  │  │
│  │  9.1 转换为 ISO 格式                                                             │  │
│  │  9.2 按日期分组                                                                   │  │
│  │  9.3 添加座位信息（如果是座位事件）                                               │  │
│  │                                                                                  │  │
│  │  最终返回：                                                                       │  │
│  │  {                                                                              │  │
│  │    slots: {                                                                     │  │
│  │      "2026-05-05": [                                                           │  │
│  │          { time: "2026-05-05T10:00:00Z", ... },                             │  │
│  │          { time: "2026-05-05T11:00:00Z", ... },                             │  │
│  │          ...                                                                    │  │
│  │      ],                                                                         │  │
│  │      "2026-05-06": [...],                                                      │  │
│  │    }                                                                            │  │
│  │  }                                                                              │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 关键判断点的位置汇总

| 判断类型 | 阶段 | 函数 | 位置 | 行为 |
|---------|------|------|------|------|
| 工作时间裁剪 | 阶段2.2 | `buildDateRanges` | 主机侧 | 通过对象覆盖和 0-length 过滤处理日期覆盖和请假 |
| 忙碌时间裁剪 | 阶段2.4 | `subtract` | 主机侧 | 从可用范围中减去所有忙碌时间 |
| 最小提前通知（第一次） | 阶段4.2 | `getSlots` | 时间槽生成 | 计算有效开始时间时考虑 |
| 预留槽位冲突 | 阶段6.3 | `checkForConflicts` | 时间槽过滤 | 过滤其他访客临时锁定的槽位 |
| 未来限制 | 阶段7.2.1 | `isTimeViolatingFutureLimit` | 期限检查 | 检查是否超过滚动窗口或日期范围 |
| **过去时间** | 阶段7.2.2 | `guardAgainstBookingInThePast` | 期限检查 | **抛出异常** `BookingDateInPastError` |
| **最小提前通知（第二次）** | 阶段7.2.2 | `isTimeOutOfBounds` | 期限检查 | **返回 true/false** 过滤 |
| 请求范围过滤 | 阶段8 | `filterSlotsByRequestedDateRange` | 最终过滤 | 只保留请求范围内的槽位 |

### 3.3 isTimeOutOfBounds 在链路中的位置详解

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                     isTimeOutOfBounds 的调用位置与上下文                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  阶段7：期限检查                                                                     │
│                                                                                      │
│  代码位置：packages/trpc/server/routers/viewer/slots/util.ts:1343-1355           │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  // 遍历每个日期的槽位                                                         │  │
│  │  for (const [date, slots] of Object.entries(slotsMappedToDate)) {          │  │
│  │      const filteredSlots = slots.filter((slot) => {                          │  │
│  │                                                                               │  │
│  │          // 【第一步】检查未来限制（滚动窗口/日期范围）                        │  │
│  │          const isFutureLimitViolation = isTimeViolatingFutureLimit({        │  │
│  │              time: slot.time,                                                  │  │
│  │              periodLimits,                                                      │  │
│  │          });                                                                   │  │
│  │                                                                               │  │
│  │          // 【第二步】检查过去时间和最小提前通知                               │  │
│  │          let isOutOfBounds = false;                                           │  │
│  │          try {                                                                │  │
│  │              isOutOfBounds = isTimeOutOfBounds({                             │  │
│  │                  time: slot.time,                                              │  │
│  │                  minimumBookingNotice: eventType.minimumBookingNotice,        │  │
│  │              });                                                               │  │
│  │          } catch (error) {                                                    │  │
│  │              // 捕获异常                                                       │  │
│  │              if (error instanceof BookingDateInPastError) {                  │  │
│  │                  // 明确的过去日期错误 → 抛出 TRPCError                       │  │
│  │                  throw new TRPCError({                                        │  │
│  │                      code: "BAD_REQUEST",                                      │  │
│  │                      message: error.message,                                   │  │
│  │                  });                                                           │  │
│  │              }                                                                 │  │
│  │              throw error;  // 其他异常继续抛出                                 │  │
│  │          }                                                                     │  │
│  │                                                                               │  │
│  │          // 【滚动窗口特殊处理】                                               │  │
│  │          if (isFutureLimitViolation && doesStartFromToday) {                 │  │
│  │              foundAFutureLimitViolation = true;                               │  │
│  │              // 找到第一个违规后，后续所有日期都跳过                           │  │
│  │          }                                                                     │  │
│  │                                                                               │  │
│  │          return (                                                              │  │
│  │              !isFutureLimitViolation &&                                       │  │
│  │              !isOutOfBounds                                                    │  │
│  │          );                                                                    │  │
│  │      });                                                                       │  │
│  │  }                                                                            │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  【关键理解】                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  isTimeOutOfBounds 内部逻辑（packages/lib/isOutOfBounds.tsx:243-262）      │  │
│  │                                                                              │  │
│  │  export function isTimeOutOfBounds({ time, minimumBookingNotice }) {        │  │
│  │      const date = dayjs(time);                                               │  │
│  │                                                                              │  │
│  │      // 【子步骤1】检查是否在过去                                             │  │
│  │      guardAgainstBookingInThePast(date.toDate());                           │  │
│  │      // → 如果在过去，抛出 BookingDateInPastError                            │  │
│  │      // → 如果在未来或等于当前时间，正常返回                                  │  │
│  │                                                                              │  │
│  │      // 【子步骤2】检查最小提前通知                                           │  │
│  │      if (minimumBookingNotice) {                                             │  │
│  │          const minimumBookingStartDate = dayjs().add(minimumBookingNotice, "minutes"); │  │
│  │          if (date.isBefore(minimumBookingStartDate)) {                      │  │
│  │              return true;  // 越界                                           │  │
│  │          }                                                                    │  │
│  │      }                                                                        │  │
│  │                                                                              │  │
│  │      return false;  // 正常                                                   │  │
│  │  }                                                                            │  │
│  │                                                                              │  │
│  │  【子步骤1详解】guardAgainstBookingInThePast                                  │  │
│  │  function guardAgainstBookingInThePast(date: Date) {                         │  │
│  │      if (date >= new Date()) {   // 注意：是 >= 不是 >                       │  │
│  │          return;  // 在未来或等于当前时间，正常                                │  │
│  │      }                                                                        │  │
│  │      throw new BookingDateInPastError();  // 在过去，抛出异常                 │  │
│  │  }                                                                            │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  【行为差异】                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────────────┐  │
│  │  条件                         │ 行为                                          │  │
│  ├─────────────────────────────────────────────────────────────────────────────┤  │
│  │  date < now                   │ 抛出 BookingDateInPastError                  │  │
│  │                               │ → 调用方捕获后抛出 TRPCError (BAD_REQUEST)    │  │
│  ├─────────────────────────────────────────────────────────────────────────────┤  │
│  │  now ≤ date < now + notice    │ 过去时间检查通过，但最小提前通知不通过        │  │
│  │                               │ → 返回 true（isOutOfBounds = true）          │  │
│  │                               │ → 槽位被过滤（不显示）                        │  │
│  ├─────────────────────────────────────────────────────────────────────────────┤  │
│  │  date ≥ now + notice          │ 所有检查通过                                  │  │
│  │                               │ → 返回 false（isOutOfBounds = false）         │  │
│  │                               │ → 槽位保留（显示）                             │  │
│  └─────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 修正后的边界示例

### 4.1 示例：时区转换导致的溢出（UTC+8）

**场景**：
- 主机设置工作时间：**UTC 01:00-10:00**（对应 UTC+8 的 09:00-18:00）
- 目标时区：**UTC+8**（Asia/Shanghai）
- `utcOffset = 480` 分钟

**计算过程**：
```
startTime = 1*60 + 0 - 480 = -420 分钟
endTime = 10*60 + 0 - 480 = 120 分钟
```

**分支1 - 同一天**：
```
sameDayStartTime = max(0, min(1439, -420)) = 0
sameDayEndTime = max(0, min(1439, 120)) = 120
sameDayEndTime >= sameDayStartTime  →  120 >= 0  →  true
→ 添加同一天工作时间：00:00-02:00（UTC+8）
```

**分支2 - 溢出到前一天**：
```
startTime < 0  →  -420 < 0  →  true
→ 溢出到前一天
days = schedule.days.map((day) => (day - 1 >= 0 ? day - 1 : 6))
startTime = -420 + 1440 = 1020 分钟（17:00）
endTime = min(120 + 1440, 1439) = 1439
→ 添加前一天工作时间：17:00-23:59（UTC+8）
```

**最终结果**：
- **同一天**：00:00-02:00
- **前一天**：17:00-23:59
- 合并后：17:00-02:00（+1 天）

---

### 4.2 示例：过去时间导致的异常

**场景**：
- 当前时间：2026-05-05 10:00:00
- 槽位时间：2026-05-05 09:00:00
- 最小提前通知：30 分钟

**执行流程**：
```
阶段7.2.2：调用 isTimeOutOfBounds(slot.time, 30)

1. guardAgainstBookingInThePast(new Date("2026-05-05T09:00:00Z"))
   → 09:00 < 10:00 → 抛出 BookingDateInPastError

2. 调用方捕获异常
   if (error instanceof BookingDateInPastError) {
       throw new TRPCError({
           code: "BAD_REQUEST",
           message: "Attempting to book a meeting in the past."
       })
   }
```

**结果**：API 返回 HTTP 400 错误，消息为"Attempting to book a meeting in the past."

---

### 4.3 示例：最小提前通知导致的过滤

**场景**：
- 当前时间：2026-05-05 10:00:00
- 槽位时间：2026-05-05 10:15:00
- 最小提前通知：30 分钟

**执行流程**：
```
阶段7.2.2：调用 isTimeOutOfBounds(slot.time, 30)

1. guardAgainstBookingInThePast(new Date("2026-05-05T10:15:00Z"))
   → 10:15 >= 10:00 → 正常返回

2. 检查最小提前通知
   minimumBookingStartDate = 10:00 + 30 分钟 = 10:30
   slot.time (10:15) < minimumBookingStartDate (10:30) → true
   → 返回 true
```

**结果**：`isOutOfBounds = true`，槽位被过滤（不显示给访客）

---

### 4.4 示例：所有检查通过

**场景**：
- 当前时间：2026-05-05 10:00:00
- 槽位时间：2026-05-05 10:45:00
- 最小提前通知：30 分钟

**执行流程**：
```
阶段7.2.2：调用 isTimeOutOfBounds(slot.time, 30)

1. guardAgainstBookingInThePast(new Date("2026-05-05T10:45:00Z"))
   → 10:45 >= 10:00 → 正常返回

2. 检查最小提前通知
   minimumBookingStartDate = 10:30
   slot.time (10:45) < minimumBookingStartDate (10:30) → false
   → 返回 false
```

**结果**：`isOutOfBounds = false`，槽位保留（显示给访客）

---

## 5. 关键代码位置索引

| 功能 | 文件路径 | 行号 |
|-----|---------|------|
| getWorkingHours 核心逻辑 | `packages/lib/availability.ts` | 61-127 |
| 同一天分支 | `packages/lib/availability.ts` | 86-99 |
| 溢出到前一天分支 | `packages/lib/availability.ts` | 100-110 |
| 溢出到后一天分支 | `packages/lib/availability.ts` | 111-120 |
| isTimeOutOfBounds 函数 | `packages/lib/isOutOfBounds.tsx` | 243-262 |
| guardAgainstBookingInThePast | `packages/lib/isOutOfBounds.tsx` | 15-21 |
| BookingDateInPastError | `packages/lib/isOutOfBounds.tsx` | 9-13 |
| 期限检查调用 | `packages/trpc/server/routers/viewer/slots/util.ts` | 1343-1355 |
| 时间槽生成 | `packages/features/schedules/lib/slots.ts` | 232+ |
| subtract 统一裁剪 | `packages/features/schedules/lib/date-ranges.ts` | 423-452 |
| buildDateRanges | `packages/features/schedules/lib/date-ranges.ts` | 226+ |

---

*报告生成日期: 2026-05-02*

*基于实际代码逻辑修正：*
- *getWorkingHours 的三个分支（同一天、溢出到前一天、溢出到后一天）*
- *isTimeOutOfBounds 的实际判断条件（先检查过去时间抛出异常，再检查最小提前通知返回布尔值）*
