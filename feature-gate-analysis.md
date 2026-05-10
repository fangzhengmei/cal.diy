# Cal.diy 企业功能 Feature Flag & License Gate 完整链路分析

## 1. 概述

Cal.diy 项目采用了**双层控制机制**来管理企业功能的可见性，但两者的实现阶段不同：

| 机制 | 状态 | 说明 |
|------|------|------|
| **Feature Flag 系统** | ✅ 完全实现 | 细粒度功能开关，支持全局/团队/用户三层控制 |
| **License Gate 系统** | ⚠️ 骨架阶段 | 大部分为占位实现，实际验证逻辑尚未完成 |

---

## 2. 完整请求链路（从入口到最终可见性）

```
用户请求进入
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 1: 服务器端初始校验 (Server-side Initial Check)              │
│                                                                  │
│  2.1 Session 初始化                                              │
│     ├─ getServerSession() 被调用                                 │
│     ├─ 从 JWT token 提取用户信息                                 │
│     ├─ 调用 LicenseKeySingleton.checkLicense()                  │
│     │   └─ ⚠️ 占位实现: 直接返回 true                            │
│     └─ 将 hasValidLicense 注入 session                           │
│                                                                  │
│  2.2 布局/路由组选择                                             │
│     ├─ (use-page-wrapper) 组 → PageWrapperLayout                │
│     └─ (booking-page-wrapper) 组 → BookingPageLayout            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 2: 页面包装层 (Page Wrapper Layer)                           │
│                                                                  │
│  PageWrapper requiresLicense 属性                               │
│  ├─ 定义存在，但两个分支渲染相同内容                             │
│  ├─ ⚠️ 占位实现: 没有实际的许可证校验逻辑                        │
│  └─ 所有页面当前 requiresLicense=false                          │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 3: 客户端 Provider 初始化 (Client-side Provider Setup)      │
│                                                                  │
│  3.1 Providers 组件 (Root)                                       │
│     ├─ GeoProvider                                              │
│     ├─ SessionProvider                                          │
│     ├─ TrpcProvider                                             │
│     └─ ToastProvider                                            │
│                                                                  │
│  3.2 AppProviders 组件                                          │
│     ├─ ThemeProvider                                            │
│     ├─ NuqsAdapter                                              │
│     └─ FeatureFlagsProvider                                     │
│         ├─ 调用 useFlags() Hook                                 │
│         │   └─ trpc.viewer.features.map.useQuery()              │
│         │       └─ 获取全局 Feature Flag 状态                    │
│         └─ 通过 React Context 注入到子组件                       │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 4: 组件级 Feature Flag 判定 (Component-level Checks)        │
│                                                                  │
│  方式 A: useFlagMap() - 从 Context 读取                         │
│  方式 B: 直接调用 tRPC 查询特定团队/用户 Feature                 │
│  方式 C: 后端 Service 直接调用 Repository                       │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ 阶段 5: 最终可见性决策 (Final Visibility)                         │
│                                                                  │
│  根据 Feature Flag 状态决定:                                    │
│  ├─ 渲染组件 vs return null                                     │
│  ├─ 显示按钮 vs 隐藏按钮                                        │
│  ├─ 启用功能 vs 降级功能                                        │
│  └─ 重定向到其他页面                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. License Gate 详细分析

### 3.1 当前实现状态总览

| 文件/组件 | 位置 | 实际行为 | 状态 |
|-----------|------|---------|------|
| `LicenseKeySingleton` | `packages/features/auth/lib/getServerSession.ts:11-15` | 所有方法直接返回 `true` | ⚠️ 占位 |
| `LicenseKeyService` | `packages/trpc/server/routers/viewer/deploymentSetup/validateLicense.handler.ts:5-8` | `validateLicenseKey()` 直接返回 `true` | ⚠️ 占位 |
| `DeploymentsService.checkLicense()` | `apps/api/v2/src/modules/deployments/deployments.service.ts:15-17` | 注释说明 "完全开源，无需许可证" | ⚠️ 占位 |
| `PageWrapper.requiresLicense` | `apps/web/components/PageWrapperAppDir.tsx:29-33` | 两个分支渲染相同内容 | ⚠️ 占位 |
| `getServerSideProps.hasValidLicense` | `apps/web/server/lib/setup/getServerSideProps.tsx:39` | 硬编码为 `false` | ⚠️ 占位 |
| `getServerSideProps.isFreeLicense` | `apps/web/server/lib/setup/getServerSideProps.tsx:41` | 硬编码为 `true` | ⚠️ 占位 |

### 3.2 Session 中的 License 注入

**文件**: `packages/features/auth/lib/getServerSession.ts`

```typescript
// ⚠️ 这是占位实现 - 所有方法直接返回 true
class LicenseKeySingleton {
  static async getInstance(..._args: unknown[]) { return new LicenseKeySingleton(); }
  async checkLicense() { return true; }           // ⚠️ 占位
  async validateLicenseKey() { return true; }     // ⚠️ 占位
}

