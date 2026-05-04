# Cal.diy API Key & OAuth 权限范围分析报告

## 概述

本文档详细分析了 Cal.diy 平台中 API Key、OAuth Client 的注册管理机制，以及 Token 权限范围如何映射到用户、组织和资源访问。

---

## 第一部分：API Key 管理机制

### 1.1 数据库模型 (ApiKey)

API Key 的数据模型定义在 `packages/prisma/schema.prisma:1172`：

```prisma
model ApiKey {
  id         String      @id @unique @default(cuid())
  userId     Int
  teamId     Int?
  note       String?
  createdAt  DateTime    @default(now())
  expiresAt  DateTime?
  lastUsedAt DateTime?
  hashedKey  String      @unique()
  
  // 关联关系
  user       User?       @relation(fields: [userId], references: [id], onDelete: Cascade)
  team       Team?       @relation(fields: [teamId], references: [id], onDelete: Cascade)
  app        App?        @relation(fields: [appId], references: [slug], onDelete: Cascade)
  appId      String?
  rateLimits RateLimit[]
}
```

**关键字段说明**：
- `hashedKey`: 存储 SHA256 哈希后的 API Key（安全存储，不存储明文）
- `userId`: API Key 所属用户的 ID
- `teamId`: API Key 关联的团队/组织 ID（可选）
- `expiresAt`: 过期时间，`null` 表示永不过期
- `note`: 备注信息，用于标识 API Key 的用途

### 1.2 API Key 服务层实现

**Repository 层** (`apps/api/v2/src/modules/api-keys/api-keys-repository.ts`)：

```typescript
@Injectable()
export class ApiKeysRepository {
  // 通过哈希值查找 API Key
  async getApiKeyFromHash(hashedKey: string) {
    return this.dbRead.prisma.apiKey.findUnique({
      where: { hashedKey },
    });
  }

  // 获取团队的所有 API Key
  async getTeamApiKeys(teamId: number) {
    return this.dbRead.prisma.apiKey.findMany({
      where: { teamId },
    });
  }

  // 删除 API Key
  async deleteById(id: string) {
    return this.dbWrite.prisma.apiKey.delete({
      where: { id },
    });
  }
}
```

**Service 层** (`apps/api/v2/src/modules/api-keys/services/api-keys.service.ts`)：

核心功能：
1. **创建 API Key** (`createApiKey`):
   - 支持设置有效期天数 (`apiKeyDaysValid`) 或永不过期 (`apiKeyNeverExpires`)
   - 默认有效期为 30 天
   - 调用 `@calcom/platform-libraries` 中的 `createApiKeyHandler`
   - 可关联到特定的 `teamId`

2. **刷新 API Key** (`refreshApiKey`):
   - 验证原 API Key 的有效性
   - 创建新的 API Key（保留原有的 note 和 teamId）
   - 删除旧的 API Key

3. **从请求获取 API Key** (`getRequestApiKey`):
   - 验证请求使用的是 API_KEY 认证方式
   - 从 `Authorization` 头提取 Bearer token

### 1.3 API Key 认证流程

API Key 的认证在 `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts` 中实现：

```typescript
async apiKeyStrategy(apiKey: string, request: ApiAuthGuardRequest) {
  // 1. 剥离 API Key 前缀（如 "cal_"）
  const strippedApiKey = stripApiKey(apiKey, this.config.get<string>("api.keyPrefix"));
  
  // 2. 计算 SHA256 哈希
  const apiKeyHash = sha256Hash(strippedApiKey);
  
  // 3. 在数据库中查找
  const keyData = await this.apiKeyRepository.getApiKeyFromHash(apiKeyHash);
  if (!keyData) {
    throw new UnauthorizedException("Your api key is not valid");
  }

  // 4. 检查是否过期
  const isKeyExpired = keyData.expiresAt && 
    new Date().setHours(0, 0, 0, 0) > keyData.expiresAt.setHours(0, 0, 0, 0);
  if (isKeyExpired) {
    throw new UnauthorizedException("Your api key is expired");
  }

  // 5. 获取 API Key 所有者用户
  const user = await this.userRepository.findByIdWithProfile(keyData.userId);
  
  // 6. 设置组织 ID（来自 teamId）
  request.organizationId = keyData.teamId;

  return user;
}
```

