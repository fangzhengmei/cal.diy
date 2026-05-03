# 访客表单路由逻辑分析报告

## 文档说明

本文档详细分析了 Cal.diy 系统中访客填表后如何决定跳转到哪个预约入口的完整逻辑，可直接用于**排障定位**。

**阅读指南：**
- 排查路由问题时，按「完整链路流程图」逐步检查
- 排查异常时，参考「异常场景排查表」快速定位
- 排查回退问题时，参考「回退机制详解」中的触发条件

---

## 一、核心现状：Routing Forms 功能已被移除

### 1.1 关键发现

**Routing Forms 功能已于 2026 年 3 月被完全移除。**

原有的基于表单答案的动态条件匹配引擎已不再存在。删除的内容包括：

**已删除的数据库表：**
| 表名 | 说明 |
|------|------|
| `App_RoutingForms_Form` | 路由表单定义 |
| `App_RoutingForms_FormResponse` | 表单响应 |
| `App_RoutingForms_QueuedFormResponse` | 排队表单响应 |
| `RoutingTrace` | 路由追踪 |
| `RoutingFormResponseDenormalized` | 非规范化表单响应 |
| 以及 5 个其他相关表 | |

**已删除的枚举值：**
- `AssignmentReasonEnum`: `ROUTING_FORM_ROUTING`, `ROUTING_FORM_ROUTING_FALLBACK`
- `WebhookTriggerEvents`: `ROUTING_FORM_FALLBACK_HIT`
- `WorkflowType`: `ROUTING_FORM`

**代码证据：** `packages/features/users/lib/getRoutedUsers.ts:165-180`
```typescript
// Routing forms feature removed - segment matching always returns all hosts
export async function findMatchingHostsWithEventSegment<User extends BaseUser>({
  eventType,
  hosts,
}: {...}) {
  return hosts;  // 直接返回所有 hosts，不再进行条件匹配
}
```

### 1.2 当前路由方式

当前系统使用 **简化的 URL 参数机制** 来控制预约入口的路由：

| 参数名 | 示例 | 说明 |
|--------|------|------|
| `cal.routedTeamMemberIds` | `?cal.routedTeamMemberIds=1,2,3` | 逗号分隔的团队成员 ID 列表 |
| `cal.teamMemberEmail` / `teamMemberEmail` | `?cal.teamMemberEmail=owner@example.com` | 联系人所有者邮箱 |
| `cal.skipContactOwner` | `?cal.skipContactOwner=true` | 是否跳过联系人所有者逻辑 |

---

## 二、完整链路流程图

