# Cal.diy OAuth Client 与 API Key 权限范围差异分析报告

## 文档说明

本文档详细分析了 Cal.diy 平台中 OAuth Client 体系的完整实现，包括：
- OAuth Client 注册与审批流程
- 密钥保存策略
- 授权码与刷新 Token 的发放链路
- Scope 与 User/Team 身份及资源访问的映射关系
- 与 Platform API Key / Permissions 位掩码的差异对比

---

## 第一部分：OAuth Client 注册与审批流程

### 1.1 当前实现分析

#### 注册流程概述

OAuth Client 的注册实现位于 `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients.service.ts`:

```typescript
async createOAuthClient(organizationId: number, input: CreateOAuthClientInput) {
  const transformedInput = this.oauthClientsInputService.transformCreateOAuthClientInput(input);

  const { id, secret } = await this.oauthClientRepository.createOAuthClient(
    organizationId,
    transformedInput
  );

  return {
    clientId: id,
    clientSecret: secret,
  };
}
```

#### 输入转换 (权限处理)

位于 `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients-input.service.ts`:

```typescript
transformCreateOAuthClientInput(input: CreateOAuthClientInput) {
  const transformed = {
    ...input,
    permissions: this.transformPermissions(input.permissions),
  };

  return {
    ...transformed,
    secret: this.jwtService.sign(transformed),  // JWT 签名生成 secret
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

#### 注册流程步骤

```
┌─────────────────────────────────────────────────────────────────┐
│                    OAuth Client 注册流程                          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  1. 接收创建请求        │
              │  - name: 客户端名称     │
              │  - redirectUris: 回调URI│
              │  - permissions: 权限数组 │
              │  - organizationId: 组织ID│
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  2. 权限转换           │
              │  字符串数组 → 位掩码整数 │
              │  例如: ["BOOKING_READ",│
              │        "BOOKING_WRITE"]│
              │       → 4 | 8 = 12     │
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  3. 生成 Client Secret │
              │  JWT 签名:              │
              │  jwtService.sign(        │
              │    {                  │
              │      ...input,        │
              │      permissions: 12 │
              │    }                  │
              │  )                     │
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  4. 数据库持久化         │
              │  PlatformOAuthClient  │
              │  - id: cuid()        │
              │  - name: 客户端名称     │
              │  - secret: JWT签名    │
              │  - permissions: 位掩码  │
              │  - organizationId: 组织 │
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  5. 返回客户端凭证          │
              │  {                     │
              │    clientId: "id",   │
              │    clientSecret: "jwt"│
              │  }                     │
              └───────────────────────┘
```

### 1.2 审批流程分析

**关键发现：当前实现中**

1. **无审批流程**：OAuth Client 的创建是直接创建，无需审批。

2. **组织绑定**：
   - OAuth Client 必须绑定到特定的 `organizationId`
   - 创建时需要提供 `organizationId` 是必填参数

3. **权限设置**：
   - 创建时直接指定 `permissions` 数组
   - 支持 `"*"` 通配符授予所有权限

**潜在问题**：
- 没有审批机制缺失，可能导致权限过度授予风险
- `"*"` 通配符过于宽松，不符合最小权限原则

---

## 第二部分：密钥保存策略

### 2.1 Client Secret 生成与存储

#### 生成策略

位于 `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients-input.service.ts`:

```typescript
transformCreateOAuthClientInput(input: CreateOAuthClientInput) {
  const transformed = {
    ...input,
    permissions: this.transformPermissions(input.permissions),
  };

  return {
    ...transformed,
    secret: this.jwtService.sign(transformed),  // JWT 签名
  };
}
```

#### JWT Service 实现 (`apps/api/v2/src/modules/jwt/jwt.service.ts`:

```typescript
@Injectable()
export class JwtService {
  constructor(private readonly nestJwtService: NestJwtService) {}

  sign(payload: Payload) {
    const issuedAtTime = this.getIssuedAtTime();
    const token = this.nestJwtService.sign({ ...payload, iat: issuedAtTime });
    return token;
  }

  getIssuedAtTime() {
    return Math.floor(Date.now() / 1000);  // 秒级时间戳
  }

  decode(token: string): Payload {
    return this.nestJwtService.decode(token) as Payload;
  }
}
```

#### 存储结构

数据库模型 `packages/prisma/schema.prisma:1716`:

```prisma
model PlatformOAuthClient {
  id             String   @id @default(cuid())
  name           String
  secret         String    // JWT 格式的 secret
  permissions    Int       // 权限位掩码
  organizationId Int
  redirectUris   String[]
  logo           String?
  // ... 其他字段
}
```

### 2.2 Access Token 与 Refresh Token 生成策略

#### Token 生成

位于 `apps/api/v2/src/modules/tokens/tokens.repository.ts`:

```typescript
async createOAuthTokens(clientId: string, ownerId: number) {
  const accessExpiry = DateTime.now().plus({ minute: 60 }).startOf("minute").toJSDate();
  const refreshExpiry = DateTime.now().plus({ year: 1 }).startOf("day").toJSDate();
  
  const [accessToken, refreshToken] = await this.dbWrite.prisma.$transaction([
    this.dbWrite.prisma.accessToken.create({
      data: {
        secret: this.jwtService.signAccessToken({
          clientId,
          ownerId,
          expiresAt: accessExpiry.valueOf(),
          jti: uuidv4(),
        }),
        expiresAt: accessExpiry,
        client: { connect: { id: clientId } },
        owner: { connect: { id: ownerId } },
      },
    }),
    this.dbWrite.prisma.refreshToken.create({
      data: {
        secret: this.jwtService.signRefreshToken({
          clientId,
          ownerId,
          expiresAt: refreshExpiry.valueOf(),
          jti: uuidv4(),
        }),
        expiresAt: refreshExpiry,
        client: { connect: { id: clientId } },
        owner: { connect: { id: ownerId } },
      },
    }),
  ]);

  return {
    accessToken: accessToken.secret,
    accessTokenExpiresAt: accessToken.expiresAt,
    refreshToken: refreshToken.secret,
    refreshTokenExpiresAt: refreshToken.expiresAt,
  };
}
```

#### JWT 签名方法

```typescript
signAccessToken(payload: Payload) {
  return this.sign({ type: "access_token", ...payload });
}

