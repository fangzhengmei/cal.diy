# Cal.diy 活动类型权限完整链路分析报告 (修正版)

## 文档修订历史

| 版本 | 日期 | 修订内容 | 作者 |
|------|------|----------|------|
| v2.0 | 2026-05-02 | 修正版：补充完整的权限守卫接口列表、修正私有团队可见性行为、删除无依据的锁定用户可见性结论 | AI Assistant |
| v1.0 | 2026-05-02 | 初始版本 | AI Assistant |

---

## 1. 权限链路总览

### 1.1 核心权限边界划分

Cal.diy 的活动类型权限系统存在 **两个独立的权限边界**，需明确区分：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         活动类型权限边界模型                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────┐      ┌─────────────────────────────────┐     │
│  │   公开页面可见性边界      │      │       发起预约鉴权边界           │     │
│  │   (Visibility Control)   │      │    (Booking Authorization)     │     │
│  └───────────┬─────────────┘      └───────────────┬─────────────────┘     │
│              │                                      │                       │
│              ▼                                      ▼                       │
│  ┌───────────────────────────────────────────────────────────────────┐    │
│  │                    影响因子矩阵                                    │    │
│  ├──────────────────────┬────────────────────────────────────────────┤    │
│  │      字段             │              影响范围                      │    │
│  ├──────────────────────┼────────────────────────────────────────────┤    │
│  │ eventType.hidden      │ ✅ 仅影响可见性边界                      │    │
│  │                      │    ❌ 不影响预约鉴权                      │    │
│  ├──────────────────────┼────────────────────────────────────────────┤    │
│  │ team.isPrivate        │ ✅ 影响成员列表可见性 (见第3节详细分析)  │    │
│  │                      │    ❌ 不影响预约鉴权                      │    │
│  ├──────────────────────┼────────────────────────────────────────────┤    │
│  │ bookingRequiresAuth  │ ❌ 不影响可见性边界                      │    │
│  │                      │ ✅ 仅影响预约鉴权边界 (关键开关)          │    │
│  └──────────────────────┴────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键权限字段速查表

| 字段名 | 类型 | 默认值 | 作用边界 | 权限含义 |
|--------|------|--------|----------|----------|
| `eventType.hidden` | Boolean | `false` | 可见性边界 | `true` = 仅所有者可见，外部访客无法在列表中看到 |
| `eventType.bookingRequiresAuthentication` | Boolean | `false` | 预约鉴权边界 | `true` = 必须登录且具备特定权限才能预约 |
| `team.isPrivate` | Boolean | `false` | 可见性边界 | `true` = 影响成员列表可见性 (详见第3节) |
| `user.locked` | Boolean | `false` | 用户查询边界 | `true` = 用户账户被锁定，无法登录 (详见第5节) |

---

## 2. 访问已有预约详情的权限守卫接口完整列表

### 2.1 API v2 2024-08-13 版本端点分类

#### 分类 A: 使用 `BookingPbacGuard` 的敏感端点

**守卫组合**: `@UseGuards(ApiAuthGuard, BookingUidGuard, BookingPbacGuard)`

这些端点需要 **必须认证** + **严格的预约访问权限校验**：

| 端点 | HTTP 方法 | 额外装饰器 | 权限要求 |
|------|-----------|------------|----------|
| `/v2/bookings/:bookingUid/recordings` | GET | `@Pbac(["booking.readRecordings"])` `@Permissions([BOOKING_READ])` | 获取会议录音 |
| `/v2/bookings/:bookingUid/transcripts` | GET | `@Pbac(["booking.readRecordings"])` `@Permissions([BOOKING_READ])` | 获取会议转录 |
| `/v2/bookings/:bookingUid/conferencing-sessions` | GET | `@Pbac(["booking.readRecordings"])` `@Permissions([BOOKING_READ])` | 获取视频会议会话 |

**代码位置**: `apps/api/v2/src/platform/bookings/2024-08-13/controllers/bookings.controller.ts`
- Line 223-242: `getBookingRecordings`
- Line 244-267: `getBookingTranscripts`
- Line 566-585: `getVideoSessions`

