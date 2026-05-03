# 访客表单路由逻辑分析报告

## 概述

本文档详细分析了 Cal.diy 系统中访客填表后如何决定跳转到哪个预约入口的完整逻辑。

**重要发现：**：Routing Forms 功能已于 2026 年 3 月被完全移除。原有的基于表单答案的动态条件匹配引擎已不再存在。当前系统使用简化的 URL 参数机制来控制预约入口的路由。

---

## 一、路由参数来源与传递链

### 1.1 URL 查询参数

访客访问预约页面时，路由参数通过以下 URL 查询参数传递：

| 参数名 | 来源 | 说明 |
|---------|------|------|
| `cal.routedTeamMemberIds` | 外部系统/URL 直接传入 | 逗号分隔的团队成员 ID 列表，如 `?cal.routedTeamMemberIds=1,2,3` |
| `cal.teamMemberEmail` / `teamMemberEmail` | 外部系统/URL 直接传入 | 联系人所有者邮箱，如 `?cal.teamMemberEmail=owner@example.com` |
| `cal.skipContactOwner` | URL 直接传入 | 是否跳过联系人所有者逻辑，`true`/`false` |
| `cal.queuedFormResponseId` | URL 直接传入 | 排队的表单响应 ID（相关表已删除，保留参数） |
| `email` | URL 直接传入 | 访客邮箱，用于日志记录 |

**参数解析位置：**
- **前端解析：`apps/web/modules/schedules/hooks/useSchedule.ts:77-80`

```typescript
const routedTeamMemberIds = searchParams
  ? getRoutedTeamMemberIdsFromSearchParams(new URLSearchParams(searchParams.toString()))
  : null;
const skipContactOwner = searchParams ? searchParams.get("cal.skipContactOwner") === "true" : false;
```

- **参数解析函数：** `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts:1-11`

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

### 1.2 参数传递到后端

前端获取参数后，通过 tRPC 调用传递到后端：

**位置：** `apps/web/modules/schedules/hooks/useSchedule.ts:88-114`

```typescript
const input = {
  isTeamEvent,
  usernameList: getUsernameList(username ?? ""),
  ...(eventSlug ? { eventTypeSlug: eventSlug } : { eventTypeId: eventId ?? 0 }),
  startTime,
  endTime,
  timeZone: timezone ?? "PLACEHOLDER_TIMEZONE",
  duration: duration ? `${duration}` : undefined,
  rescheduleUid,
  orgSlug,
  teamMemberEmail,           // 联系人所有者邮箱
  routedTeamMemberIds,       // 路由成员 ID 列表
  skipContactOwner,           // 是否跳过联系人所有者
  ...(queuedFormResponseId ? { queuedFormResponseId } : {}),
  email,
  ...
};
```

### 1.3 后端 Schema 定义

**位置：** `packages/trpc/server/routers/viewer/slots/types.ts:32-35`

```typescript
teamMemberEmail: z.string().nullish(),
routedTeamMemberIds: z.array(z.number()).nullish(),
skipContactOwner: z.boolean().nullish(),
rrHostSubsetIds: z.array(z.number()).nullish(),
queuedFormResponseId: z.string().nullish(),
```

---

## 二、条件优先级与分支触发顺序

### 2.1 核心概念说明

在理解条件匹配之前，需要明确几个核心概念：

| 概念 | 定义 | 来源 |
|------|------|------|
| **Round-Robin (RR) Hosts** | 轮询主持人，团队事件中配置为轮询分配的主持人 | `EventType.hosts` 配置 |
| **Fixed Hosts** | 固定主持人，始终包含在可用主持人列表中 | `host.isFixed === true` 或 `schedulingType !== ROUND_ROBIN` |
| **Qualified RR Hosts** | 合格的轮询主持人，经过条件筛选后的 RR 主持人 | 由 `contactOwnerEmail` 或 `routedTeamMemberIds` 筛选 |
| **Fallback RR Hosts** | 回退轮询主持人，所有可用的 RR 主持人（用于回退场景） | 所有 `allRRHosts` |
| **Contact Owner** | 联系人所有者，通常来自 CRM 系统（如 Salesforce）的联系人归属 | `teamMemberEmail` 参数 |

### 2.2 条件优先级（从高到低）

**核心逻辑位置：** `packages/features/di/modules/QualifiedHosts.ts:45-63`

```typescript
// Filter qualifiedRRHosts based on contactOwnerEmail or routedTeamMemberIds
let qualifiedRRHosts = allRRHosts;  // 默认：所有 RR 主持人

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
// 优先级 3：默认使用所有 RR 主持人（qualifiedRRHosts = allRRHosts）

return { qualifiedRRHosts, allFallbackRRHosts: allRRHosts, fixedHosts };
```