class DeploymentRepository {
  constructor(_prisma?: unknown) {}
  async findFirst(..._args: unknown[]) { return null; }  // ⚠️ 占位
}

// 实际调用链路
export async function getServerSession(options: { ... }) {
  // ... 验证 token ...
  
  // ⚠️ 这里调用的是占位实现
  const deploymentRepo = new DeploymentRepository(prisma);
  const licenseKeyService = await LicenseKeySingleton.getInstance(deploymentRepo);
  const hasValidLicense = await licenseKeyService.checkLicense();  // 永远返回 true
  
  // hasValidLicense 被注入到 session 中
  const session: Session = {
    hasValidLicense,  // ⚠️ 永远为 true
    // ... 其他字段
  };
  
  return session;
}
```

**重要发现**：
- `hasValidLicense` 被注入到 Session 中
- 但由于 `LicenseKeySingleton.checkLicense()` 是占位实现，**所有用户的 session.hasValidLicense 永远为 true**
- 这意味着当前任何基于 `hasValidLicense` 的判断都不会真正限制功能

### 3.3 tRPC License 验证端点

**文件**: `packages/trpc/server/routers/viewer/deploymentSetup/validateLicense.handler.ts`

```typescript
// ⚠️ 这是占位实现
class LicenseKeyService {
  async checkLicense() { return true; }
  static async validateLicenseKey(_key?: string) { return true; }  // ⚠️ 忽略输入，直接返回 true
}

export const validateLicenseHandler = async ({ input }: ValidateLicenseOptions) => {
  const { licenseKey } = input;

  // E2E 测试模式
  if (process.env.NEXT_PUBLIC_IS_E2E === "1") {
    return { valid: true, message: "License key is valid (E2E mode)" };
  }

  try {
    // ⚠️ 调用占位实现，永远返回 true
    const isValid = await LicenseKeyService.validateLicenseKey(licenseKey);

    return {
      valid: isValid,
      message: isValid ? "License key is valid" : "License key is invalid",
    };
  } catch (error) {
    console.error("License validation failed:", error);
    return { valid: false, message: "License key validation failed" };
  }
};
```

**实际行为**：
- 无论传入什么 `licenseKey`，`validateLicenseKey()` 都返回 `true`
- 只有 E2E 测试模式有特殊处理

### 3.4 页面级 License 控制

**文件**: `apps/web/components/PageWrapperAppDir.tsx`

```typescript
export type PageWrapperProps = Readonly<{
  children: React.ReactNode;
  requiresLicense: boolean;  // 属性定义存在
  nonce: string | undefined;
  isBookingPage?: boolean;
}>;

