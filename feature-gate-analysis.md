# Cal.diy 企业功能 Feature Flag & License Gate 实现机制分析

## 目录

- [1. 概述](#1-概述)
- [2. Feature Flag 系统架构](#2-feature-flag-系统架构)
  - [2.1 数据库模型](#21-数据库模型)
  - [2.2 核心类型定义](#22-核心类型定义)
  - [2.3 三态语义 (Tri-state Semantics)](#23-三态语义-tri-state-semantics)
- [3. Feature Flag 实现细节](#3-feature-flag-实现细节)
  - [3.1 仓库层 (Repository Layer)](#31-仓库层-repository-layer)
  - [3.2 缓存机制](#32-缓存机制)
  - [3.3 用例层 (Use Case Layer)](#33-用例层-use-case-layer)
  - [3.4 tRPC API 层](#34-trpc-api-层)
  - [3.5 前端集成](#35-前端集成)
- [4. License Gate 实现](#4-license-gate-实现)
  - [4.1 许可证验证流程](#41-许可证验证流程)
  - [4.2 自托管许可证创建](#42-自托管许可证创建)
  - [4.3 页面级许可证控制](#43-页面级许可证控制)
- [5. 实际使用场景](#5-实际使用场景)
  - [5.1 URL 缩短器功能控制](#51-url-缩短器功能控制)
  - [5.2 团队层次结构继承](#52-团队层次结构继承)
- [6. 架构总结](#6-架构总结)
  - [6.1 数据流图](#61-数据流图)
  - [6.2 设计特点](#62-设计特点)
- [7. 关键文件索引](#7-关键文件索引)

---

## 1. 概述

Cal.diy 项目采用了双层控制机制来管理企业功能的可见性：

1. **Feature Flag 系统**：细粒度的功能开关，支持全局、团队、用户三个层级的控制
2. **License Gate 系统**：部署级别的许可证验证，用于控制整个部署的企业功能访问权限

这两个系统协同工作，形成了从"部署级别"到"功能级别"再到"用户级别"的完整访问控制链条。

---

## 2. Feature Flag 系统架构

### 2.1 数据库模型

项目在 `packages/prisma/schema.prisma` 中定义了三个核心模型：

#### Feature 模型
```prisma
model Feature {
  slug        String         @id @unique
  enabled     Boolean        @default(false)
  description String?
  type        FeatureType?   @default(RELEASE)
  stale       Boolean?       @default(false)
  lastUsedAt  DateTime?
  createdAt   DateTime?      @default(now())
  updatedAt   DateTime?      @default(now()) @updatedAt
  updatedBy   Int?
  users       UserFeatures[]
  teams       TeamFeatures[]
}
```

#### UserFeatures 模型
```prisma
model UserFeatures {
  user       User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  userId     Int
  feature    Feature  @relation(fields: [featureId], references: [slug], onDelete: Cascade)
  featureId  String
  enabled    Boolean
  assignedAt DateTime @default(now())
  assignedBy String
  updatedAt  DateTime @updatedAt

  @@id([userId, featureId])
  @@index([userId, featureId])
}
```

#### TeamFeatures 模型
```prisma
model TeamFeatures {
  team       Team     @relation(fields: [teamId], references: [id], onDelete: Cascade)
  teamId     Int
  feature    Feature  @relation(fields: [featureId], references: [slug], onDelete: Cascade)
  featureId  String
  enabled    Boolean
  assignedAt DateTime @default(now())
  assignedBy String
  updatedAt  DateTime @updatedAt

  @@id([teamId, featureId])
  @@index([teamId, featureId])
}
```

#### Deployment 模型
```prisma
model Deployment {
  id                      Int       @id @default(1)
  logo                    String?
  theme                   Json?
  licenseKey              String?
  signatureTokenEncrypted String?
  agreedLicenseAt         DateTime?
}
```

### 2.2 核心类型定义

在 `packages/features/flags/config.ts` 中定义了功能标志的类型系统：

```typescript
export type AppFlags = {
  "calendar-cache": boolean;
  "calendar-cache-serve": boolean;
  emails: boolean;
  webhooks: boolean;
  "email-verification": boolean;
  "disable-signup": boolean;
  "organizer-request-email-v2": boolean;
  "delegation-credential": boolean;
  "salesforce-crm-tasker": boolean;
  "cal-video-log-in-overlay": boolean;
  "restriction-schedule": boolean;
  "calendar-subscription-cache": boolean;
  "calendar-subscription-sync": boolean;
  "onboarding-v3": boolean;
  "booker-botid": boolean;
  "booking-calendar-view": boolean;
  "booking-email-sms-tasker": boolean;
  "bookings-v3": boolean;
  "booking-audit": boolean;
  "hwm-seating": boolean;
  "signup-watchlist-review": boolean;
  "sink-shortener": boolean;
};

export type FeatureState = "enabled" | "disabled" | "inherit";
export type FeatureId = keyof AppFlags;
export type TeamFeatures = Record<keyof AppFlags, boolean>;
```

### 2.3 三态语义 (Tri-state Semantics)

Feature Flag 系统采用了三态语义来实现灵活的继承机制：

| 状态 | 数据库表示 | 含义 |
|------|-----------|------|
| `enabled` | 存在记录 + `enabled=true` | 显式启用该功能 |
| `disabled` | 存在记录 + `enabled=false` | 显式禁用该功能（阻止继承） |
| `inherit` | 不存在记录 | 从上级（团队/组织）继承 |

**优先级规则**：
1. 用户级设置 > 团队级设置 > 全局设置
2. 显式的 `disabled` 会阻止继承，返回 `false`
3. 没有记录时，继续向上级查询

---

## 3. Feature Flag 实现细节

### 3.1 仓库层 (Repository Layer)

#### 接口定义 (`IFeaturesRepository`)

在 `packages/features/flags/features.repository.interface.ts` 中定义了核心接口：

```typescript
export interface IFeaturesRepository {
  checkIfFeatureIsEnabledGlobally(slug: FeatureId): Promise<boolean>;
  checkIfUserHasFeature(userId: number, slug: string): Promise<boolean>;
  checkIfUserHasFeatureNonHierarchical(userId: number, slug: string): Promise<boolean>;
  checkIfTeamHasFeature(teamId: number, slug: FeatureId): Promise<boolean>;
  getTeamsWithFeatureEnabled(slug: FeatureId): Promise<number[]>;
  setUserFeatureState(...): Promise<void>;
  setTeamFeatureState(...): Promise<void>;
}
```

#### 核心实现类 (`FeaturesRepository`)

在 `packages/features/flags/features.repository.ts` 中实现了主要逻辑：

**用户功能检查流程**：
```typescript
async checkIfUserHasFeature(userId: number, slug: string) {
  // 1. 首先检查用户级别的显式设置
  const userFeature = await this.prismaClient.userFeatures.findFirst({
    where: { userId, featureId: slug },
    select: { enabled: true },
  });

  if (userFeature) {
    return userFeature.enabled;
  }

  // 2. 如果没有用户级设置，检查用户所属的团队（支持递归继承）
  const userBelongsToTeamWithFeature = await this.checkIfUserBelongsToTeamWithFeature(userId, slug);
  return userBelongsToTeamWithFeature;
}
```

**团队层次递归检查**（使用 PostgreSQL CTE）：
```sql
WITH RECURSIVE TeamHierarchy AS (
  -- 从用户所属的团队开始
  SELECT DISTINCT t.id, t."parentId",
    CASE WHEN EXISTS (
      SELECT 1 FROM "TeamFeatures" tf
      WHERE tf."teamId" = t.id AND tf."featureId" = ${slug} AND tf."enabled" = true
    ) THEN true ELSE false END as has_feature
  FROM "Team" t
  INNER JOIN "Membership" m ON m."teamId" = t.id
  WHERE m."userId" = ${userId} AND m.accepted = true

  UNION ALL

  -- 递归获取父级团队
  SELECT DISTINCT p.id, p."parentId",
    CASE WHEN EXISTS (
      SELECT 1 FROM "TeamFeatures" tf
      WHERE tf."teamId" = p.id AND tf."featureId" = ${slug} AND tf."enabled" = true
    ) THEN true ELSE false END as has_feature
  FROM "Team" p
  INNER JOIN TeamHierarchy c ON p.id = c."parentId"
  WHERE NOT c.has_feature  -- 找到启用的团队后停止递归
)
SELECT 1 FROM TeamHierarchy WHERE has_feature = true LIMIT 1;
```

#### 分离的仓库类

项目还实现了更细粒度的分离仓库：

- `PrismaFeatureRepository` - 管理全局 Feature 定义
- `PrismaUserFeatureRepository` - 管理用户级 Feature 设置
- `PrismaTeamFeatureRepository` - 管理团队级 Feature 设置

### 3.2 缓存机制

项目实现了缓存装饰器模式来优化性能：

#### CachedTeamFeatureRepository
```typescript
export class CachedTeamFeatureRepository implements ITeamFeatureRepository {
  constructor(private prismaTeamFeatureRepository: ITeamFeatureRepository) {}

  @Memoize({
    key: (teamId, featureId) => `features:team:${teamId}:${featureId}`,
    schema: TeamFeaturesDtoSchema,
  })
  async findByTeamIdAndFeatureId(teamId: number, featureId: FeatureId): Promise<TeamFeaturesDto | null> {
    return this.prismaTeamFeatureRepository.findByTeamIdAndFeatureId(teamId, featureId);
  }

  @Unmemoize({
    keys: (teamId, featureId) => [
      `features:team:${teamId}:${featureId}`,
      `features:team:enabledFeatures:${teamId}`,
    ],
  })
  async upsert(teamId: number, featureId: FeatureId, enabled: boolean, assignedBy: string) {
    return this.prismaTeamFeatureRepository.upsert(teamId, featureId, enabled, assignedBy);
  }
}
```

**缓存策略**：
- 读取操作使用 `@Memoize` 装饰器缓存
- 写入操作使用 `@Unmemoize` 装饰器清除相关缓存
- 缓存键格式：`features:{entity}:{id}:{featureId}`

### 3.3 用例层 (Use Case Layer)

在 `packages/features/flags/operations/` 目录下：

```typescript
export function checkIfUserHasFeatureUseCase(userId: number, slug: string): Promise<boolean> {
  return startSpan({ name: "checkIfUserHasFeature UseCase", op: "function" }, async () => {
    const featuresRepository = new FeaturesRepository(prisma);
    return await featuresRepository.checkIfUserHasFeature(userId, slug);
  });
}
```

### 3.4 tRPC API 层

在 `packages/trpc/server/routers/features/_router.ts`：

```typescript
export const featureFlagRouter = router({
  list: publicProcedure.query(async () => {
    const featuresRepository = getFeaturesRepository();
    return featuresRepository.getAllFeatures();
  }),
  checkTeamFeature: publicProcedure
    .input(z.object({ teamId: z.number(), feature: z.string() }))
    .query(async ({ input }) => {
      const featuresRepository = getFeaturesRepository();
      return featuresRepository.checkIfTeamHasFeature(
        input.teamId,
        input.feature as keyof AppFlags
      );
    }),
  map,
});
```

### 3.5 前端集成

#### 1. Feature Provider
在 `apps/web/lib/app-providers-app-dir.tsx` 中：

```typescript
function FeatureFlagsProvider({ children }: { children: React.ReactNode }) {
  const flags = useFlags();
  return <FeatureProvider value={flags}>{children}</FeatureProvider>;
}

// 在 AppProviders 中使用
const AppProviders = (props: PageWrapperProps) => {
  return (
    <FeatureFlagsProvider>
      <OrgBrandProvider>{props.children}</OrgBrandProvider>
    </FeatureFlagsProvider>
  );
};
```

#### 2. useFlags Hook
在 `apps/web/modules/feature-flags/hooks/useFlags.ts`：

```typescript
const initialData: AppFlags = {
  "calendar-cache": false,
  "webhooks": false,
  "sink-shortener": false,
  // ... 所有功能默认关闭
};

export function useFlags(): Partial<AppFlags> {
  const query = trpc.viewer.features.map.useQuery();
  return query.data ?? initialData;
}
```

#### 3. useFlagMap Context Hook
在 `packages/features/flags/context/provider.ts`：

```typescript
export function useFlagMap() {
  const flagMapContext = useContext(FeatureContext);
  if (flagMapContext === null) 
    throw new Error("Error: useFlagMap was used outside of FeatureProvider.");
  return flagMapContext as Flags;
}
```

#### 4. 团队功能检查 Hook
在 `packages/features/flags/hooks/useIsFeatureEnabledForTeam.ts`：

```typescript
export const useIsFeatureEnabledForTeam = ({
  teamFeatures,
  teamId,
  feature,
}: {
  teamFeatures?: TeamFeatures;
  teamId?: number;
  feature: keyof AppFlags;
}) => {
  return useMemo(() => {
    if (!teamId || !teamFeatures?.[teamId]) return false;
    return teamFeatures[teamId][feature];
  }, [teamFeatures, teamId, feature]);
};
```

---

## 4. License Gate 实现

### 4.1 许可证验证流程

在 `packages/trpc/server/routers/viewer/deploymentSetup/validateLicense.handler.ts`：

```typescript
class LicenseKeyService {
  async checkLicense() { return true; }
  static async validateLicenseKey(_key?: string) { return true; }
}

export const validateLicenseHandler = async ({ input }: ValidateLicenseOptions) => {
  const { licenseKey } = input;

  // E2E 测试模式跳过验证
  if (process.env.NEXT_PUBLIC_IS_E2E === "1") {
    return {
      valid: true,
      message: "License key is valid (E2E mode)",
    };
  }

  try {
    const isValid = await LicenseKeyService.validateLicenseKey(licenseKey);

    return {
      valid: isValid,
      message: isValid ? "License key is valid" : "License key is invalid",
    };
  } catch (error) {
    console.error("License validation failed:", error);
    return {
      valid: false,
      message: "License key validation failed",
    };
  }
};
```

**当前状态**：LicenseKeyService 的 `validateLicenseKey` 方法目前返回 `true`，说明许可证验证系统处于**骨架阶段**，实际验证逻辑尚未实现。

### 4.2 自托管许可证创建

在 `packages/trpc/server/routers/viewer/admin/createSelfHostedLicenseKey.handler.ts`：

```typescript
const createSelfHostedInstance = async ({ input, ctx }: GetOptions) => {
  const privateApiUrl = CALCOM_PRIVATE_API_ROUTE;
  const signatureToken = process.env.CAL_SIGNATURE_TOKEN;

  if (!privateApiUrl || !signatureToken) {
    throw new Error("Private Api route does not exist in .env");
  }

  // 确保是管理员
  if (ctx.user.role !== "ADMIN") {
    console.warn(`${ctx.user.username} just tried to create a license key without permission`);
    throw new Error("You do not have permission to do this.");
  }

  // 使用签名认证调用私有 API
  const request = await fetchWithSignature(
    `${privateApiUrl}/v1/license`, 
    input, 
    signatureToken,
    { method: "POST" }
  );

  const data = await request.json();
  const schema = z.object({ stripeCheckoutUrl: z.string() });
  return schema.parse(data);
};
```

**签名机制**：
```typescript
const createSignature = (body: Record<string, unknown>, nonce: string, secretKey: string): string => {
  return crypto
    .createHmac("sha256", secretKey)
    .update(JSON.stringify(body) + nonce)
    .digest("hex");
};
```

### 4.3 页面级许可证控制

在 `apps/web/components/PageWrapperAppDir.tsx` 和 `apps/web/components/PageWrapper.tsx` 中：

```typescript
export type PageWrapperProps = Readonly<{
  children: React.ReactNode;
  requiresLicense: boolean;
  nonce: string | undefined;
  isBookingPage?: boolean;
}>;

function PageWrapper(props: PageWrapperProps) {
  return (
    <AppProviders {...providerProps}>
      {props.requiresLicense ? (
        <>{props.children}</>
      ) : (
        <>{props.children}</>
      )}
    </AppProviders>
  );
}
```

**注意**：当前实现中 `requiresLicense` 属性存在但未被实际用于条件渲染，两个分支都渲染相同的内容。这表明许可证控制逻辑可能尚未完全实现或被临时绕过。

---

## 5. 实际使用场景

### 5.1 URL 缩短器功能控制

在 `packages/features/url-shortener/UrlShortenerFactory.ts` 中展示了完整的检查流程：

```typescript
export class UrlShortenerFactory {
  static async create({ userId, teamId }: { userId?: number | null; teamId?: number | null } = {}) {
    if (SinkShortener.isConfigured()) {
      const featuresRepository = new FeaturesRepository(prisma);

      // 1. 检查全局开关
      const globallyEnabled = await featuresRepository.checkIfFeatureIsEnabledGlobally("sink-shortener");
      if (globallyEnabled) {
        return new SinkShortener(new SinkClient());
      }

      // 2. 检查用户级别
      if (userId) {
        const useSink = await featuresRepository.checkIfUserHasFeature(userId, "sink-shortener");
        if (useSink) {
          return new SinkShortener(new SinkClient());
        }
      }

      // 3. 检查团队级别
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

**检查优先级**：全局开关 > 用户设置 > 团队设置

### 5.2 团队层次结构继承

`checkIfTeamHasFeature` 方法支持递归向上查找父团队：

```typescript
async checkIfTeamHasFeature(teamId: number, featureId: FeatureId): Promise<boolean> {
  // 1. 先检查当前团队是否有显式设置
  const teamFeature = await this.prismaClient.teamFeatures.findUnique({
    where: { teamId_featureId: { teamId, featureId } },
    select: { enabled: true },
  });
  if (teamFeature) return teamFeature.enabled;

  // 2. 使用递归 CTE 向上查找父级团队
  // 如果找到任意父团队启用了该功能，则返回 true
}
```

**适用场景**：组织级功能可以自动继承到子团队，但子团队可以通过显式禁用来覆盖。

---

## 6. 架构总结

### 6.1 数据流图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         前端应用层                                  │
│  ┌─────────────┐    ┌────────────────────┐    ┌─────────────────┐  │
│  │ useFlags()  │    │ useFlagMap()       │    │ 页面组件         │  │
│  │             │    │ (Context)          │    │ (requiresLicense)│  │
│  └──────┬──────┘    └─────────┬──────────┘    └────────┬────────┘  │
│         │                     │                        │           │
└─────────┼─────────────────────┼────────────────────────┼───────────┘
          │                     │                        │
          ▼                     ▼                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         tRPC API 层                                 │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  featureFlagRouter                                          │    │
│  │  - list: 获取所有 Feature 定义                               │    │
│  │  - map: 获取 Feature 状态映射                               │    │
│  │  - checkTeamFeature: 检查团队功能                           │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         用例层 (Use Cases)                          │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  checkIfUserHasFeatureUseCase()                             │    │
│  │  - 包装 Repository 调用                                      │    │
│  │  - 添加 Sentry 性能追踪                                      │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────┬───────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         仓库层 (Repository)                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────────┐ │
│  │ Cached*Repo    │  │ Prisma*Repo     │  │ FeaturesRepository │ │
│  │ (装饰器模式)    │  │ (数据库访问)     │  │ (统一门面)         │ │
│  └────────┬────────┘  └────────┬────────┘  └─────────┬──────────┘ │
│           │                    │                      │            │
└───────────┼────────────────────┼──────────────────────┼────────────┘
            │                    │                      │
            ▼                    ▼                      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         数据层                                       │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────────┐ │
│  │ Feature      │  │ UserFeatures     │  │ TeamFeatures         │ │
│  │ (全局定义)   │  │ (用户级覆盖)      │  │ (团队级覆盖 + 继承)  │ │
│  └──────────────┘  └──────────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 设计特点

| 特点 | 实现方式 | 优势 |
|------|---------|------|
| **多层级控制** | 全局 → 团队 → 用户 | 灵活的权限分配 |
| **三态语义** | enabled / disabled / inherit | 支持继承和显式覆盖 |
| **递归继承** | PostgreSQL CTE | 支持复杂的团队层次结构 |
| **缓存优化** | @Memoize / @Unmemoize 装饰器 | 减少数据库查询 |
| **前后端一致** | tRPC 类型安全 | 避免类型不匹配 |
| **可测试性** | 接口抽象 + 依赖注入 | 便于单元测试 |

---

## 7. 关键文件索引

### Feature Flag 系统
| 路径 | 描述 |
|------|------|
| `packages/features/flags/config.ts` | 功能标志类型定义 |
| `packages/features/flags/features.repository.interface.ts` | 仓库接口定义 |
| `packages/features/flags/features.repository.ts` | 核心仓库实现 |
| `packages/features/flags/repositories/PrismaFeatureRepository.ts` | 全局 Feature 仓库 |
| `packages/features/flags/repositories/PrismaUserFeatureRepository.ts` | 用户级 Feature 仓库 |
| `packages/features/flags/repositories/PrismaTeamFeatureRepository.ts` | 团队级 Feature 仓库 |
| `packages/features/flags/repositories/Cached*Repository.ts` | 缓存装饰器实现 |
| `packages/features/flags/hooks/useIsFeatureEnabledForTeam.ts` | 前端团队功能检查 Hook |
| `packages/features/flags/context/provider.ts` | React Context 提供者 |
| `apps/web/modules/feature-flags/hooks/useFlags.ts` | 前端获取 Feature 标志 Hook |
| `packages/trpc/server/routers/features/_router.ts` | Feature Flag tRPC 路由 |

### License Gate 系统
| 路径 | 描述 |
|------|------|
| `packages/trpc/server/routers/viewer/deploymentSetup/validateLicense.handler.ts` | 许可证验证处理器 |
| `packages/trpc/server/routers/viewer/admin/createSelfHostedLicenseKey.handler.ts` | 自托管许可证创建 |
| `packages/features/deployment/repositories/DeploymentRepository.ts` | Deployment 数据访问 |
| `apps/web/components/PageWrapper.tsx` | Pages Router 页面包装器 |
| `apps/web/components/PageWrapperAppDir.tsx` | App Router 页面包装器 |

### 数据模型
| 路径 | 描述 |
|------|------|
| `packages/prisma/schema.prisma` | 数据库 schema（包含 Feature/UserFeatures/TeamFeatures/Deployment 模型） |

---

*本报告基于 2026-05-10 的代码库状态分析生成。*
