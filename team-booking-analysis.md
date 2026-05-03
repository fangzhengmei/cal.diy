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
│  │                                                                 │   │
│  │ ⚠️ 关键: 优先级过滤是在【权重过滤的结果】上进行的               │   │
│  │    - 如果启用权重: 先权重，再优先级，再 LRB                    │   │
│  │    - 如果未启用权重: 直接优先级，再 LRB                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼ (权重过滤后的可用用户)                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 步骤 3: 优先级过滤 (总是执行)                                  │   │
│  │ getUsersWithHighestPriority():                                │   │
│  │                                                                 │   │
│  │ - 选择 priority 值最高的用户组                                 │   │
│  │ - null 优先级视为默认值 2                                      │   │
│  │ - 优先级分层: 高优先级用户完全不与低优先级竞争                  │   │
│  │ - 如果只剩一个用户: 直接返回，跳过 LRB                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼ (优先级过滤后的可用用户)                │
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

#### 4.1.1 权重与优先级的实际执行顺序

从 `getLuckyUser_requiresDataToBePreFetched` 方法（`packages/features/bookings/lib/getLuckyUser.ts:716-776`）可以看到实际执行顺序：

```typescript
// 步骤 1: 权重过滤 (仅 isRRWeightsEnabled=true 时)
if (eventType.isRRWeightsEnabled) {
  const { remainingUsersAfterWeightFilter, ... } = this.filterUsersBasedOnWeights({...});
  availableUsers = remainingUsersAfterWeightFilter;  // 更新可用用户列表
}

// 步骤 2: 优先级过滤 (总是执行，在权重过滤的结果上)
const highestPriorityUsers = this.getUsersWithHighestPriority({ availableUsers });

// 步骤 3: LRB 兜底 (在优先级过滤的结果上)
if (highestPriorityUsers.length === 1) {
  return highestPriorityUsers[0];  // 只剩一个，跳过 LRB
}
return this.leastRecentlyBookedUser({ availableUsers: highestPriorityUsers, ... });
```

**关键理解**：

| 场景 | 执行顺序 | 说明 |
|-----|---------|------|
| **启用权重模式** | 权重过滤 → 优先级过滤 → LRB | 先基于历史缺口选出候选，再在候选中选最高优先级 |
| **未启用权重模式** | 优先级过滤 → LRB | 跳过权重，直接基于优先级和 LRB 选择 |

**优先级分层的含义**：
- 优先级是**硬隔离**的，不是软排序
- 高优先级用户（priority=4）完全不与低优先级用户（priority=0）竞争
- 即使高优先级用户权重低、历史预订多，只要他可用，就会被优先选中
- 只有当同一优先级内有多个用户时，才会使用权重/LRB 逻辑

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

校准机制是保证公平性的关键，处理两种特殊场景：OOO（休假）和新主持人加入。

#### 4.3.0 OOO 数据来源：显式记录 + 历史全日忙碌事件

在 `fetchAllDataNeededForCalculations` 方法中（`packages/features/bookings/lib/getLuckyUser.ts:618-664`），OOO 数据来源于两个渠道：