function PageWrapper(props: PageWrapperProps) {
  return (
    <AppProviders {...providerProps}>
      {/* ⚠️ 占位实现 - 两个分支渲染完全相同的内容 */}
      {props.requiresLicense ? (
        <>{props.children}</>
      ) : (
        <>{props.children}</>
      )}
    </AppProviders>
  );
}
```

**文件**: `apps/web/app/(use-page-wrapper)/layout.tsx`

```typescript
export default async function PageWrapperLayout({ children }: { children: React.ReactNode }) {
  // ...
  return (
    <>
      {/* ⚠️ 所有页面当前都设置为 requiresLicense=false */}
      <PageWrapper requiresLicense={false} nonce={nonce}>
        {children}
      </PageWrapper>
    </>
  );
}
```

**分析**：
- `requiresLicense` 属性定义存在，但**没有实际的条件渲染逻辑**
- 两个分支都渲染相同的 `{props.children}`
- 所有路由组当前都设置为 `requiresLicense=false`
- 即使设置为 `true`，也不会产生任何行为差异

### 3.5 API v2 中的 License 处理

**文件**: `apps/api/v2/src/modules/deployments/deployments.service.ts`

```typescript
@Injectable()
export class DeploymentsService {
  // Cal.diy is fully open source — no license key is required.
  async checkLicense() {
    return true;  // ⚠️ 直接返回 true
  }
}
```

**注释明确说明**：Cal.diy 是完全开源的，不需要许可证密钥。这解释了为什么所有许可证检查都是占位实现。

---

## 4. Feature Flag 详细分析（完全实现）

### 4.1 数据模型

**文件**: `packages/prisma/schema.prisma`

```prisma
model Feature {
  slug        String         @id @unique   // 功能标识，如 'sink-shortener'
  enabled     Boolean        @default(false)  // 全局开关状态
  description String?
  type        FeatureType?   @default(RELEASE)
  stale       Boolean?       @default(false)
  users       UserFeatures[]
  teams       TeamFeatures[]
}

model UserFeatures {
  user       User     @relation(...)
  userId     Int
  feature    Feature  @relation(...)
  featureId  String
  enabled    Boolean         // 用户级覆盖
  assignedAt DateTime @default(now())
  assignedBy String
  
  @@id([userId, featureId])
}

model TeamFeatures {
  team       Team     @relation(...)
  teamId     Int
  feature    Feature  @relation(...)
  featureId  String
  enabled    Boolean         // 团队级覆盖
  assignedAt DateTime @default(now())
  assignedBy String
  
  @@id([teamId, featureId])
}
```

### 4.2 三态语义与优先级

**优先级规则**（从高到低）：

```
用户级显式设置 (UserFeatures)
    │
    ├─ enabled=true  → 返回 true（最高优先级）
    ├─ enabled=false → 返回 false（阻止继承）
    └─ 无记录 → 继续向下检查
              ↓
团队级显式设置 (TeamFeatures)
    │
    ├─ enabled=true  → 返回 true
    ├─ enabled=false → 返回 false（阻止继承）
    └─ 无记录 → 递归检查父团队
              ↓
父级团队继承（通过 CTE 递归）
    │
    └─ 找到任一父团队 enabled=true → 返回 true
              ↓
全局设置 (Feature.enabled)
    │
    ├─ true  → 返回 true
    └─ false → 返回 false（默认）