signRefreshToken(payload: Payload) {
  return this.sign({ type: "refresh_token", ...payload });
}
```

### 2.3 Token 有效期配置

| Token 类型 | 有效期 | 说明 |
|-----------|--------|------|
| Access Token | 60 分钟 | 短期有效 |
| Refresh Token | 1 年 | 长期有效 |
| Authorization Code | 单次使用 | 交换后立即失效 |

### 2.4 安全策略对比

| 项目 | OAuth Client Secret | Access/Refresh Token | API Key |
|------|---------------------|----------------------|---------|
| **生成方式** | JWT 签名 | JWT 签名 | 随机字符串 |
| **存储方式** | 明文 JWT | 明文 JWT | SHA256 哈希 |
| **验证方式** | 字符串比对 | JWT 验证 | 哈希比对 |
| **包含信息** | 客户端配置信息 | clientId, ownerId, jti | 无 |
| **过期时间** | 永不过期 | Access: 60分钟<br>Refresh: 1年 | 可配置 |

---

## 第三部分：授权码与刷新 Token 发放链路

### 3.1 授权码流程 (Authorization Code Flow)

#### 完整流程架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OAuth 2.0 授权码完整流程                                │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌──────────┐                         ┌──────────────────┐
  │  用户浏览器 │                         │  OAuth Client    │
  │ (Resource│                         │  (Third-Party │
  │  Owner)  │                         │    App         │
  └─────┬──────┘                         └────────┬─────────┘
        │                                            │
        │ 1. 用户访问应用                          │
        ▼                                            │
  ┌──────────────────┐                         │
  │  OAuth Flow    │                         │
  │  Authorize     │◄────────────────────────┘
  │  Endpoint      │  2. 重定向到授权页面
  └────────┬─────────┘
           │
           │ 3. 用户登录并授权
           ▼
  ┌──────────────────┐
  │  NextAuth Guard   │
  │  验证用户身份    │
  └────────┬─────────┘
           │
           │ 4. 创建授权码
           ▼
  ┌──────────────────┐
  │  PlatformAuthorizationToken│
  │  (Authorization  │
  │   Code Storage)  │
  └────────┬─────────┘
           │
           │ 5. 重定向回客户端
           ▼
  ┌──────────────────┐
  │  OAuth Client    │
  │  (Third-Party    │
  │    App)           │
  └────────┬─────────┘
           │
           │ 6. 用 code 交换 token
           ▼
  ┌──────────────────┐
  │  Exchange        │
  │  Endpoint        │
  └────────┬─────────┘
           │
           │ 7. 验证授权码，生成 Token
           ▼
  ┌─────────────────────────────────────┐
  │  创建 Access Token & Refresh Token │
  │  - AccessToken 表               │
  │  - RefreshToken 表                │
  └─────────────────────────────────────┘
           │
           ▼
  ┌──────────────────┐
  │  返回 Token 对  │
  │  - access_token │
  │  - refresh_token│
  │  - expires_in   │
  └──────────────────┘
```

#### 授权端点实现

位于 `apps/api/v2/src/modules/oauth-clients/controllers/oauth-flow/oauth-flow.controller.ts`:

```typescript
@Post("/authorize")
@HttpCode(HttpStatus.OK)
@UseGuards(NextAuthGuard)  // 需要用户已登录
async authorize(
  @Param("clientId") clientId: string,
  @Body() body: OAuthAuthorizeInput,
  @GetUser("id") userId: number,
  @Response() res: ExpressResponse
): Promise<void> {
  // 1. 验证 OAuth Client 是否存在
  const oauthClient = await this.oauthClientRepository.getOAuthClient(clientId);
  if (!oauthClient) {
    throw new BadRequestException(`OAuth client with ID '${clientId}' not found`);
  }

  // 2. 验证 redirectUri 是否在白名单中
  if (!isOriginAllowed(body.redirectUri, oauthClient.redirectUris)) {
    throw new BadRequestException("Invalid 'redirect_uri' value.");
  }

  // 3. 检查用户是否已授权过该客户端
  const alreadyAuthorized = await this.tokensRepository.getAuthorizationTokenByClientUserIds(
    clientId,
    userId
  );

  if (alreadyAuthorized) {
    throw new BadRequestException(
      `User with id=${userId} has already authorized client with id=${clientId}.`
    );
  }

  // 4. 创建授权码 (PlatformAuthorizationToken)
  const { id } = await this.tokensRepository.createAuthorizationToken(clientId, userId);

  // 5. 重定向回客户端，附带 code
  return res.redirect(`${body.redirectUri}?code=${id}`);
}
```

#### 授权码存储

```typescript
async createAuthorizationToken(clientId: string, userId: number): Promise<PlatformAuthorizationToken> {
  return this.dbWrite.prisma.platformAuthorizationToken.create({
    data: {
      client: {
        connect: {
          id: clientId,
        },
      },
      owner: {
        connect: {
          id: userId,
        },
      },
    },
  });
}
```

#### 授权码交换 Token

