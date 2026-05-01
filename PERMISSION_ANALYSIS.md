# Cal.diy 用户与活动类型权限关系分析报告

## 1. 系统架构概览

### 1.1 核心权限组件

Cal.diy 的权限系统采用分层架构设计，主要包含以下核心组件：

- **认证中间件** (`sessionMiddleware.ts`): 负责用户身份验证和角色检查
- **权限守卫** (Guards): 如 `EventTypeOwnershipGuard`、`BookingPbacGuard` 等
- **访问服务** (Access Services): 如 `EventTypeAccessService`、`BookingAccessService`
- **数据模型关系**: 通过 Prisma schema 定义的用户、活动类型、预约等实体关系

### 1.2 关键文件位置

| 组件 | 文件路径 |
|------|----------|
| 认证中间件 | `packages/trpc/server/middlewares/sessionMiddleware.ts` |
| 活动类型权限守卫 | `apps/api/v2/src/modules/event-types/guards/event-type-ownership.guard.ts` |
| 预约权限守卫 | `apps/api/v2/src/platform/bookings/2024-08-13/guards/booking-pbac.guard.ts` |
| 活动类型服务 | `apps/api/v2/src/platform/event-types/event-types_2024_06_14/services/event-types.service.ts` |
| 公开事件获取 | `packages/features/eventtypes/lib/getPublicEvent.ts` |

---

## 2. 数据库模型关系

### 2.1 用户与活动类型关系

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│    User     │         │   EventType  │         │    Team     │
├─────────────┤         ├──────────────┤         ├─────────────┤
│ id          │◄────────┤ userId (FK)  │         │ id          │
│ role        │         │ teamId (FK)  │────────►│             │
│ username    │         │ hidden       │         │ isPrivate   │
│ locked      │         │ bookingRequires │      │ parentId    │
└─────────────┘         │ Authentication │       └──────┬──────┘
       │                └──────────────┘              │
       │                      ▲                         │
       │                      │                         │
       ▼                      │                         ▼
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│ Membership  │         │     Host     │         │  Profile    │
├─────────────┤         ├──────────────┤         ├─────────────┤
│ userId      │         │ userId (FK)  │         │ userId      │
│ teamId      │         │ eventTypeId  │         │ organizationId│
│ role        │◄────────┤ memberId (FK)│         │ username    │
│ accepted    │         │ isFixed      │         └─────────────┘
└─────────────┘         └──────────────┘
```

### 2.2 关键模型字段说明

#### User 模型 (schema.prisma:401-522)

```typescript
model User {
  id                  Int                  @id @default(autoincrement())
  username            String?
  email               String
  role                UserPermissionRole   @default(USER)  // USER | ADMIN
  locked              Boolean              @default(false)
  organizationId      Int?
  organization        Team?                @relation("scope", fields: [organizationId], references: [id], onDelete: SetNull)
  
  // 关系
  eventTypes          EventType[]          @relation("user_eventtype")
  ownedEventTypes     EventType[]          @relation("owner")
  teams               Membership[]
  hosts               Host[]
  profiles            Profile[]
}
```

#### EventType 模型 (schema.prisma:156-306)

```typescript
model EventType {
  id                Int     @id @default(autoincrement())
  title             String
  slug              String
  hidden            Boolean @default(false)
  
  // 访问控制字段
  bookingRequiresAuthentication  Boolean  @default(false)
  
  // 所属关系
  users             User[]  @relation("user_eventtype")
  owner             User?   @relation("owner", fields: [userId], references: [id], onDelete: Cascade)
  userId            Int?
  team              Team?   @relation(fields: [teamId], references: [id], onDelete: Cascade)
  teamId            Int?
  
  // 多主机支持
  hosts             Host[]
  schedulingType    SchedulingType?  // ROUND_ROBIN | COLLECTIVE | MANAGED
}
```

#### Membership 模型 (schema.prisma:744-765)

```typescript
model Membership {
  id              Int               @id @default(autoincrement())
  teamId          Int
  userId          Int
  accepted        Boolean           @default(false)
  role            MembershipRole    // MEMBER | ADMIN | OWNER
  customRoleId    String?
  team            Team              @relation(fields: [teamId], references: [id], onDelete: Cascade)
  user            User              @relation(fields: [userId], references: [id], onDelete: Cascade)
}
```

---

## 3. 角色系统设计

### 3.1 角色层次结构

```
                    ┌───────────────────┐
                    │  系统级角色        │
                    ├───────────────────┤
                    │  ADMIN (全局管理员) │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │  组织级角色        │
                    ├───────────────────┤
                    │  OWNER (组织拥有者) │
                    │  ADMIN (组织管理员) │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │  团队级角色        │
                    ├───────────────────┤
                    │  OWNER (团队拥有者) │
                    │  ADMIN (团队管理员) │
                    │  MEMBER (团队成员)  │
                    └─────────┬─────────┘
                              │
                    ┌─────────▼─────────┐
                    │  基础角色          │
                    ├───────────────────┤
                    │  USER (普通用户)    │
                    │  外部访客 (未登录)  │
                    └───────────────────┘