**安全设计要点**：
- API Key 仅存储哈希值，不存储明文
- 支持过期时间，防止长期有效的密钥泄露风险
- 通过 `teamId` 关联到组织级别

---

## 第二部分：OAuth Client 管理机制

### 2.1 数据库模型 (PlatformOAuthClient)

OAuth Client 的数据模型定义在 `packages/prisma/schema.prisma:1716`：

```prisma
model PlatformOAuthClient {
  id             String   @id @default(cuid())
  name           String
  secret         String
  permissions    Int
  
  // 基础配置
  logo           String?
  redirectUris   String[]
  organizationId Int
  organization   Team     @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  
  // 关联关系
  users          User[]
  teams          Team[]   @relation("CreatedByOAuthClient")
  accessTokens        AccessToken[]
  refreshToken        RefreshToken[]
  authorizationTokens PlatformAuthorizationToken[]
  webhook             Webhook[]
  
  // 预订相关重定向 URI
  bookingRedirectUri           String?
  bookingCancelRedirectUri     String?
  bookingRescheduleRedirectUri String?
  
  // 功能开关
  areEmailsEnabled             Boolean @default(false)
  areDefaultEventTypesEnabled  Boolean @default(true)
  areCalendarEventsEnabled     Boolean @default(true)
  
  createdAt DateTime @default(now())
}
```

**关键字段说明**：
- `secret`: 客户端密钥，使用 JWT 签名生成
- `permissions`: 权限位掩码（整数，使用位运算管理权限）
- `organizationId`: 客户端所属的组织 ID
- `redirectUris`: OAuth 授权回调 URI 列表
- `areEmailsEnabled`: 是否启用邮件功能
- `areDefaultEventTypesEnabled`: 创建托管用户时是否创建默认事件类型
- `areCalendarEventsEnabled`: 是否启用日历事件功能

### 2.2 相关 Token 模型

#### AccessToken 模型

```prisma
model AccessToken {
  id Int @id @default(autoincrement())
  
  secret    String   @unique
  createdAt DateTime @default(now())
  expiresAt DateTime
  
  // 关联关系
  owner  User                @relation(fields: [userId], references: [id], onDelete: Cascade)
  client PlatformOAuthClient @relation(fields: [platformOAuthClientId], references: [id], onDelete: Cascade)
  
  platformOAuthClientId String
  userId                Int
}
```

#### RefreshToken 模型

```prisma
model RefreshToken {
  id Int @id @default(autoincrement())
  
  secret    String   @unique
  createdAt DateTime @default(now())
  expiresAt DateTime
  
  // 关联关系
  owner  User                @relation(fields: [userId], references: [id], onDelete: Cascade)
  client PlatformOAuthClient @relation(fields: [platformOAuthClientId], references: [id], onDelete: Cascade)
  
  platformOAuthClientId String
  userId                Int
}
```

### 2.3 OAuth Client 服务层实现

**Repository 层** (`apps/api/v2/src/modules/oauth-clients/oauth-client.repository.ts`)：

核心功能：
- `createOAuthClient`: 创建 OAuth 客户端
- `getOAuthClient`: 根据 clientId 获取客户端
- `getOAuthClientWithAuthTokens`: 获取客户端及其授权令牌
- `getOAuthClientWithRefreshSecret`: 获取客户端及其刷新令牌
- `getOrganizationOAuthClients`: 获取组织的所有 OAuth 客户端
- `updateOAuthClient`: 更新客户端
- `deleteOAuthClient`: 删除客户端
- `getByUserId`/`getByTeamId`/`getByOrgId`: 通过不同维度查找客户端

**Service 层** (`apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients.service.ts`)：

