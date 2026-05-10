# Cal.diy 企业功能可见性控制机制分析

## 目录

- [1. 核心结论](#1-核心结论)
- [2. 已实现的企业功能：hideBranding（品牌去除）](#2-已实现的企业功能hidebranding品牌去除)
  - [2.1 功能概述](#21-功能概述)
  - [2.2 hasPaidPlan 判定链（设置页面侧）](#22-haspaidplan-判定链设置页面侧)
  - [2.3 hideBranding 判定链（预订页面侧）](#23-hidebranding-判定链预订页面侧)
  - [2.4 完整链路图](#24-完整链路图)
- [3. 未实现的 License Gate 机制](#3-未实现的-license-gate-机制)
  - [3.1 session.hasValidLicense 路径](#31-sessionhasvalidlicense-路径)
  - [3.2 setup 页面 hasValidLicense 路径](#32-setup-页面-hasvalidlicense-路径)
  - [3.3 两条路径为何都不生效](#33-两条路径为何都不生效)
- [4. 关键代码索引](#4-关键代码索引)

---

## 1. 核心结论

| 机制 | 状态 | 说明 |
|------|------|------|
| **hasPaidPlan → hideBranding** | ✅ **已实现** | 控制"去除品牌标识"功能的可见性和启用 |
| **session.hasValidLicense** | ⚠️ **占位** | 永远为 `true`，传递但未被实际使用 |
| **setup.hasValidLicense** | ⚠️ **占位** | 硬编码为 `false`，不影响任何 UI |
| **UpgradeTip** | ⚠️ **占位** | 直接渲染 children，无付费墙 |
| **requiresLicense** | ⚠️ **占位** | 两个分支渲染相同内容 |

**重要区分**：
- Cal.diy 有**实际生效**的企业功能 gate（`hasPaidPlan` 控制的 `hideBranding`）
- 但 License Gate 系统（`hasValidLicense` 相关）**全部是占位实现**，未实际控制任何功能

---

## 2. 已实现的企业功能：hideBranding（品牌去除）

### 2.1 功能概述

**hideBranding** 是 Cal.diy 中唯一完全实现的企业功能 gate：

| 功能点 | 说明 |
|--------|------|
| **功能内容** | 去除预订页面底部/角落的 "Powered by Cal.com" 等品牌标识 |
| **控制层级** | 团队/组织/用户三级 |
| **gate 机制** | `hasPaidPlan` 控制开关可用性，`hideBranding` 设置控制实际展示 |

### 2.2 hasPaidPlan 判定链（设置页面侧）

#### 入口页面
- **URL**: `/settings/my-account/appearance`
- **文件**: `apps/web/app/(use-page-wrapper)/settings/(settings-layout)/my-account/appearance/page.tsx`

#### 完整判定链路

```
用户请求: GET /settings/my-account/appearance
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 检查登录状态                                                    │
│    const session = await getServerSession({ req: buildLegacyRequest(...) });│
│    const userId = session?.user?.id;                             │
│    if (!userId) redirect(redirectUrl);                            │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 并行获取两个关键数据                                            │
│                                                                  │
│    2a. 创建 meRouter 调用器                                       │
│    const [meCaller, hasTeamPlan] = await Promise.all([           │
│      createRouterCaller(meRouter),                                │
│      getCachedHasTeamPlan(userId),  // ← 检查是否属于团队        │
│    ]);                                                           │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. getCachedHasTeamPlan 的详细逻辑                                │
│    位置: apps/web/app/cache/membership.ts:11-22                  │
│                                                                  │
│    export const getCachedHasTeamPlan = unstable_cache(           │
│      async (userId: number) => {                                  │
│        const hasTeamPlan = await MembershipRepository            │
│          .hasAnyAcceptedMembershipByUserId(userId);              │
│        // → 查询数据库是否有已接受的团队成员关系                   │
│        return { hasTeamPlan: !!hasTeamPlan };                     │
│      },                                                          │
│      ["getCachedHasTeamPlan"],                                   │
│      { revalidate: NEXTJS_CACHE_TTL, tags: [...] }               │
│    );                                                           │
│                                                                  │
│    结果: { hasTeamPlan: true/false }                              │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 获取用户数据                                                   │
│    const user = await meCaller.get();                            │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 计算 hasPaidPlan（关键判定点）                                  │
│    位置: apps/web/app/(use-page-wrapper)/settings/(settings-layout)/my-account/appearance/page.tsx:46│
│                                                                  │
│    // 检查用户 metadata 中是否有 isPremium 标记                    │
│    const isCurrentUsernamePremium =                              │
│      user && hasKeyInMetadata(user, "isPremium")                 │
│        ? !!user.metadata.isPremium                               │
│        : false;                                                  │
│                                                                  │
│    // 最终计算:                                                   │
│    const hasPaidPlan = IS_SELF_HOSTED                            │
│      ? true                     // 自托管: 永远 true            │
│      : hasTeamPlan?.hasTeamPlan   // 非自托管: 检查是否有团队     │
│        || isCurrentUsernamePremium; // 或是否是 Premium 用户      │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 传递给 AppearancePage 组件                                    │
│    return <AppearancePage user={user} hasPaidPlan={hasPaidPlan} />;│
└─────────────────────────────────────────────────────────────────┘
```

#### hasPaidPlan 判定逻辑总结

```
hasPaidPlan = ?

    IS_SELF_HOSTED = true?
        │
        ├─ 是 → hasPaidPlan = true (自托管版本认为所有用户都是付费用户)
        │
        └─ 否 → hasPaidPlan = hasTeamPlan || isCurrentUsernamePremium
                    │
                    ├─ hasTeamPlan: 用户是否属于任意团队
                    │              MembershipRepository.hasAnyAcceptedMembershipByUserId()
                    │              数据库查询: Membership where userId = ? AND accepted = true
                    │
                    └─ isCurrentUsernamePremium: 用户 metadata.isPremium = true
```

#### 在 AppearancePage 中的使用

**文件**: `apps/web/modules/settings/my-account/appearance-view.tsx:390-402`

```tsx
<SettingsToggle
  toggleSwitchAtTheEnd={true}
  title={t("disable_cal_branding", { appName: APP_NAME })}
  
  // ⚠️ 关键 1: 如果 !hasPaidPlan，开关被禁用（灰色不可点击）
  disabled={!hasPaidPlan || mutation?.isPending}
  
  description={t("removes_cal_branding", { appName: APP_NAME })}
  
  // ⚠️ 关键 2: 开关显示值
  // 如果有 paid plan，显示用户的 hideBranding 设置
  // 如果没有 paid plan，永远显示 false
  checked={hasPaidPlan ? hideBrandingValue : false}
  
  onCheckedChange={(checked) => {
    setHideBrandingValue(checked);
    mutation.mutate({ hideBranding: checked });
  }}
  switchContainerClassName="mt-6"
/>
```

**行为**：

| 场景 | 开关状态 | 用户能做什么 |
|------|---------|-------------|
| `hasPaidPlan = true` | 显示用户的 `hideBrandingValue` | 可点击切换，保存到数据库 |
| `hasPaidPlan = false` | 永远显示 false | 灰色禁用，无法切换 |

### 2.3 hideBranding 判定链（预订页面侧）

#### 预订页面入口
- **URL**: `/[user]/[type]`（如 `/john/30min-meeting`）
- **类型**: Pages Router (getServerSideProps)

#### 完整判定链路

```
用户请求: GET /[user]/[type]
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. getServerSideProps 服务端执行                                  │
│    位置: apps/web/server/lib/[user]/[type]/getServerSideProps.tsx│
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 计算 isBrandingHidden（关键判定点）                             │
│    位置: 第 271-274 行                                           │
│                                                                  │
│    isBrandingHidden: shouldHideBrandingForUserEvent({            │
│      eventTypeId: eventData.id,                                  │
│      owner: user,                                                │
│    }),                                                          │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. shouldHideBrandingForUserEvent 详细逻辑                       │
│    位置: packages/features/profile/lib/hideBranding.ts:172-184  │
│                                                                  │
│    export function shouldHideBrandingForUserEvent({              │
│      eventTypeId,                                                │
│      owner,                                                      │
│    }: {                                                          │
│      eventTypeId: number;                                        │
│      owner: UserWithProfile;                                     │
│    }) {                                                          │
│      return shouldHideBrandingForEventUsingProfile({             │
│        owner,                                                    │
│        team: null,                                               │
│        eventTypeId,                                              │
│      });                                                         │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. shouldHideBrandingForEventUsingProfile 核心判定               │
│    位置: packages/features/profile/lib/hideBranding.ts:88-113  │
│                                                                  │
│    export function shouldHideBrandingForEventUsingProfile({      │
│      eventTypeId,                                                │
│      owner,                                                      │
│      team,                                                       │
│    }) {                                                          │
│      let hideBranding;                                           │
│      if (team) {                                                 │
│        // 团队事件: 检查团队 + 组织设置                            │
│        hideBranding = resolveHideBranding({                      │
│          entityHideBranding: team.hideBranding ?? null,          │
│          organizationHideBranding: team.parent?.hideBranding ?? null,│
│        });                                                       │
│      } else if (owner) {                                         │
│        // 用户事件: 检查用户 + 组织设置                            │
│        hideBranding = resolveHideBranding({                      │
│          entityHideBranding: owner.hideBranding ?? null,         │
│          organizationHideBranding:                               │
│            owner.profile?.organization?.hideBranding ?? null,    │
│        });                                                       │
│      } else {                                                    │
│        log.error(`No owner or team found for event: ${eventTypeId}`);│
│        return false;                                             │
│      }                                                           │
│      return hideBranding;                                        │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. resolveHideBranding 优先级规则                                 │
│    位置: packages/features/profile/lib/hideBranding.ts:33-43   │
│                                                                  │
│    function resolveHideBranding(options: {                       │
│      entityHideBranding: boolean | null;                         │
│      organizationHideBranding: boolean | null;                   │
│    }): boolean {                                                 │
│      // ⚠️ 关键: 组织级别优先级高于实体级别                        │
│      // 如果组织设置了 hideBranding=true，忽略实体自己的设置        │
│      if (options.organizationHideBranding) {                     │
│        return true;                                              │
│      }                                                           │
│      // 否则使用实体自己的设置，null 回退到 false                  │
│      return options.entityHideBranding ?? false;                 │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 传递给前端页面组件                                             │
│    返回 props: { isBrandingHidden, ... }                         │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. 前端页面组件接收                                               │
│    位置: apps/web/modules/users/views/users-type-public-view.tsx:27│
│                                                                  │
│    function Type({ ..., isBrandingHidden, ... }: PageProps) {    │
│      return (                                                    │
│        <Booker                                                   │
│          ...                                                     │
│          hideBranding={isBrandingHidden}  // ⚠️ 传递给 Booker   │
│          ...                                                     │
│        />                                                        │
│      );                                                          │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
```

#### 预订页 hideBranding 判定总结

**优先级（从高到低）**：

```
组织 (Organization) hideBranding = true?
    │
    ├─ 是 → 隐藏品牌（最高优先级，覆盖实体设置）
    │
    └─ 否 → 实体 (User/Team) hideBranding = true?
             │
             ├─ 是 → 隐藏品牌
             │
             └─ 否 / null → 显示品牌（默认）
```

**注意**：预订页的 `shouldHideBrandingForEvent` **不检查 `hasPaidPlan`**

这意味着：
- **设置页面**：`hasPaidPlan` 控制"是否允许启用"
- **预订页面**：只看数据库中 `hideBranding` 字段的值，不检查是否付费

**潜在问题**：如果直接修改数据库绕过设置页面的限制，可能可以启用这个功能而不付费。

### 2.4 完整链路图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          已实现的企业功能: hideBranding                     │
└─────────────────────────────────────────────────────────────────────────────┘

                         ┌─────────────────────────────────┐
                         │         设置页面侧              │
                         │  /settings/my-account/appearance │
                         └──────────────┬──────────────────┘
                                        │
                                        ▼
                    ┌───────────────────────────────────────┐
                    │ 1. 计算 hasPaidPlan                   │
                    │                                      │
                    │    自托管? ────┐                      │
                    │       │        │                      │
                    │      是        否                     │
                    │       │        │                      │
                    │       ▼        ▼                      │
                    │     true    hasTeamPlan ||            │
                    │               isCurrentUsernamePremium│
                    │         ↓                            │
                    │    hasPaidPlan = ?                   │
                    └──────────────┬───────────────────────┘
                                   │
                                   ▼
                    ┌───────────────────────────────────────┐
                    │ 2. 控制 hideBranding 开关              │
                    │                                      │
                    │    hasPaidPlan = true?               │
                    │       │                               │
                    │       ├─ 是 → 开关可用，保存到数据库   │
                    │       └─ 否 → 开关禁用，无法切换       │
                    └──────────────┬───────────────────────┘
                                   │
                                   │ 保存到:
                                   │  - User.hideBranding
                                   │  - Team.hideBranding
                                   │  - Organization.hideBranding
                                   ▼

                         ┌─────────────────────────────────┐
                         │         预订页面侧              │
                         │        /[user]/[type]          │
                         └──────────────┬──────────────────┘
                                        │
                                        ▼
                    ┌───────────────────────────────────────┐
                    │ 1. 计算 isBrandingHidden              │
                    │                                      │
                    │    shouldHideBrandingForEvent()      │
                    │         ↓                            │
                    │    resolveHideBranding()             │
                    │         ↓                            │
                    │    组织 hideBranding = true?         │
                    │       │                               │
                    │       ├─ 是 → isBrandingHidden = true│
                    │       └─ 否 → 实体 hideBranding = ?   │
                    │                │                      │
                    │                ├─ true → 隐藏         │
                    │                └─ false/null → 显示   │
                    └──────────────┬───────────────────────┘
                                   │
                                   ▼
                    ┌───────────────────────────────────────┐
                    │ 2. 传递给 Booker 组件                 │
                    │    hideBranding={isBrandingHidden}   │
                    │                                      │
                    │    Booker 根据此值决定是否渲染品牌标识 │
                    └───────────────────────────────────────┘
```

---

## 3. 未实现的 License Gate 机制

### 3.1 session.hasValidLicense 路径

#### 入口
- **注入点**: Session 创建时
- **文件**: `packages/features/auth/lib/getServerSession.ts`

#### 完整链路

```
Session 初始化
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 创建 LicenseKeySingleton                                      │
│    位置: packages/features/auth/lib/getServerSession.ts:11-15   │
│                                                                  │
│    class LicenseKeySingleton {                                   │
│      static async getInstance(..._args: unknown[]) {             │
│        return new LicenseKeySingleton();  // ⚠️ 忽略所有参数   │
│      }                                                           │
│                                                                  │
│      async checkLicense() {                                      │
│        return true;  // ⚠️ 直接返回 true，无任何校验            │
│      }                                                           │
│                                                                  │
│      async validateLicenseKey() {                                │
│        return true;  // ⚠️ 直接返回 true，无任何校验            │
│      }                                                           │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 调用 checkLicense()                                           │
│                                                                  │
│    const deploymentRepo = new DeploymentRepository(prisma);      │
│    const licenseKeyService = await LicenseKeySingleton.getInstance(│
│      deploymentRepo                                              │
│    );                                                            │
│    const hasValidLicense = await licenseKeyService.checkLicense();│
│    // → hasValidLicense = true (永远)                            │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 注入到 Session                                                │
│                                                                  │
│    const session: Session = {                                    │
│      hasValidLicense,  // = true (永远)                          │
│      // ... 其他字段                                              │
│    };                                                             │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 前端传递值                                                    │
│    位置: apps/web/modules/bookings/components/BookerWebWrapper.tsx:223│
│                                                                  │
│    hasValidLicense={session?.hasValidLicense ?? false}           │
│    // → 值永远为 true（因为 session.hasValidLicense 永远是 true）│
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. Booker 组件接收                                                │
│    位置: apps/web/modules/bookings/components/Booker.tsx:80      │
│                                                                  │
│    const BookerComponent = ({                                    │
│      ...                                                         │
│      hasValidLicense,  // ⚠️ 接收了这个 prop                     │
│      ...                                                         │
│    })                                                            │
│                                                                  │
│    ⚠️ 关键问题: 整个 Booker.tsx 文件中                            │
│       没有任何地方使用 hasValidLicense 做条件判断                 │
│       没有 if (hasValidLicense)                                  │
│       没有 hasValidLicense ? 三元表达式                          │
│       完全未被使用！                                             │
└─────────────────────────────────────────────────────────────────┘
```

#### 结论

`session.hasValidLicense` **不生效**的原因：
1. **值层面**: `LicenseKeySingleton.checkLicense()` 永远返回 `true`
2. **使用层面**: Booker 组件接收了这个 prop 但**从未使用**

### 3.2 setup 页面 hasValidLicense 路径

#### 入口
- **URL**: `/auth/setup?step=1`
- **文件**: `apps/web/server/lib/setup/getServerSideProps.tsx`

#### 完整链路

```
用户请求: GET /auth/setup?step=1
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. getServerSideProps 执行                                       │
│    位置: apps/web/server/lib/setup/getServerSideProps.tsx:38-49 │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 硬编码 hasValidLicense                                        │
│                                                                  │
│    // Check if there's already a valid license using            │
│    // LicenseKeyService                                          │
│    const hasValidLicense = false;   // ⚠️ 硬编码为 false        │
│    const isFreeLicense = true;      // ⚠️ 硬编码为 true         │
│                                                                  │
│    ⚠️ 关键问题: 注释说使用 LicenseKeyService                     │
│       但实际代码直接硬编码，没有调用任何服务                      │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 返回给页面                                                    │
│                                                                  │
│    return {                                                      │
│      props: {                                                    │
│        isFreeLicense,     // = true                              │
│        userCount,                                                 │
│        hasValidLicense,   // = false                             │
│      },                                                          │
│    };                                                            │
└─────────────────────────────────────────────────────────────────┘
```

#### 结论

`setup` 页面的 `hasValidLicense` **不生效**的原因：
1. **值层面**: 直接硬编码为 `false`
2. **使用层面**: Setup 页面可能也没有根据这个值做任何条件判断

### 3.3 两条路径为何都不生效

#### 问题一：LicenseKeyService 是占位实现

所有与许可证相关的类/方法都是占位：

| 方法 | 位置 | 实际行为 |
|------|------|---------|
| `LicenseKeySingleton.checkLicense()` | `getServerSession.ts` | `return true` |
| `LicenseKeySingleton.validateLicenseKey()` | `getServerSession.ts` | `return true` |
| `LicenseKeyService.validateLicenseKey()` | `validateLicense.handler.ts` | `return true` |
| `DeploymentsService.checkLicense()` | `API v2` | `return true` |

**没有任何方法**：
- 读取数据库 `Deployment.licenseKey`
- 验证许可证签名
- 检查过期时间
- 连接外部许可证服务

#### 问题二：没有实际的条件判断

即使 `hasValidLicense` 值被传递：

| 使用位置 | 实际行为 |
|---------|---------|
| Booker 组件 | 接收 prop 但从未使用 |
| PageWrapper.requiresLicense | 两个分支渲染相同内容 |
| UpgradeTip 组件 | 直接渲染 children |

**没有任何地方**：
- `if (!hasValidLicense) return <UpgradeBanner />`
- `if (!hasValidLicense) redirect to /upgrade`
- 禁用某些按钮或隐藏某些菜单

#### 问题三：代码注释明确说明是开源版本

**证据 1**: UpgradeTip 组件
```typescript
// In the open-source distribution there is no paywall – always render children.
return <>{children}</>;
```

**证据 2**: API v2 DeploymentsService
```typescript
// Cal.diy is fully open source — no license key is required.
async checkLicense() {
  return true;
}
```

---

## 4. 关键代码索引

### 4.1 已实现的企业功能（hideBranding 链路）

| 文件路径 | 说明 | 关键代码 |
|---------|------|---------|
| `apps/web/app/(use-page-wrapper)/settings/(settings-layout)/my-account/appearance/page.tsx:46` | hasPaidPlan 计算 | `const hasPaidPlan = IS_SELF_HOSTED ? true : hasTeamPlan?.hasTeamPlan \|\| isCurrentUsernamePremium` |
| `apps/web/app/cache/membership.ts:13-15` | hasTeamPlan 查询 | `MembershipRepository.hasAnyAcceptedMembershipByUserId(userId)` |
| `apps/web/modules/settings/my-account/appearance-view.tsx:393` | 开关禁用条件 | `disabled={!hasPaidPlan \|\| mutation?.isPending}` |
| `apps/web/modules/settings/my-account/appearance-view.tsx:395` | 开关显示值 | `checked={hasPaidPlan ? hideBrandingValue : false}` |
| `packages/features/profile/lib/hideBranding.ts:33-43` | resolveHideBranding | 组织优先级高于实体 |
| `packages/features/profile/lib/hideBranding.ts:88-113` | shouldHideBrandingForEventUsingProfile | 预订页判定逻辑 |
| `apps/web/server/lib/[user]/[type]/getServerSideProps.tsx:271-274` | 预订页 isBrandingHidden | `shouldHideBrandingForUserEvent({...})` |
| `apps/web/modules/users/views/users-type-public-view.tsx:37` | 传递给 Booker | `hideBranding={isBrandingHidden}` |

### 4.2 未实现的 License Gate（占位）

| 文件路径 | 说明 | 状态 |
|---------|------|------|
| `packages/features/auth/lib/getServerSession.ts:11-15` | `LicenseKeySingleton` 占位实现 | ⚠️ 永远返回 true |
| `apps/web/modules/bookings/components/BookerWebWrapper.tsx:223` | 传递 hasValidLicense | ⚠️ 传递但 Booker 不使用 |
| `apps/web/server/lib/setup/getServerSideProps.tsx:38-49` | setup 页面硬编码 | ⚠️ 永远为 false |
| `apps/web/components/PageWrapperAppDir.tsx:29-33` | requiresLicense 渲染逻辑 | ⚠️ 两个分支相同 |
| `apps/web/modules/shell/UpgradeTip.tsx:1-17` | UpgradeTip 组件 | ⚠️ 直接渲染 children |
| `apps/api/v2/src/modules/deployments/deployments.service.ts:14-17` | API v2 checkLicense | ⚠️ 注释说明无需许可证 |

---

## 5. 最终总结

### 实际生效的企业功能 Gate

Cal.diy 有**一个**实际生效的企业功能控制机制：

```
hasPaidPlan 判定链（已实现）
├─ 自托管: 永远为 true
└─ 非自托管:
   ├─ hasTeamPlan: 用户是否属于任意团队
   └─ isCurrentUsernamePremium: 用户 metadata.isPremium

结果:
├─ hasPaidPlan = true → hideBranding 开关可用
└─ hasPaidPlan = false → hideBranding 开关禁用
```

### 未生效的 License Gate

所有 `hasValidLicense` 相关的机制**都不生效**：

```
session.hasValidLicense 路径（不生效）
├─ LicenseKeySingleton.checkLicense() → 永远返回 true
└─ Booker 组件接收但未使用

setup.hasValidLicense 路径（不生效）
└─ 直接硬编码为 false，不影响任何 UI
```

### 根本原因

代码注释明确说明 Cal.diy 是开源版本：
> "In the open-source distribution there is no paywall"
> "Cal.diy is fully open source — no license key is required"

因此，所有 License Gate 相关的代码都是从上游 Cal.com 继承的骨架代码，在 Cal.diy 中被有意地简化或绕过。

---

*本报告基于 2026-05-10 的代码库状态分析生成。*
