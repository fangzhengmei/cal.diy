# Cal.diy 预约访问权限链路分析报告 - 最终复核版 (R3)

**分析日期：** 2026-05-10  
**分析目标：** 复核成员 accepted 状态在预约访问权限链路中的实际生效边界，分层确认可复现问题与仅在当前桩代码实现下成立的风险

---

## 目录

1. [执行摘要](#执行摘要)
2. [复核方法论](#复核方法论)
3. [问题分层确认](#问题分层确认)
   - [3.1 可直接确认的真实漏洞](#31-可直接确认的真实漏洞)
   - [3.2 仅在当前桩代码实现下成立的风险](#32-仅在当前桩代码实现下成立的风险)
   - [3.3 修正后的问题3表述](#33-修正后的问题3表述)
4. [BookingAccessService 权限链路完整分析](#bookingaccessservice-权限链路完整分析)
5. [收敛版结论](#收敛版结论)
6. [修复优先级与建议](#修复优先级与建议)
7. [附录：关键代码证据](#附录关键代码证据)

---

## 执行摘要

### 关键发现（分层后）

| 编号 | 问题 | 严重程度 | 分类 | 可利用性 | 优先级 |
|------|------|---------|------|---------|--------|
| 1 | Managed Event 分支 `isAdminOfTeamOrParentOrg` 缺少 `accepted` 检查 | **高危** | 真实漏洞 | 可直接复现 | **P1 - 立即修复** |
| 2 | PermissionCheckService 使用本地 stub 实现 | **高危（当前状态）** | 待实现功能 | 取决于 PBAC 实现进度 | **P2 - 技术债务** |
| 3 | 测试覆盖不足 | 中危 | 测试缺失 | N/A | P2 |
| 4 | (已修正) getUserOrganizationAndTeams 有 accepted 过滤 | **无问题** | 代码正确 | N/A | N/A |

### 核心修正

1. **问题2（PermissionCheckService）**：这不是"恶意代码"或"黑客后门"，而是**待实现的 PBAC 系统桩代码**。设计上期望从 `@calcom/features/pbac/services/permission-check.service` 导入真正的实现。

2. **问题3（getUserOrganizationAndTeams）**：原始分析存在表述冲突。**实际代码是正确的**，有 `where: { accepted: true }` 过滤，不是漏洞。

3. **问题1（isAdminOfTeamOrParentOrg）**：**确认是真实漏洞**。与同文件中的 `isAdminOrOwnerOfTeam` 方法对比，显式缺少 `accepted` 检查。

---

## 复核方法论

本复核采用**分层验证策略**：

### 分层标准

#### A. 可直接确认的真实漏洞
- 存在代码不一致（对比同一模块中的其他方法）
- 不依赖外部依赖或待实现功能
- 可以通过构造数据库状态直接触发
- 有明确的安全边界违反

#### B. 仅在当前桩代码实现下成立的风险
- 代码明确标记为 stub/placeholder
- 设计上有外部依赖路径（如 `@calcom/features/pbac/...`）
- 测试文件中明确注释这是临时实现
- 风险随着真正实现的完成会消失

#### C. 不是问题（表述冲突修正）
- 代码实际实现是正确的
- 原始分析存在误解或表述矛盾
- 需要澄清而不是修复

---

## 问题分层确认

## 3.1 可直接确认的真实漏洞

### 漏洞 1：Managed Event 分支缺少 `accepted` 状态检查

**状态：** ✅ 确认真实漏洞  
**风险等级：** 高危  
**分类：** A 类 - 可直接确认的真实漏洞  
**利用难度：** 低  
**影响范围：** 所有 Managed Event（子事件类型）的预约访问权限

---

#### 代码证据

**位置：** `packages/features/users/repositories/UserRepository.ts:1014-1038`

**有漏洞的代码：**
```typescript
// UserRepository.ts:1014-1038
async isAdminOfTeamOrParentOrg({ userId, teamId }: { userId: number; teamId: number }) {
  const membershipQuery = {
    members: {
      some: {
        userId,
        role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
        // ⚠️ 缺少 accepted: true 检查！
      },
    },
  };
  const teams = await this.prismaClient.team.findMany({
    where: {
      id: teamId,
      OR: [
        membershipQuery,
        {
          parent: { ...membershipQuery },  // 同样缺少 accepted 检查
        },
      ],
    },
    // ...
  });
  return !!teams.length;
}
```

**对比同一文件中正确的实现：**
```typescript
// UserRepository.ts:1039-1054
async isAdminOrOwnerOfTeam({ userId, teamId }: { userId: number; teamId: number }) {
  const isAdminOrOwnerOfTeam = await this.prismaClient.membership.findUnique({
    where: {
      userId_teamId: { userId, teamId },
      role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
      accepted: true,  // ✅ 正确检查了 accepted 状态
    },
    select: { id: true },
  });
  return !!isAdminOrOwnerOfTeam;
}
```

**调用链位置：** `BookingAccessService.ts:96-103`
```typescript
// BookingAccessService.ts:96-103
// For managed events (child event types), check the parent's teamId
if (booking.eventType?.parent?.teamId) {
  const isAdminOrUser = await userRepo.isAdminOfTeamOrParentOrg({
    userId,
    teamId: booking.eventType.parent.teamId,
  });
  return isAdminOrUser;
}
```

---

#### 问题分析

**为什么这是漏洞？**

1. **不一致性证明**：同文件中 `isAdminOrOwnerOfTeam` 明确有 `accepted: true`，但 `isAdminOfTeamOrParentOrg` 没有。这不是"设计选择"，而是遗漏。

2. **安全语义**：`membership.accepted` 字段的业务语义是"成员是否接受了邀请"。未接受邀请的用户应该被视为"还不是正式成员"，不应该享有任何权限。

3. **调用场景**：这个方法专门用于 **Managed Event** 权限检查。如果没有 `accepted` 检查，任何被邀请但尚未接受邀请的"准管理员"都可以访问团队的所有 Managed Event 预约。

---

#### 可复现条件

**前置条件：**
1. 存在一个组织或团队
2. 用户被邀请加入该组织/团队，角色为 `ADMIN` 或 `OWNER`
3. 用户 **尚未接受邀请**（`membership.accepted = false`）
4. 该团队有 Managed Event（子事件类型，有 `parentId`）

**攻击路径：**
```
1. 攻击者（未接受邀请的用户）登录系统
2. 构造对 Managed Event 预约的访问请求
3. BookingAccessService 进入 managed event 分支
4. 调用 isAdminOfTeamOrParentOrg()
5. Prisma 查询只检查 role，不检查 accepted
6. 返回 true，攻击者获得访问权限
```

**数据库状态构造：**
```sql
-- 创建用户
INSERT INTO "User" (id, email, name) VALUES (100, 'attacker@example.com', 'Attacker');

-- 创建组织/团队
INSERT INTO "Team" (id, name, isOrganization, parentId) VALUES (50, 'Test Org', true, null);

-- 创建未接受的管理员邀请
INSERT INTO "Membership" (userId, teamId, role, accepted)
VALUES (100, 50, 'ADMIN', false);  -- accepted = false

-- 创建团队事件类型（父事件）
INSERT INTO "EventType" (id, title, teamId, userId) VALUES (200, 'Parent Event', 50, null);

-- 创建 Managed Event（子事件）
INSERT INTO "EventType" (id, title, parentId, teamId, userId) 
VALUES (201, 'Managed Child Event', 200, 50, null);

-- 创建预约
INSERT INTO "Booking" (id, uid, userId, eventTypeId) 
VALUES (999, 'booking-uid-123', 1, 201);
```

**预期行为（修复后）：** 用户 100（未接受邀请）无法访问 booking 999  
**实际行为（当前）：** 用户 100 可以访问 booking 999

---

#### 影响范围

| 场景 | 影响 | 修复前 | 修复后 |
|------|------|--------|--------|
| 未接受邀请的 ADMIN | 可访问 Managed Event 预约 | ✅ 可访问 | ❌ 不可访问 |
| 已接受邀请的 ADMIN | 可访问 Managed Event 预约 | ✅ 可访问 | ✅ 可访问（正常） |
| 未接受邀请的 MEMBER | 可访问？ | ❌ 不可访问（role 不满足） | ❌ 不可访问 |
| 组织父级的未接受邀请 ADMIN | 可访问子团队的 Managed Event | ✅ 可访问 | ❌ 不可访问 |

**特别注意：** 漏洞同时影响**团队**和**父组织**两个层级（代码中的 `OR: [membershipQuery, { parent: { ...membershipQuery } }]`）。

---

## 3.2 仅在当前桩代码实现下成立的风险

### 问题 2：PermissionCheckService 使用本地 stub 实现

**状态：** ⚠️ 待实现功能，不是漏洞  
**风险等级：** 高危（当前状态下），但会随 PBAC 实现完成而消失  
**分类：** B 类 - 仅在当前桩代码实现下成立的风险  
**技术状态：** 技术债务，明确设计为临时实现

---

#### 代码证据

**发现的 stub 文件列表（共 10+ 个）：**

| 文件路径 | 本地 stub 定义位置 | 被调用位置 |
|---------|-------------------|-----------|
| `packages/features/bookings/services/BookingAccessService.ts` | 第 6-11 行 | 第 87 行、115 行、128 行 |
| `packages/trpc/server/routers/viewer/eventTypes/util.ts` | 第 15-20 行 | 第 161 行 |
| `packages/trpc/server/routers/viewer/eventTypes/teamAccessUseCase.ts` | 第 3-8 行 | 第 24 行 |
| `packages/trpc/server/routers/viewer/eventTypes/heavy/create.handler.ts` | 第 13-18 行 | 第 60 行 |
| `packages/trpc/server/routers/viewer/eventTypes/getActiveOnOptions.handler.ts` | 第 12-19 行 | 第 222 行 |
| `packages/trpc/server/routers/viewer/eventTypes/getUserEventGroups.handler.ts` | 第 14-19 行 | 第 67 行 |
| `packages/trpc/server/routers/viewer/bookings/get.handler.ts` | 第 21-26 行 | 多处 |
| `packages/trpc/server/routers/viewer/ooo/outOfOffice.utils.ts` | 第 4-9 行 | 第 12 行 |
| `packages/trpc/server/routers/viewer/me/checkForInvalidAppCredentials.ts` | 第 7-12 行 | 第 23 行 |
| `packages/trpc/server/routers/viewer/me/get.handler.ts` | 第 11-18 行 | 第 110 行 |
| `packages/trpc/server/procedures/pbacProcedures.ts` | 第 7-12 行 | 多处 |

**Stub 实现（所有文件重复）：**
```typescript
// 所有文件中的共同模式
class PermissionCheckService {
  constructor(_prisma?: unknown) {}
  async checkPermission(..._args: unknown[]) { return true; }  // ⚠️ 总是返回 true
  async hasPermission(..._args: unknown[]) { return true; }    // ⚠️ 总是返回 true
  async getTeamIdsWithPermission(..._args: unknown[]): Promise<number[]> { return []; }
}
```

**关键证据：测试文件明确标记为 stub**
```typescript
// packages/trpc/server/routers/viewer/eventTypes/__tests__/util.test.ts:205
// PermissionCheckService stub always returns true, so org admin access is always granted
```

**关键证据：测试文件 mock 了设计中的导入路径**
```typescript
// packages/trpc/server/routers/viewer/me/get.handler.test.ts:51-53
vi.mock("@calcom/features/pbac/services/permission-check.service", () => ({
  PermissionCheckService: MockPermissionCheckService,
}));
```

**关键证据：目标目录不存在**
```
packages/features/pbac/  <-- 这个目录在仓库中不存在
```

---

#### 问题分析

**为什么这不是"漏洞"而是"技术债务"？**

1. **设计意图明确**：测试文件 mock 的路径 `@calcom/features/pbac/services/permission-check.service` 清楚地说明了**设计上期望有一个真正的 PBAC 服务**。

2. **代码模式一致**：所有文件都定义了相同的 stub 类，这是典型的**待实现占位符模式**，不是疏忽。

3. **fallback 机制设计**：
   ```typescript
   // BookingAccessService.ts:87-92
   const hasAccess = await this.permissionCheckService.checkPermission({
     userId,
     teamId,
     permission: "booking.readTeamBookings",
     fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],  // ← 设计了 fallback
   });
   ```
   代码设计了 `fallbackRoles` 参数，说明**当 PBAC 启用时使用 PBAC，否则回退到角色检查**。

**当前行为分析：**

| 代码位置 | 当前行为（stub） | 设计意图（真正实现后） |
|---------|-----------------|----------------------|
| 总是返回 `true` | 权限检查绕过 | 基于 PBAC 策略判定 |
| `fallbackRoles` 被忽略 | 无角色检查 | PBAC 禁用时回退角色检查 |
| 多个文件重复定义 | 分散式 stub | 从共享模块导入 |

**这意味着什么？**

- **当前状态**：`checkPermission()` 总是返回 `true`，**所有权限检查被绕过**
- **设计状态**：应该有真正的 PBAC 实现，或者至少使用 `fallbackRoles` 进行角色检查
- **风险**：**当前确实有风险**，但这是**待实现功能的阶段性风险**，不是设计错误

---

#### 当前影响评估

**在 PBAC 实现之前，这些路径的权限控制完全失效：**

1. **BookingAccessService 中的三条路径：**
   - Case 3：团队预约（`booking.eventType?.teamId`）
   - Case 4：组织管理员（`bookingOwner.organizationId`）
   - Case 5：任意团队管理员（`bookingOwner.teams` 循环）

2. **影响范围（当前）：**
   ```
   任何登录用户 → 可以访问任何团队的预约
   任何登录用户 → 可以访问任何组织成员的个人预约
   ```

**区分：这与漏洞 1 的本质区别**

| 维度 | 漏洞 1（isAdminOfTeamOrParentOrg） | 问题 2（PermissionCheckService stub） |
|------|-----------------------------------|-------------------------------------|
| 类型 | 编码错误（遗漏检查） | 待实现功能（技术债务） |
| 同一文件对比 | 有正确实现作为对照（isAdminOrOwnerOfTeam） | 所有地方都是 stub |
| 设计模式 | 应该立即修复 | 需要完整功能实现 |
| 修复方式 | 添加 accepted: true | 实现 PBAC 服务或添加 fallback 逻辑 |

---

## 3.3 修正后的问题3表述

### 原始问题3的修正：getUserOrganizationAndTeams 有 accepted 过滤

**状态：** ✅ 代码正确，不是漏洞  
**分类：** C 类 - 不是问题（表述冲突修正）

---

#### 代码证据

**正确实现：**
```typescript
// UserRepository.ts:1056-1067
async getUserOrganizationAndTeams({ userId }: { userId: number }) {
  return await this.prismaClient.user.findUnique({
    where: { id: userId },
    select: {
      organizationId: true,
      teams: {
        where: { accepted: true },  // ✅ 正确过滤了 accepted 状态
        select: { teamId: true },
      },
    },
  });
}
```

**调用位置：**
```typescript
// BookingAccessService.ts:107
const bookingOwner = await userRepo.getUserOrganizationAndTeams({ userId: booking.userId });
```

---

#### 修正说明

**原始分析中的表述冲突：**
- 标题："getUserOrganizationAndTeams 未过滤 accepted 状态" ❌
- 实际分析："teams 查询中包含了 `where: { accepted: true }`" ✅

**结论：**
- 这段代码是**正确的**
- 这**不是漏洞**
- 原始分析存在标题与内容不一致的错误，已修正

---

## BookingAccessService 权限链路完整分析

让我系统梳理 `BookingAccessService.doesUserIdHaveAccessToBooking()` 的完整权限检查链路。

### 5 层检查结构

```typescript
async doesUserIdHaveAccessToBooking({ userId, bookingUid, bookingId }): Promise<boolean> {
  // Case 1: 组织者检查
  if (userId === booking.userId) return true;
  
  // Case 2: 主持人检查
  if (this.isUserAHost(userId, booking)) return true;
  
  // Case 3: 团队管理员检查（使用 PermissionCheckService）
  if (booking.eventType?.teamId) {
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId, teamId,
      permission: "booking.readTeamBookings",
      fallbackRoles: [OWNER, ADMIN],
    });
    return hasAccess;  // ⚠️ 当前 stub 总是返回 true
  }
  
  // Case 4: Managed Event 分支（漏洞所在！）
  if (booking.eventType?.parent?.teamId) {
    const isAdminOrUser = await userRepo.isAdminOfTeamOrParentOrg({
      userId,
      teamId: booking.eventType.parent.teamId,
    });
    return isAdminOrUser;  // ❌ isAdminOfTeamOrParentOrg 缺少 accepted 检查
  }
  
  // Case 5: 个人预约 - 组织/团队管理员（使用 PermissionCheckService）
  if (!booking.userId) return false;
  const bookingOwner = await userRepo.getUserOrganizationAndTeams({ userId: booking.userId });
  
  // Case 5a: 组织管理员
  if (bookingOwner.organizationId) {
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId, teamId: orgId,
      permission: "booking.readOrgBookings",
      fallbackRoles: [OWNER, ADMIN],
    });
    if (hasAccess) return true;  // ⚠️ 当前 stub 总是返回 true
  }
  
  // Case 5b: 任意团队管理员
  for (const membership of bookingOwner.teams) {
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId, teamId: membership.teamId,
      permission: "booking.readTeamBookings",
      fallbackRoles: [OWNER, ADMIN],
    });
    if (hasAccess) return true;  // ⚠️ 当前 stub 总是返回 true
  }
  
  return false;
}
```

### 各层权限检查的当前状态

| Case | 检查类型 | 权限控制方式 | 当前状态 | 风险评估 |
|------|---------|-------------|---------|---------|
| 1 | 组织者 | 直接 ID 对比 | ✅ 正确 | 无 |
| 2 | 主持人 | Host/User 映射匹配 | ✅ 正确 | 无 |
| 3 | 团队管理员 | PermissionCheckService | ⚠️ stub 总是 true | 待实现 |
| 4 | Managed Event | isAdminOfTeamOrParentOrg | ❌ 缺少 accepted 检查 | **真实漏洞** |
| 5a | 组织管理员 | PermissionCheckService | ⚠️ stub 总是 true | 待实现 |
| 5b | 任意团队管理员 | PermissionCheckService | ⚠️ stub 总是 true | 待实现 |

### 关键观察

1. **Case 1 和 Case 2 是正确的**：直接的 ID 对比和 Host 映射检查，不涉及 membership 状态
2. **Case 3、5a、5b 使用 PermissionCheckService**：当前是 stub，设计上有 fallback 机制
3. **Case 4（Managed Event）直接使用 UserRepository**：这里是**真实漏洞**所在

**为什么 Case 4 特别危险？**

- 它**不经过 PermissionCheckService**
- 直接调用 `isAdminOfTeamOrParentOrg()`
- 这个方法**显式缺少** `accepted` 检查
- **即使 PermissionCheckService 实现了，这个漏洞仍然存在**

---

## 收敛版结论

### 确认的真实漏洞（必须修复）

#### 漏洞 1：Managed Event 分支缺少 accepted 检查

**结论：** 这是一个**确认的高危漏洞**。

**证据链：**
1. `isAdminOfTeamOrParentOrg()` 与 `isAdminOrOwnerOfTeam()` 在同一文件中
2. 后者有 `accepted: true`，前者没有
3. 这是**编码错误**，不是设计选择
4. 可以通过构造数据库状态直接触发
5. **不依赖** PermissionCheckService 的实现状态

**风险：** 任何被邀请但未接受邀请的用户，如果角色是 ADMIN/OWNER，都可以访问 Managed Event 的预约数据。

---

### 待实现的技术债务（需要跟踪）

#### 问题 2：PermissionCheckService 是 stub

**结论：** 这是**技术债务**，不是"漏洞"，但**当前状态下确实有风险**。

**证据链：**
1. 所有文件都使用相同的 stub 模式
2. 测试文件明确注释这是 stub
3. 测试文件 mock 了 `@calcom/features/pbac/services/permission-check.service`
4. 目标目录 `packages/features/pbac/` 不存在
5. 代码设计了 `fallbackRoles` 机制

**风险：** 当前状态下，所有使用 `PermissionCheckService` 的权限检查都被绕过。

**但需注意：**
- 这是**阶段性风险**，随着 PBAC 实现完成会消失
- 或者，可以先实现 `fallbackRoles` 逻辑作为过渡

---

### 已修正的误解

#### 问题 3：getUserOrganizationAndTeams 不是漏洞

**结论：** 代码正确，`teams` 查询有 `where: { accepted: true }`。

---

## 修复优先级与建议

### 优先级矩阵

| 优先级 | 问题 | 行动项 | 时间要求 |
|--------|------|--------|---------|
| **P0 - 立即** | 漏洞 1：isAdminOfTeamOrParentOrg 缺少 accepted | 添加 `accepted: true` 检查 | 本周内 |
| **P1 - 短期** | 问题 2：PermissionCheckService stub | 添加 fallback 逻辑作为过渡 | 2 周内 |
| **P2 - 中期** | 问题 2：实现真正的 PBAC | 创建 `@calcom/features/pbac/` 模块 | 里程碑内 |
| **P2 - 中期** | 测试覆盖不足 | 添加 accepted 状态边界测试 | 与漏洞 1 同步 |

---

### 详细修复方案

#### P0：修复漏洞 1

**文件：** `packages/features/users/repositories/UserRepository.ts:1014-1038`

**修复代码：**
```typescript
// 修复前
const membershipQuery = {
  members: {
    some: {
      userId,
      role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
      // 缺少 accepted: true
    },
  },
};

// 修复后
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

**影响评估：**
- 改动范围：1 个方法，2 行代码
- 风险：低（使权限检查更严格）
- 回退测试：确认已接受邀请的管理员仍能正常访问

---

#### P1：为 PermissionCheckService 添加 fallback 逻辑

**临时方案：** 在真正的 PBAC 实现之前，先让 stub 使用 `fallbackRoles` 进行检查。

**推荐实现：**
```typescript
class PermissionCheckService {
  constructor(private prisma?: PrismaClient) {}

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
    // TODO: 当 PBAC 实现后，替换此处
    // 临时使用 fallback 逻辑
    if (!this.prisma) return false;

    const membership = await this.prisma.membership.findUnique({
      where: {
        userId_teamId: { userId, teamId },
        role: { in: fallbackRoles },
        accepted: true,  // ✅ 必须检查 accepted
      },
    });

    return !!membership;
  }

  // ... 其他方法类似
}
```

**优点：**
1. 立即修复权限绕过问题
2. 不破坏现有 API
3. 为真正的 PBAC 实现保留接口

---

#### P2：实现真正的 PBAC 系统

**路径：** 创建 `packages/features/pbac/` 目录结构

**核心文件：**
- `services/permission-check.service.ts` - 真正的 PermissionCheckService
- `lib/policy-engine.ts` - 策略评估引擎
- `repositories/role-permission.repository.ts` - 权限存储

**迁移步骤：**
1. 创建 pbac 包
2. 实现基于 `RolePermission` 表的检查
3. 逐个替换各文件中的本地 stub 类
4. 统一从 `@calcom/features/pbac/services/permission-check.service` 导入

---

#### P2：添加测试覆盖

**缺失的测试场景：**

1. **accepted=false 的管理员**
   - 构造：用户被邀请，role=ADMIN，accepted=false
   - 预期：不能访问 Managed Event 预约
   - 覆盖：漏洞 1

2. **非成员用户**
   - 构造：用户不在团队中
   - 预期：不能访问团队预约
   - 覆盖：边界条件

3. **MEMBER 角色用户**
   - 构造：用户在团队中，role=MEMBER，accepted=true
   - 预期：不能访问（需要 ADMIN/OWNER）
   - 覆盖：角色权限边界

4. **组织层级权限**
   - 构造：用户是组织管理员，不是子团队直接成员
   - 预期：应该能访问子团队的预约
   - 覆盖：组织-团队层级继承

---

## 附录：关键代码证据

### A. isAdminOfTeamOrParentOrg 完整代码

```typescript
// packages/features/users/repositories/UserRepository.ts:1014-1038
async isAdminOfTeamOrParentOrg({ userId, teamId }: { userId: number; teamId: number }) {
  const membershipQuery = {
    members: {
      some: {
        userId,
        role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
        // ⚠️ 缺少 accepted: true - 这是漏洞
      },
    },
  };
  const teams = await this.prismaClient.team.findMany({
    where: {
      id: teamId,
      OR: [
        membershipQuery,
        {
          parent: { ...membershipQuery },  // 同样缺少 accepted 检查
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

### B. isAdminOrOwnerOfTeam 完整代码（对照）

```typescript
// packages/features/users/repositories/UserRepository.ts:1039-1054
async isAdminOrOwnerOfTeam({ userId, teamId }: { userId: number; teamId: number }) {
  const isAdminOrOwnerOfTeam = await this.prismaClient.membership.findUnique({
    where: {
      userId_teamId: {
        userId,
        teamId,
      },
      role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
      accepted: true,  // ✅ 正确检查了 accepted
    },
    select: {
      id: true,
    },
  });
  return !!isAdminOrOwnerOfTeam;
}
```

### C. PermissionCheckService stub 示例

```typescript
// packages/features/bookings/services/BookingAccessService.ts:6-11
class PermissionCheckService {
  constructor(_prisma?: unknown) {}
  async checkPermission(..._args: unknown[]) { return true; }
  async hasPermission(..._args: unknown[]) { return true; }
  async getTeamIdsWithPermission(..._args: unknown[]): Promise<number[]> { return []; }
}
```

### D. 测试文件中的 stub 注释

```typescript
// packages/trpc/server/routers/viewer/eventTypes/__tests__/util.test.ts:205
// PermissionCheckService stub always returns true, so org admin access is always granted
```

### E. 测试文件中的 mock 路径

```typescript
// packages/trpc/server/routers/viewer/me/get.handler.test.ts:51-53
vi.mock("@calcom/features/pbac/services/permission-check.service", () => ({
  PermissionCheckService: MockPermissionCheckService,
}));
```

### F. getUserOrganizationAndTeams 完整代码

```typescript
// packages/features/users/repositories/UserRepository.ts:1056-1067
async getUserOrganizationAndTeams({ userId }: { userId: number }) {
  return await this.prismaClient.user.findUnique({
    where: { id: userId },
    select: {
      organizationId: true,
      teams: {
        where: { accepted: true },  // ✅ 代码正确，有过滤
        select: { teamId: true },
      },
    },
  });
}
```

---

**报告结束**
