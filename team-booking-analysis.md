# 团队预约链路深度分析报告

## 1. 系统架构概览

团队预约（Team Booking）是 Cal.diy 中最复杂的业务场景之一，涉及多成员协同、可用性合并、冲突检测和智能分配等多个核心环节。

### 1.1 团队预约类型

| 调度类型 | SchedulingType | 行为描述 |
|---------|----------------|---------|
| **集体预约** | `COLLECTIVE` | 所有主持人必须同时可用，预订时所有主持人都被分配 |
| **轮询分配** | `ROUND_ROBIN` | 任一主持人可用即可，预订时选择一个"幸运主持人" |
| **固定 + 轮询混合** | `ROUND_ROBIN` + `isFixed` | 固定主持人必须可用，轮询主持人至少一个可用 |

### 1.2 核心数据流

```
┌─────────────────────────────────────────────────────────────────────┐
│                    团队预约请求 (Team Booking Request)                │
│  GET /slots 或 POST /book 触发                                        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 1: 主持人筛选与屏蔽过滤                                         │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ 1. QualifiedHostsService: 识别固定主持人 vs 轮询主持人            │ │
│  │ 2. filterBlockedHosts: 过滤被锁定/被 watchlist 屏蔽的主持人       │ │
│  │ 3. contactOwnerEmail 路由: 优先分配给联系人所有者                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 2: 多成员可用性计算                                             │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ 1. getUserAvailability: 为每个主持人计算独立可用性                │ │
│  │    - 工作时间 (Working Hours)                                     │ │
│  │    - 日历忙时 (Calendar Busy Times)                               │ │
│  │    - 预订限制 (Booking/Duration Limits)                           │ │
│  │    - 休假 (Out of Office)                                         │ │
│  │                                                                   │ │
│  │ 2. getAggregatedAvailability: 合并多成员可用性                    │ │
│  │    - 固定主持人: 交集 (AND) - 全部可用                           │ │
│  │    - 轮询主持人: 并集 (OR) - 任一可用 (按分组)                    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 3: 冲突检测与时隙过滤                                           │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ 1. getSlots: 将可用性区间转换为时隙                                │ │
│  │ 2. checkForConflicts: 检测冲突                                    │ │
│  │    - 预留时隙 (Reserved Slots) - 其他用户正在选择                 │ │
│  │    - 座位预订 (Seats) - 多座位事件的特殊处理                      │ │
│  │ 3. 边界过滤: 最小预订通知、未来限制                                 │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (仅轮询分配)
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 4: 轮询分配与公平性保障                                         │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ LuckyUserService.getLuckyUser: 选择"幸运主持人"                  │ │
│  │                                                                   │ │
│  │ 优先级流程:                                                        │ │
│  │ 1. 单用户短路: 只有一个可用用户时直接返回                          │ │
│  │ 2. 权重过滤 (isRRWeightsEnabled): 基于历史预订缺口                │ │
│  │ 3. 优先级过滤: 选择最高优先级用户组                                │ │
│  │ 4. LRB 兜底: 同优先级内选择最近最少预订的用户                      │ │
│  │                                                                   │ │
│  │ 公平性校准:                                                        │ │
│  │ - OOO 校准: 休假期间的预订不计入                                   │ │
│  │ - 新主持人校准: 新加入成员的历史补偿                               │ │
│  │ - 周期重置: 每日/每月重置计数                                     │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 5: 预订确认与最终冲突检测                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ 1. ensureAvailableUsers: 最终验证所选主持人是否可用               │ │
│  │ 2. checkForConflicts (再执行): 防止竞态条件                       │ │
│  │ 3. 创建预订记录                                                    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. 多成员可用时间合并逻辑

### 2.1 核心合并策略

`getAggregatedAvailability` 是团队可用性合并的核心函数，其策略取决于事件的调度类型和主持人配置：

#### 2.1.1 主持人类型识别

```typescript
// packages/features/di/modules/QualifiedHosts.ts:31-43
for (const host of hosts) {
  const qualifiedHost: QualifiedHost = {
    user: host.user,
    isFixed: host.isFixed,
    groupId: host.groupId ?? null,
  };

  if (host.isFixed || schedulingType !== "ROUND_ROBIN") {
    fixedHosts.push(qualifiedHost);  // 固定主持人
  } else {
    allRRHosts.push(qualifiedHost);   // 轮询主持人
  }
}
```

#### 2.1.2 合并算法详解

```typescript
// packages/features/availability/lib/getAggregatedAvailability/getAggregatedAvailability.ts:25-77
export const getAggregatedAvailability = (
  userAvailability: {
    dateRanges: DateRange[];
    oooExcludedDateRanges: DateRange[];
    user?: { isFixed?: boolean; groupId?: string | null };
  }[],
  schedulingType: SchedulingType | null
): DateRange[] => {
  const isTeamEvent =
    schedulingType === SchedulingType.COLLECTIVE ||
    schedulingType === SchedulingType.ROUND_ROBIN ||
    userAvailability.length > 1;

  // ========== 步骤 1: 处理固定主持人 ==========
  // 固定主持人必须全部同时可用 (交集逻辑)
  const fixedHosts = userAvailability.filter(
    ({ user }) => !schedulingType || 
                   schedulingType === SchedulingType.COLLECTIVE || 
                   user?.isFixed
  );

  // 对固定主持人的可用性取交集
  const fixedDateRanges = mergeOverlappingDateRanges(
    intersect(fixedHosts.map((s) => 
      !isTeamEvent ? s.dateRanges : s.oooExcludedDateRanges
    ))
  );

  // ========== 步骤 2: 处理轮询主持人 (按分组) ==========
  const roundRobinHosts = userAvailability.filter(({ user }) => user?.isFixed !== true);
  
  if (roundRobinHosts.length) {
    // 按 groupId 分组
    const hostsByGroup = roundRobinHosts.reduce(
      (groups, host) => {
        const groupId = host.user?.groupId || DEFAULT_GROUP_ID;
        if (!groups[groupId]) {
          groups[groupId] = [];
        }
        groups[groupId].push(host);
        return groups;
      },
      {} as Record<string, typeof roundRobinHosts>
    );

    // 每个组内取并集 (任一主持人可用即可)
    // 但组与组之间取交集 (每个组都需要有可用主持人)
    Object.values(hostsByGroup).forEach((groupHosts) => {
      if (groupHosts.length > 0) {
        const groupDateRanges = groupHosts.flatMap((s) =>
          !isTeamEvent ? s.dateRanges : s.oooExcludedDateRanges
        );
        dateRangesToIntersect.push(groupDateRanges ?? []);
      }
    });
  }

  // ========== 步骤 3: 最终合并 ==========
  // 固定主持人交集 ∩ 各组轮询主持人的并集
  const availability = intersect(dateRangesToIntersect);
  
  // 去重、排序、过滤冗余
  const uniqueRanges = uniqueAndSortedDateRanges(availability);
  return filterRedundantDateRanges(uniqueRanges);
};
```

### 2.2 合并策略矩阵

| 调度类型 | 固定主持人 | 轮询主持人 | 合并逻辑 |
|---------|-----------|-----------|---------|
| **COLLECTIVE** | 全部主持人 | 无 | 所有主持人的交集 |
| **ROUND_ROBIN (无分组)** | 必须全部可用 | 任一可用 | 固定交集 ∩ 轮询并集 |
| **ROUND_ROBIN (有分组)** | 必须全部可用 | 每组至少一个可用 | 固定交集 ∩ (组1并集 ∩ 组2并集 ∩ ...) |

### 2.3 测试用例验证

从测试用例可以看到具体的合并行为：

**场景 1: 纯轮询 (无固定主持人)**
```typescript
// User A: 11:00-11:20, 16:10-16:30 可用
// User B: 11:15-11:30, 13:20-13:30 可用
// 期望: 11:00-11:30 期间应该有可用时隙
// 但 11:00-11:30 整个区间不是同时可用的
// 实际: 11:00-11:20 (A可用), 11:15-11:30 (B可用)
//      合并后显示为两个独立的可用区间
```

**场景 2: 固定 + 轮询混合**
```typescript
// Fixed Hosts A, B: 都必须可用
// RR Host C: 11:00-11:30 可用
// RR Host D: 12:30-13:00 可用
// 期望: 
// - 11:00-11:30: A+B 交集可用 + C 可用 → 显示
// - 12:30-13:00: A+B 交集可用 + D 可用 → 显示
// - 13:15-13:30: A+B 可用但无 RR 可用 → 不显示
```

**场景 3: 多分组轮询**
```typescript
// Group 1: A (11:00-11:30), B (12:00-12:30)
// Group 2: C (11:15-11:45), D (12:15-12:45)
// Fixed Host: 11:00-13:00 可用
// 期望:
// - 11:15-11:30: Group1有A, Group2有C → 可用
// - 12:15-12:30: Group1有B, Group2有D → 可用
// - 其他时间: 至少一个组无可用主持人 → 不可用
```

---

## 3. 冲突检测环节分析

冲突检测在团队预约链路中发生在多个层次，形成**纵深防御**体系。

### 3.1 冲突检测层次图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    冲突检测层次体系                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 层次 1: 预订限制冲突 (Booking/Duration Limits)                │   │
│  │ 位置: getBusyTimesFromLimitsForUsers                          │   │
│  │ 检测:                                                          │   │
│  │ - PER_DAY/PER_WEEK/PER_MONTH/PER_YEAR 预订数量限制            │   │
│  │ - PER_DAY/PER_WEEK/PER_MONTH/PER_YEAR 时长限制                │   │
│  │ - 团队级限制 vs 用户级限制                                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 层次 2: 日历忙时冲突 (Calendar Busy Times)                   │   │
│  │ 位置: getUserAvailability → getBusyTimesService             │   │
│  │ 检测:                                                          │   │
│  │ - 外部日历 (Google/Outlook/Exchange) 的事件                   │   │
│  │ - 前后缓冲时间 (Buffer Time)                                   │   │
│  │ - 已有预订 (Existing Bookings)                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 层次 3: 预留时隙冲突 (Reserved Slots)                        │   │
│  │ 位置: _getAvailableSlots → checkForConflicts                 │   │
│  │ 检测:                                                          │   │
│  │ - 其他用户正在选择的时隙 (临时锁定)                            │   │
│  │ - 防止竞态条件 (Race Condition)                                │   │
│  │ - 多用户同时选择同一时隙                                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 层次 4: 座位事件冲突 (Seats Events)                          │   │
│  │ 位置: checkForConflicts → currentSeats 特殊处理              │   │
│  │ 检测:                                                          │   │
│  │ - 多座位预订，同一时隙可容纳多个预订                            │   │
│  │ - 座位数未满时不视为冲突                                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 层次 5: 预订时最终验证 (Final Validation)                    │   │
│  │ 位置: ensureAvailableUsers                                    │   │
│  │ 检测:                                                          │   │
│  │ - 防止时隙显示后到预订前的状态变化                             │   │
│  │ - 最终确认所选主持人可用性                                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心冲突检测函数

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
  // 快速返回: 无忙时时无冲突
  if (!Array.isArray(busy) || busy.length < 1) {
    return false;
  }
  
  // 座位事件特殊处理: 同一时隙已存在预订但座位未满时不视为冲突
  if (currentSeats?.some((booking) => 
    booking.startTime.toISOString() === time.toISOString()
  )) {
    return false;
  }
  
  // 计算时隙的开始和结束时间戳
  const slotStart = time.valueOf();
  const slotEnd = slotStart + eventLength * 60 * 1000;
  
  // 对忙时排序以便于范围检测
  const sortedBusyTimes = busy
    .map((busyTime) => ({
      start: dayjs.utc(busyTime.start).valueOf(),
      end: dayjs.utc(busyTime.end).valueOf(),
    }))
    .sort((a, b) => a.start - b.start);
  
  // 区间重叠检测算法
  for (const busyTime of sortedBusyTimes) {
    // 忙时开始 >= 时隙结束: 后续都不会重叠，提前退出
    if (busyTime.start >= slotEnd) {
      break;
    }
    // 忙时结束 <= 时隙开始: 完全不重叠，继续
    if (busyTime.end <= slotStart) {
      continue;
    }
    // 存在重叠: 返回冲突
    return true;
  }
  
  return false;
}
```

