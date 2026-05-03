# Cal.diy 事件类型权限机制分析报告

> 分析日期：2026-05-03
> 分析范围：事件类型归属者权限、预约访客权限、前后端数据一致性、权限管理架构、事件类型模型影响、可见性变更处理

---

## 目录

1. [事件类型数据模型](#1-事件类型数据模型)
2. [归属者权限判断机制](#2-归属者权限判断机制)
3. [预约访客权限判断逻辑](#3-预约访客权限判断逻辑)
4. [前后端数据一致性](#4-前后端数据一致性)
5. [预约权限管理架构](#5-预约权限管理架构)
6. [事件类型模型对权限变更的影响](#6-事件类型模型对权限变更的影响)
7. [可见性变更后原有预约的处理](#7-可见性变更后原有预约的处理)
8. [关键代码位置索引](#8-关键代码位置索引)

---

## 1. 事件类型数据模型

### 1.1 核心模型结构

事件类型（EventType）是 Cal.diy 中预约配置的核心模型，其权限相关字段定义在 `packages/prisma/schema.prisma:156-306`：

```prisma
model EventType {
  id                Int     @id @default(autoincrement())
  title             String
  slug              String
  hidden            Boolean @default(false)  // 可见性控制
  
  // 归属关系
  users             User[]  @relation("user_eventtype")  // 多对多
  owner             User?   @relation("owner", fields: [userId], references: [id])
  userId            Int?
  team              Team?   @relation(fields: [teamId], references: [id])
  teamId            Int?
  
  // 关联关系
  hosts             Host[]
  hashedLink        HashedLink[]
  bookings          Booking[]
  
  // 预订控制
  disableCancelling     Boolean? @default(false)
  disableRescheduling   Boolean? @default(false)
  minimumRescheduleNotice  Int?
  // ... 其他字段
}
```

### 1.2 关联模型

#### Host 模型（主持人）
```prisma
model Host {
  userId           Int
  eventTypeId      Int
  user             User      @relation(...)
  eventType        EventType @relation(...)
  isFixed          Boolean   @default(false)
  // ...
  @@id([userId, eventTypeId])
}
```

#### HashedLink 模型（私有链接）
```prisma
model HashedLink {
  id            Int       @id @default(autoincrement())
  link          String    @unique
  eventTypeId   Int
  eventType     EventType @relation(...)
  expiresAt     DateTime?
  maxUsageCount Int       @default(1)
  usageCount    Int       @default(0)
}
```

#### Booking 模型（预订）
```prisma
model Booking {
  id              Int           @id @default(autoincrement())
  uid             String        @unique
  userId          Int?          // 组织者（冗余存储）
  userPrimaryEmail String?      // 组织者邮箱（冗余存储）
  eventTypeId     Int?          // 关联事件类型
  eventType       EventType?    @relation(...)
  attendees       Attendee[]    // 参与者
  status          BookingStatus @default(ACCEPTED)
  // ...
}
```

---

## 2. 归属者权限判断机制

### 2.1 权限判断的多维度

事件类型的归属者权限判断是**多维度、层次化**的，主要在两个 API 层实现：

#### API v2 层（平台 API）

**核心服务**：`apps/api/v2/src/modules/event-types/services/event-type-access.service.ts:19-48`

```typescript
async userIsEventTypeAdminOrOwner(authUser: ApiAuthGuardUser, eventType: EventType): Promise<boolean> {
  const authUserId = authUser.id;
  const eventTypeId = eventType.id;
  const teamId = eventType.teamId;
  const eventTypeOwnerId = eventType.userId || null;

  // 1. 系统管理员直接放行
  if (authUser.isSystemAdmin) return true;

  // 2. 直接所有者（owner）
  if (eventTypeOwnerId === authUserId) return true;

  // 3. 主持人或被分配用户（host/assigned）
  if (eventTypeId) {
    const isHostOrAssigned = await this.isUserHostOrAssignedToEventType(authUserId, eventTypeId);
    if (isHostOrAssigned) return true;
  }

  // 4. 团队管理员或父组织管理员
  if (teamId) {
    const isTeamOrParentOrgAdmin = await this.isUserTeamAdminOrParentOrgAdmin(authUserId, teamId);
    if (isTeamOrParentOrgAdmin) return true;
  }

  // 5. 事件类型所有者的组织管理员
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

**守卫层**：`apps/api/v2/src/modules/event-types/guards/event-type-ownership.guard.ts:17-41`

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  // ... 解析参数
  const eventType = await this.eventTypesService.getUserEventType(user.id, eventTypeId);
  if (!eventType) {
    // 如果不是所有者或不存在，返回 404（而非 403）
    throw new NotFoundException(`Event type with id ${eventTypeId} not found`);
  }
  return true;
}
```

#### tRPC 层（Web 应用）

**核心查询**：`packages/features/eventtypes/repositories/eventTypeRepository.ts:795-821`

```typescript
async findById({ id, userId }: { id: number; userId: number }) {
  // 先获取用户所在的所有团队
  const userTeamIds = await MembershipRepository.findUserTeamIds({ userId });

  return await this.prismaClient.eventType.findFirst({
    where: {
      AND: [
        {
          OR: [
            // 维度 1: 在 users 多对多关系中
            { users: { some: { id: userId } } },
            // 维度 2: 团队事件且用户在该团队中
            { AND: [{ teamId: { not: null } }, { teamId: { in: userTeamIds } }] },
            // 维度 3: 直接所有者（userId 字段）
            { userId: userId },
          ],
        },
        { id },
      ],
    },
    select: CompleteEventTypeSelect,
  });
}
```

### 2.2 团队角色权限层次

**角色定义**：`packages/prisma/schema.prisma` 中的 `MembershipRole` enum

**权限工具**：`packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts:20-90`

```typescript
// 角色权限层次（数值越大权限越高）
const MEMBERSHIP_HIERARCHY: Record<MembershipRole, number> = {
  [MembershipRole.MEMBER]: 1,   // 普通成员
  [MembershipRole.ADMIN]: 2,    // 管理员
  [MembershipRole.OWNER]: 3,     // 所有者
};

// 团队权限判断（支持 PBAC + 角色回退）
export async function getTeamPermissions(
  userId: number,
  teamId: number,
  effectiveRole: MembershipRole
): Promise<TeamPermissions> {
  try {
    // 优先使用 PBAC（基于策略的访问控制）
    const permissions = await getResourcePermissions({
      userId, teamId, resource: Resource.EventType,
      userRole: effectiveRole,
      fallbackRoles: {
        read: { roles: [ADMIN, OWNER, MEMBER] },
        create: { roles: [ADMIN, OWNER] },
        update: { roles: [ADMIN, OWNER] },
        delete: { roles: [ADMIN, OWNER] },
      },
    });
    return permissions;
  } catch (error) {
    // PBAC 失败时回退到基于角色的判断
    return getFallbackPermissions(effectiveRole);
  }
}

// 回退的角色权限
function getFallbackPermissions(role: MembershipRole): TeamPermissions {
  const isAdminOrOwner = role === ADMIN || role === OWNER;
  const isMember = role === MEMBER;
  return {
    canRead: isAdminOrOwner || isMember,    // 所有成员可读
    canCreate: isAdminOrOwner,              // 管理员/所有者可创建
    canEdit: isAdminOrOwner,                // 管理员/所有者可编辑
    canDelete: isAdminOrOwner,              // 管理员/所有者可删除
  };
}
```

### 2.3 权限判断流程图

```
用户请求访问事件类型
        │
        ▼
┌───────┴───────┐
│ 是系统管理员？ │──是──► 允许访问
└───────┬───────┘
        │否
        ▼
┌───────┴───────┐
│ 是直接所有者？ │──是──► 允许访问
│ (userId 匹配) │
└───────┬───────┘
        │否
        ▼
┌───────┴───────┐
│ 是主持人或   │──是──► 允许访问
│ 被分配用户？  │
└───────┬───────┘
        │否
        ▼
┌───────┴───────┐
│ 是团队管理？  │──是──► 允许访问
│ 或父组织管理？│
└───────┬───────┘
        │否
        ▼
┌───────┴───────┐
│ 是事件类型   │──是──► 允许访问
│ 所有者的组织  │
│ 管理员？     │
└───────┬───────┘
        │否
        ▼
    拒绝访问（返回 404）
```

---

## 3. 预约访客权限判断逻辑

### 3.1 可见性控制机制

预约访客的权限主要通过**可见性过滤**和**私有链接机制**实现。

#### hidden 字段过滤

**公共事件列表查询**：`packages/features/eventtypes/lib/getEventTypesPublic.ts:12-21`

```typescript
export async function getEventTypesPublic(userId: number) {
  // 1. 获取所有事件类型（包括隐藏的）
  const eventTypesWithHidden = await getEventTypesWithHiddenFromDB(userId);
  
  // 2. 过滤掉 hidden = true 的事件类型
  const eventTypesRaw = eventTypesWithHidden.filter((evt) => !evt.hidden);

  return eventTypesRaw.map((eventType) => ({
    ...eventType,
    metadata: EventTypeMetaDataSchema.parse(eventType.metadata || {}),
    descriptionAsSafeHTML: markdownToSafeHTML(eventType.description),
  }));
}
```

#### 公共事件查询（getPublicEvent）

**核心查询**：`packages/features/eventtypes/lib/getPublicEvent.ts:430-458`

```typescript
// 查询公共事件类型时，通过 userId/teamId 过滤，但不直接过滤 hidden
// 这样设计是为了支持私有链接可以访问隐藏的事件类型

let event = await prisma.eventType.findFirst({
  where: {
    slug: eventSlug,
    ...usersOrTeamQuery,  // 通过用户名/团队名过滤
  },
  select: getPublicEventSelect(fetchAllUsers),
});

// 如果没找到，尝试 platform org user 事件
if (!event && !orgQuery) {
  event = await prisma.eventType.findFirst({
    where: {
      slug: eventSlug,
      users: { some: { username, isPlatformManaged: false, ... } },
    },
    select: getPublicEventSelect(fetchAllUsers),
  });
}

if (!event) return null;
```

### 3.2 私有链接（HashedLink）机制

私有链接是绕过 `hidden` 限制的关键机制，允许特定访客访问隐藏的事件类型。

#### 链接验证流程

**服务端验证**：`apps/web/lib/d/[link]/[slug]/getServerSideProps.tsx:37-51`

```typescript
// 使用中心化验证逻辑
const hashedLinkService = new HashedLinkService();
try {
  await hashedLinkService.validate(link);
} catch (_error) {
  // 链接过期、无效或不存在
  return notFound;
}

// 验证通过后，获取完整数据
const hashedLink = await hashedLinkService.findLinkWithDetails(link);
if (!hashedLink) {
  return notFound;
}
```

**核心验证服务**：`packages/features/hashedLink/lib/service/HashedLinkService.ts:96-133`

```typescript
async validate(linkId: string) {
  if (!linkId || typeof linkId !== "string") {
    throw new Error("Invalid link ID");
  }

  // 1. 查询链接数据（包含过期时间、使用次数等）
  const hashedLink = await this.hashedLinkRepository.findLinkWithValidationData(linkId);

  if (!hashedLink) {
    throw new Error(ErrorCode.PrivateLinkExpired);
  }

  // 2. 验证过期状态（时间 + 使用次数）
  validateHashedLinkData(hashedLink);

  return hashedLink;
}

async validateAndIncrementUsage(linkId: string) {
  const hashedLink = await this.validate(linkId);
  // 时间过期的链接不需要增加使用次数
  if (hashedLink.expiresAt) return hashedLink;

  // 增加使用计数（带并发安全）
  if (hashedLink.maxUsageCount && hashedLink.maxUsageCount > 0) {
    try {
      await this.hashedLinkRepository.incrementUsage(hashedLink.id, hashedLink.maxUsageCount);
    } catch (e) {
      logger.error("Error incrementing usage for hashed link", safeStringify(e));
      throw new Error(ErrorCode.PrivateLinkExpired);
    }
  }
  return hashedLink;
}
```

#### 过期验证工具

**验证逻辑**：`packages/lib/hashedLinksUtils.ts:62-129`

```typescript
// 时间过期检查
function hasExpiryTimePassed(expiresAt: Date | string | null, timezone?: string | null): boolean {
  if (!expiresAt) return false;
  if (timezone) {
    const now = dayjs().tz(timezone);
    const expiration = dayjs(expiresAt).tz(timezone);
    return expiration.isBefore(now);
  }
  // ...
}

// 使用次数过期检查
export function isUsageBasedExpired(usageCount: number, maxUsageCount?: number | null): boolean {
  if (!maxUsageCount || maxUsageCount <= 0) return false;
  return usageCount >= maxUsageCount;
}

// 综合验证
export function validateHashedLinkData(linkData: HashedLinkData): void {
  if (isTimeBasedExpired(linkData.expiresAt, linkData.eventType)) {
    throw new Error(ErrorCode.PrivateLinkExpired);
  }
  if (isUsageBasedExpired(linkData.usageCount, linkData.maxUsageCount)) {
    throw new Error(ErrorCode.PrivateLinkExpired);
  }
}
```

### 3.3 访客权限流程图

```
访客尝试访问预约页面
        │
        ▼
┌───────┴───────┐
│ 访问路径类型？ │
└───────┬───────┘
        │
        ├──────────────┐
        │              │
        ▼              ▼
   公共路径         私有链接路径
   /user/slug       /d/{link}/slug
        │              │
        ▼              ▼
┌─────────────┐  ┌─────────────┐
│ 查询事件类型 │  │ 验证私有链接 │
│ 过滤 hidden │  │ (时间+次数)  │
│ = false    │  └──────┬──────┘
└──────┬──────┘         │
       │                │
       ▼                ▼
   无结果？         验证通过？
       │                │
       ├─否─┐           ├─否─┐
       │    │           │    │
       ▼    │           ▼    │
   显示事件 │        返回 404 │
   类型列表 │                │
            │                │
       ┌────┘                │
       │                     │
       ▼                     ▼
   返回 404            获取事件类型
                        (可绕过 hidden)
```

---

## 4. 前后端数据一致性

### 4.1 数据流向架构

Cal.diy 采用**服务端渲染（SSR）** + **服务端唯一可信源**的架构，确保前后端数据一致性。

#### 服务端数据获取流程

**私有链接页面**：`apps/web/lib/d/[link]/[slug]/getServerSideProps.tsx`

```typescript
export async function getServerSideProps(context: GetServerSidePropsContext) {
  // 1. 获取会话（如果有）
  const session = await getServerSession({ req: context.req });
  
  // 2. 验证私有链接
  const hashedLinkService = new HashedLinkService();
  try {
    await hashedLinkService.validate(link);
  } catch (_error) {
    return { notFound: true };
  }
  
  // 3. 获取事件类型数据（公共数据）
  const eventData = await EventRepository.getPublicEvent(
    {
      username: name,
      eventSlug: slug,
      isTeamEvent,
      org,
      fromRedirectOfNonOrgLink: context.query.orgRedirection === "true",
    },
    session?.user?.id
  );

  if (!eventData) {
    return { notFound: true };
  }

  // 4. 将数据传递给前端
  return {
    props: {
      eventData,
      entity: eventData.entity,
      duration: getMultipleDurationValue(...),
      booking,
      user: name,
      slug,
      isBrandingHidden: hideBranding,
      isTeamEvent,
      hashedLink: hashedLink?.link,
      durationConfig: eventData.metadata?.multipleDuration ?? [],
      useApiV2: false,
    },
  };
}
```

**公共页面类似流程**：通过 `getPublicEvent` 获取数据，服务端判断是否有权限，无权限则返回 `notFound: true`。

#### 前端组件接收

**Booker 组件**：`apps/web/modules/bookings/components/Booker.tsx:55-90`

```typescript
const BookerComponent = ({
  username,
  eventSlug,
  hideBranding = false,
  entity,
  // ...
  event,      // 事件类型数据
  hashedLink, // 私有链接（如果有）
  // ...
}: BookerProps & WrappedBookerProps): JSX.Element | null => {
  // 前端只使用服务端传递的数据
  // 不进行独立的权限判断
  // ...
}
```

### 4.2 一致性保证机制

#### 1. 服务端唯一可信源

所有权限判断、可见性过滤都在服务端完成：

| 判断类型 | 服务端位置 | 前端行为 |
|---------|-----------|---------|
| 可见性过滤 | `getEventTypesPublic.filter(!evt.hidden)` | 只接收过滤后的数据 |
| 私有链接验证 | `HashedLinkService.validate()` | 不验证，只使用结果 |
| 事件类型权限 | `EventTypeRepository.findById()` | 无权限则页面 404 |
| 预订操作权限 | `RegularBookingService` | 调用 API，服务端验证 |

#### 2. SSR 时直接返回 404

如果用户没有权限访问某个事件类型，服务端在 `getServerSideProps` 中直接返回：

```typescript
return {
  notFound: true,
} as const;
```

这会触发 Next.js 的 404 页面，前端甚至不会加载 Booker 组件。

#### 3. API 层的权限守卫

即使前端绕过 SSR（如直接调用 API），API 层也有守卫保护：

**tRPC 层**：`packages/trpc/server/routers/viewer/eventTypes/_router.ts:104-113`

```typescript
delete: createEventPbacProcedure("eventType.delete", [ADMIN, OWNER])
  .input(ZDeleteInputSchema)
  .mutation(async ({ ctx, input }) => {
    const { deleteHandler } = await import("./delete.handler");
    return deleteHandler({ ctx, input });
  }),
```

**API v2 层**：`apps/api/v2/src/modules/event-types/guards/event-type-ownership.guard.ts`

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  // ... 验证用户身份和事件类型所有权
  const eventType = await this.eventTypesService.getUserEventType(user.id, eventTypeId);
  if (!eventType) {
    throw new NotFoundException(`Event type with id ${eventTypeId} not found`);
  }
  return true;
}
```

### 4.3 前后端数据对照

服务端返回的 `eventData` 包含前端判断按钮显隐所需的关键字段：

| 字段 | 来源 | 前端用途 |
|------|------|---------|
| `hidden` | EventType.hidden | 控制显示逻辑 |
| `disableRescheduling` | EventType.disableRescheduling | 改期按钮显隐 |
| `disableCancelling` | EventType.disableCancelling | 取消按钮显隐 |
| `requiresConfirmation` | EventType.requiresConfirmation | 确认流程控制 |
| `seatsPerTimeSlot` | EventType.seatsPerTimeSlot | 座位预订 UI |
| `metadata.multipleDuration` | EventType.metadata | 时长选择 |

**关键代码位置**：`packages/features/eventtypes/lib/getPublicEvent.ts:70-187`

```typescript
export const getPublicEventSelect = (fetchAllUsers: boolean) => {
  return {
    id: true,
    title: true,
    hidden: true,                    // 可见性
    disableCancelling: true,         // 取消控制
    disableRescheduling: true,       // 改期控制
    minimumRescheduleNotice: true,   // 改期提前时间
    allowReschedulingCancelledBookings: true,
    requiresConfirmation: true,       // 确认要求
    seatsPerTimeSlot: true,           // 座位
    metadata: true,                   // 扩展配置
    // ... 其他字段
  } satisfies Prisma.EventTypeSelect;
};
```

---

## 5. 预约权限管理架构

### 5.1 核心统一服务：BookingAccessService

Cal.diy **存在统一的预订访问控制服务** `BookingAccessService`，用于判断用户是否有权访问特定预订。

#### 核心服务定义

**位置**：`packages/features/bookings/services/BookingAccessService.ts`

```typescript
export class BookingAccessService {
  /**
   * Determines if a user has access to a booking based on:
   * 1. Being the booking organizer
   * 2. Being one of the hosts in a multi-host booking
   * 3. Being a team/org admin where the event type belongs (uses PBAC if enabled)
   * 4. Being an org admin where the booking organizer belongs (uses PBAC if enabled, for personal bookings)
   * 5. Being a team admin of any team the booking organizer belongs to (uses PBAC if enabled, for personal bookings)
   */
  async doesUserIdHaveAccessToBooking({
    userId,
    bookingUid,
    bookingId,
  }: {
    userId: number;
    bookingUid?: string;
    bookingId?: number;
  }): Promise<boolean> {
    // Case 1: User is the booking organizer
    if (userId === booking.userId) return true;

    // Case 2: User is one of the hosts
    if (this.isUserAHost(userId, booking)) return true;

    // Case 3: If booking has a teamId, check if user has access to team bookings
    if (booking.eventType?.teamId) {
      const hasAccess = await this.permissionCheckService.checkPermission({
        userId,
        teamId: booking.eventType.teamId,
        permission: "booking.readTeamBookings",
        fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
      });
      return hasAccess;
    }

    // Case 4: Check if user is admin of booking organizer's organization
    if (bookingOwner.organizationId) {
      const hasAccess = await this.permissionCheckService.checkPermission({
        userId,
        teamId: bookingOwner.organizationId,
        permission: "booking.readOrgBookings",
        fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
      });
      if (hasAccess) return true;
    }

    // Case 5: Check if user is admin of any team the booking organizer belongs to
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

  private isUserAHost(userId: number, booking: BookingForAccessCheck): boolean {
    // 检查用户是否是事件类型的主持人或在用户列表中
    // 同时验证用户邮箱是否在参与者列表中（或用户是预订组织者）
  }
}
```

#### 权限判断流程图

```
用户请求访问预订
        │
        ▼
┌───────┴───────┐
│ 是预订组织者？ │──是──► 允许访问
│ (userId 匹配) │
└───────┬───────┘
        │否
        ▼
┌───────┴───────┐
│ 是主持人之一？ │──是──► 允许访问
│ (hosts/users) │
└───────┬───────┘
        │否
        ▼
┌───────┴───────┐
│ 事件类型有    │──是──► 检查 booking.readTeamBookings 权限
│ teamId?       │        或角色是 OWNER/ADMIN
└───────┬───────┘
        │否
        ▼
┌───────┴───────┐
│ 预订组织者有  │──是──► 检查 booking.readOrgBookings 权限
│ organization? │
└───────┬───────┘
        │否
        ▼
┌───────┴───────┐
│ 是预订组织者  │──是──► 检查 booking.readTeamBookings 权限
│ 所属任何团队  │        (遍历所有团队)
│ 的管理员？    │
└───────┬───────┘
        │否
        ▼
    拒绝访问
```

### 5.2 BookingAccessService 的使用场景

#### 统一服务被多处使用

| 使用场景 | 代码位置 | 调用方式 |
|---------|---------|---------|
| **API v2 守卫层** | `apps/api/v2/src/platform/bookings/2024-08-13/guards/booking-pbac.guard.ts` | `BookingPbacGuard` 拦截器 |
| **tRPC 确认预订** | `packages/trpc/server/routers/viewer/bookings/confirm.handler.ts` | 确认前验证权限 |
| **tRPC 报告预订** | `packages/trpc/server/routers/viewer/bookings/reportBooking.handler.ts` | 报告前验证权限 |
| **tRPC 错误分配报告** | `packages/trpc/server/routers/viewer/bookings/hasWrongAssignmentReport.handler.ts` | 访问前验证 |
| **预订详情服务** | `packages/features/bookings/services/BookingDetailsService.ts` | `getBookingDetails()` 内部调用 |
| **platform-libraries 导出** | `packages/platform/libraries/index.ts:99` | 供 API v2 跨包使用 |

#### API v2 守卫层使用示例

**位置**：`apps/api/v2/src/platform/bookings/2024-08-13/guards/booking-pbac.guard.ts:25-58`

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
    const user = request.user;
    const bookingUid = request.params.bookingUid;

    const hasAccess =
      await this.bookingAccessService.doesUserIdHaveAccessToBooking({
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

#### BookingDetailsService 使用示例

**位置**：`packages/features/bookings/services/BookingDetailsService.ts:15-47`

```typescript
export class BookingDetailsService {
  private bookingRepo: BookingRepository;
  private bookingAccessService: BookingAccessService;

  constructor(prismaClient: PrismaClient) {
    this.bookingRepo = new BookingRepository(prismaClient);
    this.bookingAccessService = new BookingAccessService(prismaClient);
  }

  async getBookingDetails({ userId, bookingUid }: { userId: number; bookingUid: string }) {
    // 先使用统一服务验证权限
    const hasAccess = await this.bookingAccessService.doesUserIdHaveAccessToBooking({
      userId,
      bookingUid,
    });

    if (!hasAccess) {
      throw ErrorWithCode.Factory.Forbidden("You do not have permission to view this booking");
    }

    // 权限验证通过后才查询详细数据
    const booking = await this.bookingRepo.findByUidForDetails({ bookingUid });
    // ...
  }
}
```

### 5.3 权限管理分布概览

虽然存在统一的 `BookingAccessService`，但不同场景使用不同的权限判断方式：

```
┌─────────────────────────────────────────────────────────────────┐
│                      权限管理分布架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌────────────────────────────────────────────────────────────┐│
│  │              统一权限服务层 (BookingAccessService)          ││
│  │  ┌─────────────────────────────────────────────────────┐   ││
│  │  │ doesUserIdHaveAccessToBooking()                      │   ││
│  │  │  - Case 1: 预订组织者                                 │   ││
│  │  │  - Case 2: 主持人/被分配用户                          │   ││
│  │  │  - Case 3: 团队管理员 (booking.readTeamBookings)     │   ││
│  │  │  - Case 4: 组织管理员 (booking.readOrgBookings)       │   ││
│  │  │  - Case 5: 预订组织者所属任何团队的管理员              │   ││
│  │  └─────────────────────────────────────────────────────┘   ││
│  └────────────────────────────────────────────────────────────┘│
│                              │                                  │
│          ┌───────────────────┼───────────────────┐           │
│          ▼                   ▼                   ▼           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  tRPC 路由层  │    │  API v2 层   │    │  服务层      │  │
│  │ (Web 应用)   │    │ (平台 API)   │    │ (业务逻辑)   │  │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
│         │                    │                    │          │
│         ├────────────────────┼────────────────────┤          │
│         ▼                    ▼                    ▼          │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                   非统一权限场景                          │ │
│  │                                                          │ │
│  │  1. 预订列表查询 (UNION SQL)                             │ │
│  │     - get.handler.ts / getAllUserBookings.ts            │ │
│  │     - 使用复杂的 SQL UNION 直接过滤                      │ │
│  │                                                          │ │
│  │  2. 预订创建时的权限判断                                  │ │
│  │     - RegularBookingService                              │ │
│  │     - 检查事件类型可见性、配额限制等                      │ │
│  │                                                          │ │
│  │  3. 改期/取消操作的权限判断                              │ │
│  │     - 检查当前 EventType 的 disableRescheduling 等      │ │
│  │     - determineReschedulePreventionRedirect()            │ │
│  │                                                          │ │
│  │  4. 公共页面访问权限                                      │ │
│  │     - getEventTypesPublic() 过滤 hidden                  │ │
│  │     - HashedLinkService 验证私有链接                     │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4 各层次权限逻辑详解

#### 1. 使用 BookingAccessService 的场景

| 场景 | 代码位置 | 权限判断方式 |
|------|---------|-------------|
| 确认预订 | `confirm.handler.ts:166-169` | `doesUserIdHaveAccessToBooking(bookingId)` |
| 报告预订 | `reportBooking.handler.ts:21-24` | `doesUserIdHaveAccessToBooking(bookingUid)` |
| 错误分配报告 | `hasWrongAssignmentReport.handler.ts:21-24` | `doesUserIdHaveAccessToBooking(bookingUid)` |
| 获取预订详情 | `BookingDetailsService.ts:16-23` | `doesUserIdHaveAccessToBooking(bookingUid)` |
| API v2 预订操作 | `BookingPbacGuard.ts:44-48` | 守卫层拦截，统一验证 |

#### 2. 不使用 BookingAccessService 的场景

##### 场景 A：预订列表查询

**位置**：`packages/trpc/server/routers/viewer/bookings/get.handler.ts`

使用复杂的 SQL UNION 查询直接过滤，而不是通过统一服务：

```typescript
// 预订列表查询使用 7 种 UNION 条件
bookingQueries.push({
  query: kysely
    .selectFrom("Booking")
    .where("Booking.userId", "=", user.id),  // 条件 1: 组织者
  tables: ["Booking"],
});

bookingQueries.push({
  query: kysely
    .selectFrom("Booking")
    .innerJoin("Attendee", "Attendee.bookingId", "Booking.id")
    .where("Attendee.email", "=", user.email),  // 条件 2: 参与者
  tables: ["Booking", "Attendee"],
});

// ... 条件 3-7: 团队/组织管理员的扩展权限
```

**原因**：列表查询需要高性能，使用 SQL 直接过滤比逐行调用服务更高效。

##### 场景 B：预订创建时的权限判断

**位置**：`packages/features/bookings/lib/service/RegularBookingService.ts`

预订创建时的权限检查与访问权限不同：

```typescript
// 创建预订的流程中涉及的检查点：
// 1. checkIfBookerEmailIsBlocked() - 检查邮箱是否被拉黑
// 2. checkActiveBookingsLimitForBooker() - 检查活跃预订限制
// 3. validateBookingTimeIsNotOutOfBounds() - 验证时间有效性
// 4. getEventType() - 获取并验证事件类型（可见性）
// 5. ensureAvailableUsers() - 确保用户可用
```

**原因**：创建预订是访客发起的操作，不需要"访问已有预订"的权限，而是检查事件类型的可见性和预订配额。

##### 场景 C：改期/取消操作的权限判断

**位置**：`apps/web/lib/reschedule/[uid]/getServerSideProps.ts`

改期权限检查的是**当前 EventType 的设置**，而非用户是否有权访问预订：

```typescript
const reschedulePreventionRedirectUrl = determineReschedulePreventionRedirect({
  booking: {
    // ...
    eventType: {
      disableRescheduling: !!eventType?.disableRescheduling,
      allowReschedulingPastBookings: eventType.allowReschedulingPastBookings,
      minimumRescheduleNotice: eventType.minimumRescheduleNotice,
      // ...
    },
  },
  // ...
});
```

**原因**：改期/取消权限是"操作权限"，不同于"访问权限"。用户可能有权访问预订，但无权改期（如果 EventType 设置了 `disableRescheduling: true`）。

### 5.5 权限管理架构总结

#### 统一与分散并存

| 维度 | 统一服务 | 分散实现 |
|------|---------|---------|
| **预订访问权限** | ✅ `BookingAccessService.doesUserIdHaveAccessToBooking()` | - |
| **预订列表查询** | ❌ | ✅ SQL UNION 直接过滤（性能原因） |
| **预订创建权限** | ❌ | ✅ 独立逻辑（可见性、配额检查） |
| **改期/取消权限** | ❌ | ✅ 独立逻辑（操作权限 vs 访问权限） |
| **事件类型管理权限** | ❌ | ✅ `EventTypeAccessService`（独立服务） |

#### 设计考量

1. **分层权限模型**：
   - **访问权限**（能看吗？）→ 统一服务 `BookingAccessService`
   - **操作权限**（能改吗？）→ 分散检查（`disableRescheduling` 等）
   - **列表权限**（能看哪些？）→ SQL 直接过滤（性能优先）

2. **双 API 架构共享同一服务**：
   - tRPC 和 API v2 都使用相同的 `BookingAccessService`
   - 通过 `platform-libraries` 实现跨包共享

3. **性能与一致性的权衡**：
   - 单条预订访问 → 使用统一服务（一致性优先）
   - 列表查询 → SQL 直接过滤（性能优先）

### 5.2 各层次权限逻辑详解

#### 1. tRPC 路由层

**位置**：`packages/trpc/server/routers/viewer/eventTypes/`

**特点**：使用 `createEventPbacProcedure` 包装需要权限的操作

```typescript
// _router.ts:104-113
delete: createEventPbacProcedure("eventType.delete", [MembershipRole.ADMIN, MembershipRole.OWNER])
  .input(ZDeleteInputSchema)
  .mutation(async ({ ctx, input }) => {
    const { deleteHandler } = await import("./delete.handler");
    return deleteHandler({ ctx, input });
  }),
```

**PBAC 过程创建**：`packages/trpc/server/routers/viewer/eventTypes/util.ts`（需结合 `permissionUtils.ts`）

#### 2. API v2 守卫层

**位置**：`apps/api/v2/src/modules/event-types/guards/`

**EventTypeOwnershipGuard**：`event-type-ownership.guard.ts:17-41`

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  const request = context.switchToHttp().getRequest<Request>();
  const user = request.user as ApiAuthGuardUser | undefined;
  const eventTypeIdParam = request.params?.eventTypeId;

  if (!user) {
    throw new ForbiddenException("EventTypeOwnershipGuard - No user associated with the request.");
  }

  // 调用服务层验证用户是否有权访问该事件类型
  const eventType = await this.eventTypesService.getUserEventType(user.id, eventTypeId);
  if (!eventType) {
    // 使用 404 而非 403，避免暴露存在性
    throw new NotFoundException(`Event type with id ${eventTypeId} not found`);
  }

  return true;
}
```

**BookingPbacGuard**：`apps/api/v2/src/platform/bookings/2024-08-13/guards/booking-pbac.guard.ts`

#### 3. 服务层

**事件类型访问服务**：`apps/api/v2/src/modules/event-types/services/event-type-access.service.ts:19-48`

（详见 2.1 节的 `userIsEventTypeAdminOrOwner` 方法）

**预订服务**：`packages/features/bookings/lib/service/RegularBookingService.ts`

预订创建时的权限检查分散在多个子函数中：

```typescript
// 创建预订的流程中涉及的权限检查点：
// 1. checkIfBookerEmailIsBlocked() - 检查邮箱是否被拉黑
// 2. checkActiveBookingsLimitForBooker() - 检查活跃预订限制
// 3. validateBookingTimeIsNotOutOfBounds() - 验证时间有效性
// 4. getEventType() - 获取并验证事件类型
// 5. ensureAvailableUsers() - 确保用户可用
// ...
```

#### 4. 存储层（Repository）

**特点**：查询时直接内嵌权限过滤条件，确保只有有权限的数据被返回

**EventTypeRepository.findById**：`packages/features/eventtypes/repositories/eventTypeRepository.ts:795-821`

```typescript
async findById({ id, userId }: { id: number; userId: number }) {
  const userTeamIds = await MembershipRepository.findUserTeamIds({ userId });

  return await this.prismaClient.eventType.findFirst({
    where: {
      AND: [
        {
          OR: [
            { users: { some: { id: userId } } },           // 条件 1
            { AND: [{ teamId: { not: null } }, { teamId: { in: userTeamIds } }] },  // 条件 2
            { userId: userId },                          // 条件 3
          ],
        },
        { id },
      ],
    },
    // ...
  });
}
```

**BookingRepository.findBookingByUidAndUserId**：`packages/features/bookings/repositories/BookingRepository.ts:981-1040`

```typescript
async findBookingByUidAndUserId({ bookingUid, userId }: { bookingUid: string; userId: number }) {
  return await this.prismaClient.booking.findFirst({
    where: {
      uid: bookingUid,
      OR: [
        { userId: userId },  // 是组织者
        {
          eventType: { hosts: { some: { userId } } },  // 是主持人
        },
        {
          eventType: { users: { some: { id: userId } } },  // 在用户列表中
        },
        {
          eventType: {
            team: {
              members: {
                some: { userId, accepted: true, role: { in: ["ADMIN", "OWNER"] } },
              },
            },
          },
        },
        {
          eventType: {
            parent: {
              team: {
                members: {
                  some: { userId, accepted: true, role: { in: ["ADMIN", "OWNER"] } },
                },
              },
            },
          },
        },
      ],
    },
  });
}
```

### 5.3 分散式架构的原因

#### 1. 双 API 架构

项目同时维护两套 API 系统：

| API 类型 | 使用场景 | 权限实现位置 |
|---------|---------|-------------|
| tRPC | Web 应用前端 | `packages/trpc/server/routers/` |
| REST API v2 | 平台集成、第三方开发者 | `apps/api/v2/src/` |

两套系统有各自的守卫、服务和存储层实现，导致逻辑分散。

#### 2. 分层设计原则

遵循典型的分层架构：

```
Guard（守卫）→ Service（业务）→ Repository（数据访问）
     │                │                  │
     │         权限验证点 1          权限验证点 2
     ▼                ▼                  ▼
  请求入口         业务逻辑            数据库查询
```

每层都有自己的权限验证：
- **Guard**：验证用户身份和资源所有权
- **Service**：验证业务规则（如改期权限、取消权限）
- **Repository**：验证数据访问范围（查询条件过滤）

#### 3. 不同场景的不同权限模型

| 场景 | 权限关注点 | 主要实现位置 |
|------|-----------|-------------|
| 事件类型 CRUD | 所有权、团队角色 | EventTypeAccessService + Guards |
| 预订创建 | 事件类型可见性、配额限制 | RegularBookingService + getPublicEvent |
| 预订改期 | 事件类型设置、提前通知时间 | determineReschedulePreventionRedirect |
| 预订取消 | 事件类型设置、组织者/参与者身份 | handleCancelBooking |
| 公共访问 | hidden 过滤、私有链接 | getEventTypesPublic + HashedLinkService |

### 5.4 分散式架构的优缺点

#### 优点

1. **分层清晰**：各层职责明确，符合单一职责原则
2. **深度防御**：多层验证提供安全冗余
3. **场景适配**：不同场景可以有不同的权限策略
4. **演进灵活**：某一层的修改不影响其他层

#### 缺点

1. **逻辑重复**：相似的权限判断在多处实现
2. **维护困难**：修改权限规则需要检查多个位置
3. **不一致风险**：不同层的判断逻辑可能不一致
4. **难以审计**：权限判断路径复杂，难以追踪

---

## 6. 事件类型模型对权限变更的影响

### 6.1 数据模型关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    事件类型与预订的关系                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────┐                           ┌──────────────┐  │
│   │  EventType   │                           │   Booking    │  │
│   │  (事件类型)   │◄──────── (N) ───────────│  (预订记录)   │  │
│   └──────────────┘     eventTypeId           └──────────────┘  │
│          │                                                │      │
│          │                                                │      │
│          ├── userId/owner ─────► 冗余存储 ─────► userId │      │
│          │                                     userPrimaryEmail│
│          ├── teamId                                         │      │
│          ├── hosts                                          │      │
│          ├── hidden                                         │      │
│          ├── hashedLink                                     │      │
│          ├── disableRescheduling                            │      │
│          └── disableCancelling                              │      │
│                                                                 │
│   关键设计决策：                                               │
│   1. Booking 通过 eventTypeId 关联 EventType                  │
│   2. Booking 同时冗余存储组织者信息（userId, userPrimaryEmail）│
│   3. 没有外键级联删除（onDelete: Cascade）                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 权限变更场景分析

#### 场景 1：所有权转移（userId/teamId 改变）

**影响范围**：

| 影响对象 | 是否受影响 | 行为说明 |
|---------|-----------|---------|
| **已有 Booking** | ❌ 不受影响 | 预订记录独立存在，通过冗余字段保留历史 |
| **新预订** | ✅ 受影响 | 新 owner 可以管理该事件类型的未来预订 |
| **预订操作（改期/取消）** | ⚠️ 可能受影响 | 取决于查询逻辑是否检查当前 EventType 的 owner |

**代码证据**：`packages/prisma/schema.prisma:851-900`

```prisma
model Booking {
  id              Int           @id @default(autoincrement())
  uid             String        @unique
  userId          Int?          // 组织者 ID（冗余存储）
  userPrimaryEmail String?      // 组织者邮箱（冗余存储）
  eventTypeId     Int?          // 关联事件类型（可空）
  eventType       EventType?    @relation(fields: [eventTypeId], references: [id])
  // 注意：没有 onDelete: Cascade
  attendees       Attendee[]
  status          BookingStatus @default(ACCEPTED)
  // ...
}
```

**关键**：`eventTypeId` 是 `Int?`（可空类型），且没有级联删除约束。

#### 场景 2：可见性变更（hidden: false → true）

**影响范围**：

| 影响对象 | 是否受影响 | 行为说明 |
|---------|-----------|---------|
| **已有 Booking** | ❌ 不受影响 | 预订记录仍然存在，参与者仍可访问 |
| **公共页面新预订** | ✅ 受影响 | 事件类型不在公共列表显示，无法通过普通路径预约 |
| **私有链接新预订** | ❌ 不受影响 | 有效的 hashed link 仍然可以访问并预约 |

**代码证据**：`packages/features/eventtypes/lib/getEventTypesPublic.ts:14-16`

```typescript
const eventTypesWithHidden = await getEventTypesWithHiddenFromDB(userId);
// 只过滤公共列表，不影响直接通过 slug 查询
const eventTypesRaw = eventTypesWithHidden.filter((evt) => !evt.hidden);
```

#### 场景 3：预订控制变更（disableRescheduling 等）

**影响范围**：

| 配置变更 | 已有预订 | 新预订 |
|---------|---------|--------|
| `disableRescheduling: false → true` | ✅ **受影响** | 受影响 |
| `disableCancelling: false → true` | ⚠️ 可能受影响 | 受影响 |
| `minimumRescheduleNotice` 增加 | ✅ **受影响** | 受影响 |

**代码证据**：`apps/web/lib/reschedule/[uid]/getServerSideProps.ts:122-152`

```typescript
// 改期时会查询**当前** EventType 的设置
const reschedulePreventionRedirectUrl = determineReschedulePreventionRedirect({
  booking: {
    uid,
    status: booking.status,
    startTime: booking.startTime,
    endTime: booking.endTime,
    responses: booking.responses,
    userId: booking.userId,
    eventType: {
      // 注意：这里使用的是从数据库查询的当前 eventType 的设置
      disableRescheduling: !!eventType?.disableRescheduling,
      allowReschedulingPastBookings: eventType.allowReschedulingPastBookings,
      allowBookingFromCancelledBookingReschedule: !!eventType.allowReschedulingCancelledBookings,
      minimumRescheduleNotice: eventType.minimumRescheduleNotice,
      teamId: eventType.team?.id ?? null,
    },
  },
  eventUrl,
  // ...
});
```

**关键发现**：改期权限检查使用的是**当前 EventType 的设置**，而非预订创建时的快照。

#### 场景 4：事件类型删除

**影响范围**：

| 影响对象 | 是否受影响 | 行为说明 |
|---------|-----------|---------|
| **已有 Booking** | ⚠️ 部分受影响 | Booking 记录保留，但 `eventTypeId` 变为 null |
| **相关操作** | ✅ 受影响 | 依赖 EventType 关联的操作可能失败 |
| **新预订** | ✅ 受影响 | 无法创建新预订 |

**代码证据**：`packages/prisma/schema.prisma:180-183`

```prisma
model EventType {
  // ...
  team                               Team?                  @relation(fields: [teamId], references: [id], onDelete: Cascade)
  teamId                             Int?
  hashedLink                         HashedLink[]
  bookings                           Booking[]  // 注意：没有 onDelete: Cascade
  // ...
}
```

对比 `team` 关系有 `onDelete: Cascade`，但 `bookings` 没有。

### 6.3 权限变更的实际行为矩阵

| 变更类型 | 已有预订数据 | 已有预订操作 | 新预订 |
|---------|------------|-------------|--------|
| 所有权转移 (userId) | 保留 | 可能受影响¹ | 新 owner 管理 |
| 所有权转移 (teamId) | 保留 | 可能受影响¹ | 新 team 管理 |
| hidden: false → true | 保留 | 不受影响 | 公共页面不可见² |
| disableRescheduling: false → true | 保留 | **无法改期**³ | 无法改期 |
| disableCancelling: false → true | 保留 | 可能无法取消 | 无法取消 |
| 事件类型删除 | 保留⁴ | 可能失败 | 无法创建 |

**注释**：
1. 取决于操作时的权限查询逻辑（是否检查当前 EventType 的 owner）
2. 私有链接仍然可用
3. 代码明确查询当前 EventType 的设置
4. `eventTypeId` 变为 `null`，但 Booking 记录本身保留

### 6.4 设计优势与潜在问题

#### 设计优势

1. **历史数据完整性**：预订一旦创建，就是独立的业务记录，不会因事件类型变更而丢失
2. **审计合规**：所有历史预订数据保留原始信息
3. **业务连续性**：即使事件类型被删除/修改，参与者仍可查看历史预订
4. **回滚能力**：权限变更可以撤销，不会影响已完成的业务

#### 潜在问题

1. **数据不一致风险**
   - 场景：事件类型的 `disableRescheduling` 从 `false` 改为 `true`
   - 问题：用户创建预订时承诺可以改期，但后续无法改期
   - 建议：考虑在预订创建时保存关键配置的快照

2. **权限漂移问题**
   - 场景：事件类型从 User A 转移给 User B
   - 问题：User A 创建的预订，User B 是否应该有权管理？
   - 当前行为：取决于查询逻辑（可能检查 `eventType.hosts` 而非 `booking.userId`）

3. **级联策略不明确**
   - 场景：事件类型删除
   - 问题：相关预订如何处理？自动取消？保留但标记？
   - 当前行为：`eventTypeId` 置空，但预订状态不变

4. **缺乏通知机制**
   - 场景：关键权限变更（如 disableRescheduling）
   - 问题：已预订的用户不知道变更
   - 建议：权限变更时通知相关预订者

---

## 7. 可见性变更后原有预约的处理

### 7.1 当前实现：无自动处理机制

代码库中**没有发现**以下自动处理逻辑：

| 期望行为 | 是否存在 | 代码位置 |
|---------|---------|---------|
| 事件类型 hidden 变更时自动取消预订 | ❌ 不存在 | - |
| 事件类型所有权转移时通知预订者 | ❌ 不存在 | - |
| 事件类型删除时处理相关预订 | ❌ 不存在 | - |
| 关键配置变更时通知预订者 | ❌ 不存在 | - |

### 7.2 各操作的实际权限检查

#### 改期操作（Reschedule）

**权限检查位置**：`apps/web/lib/reschedule/[uid]/getServerSideProps.ts`

```typescript
// 步骤 1：查询预订和关联的事件类型
const booking = await prisma.booking.findUnique({
  where: { uid },
  select: {
    // ...
    eventType: {
      select: {
        // 查询**当前**事件类型的设置
        disableRescheduling: true,
        allowReschedulingPastBookings: true,
        allowReschedulingCancelledBookings: true,
        minimumRescheduleNotice: true,
        team: { select: { id: true, parentId: true, slug: true } },
        // ...
      },
    },
  },
});

// 步骤 2：根据当前设置判断是否允许改期
const reschedulePreventionRedirectUrl = determineReschedulePreventionRedirect({
  booking: {
    // ...
    eventType: {
      disableRescheduling: !!eventType?.disableRescheduling,
      // ... 使用当前设置
    },
  },
  // ...
});

if (reschedulePreventionRedirectUrl) {
  return {
    redirect: {
      destination: reschedulePreventionRedirectUrl,
      permanent: false,
    },
  };
}
```

**关键发现**：改期权限**完全依赖当前 EventType 的设置**，而非预订创建时的配置。

#### 取消操作（Cancel）

**权限检查位置**：`packages/features/bookings/lib/handleCancelBooking.ts`

取消权限通常基于：
1. 用户身份（组织者或参与者）
2. 预订状态
3. 事件类型的 `disableCancelling` 设置（可能检查当前值）

#### 查看预订详情

**权限检查位置**：`packages/features/bookings/repositories/BookingRepository.ts:981-1040`

```typescript
async findBookingByUidAndUserId({ bookingUid, userId }: { bookingUid: string; userId: number }) {
  return await this.prismaClient.booking.findFirst({
    where: {
      uid: bookingUid,
      OR: [
        // 条件 1: 用户是预订的组织者
        { userId: userId },
        
        // 条件 2: 用户是事件类型的主持人
        { eventType: { hosts: { some: { userId } } } },
        
        // 条件 3: 用户在事件类型的用户列表中
        { eventType: { users: { some: { id: userId } } } },
        
        // 条件 4: 用户是事件类型所属团队的管理员/所有者
        {
          eventType: {
            team: {
              members: {
                some: { userId, accepted: true, role: { in: ["ADMIN", "OWNER"] } },
              },
            },
          },
        },
        
        // 条件 5: 用户是父团队的管理员/所有者
        {
          eventType: {
            parent: {
              team: {
                members: {
                  some: { userId, accepted: true, role: { in: ["ADMIN", "OWNER"] } },
                },
              },
            },
          },
        },
      ],
    },
  });
}
```

**关键发现**：查看预订详情的权限**依赖于当前 EventType 的关系**，而非预订创建时的状态。

### 7.3 权限变更场景的具体行为

#### 场景 A：事件类型设置 hidden = true

| 行为 | 结果 |
|------|------|
| 已有预订的参与者查看预订 | ✅ 可以（通过 bookingUid 直接访问） |
| 已有预订的组织者管理预订 | ✅ 可以（通过管理后台） |
| 新用户通过公共页面查找 | ❌ 找不到该事件类型 |
| 新用户通过私有链接访问 | ✅ 可以（如果链接有效） |
| 已有预订的改期/取消 | ⚠️ 取决于其他设置，与 hidden 无关 |

#### 场景 B：事件类型所有权从 User A 转移到 User B

| 行为 | 结果 |
|------|------|
| User A 查看自己创建的预订 | ⚠️ 取决于查询逻辑 |
| User B 查看该事件类型的预订 | ✅ 可以（如果查询检查 eventType.owner） |
| User A 创建的预订详情页 | ⚠️ 可能无法访问（条件 2-5 检查当前 EventType） |
| 新预订 | ✅ User B 管理 |

**风险点**：如果 User A 不是新 EventType 的 host/team member，可能无法访问自己创建的预订详情。

#### 场景 C：事件类型设置 disableRescheduling = true

| 行为 | 结果 |
|------|------|
| 改期操作触发时 | ✅ 重定向到禁止页面 |
| 前端改期按钮 | ⚠️ 可能显示也可能隐藏（取决于 eventData） |
| 已有预订 | 📋 数据保留，但操作受限 |

### 7.4 建议的改进方案

#### 方案 1：预订配置快照

在预订创建时，保存事件类型的关键配置快照：

```typescript
// Booking 模型新增字段
model Booking {
  // ... 现有字段
  
  // 新增：创建时的配置快照
  snapshotConfig  Json?  // 存储 { disableRescheduling, disableCancelling, ... }
}
```

**优点**：
- 预订创建时的承诺得到保障
- 权限变更不影响已有预订的操作
- 用户体验一致

**缺点**：
- 数据冗余
- 需要迁移现有数据

#### 方案 2：统一权限判断服务

抽取 `BookingAccessService`，统一所有预订相关的权限判断：

```typescript
class BookingAccessService {
  // 统一判断用户是否有权访问预订
  async canAccessBooking(booking: Booking, userId: number): Promise<boolean> {
    // 优先级：
    // 1. 用户是预订的组织者（booking.userId）
    if (booking.userId === userId) return true;
    
    // 2. 用户是预订的参与者（attendee）
    if (await this.isAttendee(booking.id, userId)) return true;
    
    // 3. 用户是当前事件类型的管理员
    if (booking.eventTypeId) {
      const canAccessEventType = await this.eventTypeAccessService
        .userIsEventTypeAdminOrOwner(user, eventType);
      if (canAccessEventType) return true;
    }
    
    return false;
  }

  // 统一判断是否允许改期
  async canReschedule(booking: Booking): Promise<boolean> {
    // 策略选择：
    // - 使用当前 EventType 设置？
    // - 使用创建时的快照？
    // - 混合策略？
  }
}
```

**优点**：
- 逻辑集中，易于维护
- 行为一致，避免分散实现的差异
- 便于审计和测试

#### 方案 3：显式的权限变更通知机制

当事件类型的关键权限变更时：

```typescript
class EventTypePermissionChangeHandler {
  async handleVisibilityChange(eventType: EventType, oldHidden: boolean) {
    if (!oldHidden && eventType.hidden) {
      // 从可见变为隐藏
      await this.notifyUpcomingBookings(eventType.id, {
        type: 'VISIBILITY_CHANGED',
        message: '此事件类型已不再公开可见',
      });
    }
  }

  async handleRescheduleDisable(eventType: EventType) {
    // 获取未来的预订
    const upcomingBookings = await this.bookingRepository
      .findUpcomingByEventTypeId(eventType.id);
    
    for (const booking of upcomingBookings) {
      // 发送通知
      await this.notificationService.send({
        to: booking.attendees,
        type: 'RESCHEDULE_DISABLED',
        bookingUid: booking.uid,
      });
    }
  }
}
```

#### 方案 4：可配置的级联策略

在事件类型上配置权限变更时的行为：

```prisma
model EventType {
  // ... 现有字段
  
  // 新增：权限变更策略
  permissionChangePolicy  Json?  // {
                                  //   onOwnerChange: 'transfer' | 'keep_original' | 'notify',
                                  //   onDisableRescheduling: 'enforce' | 'grandfather',
                                  //   onDelete: 'cancel_bookings' | 'keep_bookings' | 'notify'
                                  // }
}
```

| 策略选项 | 行为 |
|---------|------|
| `transfer` | 权限完全转移给新 owner |
| `keep_original` | 原创建者保留对已有预订的权限 |
| `grandfather` | 已有预订不受新限制影响 |
| `enforce` | 所有预订立即受新限制影响 |

---

## 8. 关键代码位置索引

### 8.1 权限判断核心文件

| 功能 | 文件路径 | 关键函数/类 |
|------|---------|------------|
| API v2 事件类型权限 | `apps/api/v2/src/modules/event-types/services/event-type-access.service.ts` | `userIsEventTypeAdminOrOwner()` |
| API v2 所有权守卫 | `apps/api/v2/src/modules/event-types/guards/event-type-ownership.guard.ts` | `EventTypeOwnershipGuard` |
| tRPC 事件类型路由 | `packages/trpc/server/routers/viewer/eventTypes/_router.ts` | `eventTypesRouter` |
| 团队权限工具 | `packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts` | `getTeamPermissions()`, `getFallbackPermissions()` |
| 事件类型存储层 | `packages/features/eventtypes/repositories/eventTypeRepository.ts` | `EventTypeRepository.findById()` |
| 事件类型获取（管理端） | `packages/features/eventtypes/lib/getEventTypeById.ts` | `getEventTypeById()`, `getRawEventType()` |
| 事件类型获取（公共端） | `packages/features/eventtypes/lib/getPublicEvent.ts` | `getPublicEvent()` |
| 公共事件列表过滤 | `packages/features/eventtypes/lib/getEventTypesPublic.ts` | `getEventTypesPublic()` |
| 私有链接服务 | `packages/features/hashedLink/lib/service/HashedLinkService.ts` | `HashedLinkService.validate()` |
| 私有链接验证工具 | `packages/lib/hashedLinksUtils.ts` | `validateHashedLinkData()` |
| 预订存储层 | `packages/features/bookings/repositories/BookingRepository.ts` | `findBookingByUidAndUserId()` |
| 预订服务 | `packages/features/bookings/lib/service/RegularBookingService.ts` | `RegularBookingService` |
| 改期权限判断 | `apps/web/lib/reschedule/[uid]/getServerSideProps.ts` | `determineReschedulePreventionRedirect()` |
| 私有链接页面 SSR | `apps/web/lib/d/[link]/[slug]/getServerSideProps.tsx` | `getServerSideProps()` |

### 8.2 数据模型定义

| 模型 | 文件路径 | 关键行号 |
|------|---------|---------|
| EventType | `packages/prisma/schema.prisma` | 156-306 |
| Booking | `packages/prisma/schema.prisma` | 851-950+ |
| HashedLink | `packages/prisma/schema.prisma` | 1205-1215 |
| Host | `packages/prisma/schema.prisma` | 61-85 |
| MembershipRole | `packages/prisma/schema.prisma` | (enum 定义) |

### 8.3 关键数据库字段

#### EventType 权限相关字段

| 字段 | 类型 | 默认值 | 用途 |
|------|------|--------|------|
| `userId` | Int? | null | 个人事件类型的所有者 |
| `teamId` | Int? | null | 团队事件类型的所属团队 |
| `hidden` | Boolean | false | 公共页面可见性 |
| `disableCancelling` | Boolean? | false | 禁止取消 |
| `disableRescheduling` | Boolean? | false | 禁止改期 |
| `minimumRescheduleNotice` | Int? | null | 改期提前通知时间（分钟） |

#### Booking 权限相关字段

| 字段 | 类型 | 默认值 | 用途 |
|------|------|--------|------|
| `userId` | Int? | null | 组织者 ID（冗余存储） |
| `userPrimaryEmail` | String? | null | 组织者邮箱（冗余存储） |
| `eventTypeId` | Int? | null | 关联事件类型 |
| `status` | BookingStatus | ACCEPTED | 预订状态 |

#### HashedLink 字段

| 字段 | 类型 | 默认值 | 用途 |
|------|------|--------|------|
| `link` | String | (唯一) | 私有链接标识符 |
| `eventTypeId` | Int | - | 关联事件类型 |
| `expiresAt` | DateTime? | null | 过期时间 |
| `maxUsageCount` | Int | 1 | 最大使用次数 |
| `usageCount` | Int | 0 | 已使用次数 |

---

## 总结

### 核心发现

1. **多维度权限判断**：事件类型归属者权限通过 5 个维度判断（系统管理员、直接所有者、主持人/被分配用户、团队管理员、组织管理员）

2. **双层可见性控制**：
   - `hidden` 字段控制公共页面可见性
   - `HashedLink` 私有链接机制绕过 `hidden` 限制

3. **服务端唯一可信源**：所有权限判断在服务端完成，前端只展示服务端返回的数据。无权限时服务端直接返回 404。

4. **分散式权限管理**：权限逻辑分散在 Guard、Service、Repository 多层，以及 tRPC 和 API v2 两套系统中。

5. **预订数据独立性**：Booking 通过冗余字段保存组织者信息，事件类型的所有权变更不影响已有预订数据。

6. **操作权限依赖当前配置**：改期、取消等操作的权限检查使用**当前 EventType 的设置**，而非预订创建时的快照。

### 潜在风险

1. **权限变更影响已有操作**：`disableRescheduling` 等配置变更会影响已有预订的改期能力
2. **所有权转移后的访问问题**：原创建者可能无法访问自己创建的预订（如果查询逻辑检查当前 EventType 关系）
3. **缺乏自动通知**：权限变更时没有通知相关预订者的机制
4. **逻辑分散维护困难**：相似的权限判断在多处实现，存在不一致风险

### 改进建议

1. **考虑引入预订配置快照**：确保用户创建预订时的承诺得到保障
2. **抽取统一的 BookingAccessService**：集中权限判断逻辑，确保一致性
3. **添加权限变更通知机制**：关键配置变更时通知相关预订者
4. **明确级联策略**：事件类型删除/变更时应有明确的预订处理策略

---

*报告生成时间：2026-05-03*
*分析范围：基于代码库静态分析*