```

### 3.2 角色定义与能力范围

| 角色类型 | 角色枚举 | 定义位置 | 能力范围 |
|----------|----------|----------|----------|
| 系统级 | `USER` | `UserPermissionRole` (schema.prisma:375-378) | 普通注册用户，可创建和管理自己的活动类型 |
| 系统级 | `ADMIN` | `UserPermissionRole` | 全局管理员，可访问系统管理功能，通过 `isAdminMiddleware` 验证 |
| 组织/团队级 | `MEMBER` | `MembershipRole` (schema.prisma:738-742) | 团队普通成员，可被分配为活动类型的主机 |
| 组织/团队级 | `ADMIN` | `MembershipRole` | 团队管理员，可管理团队设置和成员 |
| 组织/团队级 | `OWNER` | `MembershipRole` | 团队/组织拥有者，拥有最高权限 |

### 3.3 中间件角色验证

#### 系统管理员验证 (`sessionMiddleware.ts:26-32`)

```typescript
export const isAdminMiddleware = isAuthed.unstable_pipe(({ ctx, next }) => {
  const { user } = ctx;
  if (user?.role !== "ADMIN") {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({ ctx: { user: user } });
});
```

#### 组织管理员验证 (`sessionMiddleware.ts:34-41`)

```typescript
export const isOrgAdminMiddleware = isAuthed.unstable_pipe(({ ctx, next }) => {
  const { user } = ctx;
  if (!user?.organization?.isOrgAdmin) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({ ctx: { user: user } });
});
```

---

## 4. 用户创建活动类型的访问控制

### 4.1 访问控制边界

用户创建活动类型时，系统实施以下访问控制边界：

#### 4.1.1 身份验证边界

- **必须通过认证**: 使用 `isAuthed` 中间件 (`sessionMiddleware.ts:7-24`)
- **账户未锁定**: 通过 `exclude-locked-users` 扩展排除锁定用户

#### 4.1.2 所有权边界

活动类型创建后，建立明确的所有权关系：

```
┌──────────────┐     1:N      ┌──────────────┐
│     User     │◄─────────────┤  EventType   │
├──────────────┤              ├──────────────┤
│ id: 123      │              │ id: 1        │
│ username:    │              │ userId: 123  │◄── 明确的所属关系
│ "john"       │              │ slug: "30min"│
└──────────────┘              └──────────────┘
```

#### 4.1.3 资源独占边界

- **Slug 唯一性**: 同一用户下的活动类型 slug 必须唯一 (`checkCanCreateEventType`)
- **时间表所有权**: 活动类型使用的时间表必须属于当前用户 (`checkUserOwnsSchedule`)

### 4.2 创建流程中的权限检查

#### 4.2.1 创建前检查 (`event-types.service.ts:105-111`)

```typescript
async checkCanCreateEventType(userId: number, body: InputEventTransformed_2024_06_14) {
  // 检查 slug 唯一性
  const existsWithSlug = await this.eventTypesRepository.getUserEventTypeBySlug(userId, body.slug);
  if (existsWithSlug) {
    throw new BadRequestException("User already has an event type with this slug.");
  }
  // 检查时间表所有权
  await this.checkUserOwnsSchedule(userId, body.scheduleId);
}
```

#### 4.2.2 时间表所有权检查 (`event-types.service.ts:368-378`)

```typescript
async checkUserOwnsSchedule(userId: number, scheduleId: number | null | undefined) {
  if (!scheduleId) {
    return;
  }
  const schedule = await this.schedulesRepository.getScheduleByIdAndUserId(scheduleId, userId);
  if (!schedule) {
    throw new NotFoundException(`User with ID=${userId} does not own schedule with ID=${scheduleId}`);
  }
}
```

#### 4.2.3 预订字段验证 (`event-types.service.ts:113-121`)

```typescript
checkHasUserAccessibleEmailBookingField(bookingFields: (SystemField | CustomField)[]) {
  const emailField = bookingFields.find((field) => field.type === "email" && field.name === "email");
  const isEmailFieldRequiredAndVisible = emailField?.required && !emailField?.hidden;
  if (!isEmailFieldRequiredAndVisible) {
    throw new BadRequestException(
      "checkIsEmailUserAccessible - Email booking field must be required and visible"
    );
  }
}
```

### 4.3 组织级限制

组织管理员可以通过 `OrganizationSettings` 限制普通用户创建活动类型：

```typescript
model OrganizationSettings {
  // ...
  lockEventTypeCreationForUsers  Boolean  @default(false)
  // 设置为 true 时，普通用户无法创建活动类型
}
```

---

## 5. 外部访客预约权限验证

### 5.1 公开事件访问机制

外部访客（未登录用户）访问活动类型时，系统通过以下机制验证权限：

#### 5.1.1 公开事件获取流程

```
外部访客请求
      │
      ▼
┌─────────────────┐
│  publicProcedure │ (无需认证)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  eventHandler   │
│  (publicViewer) │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│  EventRepository.getPublicEvent │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  权限检查逻辑               │
│  ├─ 检查 hidden 字段        │
│  ├─ 检查 team.isPrivate     │
│  └─ 检查 bookingRequiresAuthentication │
└─────────────────────────────┘
```

#### 5.1.2 隐藏活动类型检查 (`event-types.service.ts:141-143`)

```typescript
if (eventType.hidden && params.authUser?.id !== user.id) {
  return null;  // 非所有者无法访问隐藏的活动类型
}
```

### 5.2 活动类型可见性控制

#### 5.2.1 可见性矩阵

| 活动类型属性 | 外部访客 | 登录用户（非所有者） | 所有者 |
|-------------|---------|---------------------|--------|
| `hidden: false` + 公开团队 | ✅ 可见 | ✅ 可见 | ✅ 可见 |
| `hidden: true` | ❌ 不可见 | ❌ 不可见 | ✅ 可见 |
| `team.isPrivate: true` | ⚠️ 受限 | ⚠️ 受限 | ✅ 可见 |
| `bookingRequiresAuthentication: true` | ⚠️ 可查看但需登录预订 | ✅ 可预订 | ✅ 可预订 |

#### 5.2.2 私有团队成员可见性检查 (`getPublicEvent.ts:534-556`)

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
  // 检查父组织权限
  if (!canViewPrivateTeamMembers && event.team?.parentId) {
    canViewPrivateTeamMembers = await permissionCheckService.checkPermission({
      userId: currentUserId,
      teamId: event.team.parentId,
      permission: "team.read",
      fallbackRoles: [MembershipRole.ADMIN, MembershipRole.OWNER],
    });
  }
}

if (event.team?.isPrivate && !canViewPrivateTeamMembers) {
  users = [];  // 私有团队对非成员隐藏成员列表
}
```

### 5.3 预订认证要求

#### 5.3.1 预订认证字段

活动类型可设置 `bookingRequiresAuthentication` 字段要求预订者登录：

```typescript
model EventType {
  // ...
  bookingRequiresAuthentication  Boolean  @default(false)
  // true = 必须登录才能预订
  // false = 访客无需登录即可预订
}
```

#### 5.3.2 预订权限守卫 (`booking-pbac.guard.ts`)

对于需要认证的预订操作，使用 `BookingPbacGuard` 进行权限验证：

```typescript
@Injectable()
export class BookingPbacGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;
    const bookingUid = request.params.bookingUid;

    if (!user) {
      throw new UnauthorizedException();  // 未登录用户被拒绝
    }

    const hasAccess = await this.bookingAccessService.doesUserIdHaveAccessToBooking({
      userId: user.id,
      bookingUid,
    });

    if (!hasAccess) {
      throw new ForbiddenException(
        `User with id=${user.id} does not have access to booking with uid=${bookingUid}`
      );
    }

    return true;
  }
}
```

---

## 6. 活动类型所有权验证

### 6.1 所有权守卫 (`event-type-ownership.guard.ts`)

对于需要修改活动类型的操作（更新、删除），使用 `EventTypeOwnershipGuard`：

```typescript
@Injectable()
export class EventTypeOwnershipGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user as ApiAuthGuardUser | undefined;
    const eventTypeIdParam = request.params?.eventTypeId;

    if (!user) {
      throw new ForbiddenException("EventTypeOwnershipGuard - No user associated with the request.");
    }

    const eventTypeId = Number(eventTypeIdParam);
    // 通过服务层验证所有权
    const eventType = await this.eventTypesService.getUserEventType(user.id, eventTypeId);
    if (!eventType) {
      throw new NotFoundException(`Event type with id ${eventTypeId} not found`);
    }

    return true;
  }
}
```

### 6.2 服务层所有权检查 (`event-types.service.ts:362-366`)

```typescript
checkUserOwnsEventType(userId: number, eventType: Pick<EventType, "id" | "userId">) {
  if (userId !== eventType.userId) {
    throw new ForbiddenException(`User with ID=${userId} does not own event type with ID=${eventType.id}`);
  }
}
```

### 6.3 更新和删除操作的权限流程

#### 更新流程 (`event-types.service.ts:297-335`)

```typescript
async updateEventType(
  eventTypeId: number,
  body: Partial<InputEventTransformed_2024_06_14>,
  user: UserWithProfile
) {
  // 1. 验证预订字段
  if (body.bookingFields) {
    this.checkHasUserAccessibleEmailBookingField(body.bookingFields);
  }
  // 2. 验证所有权和时间表
  await this.checkCanUpdateEventType(user.id, eventTypeId, body.scheduleId);
  // 3. 执行更新
  // ...
}
```

#### 删除流程 (`event-types.service.ts:351-360`)

```typescript
async deleteEventType(eventTypeId: number, userId: number) {
  const existingEventType = await this.eventTypesRepository.getEventTypeById(eventTypeId);
  if (!existingEventType) {
    throw new NotFoundException(`Event type with ID=${eventTypeId} does not exist.`);
  }
  // 验证所有权
  this.checkUserOwnsEventType(userId, existingEventType);
  return this.eventTypesRepository.deleteEventType(eventTypeId);
}
```

---

## 7. 多主机团队活动类型权限

### 7.1 团队活动类型模型

团队活动类型支持多主机（Host）模式，通过 `Host` 模型关联用户和活动类型：

```typescript
model Host {
  user             User          @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId           Int
  eventType        EventType     @relation(fields: [eventTypeId], references: [id], onDelete: Cascade)
  eventTypeId      Int
  isFixed          Boolean       @default(false)
  priority         Int?
  weight           Int?
  schedule         Schedule?     @relation(fields: [scheduleId], references: [id])
  scheduleId       Int?
  member           Membership?   @relation(fields: [memberId], references: [id], onDelete: Cascade)
  memberId         Int?
  
  @@id([userId, eventTypeId])
}
```

### 7.2 调度类型 (SchedulingType)

```typescript
enum SchedulingType {
  ROUND_ROBIN  @map("roundRobin")  // 轮询分配
  COLLECTIVE   @map("collective")   // 集体会议（所有主机都参加）
  MANAGED      @map("managed")      // 托管模式
}
```

### 7.3 团队活动类型权限边界

```
┌─────────────────────────────────────────────────────────────┐
│                    团队活动类型权限模型                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐      ┌──────────────┐      ┌───────────┐ │
│  │   Owner     │─────►│  EventType   │◄─────│   Hosts   │ │
│  │  (创建者)    │      │  (团队活动)   │      │  (多主机)  │ │
│  └─────────────┘      └──────────────┘      └───────────┘ │
│         │                    │                    │        │
│         │                    │                    │        │
│         ▼                    ▼                    ▼        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐│
│  │ 完全控制权   │    │ 团队管理员   │    │ 仅限主机权限  ││
│  │ - 创建/删除  │    │ - 管理设置   │    │ - 查看预订   ││
│  │ - 修改设置   │    │ - 管理成员   │    │ - 管理自己   ││
│  │ - 管理主机   │    │              │    │   的可用性   ││
│  └──────────────┘    └──────────────┘    └──────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. 权限检查总结

### 8.1 关键权限检查点速查表

| 操作 | 所需权限 | 检查位置 | 关键验证 |
|------|---------|----------|----------|
| 创建活动类型 | 已认证用户 | `checkCanCreateEventType` | slug 唯一性、时间表所有权 |
| 查看自己的活动类型 | 已认证用户 | `getUserEventTypes` | `userId` 匹配 |
| 查看公开活动类型 | 任何人 | `getPublicEvent` | `hidden=false`、团队可见性 |
| 更新活动类型 | 所有者 | `checkUserOwnsEventType` | `userId === eventType.userId` |
| 删除活动类型 | 所有者 | `checkUserOwnsEventType` | `userId === eventType.userId` |
| 访问预订详情 | 相关用户 | `BookingPbacGuard` | `doesUserIdHaveAccessToBooking` |
| 系统管理操作 | ADMIN 角色 | `isAdminMiddleware` | `user.role === "ADMIN"` |
| 组织管理操作 | Org Admin | `isOrgAdminMiddleware` | `user.organization.isOrgAdmin` |

### 8.2 安全设计原则

1. **最小权限原则**: 默认情况下，用户只能访问自己的资源
2. **显式所有权**: 通过 `userId`、`owner` 关系明确资源归属
3. **分层验证**: 中间件 → 守卫 → 服务层 → 数据库层多层验证
4. **隐私保护**: 敏感数据通过 `select` 而非 `include` 查询
5. **错误模糊化**: 权限不足时返回 `NotFound` 而非 `Forbidden`，避免信息泄露

---

## 9. 附录：核心代码引用

### 9.1 文件索引

| 功能模块 | 文件路径 | 行号范围 |
|----------|----------|----------|
| 认证中间件 | `packages/trpc/server/middlewares/sessionMiddleware.ts` | 1-41 |
| 活动类型服务 | `apps/api/v2/src/platform/event-types/event-types_2024_06_14/services/event-types.service.ts` | 1-379 |
| 活动类型所有权守卫 | `apps/api/v2/src/modules/event-types/guards/event-type-ownership.guard.ts` | 1-42 |
| 预约权限守卫 | `apps/api/v2/src/platform/bookings/2024-08-13/guards/booking-pbac.guard.ts` | 1-59 |
| 公开事件获取 | `packages/features/eventtypes/lib/getPublicEvent.ts` | 1-779 |
| 数据库模型 | `packages/prisma/schema.prisma` | 1-800+ |

### 9.2 关键枚举定义

```typescript
// 用户权限角色 (系统级)
enum UserPermissionRole {
  USER
  ADMIN
}

// 团队成员角色
enum MembershipRole {
  MEMBER
  ADMIN
  OWNER
}

// 调度类型
enum SchedulingType {
  ROUND_ROBIN
  COLLECTIVE
  MANAGED
}
```
