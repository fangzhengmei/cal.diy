# Cal.diy OAuth Client 权限范围与状态门禁最终分析报告

**文档版本**：v1.0 最终评审版  
**生成日期**：2026-05-04  
**分析范围**：Cal.diy OAuth 2.0 体系、Platform API 体系

---

## 执行摘要

### 关键修正结论

**之前报告中"无审批流程"的结论不准确**。代码库中实际存在**两种独立的 OAuth 客户端模型**，它们的权限模型和状态门禁机制完全不同：

| 模型 | 状态门禁 | 权限模型 | 用途 |
|------|---------|---------|------|
| **`OAuthClient`** | ✅ 完整支持 (PENDING/APPROVED/REJECTED) | `AccessScope` 枚举 (2种读权限) | 传统 OAuth 2.0 第三方应用集成 |
| **`PlatformOAuthClient`** | ❌ 无状态门禁 | `permissions` 位掩码 (10种读写权限) | Platform API 托管用户管理 |

### 核心发现

1. **状态门禁只存在于 `OAuthClient`**：在授权码签发、换 Token、刷新 Token 三个阶段都有状态检查
2. **权限模型两套并行**：
   - `OAuthClient` 使用 `AccessScope` 枚举 (READ_BOOKING, READ_PROFILE)
   - `PlatformOAuthClient` 使用位掩码 (10种权限)
3. **状态放行规则**：
   - **APPROVED**：所有用户都允许
   - **PENDING**：只有客户端所有者允许（开发者测试绕过）
   - **REJECTED**：所有用户都拒绝

---

## 第一部分：两种 OAuth 客户端模型对比分析

### 1.1 数据库模型对比

#### 模型 1：`OAuthClient` (传统 OAuth 2.0)

位置：`packages/prisma/schema.prisma:1461-1479`

```prisma
enum OAuthClientStatus {
  PENDING  @map("pending")
  APPROVED @map("approved")
  REJECTED @map("rejected")
}

model OAuthClient {
  clientId        String            @id @unique
  redirectUri     String
  clientSecret    String?
  clientType      OAuthClientType   @default(CONFIDENTIAL)
  name            String
  purpose         String?
  logo            String?
  websiteUrl      String?
  rejectionReason String?           // 拒绝原因
  isTrusted       Boolean           @default(false)
  accessCodes     AccessCode[]
  status          OAuthClientStatus @default(APPROVED)  // ⚠️ 状态字段
  userId          Int?              // 客户端所有者
  user            User?             @relation(fields: [userId], references: [id], onDelete: SetNull)
  createdAt       DateTime          @default(now())

  @@index([userId])
}
```

#### 模型 2：`PlatformOAuthClient` (Platform API)

位置：`packages/prisma/schema.prisma:1716-1741`

```prisma
model PlatformOAuthClient {
  id             String   @id @default(cuid())
  name           String
  secret         String
  permissions    Int       // ⚠️ 位掩码权限，无状态字段
  users          User[]
  logo           String?
  redirectUris   String[]
  organizationId Int       // ⚠️ 必须绑定组织
  organization   Team     @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  teams          Team[]   @relation("CreatedByOAuthClient")

  // Token 关联
  accessTokens        AccessToken[]
  refreshToken        RefreshToken[]
  authorizationTokens PlatformAuthorizationToken[]
  webhook             Webhook[]

  // 功能开关
  bookingRedirectUri           String?
  bookingCancelRedirectUri     String?
  bookingRescheduleRedirectUri String?
  areEmailsEnabled             Boolean @default(false)
  areDefaultEventTypesEnabled  Boolean @default(true)
  areCalendarEventsEnabled     Boolean @default(true)

  createdAt DateTime @default(now())
}
```

### 1.2 核心差异对照表

| 维度 | `OAuthClient` (传统) | `PlatformOAuthClient` (平台) |
|------|---------------------|------------------------------|
| **状态门禁** | ✅ 完整支持 | ❌ 无状态字段 |
| **默认状态** | `APPROVED` | 无状态，直接可用 |
| **权限模型** | `AccessScope` 枚举 | `permissions` 位掩码 |
| **权限数量** | 2 种（仅读） | 10 种（读写各5类资源） |
| **组织绑定** | ❌ 可选 (userId) | ✅ 必须 (organizationId) |
| **Token 类型** | `AccessCode` | `AccessToken` + `RefreshToken` + `PlatformAuthorizationToken` |
| **密钥存储** | 哈希存储 (`generateSecret`) | JWT 明文签名 |
| **适用场景** | 第三方 OAuth 2.0 应用 | Platform API 托管用户 |

---

## 第二部分：状态门禁机制详细分析

### 2.1 状态枚举定义

位置：`packages/prisma/schema.prisma:1455-1459`

```prisma
enum OAuthClientStatus {
  PENDING  @map("pending")
  APPROVED @map("approved")
  REJECTED @map("rejected")
}
```

### 2.2 状态门禁核心实现

位置：`packages/features/oauth/services/OAuthService.ts:167-207`

```typescript
/**
 * 确保客户端已批准，对 PENDING 客户端有特殊例外：
 * - PENDING 客户端如果由请求用户拥有，则允许（用于开发者测试）
 * - REJECTED 客户端无论所有权如何，始终阻止
 */
private ensureClientIsApprovedOrOwnedPending(
  client: { status: OAuthClientStatus; userId?: number | null },
  userId?: number | null
): void {
  // 情况 1：已批准 - 直接通过
  if (client.status === OAuthClientStatus.APPROVED) {
    return;
  }

  // 情况 2：已拒绝 - 始终拒绝
  if (client.status === OAuthClientStatus.REJECTED) {
    throw new ErrorWithCode(ErrorCode.Unauthorized, "unauthorized_client", {
      reason: "client_rejected",
    });
  }

  // 情况 3：待审批 - 仅允许客户端所有者通过（开发者测试绕过）
  if (
    client.status === OAuthClientStatus.PENDING &&
    userId !== undefined &&
    userId !== null &&
    client.userId === userId
  ) {
    return;
  }

  // 其他情况：拒绝
  throw new ErrorWithCode(ErrorCode.Unauthorized, "unauthorized_client", {
    reason: "client_not_approved",
  });
}
```