---

#### 分类 B: 使用 `ApiAuthGuard` 必须认证的端点

**守卫组合**: `@UseGuards(ApiAuthGuard, ...)` 或 `@UseGuards(ApiAuthGuard)`

这些端点需要 **必须认证**，但不使用 `BookingPbacGuard`：

| 端点 | HTTP 方法 | 守卫组合 | 额外装饰器 | 功能说明 |
|------|-----------|----------|------------|----------|
| `/v2/bookings/` | GET | `ApiAuthGuard` | `@Permissions([BOOKING_READ])` | 获取当前用户的预约列表 |
| `/v2/bookings/:bookingUid/mark-absent` | POST | `ApiAuthGuard, BookingUidGuard` | `@Permissions([BOOKING_WRITE])` | 标记用户缺席 |
| `/v2/bookings/:bookingUid/reassign` | POST | `ApiAuthGuard, BookingUidGuard` | `@Permissions([BOOKING_WRITE])` | 重新分配主机 (自动选择) |
| `/v2/bookings/:bookingUid/reassign/:userId` | POST | `ApiAuthGuard, BookingUidGuard` | `@Permissions([BOOKING_WRITE])` | 重新分配主机 (指定用户) |
| `/v2/bookings/:bookingUid/confirm` | POST | `ApiAuthGuard, BookingUidGuard` | `@Permissions([BOOKING_WRITE])` | 确认预约 |
| `/v2/bookings/:bookingUid/decline` | POST | `ApiAuthGuard, BookingUidGuard` | `@Permissions([BOOKING_WRITE])` | 拒绝预约 |
| `/v2/bookings/:bookingUid/calendar-links` | GET | `ApiAuthGuard, BookingUidGuard` | `@Permissions([BOOKING_READ])` | 获取日历链接 |
| `/v2/bookings/:bookingUid/references` | GET | `ApiAuthGuard, BookingUidGuard` | `@Permissions([BOOKING_READ])` | 获取预约关联信息 |

---

#### 分类 C: 使用 `OptionalApiAuthGuard` 可选认证的端点

**守卫组合**: `@UseGuards(OptionalApiAuthGuard, ...)`

这些端点 **允许访客访问** (通过 bookingUid 或 seatUid)，但登录用户可能获得更多数据：

| 端点 | HTTP 方法 | 守卫组合 | 功能说明 | 数据可见性差异 |
|------|-----------|----------|----------|----------------|
| `/v2/bookings/` | POST | `OptionalApiAuthGuard` | 创建预约 | 受 `bookingRequiresAuthentication` 影响 |
| `/v2/bookings/by-seat/:seatUid` | GET | `OptionalApiAuthGuard` | 通过席位 UID 获取预约 | 席位预约：非管理员仅能看到自己的参会信息 |
| `/v2/bookings/:bookingUid` | GET | `BookingUidGuard, OptionalApiAuthGuard` | 通过 bookingUid 获取预约 | 同上 |
| `/v2/bookings/:bookingUid/reschedule` | POST | `BookingUidGuard, OptionalApiAuthGuard` | 重新预约 | - |
| `/v2/bookings/:bookingUid/cancel` | POST | `BookingUidGuard, OptionalApiAuthGuard` | 取消预约 | - |

---

### 2.2 BookingPbacGuard 权限判断逻辑

**代码位置**: `packages/features/bookings/services/BookingAccessService.ts:56-138`