核心功能：
1. **创建 OAuth 客户端** (`createOAuthClient`):
   - 转换输入参数（权限转换为位掩码）
   - 生成 JWT 签名作为 secret
   - 返回 `clientId` 和 `clientSecret`

2. **获取客户端列表** (`getOAuthClients`):
   - 获取组织的所有 OAuth 客户端
   - 转换输出格式

3. **客户端 CRUD 操作**:
   - `getOAuthClientById`: 根据 ID 获取单个客户端
   - `updateOAuthClient`: 更新客户端信息
   - `deleteOAuthClient`: 删除客户端

### 2.4 OAuth Client 注册流程

创建 OAuth 客户端的输入处理在 `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients-input.service.ts`：

```typescript
transformCreateOAuthClientInput(input: CreateOAuthClientInput) {
  const transformed = {
    ...input,
    permissions: this.transformPermissions(input.permissions),
  };

  return {
    ...transformed,
    secret: this.jwtService.sign(transformed),  // 使用 JWT 签名生成 secret
  };
}

transformPermissions(permissions: Array<keyof typeof PERMISSION_MAP | "*">): number {
  // 如果使用 "*"，则授予所有权限
  const values = permissions.includes("*")
    ? Object.values(PERMISSION_MAP)
    : permissions.map((p) => PERMISSION_MAP[p as keyof typeof PERMISSION_MAP]);
  
  // 使用按位或运算组合权限
  return values.reduce((acc, val) => acc | val, 0);
}
```

**注册流程要点**：
1. 权限字符串数组转换为位掩码整数
2. 客户端 secret 使用 JWT 签名生成
3. 客户端必须关联到特定的 `organizationId`

---

## 第三部分：权限范围 (Scopes/Permissions) 机制

### 3.1 权限常量定义

权限系统采用**位运算掩码**方式，定义在 `packages/platform/constants/permissions.ts`：

```typescript
// 单个权限值（2 的幂次）
export const EVENT_TYPE_READ = 1;   // 2^0 = 0000000001
export const EVENT_TYPE_WRITE = 2;  // 2^1 = 0000000010
export const BOOKING_READ = 4;      // 2^2 = 0000000100
export const BOOKING_WRITE = 8;     // 2^3 = 0000001000
export const SCHEDULE_READ = 16;    // 2^4 = 0000010000
export const SCHEDULE_WRITE = 32;   // 2^5 = 0000100000
export const APPS_READ = 64;        // 2^6 = 0001000000
export const APPS_WRITE = 128;      // 2^7 = 0010000000
export const PROFILE_READ = 256;    // 2^8 = 0100000000
export const PROFILE_WRITE = 512;   // 2^9 = 1000000000

// 权限映射表
export const PERMISSION_MAP = {
  EVENT_TYPE_READ,
  EVENT_TYPE_WRITE,
  BOOKING_READ,
  BOOKING_WRITE,
  SCHEDULE_READ,
  SCHEDULE_WRITE,
  APPS_READ,
  APPS_WRITE,
  PROFILE_READ,
  PROFILE_WRITE,
} as const;

// 权限分组（按资源类型）
export const PERMISSIONS_GROUPED_MAP = {
  EVENT_TYPE: {
    read: EVENT_TYPE_READ,
    write: EVENT_TYPE_WRITE,
    key: "eventType",
    label: "Event Type",
  },
  BOOKING: {
    read: BOOKING_READ,
    write: BOOKING_WRITE,
    key: "booking",
    label: "Booking",
  },
  SCHEDULE: {
    read: SCHEDULE_READ,
    write: SCHEDULE_WRITE,
    key: "schedule",
    label: "Schedule",
  },
  APPS: {
    read: APPS_READ,
    write: APPS_WRITE,
    key: "apps",
    label: "Apps",
  },
  PROFILE: {
    read: PROFILE_READ,
    write: PROFILE_WRITE,
    key: "profile",
    label: "Profile",
  },
} as const;
```

### 3.2 权限位运算原理

位运算权限系统的核心原理：