```typescript
@Post("/exchange")
@HttpCode(HttpStatus.OK)
async exchange(
  @Headers("Authorization") authorization: string,
  @Param("clientId") clientId: string,
  @Body() body: ExchangeAuthorizationCodeInput
): Promise<KeysResponseDto> {
  // 1. 从 Authorization 头提取 code
  const authorizeEndpointCode = authorization.replace("Bearer ", "").trim();
  if (!authorizeEndpointCode) {
    throw new BadRequestException("Missing 'Bearer' Authorization header.");
  }

  // 2. 交换授权码为 Token
  const tokens = await this.oAuthFlowService.exchangeAuthorizationToken(
    authorizeEndpointCode,
    clientId,
    body.clientSecret
  );

  return {
    status: SUCCESS_STATUS,
    data: tokens,
  };
}
```

#### 授权码交换实现

位于 `apps/api/v2/src/modules/oauth-clients/services/oauth-flow.service.ts`:

```typescript
async exchangeAuthorizationToken(
  tokenId: string,
  clientId: string,
  clientSecret: string
): Promise<KeysDto> {
  // 1. 验证 OAuth Client 和授权码
  const oauthClient = await this.oAuthClientRepository.getOAuthClientWithAuthTokens(
    tokenId,
    clientId,
    clientSecret
  );

  if (!oauthClient) {
    throw new BadRequestException("Invalid OAuth Client.");
  }

  const authorizationToken = oauthClient.authorizationTokens[0];

  if (!authorizationToken || !authorizationToken.owner.id) {
    throw new BadRequestException("Invalid Authorization Token.");
  }

  // 2. 创建 Access Token 和 Refresh Token
  const { accessToken, refreshToken, accessTokenExpiresAt, refreshTokenExpiresAt } =
    await this.tokensRepository.createOAuthTokens(clientId, authorizationToken.owner.id);
  
  // 3. 使授权码失效 (一次性使用)
  await this.tokensRepository.invalidateAuthorizationToken(authorizationToken.id);
  
  // 4. 将 Access Token 传播到 Redis 缓存
  void this.propagateAccessToken(accessToken);

  return {
    accessToken,
    accessTokenExpiresAt: accessTokenExpiresAt.valueOf(),
    refreshToken,
    refreshTokenExpiresAt: refreshTokenExpiresAt.valueOf(),
  };
}
```

### 3.2 刷新 Token 流程

#### 刷新 Token 端点

```typescript
@Post("/refresh")
@HttpCode(HttpStatus.OK)
@UseGuards(ApiAuthGuard)
async refreshTokens(
  @Param("clientId") clientId: string,
  @Headers(X_CAL_SECRET_KEY) secretKey: string,
  @Body() body: RefreshTokenInput
): Promise<KeysResponseDto> {
  const tokens = await this.oAuthFlowService.refreshToken(clientId, secretKey, body.refreshToken);

  return {
    status: SUCCESS_STATUS,
    data: tokens,
  };
}
```

#### 刷新 Token 实现

```typescript
async refreshToken(clientId: string, clientSecret: string, tokenSecret: string): Promise<KeysDto> {
  // 1. 验证 OAuth Client 和 Refresh Token
  const oauthClient = await this.oAuthClientRepository.getOAuthClientWithRefreshSecret(
    clientId,
    clientSecret,
    tokenSecret
  );

  if (!oauthClient) {
    throw new BadRequestException("Invalid OAuthClient credentials.");
  }

  const currentRefreshToken = oauthClient.refreshToken[0];

  if (!currentRefreshToken) {
    throw new BadRequestException("Invalid refresh token");
  }

  // 2. 刷新 Token (刷新令牌轮换机制)
  const { accessToken, refreshToken } = await this.tokensRepository.refreshOAuthTokens(
    clientId,
    currentRefreshToken.secret,
    currentRefreshToken.userId
  );

  return {
    accessToken: accessToken.secret,
    accessTokenExpiresAt: accessToken.expiresAt.valueOf(),
    refreshToken: refreshToken.secret,
    refreshTokenExpiresAt: refreshToken.expiresAt.valueOf(),
  };
}
```

#### Token 刷新数据库操作

```typescript
async refreshOAuthTokens(clientId: string, refreshTokenSecret: string, tokenUserId: number) {
  const accessExpiry = DateTime.now().plus({ minute: 60 }).startOf("minute").toJSDate();
  const refreshExpiry = DateTime.now().plus({ year: 1 }).startOf("day").toJSDate();

  // 使用事务确保原子性
  const [_, _refresh, accessToken, refreshToken] = await this.dbWrite.prisma.$transaction([
    // 1. 删除过期的 Access Token
    this.dbWrite.prisma.accessToken.deleteMany({
      where: { client: { id: clientId }, expiresAt: { lte: new Date() } },
    }),
    // 2. 删除当前 Refresh Token (刷新令牌轮换)
    this.dbWrite.prisma.refreshToken.delete({ where: { secret: refreshTokenSecret } }),
    // 3. 创建新的 Access Token
    this.dbWrite.prisma.accessToken.create({
      data: {
        secret: this.jwtService.signAccessToken({
          clientId,
          ownerId: tokenUserId,
          expiresAt: accessExpiry.valueOf(),
          userId: tokenUserId,
          jti: uuidv4(),
        }),
        expiresAt: accessExpiry,
        client: { connect: { id: clientId } },
        owner: { connect: { id: tokenUserId } },
      },
    }),
    // 4. 创建新的 Refresh Token
    this.dbWrite.prisma.refreshToken.create({
      data: {
        secret: this.jwtService.signRefreshToken({
          clientId,
          ownerId: tokenUserId,
          expiresAt: refreshExpiry.valueOf(),
          userId: tokenUserId,
          jti: uuidv4(),
        }),
        expiresAt: refreshExpiry,
        client: { connect: { id: clientId } },
        owner: { connect: { id: tokenUserId } },
      },
    }),
  ]);
  return { accessToken, refreshToken };
}
```