```

**三态语义解释**：

| 状态 | 数据库表示 | 行为 |
|------|-----------|------|
| `enabled` | 存在记录 + `enabled=true` | 显式启用 |
| `disabled` | 存在记录 + `enabled=false` | 显式禁用，**阻止继承** |
| `inherit` | 不存在记录 | 继续向上级查询 |

### 4.3 用户功能检查完整链路

**文件**: `packages/features/flags/features.repository.ts:126-154`

```typescript
async checkIfUserHasFeature(userId: number, slug: string) {
  // 步骤 1: 检查用户级显式设置
  const userFeature = await this.prismaClient.userFeatures.findFirst({
    where: { userId, featureId: slug },
    select: { enabled: true },
  });

  if (userFeature) {
    // 有显式设置，直接返回（无论 true 或 false）
    // 如果是 false，会阻止继承
    return userFeature.enabled;
  }

  // 步骤 2: 检查用户所属的团队（含递归继承）
  const userBelongsToTeamWithFeature = await this.checkIfUserBelongsToTeamWithFeature(userId, slug);
  return userBelongsToTeamWithFeature;
}
```

**团队递归检查（PostgreSQL CTE）**：

```sql
WITH RECURSIVE TeamHierarchy AS (
  -- 起点: 用户直接所属的团队
  SELECT DISTINCT t.id, t."parentId",
    CASE WHEN EXISTS (
      SELECT 1 FROM "TeamFeatures" tf
      WHERE tf."teamId" = t.id 
        AND tf."featureId" = ${slug} 
        AND tf."enabled" = true
    ) THEN true ELSE false END as has_feature
  FROM "Team" t
  INNER JOIN "Membership" m ON m."teamId" = t.id
  WHERE m."userId" = ${userId} AND m.accepted = true

  UNION ALL

  -- 递归: 向上查找父团队
  SELECT DISTINCT p.id, p."parentId",
    CASE WHEN EXISTS (
      SELECT 1 FROM "TeamFeatures" tf
      WHERE tf."teamId" = p.id 
        AND tf."featureId" = ${slug} 
        AND tf."enabled" = true
    ) THEN true ELSE false END as has_feature
  FROM "Team" p
  INNER JOIN TeamHierarchy c ON p.id = c."parentId"
  WHERE NOT c.has_feature  -- 优化: 找到启用的团队后停止递归
)
SELECT 1 FROM TeamHierarchy WHERE has_feature = true LIMIT 1;
```

### 4.4 团队功能检查

**文件**: `packages/features/flags/features.repository.ts:398-451`

```typescript
async checkIfTeamHasFeature(teamId: number, featureId: FeatureId): Promise<boolean> {
  // 步骤 1: 检查当前团队的显式设置
  const teamFeature = await this.prismaClient.teamFeatures.findUnique({
    where: { teamId_featureId: { teamId, featureId } },
    select: { enabled: true },
  });
  if (teamFeature) return teamFeature.enabled;  // 显式设置直接返回

  // 步骤 2: 递归检查父团队
  // 使用与上面类似的 CTE 查询
  // 如果任一父团队 enabled=true → 返回 true
}
```

### 4.5 全局功能检查

**文件**: `packages/features/flags/features.repository.ts:100-112`

```typescript
async checkIfFeatureIsEnabledGlobally(slug: FeatureId): Promise<boolean> {
  try {
    const features = await this.getAllFeatures();  // 有 5 分钟缓存
    const flag = features.find((f) => f.slug === slug);
    return Boolean(flag && flag.enabled);
  } catch (err) {
    captureException(err);
    throw err;
  }
}
```

**缓存策略**：
- `getAllFeatures()` 使用静态缓存，TTL 为 5 分钟
- 缓存键: `FeaturesRepository.featuresCache`

---

## 5. Feature Flag 前端数据流

### 5.1 完整数据流图

```
┌────────────────────────────────────────────────────────────────────┐
│  数据库层                                                          │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────┐                │
│  │ Feature  │  │ UserFeatures │  │ TeamFeatures │                │
│  │ (全局)   │  │ (用户覆盖)   │  │ (团队覆盖)   │                │
│  └────┬─────┘  └──────┬───────┘  └──────┬───────┘                │
└───────┼───────────────┼─────────────────┼─────────────────────────┘
        │               │                 │
        ▼               ▼                 ▼
┌────────────────────────────────────────────────────────────────────┐
│  仓库层 (Repository Layer)                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  FeaturesRepository                                          │  │
│  │  ├─ checkIfFeatureIsEnabledGlobally()                        │  │
│  │  ├─ checkIfUserHasFeature()     → 递归继承逻辑               │  │
│  │  ├─ checkIfTeamHasFeature()     → 递归继承逻辑               │  │
│  │  └─ 缓存: @Memoize / @Unmemoize 装饰器                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│  tRPC API 层                                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  featureFlagRouter                                           │  │
│  │  ├─ list:       GET /api/trpc/features.list                 │  │
│  │  │              返回所有 Feature 定义                         │  │
│  │  │                                                            │  │
│  │  ├─ map:        GET /api/trpc/features.map                  │  │
│  │  │              返回 { slug: boolean } 映射                  │  │
│  │  │              ← 前端 useFlags() Hook 调用此端点            │  │
│  │  │                                                            │  │
│  │  └─ checkTeamFeature: GET /api/trpc/features.checkTeamFeature│  │
│  │                 检查特定团队是否有特定功能                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
                    ▼                     ▼