```typescript
async doesUserIdHaveAccessToBooking({
  userId,
  bookingUid,
  bookingId,
}: {
  userId: number;
  bookingUid?: string;
  bookingId?: number;
}): Promise<boolean> {
  // Case 1: 用户是预约组织者 (booking.userId 匹配)
  if (userId === booking.userId) return true;

  // Case 2: 用户是活动类型的主机之一
  if (this.isUserAHost(userId, booking)) return true;

  // Case 3: 团队活动类型 - 用户是团队/组织管理员 (OWNER/ADMIN)
  if (booking.eventType?.teamId) {
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId,
      teamId: booking.eventType.teamId,
      permission: "booking.readTeamBookings",
      fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
    });
    return hasAccess;
  }

  // Case 4: 个人活动类型 - 用户是预约组织者所在组织的管理员
  if (bookingOwner.organizationId) {
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId,
      teamId: bookingOwner.organizationId,
      permission: "booking.readOrgBookings",
      fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
    });
    if (hasAccess) return true;
  }

  // Case 5: 个人活动类型 - 用户是预约组织者所属任意团队的管理员
  for (const membership of bookingOwner.teams) {
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId,
      teamId: membership.teamId,
      permission: "booking.readTeamBookings",
      fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
    });
    if (hasAccess) return true;
  }

  return false;
}
```

### 2.3 预约详情访问权限矩阵

#### 场景: 访问 `BookingPbacGuard` 保护的敏感端点

| 访问者身份 | 团队活动类型 | 个人活动类型 |
|-----------|-------------|-------------|
| **预约组织者本人** | ✅ 允许 | ✅ 允许 |
| **活动类型主机** | ✅ 允许 | ✅ 允许 |
| **团队管理员 (OWNER/ADMIN)** | ✅ 允许 | 不适用 |
| **组织管理员 (父组织)** | ✅ 允许 | ✅ 允许 (若组织者在该组织) |
| **组织者所属任意团队的管理员** | 不适用 | ✅ 允许 |
| **预约参会者** | ❌ 拒绝 | ❌ 拒绝 |
| **其他登录用户** | ❌ 拒绝 | ❌ 拒绝 |
| **外部访客** | ❌ 401 Unauthorized | ❌ 401 Unauthorized |

---

## 3. 私有团队成员可见性判断的真实行为

### 3.1 代码分析

**代码位置**: `packages/features/eventtypes/lib/getPublicEvent.ts:534-556`

```typescript
// ⚠️ 关键发现: PermissionCheckService 是一个存根实现
class PermissionCheckService {
  constructor(_prisma?: unknown) {}
  async checkPermission(..._args: unknown[]) {
    return true;  // 总是返回 true!
  }
  async hasPermission(..._args: unknown[]) {
    return true;
  }
  async getTeamIdsWithPermission(..._args: unknown[]): Promise<number[]> {
    return [];
  }
}

// 可见性判断逻辑
let canViewPrivateTeamMembers = false;
if (currentUserId && event.teamId) {
  const permissionCheckService = new PermissionCheckService();
  // ⚠️ 这里总是返回 true，因为 checkPermission 是存根实现
  canViewPrivateTeamMembers = await permissionCheckService.checkPermission({
    userId: currentUserId,
    teamId: event.teamId,
    permission: "team.read",
    fallbackRoles: [MembershipRole.ADMIN, MembershipRole.OWNER],
  });
  // ...
}

// 最终判断
if (event.team?.isPrivate && !canViewPrivateTeamMembers) {
  users = [];
}
```

### 3.2 真实行为分析

基于代码分析，**当前实现**的私有团队成员可见性行为：

| 场景 | `currentUserId` (是否登录) | `canViewPrivateTeamMembers` | 成员列表 `users` |
|------|----------------------------|------------------------------|------------------|
| 外部访客访问 | 不存在 (`undefined`) | `false` | `[]` (空列表) |
| 任意登录用户访问 | 存在 (任意用户 ID) | `true` (存根总是返回 true) | **完整列表** |

### 3.3 预期行为 vs 当前实现

| 维度 | 预期行为 (按代码语义) | 当前实际行为 (存根实现) |
|------|----------------------|------------------------|
| 外部访客 | 看不到成员列表 | 看不到成员列表 (一致) |
| 团队普通成员 (MEMBER) | 看不到成员列表 | **能看到成员列表** (不一致) |
| 团队管理员 (ADMIN/OWNER) | 能看到成员列表 | 能看到成员列表 (一致) |
| 其他登录用户 (非团队成员) | 看不到成员列表 | **能看到成员列表** (不一致) |

