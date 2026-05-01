# Cal.diy 活动类型权限完整链路分析报告

## 文档修订历史

| 版本 | 日期 | 修订内容 | 作者 |
|------|------|----------|------|
| v1.0 | 2026-05-02 | 初始版本，完成权限链路完整分析 | AI Assistant |

---

## 1. 权限链路总览

### 1.1 核心权限边界划分

Cal.diy 的活动类型权限系统存在 **两个独立的权限边界**：

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
│  │ team.isPrivate        │ ✅ 影响成员列表可见性                    │    │
│  │                      │    ❌ 不影响预约鉴权                      │    │
│  ├──────────────────────┼────────────────────────────────────────────┤    │
│  │ bookingRequiresAuth  │ ❌ 不影响可见性边界                      │    │
│  │                      │ ✅ 仅影响预约鉴权边界 (关键开关)         │    │
│  └──────────────────────┴────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键权限字段速查表

| 字段名 | 类型 | 默认值 | 作用范围 | 权限含义 |
|--------|------|--------|----------|----------|
| `eventType.hidden` | Boolean | `false` | 可见性边界 | `true` = 仅所有者可见，外部访客无法在列表中看到 |
| `eventType.bookingRequiresAuthentication` | Boolean | `false` | 预约鉴权边界 | `true` = 必须登录且具备特定权限才能预约 |
| `team.isPrivate` | Boolean | `false` | 可见性边界 | `true` = 非成员无法看到团队成员列表 |
| `organizationSettings.lockEventTypeCreationForUsers` | Boolean | `false` | 组织级控制 | `true` = 普通用户无法创建活动类型 |

---

## 2. 公开页面可见性边界详解

### 2.1 可见性控制流程

```
外部访客请求活动类型列表/详情
              │
              ▼
    ┌─────────────────────┐
    │  publicProcedure    │
    │  (无需身份认证)      │
    └──────────┬──────────┘
               │
               ▼
    ┌─────────────────────┐
    │  getPublicEvent()   │
    │  getEventTypesPublic() │
    └──────────┬──────────┘
               │
               ▼
    ┌──────────────────────────────────────────┐
    │           可见性检查点                     │
    ├──────────────────────────────────────────┤
    │                                          │
    │  1. 检查 eventType.hidden                 │
    │     ├── 若 hidden=true 且请求者非所有者   │
    │     └── 返回 null (不可见)                 │
    │                                          │
    │  2. 检查 team.isPrivate                   │
    │     ├── 若 team.isPrivate=true            │
    │     ├── 检查请求者是否为团队 ADMIN/OWNER   │
    │     └── 非成员: 隐藏成员列表               │
    │                                          │
    │  3. 检查用户是否锁定                      │
    │     └── 锁定用户的活动类型不可见           │
    │                                          │
    └──────────────────────────────────────────┘
```

### 2.2 隐藏活动类型检查逻辑

**代码位置**: `apps/api/v2/src/platform/event-types/event-types_2024_06_14/services/event-types.service.ts:141-143`

```typescript
async getEventTypeByUsernameAndSlug(params: {
  username: string;
  eventTypeSlug: string;
  orgSlug?: string;
  orgId?: number;
  authUser?: AuthOptionalUser;
}) {
  const user = await this.usersRepository.findByUsername(params.username, params.orgSlug, params.orgId);
  if (!user) return null;

  const eventType = await this.eventTypesRepository.getUserEventTypeBySlug(user.id, params.eventTypeSlug);
  if (!eventType) return null;

  // ⚠️ 关键: 仅检查可见性，不检查预约权限
  if (eventType.hidden && params.authUser?.id !== user.id) {
    return null;  // 非所有者无法访问隐藏的活动类型
  }

  return {
    ownerId: user.id,
    ...eventType,
  };
}
```

### 2.3 私有团队成员列表可见性

**代码位置**: `packages/features/eventtypes/lib/getPublicEvent.ts:534-556`