### 2.3 状态门禁行为矩阵

| 客户端状态 | 请求者身份 | 是否允许 | 错误原因 |
|-----------|-----------|---------|---------|
| **APPROVED** | 任意用户 | ✅ 允许 | - |
| **PENDING** | 客户端所有者 (userId 匹配) | ✅ 允许 (开发者测试绕过) | - |
| **PENDING** | 非所有者用户 | ❌ 拒绝 | `client_not_approved` |
| **REJECTED** | 任意用户 (包括所有者) | ❌ 拒绝 | `client_rejected` |

### 2.4 状态门禁作用阶段

状态门禁在以下四个关键阶段生效：

#### 阶段 1：授权码签发

位置：`packages/features/oauth/services/OAuthService.ts:108-165`

```typescript
async generateAuthorizationCode(
  clientId: string,
  userId: number,
  redirectUri: string,
  scopes: AccessScope[],
  state?: string,
  _teamSlug?: string,
  codeChallenge?: string,
  codeChallengeMethod?: string
): Promise<AuthorizeResult> {
  const client = await this.oAuthClientRepository.findByClientId(clientId);

  if (!client) {
    throw new ErrorWithCode(ErrorCode.Unauthorized, "unauthorized_client", { reason: "client_not_found" });
  }

  // ⚠️ 状态门禁检查
  // Allow PENDING clients if the logged-in user owns them (for developer testing).
  // REJECTED clients are always blocked regardless of ownership.
  this.ensureClientIsApprovedOrOwnedPending(client, userId);

  // ... 后续逻辑
}
```

#### 阶段 2：获取授权客户端

位置：`packages/features/oauth/services/OAuthService.ts:81-106`

```typescript
async getClientForAuthorization(
  clientId: string,
  redirectUri: string,
  userId?: number
): Promise<OAuth2Client> {
  const client = await this.oAuthClientRepository.findByClientId(clientId);

  if (!client) {
    throw new ErrorWithCode(ErrorCode.NotFound, "unauthorized_client", { reason: "client_not_found" });
  }

  this.validateRedirectUri(client.redirectUri, redirectUri);

  // ⚠️ 状态门禁检查
  this.ensureClientIsApprovedOrOwnedPending(client, userId);

  return {
    clientId: client.clientId,
    redirectUri: client.redirectUri,
    name: client.name,
    logo: client.logo,
    isTrusted: client.isTrusted,
    clientType: client.clientType,
  };
}
```

#### 阶段 3：授权码换 Token

位置：`packages/features/oauth/services/OAuthService.ts:280-330`

```typescript
async exchangeCodeForTokens(
  clientId: string,
  code: string,
  clientSecret?: string,
  redirectUri?: string,
  codeVerifier?: string
): Promise<OAuth2Tokens> {
  const client = await this.oAuthClientRepository.findByClientIdWithSecret(clientId);
  if (!client) {
    throw new ErrorWithCode(ErrorCode.Unauthorized, "invalid_client", { reason: "client_not_found" });
  }

  // ... 验证 clientSecret、redirectUri、PKCE 等

  const accessCode = await this.accessCodeRepository.findValidCode(code, clientId);

  if (!accessCode) {
    throw new ErrorWithCode(ErrorCode.BadRequest, "invalid_grant", { reason: "code_invalid_or_expired" });
  }

  // ⚠️ 状态门禁检查 (使用 accessCode.userId)
  this.ensureClientIsApprovedOrOwnedPending(client, accessCode.userId);

  // ... 创建 Token
}
```

#### 阶段 4：刷新 Token

位置：`packages/features/oauth/services/OAuthService.ts:332-370`

```typescript
async refreshAccessToken(
  clientId: string,
  refreshToken: string,
  clientSecret?: string
): Promise<OAuth2Tokens> {
  const client = await this.oAuthClientRepository.findByClientIdWithSecret(clientId);

  if (!client) {
    throw new ErrorWithCode(ErrorCode.Unauthorized, "invalid_client", { reason: "client_not_found" });
  }

  // ... 验证 clientSecret

  const decodedToken = this.verifyRefreshToken(refreshToken);

  if (!decodedToken || decodedToken.token_type !== "Refresh Token") {
    throw new ErrorWithCode(ErrorCode.BadRequest, "invalid_grant", { reason: "invalid_refresh_token" });
  }

  // ⚠️ 状态门禁检查 (使用 decodedToken.userId)
  this.ensureClientIsApprovedOrOwnedPending(client, decodedToken.userId);

  // ... 创建新 Token
}
```

