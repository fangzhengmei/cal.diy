# BookingAccessService - Managed Event 分支可达性深度分析

## 执行摘要

本报告聚焦于 `BookingAccessService` 中 **managed event 权限分支**的可达性证明。通过深入分析 `eventType.teamId` 与 `eventType.parent.teamId` 在真实数据中的关系、查询选择字段以及分支顺序逻辑，**确认该分支确实可实际触发**，且存在一个**高危漏洞**：`isAdminOfTeamOrParentOrg()` 方法缺少 `accepted` 状态检查。

---

## 1. 数据模型关系分析

### 1.1 EventType 模型定义

**文件：** `packages/prisma/schema.prisma:156-304`

```prisma
model EventType {
  id        Int     @id @default(autoincrement())
  // ...
  teamId    Int?
  team      Team?   @relation(fields: [teamId], references: [id])
  
  parentId  Int?
  parent    EventType? @relation("managed_eventtype", fields: [parentId], references: [id])
  children  EventType[]  @relation("managed_eventtype")
  
  schedulingType SchedulingType?
  // ...
}
```

**关键字段：**
- `teamId`: 事件类型所属团队（可为 null）
- `parentId`: 父事件类型 ID（可为 null）
- `parent`: 父事件类型关联
- `schedulingType`: 调度类型，`MANAGED` 表示托管事件类型

### 1.2 真实数据中的关系约束

通过分析测试用例和代码逻辑，发现 managed event（托管事件）的父子关系如下：

**父事件类型（Managed Event Template）：**
- `teamId`: 非 null（属于某个团队）
- `schedulingType`: `MANAGED`
- `parentId`: null

**子事件类型（Assigned Event）：**
- `teamId`: **null**（不属于任何团队，属于个人用户）
- `userId`: 非 null（分配给特定用户）
- `parentId`: 指向父事件类型的 ID
- `schedulingType`: **null**

**证据：** `packages/features/bookings/lib/handleNewBooking/global-booking-limits.test.ts:106-126`

```typescript
// 父事件类型
{
  id: 3,
  teamId: 1,           // ← 有 teamId
  schedulingType: SchedulingType.MANAGED,
  // ...
}

// 子事件类型
{
  id: 4,
  userId: 101,         // ← 有 userId
  parent: { id: 3 },   // ← 指向父事件类型
  // ← 没有 teamId（即 null）
}
```

### 1.3 getTeamIdFromEventType 工具函数验证

**文件：** `packages/lib/getTeamIdFromEventType.ts:1-28`

```typescript
export async function getTeamIdFromEventType({ eventType }) {
  if (eventType?.team?.id) {
    return eventType.team.id;
  }
  
  // 如果是 managed event，需要从父事件类型获取 teamId
  if (eventType?.parentId) {
    const managedEvent = await prisma.eventType.findUnique({
      where: { id: eventType.parentId },
      select: { teamId: true },
    });
    return managedEvent?.teamId;
  }
}
```

这个函数明确表明：**子事件类型自身没有 teamId，需要通过 parentId 从父事件类型获取**。

---

## 2. 查询选择字段分析

### 2.1 BookingRepository 查询

**文件：** `packages/features/bookings/repositories/BookingRepository.ts:481-575`

`findByUidIncludeEventType` 和 `findByIdIncludeEventType` 的 select 字段：

```typescript
eventType: {
  select: {
    teamId: true,           // ← Case 3 使用
    parent: {
      select: {
        teamId: true,       // ← Case 4 使用
      },
    },
    hosts: { /* ... */ },
    users: { /* ... */ },
  },
},
```

**关键观察：**
- 查询同时获取了 `eventType.teamId` 和 `eventType.parent.teamId`
- 没有获取 `eventType.parentId` 或 `eventType.schedulingType`
- 判断逻辑完全依赖于这两个字段的 null/not null 状态

---

## 3. 分支顺序与短路逻辑分析

### 3.1 BookingAccessService 分支流程

**文件：** `packages/features/bookings/services/BookingAccessService.ts:56-138`

