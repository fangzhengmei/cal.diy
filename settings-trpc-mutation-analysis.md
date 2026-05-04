# Settings 页面 tRPC Mutation 协同工作分析

## 1. 整体架构概览

在 Cal.diy 项目中，Settings 页面的表单提交、数据验证、tRPC 调用和数据库操作形成了一个完整的数据流。这个流程主要包含以下几个层次：

1. **前端表单层**：使用 React Hook Form 管理表单状态
2. **客户端验证层**：使用 Zod 进行前端数据验证
3. **tRPC 客户端层**：通过 tRPC hooks 调用后端 API
4. **tRPC 服务端层**：处理业务逻辑和服务端验证
5. **数据访问层**：使用 Prisma 进行数据库操作
6. **副作用处理层**：缓存失效、状态刷新、通知触发等

## 2. 设置页面分类与 tRPC Procedure 映射

### 2.1 设置页面总览

| 设置类别 | 页面路径 | 主要 tRPC Router | 核心 Mutations |
|---------|---------|------------------|----------------|
| 个人资料 | `/settings/my-account/profile` | `viewer.me` | `updateProfile`, `deleteMe`, `deleteMeWithoutPassword` |
| 通用设置 | `/settings/my-account/general` | `viewer.me` | `updateProfile` (timeZone, locale, weekStart 等) |
| 外观设置 | `/settings/my-account/appearance` | `viewer.me` | `updateProfile` (theme, brandColor 等) |
| 通知设置 | `/settings/my-account/push-notifications` + `/settings/my-account/general` | `viewer.me`, Web Push API | `updateProfile` (receiveMonthlyDigestEmail 等) |
| 可用性设置 | `/availability` | `viewer.availability`, `viewer.availability.schedule` | `create`, `update`, `delete`, `duplicate`, `bulkUpdateToDefaultAvailability` |
| 集成设置 | `/settings/apps` | `viewer.apps` | `toggle`, `saveKeys`, `updateAppCredentials`, `setDefaultConferencingApp`, `updateUserDefaultConferencingApp` |
| 安全设置 | `/settings/security` | `viewer.auth` | `changePassword`, `verifyPassword` |

### 2.2 tRPC Router 层级结构

```
viewer
├── me                    # 用户个人设置
│   ├── updateProfile     # 更新个人资料（包含大多数设置字段）
│   ├── get               # 获取用户信息
│   ├── deleteMe          # 删除账户
│   └── deleteMeWithoutPassword
│
├── availability          # 可用性设置
│   ├── list              # 列出日程
│   ├── user              # 获取用户可用性
│   ├── listTeam          # 团队可用性
│   ├── schedule          # 日程管理（子 router）
│   │   ├── get           # 获取单个日程
│   │   ├── create        # 创建日程
│   │   ├── update        # 更新日程
│   │   ├── delete        # 删除日程
│   │   ├── duplicate     # 复制日程
│   │   ├── getScheduleByUserId
│   │   └── bulkUpdateToDefaultAvailability
│   └── calendarOverlay
│
├── apps                  # 应用/集成设置
│   ├── appById
│   ├── appCredentialsByType
│   ├── getUsersDefaultConferencingApp
│   ├── integrations
│   ├── listLocal
│   ├── locationOptions
│   ├── toggle                    # Mutation: 切换应用
│   ├── saveKeys                  # Mutation: 保存密钥
│   ├── checkForGCal
│   ├── setDefaultConferencingApp # Mutation: 设置默认会议应用
│   ├── updateAppCredentials      # Mutation: 更新应用凭证
│   ├── queryForDependencies
│   ├── checkGlobalKeys
│   └── updateUserDefaultConferencingApp # Mutation: 更新用户默认会议应用
│
└── auth                  # 认证/安全设置
    ├── verifyPassword
    ├── changePassword
    ├── sendVerifyEmailCode
    ├── resendVerifyEmail
    └── createAccountPassword
```

## 3. 前端表单与客户端验证

### 3.1 表单管理

Cal.diy 使用 `react-hook-form` 作为表单状态管理库，结合 `zod` 和 `@hookform/resolvers/zod` 进行数据验证。

**关键依赖导入**：
```typescript
import { Controller, useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
```
[profile-view.tsx:41-42](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L41-L42)

### 3.2 表单值类型定义

每个设置页面通常会定义自己的表单值类型，例如：

**个人资料表单**：
```typescript
export type FormValues = {
  username: string;
  avatarUrl: string | null;
  name: string;
  email: string;
  bio: string;
  secondaryEmails: Email[];
};
```
[profile-view.tsx:56-63](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L56-L63)