```typescript
let canViewPrivateTeamMembers = false;
if (currentUserId && event.teamId) {
  const permissionCheckService = new PermissionCheckService();
  canViewPrivateTeamMembers = await permissionCheckService.checkPermission({
    userId: currentUserId,
    teamId: event.teamId,
    permission: "team.read",
    fallbackRoles: [MembershipRole.ADMIN, MembershipRole.OWNER],
  });

  // 同时检查父组织权限
  if (!canViewPrivateTeamMembers && event.team?.parentId) {
    canViewPrivateTeamMembers = await permissionCheckService.checkPermission({
      userId: currentUserId,
      teamId: event.team.parentId,
      permission: "team.read",
      fallbackRoles: [MembershipRole.ADMIN, MembershipRole.OWNER],
    });
  }
}

// 私有团队对非成员隐藏成员列表
if (event.team?.isPrivate && !canViewPrivateTeamMembers) {
  users = [];
}
```

### 2.4 可见性边界条件矩阵

| 场景 | `hidden=true` | `team.isPrivate=true` | 外部访客可见性 | 登录非成员可见性 |
|------|---------------|-----------------------|----------------|------------------|
| 个人公开活动类型 | ❌ | 不适用 | ✅ 完全可见 | ✅ 完全可见 |
| 个人隐藏活动类型 | ✅ | 不适用 | ❌ 不可见 | ❌ 不可见 |
| 团队公开活动 + 公开团队 | ❌ | ❌ | ✅ 可见成员列表 | ✅ 可见成员列表 |
| 团队公开活动 + 私有团队 | ❌ | ✅ | ✅ 活动可见，成员列表隐藏 | ✅ 活动可见，成员列表隐藏 |
| 团队隐藏活动 + 公开团队 | ✅ | ❌ | ❌ 不可见 | ❌ 不可见 |

---

## 3. 发起预约鉴权边界详解

### 3.1 预约创建流程

```
用户发起预约请求 (POST /v2/bookings)
              │
              ▼
    ┌─────────────────────────────┐
    │  OptionalApiAuthGuard       │
    │  (认证是可选的，但会影响权限) │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌─────────────────────────────┐
    │  createBooking()            │
    │  bookings.service.ts:112-158 │
    └─────────────┬───────────────┘
                  │
                  ▼
    ┌──────────────────────────────────────────────────────┐
    │         预约鉴权检查点 (关键路径)                     │
    ├──────────────────────────────────────────────────────┤
    │                                                      │
    │  1. 获取活动类型 getBookedEventType()                 │
    │     └── 通过 eventTypeId 或 username+slug 获取        │
    │                                                      │
    │  2. ⚠️ 关键: 检查 bookingRequiresAuthentication      │
    │     └── checkBookingRequiresAuthenticationSetting()  │
    │                                                      │
    │  3. 检查调度类型有效性                                 │
    │     ├── MANAGED: 禁止直接预约                         │
    │     ├── COLLECTIVE/ROUND_ROBIN: 必须有主机           │
    │     └── 其他: 正常继续                                │
    │                                                      │
    │  4. 检查必填预订字段                                  │
    │     └── hasRequiredBookingFieldsResponses()           │
    │                                                      │
    │  5. 执行创建                                         │
    │     ├── 普通预约: createRegularBooking()              │
    │     ├── 重复预约: createRecurringBooking()            │
    │     └── 席位预约: createSeatedBooking()               │
    │                                                      │
    └──────────────────────────────────────────────────────┘
```

### 3.2 "需登录预约" 开关的完整逻辑

**代码位置**: `apps/api/v2/src/platform/bookings/2024-08-13/services/bookings.service.ts:169-186`

```typescript
async checkBookingRequiresAuthenticationSetting(
  eventType: EventTypeWithOwnerAndTeam,
  authUser: AuthOptionalUser,
  userIsEventTypeAdminOrOwner: boolean
) {
  // ⚠️ 关键: 仅当 bookingRequiresAuthentication=true 时才执行检查
  if (!eventType.bookingRequiresAuthentication) return true;

  // 检查 1: 用户必须已登录
  if (!authUser) {
    throw new UnauthorizedException(
      "checkBookingRequiresAuthentication - request must be authenticated by passing credentials belonging to event type owner, host or team or org admin or owner."
    );
  }

  // 检查 2: 用户必须是活动类型管理员或所有者
  if (!userIsEventTypeAdminOrOwner) {
    throw new ForbiddenException(
      "checkBookingRequiresAuthentication - user is not authorized to access this event type. User has to be either event type owner, host, team admin or owner or org admin or owner."
    );
  }
}
```