### 2.3 完整的条件匹配流程图

```
访客访问预约页面
    │
    ▼
检查是否有路由参数
    │
    ├── 有 contactOwnerEmail 参数
    │       │
    │       ▼
    │   查找 email === contactOwnerEmail 的 RR 主持人
    │       │
    │       ├── 找到 ──► qualifiedRRHosts = [联系人所有者]
    │       │
    │       └── 未找到 ──► qualifiedRRHosts = 所有 RR 主持人（保持默认）
    │
    └── 无 contactOwnerEmail，但有 routedTeamMemberIds 参数
            │
            ▼
        查找 id 在 routedTeamMemberIds 中的 RR 主持人
            │
            ├── 找到 ──► qualifiedRRHosts = [匹配的路由成员]
            │
            └── 未找到 ──► qualifiedRRHosts = 所有 RR 主持人（保持默认）
```

### 2.4 Fixed Hosts 的特殊处理

Fixed Hosts（固定主持人）始终包含在可用主持人列表中，**不受条件筛选影响。

**位置：** `packages/features/di/modules/QualifiedHosts.ts:27-43`

```typescript
for (const host of hosts) {
  const qualifiedHost: QualifiedHost = {
    user: host.user,
    isFixed: host.isFixed,
    groupId: host.groupId ?? null,
  };

  if (host.isFixed || schedulingType !== "ROUND_ROBIN") {
    // 固定主持人 或 非轮询事件（集体事件）→ 加入 fixedHosts
    fixedHosts.push(qualifiedHost);
  } else {
    // 轮询主持人 → 加入 allRRHosts
    allRRHosts.push(qualifiedHost);
  }
}
```

**关键规则：**
1. `host.isFixed === true` → 固定主持人
2. `schedulingType !== "ROUND_ROBIN"` → 所有主持人都是固定的（如 COLLECTIVE 集体事件）
3. Fixed Hosts 不参与 `contactOwnerEmail` 和 `routedTeamMemberIds` 的筛选
4. Fixed Hosts 始终出现在最终的可用主持人列表中

### 2.5 用户过滤的 OR 逻辑

在 `getRoutedUsersWithContactOwnerAndFixedUsers` 函数中使用 OR 逻辑进行用户过滤：

**位置：** `packages/features/users/lib/getRoutedUsers.ts:12-36`

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
  // 回退机制：如果没有路由参数，返回所有用户
  if (!routedTeamMemberIds || !routedTeamMemberIds.length) {
    return users;
  }

  // OR 逻辑：满足任一条件即可
  return users.filter(
    (user) => 
      routedTeamMemberIds.includes(user.id) ||  // 条件 1：在路由列表中
      user.isFixed ||                            // 条件 2：是固定主持人
      user.email === contactOwnerEmail           // 条件 3：是联系人所有者
  );
};
```

**注意：** 这个函数与 `QualifiedHosts` 服务的逻辑略有不同：
- `QualifiedHosts` 服务：优先级逻辑（contactOwnerEmail 优先于 routedTeamMemberIds）
- `getRoutedUsersWithContactOwnerAndFixedUsers`：OR 逻辑（三个条件任一满足即可

---

## 三、回退机制详解

### 3.1 回退机制总览

系统设计了多层回退机制，确保预约流程不会因参数问题而中断：

| 回退类型 | 触发条件 | 回退行为 | 位置 |
|---------|---------|---------|------|
| **回退 1：空路由参数回退 | `routedTeamMemberIds` 为空或 null | 返回所有用户 | `getRoutedUsers.ts:25-27` |
| **回退 2：条件未命中回退** | `contactOwnerEmail` 或 `routedTeamMemberIds` 未匹配到任何 RR 主持人 | `qualifiedRRHosts` 保持为所有 RR 主持人 | `QualifiedHosts.ts:45-63` |
| **回退 3：两周内无可用时隙回退 | 合格主持人两周内无可用时隙 | 使用所有 RR 主持人（fallback） | `slots/util.ts:1001-1089` |
| **回退 4：无合格用户回退** | 无合格 RR 用户且无固定用户 | 使用所有用户作为固定用户 | `loadAndValidateUsers.ts:223-238` |

### 3.2 回退 1：空路由参数回退

**位置：** `packages/features/users/lib/getRoutedUsers.ts:25-27`

```typescript
// We don't want to enter a scenario where we have no team members to be booked
// So, let's just fallback to regular flow if no routedTeamMemberIds are provided
if (!routedTeamMemberIds || !routedTeamMemberIds.length) {
  return users;
}
```

**触发条件：**
- `routedTeamMemberIds === null`
- 或 `routedTeamMemberIds.length === 0`

**行为：** 直接