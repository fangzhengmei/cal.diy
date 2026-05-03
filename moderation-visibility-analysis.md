# 内容审核、屏蔽与可见性控制系统分析报告

## 1. 系统架构概览

Cal.diy 的内容审核、屏蔽与可见性控制系统采用**多层级、联邦式**架构设计，在多个层面协同工作以确保平台安全和用户体验。

### 1.1 核心组件层次

```
┌─────────────────────────────────────────────────────────────┐
│                      应用层 (Application Layer)               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │预订流程      │  │时隙服务      │  │用户管理界面       │ │
│  │loadAndValidate│ │slots/util.ts │  │Admin UI          │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      操作层 (Operations Layer)                │
│  ┌──────────────────┐  ┌──────────────────────────────────┐ │
│  │filterBlockedUsers│  │filterBlockedHosts                 │ │
│  │checkIfBookerEmail│  │getBlockedUsersMap                 │ │
│  │IsBlocked         │  │checkWatchlistBlocking             │ │
│  └──────────────────┘  └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      服务层 (Service Layer)                   │
│  ┌──────────────────┐  ┌──────────────────────────────────┐ │
│  │GlobalBlocking    │  │OrganizationBlockingService        │ │
│  │Service           │  │(组织级屏蔽)                        │ │
│  │(全局屏蔽)         │  │                                  │ │
│  ├──────────────────┤  ├──────────────────────────────────┤ │
│  │BotDetection      │  │WatchlistService (CRUD)            │ │
│  │Service           │  │                                  │ │
│  │(机器人检测)       │  │                                  │ │
│  └──────────────────┘  └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      数据层 (Data Layer)                      │
│  ┌──────────────────┐  ┌──────────────────────────────────┐ │
│  │Watchlist 表      │  │User 表 (locked 字段)              │ │
│  │(isGlobal, orgId) │  │                                  │ │
│  ├──────────────────┤  ├──────────────────────────────────┤ │
│  │WatchlistAudit    │  │BookingReport 表                   │ │
│  │(审计追踪)         │  │(举报记录)                          │ │
│  └──────────────────┘  └──────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 屏蔽策略叠加逻辑

### 2.1 策略层次与优先级

系统采用**多层递进**的屏蔽策略，各层之间有明确的优先级关系：

```
┌────────────────────────────────────────────────────────────────┐
│                    策略叠加逻辑 (Policy Stacking)                │
├────────────────────────────────────────────────────────────────┤
│  优先级 1: 用户锁定 (User Locked)                               │
│  ───────────────────────────────────────────────────────────── │
│  - 检查: user.locked === true                                   │
│  - 来源: User 表字段                                             │
│  - 特性: 立即阻断，无需数据库查询                                 │
│  - 优先级: 最高 (短路逻辑)                                        │
├────────────────────────────────────────────────────────────────┤
│  优先级 2: 全局屏蔽列表 (Global Watchlist)                      │
│  ───────────────────────────────────────────────────────────── │
│  - 检查: Watchlist.isGlobal === true                            │
│  - 支持类型: EMAIL, DOMAIN (含通配符 *.domain.com)              │
│  - 适用范围: 整个平台所有组织                                      │
├────────────────────────────────────────────────────────────────┤
│  优先级 3: 组织级屏蔽列表 (Organization Watchlist)              │
│  ───────────────────────────────────────────────────────────── │
│  - 检查: Watchlist.organizationId === currentOrgId              │
│  - 支持类型: EMAIL, DOMAIN (含通配符)                            │
│  - 适用范围: 仅当前组织                                           │
├────────────────────────────────────────────────────────────────┤
│  优先级 4: 环境变量黑名单 (Env Blacklist)                       │
│  ───────────────────────────────────────────────────────────── │
│  - 检查: process.env.BLACKLISTED_GUEST_EMAILS                  │
│  - 适用场景: 紧急屏蔽、运维临时策略                                 │
├────────────────────────────────────────────────────────────────┤
│  优先级 5: 用户级邮箱验证要求 (User Setting)                     │
│  ───────────────────────────────────────────────────────────── │
│  - 检查: user.requiresBookerEmailVerification                   │
│  - 逻辑: 非登录用户预订需验证邮箱                                  │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 核心实现代码分析

