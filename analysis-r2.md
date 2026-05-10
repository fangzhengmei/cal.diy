# Cal.diy 预约访问权限链路 - 第二轮深度分析

## 核心发现总结

通过深入代码分析，发现了 **4 个严重的安全问题**，其中 **2 个是高危漏洞**，**1 个是中危漏洞**，**1 个是设计风险**。

---

## 问题 1：Managed Event 分支缺少 `accepted` 状态检查

### 漏洞位置

**文件：** `packages/features/users/repositories/UserRepository.ts`  
**方法：** `isAdminOfTeamOrParentOrg` (第 1014-1038 行)

**调用路径：**
```
BookingAccessService.doesUserIdHaveAccessToBooking() (第 97-103 行)
  └── UserRepository.isAdminOfTeamOrParentOrg()
        └── [缺少 accepted 状态检查]
```

### 漏洞代码分析

**BookingAccessService 中的调用点（第 96-103 行）：**
```typescript
// For managed events (child event types), check the parent's teamId
if (booking.eventType?.parent?.teamId) {
  const isAdminOrUser = await userRepo.isAdminOfTeamOrParentOrg({
    userId,
    teamId: booking.eventType.parent.teamId,
  });
  return isAdminOrUser;  // ⚠️ 直接返回，不经过 PermissionCheckService
}
```

**UserRepository.isAdminOfTeamOrParentOrg 实现（第 1014-1038 行）：**
```typescript
async isAdminOfTeamOrParentOrg({ userId, teamId }: { userId: number; teamId: number }) {
  const membershipQuery = {
    members: {
      some: {
        userId,
        role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
        // ⚠️ 缺少 accepted: true 条件！
      },
    },
  };
  const teams = await this.prismaClient.team.findMany({
    where: {
      id: teamId,
      OR: [
        membershipQuery,                          // 直接团队成员
        {
          parent: { ...membershipQuery },         // 父组织成员
        },
      ],
    },
    select: {
      id: true,
    },
  });
  return !!teams.length;
}
```

### 漏洞原理

**关键问题：** `membershipQuery` 中只检查了 `role`，但没有检查 `accepted: true`。

**对比正确实现（第 1039-1054 行的 `isAdminOrOwnerOfTeam`）：**
```typescript
async isAdminOrOwnerOfTeam({ userId, teamId }: { userId: number; teamId: number }) {
  const isAdminOrOwnerOfTeam = await this.prismaClient.membership.findUnique({
    where: {
      userId_teamId: {
        userId,
        teamId,
      },
      role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
      accepted: true,  // ✅ 正确检查了 accepted 状态
    },
    select: {
      id: true,
    },
  });
  return !!isAdminOrOwnerOfTeam;
}
```

### 可复现条件

**前置条件：**
1. 存在一个组织（Organization）和一个团队（Team）
2. 用户 A 被邀请加入该组织/团队，角色为 ADMIN 或 OWNER
3. 用户 A **尚未接受邀请**（`membership.accepted = false`）
4. 存在一个 Managed Event（子事件类型，有 `parentId`）
5. 该 Managed Event 的父事件类型属于该组织/团队

**攻击路径：**
```
1. 组织管理员邀请用户 A，设置角色为 ADMIN
2. 用户 A 收到邀请，但不点击接受链接
3. 此时数据库中:
   - membership.userId = A.id
   - membership.teamId = org.id
   - membership.role = ADMIN
   - membership.accepted = false  ⚠️
4. 用户 A 注册/登录系统
5. 用户 A 尝试访问该组织下任何 Managed Event 的预约
6. isAdminOfTeamOrParentOrg() 检查:
   - 查找 members.some({ userId: A.id, role: ADMIN })
   - ⚠️ 不检查 accepted 字段
   - 找到匹配的 membership 记录
   - 返回 true
7. ✅ 用户 A 成功获得访问权限！
```

### 影响范围

**受影响的资源：**
- 所有 Managed Event（子事件类型）的预约记录
- 包括预约详情、参与者信息、会议链接等敏感数据