### 3.3 冲突检测在时隙获取流程中的位置

```typescript
// packages/trpc/server/routers/viewer/slots/util.ts:1166-1244
// 预留时隙获取与清理
const reservedSlots = await this._getReservedSlotsAndCleanupExpired({
  bookerClientUid,
  usersWithCredentials,
  eventTypeId,
});

// 冲突检测与过滤
if (reservedSlots?.length > 0) {
  // 分离座位预留和时隙预留
  let occupiedSeats: typeof reservedSlots = reservedSlots.filter(
    (item) => item.isSeat && item.eventTypeId === eventType.id
  );
  
  const busySlotsFromReservedSlots = reservedSlots.reduce<EventBusyDate[]>((r, c) => {
    if (!c.isSeat) {
      r.push({ start: c.slotUtcStartDate, end: c.slotUtcEndDate });
    }
    return r;
  }, []);
  
  // 过滤掉有冲突的时隙
  availableTimeSlots = availableTimeSlots
    .map((slot) => {
      if (
        !checkForConflicts({
          time: slot.time,
          busy: busySlotsFromReservedSlots,
          ...availabilityCheckProps,
        })
      ) {
        return slot;
      }
      return undefined;
    })
    .filter((item): item is { time: Dayjs; userIds?: number[] } => {
      return !!item;
    });
}
```