**策略叠加的核心逻辑**位于 `check-user-blocking.ts`:

```typescript
// packages/features/watchlist/operations/check-user-blocking.ts:86-140
export async function getBlockedUsersMap<T extends BlockableUser>(
  users: T[],
  organizationId?: number | null
): Promise<CheckUserBlockingResult> {
  
  // 第一遍: 检查锁定用户 (内存操作，无DB查询)
  const nonLockedUsers: (typeof validUsers)[number][] = [];
  for (const user of validUsers) {
    if (user.locked) {
      blockingMap.set(normalizedEmail, { 
        isBlocked: true, 
        reason: "locked" 
      });
      lockedCount++;
    } else {
      nonLockedUsers.push(user);
    }
  }

  // 第二遍: 检查 watchlist (仅非锁定用户)
  if (nonLockedUsers.length > 0) {
    const watchlistResults = await checkWatchlistBlocking(
      emails, 
      organizationId
    );
    // ... 处理 watchlist 结果
  }
}
```

**全局与组织级屏蔽的 OR 逻辑**:

```typescript
// packages/features/watchlist/operations/check-user-blocking.ts:33-72
export async function checkWatchlistBlocking(
  emails: string[],
  organizationId?: number | null
): Promise<Map<string, boolean>> {
  
  // 1. 查询全局屏蔽
  const globalBlockedMap = await watchlist.globalBlocking.areBlocked(emails);
  
  // 2. 如提供 organizationId，查询组织级屏蔽
  let orgBlockedMap = null;
  if (organizationId) {
    orgBlockedMap = await watchlist.orgBlocking.areBlocked(emails, organizationId);
  }

  // 3. 结果合并: OR 逻辑 - 任一屏蔽即视为屏蔽
  for (const email of emails) {
    const globalResult = globalBlockedMap.get(normalizedEmail);
    const orgResult = orgBlockedMap?.get(normalizedEmail);
    
    result.set(normalizedEmail, 
      !!(globalResult?.isBlocked || orgResult?.isBlocked)  // OR 逻辑
    );
  }
}
```

### 2.3 域名通配符匹配机制

系统支持灵活的域名屏蔽策略，包括精确匹配和通配符匹配：

```typescript
// packages/features/watchlist/lib/utils/normalization.ts:150-164
export function domainMatchesWatchlistEntry(
  emailDomain: string, 
  watchlistValue: string
): boolean {
  // 通配符匹配: *.cal.com 匹配 app.cal.com, sub.app.cal.com 等
  if (normalizedWatchlistValue.startsWith("*.")) {
    const baseDomain = normalizedWatchlistValue.slice(2);
    return normalizedEmailDomain.endsWith(`.${baseDomain}`);
  }
  
  // 精确匹配: cal.com 仅匹配 cal.com
  return normalizedEmailDomain === normalizedWatchlistValue;
}

// 通配符模式生成: 为 domain 生成可能的上级通配符
export function getWildcardPatternsForDomain(domain: string): string[] {
  // app.cal.com → ["*.cal.com"]
  // cal.com → [] (父域只是 com，不生成)
}
```

---

## 3. 联邦边界设计

### 3.1 数据模型中的联邦标识

Watchlist 表通过两个关键字段实现联邦边界：

```prisma
// packages/prisma/schema.prisma:2079-2096
model Watchlist {
  id             String          @id @default(uuid())
  type           WatchlistType   // EMAIL | DOMAIN | USERNAME
  value          String
  
  // 联邦边界关键字段
  isGlobal       Boolean         @default(false)
  organizationId Int?
  
  action         WatchlistAction // REPORT | BLOCK | ALERT
  source         WatchlistSource // MANUAL | FREE_DOMAIN_POLICY | SIGNUP
  
  @@unique([type, value, organizationId])  // 联合唯一约束
  @@index([isGlobal, action, organizationId, type, value])
}
```