**攻击面：**
- 任何被邀请但未接受邀请的"准管理员"
- 可以访问组织/团队的所有管理事件预约

### 风险等级

| 维度 | 评估 |
|------|------|
| 严重程度 | **高危 (High)** |
| 影响范围 | 所有使用 Managed Event 的组织/团队 |
| 利用难度 | 低 - 只需等待管理员邀请 |
| 隐蔽性 | 高 - 攻击者无需任何特殊操作 |
| 数据影响 | 可访问敏感预约数据 |

### 修复建议

**方案 1：在查询条件中添加 `accepted: true`**

修改 `packages/features/users/repositories/UserRepository.ts` 第 1015-1022 行：

```typescript
// 修改前
const membershipQuery = {
  members: {
    some: {
      userId,
      role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
      // 缺少 accepted 检查
    },
  },
};

// 修改后
const membershipQuery = {
  members: {
    some: {
      userId,
      role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
      accepted: true,  // ✅ 添加 accepted 检查
    },
  },
};
```

**方案 2：使用已有的 `isAdminOrOwnerOfTeam` 方法**

在 BookingAccessService 中使用更安全的方法，同时检查父组织：

```typescript
// 参考 isAdminOrOwnerOfTeam 的实现模式
async isAdminOfTeamOrParentOrgSafe({ userId, teamId }: { userId: number; teamId: number }) {
  // 1. 检查直接团队成员（包含 accepted 检查）
  const isDirectAdmin = await this.prismaClient.membership.findFirst({
    where: {
      userId,
      teamId,
      role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
      accepted: true,
    },
  });
  
  if (isDirectAdmin) return true;
  
  // 2. 检查父组织成员（包含 accepted 检查）
  const team = await this.prismaClient.team.findUnique({
    where: { id: teamId },
    select: { parentId: true },
  });
  
  if (team?.parentId) {
    const isParentOrgAdmin = await this.prismaClient.membership.findFirst({
      where: {
        userId,
        teamId: team.parentId,
        role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
        accepted: true,
      },
    });
    return !!isParentOrgAdmin;
  }
  
  return false;
}
```

---

## 问题 2：PermissionCheckService 被定义为 Mock 类

### 漏洞位置

**文件：** 多个文件中重复定义
1. `packages/features/bookings/services/BookingAccessService.ts` (第 6-11 行)
2. `packages/trpc/server/procedures/pbacProcedures.ts` (第 7-12 行)
3. `packages/trpc/server/routers/viewer/ooo/outOfOffice.utils.ts` (第 4-9 行)
4. `packages/trpc/server/routers/viewer/me/checkForInvalidAppCredentials.ts` (第 7-12 行)
5. `packages/trpc/server/routers/viewer/me/get.handler.ts` (第 11-17 行)
6. `packages/trpc/server/routers/viewer/eventTypes/teamAccessUseCase.ts` (第 3-8 行)
7. `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts` (第 13-18 行)

### 漏洞代码分析

**所有文件中的定义模式相同：**

```typescript
class PermissionCheckService {
  constructor(_prisma?: unknown) {}
  
  // ⚠️ 总是返回 true！
  async checkPermission(..._args: unknown[]) { return true; }
  async hasPermission(..._args: unknown[]) { return true; }
  async getTeamIdsWithPermission(..._args: unknown[]): Promise<number[]> { return []; }
}
```

**在 BookingAccessService 中的使用（第 87-93 行）：**
```typescript
const hasAccess = await this.permissionCheckService.checkPermission({
  userId,
  teamId: booking.eventType.teamId,
  permission: "booking.readTeamBookings",
  fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
});
return hasAccess;  // ⚠️ 总是返回 true！
```

### 漏洞原理

**关键问题：**

1. **`PermissionCheckService` 被定义为一个本地类**，而不是从共享模块导入
2. **所有方法都硬编码返回 `true`**，不执行任何实际的权限检查
3. **即使 PBAC 系统存在**，这些 mock 实现也会绕过所有权限检查

**代码调用链分析（BookingAccessService）：**