┌──────────────────────────┐    ┌───────────────────────────────────┐
│  全局 Feature Flags       │    │  特定团队/用户 Feature           │
│  ┌──────────────────────┐ │    │  ┌─────────────────────────────┐ │
│  │ useFlags() Hook      │ │    │  │ trpc.viewer.features.       │ │
│  │                      │ │    │  │   checkTeamFeature.useQuery │ │
│  │ 调用 trpc.viewer.    │ │    │  └─────────────────────────────┘ │
│  │   features.map       │ │    └───────────────────────────────────┘
│  └──────────┬───────────┘ │
└─────────────┼─────────────┘
              │
              ▼
┌────────────────────────────────────────────────────────────────────┐
│  React Context 层                                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  FeatureFlagsProvider                                         │  │
│  │  └─ <FeatureProvider value={flags}>                           │  │
│  │     将 useFlags() 的结果注入 Context                          │  │
│  └──────────────────────────┬───────────────────────────────────┘  │
└─────────────────────────────┼──────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────────┐
│  组件层 (实际使用)                                                   │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  方式 1: useFlagMap() - 从 Context 读取全局 flag              │  │
│  │     const flags = useFlagMap();                              │  │
│  │     if (!flags["email-verification"]) return null;           │  │
│  │                                                              │  │
│  │  方式 2: 直接调用 tRPC 查询特定团队功能                       │  │
│  │     const { data } = trpc.viewer.features.checkTeamFeature   │  │
│  │       .useQuery({ teamId, feature });                        │  │
│  │                                                              │  │
│  │  方式 3: 后端 Service 直接调用 Repository                    │  │
│  │     const repo = new FeaturesRepository(prisma);             │  │
│  │     const enabled = await repo.checkIfUserHasFeature(        │  │
│  │       userId, "sink-shortener"                               │  │
│  │     );                                                       │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键代码解析

**1. useFlags() Hook - 获取全局 Feature Flags**

**文件**: `apps/web/modules/feature-flags/hooks/useFlags.ts`

```typescript
const initialData: AppFlags = {
  "calendar-cache": false,
  "webhooks": false,
  "sink-shortener": false,
  // ... 所有功能默认关闭
};

export function useFlags(): Partial<AppFlags> {
  // 调用 tRPC 端点获取全局 Feature Flag 映射
  const query = trpc.viewer.features.map.useQuery();
  // 如果请求失败或未完成，返回默认值（全部关闭）
  return query.data ?? initialData;
}
```

**2. FeatureProvider - 注入 Context**

**文件**: `apps/web/lib/app-providers-app-dir.tsx:87-90`

```typescript
function FeatureFlagsProvider({ children }: { children: React.ReactNode }) {
  // 1. 先获取全局 flags
  const flags = useFlags();
  // 2. 通过 Context 提供给子组件
  return <FeatureProvider value={flags}>{children}</FeatureProvider>;
}
```

**3. useFlagMap() - 从 Context 读取**

**文件**: `packages/features/flags/context/provider.ts`

```typescript
export function useFlagMap() {
  const flagMapContext = useContext(FeatureContext);
  if (flagMapContext === null) {
    throw new Error("Error: useFlagMap was used outside of FeatureProvider.");
  }
  return flagMapContext as Flags;
}
```

### 5.3 实际使用场景示例

#### 场景 1: 组件级条件渲染

**文件**: `apps/web/modules/users/components/VerifyEmailBanner.tsx`

```typescript
function VerifyEmailBanner({ data }: VerifyEmailBannerProps) {
  // 从 Context 读取全局 Feature Flags
  const flags = useFlagMap();
  
  // ⚠️ 关键决策点: 根据 flag 决定是否渲染
  if (!data || !flags["email-verification"]) return null;

  return (
    <TopBanner
      icon="mail"
      text={t("verify_email_banner_body", { appName: APP_NAME })}
      variant="warning"
      // ...
    />
  );
}
```