```typescript
async doesUserIdHaveAccessToBooking({ userId, bookingUid, bookingId }) {
  // Case 1: 用户是预约组织者
  if (userId === booking.userId) return true;  // 短路返回
  
  // Case 2: 用户是主持人
  if (this.isUserAHost(userId, booking)) return true;  // 短路返回
  
  // Case 3: 检查 eventType.teamId（团队预约）
  if (booking.eventType?.teamId) {
    const hasAccess = await this.permissionCheckService.checkPermission({
      userId, teamId,
      permission: "booking.readTeamBookings",
      fallbackRoles: [OWNER, ADMIN],
    });
    return hasAccess;  // 短路返回
  }
  
  // Case 4: Managed Event 分支 - 检查 parent.teamId
  if (booking.eventType?.parent?.teamId) {
    const isAdminOrUser = await userRepo.isAdminOfTeamOrParentOrg({
      userId,
      teamId: booking.eventType.parent.teamId,
    });
    return isAdminOrUser;  // 短路返回 ← 漏洞所在！
  }
  
  // Case 5: 个人预约 - 检查组织者的组织/团队
  // ...
}
```

### 3.2 可达性证明

**对于子事件类型（Managed Event）的预约：**

| 字段 | 值 | 分支判断 |
|------|-----|---------|
| `booking.eventType.teamId` | `null` | Case 3: `if (null)` → **false，跳过** |
| `booking.eventType.parent?.teamId` | 父事件类型的 teamId（非 null） | Case 4: `if (nonNull)` → **true，进入分支** |

**结论：Managed Event 分支（Case 4）确实可以被触发。**

### 3.3 分支顺序的影响

Case 3（`eventType.teamId`）在 Case 4（`eventType.parent.teamId`）之前执行：

- **对于普通团队预约**：`eventType.teamId` 非 null → 进入 Case 3
- **对于子事件类型预约**：`eventType.teamId` 为 null → 跳过 Case 3 → 进入 Case 4

**这是正确的设计意图**，但 Case 4 使用了错误的权限检查方法。

---

## 4. 漏洞分析

### 4.1 漏洞位置

**文件：** `packages/features/users/repositories/UserRepository.ts:1014-1038`

**方法：** `isAdminOfTeamOrParentOrg()`

```typescript
async isAdminOfTeamOrParentOrg({ userId, teamId }) {
  const membershipQuery = {
    members: {
      some: {
        userId,
        role: { in: [ADMIN, OWNER] },
        // ⚠️ 缺少 accepted: true 检查！
      },
    },
  };
  
  const teams = await this.prismaClient.team.findMany({
    where: {
      id: teamId,
      OR: [
        membershipQuery,                    // 团队自身的管理员
        { parent: { ...membershipQuery } }, // 父组织的管理员
      ],
    },
    select: { id: true },
  });
  
  return !!teams.length;
}
```

### 4.2 对比正确实现

同一文件中的 `isAdminOrOwnerOfTeam()` 方法有正确的 `accepted` 检查：

**文件：** `packages/features/users/repositories/UserRepository.ts:1039-1054`

```typescript
async isAdminOrOwnerOfTeam({ userId, teamId }) {
  const isAdminOrOwnerOfTeam = await this.prismaClient.membership.findUnique({
    where: {
      userId_teamId: { userId, teamId },
      role: { in: [ADMIN, OWNER] },
      accepted: true,  // ✅ 正确检查
    },
    select: { id: true },
  });
  return !!isAdminOrOwnerOfTeam;
}
```

### 4.3 漏洞原理

`isAdminOfTeamOrParentOrg()` 使用 `team.members.some()` 查询，**没有过滤 `accepted: true`**：

- **未接受邀请**的用户（`accepted: false`）如果有 `ADMIN` 或 `OWNER` 角色
- 也会被误认为是团队/组织管理员
- 从而获得访问 Managed Event 预约的权限

---

## 5. 最小可复现数据样例

### 5.1 数据库状态

