# 访客表单路由逻辑分析报告

## 概述

本文档分析了 Cal.diy 系统中访客填表后如何决定跳转到哪个预约入口的逻辑。通过代码分析，发现 **Routing Forms 功能已在 2026 年 3 月被完全移除**，当前系统使用简化的路由参数机制来控制预约入口。

---

## 关键发现：Routing Forms 功能已被移除

### 迁移证据

根据数据库迁移文件 `packages/prisma/migrations/20260305043434_remove_routing_forms/migration.sql`，以下内容已被删除：

**删除的表：**
- `App_RoutingForms_Form` - 路由表单定义表
- `App_RoutingForms_FormResponse` - 表单响应表
- `App_RoutingForms_IncompleteBookingActions` - 未完成预订操作表
- `App_RoutingForms_QueuedFormResponse` - 排队表单响应表
- `PendingRoutingTrace` - 待处理路由追踪表
- `RoutingFormResponseDenormalized` - 非规范化表单响应表
- `RoutingFormResponseField` - 表单响应字段表
- `RoutingTrace` - 路由追踪表
- `WorkflowsOnRoutingForms` - 路由表单工作流关联表

**删除的枚举值：**
- `AssignmentReasonEnum`: `ROUTING_FORM_ROUTING`, `ROUTING_FORM_ROUTING_FALLBACK`
- `WebhookTriggerEvents`: `ROUTING_FORM_FALLBACK_HIT`
- `WorkflowType`: `ROUTING_FORM`

**删除的函数和触发器：**
- 约 20+ 个相关数据库函数和触发器

### 代码中的注释证据

在 `packages/features/users/lib/getRoutedUsers.ts:165-180` 中有明确注释：

```typescript
// Routing forms feature removed - segment matching always returns all hosts
export async function findMatchingHostsWithEventSegment<User extends BaseUser>({
  eventType,
  hosts,
}: {
  eventType: EventType;
  hosts: {...};
}) {
  return hosts;  // 直接返回所有 hosts，不再进行条件匹配
}
```

---

## 当前的路由逻辑

### 路由参数来源

当前系统通过以下参数控制预约入口的路由：

| 参数 | 来源 | 说明 |
|------|------|------|
| `routedTeamMemberIds` | URL 查询参数 `cal.routedTeamMemberIds` | 指定可预约的团队成员 ID 列表 |
| `contactOwnerEmail` | API 输入或外部系统（如 Salesforce） | 联系人所有者邮箱 |
| `isFixed` | EventType hosts 配置 | 固定主持人标志 |

**参数解析位置：**
- `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts`: 从 URL 查询参数解析

```typescript
export const getRoutedTeamMemberIdsFromSearchParams = (searchParams: URLSearchParams) => {
  const routedTeamMemberIdsParam = searchParams.get("cal.routedTeamMemberIds");
  const routedTeamMemberIds =
    typeof routedTeamMemberIdsParam === "string"
      ? routedTeamMemberIdsParam
          .split(",")
          .filter(Boolean)
          .map((id) => parseInt(id, 10))
      : null;
  return routedTeamMemberIds;
};
```

---

### 用户过滤核心逻辑

核心过滤函数位于 `packages/features/users/lib/getRoutedUsers.ts:12-36`：

```typescript
export const getRoutedUsersWithContactOwnerAndFixedUsers = <
  T extends { id: number; isFixed?: boolean; email: string },
>({
  routedTeamMemberIds,
  users,
  contactOwnerEmail,
}: {
  routedTeamMemberIds: number[] | null;
  users: T[];
  contactOwnerEmail: string | null;
}) => {
  // 回退机制：如果没有指定路由成员，返回所有用户
  if (!routedTeamMemberIds || !routedTeamMemberIds.length) {
    return users;
  }

  // 条件匹配逻辑
  return users.filter(
    (user) => 
      routedTeamMemberIds.includes(user.id) ||  // 条件 1：在路由列表中
      user.isFixed ||                            // 条件 2：是固定主持人
      user.email === contactOwnerEmail           // 条件 3：是联系人所有者
  );
};
```

### Qualified Hosts 服务逻辑

在 `packages/features/di/modules/QualifiedHosts.ts` 中实现了更详细的主机过滤：