### 2.5 状态门禁流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    OAuthClient 状态门禁完整流程                               │
└─────────────────────────────────────────────────────────────────────────────────┘

  请求到达
      │
      ▼
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  阶段 1: generateAuthorizationCode (授权码签发)                             │
  │  - 验证 clientId 存在                                                        │
  │  - 验证 redirectUri 匹配                                                     │
  │  - ⚠️ 状态门禁: ensureClientIsApprovedOrOwnedPending(client, userId)         │
  │  - 验证 PKCE (如果是 PUBLIC 客户端)                                          │
  │  - 创建 AccessCode                                                            │
  └─────────────────────────────────────────────────────────────────────────────┘
      │
      ▼ (用户重定向回第三方应用，携带 code)
      │
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  阶段 2: exchangeCodeForTokens (授权码换 Token)                              │
  │  - 验证 clientId + clientSecret (CONFIDENTIAL 客户端)                        │
  │  - 验证 code 有效且未过期                                                     │
  │  - ⚠️ 状态门禁: ensureClientIsApprovedOrOwnedPending(client, accessCode.userId) │
  │  - 验证 PKCE code_verifier                                                    │
  │  - 删除已使用的 AccessCode                                                     │
  │  - 签发 Access Token + Refresh Token                                           │
  └─────────────────────────────────────────────────────────────────────────────┘
      │
      ▼ (Token 过期后)
      │
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  阶段 3: refreshAccessToken (刷新 Token)                                      │
  │  - 验证 clientId + clientSecret                                               │
  │  - 验证 refreshToken 签名和有效性                                             │
  │  - ⚠️ 状态门禁: ensureClientIsApprovedOrOwnedPending(client, decodedToken.userId) │
  │  - 签发新的 Access Token + Refresh Token                                       │
  └─────────────────────────────────────────────────────────────────────────────┘
```

### 2.6 状态变更管理

#### 客户端创建

位置：`packages/trpc/server/routers/viewer/oAuth/createClient.handler.ts:34`

```typescript
// 创建时默认状态为 APPROVED
status: "APPROVED",
```

#### 提交审批

位置：`packages/trpc/server/routers/viewer/oAuth/submitClientForReview.handler.ts:44`

```typescript
// 提交审批时状态变为 PENDING
status: "PENDING",
```

#### 状态更新（管理员）

位置：`packages/trpc/server/routers/viewer/oAuth/updateClient.handler.ts:30-47`

```typescript
// 只有管理员可以修改状态
if (status) {
  if (ctx.user.role !== "ADMIN") {
    throw new TRPCError({ code: "FORBIDDEN", message: "Only admins can change client status" });
  }
  await oAuthClientRepository.updateStatus(clientId, status);
  
  // 如果状态为 REJECTED，需要记录拒绝原因
  if (status === OAuthClientStatus.REJECTED && rejectionReason) {
    await ctx.prisma.oAuthClient.update({
      where: { clientId },
      data: { rejectionReason },
    });
  }
}
```

#### 重定向 URI 变更触发重新审批

位置：`packages/trpc/server/routers/viewer/oAuth/updateClient.handler.ts:69-76`

```typescript
// 如果已批准的客户端修改了 redirectUri，重置为 PENDING 重新审批
if (
  updateFields.redirectUri &&
  existingClient.status === OAuthClientStatus.APPROVED &&
  updateFields.redirectUri !== existingClient.redirectUri
) {
  await oAuthClientRepository.updateStatus(clientId, OAuthClientStatus.PENDING);
}
```

---

## 第三部分：权限模型与端到端映射链

### 3.1 两套权限模型对比

#### 模型 1：`OAuthClient` + `AccessScope`

位置：`packages/prisma/schema.prisma:1496-1499`

```prisma
enum AccessScope {
  READ_BOOKING
  READ_PROFILE
}
```

**使用场景**：
- 传统 OAuth 2.0 授权码流程
- `AccessCode` 表的 `scopes` 字段
- `OAuthService.createTokens` 中的 JWT Payload

#### 模型 2：`PlatformOAuthClient` + 位掩码权限

位置：`packages/platform/constants/permissions.ts:1-69`

```typescript
// 10 种权限，使用 2 的幂次
export const EVENT_TYPE_READ = 1;   // 2^0
export const EVENT_TYPE_WRITE = 2;  // 2^1
export const BOOKING_READ = 4;      // 2^2
export const BOOKING_WRITE = 8;     // 2^3
export const SCHEDULE_READ = 16;    // 2^4
export const SCHEDULE_WRITE = 32;   // 2^5
export const APPS_READ = 64;        // 2^6
export const APPS_WRITE = 128;      // 2^7
export const PROFILE_READ = 256;    // 2^8
export const PROFILE_WRITE = 512;   // 2^9