**决策逻辑**：
```
flags["email-verification"] = ?
    │
    ├─ true  → 渲染 VerifyEmailBanner 组件
    └─ false → return null（组件不显示）
```

#### 场景 2: 路由重定向

**文件**: `apps/web/modules/auth/hooks/useRedirectToOnboardingIfNeeded.tsx`

```typescript
export function useRedirectToOnboardingIfNeeded() {
  const flags = useFlagMap();
  
  // 根据 Feature Flag 决定是否需要邮箱验证
  const needsEmailVerification =
    !user?.emailVerified && 
    user?.identityProvider === "CAL" && 
    flags["email-verification"];  // ⚠️ 依赖 Feature Flag
  
  // 根据 Feature Flag 选择不同的 onboarding 路径
  const gettingStartedPath = flags["onboarding-v3"] 
    ? "/onboarding/getting-started" 
    : "/getting-started";
  
  // ... 后续重定向逻辑
}
```

**决策逻辑**：
```
flags["onboarding-v3"] = ?
    │
    ├─ true  → 重定向到 /onboarding/getting-started (新版)
    └─ false → 重定向到 /getting-started (旧版)
```

#### 场景 3: 后端 Service 中的功能开关

**文件**: `packages/features/url-shortener/UrlShortenerFactory.ts`

```typescript
export class UrlShortenerFactory {
  static async create({ userId, teamId }: { userId?: number | null; teamId?: number | null } = {}) {
    if (SinkShortener.isConfigured()) {
      const featuresRepository = new FeaturesRepository(prisma);

      // 优先级 1: 全局开关
      const globallyEnabled = await featuresRepository.checkIfFeatureIsEnabledGlobally("sink-shortener");
      if (globallyEnabled) {
        return new SinkShortener(new SinkClient());
      }

      // 优先级 2: 用户级设置
      if (userId) {
        const useSink = await featuresRepository.checkIfUserHasFeature(userId, "sink-shortener");
        if (useSink) {
          return new SinkShortener(new SinkClient());
        }
      }

      // 优先级 3: 团队级设置（含递归继承）
      if (teamId) {
        const useSink = await featuresRepository.checkIfTeamHasFeature(teamId, "sink-shortener");
        if (useSink) {
          return new SinkShortener(new SinkClient());
        }
      }
    }

    // 降级方案
    if (DubShortener.isConfigured()) {
      return new DubShortener();
    }
    return new NoopShortener();
  }
}
```

**完整决策树**：
```
SinkShortener 已配置?
    │
    ├─ 否 → 跳过，检查 DubShortener
    │
    └─ 是
         │
         ▼
    全局 "sink-shortener" 启用?
         │
         ├─ 是 → 返回 SinkShortener
         │
         └─ 否
              │
              ├─ 有 userId? ──→ 用户级启用? ──→ 是 → 返回 SinkShortener
              │
              ├─ 有 teamId? ──→ 团队级启用? ──→ 是 → 返回 SinkShortener
              │                          (递归检查父团队)
              │
              └─ 都不满足 → 检查 DubShortener
                            ├─ 已配置 → 返回 DubShortener
                            └─ 未配置 → 返回 NoopShortener（空实现）
```

### 5.4 缓存机制

**文件**: `packages/features/flags/repositories/CachedTeamFeatureRepository.ts`

```typescript
const CACHE_PREFIX = "features:team";

export class CachedTeamFeatureRepository implements ITeamFeatureRepository {
  constructor(private prismaTeamFeatureRepository: ITeamFeatureRepository) {}

  // 读取操作: 缓存结果
  @Memoize({
    key: (teamId, featureId) => `${CACHE_PREFIX}:${teamId}:${featureId}`,
    schema: TeamFeaturesDtoSchema,
  })
  async findByTeamIdAndFeatureId(teamId: number, featureId: FeatureId) {
    return this.prismaTeamFeatureRepository.findByTeamIdAndFeatureId(teamId, featureId);
  }

  // 写入操作: 清除相关缓存
  @Unmemoize({
    keys: (teamId, featureId) => [
      `${CACHE_PREFIX}:${teamId}:${featureId}`,
      `${CACHE_PREFIX}:enabledFeatures:${teamId}`,
    ],
  })
  async upsert(teamId: number, featureId: FeatureId, enabled: boolean, assignedBy: string) {
    return this.prismaTeamFeatureRepository.upsert(teamId, featureId, enabled, assignedBy);
  }
}
```

