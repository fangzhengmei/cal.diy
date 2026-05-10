# Cal.diy 组织与团队权限系统分析

## 1. 数据模型结构

### 1.1 组织与团队模型

cal.diy 采用统一的 `Team` 模型来表示组织和团队，通过以下字段区分：

- **`isOrganization`**: `Boolean` - 标识是否为组织
- **`parentId`**: `Int?` - 父团队/组织 ID
  - `parentId === null` 且 `isOrganization === true`：表示组织
  - `parentId !== null`：表示团队（子团队）
- **`parent` / `children`**: 自引用关系，支持层级结构

**关键模型关系：**
```
Organization (Team.isOrganization = true)
  └── Team (Team.parentId = Organization.id)
       └── Membership (用户与团队的关联)
            └── Role (角色定义，支持自定义角色)
```

**参考文件：**
- [packages/prisma/schema.prisma:557-650](packages/prisma/schema.prisma#L557-L650) - Team 模型定义

### 1.2 成员模型 (Membership)

`Membership` 模型定义了用户与团队/组织的关联关系：

**关键字段：**
- `teamId`: 关联的团队/组织 ID
- `userId`: 关联的用户 ID
- `accepted`: `Boolean` - 是否接受邀请（`@default(false)`）
- `role`: `MembershipRole` - 成员角色（MEMBER, ADMIN, OWNER）
- `customRoleId`: `String?` - 自定义角色 ID
- `customRole`: `Role?` - 自定义角色关联

**唯一约束：** `@@unique([userId, teamId])` - 一个用户在一个团队中只能有一个成员身份

**角色枚举：**
```typescript
enum MembershipRole {
  MEMBER  // 普通成员
  ADMIN   // 管理员
  OWNER   // 拥有者
}
```

**参考文件：**
- [packages/prisma/schema.prisma:744-765](packages/prisma/schema.prisma#L744-L765) - Membership 模型定义
- [packages/prisma/schema.prisma:738-742](packages/prisma/schema.prisma#L738-L742) - MembershipRole 枚举

### 1.3 角色系统 (Role & RolePermission)

支持自定义角色系统，通过 PBAC（基于策略的访问控制）进行权限管理：

**Role 模型：**
- `id`: 角色 ID
- `name`: 角色名称
- `teamId`: 所属团队 ID（null 表示全局角色）
- `type`: `RoleType` - 角色类型（SYSTEM, CUSTOM）
- `permissions`: `RolePermission[]` - 角色权限列表
- `memberships`: `Membership[]` - 关联的成员

**RolePermission 模型：**
- `roleId`: 角色 ID
- `resource`: 资源类型（如 "eventType", "booking"）
- `action`: 操作类型（如 "read", "create", "update", "delete"）

**角色类型枚举：**
```typescript
enum RoleType {
  SYSTEM  // 系统内置角色
  CUSTOM  // 自定义角色
}
```

**参考文件：**
- [packages/prisma/schema.prisma:2354-2388](packages/prisma/schema.prisma#L2354-L2388) - Role 和 RolePermission 模型定义

### 1.4 邀请与成员状态

邀请机制通过 `VerificationToken` 和 `Membership.accepted` 字段实现：

- **邀请流程：**
  1. 发送邀请时创建 `VerificationToken`，关联 `teamId`
  2. 创建 `Membership` 记录，`accepted = false`
  3. 用户接受邀请后，更新 `Membership.accepted = true`

**关键状态：**
- `Membership.accepted = false`: 待接受的邀请
- `Membership.accepted = true`: 正式成员

**参考文件：**
- [packages/prisma/schema.prisma:767-784](packages/prisma/schema.prisma#L767-L784) - VerificationToken 模型定义

### 1.5 席位系统

席位系统包含两个层面：

#### 1.5.1 预约席位 (BookingSeat)

用于处理"席位式预约"（如多人会议、研讨会等）：

**EventType 相关字段：**
- `seatsPerTimeSlot`: `Int?` - 每个时间段的最大席位数量
- `seatsShowAttendees`: `Boolean?` - 是否显示参与者
- `seatsShowAvailabilityCount`: `Boolean?` - 是否显示剩余席位数量

**BookingSeat 模型：** 记录每个席位的预订详情

#### 1.5.2 组织席位 (Organization Seats)

用于组织计费和成员数量限制：

**OrganizationOnboarding 模型字段：**
- `pricePerSeat`: `Float` - 每个席位的价格
- `seats`: `Int` - 购买的席位数量

**Team 模型字段：**
- `seatChangeLogs`: `SeatChangeLog[]` - 席位变更记录
- `monthlyProrations`: `MonthlyProration[]` - 月度按比例计费记录

**参考文件：**
- [packages/prisma/schema.prisma:222-230](packages/prisma/schema.prisma#L222-L230) - EventType 席位相关字段
- [packages/prisma/schema.prisma:2232-2240](packages/prisma/schema.prisma#L2232-L2240) - OrganizationOnboarding 席位相关字段
- [packages/prisma/schema.prisma:1330](packages/prisma/schema.prisma#L1330) - BookingSeat 模型

---

## 2. 角色系统与权限控制机制

### 2.1 权限检查流程

cal.diy 采用 PBAC（Policy-Based Access Control）权限模型，同时提供基于角色的回退机制：

**权限检查服务接口：**
```typescript
class PermissionCheckService {
  // 检查特定权限
  async checkPermission({
    userId,
    teamId,
    permission,    // 如 "eventType.read", "booking.readTeamBookings"
    fallbackRoles  // 回退角色列表
  }): Promise<boolean>;
  
  // 获取具有特定权限的团队 ID 列表
  async getTeamIdsWithPermission(...args): Promise<number[]>;
}
```

**权限检查流程：**
1. 尝试执行 PBAC 权限检查
2. 如果 PBAC 检查失败或不可用，回退到基于角色的检查
3. 检查用户在该团队中的角色是否在 `fallbackRoles` 列表中

**参考文件：**
- [packages/trpc/server/procedures/pbacProcedures.ts](packages/trpc/server/procedures/pbacProcedures.ts) - PBAC 过程定义
- [packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts](packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts) - 权限工具函数

### 2.2 团队级权限控制

**创建团队权限检查过程：**
```typescript
function createTeamPbacProcedure(
  permission: PermissionString,
  fallbackRoles: MembershipRole[] = [MembershipRole.ADMIN, MembershipRole.OWNER]
)
```

**工作流程：**
1. 从 `input.teamId` 获取团队 ID
2. 调用 `permissionCheckService.checkPermission()`
3. 如果权限检查失败，抛出 `FORBIDDEN` 错误

**示例权限：**
- `"team.read"`, `"team.update"` - 团队操作权限
- `"eventType.read"`, `"eventType.create"` - 事件类型权限
- `"booking.readTeamBookings"` - 团队预约读取权限

**参考文件：**
- [packages/trpc/server/procedures/pbacProcedures.ts:22-50](packages/trpc/server/procedures/pbacProcedures.ts#L22-L50) - 团队 PBAC 过程

### 2.3 组织级权限控制

**创建组织权限检查过程：**
```typescript
function createOrgPbacProcedure(
  permission: PermissionString,
  fallbackRoles: MembershipRole[] = [MembershipRole.ADMIN, MembershipRole.OWNER]
)
```

**工作流程：**
1. 从 `ctx.user.organizationId` 获取组织 ID
2. 检查用户是否属于某个组织
3. 调用 `permissionCheckService.checkPermission()`
4. 将 `organizationId` 传递到上下文

**示例权限：**
- `"organization.read"`, `"organization.update"` - 组织操作权限
- `"booking.readOrgBookings"` - 组织预约读取权限

**参考文件：**
- [packages/trpc/server/procedures/pbacProcedures.ts:60-96](packages/trpc/server/procedures/pbacProcedures.ts#L60-L96) - 组织 PBAC 过程

### 2.4 权限工具函数

**角色层级定义：**
```typescript
const MEMBERSHIP_HIERARCHY: Record<MembershipRole, number> = {
  [MembershipRole.MEMBER]: 1,   // 最低权限
  [MembershipRole.ADMIN]: 2,    // 中等权限
  [MembershipRole.OWNER]: 3,    // 最高权限
};
```

**权限判断函数：**
- `hasHigherPrivilege(role1, role2)`: 比较两个角色的权限级别
- `getEffectiveRole(orgMembership, teamMembership)`: 计算有效角色

**团队权限获取：**
```typescript
async function getTeamPermissions(
  userId: number,
  teamId: number,
  effectiveRole: MembershipRole
): Promise<TeamPermissions>
```

**默认权限映射：**
| 操作 | ADMIN/OWNER | MEMBER |
|------|------------|--------|
| 读取 (Read) | ✅ | ✅ |
| 创建 (Create) | ✅ | ❌ |
| 编辑 (Edit) | ✅ | ❌ |
| 删除 (Delete) | ✅ | ❌ |

**参考文件：**
- [packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts:20-90](packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts#L20-L90) - 权限工具函数

---

## 3. 角色叠加顺序与有效角色计算

### 3.1 角色叠加原理

**核心规则：组织角色优先于团队角色**

在 cal.diy 中，用户可以同时拥有：
1. **组织角色**：在组织级别的 `Membership` 中的角色
2. **团队角色**：在具体团队中的 `Membership` 中的角色

**有效角色计算函数：**
```typescript
export function getEffectiveRole(
  orgMembership: MembershipRole | undefined,
  teamMembershipRole: MembershipRole
): MembershipRole {
  return orgMembership && hasHigherPrivilege(orgMembership, teamMembershipRole) 
    ? orgMembership 
    : teamMembershipRole;
}
```

**计算逻辑：**
1. 如果用户在组织中有角色（`orgMembership` 存在）
2. 比较组织角色和团队角色的权限级别
3. 如果组织角色权限更高，使用组织角色
4. 否则使用团队角色

**参考文件：**
- [packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts:30-35](packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts#L30-L35) - getEffectiveRole 函数

### 3.2 角色叠加示例

**示例 1：组织管理员 + 团队成员**
- 组织角色：ADMIN
- 团队角色：MEMBER
- 有效角色：ADMIN（组织角色权限更高）

**示例 2：组织成员 + 团队管理员**
- 组织角色：MEMBER
- 团队角色：ADMIN
- 有效角色：ADMIN（团队角色权限更高）

**示例 3：组织所有者 + 团队管理员**
- 组织角色：OWNER
- 团队角色：ADMIN
- 有效角色：OWNER（组织角色权限更高）

### 3.3 团队权限映射构建

**构建流程：**
```typescript
export async function buildTeamPermissionsMap(
  memberships: Array<{ team: { id: number; parentId?: number | null }; role: MembershipRole }>,
  teamMemberships: MembershipWithRole[],
  userId: number
): Promise<Map<number, TeamPermissions>>
```

**执行步骤：**
1. 遍历用户的所有团队成员身份
2. 查找每个团队的父组织（通过 `team.parentId`）
3. 获取用户在父组织中的角色
4. 计算有效角色
5. 获取该团队的权限集
6. 构建团队 ID 到权限的映射

**参考文件：**
- [packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts:92-110](packages/trpc/server/routers/viewer/eventTypes/utils/permissionUtils.ts#L92-L110) - buildTeamPermissionsMap 函数

### 3.4 团队事件类型访问控制

**访问控制逻辑：**
```typescript
async filterTeamsByEventTypeReadPermission(
  memberships: TeamMembershipWithTeam[],
  userId: number
): Promise<TeamMembershipWithTeam[]>
```

**过滤规则：**
1. **排除组织成员身份**：`if (membership.team.isOrganization) return null`
2. **检查事件类型读取权限**：
   - 权限名：`"eventType.read"`
   - 回退角色：`["ADMIN", "OWNER", "MEMBER"]`（所有角色都可以读取）

**注意：** 组织成员身份被排除在事件类型访问控制之外，因为事件类型主要在团队级别管理。

**参考文件：**
- [packages/trpc/server/routers/viewer/eventTypes/teamAccessUseCase.ts:26-52](packages/trpc/server/routers/viewer/eventTypes/teamAccessUseCase.ts#L26-L52) - 团队访问控制

---

## 4. 成员状态传播路径

### 4.1 成员服务与状态检查

**成员状态检查服务：**
```typescript
export class MembershipService {
  async checkMembership(teamId: number, userId: number): Promise<MembershipCheckResult>
}
```

**返回结果结构：**
```typescript
type MembershipCheckResult = {
  isMember: boolean;   // 是否为有效成员（已接受邀请）
  isAdmin: boolean;    // 是否为管理员或拥有者
  isOwner: boolean;    // 是否为拥有者
  role?: MembershipRole; // 具体角色
};
```

**状态检查逻辑：**
1. 查找用户在该团队的 `Membership` 记录
2. 检查 `membership.accepted` 是否为 `true`
3. 如果未接受邀请，返回 `isMember: false`
4. 根据 `membership.role` 确定管理员和拥有者状态

**参考文件：**
- [packages/features/membership/services/membershipService.ts](packages/features/membership/services/membershipService.ts) - 成员服务

### 4.2 成员状态对权限的影响

**关键规则：未接受邀请的成员没有权限**

```typescript
if (!membership || !membership.accepted) {
  return {
    isMember: false,
    isAdmin: false,
    isOwner: false,
    role: undefined,
  };
}
```

**状态传播路径：**

1. **邀请发送阶段**
   - 创建 `Membership` 记录：`accepted = false`
   - 创建 `VerificationToken`：包含邀请链接

2. **邀请接受阶段**
   - 用户点击邀请链接
   - 更新 `Membership.accepted = true`
   - 用户成为有效成员

3. **权限生效阶段**
   - `MembershipService.checkMembership()` 开始返回 `isMember: true`
   - 角色权限（`role`）开始生效
   - 可以访问团队资源

4. **角色变更阶段**
   - 更新 `Membership.role`
   - 权限立即生效（基于新角色）

5. **成员移除阶段**
   - 删除 `Membership` 记录
   - 权限立即失效

---

## 5. 多租户层对个人预约模型的侵入

### 5.1 预约访问控制服务

**预约访问权限检查：**
```typescript
export class BookingAccessService {
  async doesUserIdHaveAccessToBooking({
    userId,
    bookingUid,
    bookingId,
  }): Promise<boolean>
}
```

**参考文件：**
- [packages/features/bookings/services/BookingAccessService.ts](packages/features/bookings/services/BookingAccessService.ts) - 预约访问服务

### 5.2 预约访问权限层级

**访问权限检查顺序（优先级从高到低）：**

**Case 1: 用户是预约组织者**
```typescript
if (userId === booking.userId) return true;
```

**Case 2: 用户是预约主持人之一**
```typescript
if (this.isUserAHost(userId, booking)) return true;
```

**主持人判断逻辑：**
- 检查 `booking.eventType.hosts` 中的用户
- 检查 `booking.eventType.users` 中的用户
- 检查 `booking.user`（组织者）

**Case 3: 用户是事件类型所属团队的管理员**
```typescript
if (booking.eventType?.teamId) {
  const hasAccess = await this.permissionCheckService.checkPermission({
    userId,
    teamId: booking.eventType.teamId,
    permission: "booking.readTeamBookings",
    fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
  });
  return hasAccess;
}
```

**Case 4: 用户是预约组织者所属组织的管理员**
```typescript
if (bookingOwner.organizationId) {
  const hasAccess = await this.permissionCheckService.checkPermission({
    userId,
    teamId: bookingOwner.organizationId,
    permission: "booking.readOrgBookings",
    fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
  });
  if (hasAccess) return true;
}
```

**Case 5: 用户是预约组织者所属任何团队的管理员**
```typescript
for (const membership of bookingOwner.teams) {
  const hasAccess = await this.permissionCheckService.checkPermission({
    userId,
    teamId: membership.teamId,
    permission: "booking.readTeamBookings",
    fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
  });
  if (hasAccess) return true;
}
```

**参考文件：**
- [packages/features/bookings/services/BookingAccessService.ts:56-138](packages/features/bookings/services/BookingAccessService.ts#L56-L138) - 预约访问控制

### 5.3 多租户层侵入分析

**侵入点 1：组织管理员访问个人预约**

个人预约（`booking.eventType.teamId === null`）的组织者如果属于某个组织：
- 组织管理员（OWNER/ADMIN）可以通过 `booking.readOrgBookings` 权限访问
- 这是多租户层对个人预约的直接侵入

**侵入点 2：团队管理员访问团队成员的个人预约**

个人预约的组织者如果属于某个团队：
- 该团队的管理员（OWNER/ADMIN）可以通过 `booking.readTeamBookings` 权限访问
- 即使预约本身不是团队预约，团队管理员仍然可以访问

**侵入点 3：管理事件类型的权限继承**

对于管理事件类型（有 `parent` 的事件类型）：
```typescript
if (booking.eventType?.parent?.teamId) {
  const isAdminOrUser = await userRepo.isAdminOfTeamOrParentOrg({
    userId,
    teamId: booking.eventType.parent.teamId,
  });
  return isAdminOrUser;
}
```

**权限侵入的设计意图：**
1. **组织管理需求**：组织需要统一管理成员的预约
2. **团队协作需求**：团队管理员需要了解团队成员的工作安排
3. **合规性需求**：某些行业要求对员工日程进行监督

**潜在问题：**
1. **隐私泄露风险**：个人预约可能包含敏感信息
2. **权限边界模糊**：个人预约和团队预约的访问权限重叠
3. **层级复杂性**：组织 → 团队 → 个人的权限传播路径复杂

---

## 6. 席位限制与预约访问控制

### 6.1 席位式预约的访问控制

**席位预留处理：**
```typescript
export const reserveSlotHandler = async ({ ctx, input }: ReserveSlotOptions)
```

**关键逻辑：**

1. **检查是否为席位式事件**
```typescript
if (eventType.seatsPerTimeSlot) {
  // 是席位式预约，特殊处理
}
```

2. **检查剩余席位**
```typescript
const bookingWithAttendees = await prisma.booking.findFirst({
  where: {
    eventTypeId,
    startTime: slotUtcStartDate,
    endTime: slotUtcEndDate,
    status: BookingStatus.ACCEPTED,
  },
  select: { attendees: true },
});

const seatsLeft = eventType.seatsPerTimeSlot - bookingAttendeesLength;
if (seatsLeft < 1) shouldReserveSlot = false;
```

3. **席位式预约不预留整个时间段**
```typescript
// 对于席位式事件，不预留整个时间段
// 只在最后一个席位被预定时才阻止其他预订
if (seatsLeft < 1) shouldReserveSlot = false;
else if (bookingAttendeesLength) shouldReserveSlot = true;
else shouldReserveSlot = false; // 还没有预订，不预留
```

**参考文件：**
- [packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts](packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts) - 席位预留处理

### 6.2 席位限制对预约的影响

**席位式预约 vs 普通预约：**

| 特性 | 普通预约 | 席位式预约 |
|------|--------|----------|
| 同一时间段预约人数 | 1 人 | `seatsPerTimeSlot` 人 |
| 时间段占用逻辑 | 一个预订占用整个时间段 | 每个预订占用一个席位 |
| 预留机制 | 立即预留整个时间段 | 只在席位不足时预留 |
| 显示信息 | 不显示参与者 | 可配置显示参与者和剩余席位 |

**席位限制的访问控制：**

1. **预订阶段**
   - 检查 `seatsPerTimeSlot` 是否设置
   - 计算已预订席位数量
   - 检查是否有剩余席位

2. **显示阶段**
   - `seatsShowAvailabilityCount`: 是否显示剩余席位数量
   - `seatsShowAttendees`: 是否显示参与者列表

3. **取消阶段**
   - 取消席位后，席位释放
   - 其他用户可以预订释放的席位

### 6.3 组织席位（计费席位）

**组织席位与预约席位的区别：**

| 维度 | 组织席位 | 预约席位 |
|------|--------|----------|
| 用途 | 组织成员数量限制 | 单次预约参与人数限制 |
| 计费 | 按席位数量计费 | 通常不单独计费 |
| 模型 | `OrganizationOnboarding.seats` | `EventType.seatsPerTimeSlot` |
| 关联 | 组织 → 成员 | 事件类型 → 预约 |

**组织席位限制的传播：**

1. **成员添加限制**
   - 组织成员数量不能超过购买的席位数量
   - 超过限制时无法添加新成员

2. **席位变更追踪**
   - `SeatChangeLog` 记录席位变更历史
   - `MonthlyProration` 处理月度按比例计费

---

## 7. 总结：权限系统架构

### 7.1 权限控制层次结构

```
┌─────────────────────────────────────────────────────────────┐
│                    权限控制层次                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              组织级权限 (Organization)               │    │
│  │  • booking.readOrgBookings (组织预约访问)            │    │
│  │  • organization.read/update (组织管理)              │    │
│  │  • 角色: OWNER, ADMIN, MEMBER                       │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │ 角色叠加 (高权限优先)              │
│  ┌──────────────────────▼──────────────────────────────┐    │
│  │                团队级权限 (Team)                     │    │
│  │  • booking.readTeamBookings (团队预约访问)           │    │
│  │  • eventType.read/create/update/delete              │    │
│  │  • team.read/update (团队管理)                       │    │
│  │  • 角色: OWNER, ADMIN, MEMBER (可自定义)             │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │ 成员状态检查 (accepted = true)     │
│  ┌──────────────────────▼──────────────────────────────┐    │
│  │              个人级权限 (User)                       │    │
│  │  • 自己的预约 (booking.userId === userId)            │    │
│  │  • 自己的事件类型                                    │    │
│  │  • 主持人身份 (eventType.hosts 包含 userId)          │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              席位限制 (Seats)                        │    │
│  │  • 预约席位: seatsPerTimeSlot (参与人数限制)          │    │
│  │  • 组织席位: OrganizationOnboarding.seats            │    │
│  │    (成员数量限制 + 计费)                              │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 预约访问权限决策树

```
用户请求访问预约
        │
        ▼
┌─────────────────────┐
│ 用户是预约组织者？   │
│ (userId === booking.userId) │
└─────────────────────┘
        │ 是
        ▼
      ✅ 允许访问
        │ 否
        ▼
┌─────────────────────┐
│ 用户是主持人之一？   │
│ (isUserAHost)       │
└─────────────────────┘
        │ 是
        ▼
      ✅ 允许访问
        │ 否
        ▼
┌─────────────────────────────────┐
│ 预约属于某个团队？               │
│ (booking.eventType?.teamId)     │
└─────────────────────────────────┘
        │ 是
        ▼
┌─────────────────────────────────┐
│ 用户有团队预约访问权限？         │
│ (booking.readTeamBookings)      │
│ 回退角色: OWNER, ADMIN          │
└─────────────────────────────────┘
        │ 是
        ▼
      ✅ 允许访问
        │ 否
        ▼
┌─────────────────────────────────┐
│ 是管理事件类型？                 │
│ (booking.eventType?.parent?)    │
└─────────────────────────────────┘
        │ 是
        ▼
┌─────────────────────────────────┐
│ 用户是父团队/组织的管理员？      │
└─────────────────────────────────┘
        │ 是
        ▼
      ✅ 允许访问
        │ 否
        ▼
┌─────────────────────────────────┐
│ 预约组织者属于某个组织？         │
│ (bookingOwner.organizationId)   │
└─────────────────────────────────┘
        │ 是
        ▼
┌─────────────────────────────────┐
│ 用户有组织预约访问权限？         │
│ (booking.readOrgBookings)       │
│ 回退角色: OWNER, ADMIN          │
└─────────────────────────────────┘
        │ 是
        ▼
      ✅ 允许访问
        │ 否
        ▼
┌─────────────────────────────────┐
│ 预约组织者属于某个团队？         │
│ (bookingOwner.teams)            │
└─────────────────────────────────┘
        │ 是
        ▼
┌─────────────────────────────────┐
│ 用户是任一团队的管理员？         │
│ (booking.readTeamBookings)      │
│ 回退角色: OWNER, ADMIN          │
└─────────────────────────────────┘
        │ 是
        ▼
      ✅ 允许访问
        │ 否
        ▼
      ❌ 拒绝访问
```

### 7.3 关键设计原则

1. **角色叠加原则**：组织角色优先于团队角色，高权限角色覆盖低权限角色

2. **成员状态原则**：只有接受邀请的成员（`accepted = true`）才具有权限

3. **多租户侵入原则**：组织和团队管理员可以访问成员的个人预约

4. **权限回退原则**：PBAC 检查失败时回退到基于角色的检查

5. **席位分离原则**：预约席位（参与人数）和组织席位（成员数量）是两个独立概念

### 7.4 潜在风险与改进建议

**潜在风险：**

1. **隐私泄露**：组织/团队管理员可以访问成员的个人预约，可能包含敏感信息

2. **权限边界模糊**：个人预约和团队预约的访问权限重叠，可能导致误访问

3. **角色复杂性**：组织角色 + 团队角色 + 自定义角色的组合可能导致权限计算复杂

**改进建议：**

1. **隐私保护**：
   - 为个人预约添加隐私级别设置
   - 限制管理员访问的预约详情字段

2. **权限细分**：
   - 将 `booking.readOrgBookings` 细分为 `booking.readOrgTeamBookings` 和 `booking.readOrgPersonalBookings`
   - 允许组织配置是否允许管理员访问成员的个人预约

3. **审计日志**：
   - 记录所有管理员访问成员预约的行为
   - 提供访问日志查询界面

4. **席位统一**：
   - 统一预约席位和组织席位的术语，减少混淆
   - 提供更清晰的席位管理界面