```typescript
// ========== 数据源 1: 显式 OOO 记录 ==========
const oooEntries = await this.oooRepository.findOOOEntriesInInterval({
  userIds: allRRHosts.map((host) => host.user.id),
  startDate: intervalStartDate,
  endDate: intervalEndDate,
});

// ========== 数据源 2: 日历中的历史全日忙碌事件 ==========
const userFullDayBusyTimes = new Map<number, { start: Date; end: Date }[]>();

userBusyTimesOfInterval.forEach((userBusyTime) => {
  const fullDayBusyTimes = userBusyTime.busyTimes
    .filter((busyTime) => {
      if (!busyTime.timeZone) return false;
      const timezoneOffset = dayjs(busyTime.start).tz(busyTime.timeZone).utcOffset() * 60000;
      let start = new Date(new Date(busyTime.start).getTime() + timezoneOffset);
      const end = new Date(new Date(busyTime.end).getTime() + timezoneOffset);

      // 关键条件 1: 必须是已结束的事件 (end < now)
      // 关键条件 2: 时差必须是 24 小时的整数倍
      return end.getTime() < Date.now() && isFullDayEvent(start, end);
    })
    .map((busyTime) => ({ start: new Date(busyTime.start), end: new Date(busyTime.end) }));

  userFullDayBusyTimes.set(userBusyTime.userId, fullDayBusyTimes);
});

// ========== 合并两种数据源 ==========
userFullDayBusyTimes.forEach((fullDayBusyTimes, userId) => {
  const oooEntriesForUser = oooEntriesGroupedByUserId.get(userId) || [];
  // 合并显式 OOO + 历史全日忙碌
  const combinedEntries = [...oooEntriesForUser, ...fullDayBusyTimes];
  // 合并重叠区间
  const oooEntries = mergeOverlappingRanges(combinedEntries);

  oooData.push({
    userId,
    oooEntries,
  });
});
```

**全日忙碌事件的判定条件**（`isFullDayEvent` 函数）：

```typescript
function isFullDayEvent(date1: Date, date2: Date) {
  const MILLISECONDS_IN_A_DAY = 24 * 60 * 60 * 1000;
  const difference = Math.abs(date1.getTime() - date2.getTime());
  // 时差必须是 24 小时的整数倍
  return difference % MILLISECONDS_IN_A_DAY === 0;
}
```

**OOO 数据来源汇总**：

| 来源类型 | 获取方式 | 条件限制 | 典型场景 |
|---------|---------|---------|---------|
| **显式 OOO 记录** | `PrismaOOORepository.findOOOEntriesInInterval` | 无额外限制（除时间区间） | 用户手动设置的休假、请假 |
| **历史全日忙碌事件** | 从 `getCalendarBusyTimesOfInterval` 筛选 | 1. 必须有 `timeZone` 信息<br>2. 必须是**已结束**的事件 (`end < now`)<br>3. 时差必须是 24 小时的整数倍 | Google Calendar 等日历中的全天事件（如"出差"、"会议"等） |

**设计意图**：
- 显式 OOO 记录：用户主动标记的不可用时间
- 历史全日忙碌事件：隐式推断用户可能处于"不可用"状态（如出差、全天会议）
- 只统计**已结束**的事件：因为这些事件已经"错过"了分配机会，需要进行补偿校准

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

### 4.5 No-Show 配置对公平性的影响

#### 4.5.1 配置项说明

轮询统计中是否计入 No-Show（爽约）由 `EventType.includeNoShowInRRCalculation` 字段控制：

```typescript
// packages/prisma/schema.prisma:263
model EventType {
  // ...
  includeNoShowInRRCalculation  Boolean  @default(false)
  // ...
}
```

**默认值**: `false`（No-Show 不计入轮询统计）

#### 4.5.2 查询过滤逻辑

在 `BookingRepository.buildWhereClauseForActiveBookings` 方法中（`packages/features/bookings/repositories/BookingRepository.ts:111-165`）：

```typescript
const buildWhereClauseForActiveBookings = ({
  includeNoShowInRRCalculation = false,  // 默认不计入
  users,
  ...
}: {...}): Prisma.BookingWhereInput => ({
  OR: [
    {
      userId: { in: users.map((user) => user.id) },
      // 如果 includeNoShowInRRCalculation 为 false，排除 noShowHost=true 的预订
      ...(!includeNoShowInRRCalculation
        ? {
            OR: [{ noShowHost: false }, { noShowHost: null }],
          }
        : {}),
    },
    {
      attendees: {
        some: {
          email: { in: users.map((user) => user.email) },
        },
      },
    },
  ],
  // 如果 includeNoShowInRRCalculation 为 false，排除 noShow=true 的参与者预订
  ...(!includeNoShowInRRCalculation ? { attendees: { some: { noShow: false } } } : {}),
  status: BookingStatus.ACCEPTED,
  // ...
});
```