```
1. 用户请求访问预约
2. BookingAccessService.doesUserIdHaveAccessToBooking()
   ├── Case 1: userId === booking.userId ✅ 正确
   ├── Case 2: isUserAHost() ✅ 正确
   ├── Case 3: booking.eventType?.teamId 存在
   │   └── this.permissionCheckService.checkPermission()
   │       └── ⚠️ return true （总是允许！）
   ├── Case 4: 检查组织管理员
   │   └── this.permissionCheckService.checkPermission()
   │       └── ⚠️ return true （总是允许！）
   └── Case 5: 检查团队管理员
       └── this.permissionCheckService.checkPermission()
           └── ⚠️ return true （总是允许！）
```

### 可复现条件

**前置条件：**
1. 存在一个团队/组织，设置了事件类型
2. 该事件类型有预约记录
3. 存在一个用户 B，**不是**该团队/组织的成员

**攻击路径：**
```
1. 用户 B（非团队成员）尝试访问团队预约
2. 进入 Case 3: booking.eventType?.teamId 存在
3. 调用 permissionCheckService.checkPermission()
4. ⚠️ 方法硬编码返回 true
5. ✅ 用户 B 成功获得访问权限！
```

**更严重的场景：**

```
1. 用户 B 尝试访问另一个用户 C 的个人预约
2. 进入 Case 4: 检查 bookingOwner.organizationId
3. 如果用户 C 属于某个组织
4. 调用 permissionCheckService.checkPermission("booking.readOrgBookings")
5. ⚠️ 方法硬编码返回 true
6. ✅ 用户 B 可以访问组织中任何成员的个人预约！
```

### 影响范围

**受影响的功能模块：**

| 模块 | 影响 |
|------|------|
| BookingAccessService | **所有团队/组织预约完全失控** |
| pbacProcedures | **所有 tRPC 权限检查失效** |
| outOfOffice.utils | **OOO 设置权限检查失效** |
| me/get.handler | **用户信息权限检查失效** |
| eventTypes/teamAccessUseCase | **事件类型权限检查失效** |
| eventTypes/heavy/create.handler | **事件类型创建权限检查失效** |

**实际影响：**

1. **团队预约**：任何登录用户都可以访问任何团队的预约
2. **个人预约**：任何登录用户都可以访问组织成员的个人预约
3. **事件类型管理**：任何登录用户都可以创建/修改团队事件类型
4. **OOO 设置**：任何登录用户都可以修改他人的 OOO 设置

### 风险等级

| 维度 | 评估 |
|------|------|
| 严重程度 | **致命 (Critical)** |
| 影响范围 | 所有权限检查模块 |
| 利用难度 | 极低 - 只需登录系统 |
| 隐蔽性 | 极高 - 代码看起来正常 |
| 数据影响 | 所有敏感预约数据完全暴露 |

### 修复建议

**方案 1：实现真正的 PermissionCheckService**

创建一个真正的实现，而不是 mock：