### 3.4 预订限制的忙时计算

```typescript
// packages/trpc/server/routers/viewer/slots/util.ts:478-643
// 预订限制和时长限制的忙时计算
private async _getBusyTimesFromLimitsForUsers(
  users: { id: number; email: string }[],
  bookingLimits: IntervalLimit | null,
  durationLimits: IntervalLimit | null,
  dateFrom: Dayjs,
  dateTo: Dayjs,
  duration: number | undefined,
  eventType: NonNullable<EventType>,
  timeZone: string,
  rescheduleUid?: string
) {
  const userBusyTimesMap = new Map<number, EventBusyDetails[]>();
  const globalLimitManager = new LimitManager();
  
  // ========== 预订限制处理 ==========
  if (bookingLimits) {
    for (const key of descendingLimitKeys) {
      const limit = bookingLimits?.[key];
      if (!limit) continue;
      
      const unit = intervalLimitKeyToUnit(key);
      const periodStartDates = this.dependencies.userAvailabilityService
        .getPeriodStartDatesBetween(dateFrom, dateTo, unit, timeZone);
      
      for (const periodStart of periodStartDates) {
        if (globalLimitManager.isAlreadyBusy(periodStart, unit, timeZone)) continue;
        
        let totalBookings = 0;
        for (const booking of busyTimesFromLimitsBookings) {
          if (!isBookingWithinPeriod(booking, periodStart, periodEnd, timeZone)) {
            continue;
          }
          totalBookings++;
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
  }
  
  // ========== 时长限制处理 ==========
  if (durationLimits) {
    // 类似逻辑: 累计时长超过限制时标记为忙时
  }
  
  return userBusyTimesMap;
}
```

---

## 4. 轮询分配机制详解

轮询分配（Round Robin Assignment）是团队预约中最智能的部分，旨在实现**公平性**和**负载均衡**。