### 3.3 活动类型管理员/所有者判断标准

**代码位置**: `apps/api/v2/src/modules/event-types/services/event-type-access.service.ts:19-48`

```typescript
async userIsEventTypeAdminOrOwner(authUser: ApiAuthGuardUser, eventType: EventType): Promise<boolean> {
  const authUserId = authUser.id;
  const eventTypeId = eventType.id;
  const teamId = eventType.teamId;
  const eventTypeOwnerId = eventType.userId || null;

  // 条件 1: 系统管理员 (isSystemAdmin=true)
  if (authUser.isSystemAdmin) return true;

  // 条件 2: 活动类型直接所有者 (userId 匹配)
  if (eventTypeOwnerId === authUserId) return true;

  // 条件 3: 活动类型的主机 (Host) 或被分配者
  if (eventTypeId) {
    const isHostOrAssigned = await this.isUserHostOrAssignedToEventType(authUserId, eventTypeId);
    if (isHostOrAssigned) return true;
  }

  // 条件 4: 团队管理员或父组织管理员 (团队活动类型)
  if (teamId) {
    const isTeamOrParentOrgAdmin = await this.isUserTeamAdminOrParentOrgAdmin(authUserId, teamId);
    if (isTeamOrParentOrgAdmin) return true;
  }

  // 条件 5: 活动类型所有者的组织管理员 (个人活动类型)
  if (eventTypeOwnerId) {
    const isOrgAdminOrOwnerOfEventOwner = await this.isUserOrgAdminOrOwnerOfEventOwner(
      authUserId,
      eventTypeOwnerId
    );
    if (isOrgAdminOrOwnerOfEventOwner) return true;
  }

  return false;
}
```

### 3.4 主机/分配者检查

**代码位置**: `event-type-access.service.ts:50-56`

```typescript
private async isUserHostOrAssignedToEventType(authUserId: number, eventTypeId: number): Promise<boolean> {
  const [isUserHost, isUserAssigned] = await Promise.all([
    this.eventTypesRepository.isUserHostOfEventType(authUserId, eventTypeId),
    this.eventTypesRepository.isUserAssignedToEventType(authUserId, eventTypeId),
  ]);
  return isUserHost || isUserAssigned;
}
```

### 3.5 团队/组织管理员检查

**代码位置**: `event-type-access.service.ts:58-70`

```typescript
private async isUserTeamAdminOrParentOrgAdmin(authUserId: number, teamId: number): Promise<boolean> {
  // 检查: 是否为当前团队的 ADMIN 或 OWNER
  const membership = await this.membershipsRepository.getUserAdminOrOwnerTeamMembership(authUserId, teamId);
  if (membership) return true;

  // 检查: 是否为父组织的管理员 (当 team 是子团队时)
  const team = await this.teamsRepository.getById(teamId);
  const parentOrgId = team?.parentId ?? null;
  if (parentOrgId) {
    const isOrgAdmin = await this.membershipsRepository.isUserOrganizationAdmin(authUserId, parentOrgId);
    if (isOrgAdmin) return true;
  }

  return false;
}
```

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

## 5. 访问已有预约详情的权限校验

### 5.1 权限校验适用范围

**BookingPbacGuard** 适用于以下 API 端点：

| 端点 | HTTP 方法 | 守卫组合 | 适用场景 |
|------|-----------|----------|----------|
| `/v2/bookings/:bookingUid/recordings` | GET | `ApiAuthGuard + BookingUidGuard + BookingPbacGuard` | 获取会议录音 |
| `/v2/bookings/:bookingUid/transcripts` | GET | `ApiAuthGuard + BookingUidGuard + BookingPbacGuard` | 获取会议转录 |
| `/v2/bookings/:bookingUid/conferencing-sessions` | GET | `ApiAuthGuard + BookingUidGuard + BookingPbacGuard` | 获取视频会议会话 |