### 3.2 联邦边界规则

| 边界类型 | isGlobal | organizationId | 适用范围 | 查询条件 |
|---------|----------|----------------|---------|---------|
| **全局规则** | `true` | `null` | 所有组织、所有用户 | `isGlobal = true` |
| **组织规则** | `false` | `[orgId]` | 仅指定组织内部 | `organizationId = ?` |
| **用户规则** | `false` | `null` | (预留，当前未启用) | - |

### 3.3 服务层的职责分离

系统通过两个独立的服务实现联邦边界：

```typescript
// packages/features/watchlist/lib/service/GlobalBlockingService.ts
export class GlobalBlockingService implements IBlockingService {
  // 仅查询 isGlobal = true 的记录
  async areBlocked(emails: string[]): Promise<BulkBlockingResult> {
    const blockingEntries = await this.deps.globalRepo
      .findBlockingEntriesForEmailsAndDomains({
        emails: normalizedEmails,
        domains: [...],
        // 隐式: 仅查询全局条目
      });
  }
}

// packages/features/watchlist/lib/service/OrganizationBlockingService.ts
export class OrganizationBlockingService implements IBlockingService {
  // 必须传入 organizationId，仅查询该组织的记录
  async areBlocked(
    emails: string[], 
    organizationId: number  // 必填参数
  ): Promise<BulkBlockingResult> {
    const blockingEntries = await this.deps.orgRepo
      .findBlockingEntriesForEmailsAndDomains({
        emails: normalizedEmails,
        domains: [...],
        organizationId,  // 显式组织过滤
      });
  }
}
```

### 3.4 前端/API 层的边界感知

在预订流程中，系统会自动识别当前上下文的组织边界：

```typescript
// packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:132-141
let organizationId: number | null = 
  eventType.parent?.team?.parentId ?? eventType.team?.parentId ?? null;

// 个人事件的回退策略: 使用用户的第一个组织
if (!organizationId && eventType.userId) {
  organizationId = await ProfileRepository
    .findFirstOrganizationIdForUser({ userId: eventType.userId });
}

// 传入组织 ID 进行屏蔽检查
const { eligibleUsers, blockedCount } = await filterBlockedUsers(
  users, 
  organizationId, 
  sentrySpan
);
```

---

## 4. 一致性保障机制

### 4.1 数据一致性

#### 4.1.1 邮箱规范化

所有屏蔽检查都经过统一的规范化处理，确保大小写、空格不影响匹配：

```typescript
// packages/features/watchlist/lib/utils/normalization.ts:20-27
export function normalizeEmail(email: string): string {
  const normalized = email.trim().toLowerCase();
  
  if (!emailRegex.test(normalized)) {
    throw new Error(`Invalid email format: ${email}`);
  }
  return normalized;
}

// 使用示例
const normalizedEmail = email.trim().toLowerCase();
const isBlocked = blockingMap.get(normalizedEmail)?.isBlocked ?? false;
```

#### 4.1.2 数据库唯一约束

```prisma
@@unique([type, value, organizationId])
```

- 同一组织内，相同 type+value 只能有一条记录
- 全局条目 (`organizationId = null`) 也受此约束保护

### 4.2 运行时一致性

#### 4.2.1 批处理查询 (消除 N+1)

```typescript
// packages/features/watchlist/operations/check-user-blocking.ts:40
// 单次数据库查询处理 N 个用户
const { blockingMap, blockedCount } = await getBlockedUsersMap(
  users, 
  organizationId
);
```

**优势**:
- 所有用户使用相同的 `blockingMap` 快照
- 避免了查询过程中数据变化导致的不一致

#### 4.2.2 失败开放策略 (Fail-Open)