| 权限 | 位值 | 二进制表示 |
|------|------|------------|
| EVENT_TYPE_READ | 1 | 0000000001 |
| EVENT_TYPE_WRITE | 2 | 0000000010 |
| BOOKING_READ | 4 | 0000000100 |
| BOOKING_WRITE | 8 | 0000001000 |
| SCHEDULE_READ | 16 | 0000010000 |
| SCHEDULE_WRITE | 32 | 0000100000 |
| APPS_READ | 64 | 0001000000 |
| APPS_WRITE | 128 | 0010000000 |
| PROFILE_READ | 256 | 0100000000 |
| PROFILE_WRITE | 512 | 1000000000 |

**组合权限示例**：
- 拥有 `BOOKING_READ` (4) + `BOOKING_WRITE` (8) = `12` (二进制 `0000001100`)
- 拥有所有读权限 = `1 + 4 + 16 + 64 + 256` = `341`

### 3.3 权限检查工具函数

权限检查函数定义在 `packages/platform/utils/permissions.ts`：

```typescript
// 检查是否有单个权限
export const hasPermission = (userPermissions: number, permission: PLATFORM_PERMISSION): boolean => {
  // 使用位与运算检查
  // 如果 (userPermissions & permission) === permission，说明拥有该权限
  return (userPermissions & permission) === permission;
};

// 检查是否有所有要求的权限
export const hasPermissions = (userPermissions: number, permissions: PLATFORM_PERMISSION[]): boolean => {
  return permissions.every((permission) => hasPermission(userPermissions, permission));
};

// 列出用户拥有的所有权限
export const listPermissions = (userPermissions: number): PLATFORM_PERMISSION[] => {
  return PERMISSIONS.reduce((acc, permission) => {
    if (hasPermission(userPermissions, permission)) {
      return [...acc, permission];
    }
    return acc;
  }, [] as PLATFORM_PERMISSION[]);
};

// 资源特定的权限检查函数
export const hasBookingReadPermission = (userPermissions: number): boolean => {
  return hasPermission(userPermissions, BOOKING_READ);
};

export const hasBookingWritePermission = (userPermissions: number): boolean => {
  return hasPermission(userPermissions, BOOKING_WRITE);
};
// ... 其他资源类型类似
```

### 3.4 权限守卫 (PermissionsGuard)

权限守卫实现位于 `apps/api/v2/src/modules/auth/guards/permissions/permissions.guard.ts`：

```typescript
@Injectable()
export class PermissionsGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 1. 从装饰器获取要求的权限
    const requiredPermissions = this.reflector.get(Permissions, context.getHandler());
    
    // 如果没有要求权限，直接通过
    if (!requiredPermissions?.length || !Object.keys(requiredPermissions)?.length) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const bearerToken = request.get("Authorization")?.replace("Bearer ", "");
    
    // 2. 判断认证方式
    const nextAuthToken = await getToken({ req: request, secret: nextAuthSecret });
    const apiKey = bearerToken && isApiKey(bearerToken, this.config.get("api.apiKeyPrefix") ?? "cal_");
    const isThirdPartyBearerToken = bearerToken && this.getDecodedThirdPartyAccessToken(bearerToken);

    // 3. 以下认证方式跳过权限检查：
    // - NextAuth Token（会话认证）
    // - API Key（全权代表用户）
    // - Third-Party Token（外部 JWT）
    if (nextAuthToken || apiKey || isThirdPartyBearerToken) {
      return true;
    }

    // 4. OAuth Access Token 或 OAuth Client ID 需要检查权限
    if (!bearerToken && !oAuthClientId) {
      throw new ForbiddenException("No authentication provided");
    }

    // 5. 获取 OAuth 客户端的权限
    const oAuthClient = bearerToken
      ? await this.getOAuthClientByAccessToken(bearerToken)
      : await this.getOAuthClientById(oAuthClientId);

    // 6. 检查权限
    const hasRequiredPermissions = hasPermissions(oAuthClient.permissions, [...requiredPermissions]);

    if (!hasRequiredPermissions) {
      throw new ForbiddenException(
        `OAuth client does not have the required permissions. Go to platform dashboard settings and add the required permissions.`
      );
    }

    return true;
  }
}
```