### 4.1 轮询分配决策流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LuckyUserService 决策流程                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  输入: availableUsers (当前时隙可用的主持人列表)                      │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 步骤 0: 单用户短路优化                                          │   │
│  │ if (availableUsers.length === 1)                              │   │
│  │     return availableUsers[0];  // 直接返回，无需计算           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼ (多用户)                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 步骤 1: 数据预获取 (fetchAllDataNeededForCalculations)        │   │
│  │ 并行获取:                                                      │   │
│  │ - 日历忙时 (getCalendarBusyTimesOfInterval)                   │   │
│  │ - 区间内预订记录 (getBookingsOfInterval) x 3                  │   │
│  │ - 主持人创建时间 (findHostsCreatedInInterval)                 │   │
│  │ - 用户最后预订时间 (findUsersWithLastBooking)                 │   │
│  │ - OOO 记录 (findOOOEntriesInInterval)                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 步骤 2: 权重过滤 (仅 isRRWeightsEnabled=true 时)              │   │
│  │ filterUsersBasedOnWeights():                                  │   │
│  │                                                                 │   │
│  │ 公式:                                                          │   │
│  │   targetPercentage = userWeight / totalWeight                 │   │
│  │   targetBookings = (totalBookings + calibration)              │   │
│  │                  × targetPercentage                            │   │
│  │   bookingShortfall = targetBookings                           │   │
│  │                   - (actualBookings + userCalibration)       │   │
│  │                                                                 │   │
│  │ 选择 bookingShortfall 最大的用户组                             │   │
│  │ (缺口最大 = 应该获得更多预订)                                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 步骤 3: 优先级过滤                                            │   │
│  │ getUsersWithHighestPriority():                                │   │
│  │                                                                 │   │
│  │ - 选择 priority 值最高的用户组                                 │   │
│  │ - null 优先级视为默认值 2                                      │   │
│  │ - 优先级相同时继续下一步                                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 步骤 4: 最近最少预订兜底 (LRB)                                 │   │
│  │ leastRecentlyBookedUser():                                   │   │
│  │                                                                 │   │
│  │ 比较维度:                                                       │   │
│  │ 1. 作为组织者的最后预订时间 (organizer bookings)              │   │
│  │ 2. 作为参与者的最后预订时间 (attendee bookings)               │   │
│  │                                                                 │   │
│  │ 取两者中较新的时间进行比较                                      │   │
│  │ 时间最早 = 最久未被分配 = 优先选择                             │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 核心权重分配算法

```typescript
// packages/features/bookings/lib/getLuckyUser.ts:333-453
private filterUsersBasedOnWeights<
  T extends PartialUser & { weight?: number | null },
>({
  availableUsers,
  bookingsOfAvailableUsersOfInterval,
  bookingsOfNotAvailableUsersOfInterval,
  allRRHosts,
  allRRHostsBookingsOfInterval,
  allRRHostsCreatedInInterval,
  attributeWeights,
  oooData,
}: GetLuckyUserParams<T> & FetchedData) {
  
  // ========== 校准计算 ==========
  // 1. OOO 校准: 用户休假期间的预订需要"补偿"
  // 2. 新主持人校准: 新加入的用户需要历史数据补偿
  const allHostsWithCalibration = this.getHostsWithCalibration({
    hosts: allRRHosts.map((host) => {
      return { email: host.user.email, userId: host.user.id, createdAt: host.createdAt };
    }),
    allRRHostsBookingsOfInterval,
    allRRHostsCreatedInInterval,
    oooData,
  });
  
  // ========== 权重与目标计算 ==========
  // 计算总权重
  let totalWeight: number;
  if (attributeWeights && attributeWeights.length > 0) {
    totalWeight = attributeWeights.reduce((totalWeight, userWeight) => {
      totalWeight += userWeight.weight ?? 100;
      return totalWeight;
    }, 0);
  } else {
    totalWeight = allRRHosts.reduce((totalWeight, host) => {
      totalWeight += host.weight ?? 100;
      return totalWeight;
    }, 0);
  }
  
  // ========== 为每个用户计算预订缺口 ==========
  const usersWithBookingShortfalls = availableUsers.map((user) => {
    let userWeight = user.weight ?? 100;
    if (attributeWeights) {
      userWeight = attributeWeights.find((userWeight) => 
        userWeight.userId === user.id
      )?.weight ?? 100;
    }
    
    // 目标百分比 = 用户权重 / 总权重
    const targetPercentage = userWeight / totalWeight;
    
    // 用户实际获得的预订数
    const userBookings = bookingsOfAvailableUsersOfInterval.filter(
      (booking) =>
        booking.userId === user.id || 
        booking.attendees.some((attendee) => attendee.email === user.email)
    );
    
    // 目标预订数 = (总预订数 + 总校准) × 目标百分比
    const targetNumberOfBookings = 
      (allBookings.length + totalCalibration) * targetPercentage;
    
    // 用户校准值 (OOO + 新主持人)
    const userCalibration = 
      allHostsWithCalibration.find((host) => host.userId === user.id)?.calibration ?? 0;
    
    // 预订缺口 = 目标预订数 - (实际预订数 + 用户校准)
    const bookingShortfall = 
      targetNumberOfBookings - (userBookings.length + userCalibration);
    
    return {
      ...user,
      calibration: userCalibration,
      weight: userWeight,
      targetNumberOfBookings,
      bookingShortfall,
      numBookings: userBookings.length,
    };
  });
  
  // ========== 选择缺口最大的用户 ==========
  const maxShortfall = Math.max(
    ...usersWithBookingShortfalls.map((user) => user.bookingShortfall)
  );
  const usersWithMaxShortfall = usersWithBookingShortfalls.filter(
    (user) => user.bookingShortfall === maxShortfall
  );
  
  // 多个用户缺口相同时，选择权重最大的
  const maxWeight = Math.max(
    ...usersWithMaxShortfall.map((user) => user.weight ?? 100)
  );
  
  const remainingUsersAfterWeightFilter = availableUsers.filter((user) =>
    userIdsWithMaxShortfallAndWeight.has(user.id)
  );
  
  return {
    remainingUsersAfterWeightFilter,
    usersAndTheirBookingShortfalls: usersWithBookingShortfalls.map(...),
  };
}
```