```typescript
// packages/features/watchlist/operations/check-user-blocking.ts:58-71
try {
  // watchlist 检查逻辑...
} catch (error) {
  // 失败开放: 服务不可用时允许预订继续
  log.error("Watchlist check failed, allowing users through (fail-open)", {
    error: errorMessage,
    emailCount: emails.length,
    organizationId,
  });
  // 返回所有用户为非屏蔽状态
  return new Map(emails.map((e) => [e.trim().toLowerCase(), false]));
}
```

**设计权衡**:
- **可用性优先**: 防止 watchlist 服务故障导致整个预订系统瘫痪
- **安全降级**: 损失部分屏蔽能力，但保持核心功能可用
- **可观测**: 错误会被详细记录，便于事后追溯

### 4.3 审计与可追溯性

#### 4.3.1 审计日志表

```prisma
// packages/prisma/schema.prisma:2098-2113
model WatchlistAudit {
  id              String        @id @default(uuid(7))
  type            WatchlistType
  value           String
  action          WatchlistAction  // REPORT | BLOCK | ALERT
  changedAt       DateTime         @default(now())
  changedByUserId Int?             // 操作者
  watchlistId     String?          // 关联的 watchlist 条目
}
```

#### 4.3.2 事件审计

```prisma
model WatchlistEventAudit {
  id          String          @id @default(uuid(7))
  watchlistId String
  eventTypeId Int
  actionTaken WatchlistAction
  timestamp   DateTime        @default(now())
}
```

### 4.4 邮箱验证层一致性

```typescript
// packages/features/bookings/lib/handleNewBooking/checkIfBookerEmailIsBlocked.ts:7-93
export const checkIfBookerEmailIsBlocked = async ({
  bookerEmail,
  loggedInUserId,
  verificationCode,
  isReschedule,
}) => {
  // 1. 环境变量黑名单检查
  const blacklistedByEnv = blacklistedGuestEmails.find(
    (guestEmail) => guestEmail.toLowerCase() === baseEmail.toLowerCase()
  );
  
  // 2. 用户设置: 要求预订者邮箱验证
  const blockedByUserSetting = user?.requiresBookerEmailVerification ?? false;
  
  // 3. 综合判断
  const shouldBlock = !!blacklistedByEnv || (blockedByUserSetting && !isReschedule);
  
  // 4. 验证流程: 已登录用户或有验证码可绕过
  if (user.id !== loggedInUserId) {
    if (verificationCode) {
      // 验证验证码...
    }
    throw new ErrorWithCode(ErrorCode.BookerEmailRequiresLogin, "...");
  }
};
```

---

## 5. 关键调用点分析

### 5.1 用户屏蔽过滤 (预订流程)

```
调用链: 
loadAndValidateUsers → filterBlockedUsers → getBlockedUsersMap
         ↓
    checkWatchlistBlocking
         ↓
    ┌────┴────┐
    ↓         ↓
globalBlocking  orgBlocking
.areBlocked()  .areBlocked()
```

**核心逻辑位置**:
- `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts:141`
- `packages/features/watchlist/operations/filter-blocked-users.controller.ts:29-56`

### 5.2 主持人屏蔽过滤 (时隙服务)

```typescript
// packages/features/watchlist/operations/filter-blocked-hosts.controller.ts:28-49
export async function filterBlockedHosts<T extends HostWithEmail>(
  hosts: T[],
  organizationId?: number | null
): Promise<FilterBlockedHostsResult<T>> {
  // 复用相同的 getBlockedUsersMap 逻辑
  const usersToCheck = hosts.map((h) => h.user);
  const { blockingMap, blockedCount } = await getBlockedUsersMap(
    usersToCheck, 
    organizationId
  );
  
  const eligibleHosts = hosts.filter((host) => 
    !isUserBlocked(host.user.email, blockingMap)
  );
  
  return { eligibleHosts, blockedCount };
}
```