**权限守卫设计要点**：
- **API Key** 认证跳过权限检查（因为 API Key 直接代表用户）
- **OAuth Access Token** 需要检查 OAuth 客户端的权限
- 权限检查基于位运算，高效且存储紧凑

---

## 第四部分：认证方式与权限映射

### 4.1 支持的认证方式

系统支持 5 种认证方式，定义在 `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts`：

| 认证方式 | 标识 | 权限检查 | 说明 |
|---------|------|---------|------|
| OAuth Client Credentials | `OAUTH_CLIENT` | 需要 | 使用 clientId + secret |
| API Key | `API_KEY` | 跳过 | 使用 Bearer 头 |
| Access Token | `ACCESS_TOKEN` | 需要 | OAuth 2.0 访问令牌 |
| Third-Party Access Token | `THIRD_PARTY_ACCESS_TOKEN` | 跳过 | 外部 JWT 令牌 |
| NextAuth Token | `NEXT_AUTH` | 跳过 | NextAuth.js 会话 |

### 4.2 认证方式详细说明

#### 1. OAuth Client Credentials 认证

**认证方式**：
- 请求头：`X-Cal-Client-Id: <clientId>`
- 请求头：`X-Cal-Secret-Key: <clientSecret>`

**认证流程**：
```typescript
async oAuthClientStrategy(oAuthClientId: string, oAuthClientSecret: string, request: ApiAuthGuardRequest) {
  // 1. 查找客户端
  const client = await this.oauthRepository.getOAuthClient(oAuthClientId);
  if (!client) {
    throw new UnauthorizedException("Client not found");
  }

  // 2. 验证 secret
  if (client.secret !== oAuthClientSecret) {
    throw new UnauthorizedException("Invalid client secret");
  }

  // 3. 获取平台所有者/管理员用户
  const platformCreatorId = 
    (await this.membershipsRepository.findPlatformOwnerUserId(client.organizationId)) ||
    (await this.membershipsRepository.findPlatformAdminUserId(client.organizationId));

  // 4. 获取用户信息
  const user = await this.userRepository.findByIdWithProfile(platformCreatorId);
  
  // 5. 设置组织 ID
  request.organizationId = client.organizationId;

  return user;
}
```

#### 2. Access Token 认证

**认证方式**：
- 请求头：`Authorization: Bearer <accessToken>`

**认证流程**：
```typescript
async accessTokenStrategy(accessToken: string, request: ApiAuthGuardRequest, origin?: string) {
  // 1. 验证 token 有效性
  const accessTokenValid = await this.oauthFlowService.validateAccessToken(accessToken);
  if (!accessTokenValid) {
    throw new UnauthorizedException(INVALID_ACCESS_TOKEN);
  }

  // 2. 获取关联的 OAuth 客户端
  const client = await this.tokensRepository.getAccessTokenClient(accessToken);
  if (!client) {
    throw new UnauthorizedException("OAuth client not found");
  }

  // 3. 验证请求来源（CORS 检查）
  if (origin && !isOriginAllowed(origin, client.redirectUris)) {
    throw new UnauthorizedException("Invalid request origin");
  }

  // 4. 获取 token 所有者用户
  const ownerId = await this.tokensRepository.getAccessTokenOwnerId(accessToken);
  const user = await this.userRepository.findByIdWithProfile(ownerId);
  
  // 5. 设置组织 ID
  const organizationId = this.usersService.getUserMainOrgId(user) as number;
  request.organizationId = organizationId;

  return user;
}
```

#### 3. Third-Party Access Token 认证

**认证方式**：
- 请求头：`Authorization: Bearer <jwtToken>`

**Token 结构** (`apps/api/v2/src/modules/tokens/tokens.service.ts`)：
```typescript
type OAuthTokenPayload = {
  userId?: number;
  teamId?: number;
  scope: string[];
  token_type: string;
};
```