// 权限分组
export const PERMISSIONS_GROUPED_MAP = {
  EVENT_TYPE: { read: EVENT_TYPE_READ, write: EVENT_TYPE_WRITE, key: "eventType", label: "Event Type" },
  BOOKING: { read: BOOKING_READ, write: BOOKING_WRITE, key: "booking", label: "Booking" },
  SCHEDULE: { read: SCHEDULE_READ, write: SCHEDULE_WRITE, key: "schedule", label: "Schedule" },
  APPS: { read: APPS_READ, write: APPS_WRITE, key: "apps", label: "Apps" },
  PROFILE: { read: PROFILE_READ, write: PROFILE_WRITE, key: "profile", label: "Profile" },
};
```

### 3.2 权限模型覆盖范围对比

| 资源类型 | `AccessScope` (传统) | 位掩码权限 (Platform) |
|---------|---------------------|---------------------|
| **Event Type** | ❌ 不支持 | ✅ `EVENT_TYPE_READ` / `EVENT_TYPE_WRITE` |
| **Booking** | ✅ `READ_BOOKING` | ✅ `BOOKING_READ` / `BOOKING_WRITE` |
| **Schedule** | ❌ 不支持 | ✅ `SCHEDULE_READ` / `SCHEDULE_WRITE` |
| **Apps** | ❌ 不支持 | ✅ `APPS_READ` / `APPS_WRITE` |
| **Profile** | ✅ `READ_PROFILE` | ✅ `PROFILE_READ` / `PROFILE_WRITE` |
| **写权限** | ❌ 无 | ✅ 5 种写权限 |

### 3.3 端到端映射链：传统 OAuth 2.0 路径

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              传统 OAuth 2.0 路径 (OAuthClient + AccessScope)                  │
└─────────────────────────────────────────────────────────────────────────────────┘

  1. 客户端注册
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  OAuthClient 创建                                                             │
  │  - status: APPROVED (默认)                                                   │
  │  - clientSecret: 哈希存储 (generateSecret)                                    │
  │  - userId: 客户端所有者                                                        │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  2. 授权请求
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  GET /oauth/authorize                                                        │
  │  参数: client_id, redirect_uri, scope, state                                  │
  │                                                                               │
  │  scope 示例: ["READ_BOOKING", "READ_PROFILE"]                                │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  3. 状态门禁检查 (阶段 1)
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  ensureClientIsApprovedOrOwnedPending(client, userId)                        │
  │                                                                               │
  │  决策逻辑:                                                                    │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  if (client.status === APPROVED) → ✅ 通过                              ││
  │  │  else if (client.status === REJECTED) → ❌ client_rejected               ││
  │  │  else if (client.status === PENDING && client.userId === userId) → ✅ 通过││
  │  │  else → ❌ client_not_approved                                            ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼ (通过)
  4. 授权码签发
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  AccessCode 创建                                                             │
  │  - code: 随机字符串 (base64url 编码)                                         │
  │  - scopes: AccessScope[] (从请求参数获取)                                     │
  │  - userId: 当前登录用户                                                        │
  │  - expiresAt: 短期有效                                                        │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼ (重定向回 redirect_uri?code=xxx)
            │
  5. 授权码换 Token (阶段 2)
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  POST /oauth/token (grant_type=authorization_code)                           │
  │                                                                               │
  │  ⚠️ 状态门禁检查: ensureClientIsApprovedOrOwnedPending(client, accessCode.userId) │
  │                                                                               │
  │  Access Token JWT Payload:                                                   │
  │  {                                                                           │
  │    userId: number,                                                           │
  │    teamId: number | null,                                                    │
  │    scope: AccessScope[],  // ⚠️ 从 AccessCode 继承                          │
  │    token_type: "Access Token",                                               │
  │    clientId: string,                                                         │
  │    iat: number,                                                              │
  │    exp: number (30分钟后)                                                    │
  │  }                                                                           │
  │                                                                               │
  │  Refresh Token JWT Payload:                                                  │
  │  {                                                                           │
  │    userId: number,                                                           │
  │    teamId: number | null,                                                    │
  │    scope: AccessScope[],                                                     │
  │    token_type: "Refresh Token",                                              │
  │    clientId: string,                                                         │
  │    iat: number,                                                              │
  │    exp: number (30天后)                                                      │
  │  }                                                                           │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼ (使用 Access Token 访问 API)
            │
  6. 身份解析与资源访问
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  Third-Party JWT Token 认证 (tokens.service.ts)                              │
  │                                                                               │
  │  JWT Payload 解析:                                                           │
  │  type OAuthTokenPayload = {                                                  │
  │    userId?: number;     // ⚠️ 用户身份                                        │
  │    teamId?: number;     // ⚠️ 组织/团队身份                                   │
  │    scope: string[];     // ⚠️ 权限范围                                       │
  │    token_type: string;                                                       │
  │  }                                                                            │
  │                                                                               │
  │  身份解析逻辑 (api-auth.strategy.ts:320-357):                               │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  if (decodedToken.userId) {                                              ││
  │  │    // 优先使用 userId 获取用户                                             ││
  │  │    user = await userRepository.findByIdWithProfile(decodedToken.userId); ││
  │  │    organizationId = usersService.getUserMainOrgId(user);                 ││
  │  │  } else if (decodedToken.teamId) {                                        ││
  │  │    // 其次使用 teamId 获取团队所有者                                        ││
  │  │    teamOwner = await userRepository.findOwnerByTeamIdWithProfile(         ││
  │  │      decodedToken.teamId                                                   ││
  │  │    );                                                                       ││
  │  │    organizationId = decodedToken.teamId;                                  ││
  │  │  }                                                                          ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  │                                                                               │
  │  ⚠️ 注意: 传统 OAuth 2.0 路径使用的 AccessScope 不参与 PermissionsGuard 检查   │
  │       因为 Third-Party Access Token 会跳过权限守卫                            │
  └─────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 端到端映射链：Platform API 路径

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              Platform API 路径 (PlatformOAuthClient + 位掩码权限)              │
└─────────────────────────────────────────────────────────────────────────────────┘

  1. 客户端注册
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  PlatformOAuthClient 创建                                                    │
  │  - ❌ 无状态字段 (直接可用)                                                   │
  │  - secret: JWT 签名 (包含客户端配置)                                          │
  │  - permissions: Int (位掩码，如 12 = BOOKING_READ | BOOKING_WRITE)           │
  │  - organizationId: 必须绑定组织                                                │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  2. 创建托管用户
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  POST /oauth/{clientId}/users                                                │
  │  (OAuthClientUsersService.createOAuthClientUser)                             │
  │                                                                               │
  │  流程:                                                                        │
  │  1. 验证 organizationId 存在                                                 │
  │  2. 创建用户并连接到组织                                                       │
  │  3. 将用户关联到 OAuth Client                                                 │
  │  4. ⚠️ 自动签发 Access Token + Refresh Token                                   │
  │     - 不经过 OAuth 2.0 授权流程                                              │
  │     - 不涉及 AccessScope                                                      │
  │  5. 可选: 创建默认事件类型、默认日程                                            │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  3. Platform 授权流程 (可选)
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  POST /v2/oauth/{clientId}/authorize                                         │
  │  (NextAuthGuard 保护，需要用户登录)                                            │
  │                                                                               │
  │  流程:                                                                        │
  │  1. 验证 OAuth Client 存在                                                    │
  │  2. 验证 redirectUri 在白名单中                                               │
  │  3. 检查用户是否已授权 (PlatformAuthorizationToken)                           │
  │  4. 创建 PlatformAuthorizationToken                                          │
  │  5. 重定向: redirectUri?code={tokenId}                                        │
  │                                                                               │
  │  ⚠️ 注意: 此流程不检查客户端状态 (PlatformOAuthClient 无 status 字段)         │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  4. 交换授权码为 Token
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  POST /v2/oauth/{clientId}/exchange                                          │
  │                                                                               │
  │  流程 (oauth-flow.service.ts:106-138):                                       │
  │  1. 验证 OAuth Client + clientSecret                                          │
  │  2. 获取 PlatformAuthorizationToken                                           │
  │  3. ⚠️ 创建 AccessToken + RefreshToken                                         │
  │     - JWT Payload 包含: clientId, ownerId, expiresAt, jti                   │
  │     - ❌ 不包含 scope 字段                                                    │
  │  4. 使 PlatformAuthorizationToken 失效 (删除)                                 │
  │  5. 将 AccessToken 传播到 Redis 缓存                                          │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  5. 刷新 Token
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  POST /v2/oauth/{clientId}/refresh                                           │
  │                                                                               │
  │  流程 (tokens.repository.ts:171-213):                                        │
  │  1. 删除过期的 AccessToken                                                    │
  │  2. 删除当前 RefreshToken (刷新令牌轮换)                                       │
  │  3. 创建新的 AccessToken + RefreshToken                                        │
  │                                                                               │
  │  ⚠️ 注意: 此流程不检查客户端状态                                               │
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  6. 身份解析 (Access Token 认证)
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  Access Token 认证流程 (api-auth.strategy.ts:261-299)                        │
  │                                                                               │
  │  步骤 1: 验证 Token 有效性                                                    │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  // 先查 Redis 缓存                                                       ││
  │  │  const { status } = await oauthFlowService.readFromCache(secret);        ││
  │  │  if (status === "CACHE_HIT") return true;                                 ││
  │  │                                                                              ││
  │  │  // 缓存未命中，查数据库                                                    ││
  │  │  const tokenExpiresAt = await tokensRepository.getAccessTokenExpiryDate(  ││
  │  │    secret                                                                    ││
  │  │  );                                                                          ││
  │  │  if (new Date() > tokenExpiresAt) throw new TokenExpiredException();       ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  │                                                                               │
  │  步骤 2: 获取关联的 OAuth Client                                              │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  const client = await tokensRepository.getAccessTokenClient(accessToken);  ││
  │  │  // client.permissions: 位掩码 (用于后续权限检查)                           ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  │                                                                               │
  │  步骤 3: 验证请求来源 (CORS)                                                  │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  if (origin && !isOriginAllowed(origin, client.redirectUris)) {          ││
  │  │    throw new UnauthorizedException("Invalid request origin");             ││
  │  │  }                                                                          ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  │                                                                               │
  │  步骤 4: 获取用户身份                                                        │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  const ownerId = await tokensRepository.getAccessTokenOwnerId(accessToken);││
  │  │  const user = await userRepository.findByIdWithProfile(ownerId);          ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  │                                                                               │
  │  步骤 5: 确定组织归属                                                        │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  const organizationId = this.usersService.getUserMainOrgId(user) as number;││
  │  │  request.organizationId = organizationId;                                  ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  7. 权限检查 (PermissionsGuard)
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  权限守卫 (permissions.guard.ts:27-72)                                       │
  │                                                                               │
  │  步骤 1: 从装饰器获取要求的权限                                                │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  const requiredPermissions = this.reflector.get(Permissions, handler);    ││
  │  │  if (!requiredPermissions?.length) return true; // 无要求，直接通过        ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  │                                                                               │
  │  步骤 2: 判断认证方式，决定是否跳过权限检查                                    │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  // 以下认证方式跳过权限检查:                                               ││
  │  │  // - NextAuth Token (会话认证)                                           ││
  │  │  // - API Key (直接代表用户)                                               ││
  │  │  // - Third-Party Access Token (外部 JWT)                                  ││
  │  │                                                                              ││
  │  │  if (nextAuthToken || apiKey || isThirdPartyBearerToken) {                ││
  │  │    return true; // 跳过权限检查                                             ││
  │  │  }                                                                          ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  │                                                                               │
  │  步骤 3: OAuth Access Token 需要检查权限                                       │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  // 获取 OAuth Client 的 permissions 位掩码                                 ││
  │  │  const oAuthClient = bearerToken                                           ││
  │  │    ? await this.getOAuthClientByAccessToken(bearerToken)                   ││
  │  │    : await this.getOAuthClientById(oAuthClientId);                         ││
  │  │                                                                              ││
  │  │  // 位运算权限检查                                                          ││
  │  │  const hasRequiredPermissions = hasPermissions(                              ││
  │  │    oAuthClient.permissions,                                                 ││
  │  │    [...requiredPermissions]                                                  ││
  │  │  );                                                                          ││
  │  │                                                                              ││
  │  │  // hasPermissions 实现:                                                     ││
  │  │  // return permissions.every((permission) =>                                 ││
  │  │  //   (userPermissions & permission) === permission                           ││
  │  │  // );                                                                       ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  │                                                                               │
  │  步骤 4: 权限不满足时抛出错误                                                  │
  │  ┌─────────────────────────────────────────────────────────────────────────┐│
  │  │  if (!hasRequiredPermissions) {                                            ││
  │  │    throw new ForbiddenException(                                            ││
  │  │      `OAuth client with id=${oAuthClient.id} does not have the required ` +││
  │  │      `permissions. Go to platform dashboard settings and add the required `+││
  │  │      `permissions to the oAuth client.`                                     ││
  │  │    );                                                                        ││
  │  │  }                                                                           ││
  │  └─────────────────────────────────────────────────────────────────────────┘│
  └─────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  8. 资源访问
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │  资源访问控制依赖:                                                           │
  │                                                                               │
  │  1. 组织隔离: request.organizationId                                         │
  │     - 大部分 API 查询会过滤 organizationId                                   │
  │     - 确保用户只能访问所属组织的资源                                           │
  │                                                                               │
  │  2. 权限控制: PermissionsGuard                                                │
  │     - 仅对 OAuth Access Token 生效                                            │
  │     - 基于位掩码检查资源操作权限                                               │
  │                                                                               │
  │  3. 示例: Booking API                                                         │
  │     - 需要 BOOKING_READ 权限读取预约                                          │
  │     - 需要 BOOKING_WRITE 权限创建/修改/取消预约                                │
  │     - 只能访问 organizationId 对应的组织的预约                                 │
  └─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第四部分：两套体系一致性分析与风险点

### 4.1 核心不一致性汇总

| 维度 | `OAuthClient` (传统) | `PlatformOAuthClient` (平台) | 影响 |
|------|---------------------|------------------------------|------|
| **状态门禁** | ✅ 完整支持 PENDING/APPROVED/REJECTED | ❌ 无状态字段 | 安全风险：平台客户端无法审批控制 |
| **权限模型** | `AccessScope` 枚举 (2种读权限) | 位掩码 (10种读写权限) | 功能差异：传统 OAuth 无法请求写权限 |
| **权限守卫参与** | ❌ Third-Party Token 跳过检查 | ✅ Access Token 参与检查 | 安全风险：传统 OAuth 权限不生效 |
| **组织绑定** | 可选 (userId) | 必须 (organizationId) | 架构差异 |
| **密钥存储** | 哈希存储 | JWT 明文 | 安全差异 |

### 4.2 权限映射不一致风险

#### 风险 1：`AccessScope` 与位掩码权限无映射

**问题**：
- 传统 OAuth 2.0 使用 `AccessScope` 枚举 (`READ_BOOKING`, `READ_PROFILE`)
- Platform API 使用位掩码权限 (10种)
- **两者之间没有任何映射关系**

**代码证据**：
```typescript
// OAuthService.createTokens 中 (传统 OAuth)
const accessTokenPayload = {
  userId: input.userId,
  teamId: input.teamId,
  scope: input.scopes,  // AccessScope[]
  token_type: "Access Token",
  clientId: input.clientId,
};

