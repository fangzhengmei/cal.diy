# Cal.diy 企业用户自定义域名绑定深度技术指南

本文档详细说明 Cal.diy 企业用户绑定自有域名的**真实技术实现**，包括 Host 路由规则边界、SSL 证书管理的代码可控边界、以及反向代理对 OAuth 回调和 Cookie 会话的实际影响。

---

## 目录

1. [Host 映射组织的真实规则与边界](#1-host-映射组织的真实规则与边界)
2. [SSL 证书管理：代码可控 vs 平台托管](#2-ssl-证书管理代码可控-vs-平台托管)
3. [反向代理对 OAuth 回调校验的影响](#3-反向代理对-oauth-回调校验的影响)
4. [Cookie 作用域与会话连续性](#4-cookie-作用域与会话连续性)
5. [环境变量配置参考](#5-环境变量配置参考)

---

## 1. Host 映射组织的真实规则与边界

### 1.1 核心架构：Next.js Rewrites 机制

Cal.diy 使用 **Next.js Rewrites** 实现基于 Host 头的组织路由，这是一种**内部重写**（URL 不变，内部路径映射），而非外部重定向。

#### 1.1.1 正则匹配规则详解

**核心配置文件**: `apps/web/getNextjsOrgRewriteConfig.ts`

```typescript
// 核心正则匹配逻辑
export const getRegExpThatMatchesAllOrgDomains = ({ webAppUrl }: { webAppUrl: string }): string => {
  if (isSingleOrgModeEnabled) {
    return `.*`;  // 单组织模式：匹配所有域名
  }
  const subdomainRegExp = getRegExpNotMatchingLeftMostSubdomain(webAppUrl);
  return `^(?<${orgSlugCaptureGroupName}>${subdomainRegExp})\\.(?!vercel\\.app).*`;
};
```

**多组织模式正则解析**：

```
正则: ^(?<orgSlug>(?!app)[^.]+)\.(?!vercel\.app).*

分解说明:
^              - 字符串开始
(?<orgSlug>    - 命名捕获组，提取组织 slug
  (?!app)      - 负向前瞻：排除 "app" 子域名（主应用域名）
  [^.]+        - 匹配一个或多个非点字符
)
\.             - 匹配点号
(?!vercel\.app)- 负向前瞻：排除 vercel.app 内部域名
.*             - 匹配剩余部分
```

**实际匹配示例**（假设 `NEXT_PUBLIC_WEBAPP_URL=https://app.cal.com`）：

| 请求域名 | 是否匹配 | orgSlug 值 | 说明 |
|----------|----------|------------|------|
| `app.cal.com` | ❌ 不匹配 | - | 负向前瞻 `(?!app)` 排除主应用 |
| `acme.cal.com` | ✅ 匹配 | `acme` | 正常组织子域名 |
| `dunder.cal.com` | ✅ 匹配 | `dunder` | 正常组织子域名 |
| `cal-com.vercel.app` | ❌ 不匹配 | - | 负向前瞻排除 vercel.app |
| `cal.acme.com` | ✅ 匹配 | `cal` | 自定义域名（完全匹配） |
| `booking.dunder.com` | ✅ 匹配 | `booking` | 任意自定义域名 |

#### 1.1.2 单组织模式 vs 多组织模式

| 维度 | 单组织模式 | 多组织模式 |
|------|-----------|-----------|
| **触发条件** | `NEXT_PUBLIC_SINGLE_ORG_SLUG` 有值 | 未设置 `NEXT_PUBLIC_SINGLE_ORG_SLUG` |
| **Host 匹配正则** | `.*`（匹配所有域名） | 复杂正则，排除主应用子域名 |
| **orgSlug 来源** | 环境变量固定值 | 从 Host 头动态提取 |
| **适用场景** | 自托管、单一企业 | SaaS 平台、多租户 |
| **域名支持** | 任意域名都路由到同一组织 | 需按子域名区分不同组织 |

**单组织模式的真实行为**：

```typescript
// getNextjsOrgRewriteConfig.ts
if (isSingleOrgModeEnabled) {
  console.log("Single-Org-Mode enabled - Consider all domains to be org domains");
  return `.*`;  // 所有域名都被视为组织域名
}
```

这意味着：
- `cal.company.com` → 路由到 `NEXT_PUBLIC_SINGLE_ORG_SLUG` 指定的组织
- `booking.company.com` → 同样路由到同一组织
- `localhost:3000`（开发环境）→ 也路由到同一组织

### 1.2 路由重写规则与保留路径排除

**核心配置文件**: `apps/web/pagesAndRewritePaths.ts`

#### 1.2.1 保留路径排除机制

为了防止组织 slug 与系统路由冲突，Cal.diy 实现了**多层排除机制**：

```typescript
// 自动扫描所有页面路由，排除顶层路径名
export const topLevelRoutesExcludedFromOrgRewrite: string[] = globSync(
  "{pages,app,app/(booking-page-wrapper),app/(use-page-wrapper),app/(use-page-wrapper)/(main-nav)}/**/[^_]*.{tsx,js,ts}",
  { cwd: __dirname }
)
  .map((filename) => filename
    .replace(/^(app\/\(use-page-wrapper\)\/\(main-nav\)|app\/\(use-page-wrapper\)|app\/\(booking-page-wrapper\)|pages|app)\//, "")
    .replace(/(\.tsx|\.js|\.ts)/, "")
    .replace(/\/.*/, "")  // 只取顶层路径
  )
  .filter((v, i, self) => self.indexOf(v) === i && 
    !["[user]", "_trpc", "layout", ...].some((prefix) => v.startsWith(prefix))
  )
  .filter((page) => !topLevelRouteNamesWhitelistedForRewrite.includes(page));
```

**最终排除的路径类型**：

| 排除类型 | 示例路径 | 说明 |
|----------|----------|------|
| **系统路由** | `_next`, `_trpc` | Next.js 内部路径 |
| **页面路由** | `bookings`, `settings`, `auth` | 自动扫描的页面目录 |
| **虚拟路由** | `forms`, `router`, `success`, `cancel`, `app` | Rewrite 使用的虚拟路径 |
| **静态资源** | `embed`, `public` | 公共资源路径 |
| **动态路由** | `[user]`, `[slug]` | 动态路径模板 |

#### 1.2.2 白名单机制

少数常用名称被特意**允许**作为组织/团队 slug：

```typescript
export const topLevelRouteNamesWhitelistedForRewrite: string[] = [
  "onboarding",  // 常见的团队名称，允许使用
];
```

#### 1.2.3 实际路由重写配置

**核心配置文件**: `apps/web/next.config.ts`

```typescript
// Rewrites 配置（beforeFiles 阶段，优先级最高）
async rewrites() {
  const { orgSlug } = nextJsOrgRewriteConfig;
  const beforeFiles = [
    // ... 其他 rewrites
    
    ...(isOrganizationsEnabled
      ? [
          // 1. 组织根路径: acme.cal.com/ → /team/acme?isOrgProfile=1
          orgDomainMatcherConfig.root
            ? {
                ...orgDomainMatcherConfig.root,
                destination: `/team/${orgSlug}?isOrgProfile=1`,
              }
            : null,
          
          // 2. 组织根路径嵌入: acme.cal.com/embed → /team/acme/embed?isOrgProfile=1
          orgDomainMatcherConfig.rootEmbed
            ? {
                ...orgDomainMatcherConfig.rootEmbed,
                destination: `/team/${orgSlug}/embed?isOrgProfile=1`,
              }
            : null,
          
          // 3. 用户路径: acme.cal.com/john → /org/acme/john
          {
            ...orgDomainMatcherConfig.user,
            destination: `/org/${orgSlug}/:user`,
          },
          
          // 4. 事件类型路径: acme.cal.com/john/30min → /org/acme/john/30min
          {
            ...orgDomainMatcherConfig.userType,
            destination: `/org/${orgSlug}/:user/:type`,
          },
          
          // 5. 嵌入路径: acme.cal.com/john/30min/embed → /org/acme/john/30min/embed
          {
            ...orgDomainMatcherConfig.userTypeEmbed,
            destination: `/org/${orgSlug}/:user/:type/embed`,
          },
        ]
      : []),
  ].filter(isNotNull);
  // ...
}
```

### 1.3 数据层面的组织标识

**数据库模型**: `packages/prisma/schema.prisma`

#### 1.3.1 Team 模型（组织即 Team）

```prisma
model Team {
  id              Int                     @id @default(autoincrement())
  name            String
  slug            String?                 // 组织/团队 slug，全局唯一
  isOrganization  Boolean                 @default(false)  // 关键标识：是否为组织
  
  // 组织设置（仅组织有）
  organizationSettings   OrganizationSettings?
  
  // 组织拥有的用户 Profile
  orgProfiles           Profile[]
  
  // 组织的用户（通过 organizationId 关联）
  orgUsers              User[]                  @relation("scope")
  
  // ... 其他字段
}
```

#### 1.3.2 Profile 模型（用户在组织中的身份）

```prisma
model Profile {
  id              Int        @id @default(autoincrement())
  uid             String     @unique  // UUID，安全的 Profile ID
  userId          Int
  username        String     // 用户在组织内的用户名（可与全局不同）
  
  // 关键：关联到组织
  organizationId  Int
  organization    Team       @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  
  user            User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  // 唯一约束：同一组织内用户名唯一
  @@unique([username, organizationId])
  
  // 唯一约束：一个用户在一个组织内只有一个 Profile
  @@unique([userId, organizationId])
}
```

#### 1.3.3 路由匹配与数据查询的对应关系

```
请求: https://acme.cal.com/john/30min
  ↓
Step 1: Next.js Rewrite 提取 orgSlug = "acme"
  ↓
Step 2: 内部重写到 /org/acme/john/30min
  ↓
Step 3: getServerSideProps 获取 orgSlug 参数
  ↓
Step 4: 数据库查询：
        - 通过 slug 查找 Team (isOrganization=true)
        - 通过 orgSlug + username 查找 Profile
        - 返回组织上下文的数据
```

**实际查询代码**（`apps/web/server/lib/[user]/getServerSideProps.ts`）：

```typescript
export async function getUsersInOrgContext(usernameList: string[], orgSlug: string | null) {
  const userRepo = new UserRepository(prisma);

  const usersInOrgContext = await userRepo.findUsersByUsername({
    usernameList,
    orgSlug,  // 传入组织 slug
  });

  if (usersInOrgContext.length) {
    return usersInOrgContext;
  }

  // 兜底：查找平台成员（无组织域名但有 Profile）
  return await userRepo.findPlatformMembersByUsernames({
    usernameList,
  });
}
```

### 1.4 路由边界与冲突处理

#### 1.4.1 路由优先级

Next.js Rewrites 的执行顺序决定了路由优先级：

```
优先级从高到低:
1. beforeFiles rewrites（组织路由在这里）
2. 文件系统路由（pages/ 和 app/ 目录）
3. afterFiles rewrites
4. 动态路由
5. 404
```

**关键含义**：组织子域名路由优先级**高于**常规页面路由。

#### 1.4.2 冲突处理示例

假设存在一个组织 slug = `settings`，会发生什么？

```
请求: https://settings.cal.com/

Step 1: 检查是否匹配 orgHostPath 正则
        - 多组织模式: (?!app)[^.]+\.(?!vercel\.app).*
        - "settings" ≠ "app"，匹配成功
        - orgSlug = "settings"

Step 2: Rewrite 到 /team/settings?isOrgProfile=1

Step 3: 但 "settings" 也在 topLevelRoutesExcludedFromOrgRewrite 中！
        - 这是从 pages/settings/ 自动扫描的

结果: 
  - 在单组织模式下：settings 被用作 orgSlug，不会冲突
  - 在多组织模式下：如果有人创建 slug=settings 的组织，会导致 /settings 页面在该子域名下无法访问
```

**实际保护机制**：

```typescript
// pagesAndRewritePaths.ts
// 构建正则时，所有排除的路径都被加入到路由的 negative lookahead 中

export const orgUserRoutePath = 
  `/:user((?!${getRegExpMatchingAllReservedRoutes("/?$")})[a-zA-Z0-9\\-_]+)`;

// 这意味着:
// /:user 路径只匹配不在保留列表中的用户名
// 但 Host 头的匹配没有这个保护！
```

#### 1.4.3 保留子域名列表

**配置项**: `RESERVED_SUBDOMAINS`（环境变量）

虽然代码中没有直接使用这个变量进行 Host 头验证，但在创建组织时应该检查：

```typescript
// 应该但尚未完全实现的验证
const RESERVED_SUBDOMAINS = process.env.RESERVED_SUBDOMAINS?.split(",") || [
  "app", "www", "api", "mail", "smtp", "imap",
  "admin", "dashboard", "status", "docs",
];

// 在创建组织时应验证:
// if (RESERVED_SUBDOMAINS.includes(orgSlug)) {
//   throw new Error("This subdomain is reserved");
// }
```

### 1.5 组织重定向机制

**核心文件**: `apps/web/lib/handleOrgRedirect.ts`

当用户从全局命名空间迁移到组织时，系统支持临时重定向：

```typescript
// tempOrgRedirect 表的使用
export async function handleOrgRedirect({
  slugs,
  redirectType,  // User 或 Team
  eventTypeSlug,
  context,
  currentOrgDomain,
}: HandleOrgRedirectParams) {
  // 查询重定向记录
  const redirects = await prisma.tempOrgRedirect.findMany({
    where: {
      type: redirectType,
      from: { in: slugs },
      fromOrgId: 0,  // 从全局命名空间（orgId=0）迁移
    },
  });

  if (!redirects.length) {
    return null;
  }

  // 构建新的目标 URL
  const newOrigin = new URL(redirects[0].toUrl).origin;
  // ...
  
  return {
    redirect: {
      permanent: false,  // 302 临时重定向
      destination: `${newOrigin}${newPath}${query}`,
    },
  };
}
```

**迁移示例**：

```
迁移前:
- 用户: cal.com/john87 (全局命名空间)
- 团队: cal.com/acme-sales (全局命名空间)

迁移到组织 "acme" 后:
- 创建 tempOrgRedirect 记录:
  - from: "john87", type: User, toUrl: "https://acme.cal.com/john"
  - from: "acme-sales", type: Team, toUrl: "https://acme.cal.com/sales"

访问旧链接:
- https://cal.com/john87 → 302 → https://acme.cal.com/john
- https://cal.com/acme-sales → 302 → https://acme.cal.com/sales
```

---

## 2. SSL 证书管理：代码可控 vs 平台托管

### 2.1 架构概览

Cal.diy 使用 **Vercel + Cloudflare** 双层架构管理自定义域名，但**代码可控制的范围非常有限**。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           代码可控边界                                     │
│                                                                           │
│  packages/lib/domainManager/                                              │
│  ├── organization.ts          ← 高层封装：创建/删除/重命名域名           │
│  └── deploymentServices/                                                  │
│      ├── vercel.ts          ← Vercel API 调用：添加/删除域名            │
│      └── cloudflare.ts      ← Cloudflare API 调用：添加/删除 DNS 记录  │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓ API 调用
┌─────────────────────────────────────────────────────────────────────────┐
│                          平台托管（黑盒）                                  │
│                                                                           │
│  Vercel 平台:                                                             │
│  ├── 域名所有权验证（DNS/HTTP）                                          │
│  ├── SSL 证书申请（Let's Encrypt）                                       │
│  ├── SSL 证书自动续期                                                    │
│  ├── 证书部署到全球边缘节点                                              │
│  └── 域名状态监控                                                        │
│                                                                           │
│  Cloudflare (可选代理):                                                   │
│  ├── Universal SSL 证书                                                  │
│  ├── CDN 缓存                                                            │
│  └── WAF 防护                                                            │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 代码可控的操作

**核心文件**: `packages/lib/domainManager/`

#### 2.2.1 组织域名管理

```typescript
// packages/lib/domainManager/organization.ts

// 创建域名：同时调用 Vercel 和 Cloudflare
export const createDomain = async (slug: string) => {
  const domain = `${slug}.${subdomainSuffix()}`;  // 如: acme.cal.com
  
  let domainConfigured = false;
  let dnsConfigured = true;

  // 代码可控：调用 Vercel API 添加域名
  if (process.env.VERCEL_URL) {
    domainConfigured = await createVercelDomain(domain);
  }

  // 代码可控：调用 Cloudflare API 添加 CNAME 记录
  if (process.env.CLOUDFLARE_DNS) {
    dnsConfigured = await addDnsRecord(domain);
  }

  return domainConfigured && dnsConfigured;
};

// 删除域名
export const deleteDomain = async (slug: string) => {
  const domain = `${slug}.${subdomainSuffix()}`;
  
  // 代码可控：调用 Vercel API 删除域名
  if (process.env.VERCEL_URL) {
    isDomainDeleted = await deleteVercelDomain(domain);
  }

  // 代码可控：调用 Cloudflare API 删除 DNS 记录
  if (process.env.CLOUDFLARE_DNS) {
    isDnsRecordDeleted = await deleteDnsRecord(domain);
  }
};

// 重命名域名：先建新，后删旧
export const renameDomain = async (oldSlug: string | null, newSlug: string) => {
  await createDomain(newSlug);  // 先确保新域名可用
  if (oldSlug) {
    try {
      await deleteDomain(oldSlug);  // 旧域名删除失败不阻塞
    } catch (_e) {
      log.error(`renameDomain: Failed to delete old domain ${oldSlug}`);
    }
  }
};
```

#### 2.2.2 Vercel API 封装

```typescript
// packages/lib/domainManager/deploymentServices/vercel.ts

// 添加域名到 Vercel 项目
export const createDomain = async (domain: string) => {
  const response = await fetch(
    `${vercelApiForProjectUrl}/domains?teamId=${process.env.TEAM_ID_VERCEL}`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${process.env.AUTH_BEARER_TOKEN_VERCEL}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ name: domain }),
    }
  );

  const data = vercelDomainApiResponseSchema.parse(await response.json());
  
  // 代码可控：只能检查已知的错误码
  if (data.error) {
    return handleDomainCreationError(data.error);
  }
  
  return true;  // 只能知道 API 调用成功，无法知道证书状态
};
```

**Vercel API 响应的限制**：

| 错误码 | 含义 | 代码能做什么 |
|--------|------|--------------|
| `forbidden` | 无权限管理域名 | 抛出错误，提示检查权限 |
| `domain_taken` | 域名被其他项目使用 | 抛出错误 |
| `domain_already_in_use` | 域名已在当前项目中 | 静默成功（幂等）|

**关键限制**：Vercel API 不会返回：
- SSL 证书是否已签发
- 域名验证进度
- 证书到期时间
- 自动续期状态

#### 2.2.3 Cloudflare DNS 管理

```typescript
// packages/lib/domainManager/deploymentServices/cloudflare.ts

// 添加 CNAME 记录
export const addDnsRecord = async (domain: string) => {
  const data = await api(
    `${cloudflareApiForZoneUrl}/dns_records`,
    {
      method: "POST",
      body: JSON.stringify({
        type: "CNAME",
        proxied: IS_RECORD_PROXIED,  // true: 走 Cloudflare 代理
        name: domain,
        content: process.env.CLOUDFLARE_VERCEL_CNAME,  // 如: cname.vercel-dns.com
        ttl: AUTOMATIC_TTL,  // 1 = 自动
      }),
    },
    cloudflareDnsRecordApiResponseSchema
  );

  // 处理已知错误码
  if (!data.success) {
    if (isRecordAlreadyExistError(data.errors)) {
      log.info(`CNAME already exists in Cloudflare: ${domain}`);
      return true;  // 已存在则视为成功
    }
    throw new HttpError({ message: "Failed to create dns-record", statusCode: 400 });
  }
  
  return true;
};
```

**Cloudflare 错误码**：

| 错误码 | 含义 | 处理方式 |
|--------|------|----------|
| `81053` | CNAME 已存在 | 静默成功 |
| `81057` | 记录已存在 | 静默成功 |
| `81044` | 记录不存在 | （删除时）静默成功 |

### 2.3 平台托管的能力（代码无法直接控制）

#### 2.3.1 Vercel 平台的 SSL 管理

**整个 SSL 生命周期完全由 Vercel 托管**：

```
用户操作: 调用 createVercelDomain("acme.cal.com")
           ↓
Vercel 平台自动执行（代码无法干预）:
1. 检查域名是否已指向 Vercel
2. 发起域名所有权验证
   - 方式 A: DNS TXT 记录验证
   - 方式 B: HTTP /.well-known/acme-challenge/ 验证
3. 验证通过后，向 Let's Encrypt 申请证书
4. 证书签发后，部署到全球边缘节点
5. 配置自动续期（到期前 30 天自动续期）
```

**代码无法获取的状态**：

| 状态信息 | 如何获取 | 代码能否查询 |
|----------|----------|--------------|
| 证书是否已签发 | Vercel Dashboard / CLI | ❌ 不能 |
| 域名验证进度 | Vercel Dashboard | ❌ 不能 |
| 证书到期时间 | Vercel Dashboard | ❌ 不能 |
| 自动续期是否成功 | Vercel Dashboard | ❌ 不能 |
| 证书颁发者 | Vercel Dashboard | ❌ 不能 |

#### 2.3.2 Vercel SSL 证书的特性

| 特性 | 说明 | 代码可控？ |
|------|------|-----------|
| **自动申请** | 添加域名后自动申请 | ❌ 平台行为 |
| **自动验证** | 支持 DNS 和 HTTP 验证 | ❌ 平台行为 |
| **自动续期** | 到期前 30 天自动续期 | ❌ 平台行为 |
| **多域名 SAN** | 支持多个域名在同一证书 | ❌ 平台行为 |
| **通配符证书** | 需要额外配置 | ❌ 需手动 |
| **自定义证书** | 支持上传自定义证书 | ✅ 通过 API |

#### 2.3.3 Cloudflare 代理模式的影响

当 `IS_RECORD_PROXIED = true`（默认）时：

```
用户请求: https://acme.cal.com/
           ↓
    Cloudflare 边缘节点
           ↓
    SSL 终止（Cloudflare 证书）
           ↓
    回源到 Vercel（HTTP 或 HTTPS）
           ↓
    Vercel 应用
```

**双层 SSL 配置**：

| 层级 | 证书提供方 | 自动续期 |
|------|-----------|----------|
| 客户端 ↔ Cloudflare | Cloudflare Universal SSL | ✅ 自动 |
| Cloudflare ↔ Vercel | Vercel (Let's Encrypt) | ✅ 自动 |

**Cloudflare 代理模式的代码控制**：

```typescript
// cloudflare.ts
const IS_RECORD_PROXIED = true;  // 写死在代码中，不可配置

// TODO: This and other settings should really come from DB 
// when admin allows configuring which deployment services to use
```

**关键限制**：代理模式当前是硬编码的，无法按组织配置。

### 2.4 代码可验证的边界

#### 2.4.1 只能验证 API 调用成功

```typescript
// 代码能知道的：
try {
  await createVercelDomain(domain);
  await addDnsRecord(domain);
  // ✅ 知道 API 调用成功了
} catch (e) {
  // ✅ 知道 API 调用失败了，以及已知的错误码
}

// 代码无法知道的：
// ❌ 域名 DNS 是否正确指向
// ❌ SSL 证书是否已签发
// ❌ 用户是否能正常访问
// ❌ 证书何时会过期
```

#### 2.4.2 间接验证方式

**通过健康检查间接验证**：

```typescript
// 代码可以实现（但当前未实现）：
async function verifyDomainAccessibility(domain: string): Promise<boolean> {
  try {
    const response = await fetch(`https://${domain}/api/health`, {
      method: "HEAD",
      timeout: 10000,
    });
    return response.ok;  // 能访问说明 DNS 和 SSL 都正常
  } catch {
    return false;
  }
}
```

**通过证书透明度日志间接验证**：

```typescript
// 代码可以实现（但当前未实现）：
async function checkCertificateIssued(domain: string): Promise<boolean> {
  // 查询 Certificate Transparency 日志
  // 如: crt.sh, Google CT Logs
  // 判断是否有近期签发的证书
}
```

### 2.5 域名状态管理的建议

当前实现存在以下局限：

```typescript
// organization.ts 中的 TODO 注释
// TODO: Ideally we should start storing the DNS and domain entries in DB 
// for each organization
```

**建议的数据库模型**：

```prisma
// 建议添加的模型
model OrganizationDomain {
  id              Int       @id @default(autoincrement())
  organizationId  Int
  domain          String    @unique
  
  // 代码可控的状态
  dnsRecordCreated   Boolean   @default(false)
  vercelDomainAdded  Boolean   @default(false)
  
  // 代码不可控但可通过探测获取的状态
  sslVerified        Boolean   @default(false)
  sslIssuedAt        DateTime?
  sslExpiresAt       DateTime?  // 需要定期探测更新
  
  // 元数据
  createdAt         DateTime  @default(now())
  verifiedAt        DateTime?
  
  organization      Team      @relation(fields: [organizationId], references: [id])
  
  @@index([organizationId])
}
```

---

## 3. 反向代理对 OAuth 回调校验的影响

### 3.1 OAuth 架构概览

Cal.diy 涉及**两类 OAuth 流程**，受反向代理的影响不同：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OAuth 流程分类                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  1. Cal.diy 作为 OAuth 客户端（连接第三方服务）                           │
│     ├── Google Calendar / Outlook / Zoom 等日历集成                     │
│     ├── Stripe 支付集成                                                  │
│     └── 其他 App Store 集成                                              │
│                                                                           │
│  2. Cal.diy 作为 OAuth 服务端（平台 OAuth）                               │
│     ├── 第三方应用调用 Cal.diy API                                       │
│     └── Platform OAuth 2.0 实现                                          │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Cal.diy 作为 OAuth 服务端的校验机制

**核心文件**: `packages/features/oauth/services/OAuthService.ts`

#### 3.2.1 Redirect URI 严格匹配

```typescript
// OAuthService.ts - 严格匹配，无宽容度
private validateRedirectUri(registeredUri: string, providedUri: string): void {
  if (providedUri !== registeredUri) {
    throw new ErrorWithCode(
      ErrorCode.BadRequest, 
      "invalid_request", 
      { reason: "redirect_uri_mismatch" }
    );
  }
}
```

**匹配规则详解**：

| 已注册 URI | 请求 URI | 是否匹配 | 原因 |
|-----------|----------|----------|------|
| `https://app.cal.com/api/oauth/callback` | `https://app.cal.com/api/oauth/callback` | ✅ 匹配 | 完全相同 |
| `https://app.cal.com/api/oauth/callback` | `https://app.cal.com/api/oauth/callback?foo=bar` | ❌ 不匹配 | 查询字符串不同 |
| `https://app.cal.com/api/oauth/callback` | `https://acme.cal.com/api/oauth/callback` | ❌ 不匹配 | 域名不同 |
| `https://app.cal.com/api/oauth/callback` | `http://app.cal.com/api/oauth/callback` | ❌ 不匹配 | 协议不同 |
| `https://app.cal.com/api/oauth/callback` | `https://app.cal.com:443/api/oauth/callback` | ❌ 不匹配 | 端口不同（即使是默认端口）|

**反向代理场景的问题**：

```
场景：Cloudflare 代理 → Vercel 源站

用户请求: https://acme.cal.com/api/oauth/authorize
           ↓
    Cloudflare (终止 SSL)
           ↓
    转发到 Vercel: http://vercel-internal/api/oauth/authorize
           ↓
    Next.js 应用收到请求:
    - protocol: http (不是 https!)
    - host: vercel-internal (不是 acme.cal.com!)
```

**影响**：
1. 如果应用使用 `req.protocol` 构建 redirect_uri，会是 `http://`
2. 如果应用使用 `req.get('host')`，会是内部域名
3. 这会导致 OAuth 流程中的 URL 不一致

#### 3.2.2 授权阶段的 Redirect URI 校验

```typescript
// OAuthService.ts
async getClientForAuthorization(
  clientId: string,
  redirectUri: string,
  userId?: number
): Promise<OAuth2Client> {
  const client = await this.oAuthClientRepository.findByClientId(clientId);

  if (!client) {
    throw new ErrorWithCode(ErrorCode.NotFound, "unauthorized_client", ...);
  }

  // 严格校验 redirect_uri
  this.validateRedirectUri(client.redirectUri, redirectUri);

  // ...
}
```

**RFC 6749 引用**：

```
RFC 6749, 4.1.2.1:
If the request fails due to a missing, invalid, or mismatching
redirection URI, the authorization server SHOULD inform the resource
owner of the error and MUST NOT automatically redirect the user-agent
to the invalid redirection URI.
```

#### 3.2.3 Token 交换阶段的校验

```typescript
// OAuthService.ts
async exchangeCodeForTokens(
  clientId: string,
  code: string,
  clientSecret?: string,
  redirectUri?: string,  // 可选，但如果提供必须匹配
  codeVerifier?: string
): Promise<OAuth2Tokens> {
  const client = await this.oAuthClientRepository.findByClientIdWithSecret(clientId);
  
  // RFC 6749 5.2: Token 交换阶段的 redirect_uri 不匹配返回 'invalid_grant'
  if (redirectUri && client.redirectUri !== redirectUri) {
    throw new ErrorWithCode(
      ErrorCode.BadRequest, 
      "invalid_grant", 
      { reason: "redirect_uri_mismatch" }
    );
  }
  // ...
}
```

### 3.3 登录回调的安全校验

**核心文件**: `packages/features/auth/lib/next-auth-options.ts`

#### 3.3.1 NextAuth redirect 回调

```typescript
// next-auth-options.ts
callbacks: {
  async redirect({ url, baseUrl }) {
    // 规则 1: 允许相对路径（自动添加 baseUrl）
    if (url.startsWith("/")) return `${baseUrl}${url}`;
    
    // 规则 2: 允许同域名的绝对 URL
    // 关键：比较的是 hostname，不是完整 URL
    else if (new URL(url).hostname === new URL(WEBAPP_URL).hostname) return url;
    
    // 规则 3: 其他情况返回首页（防止开放重定向）
    return baseUrl;
  },
}
```

**关键分析**：

```typescript
// 假设 WEBAPP_URL = "https://app.cal.com"

// 场景 1: 主域名登录后跳转
url = "https://app.cal.com/settings"
hostname = "app.cal.com"
WEBAPP_URL.hostname = "app.cal.com"
→ ✅ 匹配，允许跳转

// 场景 2: 组织子域名登录后跳转
url = "https://acme.cal.com/settings"
hostname = "acme.cal.com"
WEBAPP_URL.hostname = "app.cal.com"
→ ❌ 不匹配，被重定向到 app.cal.com！

// 场景 3: 完全自定义域名
url = "https://cal.company.com/settings"
hostname = "cal.company.com"
WEBAPP_URL.hostname = "app.cal.com"
→ ❌ 不匹配，被重定向到 app.cal.com！
```

**这是一个严重问题**：组织子域名和自定义域名的用户登录后会被重定向回主域名！

#### 3.3.2 更深层的安全校验

**核心文件**: `packages/lib/getSafeRedirectUrl.ts`

```typescript
export const getSafeRedirectUrl = (url = "") => {
  if (!url) {
    return null;
  }

  // 强制要求绝对 URL
  if (url.search(/^https?:\/\//) === -1) {
    throw new Error("Pass an absolute URL");
  }

  const urlParsed = new URL(url);

  // 严格白名单校验
  // 只允许 CONSOLE_URL, WEBAPP_URL, WEBSITE_URL 这三个域名
  if (![CONSOLE_URL, WEBAPP_URL, WEBSITE_URL].some(
    (u) => new URL(u).origin === urlParsed.origin
  )) {
    url = `${WEBAPP_URL}/`;  // 不在白名单？强制重定向到首页
  }

  return url;
};
```

**白名单机制的问题**：

```typescript
// 假设配置：
WEBAPP_URL = "https://app.cal.com"
WEBSITE_URL = "https://cal.com"
CONSOLE_URL = "https://console.cal.com"

// 允许的 origin：
// - https://app.cal.com
// - https://cal.com
// - https://console.cal.com

// 不允许的 origin（即使是合法组织）：
// - https://acme.cal.com     ❌ 不在白名单
// - https://cal.company.com   ❌ 不在白名单
```

#### 3.3.3 资源加载的安全校验

```typescript
// getSafeRedirectUrl.ts
export function isSafeUrlToLoadResourceFrom(urlString: string) {
  try {
    const url = new URL(urlString);
    if (url.protocol !== "http:" && url.protocol !== "https:") {
      return false;
    }

    // 允许 localhost（开发环境）
    if (url.hostname === "localhost" || url.hostname === "127.0.0.1") {
      return true;
    }

    const webappUrl = new URL(WEBAPP_URL);
    const embedLibUrl = new URL(EMBED_LIB_URL);

    // 关键：使用 TLD+1 比较，允许子域名
    const urlTldPlus1 = getTldPlus1(url.hostname);
    const webappTldPlus1 = getTldPlus1(webappUrl.hostname);
    const embedLibTldPlus1 = getTldPlus1(embedLibUrl.hostname);

    // 允许同 TLD+1 的所有子域名
    return [webappTldPlus1, embedLibTldPlus1].includes(urlTldPlus1);
  } catch {
    return false;
  }
}
```

**TLD+1 比较的含义**：

```typescript
// 假设 WEBAPP_URL = "https://app.cal.com"
// getTldPlus1("app.cal.com") = "cal.com"

// 允许的 URL：
// - https://acme.cal.com/script.js    → TLD+1 = "cal.com" ✅
// - https://app.cal.com/script.js      → TLD+1 = "cal.com" ✅
// - https://cal.com/script.js           → TLD+1 = "cal.com" ✅

// 不允许的 URL：
// - https://cal.company.com/script.js   → TLD+1 = "company.com" ❌
// - https://malicious.com/script.js     → TLD+1 = "malicious.com" ❌
```

**两种校验方式的对比**：

| 校验函数 | 比较方式 | 组织子域名 | 自定义域名 | 用途 |
|----------|----------|-----------|-----------|------|
| `getSafeRedirectUrl` | **精确 origin 匹配** | ❌ 不允许 | ❌ 不允许 | 登录回调、URL 重定向 |
| `isSafeUrlToLoadResourceFrom` | **TLD+1 匹配** | ✅ 允许 | ❌ 不允许 | 资源加载、Embed |

**不一致性问题**：
- 资源加载允许组织子域名（`acme.cal.com`）
- 但重定向不允许组织子域名
- 完全自定义域名（`cal.company.com`）在两种方式下都不允许

### 3.4 NextAuth 回调 URL 白名单

**核心配置**: `apps/web/next.config.ts`

```typescript
// 防止开放重定向攻击的配置
{
  source: "/api/auth/:path*",
  has: [
    {
      type: "query",
      key: "callbackUrl",
      value: "^(?!https?://).*$",  // 匹配不以 http:// 或 https:// 开头的 URL
    },
  ],
  destination: "/404",
  permanent: false,
}
```

**规则解析**：

```
正则: ^(?!https?://).*$

含义：
^              - 字符串开始
(?!https?://)  - 负向前瞻：后面不是 "http://" 或 "https://"
.*             - 匹配任意字符

效果：
- "https://app.cal.com/settings"  → 不匹配 → 允许通过
- "/settings"                      → 匹配 → 返回 404
- "//evil.com"                     → 匹配 → 返回 404
- "javascript:alert(1)"            → 匹配 → 返回 404
```

**安全意义**：防止通过协议相对 URL（`//evil.com`）或数据 URL 进行开放重定向攻击。

### 3.5 反向代理场景的完整影响

#### 3.5.1 协议头丢失

```
场景: 用户通过 HTTPS 访问，但反向代理用 HTTP 转发到应用

请求:
  用户 → https://acme.cal.com/api/auth/signin
         ↓
    Cloudflare (SSL 终止)
         ↓
    转发 → http://vercel-internal/api/auth/signin
         ↓
    Next.js 应用:
      - req.protocol = "http"
      - req.secure = false
      - x-forwarded-proto = "https" (可能被设置)
```

**对 Cookie 的影响**：

```typescript
// default-cookies.ts
export function defaultCookies(useSecureCookies: boolean): CookiesOptions {
  const cookiePrefix = useSecureCookies ? "__Secure-" : "";
  
  const defaultOptions: CookieOption["options"] = {
    domain: NEXTAUTH_COOKIE_DOMAIN || undefined,
    sameSite: useSecureCookies ? "none" : "lax",  // 关键！
    path: "/",
    secure: useSecureCookies,  // 关键！
  };
  // ...
}

// next-auth-options.ts
cookies: defaultCookies(WEBAPP_URL?.startsWith("https://")),
```

**问题**：
- `useSecureCookies` 是根据 `WEBAPP_URL` 静态判断的，不是根据实际请求
- 如果 `WEBAPP_URL = "https://app.cal.com"`，但实际请求是 HTTP（被代理转发）
- Cookie 会设置 `Secure` 标志，但浏览器通过 HTTP 接收时会忽略

**解决方案**：配置 Next.js 信任代理

```typescript
// next.config.js (需要添加)
export default {
  trustHostHeader: true,  // 信任 X-Forwarded-Host
  // 或使用 experimental 配置
  experimental: {
    trustHost: true,
  },
};
```

#### 3.5.2 Host 头修改

```
场景: 反向代理修改 Host 头

用户请求 Host: acme.cal.com
         ↓
    反向代理配置:
      proxy_set_header Host vercel-internal;
         ↓
    应用收到 Host: vercel-internal
```

**影响**：
1. Next.js `nextUrl.host` 返回内部域名
2. 组织路由 Rewrite 规则失效（因为 Host 不匹配）
3. OAuth `redirect_uri` 构建使用错误的域名

**正确的代理配置**：

```nginx
# 推荐配置：保留原始 Host 头
proxy_set_header Host $host;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

# 不推荐：修改 Host 头
# proxy_set_header Host internal-domain;
```

#### 3.5.3 端口号问题

```
场景: 反向代理在不同端口终止

用户访问: https://acme.cal.com (端口 443，默认 HTTPS)
         ↓
    反向代理转发: http://localhost:3000 (端口 3000)
         ↓
    应用构建 redirect_uri: http://localhost:3000/api/oauth/callback
         ↓
    与注册的 https://acme.cal.com/api/oauth/callback 不匹配！
```

### 3.6 组织域名下 OAuth 集成的挑战

#### 3.6.1 第三方应用集成的回调 URL

当用户在组织域名下连接 Google Calendar 等服务时：

```
当前实现（硬编码）:
- OAuth 回调 URL 配置在环境变量或数据库中
- 通常配置为: https://app.cal.com/api/integrations/googlecalendar/callback

问题:
1. 用户在 acme.cal.com 点击"连接 Google"
2. 被重定向到 Google OAuth
3. Google 回调到 app.cal.com（配置的 URL）
4. 授权成功后，用户被重定向回 acme.cal.com？
5. Cookie 和会话状态如何跨域传递？
```

**当前代码中的开发环境特殊处理**：

```typescript
// next.config.ts
...(process.env.NODE_ENV === "development" &&
isOrganizationsEnabled &&
process.env.NEXT_PUBLIC_WEBAPP_URL !== "http://localhost:3000"
  ? [
      {
        has: [
          { type: "header", key: "host", value: "localhost:3000" },
        ],
        source: "/api/integrations/:args*",
        // 开发环境：将 localhost 的集成请求代理到主域名
        destination: `${process.env.NEXT_PUBLIC_WEBAPP_URL}/api/integrations/:args*`,
        permanent: false,
      },
    ]
  : []),
```

**这说明**：开发环境确实存在跨域回调问题，需要特殊代理配置。

#### 3.6.2 建议的 OAuth 回调架构

**方案 A：主域名统一处理（推荐）**

```
流程:
1. 用户在 acme.cal.com 点击 "连接 Google Calendar"
2. 前端构建 state 参数，包含 orgSlug: "acme"
3. 重定向到:
   https://accounts.google.com/o/oauth2/v2/auth
     ?client_id=xxx
     &redirect_uri=https://app.cal.com/api/integrations/googlecalendar/callback
     &state=orgSlug=acme&csrf=xxx
4. Google 回调到 app.cal.com
5. 后端验证 state，提取 orgSlug
6. 保存凭证，关联到正确的组织/用户
7. 重定向回: https://acme.cal.com/settings/integrations
```

**关键实现点**：

```typescript
// app-store/_utils/oauth/encodeOAuthState.ts
// 需要在 state 中编码 orgSlug

// 回调处理时:
// 1. 解码 state，获取 orgSlug
// 2. 将 credential 保存到对应用户
// 3. 重定向回 orgSlug 对应的域名
```

**方案 B：通配符域名（限制较多）**

```
配置:
- 在 Google Cloud Console 配置通配符 redirect_uri:
  https://*.cal.com/api/integrations/googlecalendar/callback

限制:
- 不是所有 OAuth 提供商都支持通配符
- 完全自定义域名（cal.company.com）无法使用
- 安全性稍差（任意子域名都可回调）
```

---

## 4. Cookie 作用域与会话连续性

### 4.1 Cookie 配置详解

**核心文件**: `packages/lib/default-cookies.ts`

```typescript
const NEXTAUTH_COOKIE_DOMAIN = process.env.NEXTAUTH_COOKIE_DOMAIN || "";

export function defaultCookies(useSecureCookies: boolean): CookiesOptions {
  const cookiePrefix = useSecureCookies ? "__Secure-" : "";

  const defaultOptions: CookieOption["options"] = {
    domain: NEXTAUTH_COOKIE_DOMAIN || undefined,  // 关键！
    sameSite: useSecureCookies ? "none" : "lax",   // 关键！
    path: "/",
    secure: useSecureCookies,                       // 关键！
  };
  
  return {
    sessionToken: {
      name: `${cookiePrefix}next-auth.session-token`,
      options: {
        ...defaultOptions,
        httpOnly: true,  // 关键！防止 XSS
      },
    },
    callbackUrl: {
      name: `${cookiePrefix}next-auth.callback-url`,
      options: defaultOptions,
    },
    csrfToken: {
      name: `${cookiePrefix}next-auth.csrf-token`,
      options: {
        ...defaultOptions,
        httpOnly: true,
      },
    },
    pkceCodeVerifier: {
      name: `${cookiePrefix}next-auth.pkce.code_verifier`,
      options: {
        ...defaultOptions,
        httpOnly: true,
      },
    },
    state: {
      name: `${cookiePrefix}next-auth.state`,
      options: {
        ...defaultOptions,
        httpOnly: true,
      },
    },
    nonce: {
      name: `${cookiePrefix}next-auth.nonce`,
      options: {
        httpOnly: true,
        sameSite: "lax",  // 注意：nonce 使用 lax
        path: "/",
        secure: useSecureCookies,
      },
    },
  };
}
```

### 4.2 Cookie 属性深度分析

#### 4.2.1 Domain 属性

| 配置值 | Cookie 作用域 | 可见域名 |
|--------|--------------|----------|
| `undefined`（默认） | 当前域名（精确匹配）| 仅设置 Cookie 的域名 |
| `".cal.com"` | 根域名 + 所有子域名 | `cal.com`, `app.cal.com`, `acme.cal.com` |
| `".company.com"` | 自定义域名 | `cal.company.com`, `booking.company.com` |

**默认行为的问题**：

```typescript
// 默认配置：NEXTAUTH_COOKIE_DOMAIN 未设置
// domain: undefined

场景:
1. 用户在 app.cal.com 登录
2. Cookie 设置: Set-Cookie: next-auth.session-token=xxx; path=/; HttpOnly
3. 这个 Cookie 的 domain 隐式为 "app.cal.com"（精确匹配）
4. 用户访问 acme.cal.com
5. 浏览器不会发送 app.cal.com 的 Cookie 到 acme.cal.com
6. 用户在 acme.cal.com 显示为未登录！
```

**设置根域名的效果**：

```typescript
// 配置: NEXTAUTH_COOKIE_DOMAIN=.cal.com

场景:
1. 用户在 app.cal.com 登录
2. Cookie 设置: Set-Cookie: next-auth.session-token=xxx; domain=.cal.com; path=/; HttpOnly
3. 用户访问 acme.cal.com
4. 浏览器发送 .cal.com 域的 Cookie
5. 用户在 acme.cal.com 保持登录状态！
```

#### 4.2.2 SameSite 属性

| SameSite 值 | HTTPS | HTTP | 跨站发送 | 说明 |
|-------------|-------|------|----------|------|
| `"strict"` | 可设置 | 可设置 | ❌ 否 | 最严格，仅同站 |
| `"lax"` | 可设置 | 可设置 | ⚠️ 部分 | 顶级导航允许 |
| `"none"` | ✅ 需同时设置 `Secure` | ❌ 浏览器拒绝 | ✅ 是 | 允许跨站 |

**Cal.diy 的配置逻辑**：

```typescript
// default-cookies.ts
sameSite: useSecureCookies ? "none" : "lax"

// next-auth-options.ts
cookies: defaultCookies(WEBAPP_URL?.startsWith("https://"))
```

**问题分析**：

```typescript
// 场景 1: SaaS 平台，WEBAPP_URL=https://app.cal.com
useSecureCookies = true
sameSite = "none"
secure = true

// 这是正确的配置，因为:
// - 嵌入场景（iframe）需要 SameSite=None
// - 跨域 OAuth 回调需要 SameSite=None

// 场景 2: 开发环境，WEBAPP_URL=http://localhost:3000
useSecureCookies = false
sameSite = "lax"
secure = false

// 这也是正确的，因为:
// - HTTP 不允许 SameSite=None
// - 开发环境通常不涉及跨站嵌入
```

**SameSite=None 的特殊要求**：

```
RFC 6265bis:
- SameSite=None 必须同时设置 Secure 属性
- 否则浏览器会拒绝该 Cookie
- 仅在 HTTPS 下有效

影响:
1. HTTP 开发环境无法使用 SameSite=None
2. 反向代理如果用 HTTP 转发，即使前端是 HTTPS 也会有问题
3. 需要正确配置 X-Forwarded-Proto 和信任代理
```

#### 4.2.3 Secure 属性

| Secure 值 | 协议 | Cookie 行为 |
|-----------|------|------------|
| `true` | HTTPS | ✅ 浏览器发送 |
| `true` | HTTP | ❌ 浏览器忽略 |
| `false` | HTTPS | ✅ 浏览器发送（不推荐）|
| `false` | HTTP | ✅ 浏览器发送 |

**反向代理场景的问题**：

```
用户访问: https://acme.cal.com (HTTPS)
         ↓
    反向代理转发: http://localhost:3000 (HTTP)
         ↓
    应用判断: WEBAPP_URL=https://app.cal.com
         ↓
    useSecureCookies = true
         ↓
    Set-Cookie: session-token=xxx; Secure; SameSite=None
         ↓
    浏览器通过 HTTP 收到带 Secure 标志的 Cookie
         ↓
    浏览器忽略该 Cookie！
```

**解决方案**：

```typescript
// 方案 1: 让应用知道真实协议
// 配置 Next.js 信任 X-Forwarded-Proto

// 方案 2: 反向代理负责添加 Cookie
// 让反向代理（如 Nginx）在转发时修改 Cookie 属性

// 方案 3: 确保整个链路都是 HTTPS
// Vercel 默认就是 HTTPS，Cloudflare 代理也会保持 HTTPS
```

#### 4.2.4 HttpOnly 属性

| HttpOnly | JavaScript 访问 | XSS 防护 |
|----------|----------------|----------|
| `true` | ❌ 无法通过 `document.cookie` 访问 | ✅ 有效防护 |
| `false` | ✅ 可以访问 | ❌ 易受 XSS 攻击 |

**Cal.diy 的配置**：

```typescript
sessionToken: {
  options: {
    httpOnly: true,  // ✅ 会话 Token 不可被 JS 访问
  },
},
csrfToken: {
  options: {
    httpOnly: true,  // ✅ CSRF Token 也不可被 JS 访问
  },
},
// 等等...
```

**例外情况**：

```typescript
callbackUrl: {
  name: `${cookiePrefix}next-auth.callback-url`,
  options: {
    // 注意：没有设置 httpOnly！
    // 这是设计意图，因为某些场景下前端需要读取
  },
},
```

### 4.3 Cookie 前缀 (__Secure-)

**核心逻辑**：

```typescript
const cookiePrefix = useSecureCookies ? "__Secure-" : "";

// HTTPS 环境:
// sessionToken cookie 名: "__Secure-next-auth.session-token"

// HTTP 环境:
// sessionToken cookie 名: "next-auth.session-token"
```

**__Secure- 前缀的意义**：

```
Cookie 前缀规范 (RFC 草稿):

1. __Secure- 前缀
   - 必须同时设置 Secure 属性
   - 必须来自安全来源（HTTPS）
   - 防止 HTTP 环境覆盖 HTTPS Cookie

2. __Host- 前缀（更严格，Cal.diy 未使用）
   - 必须同时设置 Secure 属性
   - 不能设置 Domain 属性（只能是当前域名）
   - Path 必须是 "/"
```

**安全意义**：

```
攻击场景（无 __Secure- 前缀）:
1. 用户通过 HTTPS 登录 app.cal.com
2. Cookie: next-auth.session-token=xxx; Secure; HttpOnly
3. 攻击者通过某种方式让用户访问 http://app.cal.com（HTTP）
4. 攻击者设置 Cookie: next-auth.session-token=evil; HttpOnly
5. 虽然浏览器不会通过 HTTP 发送 Secure Cookie
6. 但它可以覆盖同名 Cookie（没有 __Secure- 前缀时）

防护场景（有 __Secure- 前缀）:
1. 用户通过 HTTPS 登录 app.cal.com
2. Cookie: __Secure-next-auth.session-token=xxx; Secure; HttpOnly
3. 攻击者尝试在 HTTP 环境设置 __Secure- 前缀的 Cookie
4. 浏览器拒绝！因为 __Secure- 前缀要求 Secure 和 HTTPS
5. 攻击失败
```

### 4.4 会话连续性的挑战

#### 4.4.1 跨子域名会话

| 配置 | 会话连续性 | 说明 |
|------|-----------|------|
| 无 `NEXTAUTH_COOKIE_DOMAIN` | ❌ 跨子域名不共享 | 每个子域名独立登录 |
| `NEXTAUTH_COOKIE_DOMAIN=.cal.com` | ✅ 所有子域名共享 | 一次登录，全站通行 |

**实际影响示例**：

```typescript
// 场景：用户从主应用切换到组织子域名

// 配置 1: 无 NEXTAUTH_COOKIE_DOMAIN
1. 用户在 app.cal.com 登录
2. 点击链接跳转到 acme.cal.com
3. acme.cal.com 没有收到会话 Cookie
4. 用户显示为未登录，需要重新登录

// 配置 2: NEXTAUTH_COOKIE_DOMAIN=.cal.com
1. 用户在 app.cal.com 登录
2. Cookie 设置到 .cal.com 域
3. 点击链接跳转到 acme.cal.com
4. acme.cal.com 收到 .cal.com 的会话 Cookie
5. 用户保持登录状态 ✅
```

#### 4.4.2 完全自定义域名的会话

**问题**：完全自定义域名（如 `cal.company.com`）无法通过 Cookie Domain 配置与主域名共享会话。

```
架构:
- 主平台: app.cal.com
- 企业自定义域名: cal.company.com

Cookie 限制:
- .cal.com 的 Cookie 不会发送到 cal.company.com（不同 TLD+1）
- 浏览器的同源策略阻止跨域 Cookie 访问

结果:
- 用户需要在每个域名单独登录
- 或者需要实现其他会话同步机制
```

**解决方案**：

| 方案 | 实现复杂度 | 用户体验 | 安全性 |
|------|-----------|----------|--------|
| **单点登录 (SSO)** | 高 | ✅ 一次登录 | ✅ 高 |
| **JWT 在 URL 中传递** | 中 | ⚠️ 需要跳转 | ⚠️ 需防止泄露 |
| **iframe postMessage** | 中 | ✅ 无缝 | ⚠️ 需处理 X-Frame-Options |
| **独立会话（当前实现）** | 低 | ❌ 需多次登录 | ✅ 高 |

#### 4.4.3 反向代理后的会话问题

```
场景: Cloudflare 代理 → Vercel

问题 1: 协议不匹配
- 用户: HTTPS
- 代理到源站: HTTP
- 源站设置 Secure Cookie
- 浏览器通过 HTTP 响应收到？不，浏览器看到的是 Cloudflare 的 HTTPS

注意: Cloudflare 到浏览器是 HTTPS，所以 Cookie 的 Secure 属性是有效的
```

```
问题 2: 缓存导致 Cookie 问题
- 反向代理可能缓存某些响应
- 如果缓存了带 Set-Cookie 的响应，可能导致会话污染

解决方案:
- 确保带 Cookie 的响应不被缓存
- 设置正确的 Cache-Control 头
```

### 4.5 组织上下文中的用户识别

**核心文件**: `packages/features/auth/lib/next-auth-options.ts`

#### 4.5.1 JWT 中的组织信息

```typescript
// next-auth-options.ts - JWT 回调
async jwt({ token, user, account, trigger, session }) {
  // ...
  
  // 登录成功后，JWT 中包含组织信息
  return {
    ...existingUserWithoutTeamsField,
    ...token,
    profileId: profile.id,
    upId,
    // ...
    org: profileOrg && !profileOrg.isPlatform
      ? {
          id: profileOrg.id,
          name: profileOrg.name,
          slug: profileOrg.slug ?? "",
          logoUrl: profileOrg.logoUrl,
          fullDomain: WEBAPP_URL,  // 注意：这里是 WEBAPP_URL，不是当前域名！
          domainSuffix: "",
          role: orgRole as MembershipRole,
        }
      : null,
  } as JWT;
}
```

**问题分析**：

```typescript
// 注意这行：
fullDomain: WEBAPP_URL,

// 假设 WEBAPP_URL = "https://app.cal.com"
// 用户在 acme.cal.com 登录

// JWT 中存储的 fullDomain 是 "https://app.cal.com"
// 而不是用户实际访问的 "https://acme.cal.com"

// 这可能导致：
// 1. 构建链接时使用错误的域名
// 2. 某些校验逻辑失败
```

#### 4.5.2 Profile 与 User 的关系

```
数据模型:
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│    User     │         │   Profile   │         │    Team     │
│  (全局用户)  │◄───────►│(组织内身份) │◄───────►│  (组织/团队) │
├─────────────┤         ├─────────────┤         ├─────────────┤
│ id          │         │ id          │         │ id          │
│ email       │         │ userId      │         │ name        │
│ username    │         │ username    │         │ slug        │
│ ...         │         │ organization│         │ isOrganization│
└─────────────┘         │     Id      │         └─────────────┘
                        └─────────────┘

规则:
- 一个 User 可以有多个 Profile（在不同组织中）
- 每个 Profile 有独立的 username
- Profile 通过 organizationId 关联到 Team（isOrganization=true）
- 一个 User 在一个 Organization 中只有一个 Profile
```

**实际查询逻辑**：

```typescript
// packages/lib/server/username.ts
const usernameCheck = async (usernameRaw: string, currentOrgDomain?: string | null) => {
  // ...
  
  let organizationId: number | null = null;
  if (currentOrgDomain) {
    // 通过 orgSlug 查找组织
    const organization = await prisma.team.findFirst({
      where: {
        isOrganization: true,
        OR: [
          { slug: currentOrgDomain },
          { metadata: { path: ["requestedSlug"], equals: currentOrgDomain } }
        ],
      },
      select: { id: true },
    });
    // ...
    organizationId = organization.id;
  }

  // 在组织上下文中查找用户
  const user = await prisma.user.findFirst({
    where: {
      username,
      organizationId: organizationId ?? null,  // 关键：区分全局和组织上下文
    },
    select: { id: true, username: true },
  });
  // ...
};
```

### 4.6 Cookie 配置建议

#### 4.6.1 SaaS 多租户部署

```bash
# 推荐配置
NEXT_PUBLIC_WEBAPP_URL=https://app.cal.com
NEXTAUTH_COOKIE_DOMAIN=.cal.com

# 效果:
# - 用户在 app.cal.com 登录
# - Cookie 设置到 .cal.com 域
# - acme.cal.com, dunder.cal.com 等子域名都可访问
# - 实现跨子域名单点登录
```

#### 4.6.2 单组织自托管部署

```bash
# 方案 A: 使用自有域名（推荐）
NEXT_PUBLIC_WEBAPP_URL=https://cal.company.com
NEXT_PUBLIC_SINGLE_ORG_SLUG=company
# 不需要 NEXTAUTH_COOKIE_DOMAIN（默认即可）

# 方案 B: 多子域名访问同一组织
NEXT_PUBLIC_WEBAPP_URL=https://app.company.com
NEXT_PUBLIC_SINGLE_ORG_SLUG=company
NEXTAUTH_COOKIE_DOMAIN=.company.com
# 这样 cal.company.com, booking.company.com 都可共享会话
```

#### 4.6.3 开发环境配置

```bash
# 本地开发
NEXT_PUBLIC_WEBAPP_URL=http://localhost:3000
# 不需要 NEXTAUTH_COOKIE_DOMAIN
# 注意：开发环境 useSecureCookies=false，SameSite=lax
```

---

## 5. 环境变量配置参考

### 5.1 完整配置清单

| 环境变量 | 类型 | 必填 | 说明 |
|----------|------|------|------|
| **核心配置** | | | |
| `NEXT_PUBLIC_WEBAPP_URL` | URL | ✅ | 主应用 URL，影响 Cookie Secure 判断 |
| `NEXTAUTH_URL` | URL | ✅ | NextAuth 端点 URL |
| `NEXTAUTH_SECRET` | String | ✅ | NextAuth 加密密钥（≥32 字符）|
| `CALENDSO_ENCRYPTION_KEY` | String | ✅ | 通用加密密钥（≥32 字符）|
| **组织功能** | | | |
| `ORGANIZATIONS_ENABLED` | 0/1 | ❌ | 启用组织功能 |
| `NEXT_PUBLIC_SINGLE_ORG_SLUG` | String | ❌ | 单组织模式的组织 Slug |
| `RESERVED_SUBDOMAINS` | 逗号分隔 | ❌ | 保留子域名列表（如 `app,www,api`）|
| **Cookie 配置** | | | |
| `NEXTAUTH_COOKIE_DOMAIN` | Domain | ❌ | Cookie 作用域（如 `.cal.com`）|
| **Vercel 集成** | | | |
| `PROJECT_ID_VERCEL` | String | ❌ | Vercel 项目 ID |
| `TEAM_ID_VERCEL` | String | ❌ | Vercel 团队 ID |
| `AUTH_BEARER_TOKEN_VERCEL` | String | ❌ | Vercel API Token |
| `VERCEL_URL` | URL |