**⚠️ 重要注意**: 并非所有预约详情访问都需要 `BookingPbacGuard`：

| 端点 | 是否需要 BookingPbacGuard | 说明 |
|------|---------------------------|------|
| `GET /v2/bookings/:bookingUid` | ❌ 不需要 | 使用 `OptionalApiAuthGuard`，访客也可通过 bookingUid 访问 |
| `GET /v2/bookings/by-seat/:seatUid` | ❌ 不需要 | 同上，访客可访问 |
| `POST /v2/bookings/:bookingUid/reschedule` | ❌ 不需要 | 可选认证 |
| `POST /v2/bookings/:bookingUid/cancel` | ❌ 不需要 | 可选认证 |
| `GET /v2/bookings/:bookingUid/recordings` | ✅ 需要 | 敏感操作，需严格鉴权 |
| `GET /v2/bookings/` (列表) | ✅ 需要 | 必须认证，只能查看自己的预约 |

### 5.2 BookingPbacGuard 实现

**代码位置**: `apps/api/v2/src/platform/bookings/2024-08-13/guards/booking-pbac.guard.ts:1-59`

```typescript
@Injectable()
export class BookingPbacGuard implements CanActivate {
  private bookingAccessService: BookingAccessService;

  constructor(private readonly prismaReadService: PrismaReadService) {
    this.bookingAccessService = new BookingAccessService(
      this.prismaReadService.prisma
    );
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const bookingUid = request.params.bookingUid;

    // 检查 1: 用户必须已认证
    if (!user) {
      throw new UnauthorizedException();
    }

    // 检查 2: 用户必须对预约有访问权限
    const hasAccess = await this.bookingAccessService.doesUserIdHaveAccessToBooking({
      userId: user.id,
      bookingUid,
    });

    if (!hasAccess) {
      throw new ForbiddenException(
        `BookingPbacGuard - user with id=${user.id} does not have access to booking with uid=${bookingUid}`
      );
    }

    request.pbacAuthorizedRequest = true;
    return true;
  }
}
```