// 但 PermissionsGuard 中 (Platform API)
// 只检查 oAuthClient.permissions (位掩码)
// 完全忽略 JWT 中的 scope 字段
```

**实际影响**：
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  传统 OAuth 2.0 权限范围不生效的完整链路                                      │
└─────────────────────────────────────────────────────────────────────────────┘

  1. 用户授权时请求 scope: ["READ_BOOKING"]
            │
            ▼
  2. AccessCode 记录 scopes: ["READ_BOOKING"]
            │
            ▼
  3. 换 Token 时，JWT Payload 继承 scope: ["READ_BOOKING"]
            │
            ▼
  4. 第三方应用使用 Access Token 访问 API
            │
            ▼
  5. ApiAuthStrategy 识别为 Third-Party Access Token
            │
            ▼
  6. PermissionsGuard 判断: isThirdPartyBearerToken === true
            │
            ▼
  7. 直接返回 true，跳过权限检查！
            │
            ▼
  8. 即使 scope 只有 ["READ_BOOKING"]，也能执行写操作！
```

#### 风险 2：`PlatformOAuthClient` 无状态门禁

**问题**：
- `OAuthClient` 有完整的状态审批机制
- `PlatformOAuthClient` 没有 `status` 字段
- 创建后直接可用，无法进行审批控制