**调用位置**: `packages/trpc/server/routers/viewer/slots/util.ts`

### 5.3 机器人检测层

```typescript
// packages/features/bot-detection/BotDetectionService.ts:28-89
export class BotDetectionService {
  async checkBotDetection(config: BotDetectionConfig): Promise<void> {
    // 1. 功能开关检查
    if (!this.instanceHasBotIdEnabled()) return;
    if (!eventType?.teamId) return;  // 仅团队事件
    
    // 2. 特性标志检查
    const isBotIDEnabled = await this.featuresRepository
      .checkIfTeamHasFeature(eventType.teamId, "booker-botid");
    
    // 3. 执行检测
    const verification = await checkBotId({
      advancedOptions: { headers },
    });
    
    // 4. 阻断逻辑
    if (verification.isBot) {
      log.warn("Bot detected - blocking request", verificationDetails);
      throw new HttpError({ statusCode: 403, message: "Access denied" });
    }
  }
}
```

---

## 6. 系统边界与数据流

### 6.1 完整数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        预订请求 (Booking Request)                      │
│  POST /api/bookings 或 内部 tRPC 调用                                  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  1. 机器人检测 (BotDetectionService)                                  │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ 检查项:                                                        │  │
│     │ - NEXT_PUBLIC_VERCEL_USE_BOTID_IN_BOOKER === "1"           │  │
│     │ - 事件属于团队 (teamId 存在)                                   │  │
│     │ - 团队启用了 "booker-botid" 特性                               │  │
│     │ - BotID SDK 验证结果 (isBot === true)                        │  │
│     └─────────────────────────────────────────────────────────────┘  │
│     阻断方式: 403 Access Denied                                        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (通过)
┌─────────────────────────────────────────────────────────────────────┐
│  2. 预订者邮箱验证 (checkIfBookerEmailIsBlocked)                     │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ 检查项:                                                        │  │
│     │ - BLACKLISTED_GUEST_EMAILS 环境变量                           │  │
│     │ - 用户设置: requiresBookerEmailVerification                   │  │
│     │ - 登录状态 (loggedInUserId === user.id)                       │  │
│     │ - 验证码验证 (verificationCode)                                │  │
│     └─────────────────────────────────────────────────────────────┘  │
│     阻断方式: ErrorWithCode (BookerEmailBlocked/RequiresLogin)       │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (通过)
┌─────────────────────────────────────────────────────────────────────┐
│  3. 主持人/用户屏蔽过滤 (loadAndValidateUsers)                        │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ 3.1 锁定用户检查 (user.locked === true)                       │  │
│     │     - 内存操作，无 DB 查询                                      │  │
│     │     - 短路逻辑: 锁定用户不再检查 watchlist                     │  │
│     │                                                               │  │
│     │ 3.2 全局 Watchlist 检查                                       │  │
│     │     - Watchlist.isGlobal === true                            │  │
│     │     - 支持 EMAIL/DOMAIN (含通配符)                            │  │
│     │                                                               │  │
│     │ 3.3 组织级 Watchlist 检查                                     │  │
│     │     - Watchlist.organizationId === currentOrgId              │  │
│     │     - 仅在提供 organizationId 时执行                          │  │
│     │                                                               │  │
│     │ 3.4 结果合并                                                   │  │
│     │     - OR 逻辑: 任一屏蔽即视为被屏蔽                            │  │
│     │     - 过滤掉被屏蔽的用户/主持人                                 │  │
│     └─────────────────────────────────────────────────────────────┘  │
│     阻断方式:                                                          │
│     - 部分屏蔽: 过滤后继续 (graceful degradation)                     │
│     - 全部屏蔽: 404 eventTypeUser.notFound                            │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (通过)
┌─────────────────────────────────────────────────────────────────────┐
│  4. 时隙可用性检查 (Slots Service)                                    │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ - 调用 filterBlockedHosts 过滤被屏蔽的主持人                   │  │
│     │ - 只返回可用主持人的时隙                                         │  │
│     └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 策略优先级总结表