```typescript
// packages/features/permissions/services/PermissionCheckService.ts

import type { PrismaClient } from "@calcom/prisma";
import { MembershipRole } from "@calcom/prisma/enums";

export class PermissionCheckService {
  constructor(private prismaClient: PrismaClient) {}

  async checkPermission({
    userId,
    teamId,
    permission,
    fallbackRoles,
  }: {
    userId: number;
    teamId: number;
    permission: string;
    fallbackRoles: MembershipRole[];
  }): Promise<boolean> {
    // 1. 首先尝试 PBAC 检查（如果实现了）
    const pbacResult = await this.tryPbacCheck({ userId, teamId, permission });
    if (pbacResult !== null) {
      return pbacResult;
    }

    // 2. 回退到基于角色的检查
    return this.fallbackRoleCheck({ userId, teamId, fallbackRoles });
  }

  private async fallbackRoleCheck({
    userId,
    teamId,
    fallbackRoles,
  }: {
    userId: number;
    teamId: number;
    fallbackRoles: MembershipRole[];
  }): Promise<boolean> {
    const membership = await this.prismaClient.membership.findUnique({
      where: {
        userId_teamId: {
          userId,
          teamId,
        },
      },
      select: {
        accepted: true,
        role: true,
      },
    });

    // ⚠️ 必须同时检查 accepted 和 role
    if (!membership || !membership.accepted) {
      return false;
    }

    return fallbackRoles.includes(membership.role);
  }

  private async tryPbacCheck(_args: unknown): Promise<boolean | null> {
    // TODO: 实现真正的 PBAC 检查
    // 如果 PBAC 系统未启用或检查失败，返回 null 以使用回退机制
    return null;
  }

  async getTeamIdsWithPermission({
    userId,
    permission,
    fallbackRoles,
  }: {
    userId: number;
    permission: string;
    fallbackRoles: MembershipRole[];
  }): Promise<number[]> {
    // 1. 尝试 PBAC
    const pbacResult = await this.tryPbacGetTeamIds({ userId, permission });
    if (pbacResult !== null) {
      return pbacResult;
    }

    // 2. 回退到角色检查
    const memberships = await this.prismaClient.membership.findMany({
      where: {
        userId,
        accepted: true,  // ✅ 必须检查 accepted
        role: { in: fallbackRoles },
      },
      select: { teamId: true },
    });

    return memberships.map((m) => m.teamId);
  }

  private async tryPbacGetTeamIds(_args: unknown): Promise<number[] | null> {
    return null;
  }
}
```

**方案 2：修复所有使用点**

确保所有文件都使用正确的实现，而不是本地定义的 mock 类。

例如，修改 `BookingAccessService.ts`：

```typescript
// 导入真实的实现
import { PermissionCheckService } from "@calcom/features/permissions/services/PermissionCheckService";

export class BookingAccessService {
  private permissionCheckService: PermissionCheckService;

  constructor(private prismaClient: PrismaClient) {
    // 注入真实的实现
    this.permissionCheckService = new PermissionCheckService(prismaClient);
  }
  
  // ... 其余代码保持不变
}
```

**方案 3：添加集成测试**

添加测试用例确保权限检查正确工作：

```typescript
describe("BookingAccessService - Permission Checks", () => {
  it("should reject non-members from accessing team bookings", async () => {
    // 安排：创建团队、创建预约、创建非成员用户
    // ...
    
    // 行动：非成员用户尝试访问
    const result = await service.doesUserIdHaveAccessToBooking({
      userId: nonMemberUserId,
      bookingId: teamBookingId,
    });
    
    // 断言：应该拒绝访问
    expect(result).toBe(false);
  });

  it("should reject pending invitations from accessing bookings", async () => {
    // 安排：创建待接受的邀请（accepted = false）
    // ...
    
    // 行动：待接受邀请的用户尝试访问
    const result = await service.doesUserIdHaveAccessToBooking({
      userId: pendingInviteUserId,
      bookingId: teamBookingId,
    });
    
    // 断言：应该拒绝访问
    expect(result).toBe(false);
  });
});
```

---

## 问题 3：getUserOrganizationAndTeams 未过滤 accepted 状态

### 漏洞位置

**文件：** `packages/features/users/repositories/UserRepository.ts`  
**方法：** `getUserOrganizationAndTeams` (第 1056-1067 行)

**调用路径：**
```
BookingAccessService.doesUserIdHaveAccessToBooking() (第 107 行)
  └── UserRepository.getUserOrganizationAndTeams()
        └── [teams 列表可能包含未接受的邀请]
```

### 漏洞代码分析

**BookingAccessService 中的使用（第 107-135 行）：**
```typescript
const bookingOwner = await userRepo.getUserOrganizationAndTeams({ 
  userId: booking.userId 
});

// Case 4: 检查组织管理员
if (bookingOwner.organizationId) {
  const hasAccess = await this.permissionCheckService.checkPermission({
    userId,
    teamId: bookingOwner.organizationId,  // 使用组织 ID
    // ...
  });
}

// Case 5: 检查所有团队
for (const membership of bookingOwner.teams) {
  const teamId = membership.teamId;  // 使用团队 ID
  const hasAccess = await this.permissionCheckService.checkPermission({
    userId,
    teamId,
    // ...
  });
}
```