#### 4.5.3 No-Show 的两个维度

| 维度 | 字段名 | 含义 |
|-----|-------|------|
| **组织者维度** | `Booking.noShowHost` | 主持人（组织者）是否爽约 |
| **参与者维度** | `Attendee.noShow` | 参与者（客户）是否爽约 |

#### 4.5.4 公平性影响分析

**场景示例**：
```
团队配置: Alice (权重 100), Bob (权重 100)
周期: 5月1日 - 5月31日

事件序列:
- 5月10日: 预订分配给 Alice，但客户爽约 (noShow=true)
- 5月15日: 新预订请求
```

**情况 A: includeNoShowInRRCalculation = false（默认）**

```
5月15日计算:
- 总预订: 0（因为 Alice 的预订被标记为 noShow，不计入）
- Alice 实际: 0, 目标: 0 → 缺口: 0
- Bob 实际: 0, 目标: 0 → 缺口: 0

结果: 公平竞争，Alice 不会因"被分配但客户爽约"而被惩罚
```

**情况 B: includeNoShowInRRCalculation = true**

```
5月15日计算:
- 总预订: 1（Alice 的预订被计入）
- Alice 实际: 1, 目标: 0.5 → 缺口: -0.5 (超额)
- Bob 实际: 0, 目标: 0.5 → 缺口: 0.5 (不足)

结果: Bob 缺口更大，优先获得分配
      Alice 因"客户爽约"而被"惩罚"，暂时失去分配机会
```

**设计意图**：

| 配置值 | 公平性理念 | 适用场景 |
|-------|-----------|---------|
| **false（默认）** | 主持人只对"自己可控的事情"负责<br>客户爽约不是主持人的错 | 大多数业务场景，主持人无法控制客户行为 |
| **true** | 按"实际分配次数"统计<br>不管客户是否爽约，分配了就算数 | 特殊场景，需要严格按分配次数统计 |

**关键理解**：
- 默认配置下，No-Show 的预订**不计入**轮询统计
- 这意味着主持人不会因为"客户爽约"而影响其轮询优先级
- 这是一种"公平性保护"机制，避免主持人因不可控因素被惩罚

---

### 4.6 连续预订（Recurring Booking）流程

连续预订是指客户一次预订多个周期性的时隙（如每周一 10:00，连续 4 周）。轮询分配在连续预订中有特殊的处理逻辑。

#### 4.6.1 整体流程概览