| 层级 | 检查点 | 优先级 | 短路行为 | 阻断方式 |
|-----|-------|-------|---------|---------|
| **L1** | BotDetection | 1 | 是 (throw) | 403 Access Denied |
| **L2** | 邮箱黑名单 (env) | 2 | 是 | ErrorWithCode |
| **L3** | 用户邮箱验证要求 | 3 | 是 (可通过验证绕过) | ErrorWithCode |
| **L4** | 用户锁定状态 | 4 | 是 (不再查 watchlist) | 过滤掉，不报错 |
| **L5** | 全局 Watchlist | 5 | 否 | 过滤掉，不报错 |
| **L6** | 组织级 Watchlist | 6 | 否 | 过滤掉，不报错 |

---

## 7. 设计亮点与潜在风险

### 7.1 设计亮点

#### 7.1.1 分层架构清晰
- 职责分离明确：BotDetection、邮箱验证、用户屏蔽各司其职
- 统一的规范化层确保数据一致性
- 可复用的服务层（GlobalBlocking/OrganizationBlocking）

#### 7.1.2 性能优化
- 批处理查询消除 N+1 问题
- 锁定用户检查短路（无需 DB 查询）
- 通配符域名查询优化（单次查询匹配多种模式）

#### 7.1.3 高可用设计
- 失败开放策略（Fail-Open）防止级联故障
- 优雅降级：部分用户被屏蔽时系统继续运行
- 完善的日志和审计追踪

#### 7.1.4 联邦边界设计
- 数据层面：isGlobal + organizationId 双字段控制
- 服务层面：GlobalBlockingService 与 OrganizationBlockingService 分离
- 唯一约束：防止同一组织内重复条目

### 7.2 潜在风险与注意事项

#### 7.2.1 安全风险
| 风险点 | 描述 | 缓解措施 |
|-------|------|---------|
| **Fail-Open 安全 trade-off** | Watchlist 服务故障时屏蔽失效 | 1. 详细错误日志<br>2. 考虑关键场景强制检查 |
| **通配符范围过大** | `*.co.uk` 可能匹配过多域名 | 查询时验证父域至少包含一个 `.`<br>`getWildcardPatternsForDomain` 已实现 |
| **邮箱别名绕过** | `user+tag@gmail.com` 与 `user@gmail.com` 视为不同 | 当前未处理 + 标签，可能需要增强规范化 |

#### 7.2.2 一致性风险
| 风险点 | 描述 | 缓解措施 |
|-------|------|---------|
| **批处理快照一致性** | 批处理期间数据可能变化 | 单次查询获取快照，接受短暂不一致 |
| **全局与组织规则冲突** | 同一邮箱在全局允许但组织屏蔽，或反之 | OR 逻辑：任一屏蔽即屏蔽（保守策略） |
| **缓存失效** | 若引入缓存可能导致 stale 数据 | 当前无缓存，每次实时查询 |

#### 7.2.3 运维风险
| 风险点 | 描述 | 缓解措施 |
|-------|------|---------|
| **环境变量黑名单** | 紧急添加但可能遗忘移除 | 1. 仅用于紧急情况<br>2. 应同步到 Watchlist 表 |
| **数据库索引** | Watchlist 查询需要高效索引 | 已有 `@@index([isGlobal, action, organizationId, type, value])` |

---

## 8. 关键代码位置索引

### 8.1 核心屏蔽逻辑
| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| 屏蔽检查主入口 | `packages/features/watchlist/operations/check-user-blocking.ts` | `getBlockedUsersMap`, `checkWatchlistBlocking` |
| 用户过滤控制器 | `packages/features/watchlist/operations/filter-blocked-users.controller.ts` | `filterBlockedUsers` |
| 主持人过滤控制器 | `packages/features/watchlist/operations/filter-blocked-hosts.controller.ts` | `filterBlockedHosts` |
| 全局屏蔽服务 | `packages/features/watchlist/lib/service/GlobalBlockingService.ts` | `GlobalBlockingService` |
| 组织屏蔽服务 | `packages/features/watchlist/lib/service/OrganizationBlockingService.ts` | `OrganizationBlockingService` |