```typescript
// Filter qualifiedRRHosts based on contactOwnerEmail or routedTeamMemberIds
let qualifiedRRHosts = allRRHosts;

if (contactOwnerEmail) {
  // 优先级 1：联系人所有者
  const contactOwnerHost = allRRHosts.filter(
    (h) => (h.user as { email?: string }).email === contactOwnerEmail
  );
  if (contactOwnerHost.length > 0) {
    qualifiedRRHosts = contactOwnerHost;
  }
} else if (routedTeamMemberIds.length > 0) {
  // 优先级 2：路由成员 ID 列表
  const routedMemberIdSet = new Set(routedTeamMemberIds);
  const routedHosts = allRRHosts.filter((h) => routedMemberIdSet.has((h.user as { id: number }).id));
  if (routedHosts.length > 0) {
    qualifiedRRHosts = routedHosts;
  }
}
```

---

## 分支路径分析

### 正常流程

```
访客访问预约页面
    │
    ▼
检查是否有 routedTeamMemberIds 参数
    │
    ├── 有参数 ──► 过滤符合条件的用户
    │               │
    │               ├── 用户在 routedTeamMemberIds 中 ──► 包含
    │               ├── 用户是固定主持人 (isFixed=true) ──► 包含
    │               └── 用户是联系人所有者 ──► 包含
    │
    └── 无参数 ──► 返回所有可用用户
```

### 回退机制

#### 1. 空路由参数回退

**位置：** `packages/features/users/lib/getRoutedUsers.ts:25-27`

```typescript
// We don't want to enter a scenario where we have no team members to be booked
// So, let's just fallback to regular flow if no routedTeamMemberIds are provided
if (!routedTeamMemberIds || !routedTeamMemberIds.length) {
  return users;
}
```

**逻辑：** 如果 `routedTeamMemberIds` 为空或 null，直接返回所有用户，避免"无可预约成员"的情况。

#### 2. 两周内无可用时隙回退

**位置：** `packages/trpc/server/routers/viewer/slots/util.ts:1001-1089`

```typescript
const hasFallbackRRHosts =
  eligibleFallbackRRHosts.length > 0 && eligibleFallbackRRHosts.length > eligibleQualifiedRRHosts.length;

// 检查合格 RR 主持人在两周内是否有可用时隙
if (diff > 0) {
  // 如果第一可用时隙超过两周，使用回退主持人
  ({ allUsersAvailability, usersWithCredentials, currentSeats } =
    await this.calculateHostsAndAvailabilities({
      input,
      eventType,
      hosts: [...eligibleFallbackRRHosts, ...eligibleFixedHosts],  // 使用回退主持人
      ...
    }));
}
```

**逻辑：** 如果合格的轮询主持人在两周内没有可用时隙，自动回退到所有轮询主持人。

#### 3. 无合格用户时的最终回退

**位置：** `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:223-238`

```typescript
if (!qualifiedRRUsers.length && !fixedUsers.length) {
  // 如果没有合格的 RR 用户且没有固定用户，使用所有用户作为固定用户
  const firstUser = users[0];
  const firstUserOrgId = await getOrgIdFromMemberOrTeamId({...});
  const usersEnrichedWithDelegationCredential = await enrichUsersWithDelegationCredentials({
    orgId: firstUserOrgId ?? null,
    users,
  });
  return {
    qualifiedRRUsers,
    additionalFallbackRRUsers,
    fixedUsers: usersEnrichedWithDelegationCredential,  // 所有用户作为固定用户
  };
}
```

---

## 异常情况处理

### 1. 所有用户被阻止（Watchlist）

**位置：** `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:141-151`

```typescript
const { eligibleUsers, blockedCount } = await filterBlockedUsers(users, organizationId, sentrySpan);

if (blockedCount > 0) {
  logger.info(`Filtered out ${blockedCount} blocked user(s) from booking`);
}

// If all users are blocked, throw 404
// For team events with some eligible users, continue with graceful degradation
if (eligibleUsers.length === 0) {
  throw new HttpError({ statusCode: 404, message: "eventTypeUser.notFound" });
}
```

**处理逻辑：**
- 过滤被 Watchlist 阻止的用户
- 如果所有用户都被阻止，抛出 404 错误
- 如果部分用户可用，优雅降级继续处理

### 2. 动态预订不允许

**位置：** `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:98-107`

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

**处理逻辑：**
- 检查所有用户是否允许动态预订
- 如果有用户不允许且无事件类型 ID，抛出 400 错误

