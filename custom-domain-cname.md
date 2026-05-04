# Cal.diy 企业用户自定义域名绑定指南

本文档详细说明 Cal.diy 企业用户绑定自有域名的技术实现，包括 Host 路由机制、SSL 证书管理、以及反向代理对 OAuth 回调和 Cookie 的影响。

---

## 目录

1. [Host 路由到组织的机制](#1-host-路由到组织的机制)
2. [SSL 证书申请与自动续期](#2-ssl-证书申请与自动续期)
3. [反向代理对 OAuth 回调和 Cookie 的影响](#3-反向代理对-oauth-回调和-cookie-的影响)
4. [环境变量配置](#4-环境变量配置)

---

## 1. Host 路由到组织的机制

### 1.1 架构概述

Cal.diy 使用 **Next.js Rewrites** 机制实现基于 Host 头的组织路由。当企业用户绑定自定义域名（如 `cal.acme.com`）后，系统会自动将该域名的请求路由到对应的组织。

### 1.2 核心实现

路由配置位于 `apps/web/next.config.ts` 和 `apps/web/getNextjsOrgRewriteConfig.ts`。

#### 正则匹配规则

```typescript
// getNextjsOrgRewriteConfig.ts
export const getRegExpThatMatchesAllOrgDomains = ({ webAppUrl }: { webAppUrl: string }): string => {
  if (isSingleOrgModeEnabled) {
    return `.*`;
  }
  const subdomainRegExp = getRegExpNotMatchingLeftMostSubdomain(webAppUrl);
  return `^(?<${orgSlugCaptureGroupName}>${subdomainRegExp})\\.(?!vercel\\.app).*`;
};
```

**匹配逻辑说明：**

| 场景 | 示例域名 | 匹配结果 | orgSlug |
|------|----------|----------|---------|
| 主应用域名 | `app.cal.com` | 不匹配（排除 `app` 子域名） | - |
| 组织子域名 | `acme.cal.com` | 匹配 | `acme` |
| 自定义域名 | `cal.acme.com` | 匹配（单组织模式） | 配置值 |

#### Rewrite 配置

```typescript
// next.config.ts - 核心重写规则
const orgDomainMatcherConfig = {
  user: {
    has: [{ type: "host", value: nextJsOrgRewriteConfig.orgHostPath }],
    source: orgUserRoutePath, // /:user
    destination: `/org/${orgSlug}/:user`,
  },
  userType: {
    has: [{ type: "host", value: nextJsOrgRewriteConfig.orgHostPath }],
    source: orgUserTypeRoutePath, // /:user/:type
    destination: `/org/${orgSlug}/:user/:type`,
  },
  root: {
    has: [{ type: "host", value: nextJsOrgRewriteConfig.orgHostPath }],
    source: "/",
    destination: `/team/${orgSlug}?isOrgProfile=1`,
  },
};
```

### 1.3 路由流程图

```
用户请求: https://acme.cal.com/john/30min
                    ↓
         Next.js Rewrite 规则匹配
         Host 头: acme.cal.com
         提取 orgSlug: acme
                    ↓
         内部重写到: /org/acme/john/30min
                    ↓
         添加 X-Cal-Org-path 响应头
         值: /org/acme/john/30min
                    ↓
         页面组件读取 orgSlug 参数
         加载对应组织的配置和数据
```

### 1.4 单组织模式 vs 多组织模式

| 模式 | 环境变量 | 路由行为 | 适用场景 |
|------|----------|----------|----------|
| **单组织模式** | `NEXT_PUBLIC_SINGLE_ORG_SLUG=acme` | 所有域名都路由到指定组织 | 自托管部署、单一企业使用 |
| **多组织模式** | 未设置 `NEXT_PUBLIC_SINGLE_ORG_SLUG` | 按子域名匹配组织 | SaaS 平台、多租户 |

**单组织模式下的域名处理：**

```typescript
// 单组织模式下，所有域名都被视为组织域名
// next.config.ts 中的 redirects 配置示例
{
  source: "/support",
  missing: [
    {
      type: "header",
      key: "host",
      value: nextJsOrgRewriteConfig.orgHostPath, // 在单组织模式下为 ".*"
    },
  ],
  destination: "/event-types?openSupport=true",
  permanent: true,
}
```

### 1.5 组织重定向处理

当用户从旧域名迁移到组织时，系统支持临时重定向：

```typescript
// apps/web/lib/handleOrgRedirect.ts
export async function handleOrgRedirect({
  slugs,
  redirectType,
  eventTypeSlug,
  context,
  currentOrgDomain,
}: HandleOrgRedirectParams) {
  // 示例: 用户 "john87" 加入组织 "acme" 后
  // 旧链接: cal.com/john87
  // 重定向到: acme.cal.com/john
  
  const redirects = await prisma.tempOrgRedirect.findMany({
    where: {
      type: redirectType, // User 或 Team
      from: { in: slugs },
      fromOrgId: 0, // 从全局命名空间迁移
    },
  });
  
  // 返回重定向配置
  return {
    redirect: {
      permanent: false,
      destination: `${newOrigin}${newPath}${query}`,
    },
  };
}
```

---

## 2. SSL 证书申请与自动续期

### 2.1 架构概览

Cal.diy 使用 **双层架构** 管理自定义域名：

```
┌─────────────────────────────────────────────────────────────┐
│                    用户自定义域名                              │
│              (如: cal.acme.com, booking.example.com)          │
└─────────────────────────┬───────────────────────────────────┘
                          │ CNAME 记录
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Cloudflare DNS                            │
│         - 管理 CNAME 记录指向 Vercel 部署                     │
│         - 可选：Cloudflare 代理 (CDN + WAF)                  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Vercel 平台                                │
│         - 自动申请 Let's Encrypt SSL 证书                     │
│         - 自动续期 (到期前 30 天)                            │
│         - 自动部署到边缘节点                                   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 域名创建流程

代码位于 `packages/lib/domainManager/`

#### 2.2.1 组织域名创建

```typescript
// packages/lib/domainManager/organization.ts
export const createDomain = async (slug: string) => {
  const domain = `${slug}.${subdomainSuffix()}`; // 如: acme.cal.com
  
  let domainConfigured = false;
  let dnsConfigured = true;

  // 1. 在 Vercel 中添加域名（触发 SSL 证书申请）
  if (process.env.VERCEL_URL) {
    domainConfigured = await createVercelDomain(domain);
  }

  // 2. 在 Cloudflare 中添加 CNAME 记录
  if (process.env.CLOUDFLARE_DNS) {
    dnsConfigured = await addDnsRecord(domain);
  }

  return domainConfigured && dnsConfigured;
};
```

#### 2.2.2 Vercel 域名管理

```typescript
// packages/lib/domainManager/deploymentServices/vercel.ts
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

  // Vercel 会自动:
  // 1. 验证域名所有权
  // 2. 申请 Let's Encrypt SSL 证书
  // 3. 配置自动续期
  // 4. 部署到全球边缘节点
};
```

#### 2.2.3 Cloudflare DNS 管理

```typescript
// packages/lib/domainManager/deploymentServices/cloudflare.ts
export const addDnsRecord = async (domain: string) => {
  const data = await api(
    `${cloudflareApiForZoneUrl}/dns_records`,
    {
      method: "POST",
      body: JSON.stringify({
        type: "CNAME",
        proxied: IS_RECORD_PROXIED, // true: 通过 Cloudflare 代理
        name: domain,
        content: process.env.CLOUDFLARE_VERCEL_CNAME, // 如: cname.vercel-dns.com
        ttl: AUTOMATIC_TTL, // 1 = 自动
      }),
    },
    cloudflareDnsRecordApiResponseSchema
  );
};
```

### 2.3 SSL 证书自动续期机制

| 组件 | 证书提供方 | 自动续期 | 续期时机 |
|------|-----------|----------|----------|
| **Vercel 管理的域名** | Let's Encrypt | ✅ 自动 | 到期前 30 天 |
| **Cloudflare 代理 (橙色云)** | Cloudflare Universal SSL | ✅ 自动 | 持续管理 |
| **Cloudflare 不代理 (灰色云)** | Let's Encrypt (通过 Vercel) | ✅ 自动 | 到期前 30 天 |

**Vercel SSL 证书特性：**

1. **自动申请**：添加域名后自动触发证书申请
2. **自动验证**：通过 DNS 或 HTTP 验证域名所有权
3. **自动续期**：证书到期前 30 天自动续期
4. **多域名支持**：SAN 证书支持多个域名
5. **通配符证书**：需要额外配置

### 2.4 域名删除流程

```typescript
export const deleteDomain = async (slug: string) => {
  const domain = `${slug}.${subdomainSuffix()}`;
  
  let isDomainDeleted = false;
  let isDnsRecordDeleted = true;

  // 1. 从 Vercel 移除域名（证书会自动清理）
  if (process.env.VERCEL_URL) {
    isDomainDeleted = await deleteVercelDomain(domain);
  }

  // 2. 从 Cloudflare 删除 DNS 记录
  if (process.env.CLOUDFLARE_DNS) {
    isDnsRecordDeleted = await deleteDnsRecord(domain);
  }
  
  return isDomainDeleted && isDnsRecordDeleted;
};
```

### 2.5 域名重命名

```typescript
export const renameDomain = async (oldSlug: string | null, newSlug: string) => {
  // 先创建新域名，确保新域名可用
  await createDomain(newSlug);
  
  // 再尝试删除旧域名（失败不阻塞）
  if (oldSlug) {
    try {
      await deleteDomain(oldSlug);
    } catch (_e) {
      log.error(`renameDomain: Failed to delete old domain ${oldSlug}`);
    }
  }
};
```

---

## 3. 反向代理对 OAuth 回调和 Cookie 的影响

### 3.1 Cookie 配置机制

Cal.diy 使用 NextAuth.js 进行认证，Cookie 配置位于 `packages/lib/default-cookies.ts`

#### 3.1.1 默认 Cookie 配置

```typescript
// packages/lib/default-cookies.ts
const NEXTAUTH_COOKIE_DOMAIN = process.env.NEXTAUTH_COOKIE_DOMAIN || "";

export function defaultCookies(useSecureCookies: boolean): CookiesOptions {
  const cookiePrefix = useSecureCookies ? "__Secure-" : "";

  const defaultOptions: CookieOption["options"] = {
    domain: NEXTAUTH_COOKIE_DOMAIN || undefined,
    sameSite: useSecureCookies ? "none" : "lax",
    path: "/",
    secure: useSecureCookies,
  };

  return {
    sessionToken: {
      name: `${cookiePrefix}next-auth.session-token`,
      options: {
        ...defaultOptions,
        httpOnly: true,
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
    // ... 其他 Cookie
  };
}
```

#### 3.1.2 Cookie 配置详解

| Cookie 名称 | HttpOnly | SameSite | Secure | 用途 |
|------------|----------|----------|--------|------|
| `__Secure-next-auth.session-token` | ✅ | `none` (HTTPS) / `lax` (HTTP) | ✅ (HTTPS) | 会话 Token |
| `__Secure-next-auth.callback-url` | ❌ | `none` / `lax` | ✅ | OAuth 回调 URL |
| `__Secure-next-auth.csrf-token` | ✅ | `none` / `lax` | ✅ | CSRF 保护 |
| `__Secure-next-auth.pkce.code_verifier` | ✅ | `none` / `lax` | ✅ | PKCE 验证器 |
| `__Secure-next-auth.state` | ✅ | `none` / `lax` | ✅ | OAuth state |
| `next-auth.nonce` | ✅ | `lax` | ✅ | OpenID Connect nonce |

### 3.2 反向代理场景下的 Cookie 问题

#### 3.2.1 问题场景

当使用反向代理（如 Cloudflare、Nginx、AWS CloudFront）时，可能出现以下问题：

```
┌─────────────┐      HTTPS       ┌─────────────┐      HTTP       ┌─────────────┐
│   Browser   │ ◄──────────────► │  Cloudflare │ ◄─────────────► │   Next.js   │
│             │   (cal.acme.com) │   (Proxy)   │  (localhost)   │   (App)     │
└─────────────┘                  └─────────────┘                 └─────────────┘
     │                                  │                               │
     │ 1. 设置 Cookie: SameSite=None   │                               │
     │    Secure=true                  │                               │
     │                                  │                               │
     │ 2. Cookie domain 问题:          │                               │
     │    - 浏览器认为域名是 cal.acme.com│                               │
     │    - 但 Cookie 可能设置为         │                               │
     │      .cal.com (通配符)           │                               │
     │      或不设置 domain              │                               │
```

#### 3.2.2 常见问题及解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **Cookie 跨域不共享** | 组织子域名不同，Cookie 默认不共享 | 设置 `NEXTAUTH_COOKIE_DOMAIN` 为根域名 |
| **SameSite=None 被阻止** | 没有同时设置 Secure | 确保使用 HTTPS，`useSecureCookies` 为 true |
| **反向代理后 Cookie 丢失** | 代理修改了请求头或协议 | 配置 `NEXTAUTH_URL` 和信任代理头 |
| **CSRF Token 不匹配** | 代理修改了 Origin 或 Referer | 配置正确的 `NEXTAUTH_URL` |

### 3.3 OAuth 回调 URL 验证

#### 3.3.1 重定向回调验证

```typescript
// packages/features/auth/lib/next-auth-options.ts
callbacks: {
  async redirect({ url, baseUrl }) {
    // 1. 允许相对路径回调
    if (url.startsWith("/")) return `${baseUrl}${url}`;
    
    // 2. 仅允许同域名的绝对 URL 回调
    // 安全机制：防止开放重定向漏洞
    else if (new URL(url).hostname === new URL(WEBAPP_URL).hostname) return url;
    
    // 3. 其他情况重定向到首页
    return baseUrl;
  },
}
```

#### 3.3.2 外部 OAuth 集成回调

对于第三方应用集成（Google Calendar、Zoom 等），回调 URL 在 OAuth 客户端配置中预定义：

```typescript
// next.config.ts - 开发环境特殊处理
...(process.env.NODE_ENV === "development" &&
isOrganizationsEnabled &&
process.env.NEXT_PUBLIC_WEBAPP_URL !== "http://localhost:3000"
  ? [
      {
        has: [
          {
            type: "header",
            key: "host",
            value: "localhost:3000",
          },
        ],
        source: "/api/integrations/:args*",
        destination: `${process.env.NEXT_PUBLIC_WEBAPP_URL}/api/integrations/:args*`,
        permanent: false,
      },
    ]
  : []),
```

#### 3.3.3 回调 URL 安全规则

```typescript
// next.config.ts - 防止开放重定向攻击
{
  source: "/api/auth/:path*",
  has: [
    {
      type: "query",
      key: "callbackUrl",
      value: "^(?!https?://).*$", // 匹配非 http/https 开头的 URL
    },
  ],
  destination: "/404",
  permanent: false,
}
```

**规则说明：**
- 如果 `callbackUrl` 不是以 `http://` 或 `https://` 开头，返回 404
- 防止通过 `//evil.com` 或 `javascript:` 等方式进行开放重定向攻击

### 3.4 自定义域名下的 OAuth 配置

#### 3.4.1 多组织域名的挑战

当企业使用自定义域名时，OAuth 回调 URL 需要考虑：

| 场景 | 主域名 | 组织域名 | 回调 URL |
|------|--------|----------|----------|
| SaaS 平台 | `app.cal.com` | `acme.cal.com`, `company.cal.com` | 需要通配符或动态配置 |
| 自托管单组织 | `cal.company.com` | 无 | 固定回调 URL |

#### 3.4.2 推荐配置方案

**方案 A：使用主域名统一处理 OAuth（推荐）**

```
OAuth 流程:
1. 用户在 acme.cal.com 点击 "连接 Google 日历"
2. 重定向到 app.cal.com/api/integrations/googlecalendar/add
3. Google OAuth 回调到 app.cal.com/api/integrations/googlecalendar/callback
4. 凭证保存后，重定向回 acme.cal.com
```

**方案 B：通配符域名（需要 OAuth 提供商支持）**

```
Google Cloud Console 配置:
- 授权重定向 URI: https://*.cal.com/api/integrations/googlecalendar/callback

注意：并非所有 OAuth 提供商都支持通配符域名
```

### 3.5 环境变量配置建议

#### 3.5.1 多租户 SaaS 部署

```bash
# 主应用 URL
NEXT_PUBLIC_WEBAPP_URL=https://app.cal.com
NEXTAUTH_URL=https://app.cal.com/api/auth

# Cookie 配置 - 使用根域名实现跨子域共享
# 这样 app.cal.com, acme.cal.com, company.cal.com 可以共享登录状态
NEXTAUTH_COOKIE_DOMAIN=.cal.com

# 组织功能启用
ORGANIZATIONS_ENABLED=1
```

#### 3.5.2 单组织自托管部署

```bash
# 自定义域名
NEXT_PUBLIC_WEBAPP_URL=https://cal.company.com
NEXTAUTH_URL=https://cal.company.com/api/auth

# 单组织模式
NEXT_PUBLIC_SINGLE_ORG_SLUG=company
ORGANIZATIONS_ENABLED=1

# Cookie 配置 - 可选，通常不需要设置
# NEXTAUTH_COOKIE_DOMAIN=.company.com
```

#### 3.5.3 开发环境配置

```bash
# 本地开发
NEXT_PUBLIC_WEBAPP_URL=http://localhost:3000
NEXTAUTH_URL=http://localhost:3000/api/auth

# 如需测试组织子域名，可修改 hosts 文件:
# 127.0.0.1 app.localhost
# 127.0.0.1 acme.localhost
```

---

## 4. 环境变量配置

### 4.1 必需环境变量

#### 4.1.1 Vercel 集成（SSL 证书管理）

| 变量名 | 描述 | 示例 |
|--------|------|------|
| `PROJECT_ID_VERCEL` | Vercel 项目 ID | `prj_abc123def456` |
| `TEAM_ID_VERCEL` | Vercel 团队 ID（可选） | `team_xyz789` |
| `AUTH_BEARER_TOKEN_VERCEL` | Vercel API Token | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `VERCEL_URL` | Vercel 部署 URL | `cal-com.vercel.app` |

#### 4.1.2 Cloudflare 集成（DNS 管理）

| 变量名 | 描述 | 示例 |
|--------|------|------|
| `CLOUDFLARE_ZONE_ID` | Cloudflare 区域 ID | `a1b2c3d4e5f6g7h8i9j0` |
| `CLOUDFLARE_VERCEL_CNAME` | CNAME 目标值 | `cname.vercel-dns.com` |
| `AUTH_BEARER_TOKEN_CLOUDFLARE` | Cloudflare API Token | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| `CLOUDFLARE_DNS` | 启用 Cloudflare DNS 管理 | `1` 或 `true` |

#### 4.1.3 组织功能配置

| 变量名 | 描述 | 示例 |
|--------|------|------|
| `ORGANIZATIONS_ENABLED` | 启用组织功能 | `1` |
| `NEXT_PUBLIC_SINGLE_ORG_SLUG` | 单组织模式的组织 Slug | `acme`（可选） |
| `NEXT_PUBLIC_WEBAPP_URL` | 主应用 URL | `https://app.cal.com` |
| `NEXTAUTH_URL` | NextAuth 端点 | `https://app.cal.com/api/auth` |
| `NEXTAUTH_COOKIE_DOMAIN` | Cookie 作用域 | `.cal.com`（可选） |

### 4.2 配置示例

#### 4.2.1 完整 SaaS 平台配置

```bash
# =============================================================================
# 核心配置
# =============================================================================
NEXT_PUBLIC_WEBAPP_URL=https://app.cal.com
NEXTAUTH_URL=https://app.cal.com/api/auth
NEXTAUTH_SECRET=your-super-secret-key
CALENDSO_ENCRYPTION_KEY=your-32-char-encryption-key

# =============================================================================
# 组织功能
# =============================================================================
ORGANIZATIONS_ENABLED=1

# =============================================================================
# Cookie 配置 - 允许跨子域共享登录状态
# =============================================================================
NEXTAUTH_COOKIE_DOMAIN=.cal.com

# =============================================================================
# Vercel 集成 - 自动 SSL 证书
# =============================================================================
PROJECT_ID_VERCEL=prj_abc123def456
TEAM_ID_VERCEL=team_xyz789
AUTH_BEARER_TOKEN_VERCEL=vercel-token-here
VERCEL_URL=cal-com.vercel.app

# =============================================================================
# Cloudflare 集成 - DNS 管理
# =============================================================================
CLOUDFLARE_ZONE_ID=your-cloudflare-zone-id
CLOUDFLARE_VERCEL_CNAME=cname.vercel-dns.com
AUTH_BEARER_TOKEN_CLOUDFLARE=cloudflare-token-here
CLOUDFLARE_DNS=1
```

#### 4.2.2 单组织自托管配置

```bash
# =============================================================================
# 核心配置
# =============================================================================
NEXT_PUBLIC_WEBAPP_URL=https://cal.company.com
NEXTAUTH_URL=https://cal.company.com/api/auth
NEXTAUTH_SECRET=your-super-secret-key
CALENDSO_ENCRYPTION_KEY=your-32-char-encryption-key

# =============================================================================
# 单组织模式
# =============================================================================
ORGANIZATIONS_ENABLED=1
NEXT_PUBLIC_SINGLE_ORG_SLUG=company

# =============================================================================
# 可选：Vercel 集成（如果使用 Vercel 部署）
# =============================================================================
# PROJECT_ID_VERCEL=prj_abc123def456
# AUTH_BEARER_TOKEN_VERCEL=vercel-token-here
# VERCEL_URL=cal-company.vercel.app
```

---

## 附录

### A. 相关代码文件位置

| 功能 | 文件路径 |
|------|----------|
| Next.js 路由重写 | `apps/web/next.config.ts` |
| 组织域名路由配置 | `apps/web/getNextjsOrgRewriteConfig.ts` |
| 组织重定向处理 | `apps/web/lib/handleOrgRedirect.ts` |
| 域名管理核心 | `packages/lib/domainManager/organization.ts` |
| Vercel 集成 | `packages/lib/domainManager/deploymentServices/vercel.ts` |
| Cloudflare 集成 | `packages/lib/domainManager/deploymentServices/cloudflare.ts` |
| Cookie 配置 | `packages/lib/default-cookies.ts` |
| NextAuth 配置 | `packages/features/auth/lib/next-auth-options.ts` |

### B. 故障排查

#### B.1 SSL 证书问题

**症状**：访问自定义域名时显示证书错误

**排查步骤：**
1. 检查域名是否已正确添加到 Vercel 项目
2. 验证 DNS CNAME 记录是否指向正确的目标
3. 确认域名所有权验证已完成（Vercel 可能需要 TXT 记录）
4. 等待证书签发（通常需要几分钟到几小时）

#### B.2 Cookie 不共享问题

**症状**：在 `app.cal.com` 登录后，`acme.cal.com` 仍显示未登录

**解决方案：**
1. 确保设置了 `NEXTAUTH_COOKIE_DOMAIN=.cal.com`
2. 确认所有子域名都使用 HTTPS（`SameSite=None` 需要 `Secure`）
3. 检查浏览器 Cookie 设置中是否有第三方 Cookie 阻止

#### B.3 OAuth 回调失败

**症状**：第三方集成（Google Calendar、Zoom 等）连接失败

**排查步骤：**
1. 检查 OAuth 客户端配置中的回调 URL 是否正确
2. 确认 `NEXTAUTH_URL` 和 `NEXT_PUBLIC_WEBAPP_URL` 配置一致
3. 检查是否有反向代理修改了请求协议（HTTP → HTTPS）
4. 验证回调 URL 不在开放重定向黑名单中

### C. 安全最佳实践

1. **始终使用 HTTPS**：生产环境必须启用 HTTPS，确保 Cookie 的 `Secure` 标志生效
2. **限制 Cookie 作用域**：仅在必要时设置 `NEXTAUTH_COOKIE_DOMAIN`，避免过度共享
3. **验证回调 URL**：确保 OAuth 回调 URL 严格验证，防止开放重定向攻击
4. **使用强密钥**：`NEXTAUTH_SECRET` 和 `CALENDSO_ENCRYPTION_KEY` 使用足够随机的长字符串
5. **定期轮换密钥**：定期更换 API Token 和加密密钥

---

**文档版本**: 1.0  
**最后更新**: 2026-05-05  
**基于代码版本**: Cal.diy 主分支