**认证流程**：
```typescript
async validateThirdPartyAccessToken(token: string, request: ApiAuthGuardRequest) {
  // 1. 解码并验证 JWT
  const decodedToken = this.tokensService.getDecodedThirdPartyAccessToken(token);
  if (!decodedToken) {
    return { success: false };
  }

  // 2. 根据 userId 或 teamId 获取用户
  let user: UserWithProfile | null = null;
  let organizationId: number | null = null;

  if (decodedToken.userId) {
    // 优先使用 userId
    user = await this.userRepository.findByIdWithProfile(decodedToken.userId);
    if (user) {
      organizationId = this.usersService.getUserMainOrgId(user) as number;
    }
  } else if (decodedToken.teamId) {
    // 其次使用 teamId，获取团队所有者
    const teamOwner = await this.userRepository.findOwnerByTeamIdWithProfile(decodedToken.teamId);
    user = teamOwner;
    organizationId = teamOwner.profiles?.find((p) => p.organizationId === decodedToken.teamId)?.organizationId ?? null;
  }

  // 3. 设置组织 ID
  request.organizationId = organizationId;
  return { success: true, data: user };
}
```

### 4.3 认证方式与权限检查关系

```
┌─────────────────────────────────────────────────────────────────┐
│                      认证请求                                      │
└─────────────────────────┬───────────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┬───────────────┐
          ▼               ▼               ▼               ▼
   ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
   │ API Key    │  │ NextAuth   │  │ Third-Party│  │ OAuth      │
   │            │  │ Token      │  │ Token      │  │ (Client/   │
   └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  │ Access    │
         │                │                │          │ Token)    │
         ▼                ▼                ▼          └─────┬────┘
   ┌──────────────────────────────────────────┐              │
   │         跳过权限检查                        │              │
   │  (直接代表用户，拥有用户所有权限)             │              │
   └──────────────────────────────────────────┘              ▼
                                                     ┌──────────────┐
                                                     │ 检查权限守卫  │
                                                     │ (Permissions  │
                                                     │  Guard)       │
                                                     └──────┬───────┘
                                                            ▼
                                               ┌──────────────────────┐
                                               │ 检查 OAuth 客户端的   │
                                               │ permissions 位掩码    │
                                               │ 是否满足接口要求的权限 │
                                               └──────────────────────┘
```

---

## 第五部分：数据关系与架构

### 5.1 实体关系图

```
┌──────────────┐       ┌──────────────────────┐
│    User      │       │   Team (Organization)│
│              │       │                      │
│ id: Int      │       │ id: Int              │
│ ...          │       │ ...                  │
└──────┬───────┘       └──────────┬───────────┘
       │                            │
       │ 1:N                        │ 1:N
       ▼                            ▼
┌──────────────┐           ┌──────────────────────┐
│   ApiKey     │           │  PlatformOAuthClient │
│              │           │                      │
│ id: String   │           │ id: String           │
│ userId: Int  │◄──────────│ organizationId: Int  │
│ teamId: Int? │           │ secret: String       │
│ hashedKey:   │           │ permissions: Int (位掩码)
│ String       │           │ redirectUris: String[]
│ expiresAt:   │           └──────────┬───────────┘
│ DateTime?    │                      │ 1:N
└──────────────┘                      ▼
                            ┌──────────────────────┐
                            │    AccessToken       │
                            │                      │
                            │ id: Int              │
                            │ secret: String       │
                            │ expiresAt: DateTime  │
                            │ userId: Int          │
                            │ platformOAuthClientId│
┌─────────────────────────────────────────────────┐
│  Third-Party JWT Token (外部系统生成)             │
│                                                   │
│  结构:                                            │
│  {                                                │
│    userId?: number,    // 用户 ID                │
│    teamId?: number,    // 团队/组织 ID           │
│    scope: string[],     // 权限范围字符串数组     │
│    token_type: string   // "Access Token"       │
│  }                                                │
└─────────────────────────────────────────────────┘
```

### 5.2 关键文件位置汇总