```sql
-- 团队
INSERT INTO "Team" (id, name, slug) VALUES (100, 'Acme Team', 'acme-team');

-- 用户 A：团队所有者（已接受邀请）
INSERT INTO "User" (id, email) VALUES (1, 'owner@acme.com');
INSERT INTO "Membership" (userId, teamId, role, accepted) 
VALUES (1, 100, 'OWNER', true);

-- 用户 B：待接受邀请的管理员
INSERT INTO "User" (id, email) VALUES (2, 'pending-admin@acme.com');
INSERT INTO "Membership" (userId, teamId, role, accepted) 
VALUES (2, 100, 'ADMIN', false);  -- ⚠️ accepted = false

-- 用户 C：预约组织者
INSERT INTO "User" (id, email) VALUES (3, 'organizer@acme.com');
INSERT INTO "Membership" (userId, teamId, role, accepted) 
VALUES (3, 100, 'MEMBER', true);

-- 父事件类型（Managed Event Template）
INSERT INTO "EventType" (id, title, slug, teamId, schedulingType, length)
VALUES (1000, 'Team Meeting', 'team-meeting', 100, 'MANAGED', 30);

-- 子事件类型（分配给用户 C）
INSERT INTO "EventType" (id, title, slug, userId, parentId, length)
VALUES (1001, 'Team Meeting - User C', 'team-meeting-user-c', 3, 1000, 30);
--                               ↑ teamId = null (隐式)

-- 预约记录（使用子事件类型）
INSERT INTO "Booking" (id, uid, userId, eventTypeId, status, startTime, endTime)
VALUES (9999, 'booking-uid-xyz', 3, 1001, 'ACCEPTED', NOW(), NOW());
```

### 5.2 攻击流程

```
1. 用户 B（pending-admin@acme.com）登录系统
2. 用户 B 尝试访问预约 uid = 'booking-uid-xyz'
3. BookingAccessService 执行流程：
   a. Case 1: userId(2) === booking.userId(3)? → 否
   b. Case 2: isUserAHost(2, booking)? → 否
   c. Case 3: booking.eventType.teamId? 
      → 子事件类型的 teamId = null → 跳过
   d. Case 4: booking.eventType.parent.teamId?
      → 父事件类型的 teamId = 100 → 进入分支
   e. 调用 isAdminOfTeamOrParentOrg({ userId: 2, teamId: 100 })
   f. 查询找到 membership(role=ADMIN, accepted=false)
   g. 返回 true！
4. 用户 B（未接受邀请）成功获得访问权限
```

---

## 6. 可利用前提

### 6.1 必要条件

| 条件 | 描述 | 是否常见 |
|------|------|---------|
| 存在 Managed Event | 团队创建了 `schedulingType = MANAGED` 的事件类型 | 中等 |
| 存在子事件类型 | 父事件类型已分配给团队成员 | 中等 |
| 子事件类型有预约 | 用户使用子事件类型创建了预约 | 中等 |
| 存在待接受的管理员邀请 | 团队向某人发出 ADMIN/OWNER 邀请但未接受 | 低 |

### 6.2 攻击向量

**攻击场景 1：内部人员利用**
- 团队管理员离开团队但未清理邀请
- 攻击者（前团队成员）仍有 pending 邀请记录
- 可访问所有 Managed Event 的预约数据

**攻击场景 2：邀请劫持**
- 攻击者获取了发送给他人的邀请链接
- 使用邀请创建账户但**不点击接受**
- 通过 pending 状态获得管理员权限

---

## 7. 风险定级

### 7.1 CVSS 评分

| 维度 | 评分 | 说明 |
|------|------|------|
| 攻击向量 (AV) | **N (Network)** | 可通过网络远程利用 |
| 攻击复杂度 (AC) | **L (Low)** | 只需构造预约访问请求 |
| 权限要求 (PR) | **L (Low)** | 需要普通用户账户和待接受邀请 |
| 用户交互 (UI) | **N (None)** | 无需用户交互 |
| 范围 (S) | **C (Changed)** | 影响其他用户的预约数据 |
| 机密性 (C) | **H (High)** | 可读取敏感预约信息（时间、参与者、描述） |
| 完整性 (I) | **N (None)** | 仅读取权限，无修改能力 |
| 可用性 (A) | **N (None)** | 不影响系统可用性 |

**CVSS v3.1 评分：** 7.7 (High)

### 7.2 最终风险定级

| 维度 | 评估 |
|------|------|
| 严重程度 | **高危 (High)** |
| 影响范围 | 所有使用 Managed Event 的团队预约 |
| 利用难度 | 中低 - 需要获得待接受的管理员邀请 |
| 隐蔽性 | 高 - 代码逻辑看起来正常，容易被忽视 |
| 数据影响 | 高 - 敏感预约数据泄露 |
| 业务影响 | 中 - 影响团队预约的隐私保护 |