### 3.3 刷新令牌轮换机制 (Refresh Token Rotation)

**关键特性**：
1. **一次性使用**：每次刷新后，旧的 Refresh Token 立即被删除
2. **新 Token 对**：每次刷新都会生成新的 Access Token 和 Refresh Token
3. **安全性提升**：防止 Token 泄露风险降低

```
┌─────────────────────────────────────────────────────────────────┐
│                    刷新令牌轮换机制                              │
└─────────────────────────────────────────────────────────────────┘

  初始状态:
  ┌─────────────┐    ┌─────────────────────────────────────────────┐
  │ AccessToken   │    │ RefreshToken                            │
  │ secret: AT1 │    │ secret: RT1                            │
  │ expires: +60m│    │ expires: +1y                           │
  └─────────────┘    └─────────────────────────────────────────────┘

  刷新请求 (使用 RT1):
  ┌─────────────────────────────────────────────────────────────────┐
  │  POST /oauth/{clientId}/refresh                               │
  │  Headers: X-Cal-Secret-Key: <clientSecret>                  │
  │  Body: { refreshToken: "RT1" }                               │
  └─────────────────────────────────────────────────────────────────┘

  刷新后状态:
  ┌─────────────┐    ┌─────────────────────────────────────────────┐
  │ AccessToken   │    │ RefreshToken                            │
  │ secret: AT2 │    │ secret: RT2 (NEW!)                   │
  │ expires: +60m│    │ expires: +1y                           │
  └─────────────┘    └─────────────────────────────────────────────┘

  旧 Token 状态:
  - AT1: 仍然有效 (直到过期)
  - RT1: 已删除 (无法再使用)
```

### 3.4 Token 缓存机制

#### Redis 缓存用于提高验证效率

位于 `apps/api/v2/src/modules/oauth-clients/services/oauth-flow.service.ts`:

```typescript
async propagateAccessToken(accessToken: string) {
  try {
    const ownerId = await this.tokensRepository.getAccessTokenOwnerId(accessToken);
    let expiry = await this.tokensRepository.getAccessTokenExpiryDate(accessToken);

    if (!expiry) {
      this.logger.warn(`Token for ${ownerId} had no expiry time, assuming it's new.`);
      expiry = DateTime.now().plus({ minute: 60 }).startOf("minute").toJSDate();
    }

    const cacheKey = this._generateActKey(accessToken);
    await this.redisService.redis.hmset(cacheKey, {
      ownerId: ownerId,
      expiresAt: expiry?.toJSON(),
    });

    await this.redisService.redis.expireat(cacheKey, Math.floor(expiry.getTime() / 1000));
  } catch (err) {
    this.logger.error("Access Token Propagation Failed, falling back to DB...", err);
  }
}
```

#### Token 验证流程

```typescript
async validateAccessToken(secret: string) {
  // 1. 先查缓存
  const { status, cacheKey } = await this.readFromCache(secret);

  if (status === "CACHE_HIT") {
    return true;
  }

  // 2. 缓存未命中，查数据库
  const tokenExpiresAt = await this.tokensRepository.getAccessTokenExpiryDate(secret);

  if (!tokenExpiresAt) {
    throw new UnauthorizedException(INVALID_ACCESS_TOKEN);
  }

  if (new Date() > tokenExpiresAt) {
    throw new TokenExpiredException();
  }

  // 3. 回写缓存
  void (await this.redisService.redis.hmset(cacheKey, { expiresAt: tokenExpiresAt.toJSON() }));
  void (await this.redisService.redis.expireat(cacheKey, Math.floor(tokenExpiresAt.getTime() / 1000)));

  return true;
}
```

---

## 第四部分：Scope 与 User/Team 身份及资源访问映射关系

### 4.1 Scope 定义现状

#### 数据库枚举定义

位于 `packages/prisma/schema.prisma:1496`:

```prisma
enum AccessScope {
  READ_BOOKING
  READ_PROFILE
}
```

#### OAuth 2.0 授权输入

位于 `apps/api/v2/src/modules/auth/oauth2/inputs/authorize.input.ts`:

```typescript
export class OAuth2AuthorizeInput {
  @ApiProperty({
    description: "The scopes to request",
    enum: AccessScope,
    isArray: true,
    example: ["READ_BOOKING", "READ_PROFILE"],
  })
  @IsArray()
  @IsEnum(AccessScope, { each: true })
  scopes: AccessScope[] = [];

  // ... 其他字段
}
```

### 4.2 实际权限位掩码定义

位于 `packages/platform/constants/permissions.ts`:

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
```

### 4.3 Scope 与 Permissions 映射 Gap 分析

#### **关键发现：Scope 与 Permissions 存在明显差异

| 维度 | AccessScope (枚举) | Permissions (位掩码) |
|------|---------------------|----------------------|
| **定义位置** | Prisma Schema | platform-constants |
| **权限数量** | 2 种 | 10 种 |
| **权限类型** | 仅读权限 | 读 + 写权限 |
| **覆盖资源** | Booking, Profile | EventType, Booking, Schedule, Apps, Profile |
| **实际使用** | OAuth 2.0 标准端点 | OAuth Client 注册和权限守卫 |

#### **Gap 详细对照表