### 3.4 重要提示

⚠️ **此为存根实现，可能不是最终预期行为**

- 当前 `PermissionCheckService` 所有方法都返回 `true` 或空数组
- 这可能是开发过程中的临时实现
- 建议检查是否有其他地方注入了真正的权限检查服务
- 在生产环境中，此逻辑可能需要替换为真正的权限校验

---

## 4. "需登录预约"开关下各身份权限矩阵

### 4.1 身份类型定义

| 身份类型 | 定义标准 | 代码判断条件 |
|----------|----------|--------------|
| **外部访客** | 未登录用户 | `authUser` 为 `null` 或 `undefined` |
| **普通登录用户** | 已登录但无特殊权限 | 登录状态，但不满足以下任一条件 |
| **活动类型所有者** | 创建活动类型的用户 | `eventType.userId === authUser.id` |
| **活动类型主机** | 被添加为 Host 的用户 | `isUserHostOfEventType()` 返回 true |
| **团队管理员** | 团队的 ADMIN/OWNER | `membership.role` ∈ [ADMIN, OWNER] |
| **组织管理员** | 组织的 ADMIN/OWNER | `isUserOrganizationAdmin()` 返回 true |
| **系统管理员** | 全局 ADMIN | `authUser.isSystemAdmin === true` |

### 4.2 权限放行/拒绝条件表

#### 场景 A: `bookingRequiresAuthentication = false` (默认值)

| 身份类型 | 能否预约 | 原因说明 |
|----------|----------|----------|
| 外部访客 | ✅ 放行 | 开关关闭，无需认证 |
| 普通登录用户 | ✅ 放行 | 开关关闭，无需额外权限 |
| 活动类型所有者 | ✅ 放行 | 无限制 |
| 活动类型主机 | ✅ 放行 | 无限制 |
| 团队管理员 | ✅ 放行 | 无限制 |
| 组织管理员 | ✅ 放行 | 无限制 |
| 系统管理员 | ✅ 放行 | 无限制 |

**代码依据**: `checkBookingRequiresAuthenticationSetting()` 直接返回 `true`

---

#### 场景 B: `bookingRequiresAuthentication = true` (需登录预约)

| 身份类型 | 能否预约 | HTTP 状态码 | 原因说明 |
|----------|----------|-------------|----------|
| **外部访客** | ❌ 拒绝 | 401 Unauthorized | 未通过认证，`authUser` 为 null |
| **普通登录用户** | ❌ 拒绝 | 403 Forbidden | 已登录，但不是活动类型管理员/所有者 |
| **活动类型所有者** | ✅ 放行 | 200 OK | `eventTypeOwnerId === authUserId` 匹配 |
| **活动类型主机** | ✅ 放行 | 200 OK | `isUserHostOfEventType()` 返回 true |
| **团队管理员** | ✅ 放行 | 200 OK | 团队的 ADMIN/OWNER 角色 |
| **组织管理员** | ✅ 放行 | 200 OK | 活动类型所有者所在组织的管理员 |
| **系统管理员** | ✅ 放行 | 200 OK | `authUser.isSystemAdmin === true` |

### 4.3 错误信息对照表

| 场景 | 错误类型 | 错误消息 |
|------|----------|----------|
| 外部访客预约需登录活动 | `UnauthorizedException` | "request must be authenticated by passing credentials belonging to event type owner, host or team or org admin or owner." |
| 普通登录用户预约需登录活动 | `ForbiddenException` | "user is not authorized to access this event type. User has to be either event type owner, host, team admin or owner or org admin or owner." |

---

## 5. 锁定用户相关说明

### 5.1 `exclude-locked-users` 扩展的真实作用

**代码位置**: `packages/prisma/extensions/exclude-locked-users.ts`