### 2.1 访客答案到预约入口的完整链路

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           访客访问预约页面                                               │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Step 1: 从 URL 解析路由参数                                                          │
│  ────────────────────────────────────────────────────────────────────────────────── │
│  - routedTeamMemberIds: getRoutedTeamMemberIdsFromSearchParams()                    │
│  - teamMemberEmail: 直接从 searchParams 获取                                         │
│  - skipContactOwner: searchParams.get("cal.skipContactOwner")                       │
│                                                                                        │
│  ⚠️ 排障点：检查 URL 中是否有这些参数                                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Step 2: 判断是否重新预订                                                              │
│  ────────────────────────────────────────────────────────────────────────────────── │
│  调用 shouldIgnoreContactOwner({ rescheduleUid, routedTeamMemberIds })               │
│                                                                                        │
│  触发条件：rescheduleUid 存在 且 routedTeamMemberIds 非空                            │
│  ├─ 是 → contactOwnerEmail = null（忽略联系人所有者）                                 │
│  └─ 否 → 保留 contactOwnerEmail                                                       │
│                                                                                        │
│  ⚠️ 排障点：检查 rescheduleUid 参数是否存在                                            │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Step 3: QualifiedHosts 服务 - 优先级过滤                                             │
│  ────────────────────────────────────────────────────────────────────────────────── │
│  输入：eventType.hosts, contactOwnerEmail, routedTeamMemberIds                       │
│                                                                                        │
│  3.1 分离 Fixed Hosts 和 RR Hosts                                                     │
│  ├─ host.isFixed === true → fixedHosts（始终包含）                                   │
│  ├─ schedulingType !== "ROUND_ROBIN" → 所有 hosts 都是 fixedHosts                    │
│  └─ 其他 → allRRHosts（轮询主持人）                                                   │
│                                                                                        │
│  3.2 条件优先级判断（从高到低）                                                        │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │ 优先级 1: 有 contactOwnerEmail？                                                 │ │
│  │ ├─ 找到 email 匹配的 RR host → qualifiedRRHosts = [contactOwner]               │ │
│  │ └─ 未找到 → qualifiedRRHosts = allRRHosts（隐式回退）                           │ │
│  ├────────────────────────────────────────────────────────────────────────────────┤ │
│  │ 优先级 2: 无 contactOwnerEmail，但有 routedTeamMemberIds？                       │ │
│  │ ├─ 找到 id 匹配的 RR host → qualifiedRRHosts = [matchedHosts]                  │ │
│  │ └─ 未找到 → qualifiedRRHosts = allRRHosts（隐式回退）                           │ │
│  ├────────────────────────────────────────────────────────────────────────────────┤ │
│  │ 优先级 3: 无任何参数                                                             │ │
│  │ └─ qualifiedRRHosts = allRRHosts（默认）                                        │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                        │
│  输出：{ qualifiedRRHosts, allFallbackRRHosts: allRRHosts, fixedHosts }             │
│                                                                                        │
│  ⚠️ 排障点：检查 contactOwnerEmail 和 routedTeamMemberIds 的值                        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Step 4: Watchlist 阻止过滤                                                           │
│  ────────────────────────────────────────────────────────────────────────────────── │
│  调用 filterBlockedHosts(hosts, organizationId)                                      │
│                                                                                        │
│  过滤逻辑：                                                                            │
│  - 检查用户是否在 Watchlist 中（email 匹配）                                          │
│  - 检查用户是否被锁定                                                                  │
│                                                                                        │
│  输出：                                                                                │
│  ├─ eligibleQualifiedRRHosts: 合格且未被阻止的 RR 主持人                              │
│  ├─ eligibleFixedHosts: 固定且未被阻止的主持人                                        │
│  └─ eligibleFallbackRRHosts: 所有 RR 主持人（用于回退）                               │
│                                                                                        │
│  ⚠️ 排障点：检查 blockedCount 日志（如果有用户被阻止）                                 │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Step 5: 检查所有主持人是否被阻止                                                      │
│  ────────────────────────────────────────────────────────────────────────────────── │
│  allHosts = [...eligibleQualifiedRRHosts, ...eligibleFixedHosts]                     │
│                                                                                        │
│  触发条件：allHosts.length === 0                                                      │
│  ├─ 是 → 异常兜底 1：返回空时隙 { slots: {} }                                         │
│  └─ 否 → 继续                                                                          │
│                                                                                        │
│  ⚠️ 排障点：检查 Watchlist 配置                                                        │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Step 6: 回退机制 3 - 两周内可用性检查                                                │
│  ────────────────────────────────────────────────────────────────────────────────── │
│  回退前提条件（必须同时满足）：                                                        │
│  1. eligibleFallbackRRHosts.length > 0                                               │
│  2. eligibleFallbackRRHosts.length > eligibleQualifiedRRHosts.length                │
│                                                                                        │
│  ➡️ 不满足 → 直接使用合格主持人计算可用时隙                                            │
│  ➡️ 满足 → 执行可用性检查：                                                           │
│     ┌────────────────────────────────────────────────────────────────────────────┐   │
│     │ 场景 A: 请求的开始时间在两周内                                               │   │
│     │ ├─ 有可用时隙 且 第一时隙 ≤ 两周 → 不触发回退                               │   │
│     │ ├─ 有可用时隙 但 第一时隙 > 两周 → 触发回退（diff > 0）                    │   │
│     │ └─ 无可用时隙 → 触发回退（diff = 1）                                        │   │
│     ├────────────────────────────────────────────────────────────────────────────┤   │
│     │ 场景 B: 请求的开始时间 > 两周                                                │   │
│     │ ├─ 有可用时隙 → 不触发回退                                                  │   │
│     │ └─ 无可用时隙 → 检查前两周是否有可用时隙                                    │   │
│     │      ├─ 有 → 不触发回退                                                     │   │
│     │      └─ 无 → 触发回退（diff = 1）                                          │   │
│     └────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                        │
│  回退行为：                                                                            │
│  hosts = [...eligibleFallbackRRHosts, ...eligibleFixedHosts]（所有主持人）          │
│                                                                                        │
│  ⚠️ 排障点：检查日志中的 fallBackActive 字段                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│  Step 7: 计算可用时隙并返回                                                           │
│  ────────────────────────────────────────────────────────────────────────────────── │
│  调用 getSlots() 计算最终可用时隙                                                      │
│                                                                                        │
│  ⚠️ 排障点：检查返回的 slots 数量                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 排障检查清单