**通用设置表单**：
```typescript
export type FormValues = {
  locale: {
    value: string;
    label: string;
  };
  timeZone: string;
  timeFormat: {
    value: number;
    label: string | number;
  };
  weekStart: {
    value: string;
    label: string;
  };
  travelSchedules: {
    id?: number;
    startDate: Date;
    endDate?: Date;
    timeZone: string;
  }[];
};
```
[general-view.tsx:24-44](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/general-view.tsx#L24-L44)

### 3.3 客户端验证 Schema

使用 Zod 定义验证规则，例如个人资料表单的验证：

```typescript
const profileFormSchema = z.object({
  username: z.string(),
  avatarUrl: z.string().nullable(),
  name: z
    .string()
    .trim()
    .min(1, t("you_need_to_add_a_name"))
    .max(FULL_NAME_LENGTH_MAX_LIMIT, {
      message: t("max_limit_allowed_hint", {
        limit: FULL_NAME_LENGTH_MAX_LIMIT,
      }),
    }),
  email: emailSchema.toLowerCase(),
  bio: z.string(),
  secondaryEmails: z.array(
    z.object({
      id: z.number(),
      email: emailSchema.toLowerCase(),
      emailVerified: z.union([z.string(), z.null()]).optional(),
      emailPrimary: z.boolean().optional(),
    })
  ),
});
```
[profile-view.tsx:514-536](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L514-L536)

### 3.4 表单初始化与提交

使用 `useForm` hook 初始化表单，并设置默认值和验证解析器：

```typescript
const formMethods = useForm<FormValues>({
  defaultValues,
  resolver: zodResolver(profileFormSchema),
});
```
[profile-view.tsx:538-541](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L538-L541)

表单提交处理：
```typescript
const handleFormSubmit = (values: FormValues) => {
  onSubmit(getUpdatedFormValues(values));
};
```
[profile-view.tsx:592-594](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L592-L594)

## 4. tRPC 客户端调用

### 4.1 tRPC Mutation 初始化

在前端组件中，使用 tRPC 提供的 hooks 初始化 mutation：

```typescript
const updateProfileMutation = trpc.viewer.me.updateProfile.useMutation({
  onSuccess: async (res) => {
    await update(res);
    utils.viewer.me.invalidate();
    utils.viewer.me.shouldVerifyEmail.invalidate();
    revalidateSettingsProfile();

    if (res.hasEmailBeenChanged && res.sendEmailVerification) {
      showToast(t("change_of_email_toast", { email: tempFormValues?.email }), "success");
    } else {
      showToast(t("settings_updated_successfully"), "success");
    }

    setTempFormValues(null);
  },
  onError: (e) => {
    switch (e.message) {
      case "email_already_used":
        {
          showToast(t(e.message), "error");
        }
        return;
      default:
        showToast(t("error_updating_settings"), "error");
    }
  },
});
```
[profile-view.tsx:73-100](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L73-L100)

### 4.2 触发 Mutation

在表单提交或其他交互中触发 mutation：

```typescript
<ProfileForm
  // ... 其他属性
  onSubmit={(values) => {
    if (values.email !== user.email && isCALIdentityProvider) {
      setTempFormValues(values);
      setConfirmPasswordOpen(true);
    } else {
      updateProfileMutation.mutate(values);
    }
  }}
  // ... 其他属性
/>
```
[profile-view.tsx:271-278](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L271-L278)

### 4.3 通用设置中的 Mutation 调用

在通用设置页面，同样的模式：

```typescript
const mutation = trpc.viewer.me.updateProfile.useMutation({
  onSuccess: async (res) => {
    await utils.viewer.me.invalidate();
    revalidateSettingsGeneral();
    revalidateTravelSchedules();
    reset(getValues());
    showToast(t("settings_updated_successfully"), "success");
    await update(res);

    if (res.locale) {
      window.calNewLocale = res.locale;
      document.cookie = `calNewLocale=${res.locale}; path=/`;
    }
  },
  onError: () => {
    showToast(t("error_updating_settings"), "error");
  },
  onSettled: async () => {
    await utils.viewer.me.invalidate();
    revalidateSettingsGeneral();
    revalidateTravelSchedules();
    setIsUpdateBtnLoading(false);
  },
});
```
[general-view.tsx:62-85](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/general-view.tsx#L62-L85)

## 5. 通知设置完整链路

### 5.1 通知设置分类

通知设置包含两部分：
1. **推送通知设置** (`/settings/my-account/push-notifications`) - 浏览器推送通知
2. **邮件通知设置** (`/settings/my-account/general`) - 月度摘要邮件等

### 5.2 推送通知设置链路

#### 前端交互

**页面组件**：
```typescript
const PushNotificationsView = () => {
  const { t } = useLocale();
  const { subscribe, unsubscribe, isSubscribed, isLoading } = useWebPush();

  return (
    <SettingsHeader
      title={t("push_notifications")}
      description={t("push_notifications_description")}
      borderInShellHeader={true}>
      <div className="border-subtle rounded-b-xl border-x border-b px-4 pb-10 pt-8 sm:px-6">
        <Button color="primary" onClick={isSubscribed ? unsubscribe : subscribe} disabled={isLoading}>
          {isSubscribed ? t("disable_browser_notifications") : t("allow_browser_notifications")}
        </Button>
      </div>
    </SettingsHeader>
  );
};
```
[push-notifications-view.tsx:8-24](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/push-notifications-view.tsx#L8-L24)

#### 核心 Hook：useWebPush

```typescript
// apps/web/modules/notifications/hooks/useWebPush.ts
// 主要功能：
// 1. subscribe() - 订阅浏览器推送通知
// 2. unsubscribe() - 取消订阅
// 3. isSubscribed - 订阅状态
// 4. isLoading - 加载状态

// 流程：
// 1. 调用 Notification.requestPermission() 请求权限
// 2. 使用 serviceWorkerRegistration.pushManager.subscribe() 创建订阅
// 3. 将订阅信息发送到后端保存
// 4. 取消订阅时调用 subscription.unsubscribe() 并通知后端
```

#### 数据流

```
用户点击"允许浏览器通知"
    ↓
1. 浏览器权限请求：Notification.requestPermission()
    ↓
2. 服务工作者订阅：serviceWorkerRegistration.pushManager.subscribe()
    ├── endpoint: 推送服务端点 URL
    ├── keys: 加密密钥（p256dh, auth）
    ↓
3. 发送订阅信息到后端（通常通过 API 调用）
    ↓
4. 后端保存订阅信息到数据库（PushSubscription 表）
    ↓
5. 更新 isSubscribed 状态为 true
    ↓
显示成功提示
```

#### 副作用处理

- **订阅成功**：
  - 保存 PushSubscription 到数据库
  - 更新用户元数据（可选）
  - 显示成功 toast

- **取消订阅**：
  - 调用 subscription.unsubscribe()
  - 从数据库删除 PushSubscription 记录
  - 更新 isSubscribed 状态

### 5.3 邮件通知设置链路

#### 前端交互

邮件通知设置位于通用设置页面，使用 `SettingsToggle` 组件：

```typescript
<SettingsToggle
  toggleSwitchAtTheEnd={true}
  title={t("monthly_digest_email")}
  description={t("monthly_digest_email")}
  disabled={mutation.isPending}
  checked={isReceiveMonthlyDigestEmailChecked}
  onCheckedChange={(checked) => {
    setIsReceiveMonthlyDigestEmailChecked(checked);
    mutation.mutate({ receiveMonthlyDigestEmail: checked });
  }}
  switchContainerClassName="mt-6"
/>
```
[general-view.tsx:353-364](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/general-view.tsx#L353-L364)

#### tRPC Mutation

邮件通知设置使用 `updateProfile` mutation，通过 `receiveMonthlyDigestEmail` 字段：

```typescript
// 前端调用
mutation.mutate({ receiveMonthlyDigestEmail: checked });

// 服务端 Schema 定义
export const ZUpdateProfileInputSchema = z.object({
  // ... 其他字段
  receiveMonthlyDigestEmail: z.boolean().optional(),
  // ...
});
```
[updateProfile.schema.ts:99](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/me/updateProfile.schema.ts#L99)

#### 数据库操作

`receiveMonthlyDigestEmail` 字段存储在 `User` 表中：

```prisma
// schema.prisma 中的 User 模型
model User {
  // ... 其他字段
  receiveMonthlyDigestEmail      Boolean                  @default(false)
  // ...
}
```

#### 完整数据流

```
用户切换月度摘要邮件开关
    ↓
1. 更新本地状态：setIsReceiveMonthlyDigestEmailChecked(checked)
    ↓
2. 触发 tRPC Mutation：
   mutation.mutate({ receiveMonthlyDigestEmail: checked })
    ↓
3. tRPC 客户端发送请求
    ↓
4. 服务端验证：ZUpdateProfileInputSchema
    ↓
5. Handler 处理：updateProfileHandler
    ↓
6. 数据库更新：
   prisma.user.update({
     where: { id: user.id },
     data: { receiveMonthlyDigestEmail: checked }
   })
    ↓
7. 返回结果
    ↓
8. 前端副作用：
   - utils.viewer.me.invalidate()
   - revalidateSettingsGeneral()
   - showToast("settings_updated_successfully")
```

#### 相关邮件通知设置

通用设置页面还有其他邮件相关的 toggle：

```typescript
// 预订者邮箱验证要求
<SettingsToggle
  toggleSwitchAtTheEnd={true}
  title={t("require_booker_email_verification")}
  description={t("require_booker_email_verification_description")}
  disabled={mutation.isPending}
  checked={isRequireBookerEmailVerificationChecked}
  onCheckedChange={(checked) => {
    setIsRequireBookerEmailVerificationChecked(checked);
    mutation.mutate({ requiresBookerEmailVerification: checked });
  }}
  switchContainerClassName="mt-6"
/>
```
[general-view.tsx:366-377](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/general-view.tsx#L366-L377)

### 5.4 通知设置数据库模型

```prisma
// PushSubscription 表 - 存储浏览器推送订阅
model PushSubscription {
  id             String   @id @default(cuid())
  userId         Int
  user           User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  endpoint       String   @unique
  p256dh         String
  auth           String
  created        DateTime @default(now())
  updated        DateTime @updatedAt

  @@unique([userId, endpoint])
}

// User 表中的通知相关字段
model User {
  // ...
  receiveMonthlyDigestEmail      Boolean                  @default(false)
  requiresBookerEmailVerification Boolean                @default(false)
  // ...
}
```

## 6. 集成设置完整链路

### 6.1 集成设置概述

集成设置（Apps/Integrations）管理用户与第三方应用的连接，包括：
- 日历集成（Google Calendar, Outlook 等）
- 视频会议集成（Zoom, Google Meet, Microsoft Teams 等）
- 支付集成（Stripe, PayPal 等）
- 其他应用集成

### 6.2 tRPC Apps Router

```typescript
export const appsRouter = router({
  // Queries
  appById: authedProcedure.input(ZAppByIdInputSchema).query(...),
  appCredentialsByType: authedProcedure.input(ZAppCredentialsByTypeInputSchema).query(...),
  getUsersDefaultConferencingApp: authedProcedure.query(...),
  integrations: authedProcedure.input(ZIntegrationsInputSchema).query(...),
  listLocal: authedAdminProcedure.input(ZListLocalInputSchema).query(...),
  locationOptions: authedProcedure.input(ZLocationOptionsInputSchema).query(...),
  checkForGCal: authedProcedure.query(...),
  queryForDependencies: authedProcedure.input(ZQueryForDependenciesInputSchema).query(...),
  checkGlobalKeys: authedProcedure.input(checkGlobalKeysSchema).query(...),

  // Mutations
  toggle: authedAdminProcedure.input(ZToggleInputSchema).mutation(...),
  saveKeys: authedAdminProcedure.input(ZSaveKeysInputSchema).mutation(...),
  setDefaultConferencingApp: authedProcedure.input(ZSetDefaultConferencingAppSchema).mutation(...),
  updateAppCredentials: authedProcedure.input(ZUpdateAppCredentialsInputSchema).mutation(...),
  updateUserDefaultConferencingApp: authedProcedure.input(ZUpdateUserDefaultConferencingAppInputSchema).mutation(...),
});
```
[_router.tsx:33-135](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/apps/_router.tsx#L33-L135)

### 6.3 设置默认会议应用链路

这是最常见的集成设置操作之一。

#### 前端交互

```typescript
// 示例：设置默认会议应用
const setDefaultMutation = trpc.viewer.apps.setDefaultConferencingApp.useMutation({
  onSuccess: () => {
    utils.viewer.apps.getUsersDefaultConferencingApp.invalidate();
    showToast(t("settings_updated_successfully"), "success");
  },
  onError: (error) => {
    showToast(error.message, "error");
  },
});

// 调用
setDefaultMutation.mutate({ appId: "zoom" });
```

#### tRPC Mutation 定义

**Schema**：
```typescript
// packages/trpc/server/routers/viewer/apps/setDefaultConferencingApp.schema.ts
export const ZSetDefaultConferencingAppSchema = z.object({
  appId: z.string(),
});

export type TSetDefaultConferencingAppSchema = z.infer<typeof ZSetDefaultConferencingAppSchema>;
```

**Handler**：
```typescript
// packages/trpc/server/routers/viewer/apps/setDefaultConferencingApp.handler.ts
export const setDefaultConferencingAppHandler = async ({
  ctx,
  input,
}: {
  ctx: { user: { id: number } };
  input: TSetDefaultConferencingAppSchema;
}) => {
  const { user } = ctx;
  const { appId } = input;

  // 检查应用是否存在
  const app = await prisma.app.findUnique({
    where: { slug: appId },
  });

  if (!app) {
    throw new TRPCError({ code: "NOT_FOUND", message: "App not found" });
  }

  // 更新用户的默认会议应用
  await prisma.user.update({
    where: { id: user.id },
    data: {
      defaultConferencingApp: appId,
    },
  });

  return { success: true };
};
```

### 6.4 更新应用凭证链路

#### 前端交互

```typescript
const updateCredentialsMutation = trpc.viewer.apps.updateAppCredentials.useMutation({
  onSuccess: () => {
    utils.viewer.apps.appCredentialsByType.invalidate();
    showToast(t("credentials_updated_successfully"), "success");
  },
  onError: (error) => {
    showToast(error.message, "error");
  },
});

// 调用
updateCredentialsMutation.mutate({
  appId: "google-calendar",
  credentials: {
    // 应用特定的凭证格式
  },
});
```

#### 完整数据流

```
用户在集成设置页面更新应用凭证
    ↓
1. 收集用户输入的凭证信息（API Key, OAuth Token 等）
    ↓
2. 触发 tRPC Mutation：
   trpc.viewer.apps.updateAppCredentials.useMutation()
    ↓
3. 服务端验证 Schema
    ↓
4. Handler 处理：
   a. 验证应用是否存在
   b. 验证凭证格式（应用特定逻辑）
   c. 加密敏感凭证（如需要）
    ↓
5. 数据库操作：
   a. 查找或创建 Credential 记录
   b. 更新凭证数据
   c. 关联到用户
    ↓
6. 返回结果
    ↓
7. 前端副作用：
   - 失效相关查询缓存
   - 显示成功/失败提示
   - 刷新集成列表
```

### 6.5 切换应用启用状态链路

#### tRPC Mutation

```typescript
// Router 定义
toggle: authedAdminProcedure.input(ZToggleInputSchema).mutation(async ({ ctx, input }) => {
  const { toggleHandler } = await import("./toggle.handler");
  return toggleHandler({ ctx, input });
}),
```
[_router.tsx:69-75](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/apps/_router.tsx#L69-L75)

#### Handler 实现

```typescript
// packages/trpc/server/routers/viewer/apps/toggle.handler.ts
export const toggleHandler = async ({ ctx, input }: ToggleOptions) => {
  const { user } = ctx;
  const { slug, enabled } = input;

  // 检查应用是否存在
  const app = await prisma.app.findUnique({
    where: { slug },
  });

  if (!app) {
    throw new TRPCError({ code: "NOT_FOUND", message: "App not found" });
  }

  // 检查用户是否有管理员权限（因为是 authedAdminProcedure）
  // ...

  // 更新应用启用状态
  const updatedApp = await prisma.app.update({
    where: { slug },
    data: { enabled },
  });

  // 触发缓存失效等副作用
  // ...

  return updatedApp;
};
```

### 6.6 集成设置数据库模型

```prisma
// App 表 - 存储应用元数据
model App {
  id              String   @id @default(cuid())
  slug            String   @unique
  name            String
  categories      Json?
  dependencies    Json?
  description     String?
  // ... 其他字段

  enabled         Boolean  @default(false)
  // ...
}

// Credential 表 - 存储用户应用凭证
model Credential {
  id             Int       @id @default(autoincrement())
  type           String
  userId         Int?
  teamId         Int?
  key            Json
  appId          String?
  user           User?     @relation(fields: [userId], references: [id], onDelete: Cascade)
  team           Team?     @relation(fields: [teamId], references: [id], onDelete: Cascade)
  app            App?      @relation(fields: [appId], references: [slug], onDelete: Cascade)
  // ...
}

// User 表中的集成相关字段
model User {
  // ...
  defaultConferencingApp      String?
  // ...
}
```

## 7. 可用性设置完整链路（修正版）

### 7.1 可用性设置架构

可用性设置管理用户的日程安排，包括：
- **日程列表**：创建、编辑、删除、复制日程
- **日程详情**：配置可用性时间段、日期覆盖、时区等
- **默认日程**：设置默认日程，批量应用到事件类型

### 7.2 tRPC Availability Router 结构（修正）

```typescript
// packages/trpc/server/routers/viewer/availability/_router.tsx
export const availabilityRouter = router({
  // Queries
  list: authedProcedure.query(async ({ ctx }) => {
    const { listHandler } = await import("./list.handler");
    return listHandler({ ctx });
  }),

  user: authedProcedure.input(ZUserInputSchema).query(async ({ ctx, input }) => {
    const { userHandler } = await import("./user.handler");
    return userHandler({ ctx, input });
  }),

  listTeam: authedProcedure.input(ZListTeamAvailaiblityScheme).query(...),
  calendarOverlay: authedProcedure.input(ZCalendarOverlayInputSchema).query(...),

  // 子 Router：日程管理
  schedule: scheduleRouter,
});
```
[_router.tsx:15-49](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/availability/_router.tsx#L15-L49)

```typescript
// packages/trpc/server/routers/viewer/availability/schedule/_router.tsx
export const scheduleRouter = router({
  // Queries
  get: authedProcedure.input(ZGetInputSchema).query(...),
  getScheduleByUserId: authedProcedure.input(ZGetByUserIdInputSchema).query(...),

  // Mutations
  create: authedProcedure.input(ZCreateInputSchema).mutation(...),
  update: authedProcedure.input(ZUpdateInputSchema).mutation(...),
  delete: authedProcedure.input(ZDeleteInputSchema).mutation(...),
  duplicate: authedProcedure.input(ZScheduleDuplicateSchema).mutation(...),
  bulkUpdateToDefaultAvailability: authedProcedure.input(
    ZBulkUpdateToDefaultAvailabilityInputSchema
  ).mutation(...),
});
```
[_router.tsx:25-85](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/availability/schedule/_router.tsx#L25-L85)

### 7.3 前端组件结构

#### 可用性列表页面

```typescript
// apps/web/modules/availability/availability-view.tsx
export function AvailabilityList({ availabilities }: AvailabilityListProps) {
  const utils = trpc.useUtils();
  const router = useRouter();

  // Mutations
  const deleteMutation = trpc.viewer.availability.schedule.delete.useMutation({
    onMutate: async ({ scheduleId }) => {
      // 乐观更新
      await utils.viewer.availability.list.cancel();
      const previousValue = utils.viewer.availability.list.getData();
      if (previousValue) {
        const filteredValue = previousValue.schedules.filter(({ id }) => id !== scheduleId);
        utils.viewer.availability.list.setData(undefined, { ...previousValue, schedules: filteredValue });
      }
      return { previousValue };
    },
    onError: (err, variables, context) => {
      // 回滚
      if (context?.previousValue) {
        utils.viewer.availability.list.setData(undefined, context.previousValue);
      }
      // ... 错误处理
    },
    onSettled: () => {
      utils.viewer.availability.list.invalidate();
    },
    onSuccess: () => {
      revalidateAvailabilityList();
      showToast(t("schedule_deleted_successfully"), "success");
    },
  });

  const updateMutation = trpc.viewer.availability.schedule.update.useMutation({
    onSuccess: async ({ schedule }) => {
      await utils.viewer.availability.list.invalidate();
      revalidateAvailabilityList();
      showToast(
        t("availability_updated_successfully", { scheduleName: schedule.name }),
        "success"
      );
      // 打开批量更新对话框
      setBulkUpdateModal(true);
    },
    // ...
  });

  const duplicateMutation = trpc.viewer.availability.schedule.duplicate.useMutation({
    onSuccess: async ({ schedule }) => {
      await router.push(`/availability/${schedule.id}`);
      showToast(t("schedule_created_successfully", { scheduleName: schedule.name }), "success");
    },
    // ...
  });

  // ... 渲染日程列表
  return (
    <>
      <ul className="divide-subtle divide-y" data-testid="schedules" ref={animationParentRef}>
        {availabilities.schedules.map((schedule) => (
          <ScheduleListItem
            key={schedule.id}
            schedule={schedule}
            updateDefault={updateMutation.mutate}
            deleteFunction={deleteMutation.mutate}
            duplicateFunction={duplicateMutation.mutate}
            // ...
          />
        ))}
      </ul>
    </>
  );
}
```
[availability-view.tsx:23-187](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/availability/availability-view.tsx#L23-L187)

#### 日程编辑页面

```typescript
// apps/web/modules/availability/[schedule]/schedule-view.tsx
export const AvailabilitySettingsWebWrapper = ({
  scheduleData: schedule,
  travelSchedulesData: travelSchedules,
}: PageProps) => {
  const utils = trpc.useUtils();
  const scheduleId = schedule.id;

  // 更新日程 Mutation
  const updateMutation = trpc.viewer.availability.schedule.update.useMutation({
    onSuccess: async ({ prevDefaultId, currentDefaultId, ...data }) => {
      // 处理默认日程变更
      if (prevDefaultId && currentDefaultId) {
        if (prevDefaultId !== currentDefaultId) {
          utils.viewer.availability.schedule.get.invalidate({ scheduleId: prevDefaultId });
          utils.viewer.availability.schedule.get.refetch({ scheduleId: prevDefaultId });
        }
      }
      // 失效缓存
      utils.viewer.availability.schedule.get.invalidate({ scheduleId: data.schedule.id });
      revalidateSchedulePage(scheduleId);
      utils.viewer.availability.list.invalidate();
      revalidateAvailabilityList();
      // 显示成功提示
      showToast(
        t("availability_updated_successfully", { scheduleName: data.schedule.name }),
        "success"
      );
    },
    onError: (err) => {
      // 错误处理
      if (err instanceof HttpError) {
        const message = `${err.statusCode}: ${err.message}`;
        showToast(message, "error");
      }
    },
  });

  // 删除日程 Mutation
  const deleteMutation = trpc.viewer.availability.schedule.delete.useMutation({
    onError: withErrorFromUnknown((err) => {
      showToast(err.message, "error");
    }),
    onSettled: () => {
      utils.viewer.availability.list.invalidate();
    },
    onSuccess: () => {
      showToast(t("schedule_deleted_successfully"), "success");
      revalidateAvailabilityList();
      router.push("/availability");
    },
  });

  return (
    <AvailabilitySettings
      schedule={schedule}
      travelSchedules={isDefaultSchedule ? travelSchedules || [] : []}
      isDeleting={deleteMutation.isPending}
      isSaving={updateMutation.isPending}
      handleDelete={() => {
        scheduleId && deleteMutation.mutate({ scheduleId });
      }}
      handleSubmit={async ({ dateOverrides, ...values }) => {
        if (!values.name.trim()) {
          showToast(t("schedule_name_cannot_be_empty"), "error");
          return;
        }
        scheduleId &&
          updateMutation.mutate({
            scheduleId,
            dateOverrides: dateOverrides.flatMap((override) => override.ranges),
            ...values,
          });
      }}
      // ...
    />
  );
};
```
[schedule-view.tsx:23-145](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/availability/[schedule]/schedule-view.tsx#L23-L145)

### 7.4 tRPC Mutation 完整链路

#### 创建日程

**前端调用**：
```typescript
// 通常在 NewScheduleButton 或类似组件中
const createMutation = trpc.viewer.availability.schedule.create.useMutation({
  onSuccess: ({ schedule }) => {
    router.push(`/availability/${schedule.id}`);
    showToast(t("schedule_created_successfully"), "success");
  },
  onError: (err) => {
    showToast(err.message, "error");
  },
});

createMutation.mutate({
  name: "我的新日程",
  schedule: DEFAULT_SCHEDULE, // 包含可用性时间段
});
```

**Schema 定义**：
```typescript
// packages/trpc/server/routers/viewer/availability/schedule/create.schema.ts
export type TCreateInputSchema = {
  name: string;
  schedule?: { start: Date; end: Date }[][];
  eventTypeId?: number;
};

export const ZCreateInputSchema: z.ZodType<TCreateInputSchema> = z.object({
  name: z.string().trim().min(1, "Schedule name cannot be empty"),
  schedule: z
    .array(
      z.array(
        z.object({
          start: z.date(),
          end: z.date(),
        })
      )
    )
    .optional(),
  eventTypeId: z.number().optional(),
});
```
[create.schema.ts:1-22](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/availability/schedule/create.schema.ts#L1-L22)

**Handler 实现**：
```typescript
// packages/trpc/server/routers/viewer/availability/schedule/create.handler.ts
export const createHandler = async ({ input, ctx }: CreateOptions) => {
  const { user } = ctx;

  // 验证事件类型权限（如果指定）
  if (input.eventTypeId) {
    const eventType = await prisma.eventType.findUnique({
      where: { id: input.eventTypeId },
      select: { userId: true },
    });
    if (!eventType || eventType.userId !== user.id) {
      throw new TRPCError({
        code: "UNAUTHORIZED",
        message: "You are not authorized to create a schedule for this event type",
      });
    }
  }

  // 准备创建数据
  const data: Prisma.ScheduleCreateInput = {
    name: input.name,
    user: { connect: { id: user.id } },
    // 如果指定了 eventTypeId，关联到事件类型
    ...(input.eventTypeId && { eventType: { connect: { id: input.eventTypeId } } }),
  };

  // 处理可用性时间段
  const availability = getAvailabilityFromSchedule(input.schedule || DEFAULT_SCHEDULE);
  data.availability = {
    createMany: {
      data: availability.map((schedule) => ({
        days: schedule.days,
        startTime: schedule.startTime,
        endTime: schedule.endTime,
      })),
    },
  };

  // 设置时区
  data.timeZone = user.timeZone;

  // 创建日程
  const schedule = await prisma.schedule.create({ data });

  // 如果是第一个日程，设置为默认日程
  if (!user.defaultScheduleId) {
    await prisma.user.update({
      where: { id: user.id },
      data: { defaultScheduleId: schedule.id },
    });
  }

  return { schedule };
};
```
[create.handler.ts:19-77](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/availability/schedule/create.handler.ts#L19-L77)

#### 更新日程

**Schema 定义**：
```typescript
// packages/trpc/server/routers/viewer/availability/schedule/update.schema.ts
export {
  type TUpdateInputSchema,
  ZUpdateInputSchema,
} from "@calcom/features/schedules/services/ScheduleService";
```
[update.schema.ts:1-4](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/availability/schedule/update.schema.ts#L1-L4)

**Handler 实现**：
```typescript
// packages/trpc/server/routers/viewer/availability/schedule/update.handler.ts
export const updateHandler = async ({ input, ctx }: UpdateOptions) => {
  const { user } = ctx;
  return updateSchedule({
    input,
    user,
    prisma,
  });
};
```
[update.handler.ts:15-22](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/availability/schedule/update.handler.ts#L15-L22)

**Service 层实现**：
```typescript
// packages/features/schedules/services/ScheduleService.ts
export const updateSchedule = async ({
  input,
  user,
  prisma,
}: {
  input: TUpdateInputSchema;
  user: { id: number; defaultScheduleId: number | null; timeZone: string | null };
  prisma: PrismaClient;
}) => {
  const { scheduleId, name, isDefault, timezone, schedule, dateOverrides } = input;

  // 获取现有日程
  const currentSchedule = await prisma.schedule.findUniqueOrThrow({
    where: { id: scheduleId },
    select: {
      userId: true,
      isDefault: true,
      availability: true,
      dateOverrides: true,
      timeZone: true,
      name: true,
    },
  });

  // 验证权限
  if (currentSchedule.userId !== user.id) {
    throw new HttpError({ statusCode: 403, message: "You are not authorized to update this schedule" });
  }

  // 准备更新数据
  const data: Prisma.ScheduleUpdateInput = {
    name,
    // ... 其他字段
  };

  // 处理默认日程变更
  let prevDefaultId: number | null = null;
  let currentDefaultId: number | null = null;

  if (isDefault !== undefined && isDefault !== currentSchedule.isDefault) {
    if (isDefault) {
      // 设置为默认日程
      prevDefaultId = user.defaultScheduleId;
      currentDefaultId = scheduleId;

      // 先取消之前的默认日程
      if (user.defaultScheduleId) {
        await prisma.schedule.update({
          where: { id: user.defaultScheduleId },
          data: { isDefault: false },
        });
      }

      data.isDefault = true;
    } else {
      // 取消默认日程
      throw new HttpError({
        statusCode: 400,
        message: "You cannot unset the default schedule",
      });
    }
  }

  // 处理时区更新
  if (timezone && timezone !== currentSchedule.timeZone) {
    data.timeZone = timezone;
  }

  // 处理可用性时间段更新
  if (schedule) {
    // 删除现有可用性
    await prisma.schedule.update({
      where: { id: scheduleId },
      data: { availability: { deleteMany: {} } },
    });

    // 创建新可用性
    const newAvailability = getAvailabilityFromSchedule(schedule);
    data.availability = {
      createMany: {
        data: newAvailability.map((a) => ({
          days: a.days,
          startTime: a.startTime,
          endTime: a.endTime,
        })),
      },
    };
  }

  // 处理日期覆盖
  if (dateOverrides) {
    // 删除现有覆盖
    await prisma.schedule.update({
      where: { id: scheduleId },
      data: { dateOverrides: { deleteMany: {} } },
    });

    // 创建新覆盖
    if (dateOverrides.length > 0) {
      data.dateOverrides = {
        createMany: {
          data: dateOverrides.map((override) => ({
            start: override.start,
            end: override.end,
          })),
        },
      };
    }
  }

  // 执行更新
  const updatedSchedule = await prisma.schedule.update({
    where: { id: scheduleId },
    data,
    include: {
      availability: true,
      dateOverrides: true,
    },
  });

  return {
    schedule: updatedSchedule,
    prevDefaultId,
    currentDefaultId,
  };
};
```

### 7.5 完整数据流示例

#### 创建日程流程

```
用户点击"新建日程"按钮
    ↓
1. 打开新建日程对话框/页面
    ↓
2. 用户输入日程名称，选择可用性模板
    ↓
3. 前端验证：
   - 名称不为空
    ↓
4. 触发 tRPC Mutation：
   trpc.viewer.availability.schedule.create.useMutation()
    ↓
5. 服务端 Schema 验证：
   ZCreateInputSchema
    ↓
6. Handler 处理：
   a. 验证事件类型权限（如果指定）
   b. 准备创建数据
   c. 处理可用性时间段
   d. 设置时区
    ↓
7. 数据库操作：
   a. prisma.schedule.create()
   b. 如果是第一个日程，设置为默认日程
    ↓
8. 返回结果：{ schedule }
    ↓
9. 前端副作用：
   - router.push(`/availability/${schedule.id}`)
   - showToast("schedule_created_successfully")
```

#### 更新日程流程

```
用户在日程编辑页面修改设置
    ↓
1. 用户修改：
   - 名称
   - 可用性时间段
   - 日期覆盖
   - 是否设为默认
   - 时区
    ↓
2. 点击"保存"按钮
    ↓
3. 前端验证：
   - 名称不为空
    ↓
4. 触发 tRPC Mutation：
   trpc.viewer.availability.schedule.update.useMutation()
    ↓
5. 服务端 Schema 验证
    ↓
6. Service 层处理：
   a. 获取现有日程
   b. 验证权限
   c. 处理默认日程变更
   d. 处理时区更新
   e. 处理可用性时间段
   f. 处理日期覆盖
    ↓
7. 数据库操作：
   a. prisma.schedule.update()
   b. 如有需要，更新默认日程状态
    ↓
8. 返回结果：{ schedule, prevDefaultId, currentDefaultId }
    ↓
9. 前端副作用：
   a. 失效相关缓存：
      - utils.viewer.availability.schedule.get.invalidate()
      - utils.viewer.availability.list.invalidate()
   b. 重新验证页面：
      - revalidateSchedulePage()
      - revalidateAvailabilityList()
   c. 显示成功提示
   d. 如果默认日程变更，失效旧日程的查询
```

### 7.6 可用性设置数据库模型

```prisma
// Schedule 表 - 存储日程
model Schedule {
  id              Int              @id @default(autoincrement())
  userId          Int
  user            User             @relation(fields: [userId], references: [id], onDelete: Cascade)
  name            String
  timeZone        String?          @default("Europe/London")
  isDefault       Boolean          @default(false)

  // 关联
  availability    Availability[]
  dateOverrides   DateOverride[]
  eventType       EventType?       @relation(fields: [eventTypeId], references: [id])
  eventTypeId     Int?

  // 元数据
  createdAt       DateTime         @default(now())
  updatedAt       DateTime         @updatedAt

  @@index([userId])
}

// Availability 表 - 存储可用性时间段
model Availability {
  id          Int       @id @default(autoincrement())
  scheduleId  Int
  schedule    Schedule  @relation(fields: [scheduleId], references: [id], onDelete: Cascade)
  days        Int[]
  startTime   DateTime
  endTime     DateTime

  @@index([scheduleId])
}

// DateOverride 表 - 存储日期覆盖（例外日期）
model DateOverride {
  id          Int       @id @default(autoincrement())
  scheduleId  Int
  schedule    Schedule  @relation(fields: [scheduleId], references: [id], onDelete: Cascade)
  start       DateTime
  end         DateTime

  @@index([scheduleId])
}

// User 表中的可用性相关字段
model User {
  // ...
  defaultScheduleId   Int?
  defaultSchedule     Schedule?        @relation("UserDefaultSchedule", fields: [defaultScheduleId], references: [id])
  schedules           Schedule[]
  // ...
}
```

## 8. tRPC 服务端实现

### 8.1 Router 定义

tRPC router 定义了 API 端点和它们的处理函数：

```typescript
export const meRouter = router({
  // ... 其他 procedures
  updateProfile: authedProcedure.input(ZUpdateProfileInputSchema).mutation(async ({ ctx, input }) => {
    const handler = (await import("./updateProfile.handler")).updateProfileHandler;
    return handler({ ctx, input });
  }),
  // ... 其他 procedures
});
```
[_router.tsx:37-40](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/me/_router.tsx#L37-L40)

### 8.2 输入 Schema 定义

服务端也有自己的输入验证 Schema，使用 Zod 定义：

```typescript
export const ZUpdateProfileInputSchema: z.ZodType<
  TUpdateProfileInputSchema,
  z.ZodTypeDef,
  TUpdateProfileInputSchemaInput
> = z.object({
  username: z.string().optional(),
  name: z.string().max(FULL_NAME_LENGTH_MAX_LIMIT).optional(),
  email: z.string().optional(),
  bio: z.string().optional(),
  avatarUrl: z.string().nullable().optional(),
  timeZone: timeZoneSchema.optional(),
  weekStart: z.string().optional(),
  hideBranding: z.boolean().optional(),
  allowDynamicBooking: z.boolean().optional(),
  allowSEOIndexing: z.boolean().optional(),
  receiveMonthlyDigestEmail: z.boolean().optional(),
  requiresBookerEmailVerification: z.boolean().optional(),
  brandColor: z.string().optional(),
  darkBrandColor: z.string().optional(),
  theme: z.string().optional().nullable(),
  appTheme: z.string().optional().nullable(),
  completedOnboarding: z.boolean().optional(),
  locale: z.string().optional(),
  timeFormat: z.number().optional(),
  metadata: userMetadata.optional(),
  travelSchedules: z
    .array(
      z.object({
        id: z.number().optional(),
        timeZone: timeZoneSchema,
        endDate: z.date().optional(),
        startDate: z.date(),
      })
    )
    .optional(),
  secondaryEmails: z
    .array(
      z.object({
        id: z.number(),
        email: z.string(),
        isDeleted: z.boolean().default(false),
      })
    )
    .optional(),
});
```
[updateProfile.schema.ts:84-128](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/me/updateProfile.schema.ts#L84-L128)

### 8.3 Handler 实现

Handler 函数处理具体的业务逻辑和数据库操作：

```typescript
export const updateProfileHandler = async ({ ctx, input }: UpdateProfileOptions) => {
  const { user } = ctx;
  const billingService = await getBillingProviderService();
  const userMetadata = handleUserMetadata({ ctx, input });
  const locale = input.locale || user.locale;
  const featuresRepository = new FeaturesRepository(prisma);
  const emailVerification = await featuresRepository.checkIfFeatureIsEnabledGlobally("email-verification");

  const { travelSchedules, ...rest } = input;

  const secondaryEmails = input?.secondaryEmails || [];
  delete input.secondaryEmails;

  const data: Prisma.UserUpdateInput = {
    ...rest,
    metadata: userMetadata,
    secondaryEmails: undefined,
  };

  // ... 业务逻辑处理

  let updatedUser: UpdatedUserResult;

  try {
    updatedUser = await prisma.user.update({
      where: {
        id: user.id,
      },
      data,
      ...updatedUserSelect,
    });
  } catch (e) {
    // 处理唯一约束冲突等错误
    if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === "P2002") {
      const meta = e.meta as { target: string[] };
      if (meta.target.indexOf("email") !== -1) {
        throw new HttpError({ statusCode: 409, message: "email_already_used" });
      }
    }
    throw e;
  }

  // ... 后续处理

  return {
    ...input,
    email: emailVerification && !secondaryEmail?.emailVerified ? user.email : input.email,
    avatarUrl: updatedUser.avatarUrl,
    hasEmailBeenChanged,
    sendEmailVerification: emailVerification && !secondaryEmail?.emailVerified,
  };
};
```
[updateProfile.handler.ts:41-400](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/me/updateProfile.handler.ts#L41-L400)

## 9. 数据库操作

### 9.1 使用 Prisma 进行数据操作

Cal.diy 使用 Prisma 作为 ORM，直接在 handler 中调用 Prisma 方法：

**更新用户资料**：
```typescript
updatedUser = await prisma.user.update({
  where: {
    id: user.id,
  },
  data,
  ...updatedUserSelect,
});
```
[updateProfile.handler.ts:265-271](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/me/updateProfile.handler.ts#L265-L271)

**创建日程**：
```typescript
const schedule = await prisma.schedule.create({
  data,
});
```
[create.handler.ts:61-63](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/availability/schedule/create.handler.ts#L61-L63)

### 9.2 数据库操作的错误处理

在 handler 中处理数据库操作的错误，例如唯一约束冲突：

```typescript
try {
  updatedUser = await prisma.user.update({
    where: {
      id: user.id,
    },
    data,
    ...updatedUserSelect,
  });
} catch (e) {
  // Catch unique constraint failure on email field.
  if (e instanceof Prisma.PrismaClientKnownRequestError && e.code === "P2002") {
    const meta = e.meta as { target: string[] };
    if (meta.target.indexOf("email") !== -1) {
      throw new HttpError({ statusCode: 409, message: "email_already_used" });
    }
  }
  throw e; // make sure other errors are rethrown
}
```
[updateProfile.handler.ts:264-281](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/me/updateProfile.handler.ts#L264-L281)

## 10. 副作用处理策略

### 10.1 缓存失效策略

#### tRPC 查询缓存失效

```typescript
// 模式 1：使特定查询失效
utils.viewer.me.invalidate();
utils.viewer.availability.schedule.get.invalidate({ scheduleId });

// 模式 2：重新获取特定查询
utils.viewer.availability.schedule.get.refetch({ scheduleId: prevDefaultId });

// 模式 3：使多个相关查询失效
utils.viewer.availability.list.invalidate();
utils.viewer.apps.getUsersDefaultConferencingApp.invalidate();
```

#### Next.js 页面重新验证

```typescript
// Server Actions 中的重新验证
import { revalidatePath } from "next/cache";

export async function revalidateSettingsProfile() {
  revalidatePath("/settings/my-account/profile");
}

export async function revalidateSettingsGeneral() {
  revalidatePath("/settings/my-account/general");
}

export async function revalidateAvailabilityList() {
  revalidatePath("/availability");
}

export async function revalidateSchedulePage(scheduleId: number) {
  revalidatePath(`/availability/${scheduleId}`);
}
```

### 10.2 乐观更新策略

#### 前端乐观更新示例

```typescript
const deleteMutation = trpc.viewer.availability.schedule.delete.useMutation({
  // onMutate：在 mutation 执行前触发
  onMutate: async ({ scheduleId }) => {
    // 1. 取消正在进行的查询
    await utils.viewer.availability.list.cancel();

    // 2. 保存当前数据用于回滚
    const previousValue = utils.viewer.availability.list.getData();

    // 3. 乐观更新：立即更新本地缓存
    if (previousValue) {
      const filteredValue = previousValue.schedules.filter(({ id }) => id !== scheduleId);
      utils.viewer.availability.list.setData(undefined, { ...previousValue, schedules: filteredValue });
    }

    // 4. 返回上下文（包含旧数据）
    return { previousValue };
  },

  // onError：发生错误时回滚
  onError: (err, variables, context) => {
    if (context?.previousValue) {
      // 恢复旧数据
      utils.viewer.availability.list.setData(undefined, context.previousValue);
    }
    // 显示错误提示
    if (err instanceof HttpError) {
      const message = `${err.statusCode}: ${err.message}`;
      showToast(message, "error");
    }
  },

  // onSettled：无论成功失败都执行
  onSettled: () => {
    // 失效缓存，确保与后端同步
    utils.viewer.availability.list.invalidate();
  },

  // onSuccess：成功时的额外处理
  onSuccess: () => {
    revalidateAvailabilityList();
    showToast(t("schedule_deleted_successfully"), "success");
  },
});
```
[availability-view.tsx:30-58](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/availability/availability-view.tsx#L30-L58)

### 10.3 状态刷新与通知

#### 用户会话更新

```typescript
// 个人资料更新后更新会话
const updateProfileMutation = trpc.viewer.me.updateProfile.useMutation({
  onSuccess: async (res) => {
    // 更新 NextAuth 会话
    await update(res);
    // ...
  },
});
```
[profile-view.tsx:74-76](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L74-L76)

#### Toast 通知

```typescript
// 成功提示
showToast(t("settings_updated_successfully"), "success");

// 错误提示
showToast(t("error_updating_settings"), "error");

// 特定消息
showToast(t("schedule_created_successfully", { scheduleName: schedule.name }), "success");
showToast(t("email_already_used"), "error");
```

#### Cookie 更新

```typescript
// Locale 变更时更新 Cookie
if (res.locale) {
  window.calNewLocale = res.locale;
  document.cookie = `calNewLocale=${res.locale}; path=/`;
}
```
[general-view.tsx:72-74](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/general-view.tsx#L72-L74)

### 10.4 副作用执行时机

```
Mutation 执行流程：
    ↓
1. onMutate（可选）
   ├── 取消正在进行的查询
   ├── 保存当前状态
   ├── 乐观更新
   └── 返回上下文
    ↓
2. 执行 tRPC 请求
    ↓
3. onSuccess 或 onError
   ├── onSuccess: 成功处理
   │   ├── 失效缓存
   │   ├── 重新验证页面
   │   ├── 更新会话/Cookie
   │   └── 显示成功提示
   │
   └── onError: 错误处理
       ├── 回滚乐观更新
       └── 显示错误提示
    ↓
4. onSettled（无论成功失败）
   └── 最终清理（如失效缓存）
```

## 11. 完整数据流示例

### 11.1 修改个人资料的完整流程

1. **用户填写表单**：在 `ProfileView` 组件中，用户填写个人资料表单
2. **客户端验证**：表单提交时，`react-hook-form` 使用 `zodResolver` 验证输入
3. **触发 Mutation**：验证通过后，调用 `updateProfileMutation.mutate(values)`
4. **tRPC 请求**：tRPC 客户端将请求发送到服务端
5. **服务端验证**：tRPC 服务端使用 `ZUpdateProfileInputSchema` 验证输入
6. **执行业务逻辑**：`updateProfileHandler` 处理业务逻辑，如检查用户名可用性、处理邮箱验证等
7. **数据库操作**：使用 `prisma.user.update()` 更新数据库
8. **返回结果**：handler 返回更新结果
9. **前端处理响应**：前端根据响应更新状态，显示成功或错误提示

### 11.2 修改可用性设置的流程（修正版）

**创建日程流程：**
1. **用户点击"新建日程"**：在可用性列表页面
2. **填写表单**：输入日程名称，选择可用性模板
3. **前端验证**：名称不为空
4. **触发 Mutation**：`trpc.viewer.availability.schedule.create.useMutation()`
5. **服务端验证**：`ZCreateInputSchema`
6. **Handler 处理**：
   - 验证权限
   - 准备创建数据
   - 处理可用性时间段
7. **数据库操作**：
   - `prisma.schedule.create()`
   - 如果是第一个日程，设置为默认日程
8. **返回结果**：`{ schedule }`
9. **前端副作用**：
   - `router.push(`/availability/${schedule.id}`)`
   - `showToast("schedule_created_successfully")`

**更新日程流程：**
1. **用户修改日程设置**：在日程编辑页面
2. **点击"保存"**：触发提交
3. **前端验证**：名称不为空
4. **触发 Mutation**：`trpc.viewer.availability.schedule.update.useMutation()`
5. **服务端验证**：`ZUpdateInputSchema`
6. **Service 层处理**：
   - 获取现有日程
   - 验证权限
   - 处理默认日程变更
   - 处理时区更新
   - 处理可用性时间段
   - 处理日期覆盖
7. **数据库操作**：`prisma.schedule.update()`
8. **返回结果**：`{ schedule, prevDefaultId, currentDefaultId }`
9. **前端副作用**：
   - 使相关查询缓存失效
   - 重新验证页面
   - 显示成功提示
   - 如果默认日程变更，失效旧日程查询

### 11.3 修改通知设置的流程

**月度摘要邮件设置：**
1. **用户切换开关**：在通用设置页面
2. **更新本地状态**：`setIsReceiveMonthlyDigestEmailChecked(checked)`
3. **触发 Mutation**：`mutation.mutate({ receiveMonthlyDigestEmail: checked })`
4. **服务端验证**：`ZUpdateProfileInputSchema`
5. **Handler 处理**：`updateProfileHandler`
6. **数据库操作**：`prisma.user.update()`
7. **返回结果**
8. **前端副作用**：
   - `utils.viewer.me.invalidate()`
   - `revalidateSettingsGeneral()`
   - `showToast("settings_updated_successfully")`

**推送通知设置：**
1. **用户点击"允许/禁用"**：在推送通知设置页面
2. **浏览器权限请求**：`Notification.requestPermission()`
3. **服务工作者订阅**：`serviceWorkerRegistration.pushManager.subscribe()`
4. **发送订阅信息到后端**：API 调用保存到数据库
5. **更新状态**：`isSubscribed`
6. **显示结果**：成功或失败提示

### 11.4 修改集成设置的流程

**设置默认会议应用：**
1. **用户选择应用**：在集成设置页面
2. **触发 Mutation**：`trpc.viewer.apps.setDefaultConferencingApp.useMutation()`
3. **服务端验证**：`ZSetDefaultConferencingAppSchema`
4. **Handler 处理**：
   - 检查应用是否存在
   - 验证权限
5. **数据库操作**：`prisma.user.update({ data: { defaultConferencingApp: appId } })`
6. **返回结果**
7. **前端副作用**：
   - `utils.viewer.apps.getUsersDefaultConferencingApp.invalidate()`
   - `showToast("settings_updated_successfully")`

## 12. 关键设计模式

### 12.1 分层架构

- **表示层**：React 组件，处理 UI 和用户交互
- **API 层**：tRPC routers 和 procedures，定义 API 端点
- **业务逻辑层**：Handler 函数和 Service 类，处理业务规则
- **数据访问层**：Prisma 操作，直接与数据库交互
- **副作用层**：缓存失效、状态刷新、通知触发等

### 12.2 双重验证

- **客户端验证**：使用 Zod 在前端进行快速验证，提供即时反馈
- **服务端验证**：同样使用 Zod 在服务端进行验证，确保数据安全性

### 12.3 统一错误处理

- 前端通过 `onError` 回调处理错误
- 服务端通过抛出 `TRPCError` 或 `HttpError` 传递错误信息
- 使用错误码（如 "email_already_used"）进行国际化错误提示

### 12.4 乐观更新与缓存失效

- 使用 `onMutate` 进行乐观更新
- 使用 `onError` 进行回滚
- 使用 `utils.viewer.*.invalidate()` 使相关查询缓存失效
- 使用 `revalidatePath()` 重新验证 Next.js 页面

### 12.5 Service 层抽象

复杂业务逻辑使用 Service 层封装：

```typescript
// 简单场景：直接在 Handler 中处理
// 如 updateProfileHandler

// 复杂场景：使用 Service 层
// 如 ScheduleService.updateSchedule()
export const updateHandler = async ({ input, ctx }: UpdateOptions) => {
  const { user } = ctx;
  return updateSchedule({
    input,
    user,
    prisma,
  });
};
```
[update.handler.ts:15-22](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/packages/trpc/server/routers/viewer/availability/schedule/update.handler.ts#L15-L22)

### 12.6 细粒度 Mutations

每个操作使用独立的 Mutation，而不是单个通用 Mutation：

```typescript
// 可用性设置的多个 Mutations
trpc.viewer.availability.schedule.create.useMutation()
trpc.viewer.availability.schedule.update.useMutation()
trpc.viewer.availability.schedule.delete.useMutation()
trpc.viewer.availability.schedule.duplicate.useMutation()
trpc.viewer.availability.schedule.bulkUpdateToDefaultAvailability.useMutation()
```

**优点：**
- 更清晰的职责分离
- 更好的类型安全
- 更精细的错误处理
- 更精细的缓存控制

## 13. 代码示例对比