### 8.2 预订流程集成
| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| 预订用户验证 | `packages/features/bookings/lib/handleNewBooking/loadAndValidateUsers.ts` | `loadAndValidateUsers` |
| 预订者邮箱检查 | `packages/features/bookings/lib/handleNewBooking/checkIfBookerEmailIsBlocked.ts` | `checkIfBookerEmailIsBlocked` |
| 机器人检测 | `packages/features/bot-detection/BotDetectionService.ts` | `BotDetectionService.checkBotDetection` |

### 8.3 工具与规范化
| 功能 | 文件路径 | 关键函数/类 |
|-----|---------|------------|
| 邮箱/域名规范化 | `packages/features/watchlist/lib/utils/normalization.ts` | `normalizeEmail`, `normalizeDomain`, `domainMatchesWatchlistEntry` |
| 数据模型 | `packages/prisma/schema.prisma` | `Watchlist`, `WatchlistAudit`, `WatchlistEventAudit` |

---

## 9. 测试覆盖分析

### 9.1 现有测试文件
- `check-user-blocking.test.ts` - 核心屏蔽逻辑单元测试
- `filter-blocked-users.controller.test.ts` - 用户过滤控制器测试
- `filter-blocked-hosts.controller.test.ts` - 主持人过滤控制器测试
- `GlobalBlockingService.test.ts` - 全局屏蔽服务测试
- `OrganizationBlockingService.test.ts` - 组织屏蔽服务测试
- `normalization.test.ts` - 规范化工具测试

### 9.2 测试覆盖的关键场景
1. **空输入处理** - 空数组、空邮箱
2. **锁定用户** - locked=true 用户的短路行为
3. **Watchlist 组合** - 全局 + 组织级的 OR 逻辑
4. **邮箱规范化** - 大小写、空格处理
5. **域名通配符** - `*.domain.com` 的匹配行为
6. **失败开放** - watchlist 服务故障时的行为

---

## 10. 总结与建议

### 10.1 架构总结

Cal.diy 的内容审核、屏蔽与可见性控制系统采用了**"多层递进、联邦自治、一致性优先"**的设计理念：

1. **多层递进**: 从 BotDetection → 邮箱验证 → 用户锁定 → Watchlist，每层有明确的职责和阻断方式
2. **联邦自治**: 通过 `isGlobal` 和 `organizationId` 实现平台规则与组织规则的分离与协同
3. **一致性优先**: 统一的规范化层、批处理快照、数据库约束确保多层面决策的一致性

### 10.2 关键设计原则

| 原则 | 实现方式 |
|-----|---------|
| **可用性 > 安全性** (受控) | Fail-Open 策略，防止单一组件故障拖垮整体 |
| **性能优先** | 批处理消除 N+1，短路优化减少查询 |
| **数据一致性** | 统一规范化，数据库唯一约束 |
| **可观测性** | 完善的日志，审计追踪表 |
| **联邦边界清晰** | 双字段控制，双服务分离 |

### 10.3 后续优化建议

1. **邮箱别名处理**: 考虑规范化时去除 `+tag` 后缀，防止用户通过别名绕过屏蔽
2. **缓存层引入**: 对于高频查询的 Watchlist 条目，可考虑引入带 TTL 的缓存
3. **规则优先级配置**: 当前全局与组织是 OR 逻辑，可考虑支持可配置的优先级策略
4. **批量操作审计**: 现有的 WatchlistAudit 记录单体操作，可增强批量操作的审计

---

*报告生成时间: 2026-05-03*
*分析范围: packages/features/watchlist, packages/features/bookings, packages/features/bot-detection, packages/prisma*