### 3. 事件类型用户未找到

**位置：** `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:111-128`

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

**处理逻辑：**
- 尝试从旧数据结构中恢复用户
- 如果仍未找到，抛出 404 错误

### 4. 无效的 routedTeamMemberIds

**位置：** `packages/features/users/lib/getRoutedUsers.test.ts` 测试用例

```typescript
it("should return an empty array when neither fixed, nor routedTeamMemberIds, nor contactOwnerEmail match", () => {
  const result = getRoutedUsersWithContactOwnerAndFixedUsers({
    routedTeamMemberIds: [5],  // 不存在的用户 ID
    users: usersWithoutFixedHosts,
    contactOwnerEmail: "nonexistent@example.com",
  });
  expect(result).toEqual([]);  // 返回空数组
});
```

**处理逻辑：**
- 如果 `routedTeamMemberIds` 中的 ID 都不存在，且没有固定主持人和联系人所有者，返回空数组
- 这可能导致后续的 404 错误

### 5. 重新预订时的特殊处理

**位置：** `packages/lib/bookings/routing/utils.ts:16-26`

```typescript
export function shouldIgnoreContactOwner({
  skipContactOwner,
  rescheduleUid,
  routedTeamMemberIds,
}: {
  skipContactOwner: boolean | null;
  rescheduleUid: string | null;
  routedTeamMemberIds: number[] | null;
}) {
  // During rerouting, we don't want to consider salesforce ownership
  // as it could potentially choose a member that isn't part of routedTeamMemberIds
  // and thus ending up with no available timeslots.
  return skipContactOwner || isRerouting({ rescheduleUid, routedTeamMemberIds });
}
```

**处理逻辑：**
- 在重新预订时，忽略联系人所有者
- 避免选择不在 `routedTeamMemberIds` 中的成员，导致无可用时隙

---

## 关键代码位置汇总

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 路由用户过滤核心逻辑 | `packages/features/users/lib/getRoutedUsers.ts` | 12-36 |
| 空参数回退 | `packages/features/users/lib/getRoutedUsers.ts` | 25-27 |
| Segment 匹配（已禁用） | `packages/features/users/lib/getRoutedUsers.ts` | 165-180 |
| Qualified Hosts 服务 | `packages/features/di/modules/QualifiedHosts.ts` | 15-77 |
| 两周时隙回退 | `packages/trpc/server/routers/viewer/slots/util.ts` | 1001-1089 |
| 用户加载和验证 | `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts` | 71-245 |
| 阻止用户过滤 | `packages/features/watchlist/operations/filter-blocked-users.controller.ts` | - |
| URL 参数解析 | `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts` | 1-11 |
| 迁移删除路由表单 | `packages/prisma/migrations/20260305043434_remove_routing_forms/migration.sql` | 1-222 |

---

## 总结

### 系统现状

1. **Routing Forms 功能已完全移除**：原有的条件匹配引擎、表单响应追踪、路由规则等功能已在 2026 年 3 月删除。

2. **当前使用简化的路由参数机制**：
   - 通过 `cal.routedTeamMemberIds` URL 参数指定可预约成员
   - 通过 `contactOwnerEmail` 支持联系人所有者优先
   - 固定主持人（`isFixed=true`）始终包含

3. **多层回退机制**：
   - 空路由参数 → 返回所有用户
   - 两周内无可用时隙 → 使用回退主持人
   - 无合格用户 → 使用所有用户作为固定用户

4. **异常处理策略**：
   - 所有用户被阻止 → 404 错误
   - 动态预订不允许 → 400 错误
   - 用户未找到 → 404 错误
   - 重新预订 → 忽略联系人所有者

### 对业务的影响

- **简化配置**：不再需要配置复杂的路由表单和条件规则
- **外部控制**：路由决策需要由外部系统（如 CRM、营销自动化平台）通过 URL 参数控制
- **回退友好**：多层回退机制确保预约流程不会因参数问题而中断
- **灵活性降低**：失去了基于表单答案动态路由的能力

### 未来考虑

如果需要恢复基于表单答案的动态路由能力，需要：
1. 重新设计条件匹配引擎
2. 考虑是否需要恢复 Routing Forms 表结构
3. 或者使用更轻量的方案（如基于 EventType metadata 的条件）

---

*报告生成时间：2026-05-03*
*分析版本：基于当前代码库状态*