### 4.3 校准机制详解

校准机制是保证公平性的关键，处理两种特殊场景：

#### 4.3.1 OOO (Out of Office) 校准

```typescript
// packages/features/bookings/lib/getLuckyUser.ts:226-309
private getHostsWithCalibration({
  hosts,
  allRRHostsBookingsOfInterval,
  allRRHostsCreatedInInterval,
  oooData,
}: {
  hosts: { userId: number; email: string; createdAt: Date }[];
  allRRHostsBookingsOfInterval: PartialBooking[];
  allRRHostsCreatedInInterval: { userId: number; createdAt: Date }[];
  oooData: OOODataType;
}) {
  // ========== OOO 校准 ==========
  const oooCalibration = new Map<number, number>();
  
  oooData.forEach(({ userId, oooEntries }) => {
    // 只有一个主持人时跳过 (避免除零)
    if (hosts.length <= 1) {
      return;
    }
    
    let calibration = 0;
    
    oooEntries.forEach((oooEntry) => {
      // 找出 OOO 期间其他用户获得的预订
      const bookingsInTimeframe = existingBookings.filter(
        (booking) =>
          booking.createdAt >= oooEntry.start &&
          booking.createdAt <= oooEntry.end &&
          booking.userId !== userId
      );
      
      // 校准值 = 其他用户预订数 / (主持人总数 - 1)
      // 即: 该用户休假期间"应该"获得的预订数
      calibration += bookingsInTimeframe.length / (hosts.length - 1);
    });
    
    oooCalibration.set(userId, calibration);
  });
  
  // ========== 新主持人校准 ==========
  let newHostsWithCalibration: Map<
    number,
    { calibration: number; userId: number; createdAt: Date }
  > = new Map();
  
  if (allRRHostsCreatedInInterval.length && existingBookings.length) {
    newHostsWithCalibration = new Map(
      allRRHostsCreatedInInterval.map((newHost) => [
        newHost.userId,
        { ...newHost, calibration: calculateNewHostCalibration(newHost) },
      ])
    );
  }
  
  // 合并两种校准
  return hosts.map((host) => ({
    ...host,
    calibration:
      (newHostsWithCalibration.get(host.userId)?.calibration ?? 0) +
      (oooCalibration.get(host.userId) ?? 0),
  }));
}

// 新主持人校准计算
function calculateNewHostCalibration(newHost: { userId: number; createdAt: Date }) {
  // 找出新主持人加入之前的预订
  const existingBookingsBeforeAdded = existingBookings.filter(
    (booking) => 
      booking.userId !== newHost.userId && 
      booking.createdAt < newHost.createdAt
  );
  
  // 找出新主持人加入之前已存在的其他主持人
  const hostsAddedBefore = hosts.filter(
    (host) => 
      host.userId !== newHost.userId && 
      host.createdAt < newHost.createdAt
  );
  
  // 校准值 = 加入前总预订数 / 加入前主持人数
  // 即: 假设新主持人从一开始就存在，"应该"获得的预订数
  const calibration =
    existingBookingsBeforeAdded.length && hostsAddedBefore.length
      ? existingBookingsBeforeAdded.length / hostsAddedBefore.length
      : 0;
      
  return calibration;
}
```

#### 4.3.2 校准场景示例

**场景 A: OOO 校准**
```
团队: [Alice, Bob, Charlie] (3人)
周期: 5月1日 - 5月31日

事件:
- 5月10日-5月20日: Alice OOO (休假)
- 5月15日: 获得预订 (分配给 Bob)
- 5月18日: 获得预订 (分配给 Charlie)

OOO 期间其他用户获得 2 个预订
Alice 的校准值 = 2 / (3-1) = 1

5月下旬:
- Alice 实际: 0 个预订
- 但加上校准值 1，计算时视为 1 个
- 这让 Alice 在缺口计算中更有竞争力
```

**场景 B: 新主持人校准**
```
5月1日: 团队有 [Alice, Bob] (2人)
5月1日-5月14日: 共获得 4 个预订 (每人约 2 个)

5月15日: Charlie 加入团队
5月15日-5月31日: 共获得 2 个预订

Charlie 的校准值 = 4 (加入前总预订) / 2 (加入前主持人数) = 2

缺口计算时:
- Alice: 实际 3 个 + 校准 0 = 3
- Bob:   实际 3 个 + 校准 0 = 3
- Charlie: 实际 0 个 + 校准 2 = 2

Charlie 的缺口更大，优先获得分配
```