| 步骤 | 检查项 | 预期值 | 异常值说明 |
|------|--------|--------|-----------|
| 1 | URL 参数 | `cal.routedTeamMemberIds` 或 `cal.teamMemberEmail` | 无参数 → 使用所有主持人 |
| 2 | 重新预订判断 | `rescheduleUid` 不存在 | 存在则忽略 `contactOwnerEmail` |
| 3 | QualifiedHosts 过滤 | `qualifiedRRHosts` 不为空 | 为空则触发隐式回退 |
| 4 | Watchlist 过滤 | `blockedCount === 0` | 大于 0 说明有用户被阻止 |
| 5 | 主持人可用性 | `allHosts.length > 0` | 为 0 则返回空时隙 |
| 6 | 两周可用性 | `fallBackActive === false` | 为 true 说明触发了回退 |
| 7 | 最终时隙 | `slots` 不为空 | 为空说明无可预约时间 |

---

## 三、回退机制详解

### 3.1 回退机制总览

系统设计了 **4 层回退机制**，确保预约流程不会因参数问题而中断：

| 回退类型 | 优先级 | 触发时机 | 代码位置 |
|---------|--------|---------|----------|
| **回退 0：隐式回退** | 最高 | 条件匹配未命中时 | `QualifiedHosts.ts:45-63` |
| **回退 1：空参数回退** | 高 | 无路由参数时 | `getRoutedUsers.ts:25-27` |
| **回退 2：无合格用户回退** | 中 | 无合格 RR 用户且无固定用户时 | `loadAndValidateUsers.ts:223-238` |
| **回退 3：两周无可用时隙回退** | 低 | 合格主持人两周内无可用时隙 | `slots/util.ts:1001-1089` |
| **回退 4：可用性检查失败回退** | 最低 | 确保可用性检查失败时 | `RegularBookingService.ts:950-987` |

### 3.2 回退 0：隐式回退（条件匹配未命中）

**这是最容易被忽略的回退机制！**

#### 触发条件

在 `QualifiedHosts` 服务中，条件匹配未命中时会自动回退到所有 RR 主持人。

**代码位置：** `packages/features/di/modules/QualifiedHosts.ts:45-63`

```typescript
let qualifiedRRHosts = allRRHosts;  // 默认值：所有 RR 主持人

if (contactOwnerEmail) {
  const contactOwnerHost = allRRHosts.filter(
    (h) => (h.user as { email?: string }).email === contactOwnerEmail
  );
  if (contactOwnerHost.length > 0) {
    qualifiedRRHosts = contactOwnerHost;
  }
  // ⚠️ 关键：如果 contactOwnerHost.length === 0
  // qualifiedRRHosts 保持为 allRRHosts（隐式回退）
} else if (routedTeamMemberIds.length > 0) {
  const routedMemberIdSet = new Set(routedTeamMemberIds);
  const routedHosts = allRRHosts.filter((h) => routedMemberIdSet.has((h.user as { id: number }).id));
  if (routedHosts.length > 0) {
    qualifiedRRHosts = routedHosts;
  }
  // ⚠️ 关键：如果 routedHosts.length === 0
  // qualifiedRRHosts 保持为 allRRHosts（隐式回退）
}
```

#### 完整触发场景

| 场景 | 输入参数 | 预期行为 | 实际行为（隐式回退） |
|------|----------|---------|---------------------|
| 场景 1 | `contactOwnerEmail=nonexistent@example.com` | `qualifiedRRHosts = []` | `qualifiedRRHosts = allRRHosts` |
| 场景 2 | `routedTeamMemberIds=[999]`（不存在的 ID） | `qualifiedRRHosts = []` | `qualifiedRRHosts = allRRHosts` |
| 场景 3 | `contactOwnerEmail=owner@example.com`（存在） | `qualifiedRRHosts = [owner]` | `qualifiedRRHosts = [owner]`（正常） |

#### 排障要点

**隐式回退是静默的，没有日志记录。** 排查时需要：

1. 检查 `contactOwnerEmail` 对应的用户是否存在于 `eventType.hosts` 中
2. 检查 `routedTeamMemberIds` 对应的用户 ID 是否存在于 `eventType.hosts` 中
3. 注意：`contactOwnerEmail` 匹配的是 `host.user.email`，不是团队成员的 email

### 3.3 回退 1：空参数回退

#### 触发条件

**代码位置：** `packages/features/users/lib/getRoutedUsers.ts:25-27`

```typescript
if (!routedTeamMemberIds || !routedTeamMemberIds.length) {
  return users;
}
```

**触发条件（满足任一）：**
- `routedTeamMemberIds === null`
- `routedTeamMemberIds.length === 0`

#### 后续行为

直接返回所有用户，不进行任何过滤。

**代码位置：** `packages/features/bookings/lib/handleNewBooking/loadUsers.ts:59-69`

```typescript
const routedUsers = getRoutedUsersWithContactOwnerAndFixedUsers({
  users,
  routedTeamMemberIds,
  contactOwnerEmail,
});

if (routedUsers.length) {
  return routedUsers;
}

return users;  // 回退：返回所有用户
```