**代码证据**：
```prisma
// PlatformOAuthClient 模型 (无 status 字段)
model PlatformOAuthClient {
  id             String   @id @default(cuid())
  name           String
  secret         String
  permissions    Int  // 只有权限位掩码，无状态
  // ... 其他字段
}

// OAuthClient 模型 (有 status 字段)
model OAuthClient {
  clientId        String            @id @unique
  // ... 其他字段
  status          OAuthClientStatus @default(APPROVED)  // ⚠️ 状态字段
  rejectionReason String?           // ⚠️ 拒绝原因
}
```

**实际影响**：
```
┌─────────────────────────────────────────────────────────────────────────────┐
│  PlatformOAuthClient 无状态门禁的风险                                        │
└─────────────────────────────────────────────────────────────────────────────┘

  对比场景:

  【传统 OAuthClient】
  1. 开发者创建客户端 → status: APPROVED (默认)
  2. 开发者提交审核 → status: PENDING
  3. 管理员拒绝 → status: REJECTED
  4. 拒绝后:
     - 授权码签发失败 (client_rejected)
     - 换 Token 失败 (client_rejected)
     - 刷新 Token 失败 (client_rejected)
  5. ✅ 管理员可以有效阻止被拒绝客户端的访问

  【PlatformOAuthClient】
  1. 平台管理员创建客户端 → 直接可用
  2. 没有 status 字段，无法审批
  3. 无法拒绝客户端
  4. 即使发现恶意客户端，也只能删除
  5. ❌ 删除前已签发的 Token 仍然有效！
```