### 4.4 周期重置机制

```typescript
// packages/features/bookings/lib/getLuckyUser.ts:58-115
// 周期开始时间计算
export const getIntervalStartDate = ({
  interval,
  rrTimestampBasis,
  meetingStartTime,
}: {
  interval: RRResetInterval;
  rrTimestampBasis: RRTimestampBasis;
  meetingStartTime?: Date;
}) => {
  if (rrTimestampBasis === RRTimestampBasis.START_TIME) {
    // 基于会议开始时间
    if (!meetingStartTime) {
      throw new Error("Meeting start time is required");
    }
    if (interval === RRResetInterval.DAY) {
      return startOfDay(meetingStartTime);  // 当天开始
    }
    return startOfMonth(meetingStartTime);  // 当月开始
  }
  
  // 基于预订创建时间 (当前时间)
  if (interval === RRResetInterval.DAY) {
    return startOfDay();
  }
  return startOfMonth();
};

// 周期结束时间计算
export const getIntervalEndDate = ({
  interval,
  rrTimestampBasis,
  meetingStartTime,
}: {
  interval: RRResetInterval;
  rrTimestampBasis: RRTimestampBasis;
  meetingStartTime?: Date;
}) => {
  if (rrTimestampBasis === RRTimestampBasis.START_TIME) {
    if (!meetingStartTime) {
      throw new Error("Meeting start time is required");
    }
    if (interval === RRResetInterval.DAY) {
      return endOfDay(meetingStartTime);  // 当天结束
    }
    return endOfMonth(meetingStartTime);  // 当月结束
  }
  
  return new Date();  // 当前时间
};
```

#### 4.4.1 时间戳基准选项

| 基准类型 | RRTimestampBasis | 行为描述 | 适用场景 |
|---------|------------------|---------|---------|
| **创建时间** | `CREATED_AT` | 从预订创建时刻倒推周期 | 实时统计，每日/每月重置 |
| **会议开始时间** | `START_TIME` | 基于会议实际开始时间 | 按会议日期统计，如"5月15日的会议" |

---

## 5. 公平性边界分析

### 5.1 公平性保障体系

```
┌─────────────────────────────────────────────────────────────────────┐
│                    公平性保障层次结构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第一层: 基础公平性                                             │   │
│  │ ┌─────────────────────────────────────────────────────────┐ │   │
│  │ │ • 单用户短路: 只有一个可用用户时直接返回                   │ │   │
│  │ │ • 最近最少预订 (LRB): 同优先级内时间最早的优先             │ │   │
│  │ │ • 用户 ID 兜底: 时间完全相同时按 ID 排序                   │ │   │
│  │ └─────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第二层: 权重公平性 (isRRWeightsEnabled)                      │   │
│  │ ┌─────────────────────────────────────────────────────────┐ │   │
│  │ │ • 权重比例: 用户权重 / 总权重 = 目标分配比例              │ │   │
│  │ │ • 缺口计算: 目标预订数 - 实际预订数                       │ │   │
│  │ │ • 多维度比较: 缺口最大 → 权重最大 → LRB                   │ │   │
│  │ └─────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第三层: 异常场景校准                                          │   │
│  │ ┌─────────────────────────────────────────────────────────┐ │   │
│  │ │ • OOO 校准: 休假期间的预订不计入实际获得                  │ │   │
│  │ │ • 新主持人校准: 加入前的历史数据补偿                      │ │   │
│  │ │ • 连续预订防护: 周期重置防止累积不公平                    │ │   │
│  │ └─────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 第四层: 优先级分层                                            │   │
│  │ ┌─────────────────────────────────────────────────────────┐ │   │
│  │ │ • 优先级独立: 高优先级用户完全不与低优先级竞争            │ │   │
│  │ │ • 组内公平: 同一优先级内应用权重/LRB逻辑                  │ │   │
│  │ │ • null 处理: 未设置优先级视为默认值 2                     │ │   │
│  │ └─────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 忙闲不均场景分析

#### 场景 1: 成员可用时间差异大

```
团队配置:
- Alice: 权重 100，每周一、二、三可用
- Bob:   权重 100，每周四、五可用

问题:
- 周一至周三的预订只能分配给 Alice
- 周四、周五的预订只能分配给 Bob
- 按权重比例 50%:50% 分配不现实

系统行为分析:

1. **时隙获取阶段**:
   - 周一: 只有 Alice 可用 → 单用户短路 → Alice
   - 周二: 只有 Alice 可用 → 单用户短路 → Alice
   - 周四: 只有 Bob 可用 → 单用户短路 → Bob
   - 周五: 只有 Bob 可用 → 单用户短路 → Bob

2. **权重计算阶段**:
   - 当某时隙只有一个可用用户时，不进入权重计算
   - `availableUsers.length === 1` 直接返回
   - 这避免了"可用时间少的用户永远无法达标"的问题