#### API Key 相关
| 文件 | 路径 | 功能 |
|------|------|------|
| Repository | `apps/api/v2/src/modules/api-keys/api-keys-repository.ts` | 数据库操作 |
| Service | `apps/api/v2/src/modules/api-keys/services/api-keys.service.ts` | 业务逻辑 |
| Controller | `apps/api/v2/src/modules/api-keys/controllers/api-keys.controller.ts` | API 端点 |

#### OAuth Client 相关
| 文件 | 路径 | 功能 |
|------|------|------|
| Repository | `apps/api/v2/src/modules/oauth-clients/oauth-client.repository.ts` | 数据库操作 |
| Main Service | `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients.service.ts` | 核心业务逻辑 |
| Input Service | `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients-input.service.ts` | 输入转换（权限位掩码） |
| Output Service | `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients-output.service.ts` | 输出转换（权限解码） |
| Controller | `apps/api/v2/src/modules/oauth-clients/controllers/oauth-clients/oauth-clients.controller.ts` | API 端点 |

#### 权限系统相关
| 文件 | 路径 | 功能 |
|------|------|------|
| 权限常量 | `packages/platform/constants/permissions.ts` | 权限值定义、映射表 |
| 权限工具 | `packages/platform/utils/permissions.ts` | 权限检查函数 |
| 类型定义 | `packages/platform/types/oauth-clients/` | DTO 类型 |

#### 认证授权相关
| 文件 | 路径 | 功能 |
|------|------|------|
| 认证策略 | `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts` | 5 种认证方式实现 |
| API Auth Guard | `apps/api/v2/src/modules/auth/guards/api-auth/api-auth.guard.ts` | 认证守卫 |
| Permissions Guard | `apps/api/v2/src/modules/auth/guards/permissions/permissions.guard.ts` | 权限守卫 |
| OAuth2 Controller | `apps/api/v2/src/modules/auth/oauth2/controllers/oauth2.controller.ts` | OAuth 2.0 令牌端点 |

---

## 第六部分：权限范围与资源映射

### 6.1 权限与资源对应关系

| 权限组 | 读权限 | 写权限 | 资源说明 |
|--------|--------|--------|---------|
| **EVENT_TYPE** | `EVENT_TYPE_READ` (1) | `EVENT_TYPE_WRITE` (2) | 事件类型（预约模板） |
| **BOOKING** | `BOOKING_READ` (4) | `BOOKING_WRITE` (8) | 预约记录 |
| **SCHEDULE** | `SCHEDULE_READ` (16) | `SCHEDULE_WRITE` (32) | 日程安排 |
| **APPS** | `APPS_READ` (64) | `APPS_WRITE` (128) | 应用集成 |
| **PROFILE** | `PROFILE_READ` (256) | `PROFILE_WRITE` (512) | 用户资料 |

### 6.2 权限组合示例

**示例 1：只读预约权限**
```typescript
const permissions = ["BOOKING_READ"];
// 位掩码值: 4 (二进制: 0000000100)
```

**示例 2：预约读写权限**
```typescript
const permissions = ["BOOKING_READ", "BOOKING_WRITE"];
// 位掩码值: 4 | 8 = 12 (二进制: 0000001100)
```

**示例 3：全部权限**
```typescript
const permissions = ["*"];
// 位掩码值: 1 | 2 | 4 | 8 | 16 | 32 | 64 | 128 | 256 | 512 = 1023
```

### 6.3 权限检查示例

```typescript
// 假设 OAuth 客户端拥有 BOOKING_READ 和 BOOKING_WRITE 权限
const clientPermissions = 12; // 4 + 8

// 检查是否有 BOOKING_READ 权限
hasPermission(clientPermissions, BOOKING_READ);  // true (12 & 4 = 4)

// 检查是否有 EVENT_TYPE_READ 权限
hasPermission(clientPermissions, EVENT_TYPE_READ);  // false (12 & 1 = 0)

// 检查是否有多个权限
hasPermissions(clientPermissions, [BOOKING_READ, BOOKING_WRITE]);  // true
hasPermissions(clientPermissions, [BOOKING_READ, EVENT_TYPE_READ]);  // false
```