### 5.3 BookingAccessService 权限判断逻辑

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
  // 先获取预约详情
  const booking = bookingUid
    ? await bookingRepo.findByUidIncludeEventType({ bookingUid })
    : bookingId
      ? await bookingRepo.findByIdIncludeEventType({ bookingId })
      : null;

  if (!booking) return false;

  // ═══════════════════════════════════════════════════════════
  // Case 1: 用户是预约组织者 (booking.userId 匹配)
  // ═══════════════════════════════════════════════════════════
  if (userId === booking.userId) return true;

  // ═══════════════════════════════════════════════════════════
  // Case 2: 用户是活动类型的主机之一
  // ═══════════════════════════════════════════════════════════
  if (this.isUserAHost(userId, booking)) return true;

  // ═══════════════════════════════════════════════════════════
  // Case 3: 团队活动类型 - 用户是团队/组织管理员
  // ═══════════════════════════════════════════════════════════
  if (booking.eventType?.teamId) {
    const teamId = booking.eventType.teamId;
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId,
      teamId,
      permission: "booking.readTeamBookings",
      fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
    });
    return hasAccess;
  }

  // ═══════════════════════════════════════════════════════════
  // Case 4: 托管事件类型 - 检查父团队权限
  // ═══════════════════════════════════════════════════════════
  if (booking.eventType?.parent?.teamId) {
    const isAdminOrUser = await userRepo.isAdminOfTeamOrParentOrg({
      userId,
      teamId: booking.eventType.parent.teamId,
    });
    return isAdminOrUser;
  }

  if (!booking.userId) return false;

  // 获取预约组织者的组织和团队信息
  const bookingOwner = await userRepo.getUserOrganizationAndTeams({ userId: booking.userId });
  if (!bookingOwner) return false;

  // ═══════════════════════════════════════════════════════════
  // Case 5: 个人活动类型 - 用户是预约组织者所在组织的管理员
  // ═══════════════════════════════════════════════════════════
  if (bookingOwner.organizationId) {
    const orgId = bookingOwner.organizationId;
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId,
      teamId: orgId,
      permission: "booking.readOrgBookings",
      fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
    });
    if (hasAccess) return true;
  }

  // ═══════════════════════════════════════════════════════════
  // Case 6: 个人活动类型 - 用户是预约组织者所属任意团队的管理员
  // ═══════════════════════════════════════════════════════════
  for (const membership of bookingOwner.teams) {
    const teamId = membership.teamId;
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId,
      teamId,
      permission: "booking.readTeamBookings",
      fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
    });
    if (hasAccess) return true;
  }

  return false;
}
```

### 5.4 主机判断逻辑详解

**代码位置**: `BookingAccessService.ts:22-46`

```typescript
private isUserAHost(userId: number, booking: BookingForAccessCheck): boolean {
  const hostMap = new Map<number, { id: number; email: string }>();

  const addHost = (id: number, email: string) => {
    if (!hostMap.has(id)) {
      hostMap.set(id, { id, email });
    }
  };

  // 来源 1: eventType.hosts (多主机活动类型的主机列表)
  booking?.eventType?.hosts?.forEach((host) =>
    addHost(host.userId, host.user.email)
  );
  
  // 来源 2: eventType.users (旧版关联，向后兼容)
  booking?.eventType?.users?.forEach((user) => addHost(user.id, user.email));

  // 来源 3: booking.user (预约组织者)
  if (booking?.user?.id && booking?.user?.email) {
    addHost(booking.user.id, booking.user.email);
  }

  // 过滤: 只保留与会者邮箱匹配的主机，或预约组织者本人
  const attendeeEmails = new Set(booking.attendees?.map((attendee) => attendee.email));
  const filteredHosts = Array.from(hostMap.values()).filter(
    (host) => attendeeEmails.has(host.email) || host.id === booking.user?.id
  );

  return filteredHosts.some((host) => host.id === userId);
}
```

### 5.5 预约详情访问权限矩阵

#### 场景 A: 访问敏感端点 (需 BookingPbacGuard)

| 访问者身份 | 团队活动类型 | 个人活动类型 |
|-----------|-------------|-------------|
| **预约组织者本人** | ✅ 允许 | ✅ 允许 |
| **活动类型主机** | ✅ 允许 | ✅ 允许 |
| **团队管理员 (OWNER/ADMIN)** | ✅ 允许 | 不适用 |
| **组织管理员 (父组织)** | ✅ 允许 | ✅ 允许 (若组织者在该组织) |
| **组织者所属任意团队的管理员** | 不适用 | ✅ 允许 |
| **预约参会者** | ❌ 拒绝 | ❌ 拒绝 |
| **其他登录用户** | ❌ 拒绝 | ❌ 拒绝 |
| **外部访客** | ❌ 401 | ❌ 401 |

#### 场景 B: 访问公开预约详情 (GET /v2/bookings/:bookingUid)

| 访问者身份 | 访问权限 | 数据可见性 |
|-----------|---------|-----------|
| **拥有有效 bookingUid 的任何人** | ✅ 可访问 | 基本信息可见 |
| **活动类型管理员/所有者** | ✅ 可访问 | 额外数据: 参会者列表 (当 seatsShowAttendees=false 时) |
| **普通访客/参会者** | ✅ 可访问 | 席位预约: 仅能看到自己的参会信息 |

**代码依据**: `bookings.service.ts:554-592` 的 `getBooking()` 方法

```typescript
async getBooking(uid: string, authUser: AuthOptionalUser) {
  const booking = await this.bookingsRepository.getByUidWithAttendeesWithBookingSeatAndUserAndEvent(uid);
  const userIsEventTypeAdminOrOwner =
    authUser && booking?.eventType
      ? await this.eventTypeAccessService.userIsEventTypeAdminOrOwner(authUser, booking.eventType)
      : false;

  // 席位预约: 根据权限决定是否显示所有参会者
  if (isSeated) {
    const showAttendees = userIsEventTypeAdminOrOwner || !!booking.eventType?.seatsShowAttendees;
    return this.outputService.getOutputSeatedBooking(booking, showAttendees);
  }
  // ...
}
```

### 5.6 席位预约 (Seated Booking) 的特殊访问逻辑

**代码位置**: `bookings.service.ts:594-636`

```typescript
async getBookingBySeatUid(seatUid: string, authUser: AuthOptionalUser) {
  const bookingSeat = await this.bookingSeatRepository.getByReferenceUidIncludeBookingWithAttendeesAndUserAndEvent(seatUid);
  // ...

  const userIsEventTypeAdminOrOwner = /* 检查权限 */;
  const seatsShowAttendees = !!booking.eventType?.seatsShowAttendees;
  const showAllAttendees = userIsEventTypeAdminOrOwner || seatsShowAttendees;

  // ⚠️ 关键: 非管理员且 seatsShowAttendees=false 时，只能看到自己的参会信息
  if (!showAllAttendees) {
    const seatAttendee = booking.attendees.find(
      (attendee) => attendee.bookingSeat?.referenceUid === seatUid
    );
    const bookingWithFilteredAttendees = {
      ...booking,
      attendees: seatAttendee ? [seatAttendee] : [],  // 仅保留当前席位的参会者
    };
    return this.outputService.getOutputSeatedBooking(bookingWithFilteredAttendees, true);
  }

  // 管理员或 seatsShowAttendees=true: 显示所有参会者
  return this.outputService.getOutputSeatedBooking(booking, true);
}
```

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
│  │  ├── true 且 非团队管理员 │   │
│  │  └── false 或 团队管理员 │   │
│  │                                                                              │   │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │   │
│  │  │  ⚠️ 注意: bookingRequiresAuthentication 在此阶段不生效                    │  │   │
│  │  │     即使开关打开，只要活动类型是公开的，访客仍能看到活动详情页             │  │   │
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
│  │  │  2. 用户是活动类型管理员/所有者                                        │   │   │
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

### 6.2 预约详情访问权限决策树

```
用户请求访问预约详情 (bookingUid)
              │
              ▼
    ┌─────────────────────────┐
    │  端点类型?               │
    └───────────┬─────────────┘
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
   敏感端点         普通端点
   (录音/转录等)    (基本详情)
        │               │
        ▼               │