### 7.3 为什么不是 Critical

1. **利用条件相对受限**：需要存在待接受的 ADMIN/OWNER 邀请
2. **只有读取权限**：无法修改或删除预约
3. **影响范围有限**：仅影响 Managed Event 类型的预约

---

## 8. 代码证据链

### 8.1 完整调用链

```
BookingAccessService.doesUserIdHaveAccessToBooking() (BookingAccessService.ts:56)
  ├── Case 1: userId === booking.userId? → 否
  ├── Case 2: isUserAHost()? → 否
  ├── Case 3: booking.eventType?.teamId? 
  │   └── 子事件类型 teamId = null → 跳过
  └── Case 4: booking.eventType?.parent?.teamId? (第 97 行)
      └── 父事件类型 teamId = 100 → 进入分支
          └── UserRepository.isAdminOfTeamOrParentOrg() (UserRepository.ts:1014)
              └── ⚠️ 缺少 accepted 检查 → 返回 true
```

### 8.2 关键文件位置

| 文件 | 行号 | 说明 |
|------|------|------|
| `packages/features/bookings/services/BookingAccessService.ts` | 97-103 | Managed Event 分支 |
| `packages/features/users/repositories/UserRepository.ts` | 1014-1038 | 漏洞方法 `isAdminOfTeamOrParentOrg` |
| `packages/features/bookings/repositories/BookingRepository.ts` | 481-575 | 查询字段定义 |
| `packages/lib/getTeamIdFromEventType.ts` | 1-28 | 验证子事件类型无 teamId |
| `packages/features/bookings/lib/handleNewBooking/global-booking-limits.test.ts` | 106-126 | 测试用例验证数据结构 |

---

## 9. 修复建议

### 9.1 方案一：修复 isAdminOfTeamOrParentOrg（推荐）

**文件：** `packages/features/users/repositories/UserRepository.ts:1014-1038`

```typescript
async isAdminOfTeamOrParentOrg({ userId, teamId }: { userId: number; teamId: number }) {
  const membershipQuery = {
    members: {
      some: {
        userId,
        role: { in: [MembershipRole.ADMIN, MembershipRole.OWNER] },
        accepted: true,  // ✅ 添加 accepted 检查
      },
    },
  };
  // ... 其余代码不变
}
```

### 9.2 方案二：统一使用 PermissionCheckService

将 Case 4 改为使用 `PermissionCheckService`，与 Case 3 保持一致：

```typescript
// Case 4: Managed Event 分支
if (booking.eventType?.parent?.teamId) {
  const hasAccess = await this.permissionCheckService.checkPermission({
    userId,
    teamId: booking.eventType.parent.teamId,
    permission: "booking.readTeamBookings",
    fallbackRoles: [MembershipRole.OWNER, MembershipRole.ADMIN],
  });
  return hasAccess;
}
```

---

## 10. 结论

### 10.1 核心发现

1. **Managed Event 分支确实可达**：子事件类型的 `teamId` 为 null，`parent.teamId` 非 null，Case 3 被跳过，Case 4 被触发。

2. **存在高危漏洞**：`isAdminOfTeamOrParentOrg()` 缺少 `accepted` 状态检查，未接受邀请的 ADMIN/OWNER 也能获得访问权限。

3. **数据模型确认**：通过测试用例和 `getTeamIdFromEventType` 工具函数验证了父子事件类型的 teamId 关系。

### 10.2 最终结论

| 项目 | 结论 |
|------|------|
| 分支可达性 | ✅ 可实际触发 |
| 漏洞真实性 | ✅ 确认的高危漏洞 |
| 风险等级 | **高危 (High)** |
| 修复优先级 | **高 - 应在近期修复** |

### 10.3 测试建议

添加以下测试用例：

1. **未接受邀请的 ADMIN 不应访问 Managed Event 预约**
2. **已接受邀请的 ADMIN 应能访问**
3. **父组织中未接受邀请的 ADMIN 不应访问子团队的 Managed Event**
4. **子事件类型预约的权限检查应与父事件类型保持一致**