```
┌─────────────────────────────────────────────────────────────────────┐
│                    连续预订处理流程 (RecurringBookingService)         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  输入: 多个周期性时隙的预订请求                                       │
│  (如: 5月1日、5月8日、5月15日、5月22日 每周一 10:00)              │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 阶段 1: 首个周期选人 (isFirstRecurringSlot = true)           │   │
│  │                                                                 │   │
│  │  步骤 1.1: 正常轮询分配                                        │   │
│  │          - 调用 getLuckyUser() 选择主持人                      │   │
│  │          - 记录 luckyUsers (选中的主持人列表)                  │   │
│  │                                                                 │   │
│  │  步骤 1.2: 可用性复验 (可选，由 numSlotsToCheckForAvailability 控制) │   │
│  │          - 检查选中的主持人是否在后续 N 个时隙都可用            │   │
│  │          - N = min(总周期数, numSlotsToCheckForAvailability)  │   │
│  │                                                                 │   │
│  │  步骤 1.3: 回退/重试 (如果复验失败)                             │   │
│  │          - 将当前主持人加入 notAvailableLuckyUsers             │   │
│  │          - 从剩余候选中重新选择                                 │   │
│  │          - 如果所有候选都不可用 → 抛出错误                      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                        │
│                              ▼ (首个周期选人完成)                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 阶段 2: 后续周期复用 (isFirstRecurringSlot = false)          │   │
│  │                                                                 │   │
│  │  - 不再重新调用 getLuckyUser()                                 │   │
│  │  - 直接复用首个周期选中的 luckyUsers                           │   │
│  │  - 所有周期使用同一批主持人                                    │   │
│  │                                                                 │   │
│  │  ⚠️ 关键: 后续周期不进行可用性复验                              │   │
│  │     假设: 首个周期可用 → 后续周期也可用                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 4.6.2 首个周期选人逻辑

从 `RecurringBookingService.handleNewRecurringBooking` 方法（`packages/features/bookings/lib/create-recurring-booking.ts:22-132`）：

```typescript
// RecurringBookingService 处理流程
export const handleNewRecurringBooking = async function (...) {
  const data = input.bookingData;  // 所有周期的预订数据
  const isRoundRobin = firstBooking.schedulingType === SchedulingType.ROUND_ROBIN;

  let luckyUsers;

  if (isRoundRobin) {
    // ========== 首个周期单独处理 ==========
    const firstBooking = data[0];
    const recurringEventData = {
      ...firstBooking,
      isFirstRecurringSlot: true,  // 标记是首个周期
      numSlotsToCheckForAvailability,  // 需要检查的后续时隙数量
      currentRecurringIndex: 0,
      // ...
    };

    // 调用 RegularBookingService.createBooking
    // 在 createBooking 内部会进行轮询分配和可用性复验
    const firstBookingResult = await regularBookingService.createBooking({
      bookingData: recurringEventData,
      // ...
    });
    
    // 记录首个周期选中的主持人
    luckyUsers = firstBookingResult.luckyUsers;
  }

  // ========== 后续周期复用 ==========
  for (let key = isRoundRobin ? 1 : 0; key < data.length; key++) {
    const booking = data[key];
    
    const recurringEventData = {
      ...booking,
      isFirstRecurringSlot: key == 0,
      luckyUsers,  // 直接复用首个周期选中的主持人
      currentRecurringIndex: key,
      // ...
    };

    // 后续周期不再重新轮询，直接使用 luckyUsers
    const eachRecurringBooking = await regularBookingService.createBooking({
      bookingData: recurringEventData,
      // ...
    });
    
    createdBookings.push(eachRecurringBooking);
  }

  return createdBookings;
};
```

#### 4.6.3 可用性复验与回退机制

从 `RegularBookingService` 的选人逻辑（`packages/features/bookings/lib/service/RegularBookingService.ts:1049-1087`）：

```typescript
// 在首个周期选人时，会进行可用性复验
if (
  input.bookingData.isFirstRecurringSlot &&
  eventType.schedulingType === SchedulingType.ROUND_ROBIN &&
  input.bookingData.numSlotsToCheckForAvailability &&
  input.bookingData.allRecurringDates
) {
  // ========== 尝试选择幸运用户 ==========
  while (luckyUserPool.length > 0 && !luckUserFound) {
    const freeUsers = luckyUserPool.filter(
      (user) => !luckyUsers.concat(notAvailableLuckyUsers).find((existing) => existing.id === user.id)
    );
    
    if (freeUsers.length === 0) break;  // 没有候选了
    
    // 调用 getLuckyUser 选择用户
    const newLuckyUser = await deps.luckyUserService.getLuckyUser({
      availableUsers: freeUsers,
      // ...
    });

    // ========== 可用性复验 ==========
    try {
      // 检查选中的用户是否在后续 N 个时隙都可用
      for (
        let i = 0;
        i < input.bookingData.allRecurringDates.length &&
        i < input.bookingData.numSlotsToCheckForAvailability;
        i++
      ) {
        const start = input.bookingData.allRecurringDates[i].start;
        const end = input.bookingData.allRecurringDates[i].end;

        if (!skipAvailabilityCheck) {
          // 调用 ensureAvailableUsers 检查可用性
          await ensureAvailableUsers(
            { ...eventTypeWithUsers, users: [newLuckyUser] },
            {
              dateFrom: dayjs(start).tz(reqBody.timeZone).format(),
              dateTo: dayjs(end).tz(reqBody.timeZone).format(),
              // ...
            },
            tracingLogger,
            calendarFetchMode
          );
        }
      }
      
      // ========== 复验通过 ==========
      luckyUsers.push(newLuckyUser);
      luckUserFound = true;
      
    } catch {
      // ========== 复验失败: 回退 ==========
      // 将当前用户加入不可用列表
      notAvailableLuckyUsers.push(newLuckyUser);
      
      // 日志记录
      tracingLogger.info(
        `Round robin host ${newLuckyUser.name} not available for first two slots. Trying to find another host.`
      );
      
      // 继续循环，尝试下一个候选
    }
  }
}
```

#### 4.6.4 回退失败路径

当所有候选主持人都无法通过可用性复验时，会抛出错误：

```typescript
// RegularBookingService 中的错误处理
// 在尝试了所有候选用户之后