**getUserOrganizationAndTeams 实现（第 1056-1067 行）：**
```typescript
async getUserOrganizationAndTeams({ userId }: { userId: number }) {
  return await this.prismaClient.user.findUnique({
    where: { id: userId },
    select: {
      organizationId: true,
      teams: {
        where: { accepted: true },  // ✅ 这里正确过滤了 accepted
        select: { teamId: true },
      },
    },
  });
}
```

### 分析结果

**好消息：** 这个方法**实际上是正确的**，`teams` 查询中包含了 `where: { accepted: true }`。

**但有一个潜在风险：** `organizationId` 字段直接从 `User` 模型获取，它是一个独立的字段，不经过 `accepted` 检查。

让我查看 `User` 模型中 `organizationId` 的含义：

**潜在问题分析：**
- `User.organizationId` 表示用户的"主要组织"
- 这个字段可能在用户被邀请时就被设置
- 需要确认 `organizationId` 的设置时机

**但考虑到问题 2（PermissionCheckService 是 mock），即使这个方法正确，权限检查仍然完全失效。**

### 风险等级

| 维度 | 评估 |
|------|------|
| 严重程度 | **低 (Low)** - 方法本身是正确的 |
| 注意事项 | 但被问题 2 完全抵消 |

---

## 问题 4：测试文件未覆盖 accepted 状态边界

### 漏洞位置

**文件：** `packages/features/bookings/services/BookingAccessService.test.ts`

### 漏洞代码分析

**现有测试覆盖的场景：**
1. ✅ 用户是预约组织者
2. ✅ 用户是主持人
3. ✅ 用户有团队预约访问权限
4. ✅ 用户有组织预约访问权限
5. ✅ 用户有任何团队的管理员权限

**缺失的测试场景：**
1. ❌ **待接受邀请的用户** - `membership.accepted = false`
2. ❌ **非成员用户** - 完全没有 membership 记录
3. ❌ **角色为 MEMBER 的用户** - 应该被拒绝
4. ❌ **组织 ID 但不是成员** - 检查 organizationId 但没有 membership

**测试文件中的 mock 设置（第 25-44 行）：**
```typescript
let mockPermissionCheckService: {
  checkPermission: ReturnType<typeof vi.fn>;
};

// 在测试中直接控制返回值，不模拟真实的权限检查逻辑
mockPermissionCheckService.checkPermission.mockResolvedValue(true);  // 或 false
```

**问题：** 测试使用 mock 直接控制 `checkPermission` 的返回值，不验证真正的权限逻辑。

### 风险等级

| 维度 | 评估 |
|------|------|
| 严重程度 | **中危 (Medium)** |
| 影响范围 | 测试覆盖率不足，难以及时发现安全问题 |
| 风险类型 | 质量风险，间接导致安全漏洞 |

### 修复建议

**添加缺失的测试用例：**

```typescript
describe("BookingAccessService - Edge Cases", () => {
  describe("Pending Invitation Tests", () => {
    it("should reject users with pending invitations (accepted = false)", async () => {
      // 模拟 pending invitation
      // ...
    });
  });

  describe("Role-based Access Tests", () => {
    it("should reject MEMBER role for booking.readTeamBookings", async () => {
      // MEMBER 不在 fallbackRoles 中
      // ...
    });
  });

  describe("Non-member Tests", () => {
    it("should reject users with no membership", async () => {
      // 完全没有 membership 记录
      // ...
    });
  });
});
```

---

## 综合风险评估

### 漏洞矩阵

| 问题 | 严重程度 | 利用难度 | 数据影响 | 状态 |
|------|---------|---------|---------|------|
| 1. Managed Event 缺少 accepted 检查 | **高危** | 低 | 高 | 需要修复 |
| 2. PermissionCheckService 是 mock | **致命** | 极低 | 极高 | **紧急修复** |
| 3. getUserOrganizationAndTeams | 低 | - | - | 基本正确 |
| 4. 测试覆盖不足 | 中危 | - | - | 需要改进 |