### 3.4 回退 2：无合格用户回退

#### 触发条件

**代码位置：** `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:223-238`

```typescript
if (!qualifiedRRUsers.length && !fixedUsers.length) {
  // 回退逻辑
}
```

**触发条件（必须同时满足）：**
- `qualifiedRRUsers.length === 0`（无合格的 RR 用户）
- `fixedUsers.length === 0`（无固定用户）

#### 后续行为

将所有用户作为固定用户返回。

```typescript
const firstUser = users[0];
const firstUserOrgId = await getOrgIdFromMemberOrTeamId({...});
const usersEnrichedWithDelegationCredential = await enrichUsersWithDelegationCredentials({
  orgId: firstUserOrgId ?? null,
  users,
});
return {
  qualifiedRRUsers,        // 空数组
  additionalFallbackRRUsers,
  fixedUsers: usersEnrichedWithDelegationCredential,  // 所有用户作为固定用户
};
```

### 3.5 回退 3：两周内无可用时隙回退

这是最复杂的回退机制，涉及可用性检查。

#### 触发前提条件

**代码位置：** `packages/trpc/server/routers/viewer/slots/util.ts:1001-1002`

```typescript
const hasFallbackRRHosts =
  eligibleFallbackRRHosts.length > 0 && eligibleFallbackRRHosts.length > eligibleQualifiedRRHosts.length;
```

**必须同时满足：**
1. `eligibleFallbackRRHosts.length > 0`（有回退主持人可用）
2. `eligibleFallbackRRHosts.length > eligibleQualifiedRRHosts.length`（进行了筛选）

#### 可用性检查逻辑

**代码位置：** `packages/trpc/server/routers/viewer/slots/util.ts:1028-1060`

```typescript
if (hasFallbackRRHosts) {
  let diff = 0;
  
  if (startTime.isBefore(twoWeeksFromNow)) {
    // 场景 A：请求时间在两周内
    diff =
      aggregatedAvailability.length > 0
        ? aggregatedAvailability[0].start.diff(twoWeeksFromNow, "day")
        : 1; // 无可用时隙 → diff = 1
  } else {
    // 场景 B：请求时间 > 两周
    if (!aggregatedAvailability.length) {
      const firstTwoWeeksAvailabilities = await this.calculateHostsAndAvailabilities({
        hosts: [...eligibleQualifiedRRHosts, ...eligibleFixedHosts],
        startTime: dayjs(),
        endTime: twoWeeksFromNow,
        ...
      });
      if (!getAggregatedAvailability(...).length) {
        diff = 1;
      }
    }
  }
}
```

#### 回退触发条件（`diff > 0`）

| 场景 | `diff` 值 | 说明 |
|------|-----------|------|
| 有可用时隙，且第一时隙 ≤ 两周 | `diff ≤ 0` | ❌ 不触发回退 |
| 有可用时隙，但第一时隙 > 两周 | `diff > 0` | ✅ 触发回退 |
| 无可用时隙 | `diff = 1` | ✅ 触发回退 |
| 请求时间 > 两周，且前两周无可用时隙 | `diff = 1` | ✅ 触发回退 |

#### 回退后续行为

**代码位置：** `packages/trpc/server/routers/viewer/slots/util.ts:1073-1088`

```typescript
if (diff > 0) {
  // 使用所有 RR 主持人 + 固定主持人
  ({ allUsersAvailability, usersWithCredentials, currentSeats } =
    await this.calculateHostsAndAvailabilities({
      input,
      eventType,
      hosts: [...eligibleFallbackRRHosts, ...eligibleFixedHosts],  // 回退主持人列表
      startTime,
      endTime,
      ...
    }));
  aggregatedAvailability = getAggregatedAvailability(allUsersAvailability, eventType.schedulingType);
}
```

#### 日志记录

回退触发时会记录日志：

**代码位置：** `packages/trpc/server/routers/viewer/slots/util.ts:1062-1071`

```typescript
if (input.email) {
  loggerWithEventDetails.info({
    email: input.email,
    contactOwnerEmail,
    eligibleQualifiedRRHosts: eligibleQualifiedRRHosts.map((host) => host.user.id),
    eligibleFallbackRRHosts: eligibleFallbackRRHosts.map((host) => host.user.id),
    blockedHostsCount: qualifiedRRHosts.length - eligibleQualifiedRRHosts.length,
    fallBackActive: diff > 0,  // 关键日志字段
  });
}
```

**排障时搜索关键字：** `fallBackActive`

### 3.6 回退 4：可用性检查失败回退

这是预订创建时的最后一层回退。

#### 触发条件

**代码位置：** `packages/features/bookings/lib/service/RegularBookingService.ts:934-987`