// 检查是否每个分组都找到了幸运用户
if (
  [...qualifiedRRUsers, ...additionalFallbackRRUsers].length > 0 &&
  luckyUsers.length !== (Object.keys(nonEmptyHostGroups).length || 1)
) {
  // 抛出错误: 轮询主持人不可用
  throw new Error(ErrorCode.RoundRobinHostsUnavailableForBooking);
}
```

**错误码定义**（`ErrorCode.NoAvailableUsersFound` vs `ErrorCode.RoundRobinHostsUnavailableForBooking`）：

| 错误场景 | 错误码 | 含义 |
|---------|-------|------|
| 单个预订无可用用户 | `NoAvailableUsersFound` | 当前时隙没有可用主持人 |
| 连续预订复验失败 | `RoundRobinHostsUnavailableForBooking` | 无法找到在所有需要检查的时隙都可用的主持人 |

#### 4.6.5 连续预订的公平性考虑

**设计权衡**：

| 考量因素 | 当前设计 | 权衡说明 |
|---------|---------|---------|
| **一致性体验** | 所有周期同一批主持人 | 客户体验一致，不会"这周见 Alice，下周见 Bob" |
| **轮询公平性** | 后续周期不复用轮询 | 连续预订作为"一个整体"分配，不是多个独立预订 |
| **可用性保障** | 首个周期复验后续 N 个时隙 | 降低后续周期出现不可用的风险 |
| **性能优化** | 后续周期不重新轮询 | 避免多次调用 getLuckyUser 的开销 |

**场景示例**：

```
场景: 客户预订连续 4 周的每周一 10:00
团队: [Alice, Bob, Charlie] (3人)

正常流程:
1. 首个周期 (5月1日):
   - 轮询选择 Alice
   - 复验 5月1日、5月8日、5月15日 (假设 numSlotsToCheckForAvailability=3)
   - Alice 在这 3 天都可用 → 确认选中
   
2. 后续周期 (5月8日、5月15日、5月22日):
   - 直接复用 Alice
   - 不重新轮询
   - 不进行可用性复验

结果: 4 个周期都分配给 Alice
```

**异常场景**：

```
场景: 客户预订连续 4 周
团队: [Alice, Bob] (2人)

异常流程:
1. 首个周期选择 Alice
2. 复验发现 Alice 在 5月8日 不可用 → 回退
3. 尝试选择 Bob
4. 复验发现 Bob 在 5月15日 不可用 → 回退
5. 所有候选都不可用 → 抛出 RoundRobinHostsUnavailableForBooking

结果: 预订失败，提示无法找到合适的主持人
```

**关键理解**：
- 连续预订中的轮询分配只在**首个周期**执行
- 后续周期**复用**首个周期的选择结果
- 首个周期会**复验**后续 N 个时隙的可用性（可选）
- 如果复验失败，会**回退**并尝试其他候选人
- 如果所有候选人都无法通过复验，**预订失败**

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
