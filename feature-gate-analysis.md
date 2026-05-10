# Cal.diy 企业功能可见性控制机制分析

## 目录

- [1. 四条链路总览](#1-四条链路总览)
- [2. 链路对照总表](#2-链路对照总表)
- [3. 链路 1: hasPaidPlan → hideBranding 开关控制（已实现）](#3-链路-1-haspaidplan--hidebranding-开关控制已实现)
- [4. 链路 2: hideBranding → 预订页品牌展示（已实现）](#4-链路-2-hidebranding--预订页品牌展示已实现)
- [5. 链路 3: session.hasValidLicense（路径 A：next-auth callback）](#5-链路-3-sessionhasvalidlicense路径-anext-auth-callback)
- [6. 链路 4: session.hasValidLicense（路径 B：自定义 getServerSession）](#6-链路-4-sessionhasvalidlicense路径-b自定义-getserversession)
- [7. 关键代码索引](#7-关键代码索引)

---

## 1. 四条链路总览

Cal.diy 的企业功能可见性控制包含 **四条独立链路**，其中只有两条实际生效：

| 链路 | 类型 | 状态 | 最终影响 |
|------|------|------|---------|
| **1. hasPaidPlan → hideBranding 开关** | 用户侧 gate | ✅ 已实现 | 控制设置页面中"去除品牌"开关是否可用 |
| **2. hideBranding → 预订页品牌展示** | 展示层 | ✅ 已实现 | 根据数据库字段决定是否显示品牌标识 |
| **3. session.hasValidLicense（路径 A）** | License Gate | ⚠️ 占位（硬编码 false） | 从未被实际使用 |
| **4. session.hasValidLicense（路径 B）** | License Gate | ⚠️ 占位（永远 true） | 从未被实际使用 |

**关键发现**：
- `session.hasValidLicense` 有 **两条完全独立的路径**，返回值不一致（一个 false，一个 true）
- 但**两条路径都不影响任何实际功能**（Booker 组件接收但不使用）
- 真正控制企业功能的是 `hasPaidPlan → hideBranding` 链路

---

## 2. 链路对照总表

| 维度 | 链路 1: hasPaidPlan 开关 | 链路 2: hideBranding 展示 | 链路 3: session callback | 链路 4: getServerSession |
|------|-------------------------|--------------------------|--------------------------|--------------------------|
| **入口** | `/settings/my-account/appearance` | `/[user]/[type]` 预订页 | 前端 `useSession()` | 服务端 `getServerSession()` |
| **关键文件** | `appearance/page.tsx:46` | `getServerSideProps.tsx:271-274` | `next-auth-options.ts:765` | `getServerSession.ts:82` |
| **值来源** | `IS_SELF_HOSTED ? true : hasTeamPlan \|\| isPremium` | `User.hideBranding` / `Organization.hideBranding` | 硬编码 `false` | `LicenseKeySingleton.checkLicense()` → `return true` |
| **最终值** | true/false（动态） | true/false（数据库字段） | `false`（硬编码） | `true`（永远） |
| **是否被使用** | ✅ 是 | ✅ 是 | ❌ 否 | ❌ 否 |
| **实际影响** | 开关是否可用 | 品牌是否显示 | 无 | 无 |

---

## 3. 链路 1: hasPaidPlan → hideBranding 开关控制（已实现）

### 3.1 入口页面
- **URL**: `/settings/my-account/appearance`
- **类型**: App Router Server Component
- **文件**: `apps/web/app/(use-page-wrapper)/settings/(settings-layout)/my-account/appearance/page.tsx`

### 3.2 完整判定链

```
用户请求: GET /settings/my-account/appearance
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 检查登录状态并获取 userId                                     │
│    const session = await getServerSession({ req: buildLegacyRequest(...) });│
│    const userId = session?.user?.id;                             │
│    if (!userId) redirect(redirectUrl);                            │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 并行获取用户数据和团队状态                                     │
│                                                                  │
│    const [meCaller, hasTeamPlan] = await Promise.all([           │
│      createRouterCaller(meRouter),                                │
│      getCachedHasTeamPlan(userId),  // ← 关键数据                │
│    ]);                                                           │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. getCachedHasTeamPlan 详细逻辑                                 │
│    位置: apps/web/app/cache/membership.ts:11-22                  │
│                                                                  │
│    export const getCachedHasTeamPlan = unstable_cache(           │
│      async (userId: number) => {                                  │
│        const hasTeamPlan = await MembershipRepository            │
│          .hasAnyAcceptedMembershipByUserId(userId);              │
│        // → 查询: Membership where userId = ? AND accepted = true│
│        return { hasTeamPlan: !!hasTeamPlan };                     │
│      },                                                          │
│      ["getCachedHasTeamPlan"],                                   │
│      { revalidate: NEXTJS_CACHE_TTL }                            │
│    );                                                           │
│                                                                  │
│    结果: { hasTeamPlan: true/false }                              │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 获取用户数据并检查 isPremium                                  │
│    const user = await meCaller.get();                            │
│                                                                  │
│    // 检查用户 metadata 中是否有 isPremium 标记                   │
│    const isCurrentUsernamePremium =                              │
│      user && hasKeyInMetadata(user, "isPremium")                 │
│        ? !!user.metadata.isPremium                               │
│        : false;                                                  │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 计算 hasPaidPlan（关键判定点）                                 │
│    位置: apps/web/app/(use-page-wrapper)/settings/(settings-layout)/my-account/appearance/page.tsx:46│
│                                                                  │
│    const hasPaidPlan = IS_SELF_HOSTED                            │
│      ? true                     // 自托管: 永远 true            │
│      : hasTeamPlan?.hasTeamPlan   // 非自托管: 检查是否有团队     │
│        || isCurrentUsernamePremium; // 或是否是 Premium 用户      │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 hasPaidPlan 判定逻辑总结

```
hasPaidPlan = ?

    IS_SELF_HOSTED = true?
        │
        ├─ 是 → hasPaidPlan = true
        │       (自托管版本认为所有用户都是付费用户)
        │
        └─ 否 → hasPaidPlan = hasTeamPlan || isCurrentUsernamePremium
                    │
                    ├─ hasTeamPlan: 用户是否属于任意团队
                    │              MembershipRepository.hasAnyAcceptedMembershipByUserId()
                    │              数据库: Membership where userId = ? AND accepted = true
                    │
                    └─ isCurrentUsernamePremium: 用户 metadata.isPremium = true
```

### 3.4 在外观设置页面中的使用

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

### 3.5 行为对照表

| 场景 | hasPaidPlan 值 | 开关状态 | 用户能做什么 |
|------|---------------|---------|-------------|
| 自托管环境 | `true` | 显示用户设置 | 可切换，保存到数据库 |
| 非自托管 + 有团队 | `true` | 显示用户设置 | 可切换，保存到数据库 |
| 非自托管 + metadata.isPremium | `true` | 显示用户设置 | 可切换，保存到数据库 |
| 非自托管 + 无团队 + 非 Premium | `false` | 永远显示 false | 灰色禁用，无法切换 |

### 3.6 数据保存位置

当用户切换开关并保存时，`hideBranding` 值会保存到以下位置：

| 层级 | 数据库字段 | 说明 |
|------|-----------|------|
| 用户 | `User.hideBranding` | 用户级别设置 |
| 团队 | `Team.hideBranding` | 团队级别设置 |
| 组织 | `Team.hideBranding`（parent 团队） | 组织级别设置 |

---

## 4. 链路 2: hideBranding → 预订页品牌展示（已实现）

### 4.1 入口页面
- **URL**: `/[user]/[type]`（如 `/john/30min-meeting`）
- **类型**: Pages Router（getServerSideProps）
- **文件**: `apps/web/server/lib/[user]/[type]/getServerSideProps.tsx`

### 4.2 完整判定链

```
用户请求: GET /john/30min-meeting
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
│ 3. shouldHideBrandingForUserEvent                                │
│    位置: packages/features/profile/lib/hideBranding.ts:172-184  │
│                                                                  │
│    export function shouldHideBrandingForUserEvent({              │
│      eventTypeId,                                                │
│      owner,                                                      │
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
│ 4. shouldHideBrandingForEventUsingProfile（核心判定）             │
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

### 4.3 预订页 hideBranding 判定优先级

```
优先级（从高到低）:

组织 (Organization) hideBranding = true?
    │
    ├─ 是 → isBrandingHidden = true (隐藏品牌)
    │       (组织级别设置优先级最高，覆盖实体设置)
    │
    └─ 否 → 实体 (User/Team) hideBranding = true?
             │
             ├─ 是 → isBrandingHidden = true (隐藏品牌)
             │
             └─ 否 / null → isBrandingHidden = false (显示品牌)
```

### 4.4 关键发现：预订页不检查 hasPaidPlan

**重要**：预订页的 `shouldHideBrandingForEvent` **不检查 `hasPaidPlan`**

这意味着：
- **设置页面**：`hasPaidPlan` 控制"是否允许启用"开关
- **预订页面**：只看数据库中 `hideBranding` 字段的值，不检查是否付费

**潜在问题**：如果直接修改数据库绕过设置页面的限制，可能可以启用这个功能而不付费。

---

## 5. 链路 3: session.hasValidLicense（路径 A：next-auth callback）

### 5.1 触发方式
- **调用点**: 前端使用 `useSession()` hook（来自 `next-auth/react`）
- **调用时机**: NextAuth 自动处理 session 刷新时

### 5.2 完整链路

```
前端调用: useSession() 或 NextAuth 自动刷新
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. NextAuth JWT → Session 转换                                   │
│    通过 next-auth-options 中的 callbacks 配置                     │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. session callback 执行                                         │
│    位置: packages/features/auth/lib/next-auth-options.ts:763-787 │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 硬编码 hasValidLicense = false                                │
│    位置: 第 765 行                                               │
│                                                                  │
│    async session({ session, token, user }) {                     │
│      log.debug("callbacks:session - Session callback called", ...);│
│      const hasValidLicense = false;  // ⚠️ 硬编码为 false        │
│      const profileId = token.profileId;                          │
│      const calendsoSession: Session = {                         │
│        ...session,                                               │
│        profileId,                                                │
│        upId: token.upId || session.upId,                         │
│        hasValidLicense,  // ← 永远为 false                       │
│        user: {                                                   │
│          ...session.user,                                        │
│          // ... 其他字段                                         │
│        },                                                        │
│      };                                                          │
│      return calendsoSession;                                     │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 前端获取 session                                              │
│    const { data: session } = useSession();                      │
│    // session.hasValidLicense = false (永远)                     │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 实际调用点

| 场景 | 代码位置 |
|------|---------|
| 前端组件 | `apps/web/modules/bookings/components/BookerWebWrapper.tsx` |
| 前端组件 | `apps/web/modules/shell/Shell.tsx` |
| 前端组件 | `apps/web/modules/shell/SideBar.tsx` |

### 5.4 实际影响

**路径 A 的 session.hasValidLicense**：
- 值: 永远为 `false`（硬编码）
- 传递给: Booker 组件
- 实际使用: **Booker 组件从未使用这个值做任何条件判断**

---

## 6. 链路 4: session.hasValidLicense（路径 B：自定义 getServerSession）

### 6.1 触发方式
- **调用点**: 服务端手动调用 `getServerSession({ req, authOptions })`
- **调用时机**: Server Component、getServerSideProps、API Route 中手动获取 session

### 6.2 完整链路

```
服务端代码调用: getServerSession({ req, authOptions })
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 自定义 getServerSession 执行                                  │
│    位置: packages/features/auth/lib/getServerSession.ts:39-151  │
│                                                                  │
│    // 这是自定义的 slim 版本，不依赖完整 NextAuth options        │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. LicenseKeySingleton 定义（占位实现）                          │
│    位置: packages/features/auth/lib/getServerSession.ts:11-19   │
│                                                                  │
│    class LicenseKeySingleton {                                   │
│      static async getInstance(..._args: unknown[]) {             │
│        return new LicenseKeySingleton();  // 忽略所有参数        │
│      }                                                           │
│      async checkLicense() {                                      │
│        return true;  // ⚠️ 直接返回 true，无任何校验             │
│      }                                                           │
│      async validateLicenseKey() {                                │
│        return true;  // ⚠️ 直接返回 true，无任何校验             │
│      }                                                           │
│    }                                                             │
│                                                                  │
│    class DeploymentRepository {                                  │
│      constructor(_prisma?: unknown) {}                           │
│      async findFirst(..._args: unknown[]) { return null; }       │
│      // ⚠️ 从不读取数据库 Deployment.licenseKey                  │
│    }                                                             │
└─────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 调用 LicenseKeySingleton.checkLicense()                       │
│    位置: packages/features/auth/lib/getServerSession.ts:80-83   │
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
│ 4. 注入到返回的 session 对象                                     │
│    位置: 第 101-125 行                                           │
│                                                                  │
│    const session: Session = {                                    │
│      hasValidLicense,  // = true (永远)                          │
│      expires: ...,                                               │
│      user: {                                                     │
│        // ... 用户数据                                           │
│      },                                                          │
│      // ... 其他字段                                             │
│    };                                                            │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 实际调用点

| 场景 | 代码位置 |
|------|---------|
| App Router Server Component | `apps/web/app/(use-page-wrapper)/settings/.../appearance/page.tsx` |
| Pages Router getServerSideProps | `apps/web/server/lib/[user]/[type]/getServerSideProps.ts` |
| API Route | `apps/web/app/api/me/route.ts` |
| tRPC Context | `apps/web/app/_trpc/context.ts` |

### 6.4 实际影响

**路径 B 的 session.hasValidLicense**：
- 值: 永远为 `true`（LicenseKeySingleton.checkLicense() 直接返回 true）
- 传递给: 服务端代码逻辑
- 实际使用: **没有任何地方使用这个值做条件判断**

---

## 7. 四条链路的交叉对照

### 7.1 session.hasValidLicense 两条路径对比

| 维度 | 路径 A: next-auth callback | 路径 B: 自定义 getServerSession |
|------|---------------------------|--------------------------------|
| **触发方式** | 前端 `useSession()` | 服务端 `getServerSession()` |
| **关键代码位置** | `next-auth-options.ts:765` | `getServerSession.ts:82` |
| **实现方式** | 硬编码 `const hasValidLicense = false;` | `LicenseKeySingleton.checkLicense()` → `return true` |
| **最终值** | `false` | `true` |
| **是否读取数据库** | 否 | 否（DeploymentRepository.findFirst 返回 null） |
| **是否验证签名** | 否 | 否 |
| **是否被实际使用** | 否（Booker 接收但不使用） | 否 |

### 7.2 为什么两条路径值不一致但都不影响

```
矛盾现象:
┌─────────────────────────────────────────────────────────────────┐
│ 路径 A (next-auth): hasValidLicense = false                      │
│ 路径 B (getServerSession): hasValidLicense = true                │
│                                                                  │
│ 但两条路径都不影响任何功能!                                       │
└─────────────────────────────────────────────────────────────────┘

原因:
1. 路径 A: 前端 useSession() 返回的值，Booker 组件接收但从未使用
2. 路径 B: 服务端 getServerSession() 返回的值，没有任何地方使用

即使值不一致 → 都不会影响功能可见性
```

### 7.3 与 hasPaidPlan / hideBranding 链路的关系

| 机制 | 控制对象 | 实际效果 |
|------|---------|---------|
| **hasPaidPlan** | 设置页面开关可用性 | ✅ 生效 |
| **hideBranding 数据库字段** | 预订页品牌展示 | ✅ 生效 |
| **session.hasValidLicense (路径 A)** | 无 | ❌ 不生效 |
| **session.hasValidLicense (路径 B)** | 无 | ❌ 不生效 |

**关键关系**:
- `hasPaidPlan` 控制"是否允许设置" `hideBranding` 开关
- `hideBranding` 字段值控制"是否显示"品牌标识
- `session.hasValidLicense` 完全独立，不影响上述两者

---

## 8. 关键代码索引

### 8.1 已实现链路（真正生效）

| 文件路径 | 行号 | 说明 | 关键代码 |
|---------|------|------|---------|
| `apps/web/app/(use-page-wrapper)/settings/(settings-layout)/my-account/appearance/page.tsx` | 46 | hasPaidPlan 计算 | `const hasPaidPlan = IS_SELF_HOSTED ? true : hasTeamPlan?.hasTeamPlan \|\| isCurrentUsernamePremium` |
| `apps/web/app/cache/membership.ts` | 11-22 | hasTeamPlan 查询 | `MembershipRepository.hasAnyAcceptedMembershipByUserId(userId)` |
| `apps/web/modules/settings/my-account/appearance-view.tsx` | 393 | 开关禁用条件 | `disabled={!hasPaidPlan \|\| mutation?.isPending}` |
| `apps/web/modules/settings/my-account/appearance-view.tsx` | 395 | 开关显示值 | `checked={hasPaidPlan ? hideBrandingValue : false}` |
| `packages/features/profile/lib/hideBranding.ts` | 33-43 | resolveHideBranding | 组织优先级高于实体 |
| `packages/features/profile/lib/hideBranding.ts` | 88-113 | shouldHideBrandingForEventUsingProfile | 预订页判定逻辑 |
| `apps/web/server/lib/[user]/[type]/getServerSideProps.tsx` | 271-274 | 预订页 isBrandingHidden | `shouldHideBrandingForUserEvent({...})` |
| `apps/web/modules/users/views/users-type-public-view.tsx` | 37 | 传递给 Booker | `hideBranding={isBrandingHidden}` |

### 8.2 License Gate 链路（不生效）

| 文件路径 | 行号 | 说明 | 关键代码 |
|---------|------|------|---------|
| `packages/features/auth/lib/next-auth-options.ts` | 763-787 | session callback（路径 A） | `const hasValidLicense = false;`（硬编码） |
| `packages/features/auth/lib/getServerSession.ts` | 11-15 | LicenseKeySingleton（路径 B） | `async checkLicense() { return true; }` |
| `packages/features/auth/lib/getServerSession.ts` | 16-19 | DeploymentRepository（路径 B） | `async findFirst(..._args: unknown[]) { return null; }` |
| `packages/features/auth/lib/getServerSession.ts` | 80-83 | 调用 checkLicense（路径 B） | `const hasValidLicense = await licenseKeyService.checkLicense();` |
| `apps/web/modules/bookings/components/BookerWebWrapper.tsx` | 223 | 传递 hasValidLicense | `hasValidLicense={session?.hasValidLicense ?? false}` |

### 8.3 根本原因证据

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

## 9. 总结

### 9.1 实际生效的企业功能 Gate

Cal.diy 有**一条完整的企业功能控制链路**：

```
hasPaidPlan 判定链（已实现）
├─ 自托管: 永远为 true
└─ 非自托管:
   ├─ hasTeamPlan: 用户是否属于任意团队
   └─ isCurrentUsernamePremium: 用户 metadata.isPremium

结果:
├─ hasPaidPlan = true → hideBranding 开关可用 → 用户可设置
│                                                    ↓
│                                              User/Team/Organization.hideBranding
│                                                    ↓
└─ 预订页: shouldHideBrandingForEvent() → 决定是否显示品牌标识
```

### 9.2 不生效的 License Gate

所有 `hasValidLicense` 相关的机制**都不生效**：

```
session.hasValidLicense 两条路径（都不生效）

路径 A (next-auth callback):
├─ 硬编码 const hasValidLicense = false
└─ 传递给 Booker → 但 Booker 从未使用

路径 B (自定义 getServerSession):
├─ LicenseKeySingleton.checkLicense() → return true (永远)
└─ 没有任何代码使用这个值做条件判断
```

### 9.3 根本原因

代码注释明确说明 Cal.diy 是开源版本：
> "In the open-source distribution there is no paywall"
> "Cal.diy is fully open source — no license key is required"

因此，所有 License Gate 相关的代码都是从上游 Cal.com 继承的骨架代码，在 Cal.diy 中被有意地简化或绕过。

---

*本报告基于 2026-05-10 的代码库状态分析生成。*