3. **关键设计**:
   - 权重是**相对比例**，不是**绝对配额**
   - 只在"多个用户同时可用"的时隙进行公平竞争
   - 单用户时隙直接分配，不影响权重计算
```

#### 场景 2: 成员加入时间不同

```
时序:
- Day 1: Alice, Bob 加入 (权重各 100)
- Day 1-Day 14: 共 10 个预订 → Alice 5, Bob 5
- Day 15: Charlie 加入 (权重 100)
- Day 15-Day 30: 新预订到来

没有校准时的问题:
- Day 15 计算:
  - 总预订: 10
  - Alice 实际: 5, 目标: 3.33 → 缺口: -1.67 (超额)
  - Bob 实际: 5, 目标: 3.33 → 缺口: -1.67 (超额)
  - Charlie 实际: 0, 目标: 3.33 → 缺口: 3.33 (不足)

- Charlie 缺口最大，优先获得分配
- 但 Alice 和 Bob 前期获得的 5 个会被"惩罚"

新主持人校准的作用:
- Charlie 的校准值 = 10 (加入前总预订) / 2 (加入前主持人数) = 5
- 计算时:
  - Charlie 实际: 0 + 校准 5 = 5
  - 与 Alice、Bob 持平
  - 三人公平竞争后续预订
```

### 5.3 连续预订场景分析

#### 场景 1: 同一用户连续获得预订

```
团队: Alice (100), Bob (100)
场景: 连续 5 个预订请求，两人都可用

没有公平机制时:
- 可能出现 Alice, Alice, Alice, Alice, Alice (完全不公平)

实际系统行为 (权重模式):

预订 1:
- 总预订: 0
- Alice 目标: 0, 实际: 0 → 缺口: 0
- Bob 目标: 0, 实际: 0 → 缺口: 0
- 同缺口 → 同权重 → LRB 选择时间较早的
- 假设 Alice 被选中

预订 2:
- 总预订: 1
- Alice 目标: 0.5, 实际: 1 → 缺口: -0.5 (超额)
- Bob 目标: 0.5, 实际: 0 → 缺口: 0.5 (不足)
- Bob 缺口更大 → Bob 被选中

预订 3:
- 总预订: 2
- Alice 目标: 1, 实际: 1 → 缺口: 0
- Bob 目标: 1, 实际: 1 → 缺口: 0
- LRB 选择 → 假设 Alice

预订 4:
- 总预订: 3
- Alice 目标: 1.5, 实际: 2 → 缺口: -0.5
- Bob 目标: 1.5, 实际: 1 → 缺口: 0.5
- Bob 被选中

结果: Alice, Bob, Alice, Bob (近似轮询)
```

#### 场景 2: 周期重置的影响

```
团队: Alice (100), Bob (100)
周期: 每日重置 (RRResetInterval.DAY)

Day 1:
- 预订 1: Alice (缺口平衡)
- 预订 2: Bob (Alice 超额)
- 预订 3: Alice (平衡)
- Day 1 结束: Alice 2, Bob 1

Day 2 (周期重置):
- 前一天的计数清零
- 预订 4: 重新从平衡状态开始
- 不会出现"Alice 前一天多了，今天一直让着 Bob"

关键设计:
- 周期重置防止不公平累积
- 每日/每月是"公平统计窗口"，不是"严格配额"
- 长期来看趋于平衡，但不保证每个周期完全相等
```

### 5.4 公平性边界与限制

#### 5.4.1 设计权衡

| 考量因素 | 当前设计 | 权衡说明 |
|---------|---------|---------|
| **实时性 vs 历史积累** | 周期重置 | 防止历史数据影响未来，但可能出现短期连续分配 |
| **严格公平 vs 可用性优先** | 单用户短路 | 有可用时隙比"等轮到某人"更重要 |
| **权重比例 vs 实际可用** | 权重是相对目标 | 不能分配不可用的时隙给权重高的用户 |
| **简单性 vs 精确性** | 近似算法 | 复杂计算可能引入性能问题和边缘 case |

#### 5.4.2 已知边界情况

**边界 1: 完全不对称的可用时间**
```
- Alice: 仅周一 10:00-11:00 可用
- Bob: 其他所有时间都可用

结果:
- 周一 10:00-11:00 的预订 → 只能给 Alice
- 其他时间 → Bob 和 Alice 竞争 (但 Alice 不可用)
- 实际: Bob 获得绝大多数预订

这是**预期行为**，不是 bug:
- 公平性是"在可用的前提下的公平"
- 不能强迫 Alice 在不可用的时间接预订
- 权重是"竞争时的优先级"，不是"强制配额"
```

**边界 2: 优先级覆盖权重**
```
- Alice: 优先级 0 (最低), 权重 200
- Bob: 优先级 4 (最高), 权重 50

结果:
- 任何时隙只要 Bob 可用 → Bob 被选中
- Alice 只有在 Bob 不可用时才有机会