**缓存键格式**：
| 数据类型 | 缓存键格式 | 示例 |
|---------|-----------|------|
| 团队单个 Feature | `features:team:{teamId}:{featureId}` | `features:team:42:sink-shortener` |
| 团队所有启用 Feature | `features:team:enabledFeatures:{teamId}` | `features:team:enabledFeatures:42` |
| 用户单个 Feature | `features:user:{userId}:{featureId}` | `features:user:123:emails` |
| 用户自动加入 | `features:user:autoOptIn:{userId}` | `features:user:autoOptIn:123` |

---

## 6. 当前实现的边界情况

### 6.1 License Gate 的边界

| 场景 | 当前行为 | 预期行为（如果实现） |
|------|---------|---------------------|
| 无许可证密钥 | 功能正常（因为 checkLicense 返回 true） | 应该限制企业功能 |
| 无效许可证 | 功能正常（validateLicenseKey 返回 true） | 应该限制企业功能 |
| 设置 requiresLicense=true | 页面正常渲染（两个分支相同） | 应该检查许可证后决定是否渲染 |
| session.hasValidLicense | 永远为 true | 应该根据实际许可证状态 |

### 6.2 Feature Flag 的边界

| 场景 | 当前行为 | 说明 |
|------|---------|------|
| 新功能默认状态 | 关闭（AppFlags 初始值全部 false） | 渐进式发布策略 |
| 用户显式 disabled | 阻止继承，返回 false | 即使用户所在团队启用了该功能 |
| 团队显式 disabled | 阻止继承，返回 false | 即使父团队启用了该功能 |
| 无任何记录 | 继承逻辑，向上查找 | 最终回退到全局状态 |
| 团队层级很深 | 通过 CTE 递归，找到即停 | 性能优化：找到启用的就停止 |

### 6.3 数据一致性边界

```
问题场景:
  1. 组织(teamId=1) 启用了 "sink-shortener"
  2. 子团队(teamId=2) 没有任何设置 → 继承，功能可用
  3. 管理员禁用子团队的该功能 → TeamFeatures { teamId=2, enabled=false }
  4. 子团队用户现在无法使用该功能（显式 disabled 阻止继承）

问题场景 2:
  1. 全局禁用 "sink-shortener" (Feature.enabled=false)
  2. 某个团队显式启用 → TeamFeatures { teamId=5, enabled=true }
  3. 该团队用户能否使用? 
     → 能（团队级设置优先级高于全局）
```

---

## 7. 关键文件索引

### License Gate（含占位标记）

| 路径 | 功能 | 状态 |
|------|------|------|
| `packages/features/auth/lib/getServerSession.ts:11-15` | `LicenseKeySingleton` - Session 中注入 hasValidLicense | ⚠️ 占位 |
| `packages/trpc/server/routers/viewer/deploymentSetup/validateLicense.handler.ts:5-8` | `LicenseKeyService` - tRPC 许可证验证 | ⚠️ 占位 |
| `apps/api/v2/src/modules/deployments/deployments.service.ts:14-17` | `DeploymentsService.checkLicense()` - API v2 许可证检查 | ⚠️ 占位 |
| `apps/web/components/PageWrapperAppDir.tsx:29-33` | `requiresLicense` 属性渲染逻辑 | ⚠️ 占位 |
| `apps/web/server/lib/setup/getServerSideProps.tsx:39,41` | `hasValidLicense` 和 `isFreeLicense` 硬编码 | ⚠️ 占位 |
| `packages/trpc/server/routers/viewer/deploymentSetup/update.handler.ts` | 保存许可证密钥到数据库 | ✅ 实现 |
| `packages/features/deployment/repositories/DeploymentRepository.ts` | Deployment 数据访问 | ✅ 实现 |
| `packages/prisma/schema.prisma:1295-1305` | Deployment 模型（含 licenseKey 字段） | ✅ 实现 |