#### 风险 3：刷新令牌时状态不重新验证

**问题**：
- 传统 OAuth 在刷新 Token 时会重新检查状态
- 但如果客户端状态在 Token 签发后变更，已签发的 Token 仍然有效

**实际场景**：
```
时间线:

T0: 客户端状态 = APPROVED
    用户授权 → 获取 Access Token (30分钟) + Refresh Token (30天)

T5: 管理员发现问题，将客户端状态改为 REJECTED
    ✅ 新的授权请求会被拒绝
    ✅ 新的换 Token 请求会被拒绝
    ⚠️ 但已签发的 Access Token 还有 25 分钟有效期
    ⚠️ 已签发的 Refresh Token 还有 30 天有效期

T10: 第三方应用使用 Refresh Token 刷新
    ✅ 传统 OAuth 会检查状态 → 拒绝 (client_rejected)
    ❌ Platform OAuth 无状态检查 → 仍然可以刷新！
```

### 4.3 权限守卫行为不一致

#### 不同认证方式的权限检查行为

| 认证方式 | 标识常量 | PermissionsGuard 行为 | 实际权限 |
|---------|---------|----------------------|---------|
| **NextAuth Token** | `NEXT_AUTH` | 跳过检查 | 用户所有权限 |
| **API Key** | `API_KEY` | 跳过检查 | 用户所有权限 |
| **Third-Party Access Token** | `THIRD_PARTY_ACCESS_TOKEN` | 跳过检查 | 用户所有权限 (⚠️ scope 不生效) |
| **OAuth Access Token** | `ACCESS_TOKEN` | 检查位掩码权限 | 受 permissions 限制 |
| **OAuth Client Credentials** | `OAUTH_CLIENT` | 检查位掩码权限 | 受 permissions 限制 |

#### 关键代码位置

`apps/api/v2/src/modules/auth/guards/permissions/permissions.guard.ts:42-45`

```typescript
// 以下认证方式跳过权限检查
if (nextAuthToken || apiKey || isThirdPartyBearerToken) {
  return true;
}

// 只有 OAuth Access Token 和 OAuth Client ID 需要检查权限
const oAuthClient = bearerToken
  ? await this.getOAuthClientByAccessToken(bearerToken)
  : await this.getOAuthClientById(oAuthClientId);

const hasRequiredPermissions = hasPermissions(oAuthClient.permissions, [...requiredPermissions]);
```

### 4.4 组织归属解析不一致

#### 不同认证方式的组织 ID 来源

| 认证方式 | 组织 ID 来源 | 代码位置 |
|---------|-------------|----------|
| **API Key** | `ApiKey.teamId` | `api-auth.strategy.ts:256` |
| **OAuth Client Credentials** | `PlatformOAuthClient.organizationId` | `api-auth.strategy.ts:206` |
| **OAuth Access Token** | `usersService.getUserMainOrgId(user)` | `api-auth.strategy.ts:295` |
| **Third-Party JWT** | `teamId` 或用户主组织 | `api-auth.strategy.ts:335-346` |
| **NextAuth Token** | `usersService.getUserMainOrgId(user)` | `api-auth.strategy.ts:314-315` |

#### 潜在问题

```
场景: 用户属于多个组织

1. 用户使用 OAuth Access Token 访问 API
   - organizationId = getUserMainOrgId(user) → 主组织
   - 但用户可能想访问其他组织的资源

2. 用户使用 Third-Party JWT (包含 teamId)
   - organizationId = decodedToken.teamId → 指定组织
   - 可以精确控制访问哪个组织

3. 不一致性:
   - OAuth Access Token 无法指定组织
   - Third-Party JWT 可以通过 teamId 指定组织
```

---

## 第五部分：一致性结论与改进建议

### 5.1 核心结论

#### 结论 1：两套体系并行，权限模型不统一

**现状**：
- **传统 OAuth 2.0** (`OAuthClient` + `AccessScope`)
  - 有完整的状态审批机制
  - 权限范围有限（仅 2 种读权限）
  - Third-Party Token 跳过权限检查 → **权限不生效**

- **Platform API** (`PlatformOAuthClient` + 位掩码)
  - 无状态审批机制
  - 权限范围完整（10 种读写权限）
  - 权限检查生效

**风险等级**：🔴 高

**影响**：
1. 传统 OAuth 的 scope 字段形同虚设
2. 管理员无法有效控制平台客户端的访问
3. 开发者容易混淆两套体系

#### 结论 2：状态门禁存在但不完整

**现状**：
- `OAuthClient` 有完整的状态门禁（PENDING/APPROVED/REJECTED）
- 状态门禁在 4 个关键阶段生效
- 但 `PlatformOAuthClient` 完全没有状态机制

**风险等级**：🟠 中

**影响**：
1. 平台客户端被删除前，已签发的 Token 仍然有效
2. 无法实现"软拒绝"（拒绝但保留配置）
3. 无法追踪客户端状态变更历史

#### 结论 3：权限守卫存在设计漏洞

**现状**：
- Third-Party Access Token 直接跳过权限检查
- 即使 JWT 中包含 scope 字段，也不参与验证
- 只有 OAuth Access Token 和 OAuth Client ID 会检查权限

**风险等级**：🔴 高

**影响**：
1. 传统 OAuth 2.0 的授权范围控制失效
2. 第三方应用可以超出授权范围访问资源
3. 安全审计困难（无法知道应用实际访问了哪些资源）