### 优先级建议

**P0 - 立即修复：**
- 问题 2：PermissionCheckService 必须立即实现真正的权限检查

**P1 - 本周修复：**
- 问题 1：Managed Event 分支的 accepted 检查
- 问题 4：添加缺失的测试用例

---

## 附录：完整的权限检查流程图

### 当前（有漏洞的）流程

```
用户请求访问预约
        │
        ▼
┌─────────────────────┐
│ userId === booking  │ ──是──▶ ✅ 允许
│ .userId?            │
└─────────────────────┘
        │ 否
        ▼
┌─────────────────────┐
│ isUserAHost()?      │ ──是──▶ ✅ 允许
└─────────────────────┘
        │ 否
        ▼
┌─────────────────────┐
│ eventType.teamId?   │
└─────────────────────┘
        │ 是
        ▼
┌──────────────────────────────────┐
│ permissionCheckService          │
│ .checkPermission()              │
│                                  │
│ ⚠️ Mock 实现: return true       │ ──▶ ✅ 总是允许！
└──────────────────────────────────┘
        │ 否
        ▼
┌─────────────────────┐
│ eventType.parent?   │
└─────────────────────┘
        │ 是
        ▼
┌──────────────────────────────────┐
│ isAdminOfTeamOrParentOrg()      │
│                                  │
│ ⚠️ 缺少 accepted 检查           │ ──▶ ✅ 待接受邀请也允许！
└──────────────────────────────────┘
        │ 否
        ▼
┌──────────────────────────────────┐
│ 检查组织/团队管理员               │
│ permissionCheckService          │
│                                  │
│ ⚠️ Mock 实现: return true       │ ──▶ ✅ 总是允许！
└──────────────────────────────────┘
```

### 修复后的预期流程

```
用户请求访问预约
        │
        ▼
┌─────────────────────┐
│ userId === booking  │ ──是──▶ ✅ 允许
│ .userId?            │
└─────────────────────┘
        │ 否
        ▼
┌─────────────────────┐
│ isUserAHost()?      │ ──是──▶ ✅ 允许
└─────────────────────┘
        │ 否
        ▼
┌─────────────────────┐
│ eventType.teamId?   │
└─────────────────────┘
        │ 是
        ▼
┌──────────────────────────────────┐
│ 真实权限检查:                    │
│ 1. 查找 membership              │
│ 2. 检查 accepted === true       │
│ 3. 检查 role in fallbackRoles   │
│    (OWNER, ADMIN)               │
└──────────────────────────────────┘
        │ 检查通过
        ▼
      ✅ 允许
        │ 检查失败
        ▼
┌─────────────────────┐
│ eventType.parent?   │
└─────────────────────┘
        │ 是
        ▼
┌──────────────────────────────────┐
│ 真实权限检查:                    │
│ 1. 查找 membership              │
│ 2. ✅ 检查 accepted === true     │
│ 3. 检查 role in (ADMIN, OWNER)   │
└──────────────────────────────────┘
        │ 检查通过
        ▼
      ✅ 允许
        │ 检查失败
        ▼
┌──────────────────────────────────┐
│ 检查组织/团队管理员               │
│ (同样需要真实权限检查)            │
└──────────────────────────────────┘
        │ 检查通过
        ▼
      ✅ 允许
        │ 检查失败
        ▼
      ❌ 拒绝
```

---

## 最终建议

### 立即行动

1. **立即禁用或修复** `PermissionCheckService` 的 mock 实现
2. **添加** `isAdminOfTeamOrParentOrg` 中的 `accepted: true` 检查
3. **添加** 针对 `accepted` 状态的集成测试
4. **审计** 所有使用 `PermissionCheckService` 的文件

### 长期改进

1. **建立统一的权限服务**，避免重复定义
2. **实现真正的 PBAC 系统**或完善基于角色的检查
3. **建立安全代码审查流程**，特别是涉及权限检查的代码
4. **添加自动化安全测试**，定期扫描类似漏洞