### Feature Flag（全部实现）

| 路径 | 功能 |
|------|------|
| `packages/features/flags/config.ts` | Feature Flag 类型定义（AppFlags） |
| `packages/features/flags/features.repository.interface.ts` | Repository 接口定义 |
| `packages/features/flags/features.repository.ts` | 核心 Repository 实现（含递归继承逻辑） |
| `packages/features/flags/repositories/PrismaFeatureRepository.ts` | 全局 Feature 数据访问 |
| `packages/features/flags/repositories/PrismaUserFeatureRepository.ts` | 用户级 Feature 数据访问 |
| `packages/features/flags/repositories/PrismaTeamFeatureRepository.ts` | 团队级 Feature 数据访问 |
| `packages/features/flags/repositories/Cached*.ts` | 缓存装饰器实现 |
| `packages/trpc/server/routers/features/_router.ts` | tRPC Feature Flag 路由 |
| `apps/web/modules/feature-flags/hooks/useFlags.ts` | 前端 Hook：获取全局 Feature Flags |
| `packages/features/flags/context/provider.ts` | React Context：Provider 和 useFlagMap() |
| `packages/features/flags/hooks/useIsFeatureEnabledForTeam.ts` | 前端 Hook：检查团队功能 |
| `packages/prisma/schema.prisma:1369-1403` | Feature/UserFeatures/TeamFeatures 数据模型 |

---

## 8. 总结

### 8.1 当前架构状态

```
┌─────────────────────────────────────────────────────────────────┐
│  License Gate 系统                                               │
│                                                                  │
│  [数据库] Deployment.licenseKey  ← 可以存储                      │
│       │                                                          │
│       ▼                                                          │
│  [tRPC] validateLicense.handler  ← ⚠️ 永远返回 true             │
│       │                                                          │
│       ▼                                                          │
│  [Session] hasValidLicense  ← ⚠️ 永远为 true                    │
│       │                                                          │
│       ▼                                                          │
│  [PageWrapper] requiresLicense  ← ⚠️ 两个分支相同               │
│                                                                  │
│  结论: License Gate 处于骨架阶段，未实际启用                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Feature Flag 系统                                               │
│                                                                  │
│  [数据库] Feature / UserFeatures / TeamFeatures  ← ✅ 完整       │
│       │                                                          │
│       ▼                                                          │
│  [Repository] 三态语义 + 递归继承 + 缓存  ← ✅ 完整              │
│       │                                                          │
│       ▼                                                          │
│  [tRPC] list / map / checkTeamFeature  ← ✅ 完整                │
│       │                                                          │
│       ▼                                                          │
│  [前端] useFlags() → Context → useFlagMap()  ← ✅ 完整          │
│       │                                                          │
│       ▼                                                          │
│  [组件] 条件渲染 / 路由重定向 / 功能降级  ← ✅ 实际使用          │
│                                                                  │
│  结论: Feature Flag 完全实现并已在生产使用                        │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 关键发现

1. **License Gate 系统大部分是占位实现**
   - 所有检查方法直接返回 `true`
   - `requiresLicense` 属性存在但无实际逻辑
   - 原因：代码注释说明 "Cal.diy is fully open source"

2. **Feature Flag 系统完全实现**
   - 支持全局/团队/用户三层控制
   - 三态语义：enabled / disabled / inherit
   - 团队层级递归继承（PostgreSQL CTE）
   - 缓存机制减少数据库查询
   - 已在多个组件中实际使用

3. **两者的集成点**
   - 理论上：License Gate 应该是 Feature Flag 的前置条件
   - 实际上：由于 License Gate 是占位，Feature Flag 是唯一的控制机制
   - Session 中的 `hasValidLicense` 永远为 `true`，不影响任何判断

---

*本报告基于 2026-05-10 的代码库状态分析生成。*