### 5.2 改进建议

#### 建议 1：统一权限模型

**短期方案**：
1. 为 `AccessScope` 添加写权限：
```prisma
enum AccessScope {
  READ_BOOKING
  READ_PROFILE
  WRITE_BOOKING    // 新增
  WRITE_PROFILE    // 新增
  // ... 其他资源
}
```

2. 建立 `AccessScope` 到位掩码的映射：
```typescript
const ACCESS_SCOPE_TO_PERMISSION_MAP: Record<AccessScope, number> = {
  [AccessScope.READ_BOOKING]: BOOKING_READ,
  [AccessScope.READ_PROFILE]: PROFILE_READ,
  [AccessScope.WRITE_BOOKING]: BOOKING_WRITE,
  [AccessScope.WRITE_PROFILE]: PROFILE_WRITE,
  // ... 其他映射
};
```

**长期方案**：
1. 统一使用位掩码权限模型
2. 弃用 `AccessScope` 枚举
3. 所有 OAuth 流程使用一致的权限表示

#### 建议 2：修复权限守卫漏洞

**立即修复**：
1. Third-Party Access Token 也应该检查权限
2. 从 JWT 的 scope 字段映射到位掩码进行检查

```typescript
// 建议修改 PermissionsGuard
if (isThirdPartyBearerToken) {
  // 不再直接跳过
  // 而是从 decodedToken.scope 映射到位掩码
  const thirdPartyPermissions = mapScopesToPermissions(decodedToken.scope);
  const hasRequiredPermissions = hasPermissions(thirdPartyPermissions, [...requiredPermissions]);
  
  if (!hasRequiredPermissions) {
    throw new ForbiddenException(
      `Third-party token does not have required permissions`
    );
  }
  return true;
}
```

#### 建议 3：为 `PlatformOAuthClient` 添加状态机制

**建议修改**：
```prisma
model PlatformOAuthClient {
  // ... 现有字段
  
  // 新增状态字段
  status          OAuthClientStatus @default(APPROVED)
  rejectionReason String?
  
  // 新增审核相关字段
  submittedAt     DateTime?
  reviewedAt      DateTime?
  reviewedBy      Int?
  
  // ... 现有字段
}
```

**实现状态检查**：
1. 在 `ApiAuthStrategy.accessTokenStrategy` 中添加状态检查
2. 在 `OAuthFlowService` 各方法中添加状态检查
3. 与 `OAuthClient` 的状态门禁逻辑保持一致

#### 建议 4：Token 刷新时强制状态重验

**建议实现**：
```typescript
// 在 refreshOAuthTokens 中
async refreshOAuthTokens(clientId: string, refreshTokenSecret: string, tokenUserId: number) {
  // 1. 验证客户端状态
  const client = await this.oAuthClientRepository.getOAuthClient(clientId);
  
  // 检查状态（与 generateAuthorizationCode 相同逻辑）
  this.ensureClientIsApprovedOrOwnedPending(client, tokenUserId);
  
  // 2. 继续原有逻辑...
}
```

**额外增强**：
1. 已签发的 Access Token 应该在客户端状态变更时失效
2. 可以通过 Redis 缓存黑名单实现
3. 或者缩短 Access Token 有效期（如 5 分钟）

### 5.3 安全建议优先级

| 优先级 | 建议 | 风险等级 | 实施难度 |
|-------|------|---------|---------|
| 🔴 P0 | 修复 Third-Party Token 权限检查漏洞 | 高 | 中 |
| 🔴 P0 | 为 PlatformOAuthClient 添加状态机制 | 高 | 中 |
| 🟠 P1 | 建立 AccessScope 到位掩码的映射 | 中 | 低 |
| 🟠 P1 | Token 刷新时强制状态重验 | 中 | 低 |
| 🟡 P2 | 统一权限模型（长期） | 中 | 高 |

---

## 附录：代码引用索引

### 状态门禁相关

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 状态枚举定义 | `packages/prisma/schema.prisma` | 1455-1459 |
| 状态门禁核心实现 | `packages/features/oauth/services/OAuthService.ts` | 167-207 |
| 授权码签发状态检查 | `packages/features/oauth/services/OAuthService.ts` | 126 |
| 换 Token 状态检查 | `packages/features/oauth/services/OAuthService.ts` | 312 |
| 刷新 Token 状态检查 | `packages/features/oauth/services/OAuthService.ts` | 360 |
| 管理员更新状态 | `packages/trpc/server/routers/viewer/oAuth/updateClient.handler.ts` | 30-47 |
| 重定向 URI 变更触发重审 | `packages/trpc/server/routers/viewer/oAuth/updateClient.handler.ts` | 69-76 |

### 权限模型相关

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| AccessScope 枚举 | `packages/prisma/schema.prisma` | 1496-1499 |
| 位掩码权限常量 | `packages/platform/constants/permissions.ts` | 1-69 |
| 权限检查工具函数 | `packages/platform/utils/permissions.ts` | 16-73 |
| 权限守卫实现 | `apps/api/v2/src/modules/auth/guards/permissions/permissions.guard.ts` | 27-72 |
| 第三方 Token 解码 | `apps/api/v2/src/modules/tokens/tokens.service.ts` | 32-50 |

### 身份解析相关

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| API Key 身份解析 | `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts` | 236-259 |
| OAuth Access Token 身份解析 | `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts` | 261-299 |
| Third-Party Token 身份解析 | `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts` | 320-357 |

---

**文档版本**：v1.0  
**最后更新**：2026-05-04  
**分析人员**：AI 代码分析助手  
**审核状态**：待评审