```typescript
try {
  if (!skipAvailabilityCheck) {
    availableUsers = await ensureAvailableUsers(
      { ...eventTypeWithUsers, users: [...qualifiedRRUsers, ...fixedUsers] as IsFixedAwareUser[] },
      {...}
    );
  } else {
    availableUsers = [...qualifiedRRUsers, ...fixedUsers] as IsFixedAwareUser[];
  }
} catch {
  if (additionalFallbackRRUsers.length) {
    // 有回退用户，使用回退
  } else {
    // 无回退用户，抛出错误
  }
}
```

**触发条件：**
- `ensureAvailableUsers()` 抛出 `ErrorCode.NoAvailableUsersFound` 错误
- 且 `additionalFallbackRRUsers.length > 0`

#### 后续行为

使用回退用户进行可用性检查：

```typescript
if (additionalFallbackRRUsers.length) {
  tracingLogger.debug(
    "Qualified users not available, check for fallback users",
    safeStringify({
      qualifiedRRUsers: qualifiedRRUsers.map((user) => user.id),
      additionalFallbackRRUsers: additionalFallbackRRUsers.map((user) => user.id),
    })
  );
  
  if (!skipAvailabilityCheck) {
    availableUsers = await ensureAvailableUsers(
      {
        ...eventTypeWithUsers,
        users: [...additionalFallbackRRUsers, ...fixedUsers] as IsFixedAwareUser[],  // 使用回退用户
      },
      {...}
    );
  } else {
    availableUsers = [...additionalFallbackRRUsers, ...fixedUsers] as IsFixedAwareUser[];
  }
} else {
  tracingLogger.debug(
    "Qualified users not available, no fallback users",
    safeStringify({
      qualifiedRRUsers: qualifiedRRUsers.map((user) => user.id),
    })
  );
  throw new Error(ErrorCode.NoAvailableUsersFound);  // 无回退，抛出错误
}
```

---

## 四、异常兜底详解

### 4.1 异常兜底总览

| 异常类型 | 触发条件 | 处理方式 | HTTP 状态码 | 可恢复 |
|---------|---------|---------|-------------|--------|
| **异常 1：所有主持人被阻止（时隙查询）** | Watchlist 阻止了所有合格主持人和固定主持人 | 返回空时隙 | - | 否 |
| **异常 2：所有用户被阻止（预订创建）** | Watchlist 阻止了所有用户 | 抛出 404 错误 | 404 | 否 |
| **异常 3：动态预订不允许** | 有用户不允许动态预订，且无 eventTypeId | 抛出 400 错误 | 400 | 否 |
| **异常 4：用户未找到** | 无法找到事件类型用户 | 抛出 404 错误 | 404 | 否 |
| **异常 5：无可用用户** | 所有用户在请求时间都不可用 | 抛出错误 | - | 否 |
| **异常 6：限制时间表单问题** | 限制时间表单未找到或超出范围 | 抛出错误 | - | 否 |

### 4.2 异常 1：所有主持人被阻止（时隙查询时）

#### 触发条件

**代码位置：** `packages/trpc/server/routers/viewer/slots/util.ts:991-997`

```typescript
const allHosts = [...eligibleQualifiedRRHosts, ...eligibleFixedHosts];

if (allHosts.length === 0) {
  loggerWithEventDetails.info("All hosts are blocked by watchlist, returning empty slots");
  return {
    slots: {},
  };
}
```

**触发条件：**
- `eligibleQualifiedRRHosts.length === 0`（合格 RR 主持人都被阻止）
- `eligibleFixedHosts.length === 0`（固定主持人都被阻止）

#### 后续行为

返回空时隙对象，不抛出错误：

```typescript
return {
  slots: {},
};
```

#### 排障要点

**与异常 2 的区别：**
- 时隙查询时：返回空时隙，不抛出错误
- 预订创建时：抛出 404 错误

**排查方向：**
1. 检查 Watchlist 配置
2. 检查用户是否被锁定
3. 检查日志：`All hosts are blocked by watchlist, returning empty slots`

### 4.3 异常 2：所有用户被阻止（预订创建时）

#### 触发条件

**代码位置：** `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:141-151`

```typescript
const { eligibleUsers, blockedCount } = await filterBlockedUsers(users, organizationId, sentrySpan);

if (blockedCount > 0) {
  logger.info(`Filtered out ${blockedCount} blocked user(s) from booking`);
}

if (eligibleUsers.length === 0) {
  throw new HttpError({ statusCode: 404, message: "eventTypeUser.notFound" });
}
```

**触发条件：** `eligibleUsers.length === 0`

#### 后续行为

抛出 404 错误：