| AccessScope | 对应 Permissions | 状态 |
|-------------|-------------------|------|
| READ_BOOKING | BOOKING_READ (4) | ✅ 存在对应 |
| READ_PROFILE | PROFILE_READ (256) | ✅ 存在对应 |
| - | EVENT_TYPE_READ (1) | ❌ Scope 缺失 |
| - | EVENT_TYPE_WRITE (2) | ❌ Scope 缺失 |
| - | BOOKING_WRITE (8) | ❌ Scope 缺失 |
| - | SCHEDULE_READ (16) | ❌ Scope 缺失 |
| - | SCHEDULE_WRITE (32) | ❌ Scope 缺失 |
| - | APPS_READ (64) | ❌ Scope 缺失 |
| - | APPS_WRITE (128) | ❌ Scope 缺失 |
| - | PROFILE_WRITE (512) | ❌ Scope 缺失 |

#### **潜在问题**：

1. **Scope 定义不完整**：
   - `AccessScope` 枚举仅定义了 2 种读权限
   - 但实际系统支持 10 种权限（5 类资源 × 读写）

2. **缺少写权限 Scope：
   - 没有 `WRITE_BOOKING`、`WRITE_PROFILE` 等写权限 Scope
   - 第三方应用无法通过 OAuth 2.0 标准流程请求写权限

3. **资源覆盖不全**：
   - 缺少 `EVENT_TYPE`、`SCHEDULE`、`APPS` 三类资源的 Scope

### 4.4 身份映射机制

#### OAuth Token 身份关联

```prisma
model AccessToken {
  id Int @id @default(autoincrement())
  
  secret    String   @unique
  createdAt DateTime @default(now())
  expiresAt DateTime
  
  // 关键关联
  owner  User                @relation(fields: [userId], references: [id], onDelete: Cascade)
  client PlatformOAuthClient @relation(fields: [platformOAuthClientId], references: [id], onDelete: Cascade)
  
  platformOAuthClientId String
  userId                Int
}
```

#### JWT Payload 结构

```typescript
// Access Token JWT Payload
{
  type: "access_token",
  clientId: "oauth_client_id",      // OAuth Client ID
  ownerId: 123,                      // 用户 ID
  expiresAt: 1714896000000,        // 过期时间戳
  userId: 123,                       // 用户 ID (重复字段，向后兼容)
  jti: "uuid-v4-token-id",           // JWT ID
  iat: 1714892400                    // 签发时间
}
```

#### 身份解析流程

位于 `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts`:

```typescript
async accessTokenStrategy(accessToken: string, request: ApiAuthGuardRequest, origin?: string) {
  // 1. 验证 Token 有效性
  const accessTokenValid = await this.oauthFlowService.validateAccessToken(accessToken);
  if (!accessTokenValid) {
    throw new UnauthorizedException(INVALID_ACCESS_TOKEN);
  }

  // 2. 获取关联的 OAuth Client
  const client = await this.tokensRepository.getAccessTokenClient(accessToken);
  if (!client) {
    throw new UnauthorizedException(
      "ApiAuthStrategy - access token - OAuth client not found given the access token"
    );
  }

  // 3. 验证请求来源 (CORS)
  if (origin && !isOriginAllowed(origin, client.redirectUris)) {
    throw new UnauthorizedException(
      `ApiAuthStrategy - access token - Invalid request origin"
    );
  }

  // 4. 获取 Token 所有者用户 ID
  const ownerId = await this.tokensRepository.getAccessTokenOwnerId(accessToken);

  if (!ownerId) {
    throw new UnauthorizedException(
      `ApiAuthStrategy - access token - ${INVALID_ACCESS_TOKEN}. No owner found for this access token.`
    );
  }

  // 5. 获取用户详细信息
  const user: UserWithProfile | null = await this.userRepository.findByIdWithProfile(ownerId);
  if (!user) {
    throw new UnauthorizedException(
      "ApiAuthStrategy - access token - User associated with the access token not found."
    );
  }

  // 6. 确定组织 ID
  const organizationId = this.usersService.getUserMainOrgId(user) as number;
  request.organizationId = organizationId;

  return user;
}
```

### 4.5 组织 (Organization) 映射

#### OAuth Client 与组织的绑定

```prisma
model PlatformOAuthClient {
  id             String   @id @default(cuid())
  organizationId Int       // 必须绑定到组织
  organization   Team     @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  
  // ... 其他字段
}
```

#### 组织 ID 确定方式

| Token 类型 | 组织 ID 来源 | 代码位置 |
|------------|--------------|----------|
| **API Key** | `ApiKey.teamId` | `api-auth.strategy.ts:256 |
| **OAuth Client Credentials** | `PlatformOAuthClient.organizationId` | `api-auth.strategy.ts:206` |
| **Access Token** | 用户主组织 (`getUserMainOrgId(user)` | `api-auth.strategy.ts:295` |
| **Third-Party JWT** | `teamId` 或用户主组织 | `api-auth.strategy.ts:335-346` |

### 4.6 资源访问校验流程

#### 权限守卫 (PermissionsGuard)

位于 `apps/api/v2/src/modules/auth/guards/permissions/permissions.guard.ts`:

```typescript
async canActivate(context: ExecutionContext): Promise<boolean> {
  // 1. 从装饰器获取要求的权限
  const requiredPermissions = this.reflector.get(Permissions, context.getHandler());
  
  if (!requiredPermissions?.length || !Object.keys(requiredPermissions)?.length) {
    return true;  // 无权限要求，直接通过
  }

  const request = context.switchToHttp().getRequest();
  const bearerToken = request.get("Authorization")?.replace("Bearer ", "");
  const nextAuthToken = await getToken({ req: request, secret: nextAuthSecret });
  const apiKey = bearerToken && isApiKey(bearerToken, this.config.get("api.apiKeyPrefix") ?? "cal_");
  const isThirdPartyBearerToken = bearerToken && this.getDecodedThirdPartyAccessToken(bearerToken);

  // 2. 判断认证方式
  if (nextAuthToken || apiKey || isThirdPartyBearerToken) {
    return true;  // 跳过权限检查
  }

  // 3. OAuth Access Token 或 Client ID 需要检查权限
  if (!bearerToken && !oAuthClientId) {
    throw new ForbiddenException("No authentication provided");
  }

  // 4. 获取 OAuth Client 的权限
  const oAuthClient = bearerToken
    ? await this.getOAuthClientByAccessToken(bearerToken)
    : await this.getOAuthClientById(oAuthClientId);

  // 5. 位运算权限检查
  const hasRequiredPermissions = hasPermissions(oAuthClient.permissions, [...requiredPermissions]);

  if (!hasRequiredPermissions) {
    throw new ForbiddenException(
      `OAuth client does not have the required permissions.`
    );
  }

  return true;
}
```

#### 权限检查工具函数

位于 `packages/platform/utils/permissions.ts`:

```typescript
// 检查单个权限
export const hasPermission = (userPermissions: number, permission: PLATFORM_PERMISSION): boolean => {
  return (userPermissions & permission) === permission;
};

// 检查多个权限 (全部满足)
export const hasPermissions = (userPermissions: number, permissions: PLATFORM_PERMISSION[]): boolean => {
  return permissions.every((permission) => hasPermission(userPermissions, permission));
};
```

### 4.7 完整的资源访问控制流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    资源访问控制完整流程                                │
└─────────────────────────────────────────────────────────────────────────┘

  请求到达 API 端点
           │
           ▼
  ┌──────────────────┐
  │  ApiAuthGuard    │
  │  (认证守卫)       │
  └────────┬─────────┘
           │
           ▼ 认证通过
           │
           ▼
  ┌──────────────────┐
  │  解析身份信息  │
  │  - 用户 ID     │
  │  - 组织 ID     │
  │  - 认证方式    │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────┐
  │  PermissionsGuard │
  │  (权限守卫)       │
  └────────┬─────────┘
           │
           ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    判断认证方式                          │
  └─────────────────────────────────────────────────────────────┘
           │
           ├──────────┬──────────┬──────────┐
           ▼           ▼           ▼           ▼
      ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
      │API Key  │ │NextAuth │ │Third-Party│ │OAuth    │
      │         │ │Token    │ │JWT Token │ │Access   │
      └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
           │            │            │            │
           ▼            ▼            ▼            │
      ┌─────────────────────────────────┐           │
      │      跳过权限检查                │           │
      │  (直接代表用户，全权访问)     │           │
      └─────────────────────────────────┘           │
                                                     │
                                                     ▼
                                          ┌──────────────────┐
                                          │ 检查 OAuth Client │
                                          │ 的 permissions 位  │
                                          │ 掩码              │
                                          └────────┬─────────┘
                                                   │
                                                   ▼
                                          ┌──────────────────┐
                                          │ 位运算权限检查   │
                                          │ (clientPerms &  │
                                          │  requiredPerms) │
                                          └────────┬─────────┘
                                                   │
                        ┌──────────────────────────┼──────────────────────────┐
                        ▼                       ▼                       ▼
                  ┌─────────┐           ┌─────────┐           ┌─────────┐
                  │ 权限   │           │ 权限   │           │ 权限   │
                  │ 满足   │           │ 不满足 │           │ 部分   │
                  │        │           │        │           │ 满足   │
                  └────┬───┘           └────┬───┘           └────┬───┘
                       │                    │                    │
                       ▼                    ▼                    ▼
                  ┌─────────┐           ┌─────────┐           ┌─────────┐
                  │ 允许   │           │ 拒绝   │           │ 拒绝   │
                  │ 访问   │           │ 访问   │           │ 访问   │
                  └─────────┘           └─────────┘           └─────────┘
```

---

## 第五部分：OAuth Client 与 Platform API Key / Permissions 位掩码差异对比

### 5.1 核心差异对照表

| 维度 | OAuth Client | API Key |
|------|-------------|---------|
| **设计目的** | 第三方应用集成 | 服务器到服务器集成 |
| **权限模型** | 位掩码权限 (10种) | 无单独权限 (代表用户) |
| **权限检查** | 需要检查 PermissionsGuard | 跳过权限检查 |
| **Token 类型** | Access Token + Refresh Token | 单一 API Key |
| **有效期** | Access: 60分钟<br>Refresh: 1年 | 可配置，支持永不过期 |
| **身份关联** | Client + User | User + 可选 Team |
| **安全机制** | 刷新令牌轮换、缓存 | 哈希存储、过期检查 |
| **Scope 支持** | 部分支持 (OAuth 2.0标准) | 不支持 |

### 5.2 详细差异分析

#### 差异 1：权限检查机制

**OAuth Client**:

```typescript
// PermissionsGuard 中的逻辑
if (nextAuthToken || apiKey || isThirdPartyBearerToken) {
  return true;  // 跳过
}

// OAuth Access Token 需要检查
const oAuthClient = await this.getOAuthClientByAccessToken(bearerToken);
const hasRequiredPermissions = hasPermissions(oAuthClient.permissions, [...requiredPermissions]);
```

**API Key**:

```typescript
// 直接跳过权限检查，代表用户全权访问
// API Key 认证通过后，拥有用户的所有权限
```

**影响：
- OAuth Client：精细粒度权限控制
- API Key：粗粒度，代表用户所有权限

#### 差异 2：存储与安全

| 特性 | OAuth Token | API Key |
|------|------------|---------|
| **生成方式** | JWT 签名 | 随机字符串 |
| **存储方式** | 明文 JWT | SHA256 哈希 |
| **验证方式** | JWT 验证 + 数据库查询 | 哈希比对 |
| **泄露风险** | 短期有效，可撤销 | 长期有效，需主动撤销 |
| **刷新机制** | Refresh Token 轮换 | 无，需重新生成 |

#### 差异 3：身份与组织关联

**OAuth Client 关联链：

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ PlatformOAuth   │     │  AccessToken    │     │     User        │
│    Client       │────▶│                 │────▶│                 │
│  - id          │     │  - secret      │     │  - id          │
│  - organizationId│    │  - userId      │     │  - profiles    │
│  - permissions   │    │  - clientId    │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │                                              │
         ▼                                              ▼
┌─────────────────┐                          ┌─────────────────┐
│     Team        │                          │  User's Main    │
│ (Organization)  │◀────────────────────────│  Organization   │
└─────────────────┘                          └─────────────────┘
```

**API Key 关联链**：

```
┌─────────────────┐
│    ApiKey       │
│                 │
│  - userId       │────▶  直接关联用户
│  - teamId?      │────▶  可选关联组织
└─────────────────┘
```

#### 差异 4：权限表示方式

**OAuth Client Permissions (位掩码)**：

```typescript
// 存储: 整数
permissions: 12  // 4 | 8 = BOOKING_READ | BOOKING_WRITE

// 检查: 位运算
hasPermission(12, BOOKING_READ);   // 12 & 4 = 4 → true
hasPermission(12, EVENT_TYPE_READ);      // 12 & 1 = 0 → false
```

**API Key (无权限概念)**：

```typescript
// API Key 不存储权限信息
// 认证通过后，拥有用户的所有权限
```

### 5.3 使用场景对比

| 场景 | 推荐方案 | 原因 |
|------|---------|------|
| **第三方应用集成** | OAuth 2.0 | 需要精细权限控制、用户授权 |
| **服务器到服务器** | API Key | 简单直接，代表用户全权访问 |
| **短期访问** | Access Token | 60 分钟有效期，安全 |
| **长期集成** | API Key / Refresh Token | 可配置过期时间 |
| **需要用户授权** | OAuth 2.0 | 标准授权码流程 |
| **内部服务调用** | API Key | 简单高效 |

### 5.4 权限模型对比表

| 权限 | OAuth Client | API Key |
|------|-------------|---------|
| **EVENT_TYPE_READ** | ✅ 位掩码检查 | ❌ 无检查 |
| **EVENT_TYPE_WRITE** | ✅ 位掩码检查 | ❌ 无检查 |
| **BOOKING_READ** | ✅ 位掩码检查 | ❌ 无检查 |
| **BOOKING_WRITE** | ✅ 位掩码检查 | ❌ 无检查 |
| **SCHEDULE_READ** | ✅ 位掩码检查 | ❌ 无检查 |
| **SCHEDULE_WRITE** | ✅ 位掩码检查 | ❌ 无检查 |
| **APPS_READ** | ✅ 位掩码检查 | ❌ 无检查 |
| **APPS_WRITE** | ✅ 位掩码检查 | ❌ 无检查 |
| **PROFILE_READ** | ✅ 位掩码检查 | ❌ 无检查 |
| **PROFILE_WRITE** | ✅ 位掩码检查 | ❌ 无检查 |
| **用户所有权限** | ❌ 需要显式授予 | ✅ 默认拥有 |

---

## 第六部分：发现的 Gap 与建议

### 6.1 已识别的 Gap

#### Gap 1: Scope 定义不完整

**问题**：
- `AccessScope` 枚举仅定义了 2 种读权限
- 但系统实际支持 10 种权限

**影响**：
- 第三方应用无法通过 OAuth 2.0 标准流程请求完整权限
- 写权限和其他资源类型无法通过 Scope 请求

**建议**：
```prisma
// 建议扩展 AccessScope 枚举
enum AccessScope {
  EVENT_TYPE_READ
  EVENT_TYPE_WRITE
  BOOKING_READ
  BOOKING_WRITE
  SCHEDULE_READ
  SCHEDULE_WRITE
  APPS_READ
  APPS_WRITE
  PROFILE_READ
  PROFILE_WRITE
}
```

#### Gap 2: OAuth 2.0 标准端点与实际权限映射缺失

**问题**：
- OAuth 2.0 标准端点 (`/oauth2/authorize`, `/oauth2/token`) 使用 `AccessScope`
- 但实际权限检查使用 `PERMISSION_MAP` 位掩码
- 两者之间缺少映射关系

**影响**：
- OAuth 2.0 标准流程与实际权限系统脱节
- 第三方应用通过标准 OAuth 2.0 流程无法获得完整权限

**建议**：
```typescript
// 建议添加 Scope 到 Permissions 的映射
const SCOPE_TO_PERMISSION_MAP: Record<AccessScope, number> = {
  [AccessScope.EVENT_TYPE_READ]: EVENT_TYPE_READ,
  [AccessScope.EVENT_TYPE_WRITE]: EVENT_TYPE_WRITE,
  [AccessScope.BOOKING_READ]: BOOKING_READ,
  [AccessScope.BOOKING_WRITE]: BOOKING_WRITE,
  // ... 其他映射
};
```

#### Gap 3: API Key 无权限细粒度控制

**问题**：
- API Key 认证跳过权限检查
- 拥有用户的所有权限

**影响**：
- 无法实现最小权限原则
- API Key 泄露风险高

**建议**：
- 考虑为 API Key 添加权限位掩码字段
- 或使用更细粒度的权限控制

#### Gap 4: Client Secret 存储方式

**问题**：
- OAuth Client Secret 以 JWT 明文存储
- 虽然 JWT 包含签名，但内容可解码

**对比**：
- API Key 存储 SHA256 哈希
- OAuth Secret 存储明文 JWT

**建议**：
- 考虑对 Client Secret 进行哈希存储
- 或使用更安全的存储方式

### 6.2 架构建议

#### 建议 1: 统一权限模型

```
┌─────────────────────────────────────────────────────────────────┐
│                    统一权限模型架构                            │
└─────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────┐
  │                    权限定义层 (统一)                            │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
  │  │ PERMISSION_ │  │  Scope      │  │  Permission  │       │
  │  │ MAP (位掩码) │  │  (字符串)   │  │  Groups     │       │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
  │         │                  │                  │               │
  │         └──────────────────┼──────────────────┘               │
  │                            ▼                                  │
  │              ┌─────────────────────────┐                    │
  │              │  双向映射关系            │                    │
  │              │  Scope ↔ Permissions    │                    │
  │              └─────────────────────────┘                    │
  └─────────────────────────────────────────────────────────────┘
                            │
                            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    认证授权层                          │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
  │  │ OAuth 2.0  │  │  API Key    │  │  NextAuth  │       │
  │  │  标准流程  │  │             │  │             │       │
  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
  │         │                  │                  │               │
  │         └──────────────────┼──────────────────┘               │
  │                            ▼                                  │
  │              ┌─────────────────────────┐                    │
  │              │  统一权限检查               │                    │
  │              │  PermissionsGuard       │                    │
  │              └─────────────────────────┘                    │
  └─────────────────────────────────────────────────────────────┘
                            │
                            ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    资源访问层                              │
  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
  │  │ EventType   │  │  Booking    │  │  Schedule   │       │
  │  │             │  │             │  │             │       │
  │  └─────────────┘  └─────────────┘  └─────────────┘       │
  │  ┌─────────────┐  ┌─────────────┐                      │
  │  │ Apps        │  │  Profile    │                      │
  │  │             │  │             │                      │
  │  └─────────────┘  └─────────────┘                      │
  └─────────────────────────────────────────────────────────────┘
```

#### 建议 2: 增强 API Key 安全性

```typescript
// 建议扩展 ApiKey 模型
model ApiKey {
  id         String      @id @unique @default(cuid())
  userId     Int
  teamId     Int?
  note       String?
  createdAt  DateTime    @default(now())
  expiresAt  DateTime?
  lastUsedAt DateTime?
  hashedKey  String      @unique()
  
  // 新增: 权限位掩码
  permissions Int         @default(0)  // 0 表示所有权限 (向后兼容)
  
  // ... 现有关联
}
```

#### 建议 3: 完善 Scope 与 Permissions 映射

```typescript
// Scope 到 Permissions 的映射
export const SCOPE_PERMISSION_MAP: Record<string, number> = {
  "event_type:read": EVENT_TYPE_READ,
  "event_type:write": EVENT_TYPE_WRITE,
  "booking:read": BOOKING_READ,
  "booking:write": BOOKING_WRITE,
  "schedule:read": SCHEDULE_READ,
  "schedule:write": SCHEDULE_WRITE,
  "apps:read": APPS_READ,
  "apps:write": APPS_WRITE,
  "profile:read": PROFILE_READ,
  "profile:write": PROFILE_WRITE,
};

// 权限组映射 (如 "booking" = read + write)
export const SCOPE_GROUP_MAP: Record<string, number> = {
  "event_type": EVENT_TYPE_READ | EVENT_TYPE_WRITE,
  "booking": BOOKING_READ | BOOKING_WRITE,
  "schedule": SCHEDULE_READ | SCHEDULE_WRITE,
  "apps": APPS_READ | APPS_WRITE,
  "profile": PROFILE_READ | PROFILE_WRITE,
};
```

---

## 附录：代码引用索引

### OAuth Client 注册
- **Service**: `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients.service.ts:17-29`
- **Input Service**: `apps/api/v2/src/modules/oauth-clients/services/oauth-clients/oauth-clients-input.service.ts:11-28`
- **Repository**: `apps/api/v2/src/modules/oauth-clients/oauth-client.repository.ts:11-21`

### 密钥生成与存储
- **JWT Service**: `apps/api/v2/src/modules/jwt/jwt.service.ts:1-33`
- **Token Repository**: `apps/api/v2/src/modules/tokens/tokens.repository.ts:55-93`
- **Schema**: `packages/prisma/schema.prisma:1716-1741`

### 授权码流程
- **Authorize Endpoint**: `apps/api/v2/src/modules/oauth-clients/controllers/oauth-flow/oauth-flow.controller.ts:48-81`
- **Exchange Endpoint**: `apps/api/v2/src/modules/oauth-clients/controllers/oauth-flow/oauth-flow.controller.ts:83-106`
- **Flow Service**: `apps/api/v2/src/modules/oauth-clients/services/oauth-flow.service.ts:106-138`

### 刷新 Token 流程
- **Refresh Endpoint**: `apps/api/v2/src/modules/oauth-clients/controllers/oauth-flow/oauth-flow.controller.ts:108-133`
- **Refresh Service**: `apps/api/v2/src/modules/oauth-clients/services/oauth-flow.service.ts:140-169`
- **Token Refresh**: `apps/api/v2/src/modules/tokens/tokens.repository.ts:171-213`

### 权限系统
- **Permissions Constants**: `packages/platform/constants/permissions.ts:1-69`
- **Permissions Utils**: `packages/platform/utils/permissions.ts:16-73`
- **Permissions Guard**: `apps/api/v2/src/modules/auth/guards/permissions/permissions.guard.ts:27-72`
- **AccessScope Enum**: `packages/prisma/schema.prisma:1496-1499`

### 身份映射
- **API Auth Strategy**: `apps/api/v2/src/modules/auth/strategies/api-auth/api-auth.strategy.ts:261-299`
- **Token Validation**: `apps/api/v2/src/modules/oauth-clients/services/oauth-flow.service.ts:68-93`

---

**文档生成日期**: 2026-05-04
**分析范围**: Cal.diy API v2 OAuth 体系