这是**预期行为**:
- 优先级是"分层隔离"的
- 高优先级用户完全在另一个竞争层级
- 用途: VIP 客服、紧急联系人等场景
```

---

## 6. 关键代码位置索引

### 6.1 可用性计算

| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| 多成员可用性合并 | `packages/features/availability/lib/getAggregatedAvailability/getAggregatedAvailability.ts` | `getAggregatedAvailability` |
| 单用户可用性计算 | `packages/features/availability/lib/getUserAvailability.ts` | `UserAvailabilityService` |
| 时隙生成 | `packages/features/schedules/lib/slots.ts` | `getSlots` |
| 日期区间操作 | `packages/features/schedules/lib/date-ranges.ts` | `intersect`, `mergeOverlappingRanges` |

### 6.2 冲突检测

| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| 核心冲突检测 | `packages/features/bookings/lib/conflictChecker/checkForConflicts.ts` | `checkForConflicts` |
| 预订限制忙时 | `packages/trpc/server/routers/viewer/slots/util.ts` | `_getBusyTimesFromLimitsForUsers` |
| 预留时隙管理 | `packages/trpc/server/routers/viewer/slots/util.ts` | `_getReservedSlotsAndCleanupExpired` |
| 日历忙时获取 | `packages/features/busyTimes/services/getBusyTimes.ts` | `getBusyTimes` |

### 6.3 轮询分配

| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| 幸运用户选择 | `packages/features/bookings/lib/getLuckyUser.ts` | `LuckyUserService.getLuckyUser` |
| 主持人筛选 | `packages/features/di/modules/QualifiedHosts.ts` | `findQualifiedHostsWithDelegationCredentials` |
| 权重过滤 | `packages/features/bookings/lib/getLuckyUser.ts` | `filterUsersBasedOnWeights` |
| 校准计算 | `packages/features/bookings/lib/getLuckyUser.ts` | `getHostsWithCalibration` |
| 最近最少预订 | `packages/features/bookings/lib/getLuckyUser.ts` | `leastRecentlyBookedUser` |

### 6.4 时隙服务集成

| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| 可用时隙主服务 | `packages/trpc/server/routers/viewer/slots/util.ts` | `AvailableSlotsService._getAvailableSlots` |
| 主持人可用性计算 | `packages/trpc/server/routers/viewer/slots/util.ts` | `calculateHostsAndAvailabilities` |
| 屏蔽过滤 | `packages/features/watchlist/operations/filter-blocked-hosts.controller.ts` | `filterBlockedHosts` |

---

## 7. 测试覆盖分析

### 7.1 核心测试文件

| 测试文件 | 覆盖场景 |
|---------|---------|
| `getAggregatedAvailability.test.ts` | 固定主持人交集、轮询并集、多分组、混合模式 |
| `getLuckyUser.test.ts` | 权重轮询、优先级、OOO 校准、新主持人校准、周期计算 |
| `checkForConflicts.test.ts` | 忙时冲突检测、座位事件特殊处理 |

### 7.2 关键测试场景

**getAggregatedAvailability 测试覆盖:**
1. RR 主持人可用性合并不应产生错误时隙
2. 固定主持人正确取交集
3. 固定 + RR 混合模式
4. 多分组轮询 (每组至少一个可用)
5. 同组多 RR 主持人的并集行为
6. 空分组、无可用主持人的边界情况

**getLuckyUser 测试覆盖:**
1. 单用户短路优化
2. 无权重时的 LRB 选择
3. 优先级过滤
4. 相同权重时的缺口计算
5. 不同权重时的目标比例
6. OOO 期间的校准
7. 新主持人加入的校准
8. 周期开始/结束时间计算
9. 时间戳基准 (CREATED_AT vs START_TIME)

---

## 8. 总结与建议

### 8.1 架构亮点

1. **分层设计清晰**
   - 可用性计算、冲突检测、轮询分配各层职责明确
   - 每层都有独立的测试覆盖

2. **公平性保障周全**
   - 多维度: 权重、优先级、LRB、校准
   - 异常场景处理: OOO、新主持人、忙闲不均

3. **性能优化到位**
   - 单用户短路: 避免不必要的复杂计算
   - 并行数据获取: `Promise.all` 并行查询
   - 周期重置: 防止历史数据无限累积

4. **防御性设计**
   - 多层次冲突检测: 防止竞态条件
   - 预留时隙: 多用户同时选择的保护
   - 最终验证: 预订时的二次确认

### 8.2 潜在优化点

1. **可观测性增强**
   - 当前已有详细日志 (`loggerWithEventDetails`)
   - 可考虑添加公平性指标监控 (分配比例偏差)

2. **边缘 case 处理**
   - 主持人数量动态变化的场景
   - 权重配置极端值 (如 1:1000 比例)

3. **配置友好性**
   - 权重和优先级的 UI 配置可能需要更多引导
   - 不同调度类型的行为差异需要清晰的文档

### 8.3 关键设计原则回顾

| 原则 | 实现方式 |
|-----|---------|
| **可用性优先** | 单用户短路、不强迫不可用时隙 |
| **相对公平** | 权重是目标比例，不是强制配额 |
| **周期平衡** | 每日/每月重置，防止不公平累积 |
| **层次隔离** | 优先级分层，高优先级不与低优先级竞争 |
| **异常补偿** | OOO 校准、新主持人校准 |
| **纵深防御** | 多层次冲突检测、最终验证 |

---

*报告生成时间: 2026-05-03*
*分析范围: packages/features/availability, packages/features/bookings, packages/trpc/server/routers/viewer/slots*