```typescript
throw new HttpError({ statusCode: 404, message: "eventTypeUser.notFound" });
```

#### 排障要点

**日志关键字：** `Filtered out X blocked user(s) from booking`

**排查方向：**
1. 检查 Watchlist 中是否有相关邮箱
2. 检查用户账户状态（是否被锁定）
3. 检查 `organizationId` 是否正确

### 4.4 异常 3：动态预订不允许

#### 触发条件

**代码位置：** `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:98-107`

```typescript
const isDynamicAllowed = !users.some((user) => !user.allowDynamicBooking);
if (!isDynamicAllowed && !eventTypeId) {
  logger.warn({
    message: "NewBooking: Some of the users in this group do not allow dynamic booking",
  });
  throw new HttpError({
    message: "Some of the users in this group do not allow dynamic booking",
    statusCode: 400,
  });
}
```

**触发条件（必须同时满足）：**
- `!isDynamicAllowed`（有用户不允许动态预订）
- `!eventTypeId`（无事件类型 ID，动态预订场景）

#### 后续行为

抛出 400 错误：

```typescript
throw new HttpError({
  message: "Some of the users in this group do not allow dynamic booking",
  statusCode: 400,
});
```

#### 排障要点

**日志关键字：** `Some of the users in this group do not allow dynamic booking`

**排查方向：**
1. 检查用户的 `allowDynamicBooking` 设置
2. 检查是否应该使用 `eventTypeId` 而不是动态预订

### 4.5 异常 4：用户未找到

#### 触发条件

**代码位置：** `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:111-130`

```typescript
// If this event was pre-relationship migration
if (!users.length && eventType.userId) {
  const eventTypeUser = await prisma.user.findUnique({
    where: { id: eventType.userId },
    select: {...},
  });
  if (!eventTypeUser) {
    logger.warn({ message: "NewBooking: eventTypeUser.notFound" });
    throw new HttpError({ statusCode: 404, message: "eventTypeUser.notFound" });
  }
  users.push(withSelectedCalendars(eventTypeUser));
}

if (!users) throw new HttpError({ statusCode: 404, message: "eventTypeUser.notFound" });
```

**触发条件：**
1. `!users.length && eventType.userId` 但无法找到该用户
2. 或 `!users`（用户列表为空）

#### 后续行为

抛出 404 错误：

```typescript
throw new HttpError({ statusCode: 404, message: "eventTypeUser.notFound" });
```

#### 排障要点

**日志关键字：** `eventTypeUser.notFound`

**排查方向：**
1. 检查 `eventType.userId` 对应的用户是否存在
2. 检查 `eventType.hosts` 配置是否正确
3. 检查 `dynamicUserList` 是否正确

### 4.6 异常 5：无可用用户

#### 触发条件

**代码位置：** `packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:256-259`

```typescript
if (availableUsers.length === 0) {
  loggerWithEventDetails.error(`No available users found.`, piiFreeInputDataForLogging);
  throw new Error(ErrorCode.NoAvailableUsersFound);
}
```

**触发条件：** `availableUsers.length === 0`

**导致原因（任一）：**
1. 所有用户在请求时间都有日历冲突
2. 所有用户在请求时间都不在工作时间内
3. 所有用户都被预订限制或时长限制阻止

#### 后续行为

抛出错误：

```typescript
throw new Error(ErrorCode.NoAvailableUsersFound);
```

**测试用例证据：** `packages/features/bookings/lib/handleNewBooking/test/fresh-booking.test.ts`

```typescript
await expect(
  async () => await handleNewBooking({...})
).rejects.toThrowError(ErrorCode.NoAvailableUsersFound);
```

#### 排障要点

**日志关键字：** `No available users found`

**排查方向：**
1. 检查用户日历在请求时间是否有冲突
2. 检查用户工作时间配置
3. 检查预订限制（`bookingLimits`）和时长限制（`durationLimits`）
4. 检查 `beforeEventBuffer` 和 `afterEventBuffer` 设置

### 4.7 异常 6：限制时间表单问题

#### 触发条件 1：限制时间表单未找到

**代码位置：** `packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:165-168`

```typescript
if (!restrictionSchedule) {
  loggerWithEventDetails.error(`Restriction schedule ${eventType.restrictionScheduleId} not found`);
  throw new Error(ErrorCode.RestrictionScheduleNotFound);
}
```

**触发条件：** `eventType.restrictionScheduleId` 存在但对应的 schedule 不存在

#### 触发条件 2：无时区配置

**代码位置：** `packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:174-179`

```typescript
if (!eventType.useBookerTimezone && !restrictionSchedule.timeZone) {
  loggerWithEventDetails.error(
    `No timezone is set for the restriction schedule and useBookerTimezone is false`
  );
  throw new Error(ErrorCode.BookingNotAllowedByRestrictionSchedule);
}
```