```typescript
export function excludeLockedUsersExtension() {
  return Prisma.defineExtension({
    query: {
      user: {
        async findUnique({ args, query }) {
          return excludeLockedUsers(args, query);
        },
        async findFirst({ args, query }) {
          return excludeLockedUsers(args, query);
        },
        async findMany({ args, query }) {
          return excludeLockedUsers(args, query);
        },
        // ...
      },
    },
  });
}

async function excludeLockedUsers(args, query) {
  args.where = args.where || {};
  const whereString = safeJSONStringify(args.where);
  const shouldIncludeLocked = whereString.includes('"locked":');
  // 除非明确指定，否则排除锁定用户
  if (!shouldIncludeLocked) {
    args.where.locked = false;
  }
  return query(args);
}
```

### 5.2 作用范围说明

| 维度 | 说明 |
|------|------|
| **影响范围** | 仅影响 `User` 模型的查询操作 |
| **不影响** | 活动类型查询、预约查询、团队查询等 |
| **实际效果** | 查询用户时自动添加 `locked: false` 条件，除非查询中明确包含 `locked` 字段 |
| **与活动类型可见性的关系** | **无直接关系** |

### 5.3 之前结论的修正

❌ **删除/修正以下无代码依据的结论**：

| 之前结论 | 修正说明 |
|----------|----------|
| "锁定用户的活动类型不可见" | ❌ 无代码依据。`exclude-locked-users` 仅影响用户查询，不影响活动类型查询。 |
| "检查用户是否锁定" (可见性检查点) | ❌ 无代码依据。在 `getPublicEvent` 等可见性相关代码中，未发现对 `user.locked` 字段的检查。 |

### 5.4 锁定用户的实际影响

锁定用户 (`user.locked = true`) 的实际影响：

1. **无法登录**: 在认证流程中，锁定用户可能被拒绝登录 (需确认具体认证逻辑)
2. **无法被查询**: 通过 `exclude-locked-users` 扩展，普通用户查询不会返回锁定用户
3. **活动类型**: 锁定用户创建的活动类型**仍然存在**，如果 `hidden=false` 且活动类型属于团队/公开状态，**理论上仍可被访问** (需进一步验证)

---

## 6. 完整权限链路流程图

### 6.1 活动类型访问完整流程

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                      活动类型访问完整权限链路                                          │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  访客/用户请求                                                                        │
│       │                                                                              │
│       ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                        第一阶段: 可见性边界检查                                │   │
│  │                   (检查能否"看到"活动类型)                                     │   │
│  ├──────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                              │   │
│  │  检查点 1: eventType.hidden                                                   │   │
│  │  ├── true 且 非所有者 │   │
│  │  └── false 或 所有者 │   │
│  │                                                                              │   │
│  │  检查点 2: team.isPrivate (团队活动类型)                                      │   │
│  │  ├── ⚠️ 当前存根实现: 仅未登录用户看不到成员列表                               │   │
│  │  │         任意登录用户都能看到成员列表                                       │   │
│  │  └── false 或 有权限 │   │
│  │                                                                              │   │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │   │
│  │  │  ⚠️ bookingRequiresAuthentication 在此阶段不生效                          │  │   │
│  │  │     即使开关打开，只要活动类型是公开的，访客仍能看到活动详情页               │  │   │
│  │  └────────────────────────────────────────────────────────────────────────┘  │   │
│  │                                                                              │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│       │                                                                              │
│       │ 通过可见性检查                                                                │
│       ▼                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                      第二阶段: 预约鉴权边界检查                                │   │
│  │                   (检查能否"预约"活动类型)                                     │   │
│  ├──────────────────────────────────────────────────────────────────────────────┤   │
│  │                                                                              │   │
│  │  检查点: bookingRequiresAuthentication                                        │   │
│  │  ├── false │   │
│  │  │            └── 任何人都能预约 (包括外部访客)                                │   │
│  │  │                                                                           │   │
│  │  └── true │   │
│  │       │                                                                      │   │
│  │       ▼                                                                      │   │
│  │  ┌──────────────────────────────────────────────────────────────────────┐   │   │
│  │  │  必须同时满足:                                                         │   │   │
│  │  │  1. 用户已登录 (authUser != null)                                     │   │   │
│  │  │  2. 用户是活动类型管理员/所有者                                          │   │   │
│  │  │     ├── 系统管理员 (isSystemAdmin)                                    │   │   │
│  │  │     ├── 活动类型所有者 (userId 匹配)                                   │   │   │
│  │  │     ├── 活动类型主机 (Host)                                            │   │   │
│  │  │     ├── 团队管理员 (OWNER/ADMIN)                                       │   │   │
│  │  │     └── 组织管理员 (活动类型所有者所在组织)                              │   │   │
│  │  └──────────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                              │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 关键代码位置索引