┌───────────────┐        │
│ 需要认证吗?    │        │
└───────┬───────┘        │
        │                │
       Yes               │
        │                │
        ▼                │
┌───────────────┐        │
│ BookingPbacGuard │      │
└───────┬───────┘        │
        │                │
   ┌────┴────┐           │
   │         │           │
   ▼         ▼           │
 通过      拒绝          │
   │         │           │
   │         ▼           │
   │    401/403         │
   │                     │
   ▼                     ▼
┌─────────────────────────────────┐
│      数据返回阶段                │
├─────────────────────────────────┤
│  席位预约 (seated booking):      │
│  ┌─────────────────────────────┐│
│  │ 是管理员 或 seatsShowAttendees ││
│  │ = true?                      ││
│  └───────┬─────────────────────┘│
│          │                       │
│      ┌───┴───┐                   │
│      │       │                   │
│     Yes     No                   │
│      │       │                   │
│      ▼       ▼                   │
│  显示所有   仅显示                │
│  参会者    当前席位              │
│           参会者                 │
└─────────────────────────────────┘
```

---

## 7. 关键代码位置索引

### 7.1 可见性边界相关

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| 隐藏活动类型检查 | `apps/api/v2/src/platform/event-types/event-types_2024_06_14/services/event-types.service.ts` | 141-143 |
| 私有团队成员可见性 | `packages/features/eventtypes/lib/getPublicEvent.ts` | 534-556 |
| 公开事件类型获取 | `packages/features/eventtypes/lib/getPublicEvent.ts` | 284-605 |

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
| 席位预约访问逻辑 | `apps/api/v2/src/platform/bookings/2024-08-13/services/bookings.service.ts` | 594-636 |

---

## 8. 常见问题解答

### Q1: 为什么设置了 `bookingRequiresAuthentication=true` 后，访客仍然能看到活动类型页面？

**A**: 这是预期行为。`bookingRequiresAuthentication` 只影响**预约鉴权边界**，不影响**可见性边界**。

- 可见性边界由 `eventType.hidden` 和 `team.isPrivate` 控制
- 预约鉴权边界由 `bookingRequiresAuthentication` 控制

访客可以看到活动详情页，但点击"预约"按钮时会收到 401/403 错误。

### Q2: 活动类型的主机 (Host) 能否预约自己的活动类型？

**A**: 可以。在 `bookingRequiresAuthentication=true` 的情况下：

- 主机 (`isUserHostOfEventType`) 被视为"活动类型管理员"
- 因此主机可以正常预约自己作为主机的活动类型

### Q3: 预约参会者能否访问预约的录音/转录？

**A**: 不能。`BookingPbacGuard` 的权限判断中：

- **预约组织者** → ✅ 允许
- **活动类型主机** → ✅ 允许
- **团队/组织管理员** → ✅ 允许
- **参会者** → ❌ 拒绝 (除非同时是组织者/主机/管理员)

### Q4: 外部访客能否通过 bookingUid 访问预约详情？

**A**: 可以访问**基本详情**，但有以下限制：

- `GET /v2/bookings/:bookingUid` - ✅ 可访问 (使用 `OptionalApiAuthGuard`)
- `GET /v2/bookings/:bookingUid/recordings` - ❌ 401 拒绝 (需要 `BookingPbacGuard`)
- 席位预约场景下，访客只能看到自己席位的参会信息，看不到其他参会者

### Q5: 组织管理员能否查看组织内所有用户的预约？

**A**: 可以。`BookingAccessService.doesUserIdHaveAccessToBooking()` 的 Case 5:

```typescript
// 检查用户是否是预约组织者所在组织的管理员
if (bookingOwner.organizationId) {
  const orgId = bookingOwner.organizationId;
  const hasAccess = await this.permissionCheckService.checkPermission({
    userId,
    teamId: orgId,
    permission: "booking.readOrgBookings",
    fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
  });
  if (hasAccess) return true;
}
```

---

## 9. 附录：权限相关数据库字段

### 9.1 EventType 表

| 字段名 | 类型 | 默认值 | 权限含义 |
|--------|------|--------|----------|
| `userId` | Int | null | 活动类型所有者 ID |
| `teamId` | Int | null | 所属团队 ID (团队活动类型) |
| `hidden` | Boolean | false | 隐藏活动，仅所有者可见 |
| `bookingRequiresAuthentication` | Boolean | false | 需登录预约开关 |
| `schedulingType` | Enum | null | 调度类型 (ROUND_ROBIN/COLLECTIVE/MANAGED) |
| `seatsPerTimeSlot` | Int | null | 席位数量 (席位预约) |
| `seatsShowAttendees` | Boolean | true | 是否显示参会者列表 |

### 9.2 Team 表

| 字段名 | 类型 | 默认值 | 权限含义 |
|--------|------|--------|----------|
| `parentId` | Int | null | 父组织 ID (子团队) |
| `isPrivate` | Boolean | false | 私有团队，隐藏成员列表 |
| `isOrganization` | Boolean | false | 是否为组织 |

### 9.3 Membership 表

| 字段名 | 类型 | 默认值 | 权限含义 |
|--------|------|--------|----------|
| `userId` | Int | - | 用户 ID |
| `teamId` | Int | - | 团队/组织 ID |
| `role` | Enum | MEMBER | 角色 (MEMBER/ADMIN/OWNER) |
| `accepted` | Boolean | false | 是否已接受邀请 |

### 9.4 User 表

| 字段名 | 类型 | 默认值 | 权限含义 |
|--------|------|--------|----------|
| `role` | Enum | USER | 系统角色 (USER/ADMIN) |
| `organizationId` | Int | null | 所属组织 ID |
| `locked` | Boolean | false | 账户是否锁定 |

---

**报告生成时间**: 2026-05-02  
**基于代码版本**: 当前工作目录版本