---

## 第七部分：安全设计与最佳实践

### 7.1 API Key 安全设计

1. **哈希存储**：API Key 仅存储 SHA256 哈希值，不存储明文
   - 即使数据库泄露，攻击者无法获取原始密钥

2. **过期机制**：支持设置过期时间，默认 30 天
   - 降低长期有效密钥泄露的风险

3. **前缀识别**：使用前缀（如 `cal_`）便于识别
   - 方便日志过滤和安全扫描

### 7.2 OAuth 安全设计

1. **Client Secret 安全**：使用 JWT 签名生成
   - 包含客户端信息，可验证完整性

2. **Redirect URI 验证**：严格验证回调地址
   - 防止授权码被劫持

3. **Origin 检查**：Access Token 认证时验证请求来源
   - 防止 CSRF 攻击

### 7.3 权限系统设计优点

1. **存储高效**：使用位掩码，一个整数存储多个权限
   - 数据库仅需一个 `Int` 字段

2. **检查高效**：位运算操作，时间复杂度 O(1)
   - 无需额外的数据库查询

3. **扩展性好**：新增权限只需添加新的 2 的幂次值
   - 现有权限检查逻辑无需修改

4. **组合灵活**：可以任意组合权限
   - 使用 `|` 运算符组合，使用 `&` 检查

---

## 第八部分：总结

### 8.1 核心机制总结

| 维度 | 机制 |
|------|------|
| **API Key 管理** | 哈希存储、过期时间、用户/团队关联 |
| **OAuth Client 管理** | 组织绑定、权限位掩码、JWT Secret |
| **权限表示** | 位运算掩码，2 的幂次值 |
| **权限检查** | `(userPermissions & requiredPermission) === requiredPermission` |
| **认证方式** | 5 种：API Key、OAuth Client、Access Token、Third-Party JWT、NextAuth |
| **权限守卫** | 仅 OAuth 方式需要检查权限，其他方式跳过 |

### 8.2 权限范围映射原则

1. **API Key**：直接代表用户，拥有用户的所有权限
   - 适用于服务器到服务器的集成

2. **OAuth Client**：权限由 `permissions` 位掩码控制
   - 适用于第三方应用集成，遵循最小权限原则

3. **Third-Party JWT**：由外部系统签发，包含 `scope` 字段
   - 适用于跨平台集成，信任外部系统的权限判断

### 8.3 组织与资源访问

- **organizationId**：通过以下方式确定
  - API Key: 来自 `teamId` 字段
  - OAuth Client: 来自 `organizationId` 字段
  - Access Token: 来自关联用户的主组织
  - Third-Party JWT: 来自 `teamId` 或用户主组织

- **资源访问控制**：结合 `organizationId` 和权限
  - 用户只能访问所属组织的资源
  - 权限控制对资源的操作类型（读/写）

---

## 附录：代码引用索引

### 数据库模型
- **ApiKey**: `packages/prisma/schema.prisma:1172`
- **PlatformOAuthClient**: `packages/prisma/schema.prisma:1716`
- **AccessToken**: `packages/prisma/schema.prisma:1757`
- **RefreshToken**: `packages/prisma/schema.prisma:1771`

### 权限常量
- **权限定义**: `packages/platform/constants/permissions.ts:1-69`
- **权限检查**: `packages/platform/utils/permissions.ts:16-24`

### 认证策略
- **API Key 认证**: `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts:236-259`
- **OAuth Client 认证**: `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts:166-209`
- **Access Token 认证**: `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts:261-299`
- **Third-Party Token 认证**: `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts:320-357`

### 权限守卫
- **PermissionsGuard**: `apps/api/v2/src/modules/auth/guards/permissions/permissions.guard.ts:27-72`

### 权限转换
- **权限编码**: `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients-input.service.ts:23-28`
- **权限解码**: `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients-output.service.ts:32-43`

---

**文档生成日期**: 2026-05-04
**分析范围**: Cal.diy API v2 模块