### 7.1 可见性边界相关

| 功能 | 文件路径 | 行号 | 备注 |
|------|----------|------|------|
| 隐藏活动类型检查 | `apps/api/v2/src/platform/event-types/event-types_2024_06_14/services/event-types.service.ts` | 141-143 | - |
| 私有团队成员可见性 | `packages/features/eventtypes/lib/getPublicEvent.ts` | 534-556 | ⚠️ 存根实现 |
| PermissionCheckService 存根 | `packages/features/eventtypes/lib/getPublicEvent.ts` | 28-39 | 总是返回 true |
| 公开事件类型获取 | `packages/features/eventtypes/lib/getPublicEvent.ts` | 284-605 | - |

### 7.2 预约鉴权边界相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 需登录预约检查 | `apps/api/v2/src/platform/bookings/2024-08-13/services/bookings.service.ts` | 169-186 |
| 活动类型管理员判断 | `apps/api/v2/src/modules/event-types/services/event-type-access.service.ts` | 19-48 |
| 预约创建主流程 | `apps/api/v2/src/platform/bookings/2024-08-13/services/bookings.service.ts` | 112-158 |

### 7.3 预约详情访问相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| BookingPbacGuard | `apps/api/v2/src/platform/bookings/2024-08-13/guards/booking-pbac.guard.ts` | 1-59 |
| 预约访问权限判断 | `packages/features/bookings/services/BookingAccessService.ts` | 56-138 |
| 主机判断逻辑 | `packages/features/bookings/services/BookingAccessService.ts` | 22-46 |
| BookingsController (v2 2024-08-13) | `apps/api/v2/src/platform/bookings/2024-08-13/controllers/bookings.controller.ts` | 1-586 |

### 7.4 锁定用户相关

| 功能 | 文件路径 | 行号 | 备注 |
|------|----------|------|------|
| exclude-locked-users 扩展 | `packages/prisma/extensions/exclude-locked-users.ts` | 1-52 | 仅影响用户查询 |

---

## 8. 修正说明汇总

### 8.1 相对于 v1.0 版本的主要修正

| 修正项 | v1.0 版本 | v2.0 修正版 |
|--------|-----------|-------------|
| **受权限守卫保护的接口** | 仅列出 3 个使用 `BookingPbacGuard` 的端点 | **完整列出所有分类**: A类 (BookingPbacGuard)、B类 (ApiAuthGuard)、C类 (OptionalApiAuthGuard) |
| **私有团队成员可见性** | 描述为预期行为 | **修正**: 指出 `PermissionCheckService` 是存根实现，当前任意登录用户都能看到私有团队成员列表 |
| **锁定用户可见性** | 声称"锁定用户的活动类型不可见" | **删除**: 此结论无代码依据。`exclude-locked-users` 仅影响用户查询，不影响活动类型可见性 |
| **流程图和决策树** | 包含无依据的"检查用户是否锁定" | **移除**: 相关无依据的检查点 |

### 8.2 待确认事项

以下问题建议进一步验证：

1. **PermissionCheckService 存根问题**: 是否有其他地方注入了真正的权限检查服务？当前 `getPublicEvent` 中的实现是局部的还是全局的？

2. **锁定用户的活动类型**: 锁定用户创建的活动类型在数据库中仍然存在，这些活动类型是否真的可以被访问？

3. **组织级权限**: `isUserOrganizationAdmin` 等方法的具体实现逻辑是什么？

---

**报告生成时间**: 2026-05-02  
**基于代码版本**: 当前工作目录版本  
**报告版本**: v2.0 (修正版)