**触发条件：**
- `!eventType.useBookerTimezone`（不使用预订者时区）
- 且 `!restrictionSchedule.timeZone`（限制时间表单无时区配置）

#### 触发条件 3：超出限制时间范围

**代码位置：** `packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts:206-212`

```typescript
if (!hasDateRangeForBooking(restrictionRanges, startDateTimeUtc, endDateTimeUtc)) {
  loggerWithEventDetails.error(
    `Booking outside restriction schedule availability.`,
    piiFreeInputDataForLogging
  );
  throw new Error(ErrorCode.BookingNotAllowedByRestrictionSchedule);
}
```

**触发条件：** 请求时间不在限制时间表单的可用范围内

---

## 五、异常场景排查表

### 5.1 快速定位表

| 现象 | 可能原因 | 排查步骤 |
|------|---------|---------|
| **返回空时隙** | 所有主持人被 Watchlist 阻止 | 1. 检查日志 `All hosts are blocked by watchlist`<br>2. 检查 Watchlist 配置 |
| **404 错误：eventTypeUser.notFound** | 所有用户被阻止或用户不存在 | 1. 检查日志 `Filtered out X blocked user(s)`<br>2. 检查 `eventType.hosts` 配置<br>3. 检查 `eventType.userId` 对应的用户 |
| **400 错误：不允许动态预订** | 有用户不允许动态预订 | 1. 检查日志 `do not allow dynamic booking`<br>2. 检查用户 `allowDynamicBooking` 设置 |
| **无可用用户错误** | 所有用户在请求时间不可用 | 1. 检查日志 `No available users found`<br>2. 检查日历冲突<br>3. 检查工作时间<br>4. 检查预订限制 |
| **路由参数不起作用** | 隐式回退触发 | 1. 检查 `contactOwnerEmail` 是否匹配 `host.user.email`<br>2. 检查 `routedTeamMemberIds` 是否匹配 `host.user.id`<br>3. 注意：条件未命中时静默回退 |
| **联系人所有者不生效** | 重新预订场景 | 1. 检查是否有 `rescheduleUid` 参数<br>2. 检查 `shouldIgnoreContactOwner` 返回值 |
| **两周后才能预约** | 回退机制 3 触发 | 1. 检查日志 `fallBackActive: true`<br>2. 检查合格主持人的可用性 |

### 5.2 日志关键字对照表

| 问题 | 日志关键字 | 日志级别 |
|------|-----------|---------|
| 用户被阻止 | `Filtered out X blocked user(s)` | INFO |
| 所有主持人被阻止（时隙查询） | `All hosts are blocked by watchlist, returning empty slots` | INFO |
| 回退触发 | `fallBackActive: true` | INFO |
| 回退用户检查 | `Qualified users not available, check for fallback users` | DEBUG |
| 无回退用户 | `Qualified users not available, no fallback users` | DEBUG |
| 无可用用户 | `No available users found` | ERROR |
| 限制时间表单未找到 | `Restriction schedule X not found` | ERROR |
| 超出限制时间 | `Booking outside restriction schedule availability` | ERROR |
| 动态预订不允许 | `do not allow dynamic booking` | WARN |
| 用户未找到 | `eventTypeUser.notFound` | WARN |

---

## 六、关键代码位置速查

### 6.1 核心逻辑

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| QualifiedHosts 服务（优先级过滤） | `packages/features/di/modules/QualifiedHosts.ts` | 15-77 |
| getRoutedUsers（OR 逻辑过滤） | `packages/features/users/lib/getRoutedUsers.ts` | 12-36 |
| 两周可用性回退 | `packages/trpc/server/routers/viewer/slots/util.ts` | 1001-1089 |
| 可用性检查 | `packages/features/bookings/lib/handleNewBooking/ensureAvailableUsers.ts` | 57-265 |

### 6.2 异常处理

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| Watchlist 过滤 | `packages/features/watchlist/operations/filter-blocked-users.controller.ts` | 29-56 |
| 所有用户被阻止（预订创建） | `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts` | 141-151 |
| 所有主持人被阻止（时隙查询） | `packages/trpc/server/routers/viewer/slots/util.ts` | 991-997 |
| 动态预订不允许 | `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts` | 98-107 |
| 用户未找到 | `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts` | 111-130 |

### 6.3 参数解析

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 从 URL 解析 routedTeamMemberIds | `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts` | 1-11 |
| 重新预订判断 | `packages/lib/bookings/routing/utils.ts` | 6-26 |

---

## 七、排障案例

### 案例 1：路由参数不起作用

**现象：**
- URL 中设置了 `?cal.routedTeamMemberIds=5`
- 但预约时仍然可以选择所有团队成员

**排查步骤：**

1. **检查 QualifiedHosts 服务的输入**
   - `routedTeamMemberIds = [5]`
   - `eventType.hosts = [{ user: { id: 1 }, isFixed: false }, { user: { id: 2 }, isFixed: false }]`

2. **分析隐式回退触发**
   - `routedMemberIdSet = new Set([5])`
   - `routedHosts = allRRHosts.filter(h => routedMemberIdSet.has(h.user.id))`
   - `routedHosts.length = 0`（没有匹配）
   - 由于 `routedHosts.length === 0`，不修改 `qualifiedRRHosts`
   - `qualifiedRRHosts` 保持为 `allRRHosts`（所有主持人）

3. **结论**
   - `routedTeamMemberIds=5` 中的用户 ID 5 不在 `eventType.hosts` 中
   - 触发了**隐式回退**，所以所有主持人都可用

**解决方案：**
- 检查 `routedTeamMemberIds` 中的 ID 是否存在于 `eventType.hosts` 中
- 或者使用 `contactOwnerEmail` 方式

---

### 案例 2：联系人所有者不生效

**现象：**
- URL 中设置了 `?cal.teamMemberEmail=owner@example.com&cal.rescheduleUid=abc123`
- 但联系人所有者路由不生效

**排查步骤：**

1. **检查重新预订判断**
   - `rescheduleUid = "abc123"`（存在）
   - `routedTeamMemberIds` 可能有值
   - 调用 `shouldIgnoreContactOwner()`

2. **分析结果**
   ```typescript
   export function shouldIgnoreContactOwner({
     skipContactOwner,
     rescheduleUid,
     routedTeamMemberIds,
   }) {
     return skipContactOwner || isRerouting({ rescheduleUid, routedTeamMemberIds });
   }
   
   export function isRerouting({ rescheduleUid, routedTeamMemberIds }) {
     return !!rescheduleUid && !!routedTeamMemberIds?.length;
   }
   ```
   - `rescheduleUid` 存在 且 `routedTeamMemberIds` 非空
   - 返回 `true`，忽略 `contactOwnerEmail`

3. **结论**
   - 由于是重新预订场景，系统忽略了联系人所有者

**解决方案：**
- 如果需要在重新预订时使用联系人所有者，需要特殊处理
- 或者确保 `routedTeamMemberIds` 包含正确的成员

---

### 案例 3：所有用户都被阻止

**现象：**
- 预约时返回 404 错误 `eventTypeUser.notFound`

**排查步骤：**

1. **检查日志**
   - 搜索 `Filtered out X blocked user(s) from booking`
   - 假设日志显示 `Filtered out 3 blocked user(s) from booking`

2. **分析 Watchlist 过滤**
   ```typescript
   const { eligibleUsers, blockedCount } = await filterBlockedUsers(users, organizationId, sentrySpan);
   
   if (eligibleUsers.length === 0) {
     throw new HttpError({ statusCode: 404, message: "eventTypeUser.notFound" });
   }
   ```
   - `blockedCount = 3`
   - `eligibleUsers.length = 0`

3. **结论**
   - 所有 3 个用户都被 Watchlist 阻止

**解决方案：**
- 检查 Watchlist 配置，移除相关邮箱
- 检查用户账户状态，解锁被锁定的用户

---

## 八、附录

### 8.1 错误码定义

**代码位置：** `packages/lib/errorCodes.ts`（推测，需确认）

```typescript
export enum ErrorCode {
  NoAvailableUsersFound = "NoAvailableUsersFound",
  RestrictionScheduleNotFound = "RestrictionScheduleNotFound",
  BookingNotAllowedByRestrictionSchedule = "BookingNotAllowedByRestrictionSchedule",
  // ... 其他错误码
}
```

### 8.2 关键概念说明

| 概念 | 定义 |
|------|------|
| **Fixed Hosts（固定主持人）** | 始终包含在可用主持人列表中，不受条件筛选影响 |
| **RR Hosts（轮询主持人）** | 团队事件中配置为轮询分配的主持人，受条件筛选影响 |
| **Qualified RR Hosts（合格 RR 主持人）** | 经过条件筛选后的 RR 主持人 |
| **Fallback RR Hosts（回退 RR 主持人）** | 所有可用的 RR 主持人，用于回退场景 |
| **Contact Owner（联系人所有者）** | 来自 CRM 系统的联系人归属，优先级最高 |
| **隐式回退** | 条件匹配未命中时，自动回退到所有 RR 主持人（静默，无日志） |

---

*报告生成时间：2026-05-03*
*分析版本：基于当前代码库状态*
*适用场景：排障定位、路由问题分析*
