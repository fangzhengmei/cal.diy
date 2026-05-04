# Settings 页面 tRPC Mutation 协同工作分析

## 1. 整体架构概览

在 Cal.diy 项目中，Settings 页面的表单提交、数据验证、tRPC 调用和数据库操作形成了一个完整的数据流。这个流程主要包含以下几个层次：

1. **前端表单层**：使用 React Hook Form 管理表单状态
2. **客户端验证层**：使用 Zod 进行前端数据验证
3. **tRPC 客户端层**：通过 tRPC hooks 调用后端 API
4. **tRPC 服务端层**：处理业务逻辑和服务端验证
5. **数据访问层**：使用 Prisma 进行数据库操作

## 2. 前端表单与客户端验证

### 2.1 表单管理

Cal.diy 使用 `react-hook-form` 作为表单状态管理库，结合 `zod` 和 `@hookform/resolvers/zod` 进行数据验证。

**关键依赖导入**：
```typescript
import { Controller, useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
```
[profile-view.tsx:41-42](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/profile-view.tsx#L41-L42)

### 2.2 表单值类型定义

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

### 2.3 客户端验证 Schema

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

### 2.4 表单初始化与提交

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

## 3. tRPC 客户端调用

### 3.1 tRPC Mutation 初始化

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

### 3.2 触发 Mutation

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

### 3.3 通用设置中的 Mutation 调用

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

## 4. tRPC 服务端实现

### 4.1 Router 定义

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

### 4.2 输入 Schema 定义

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

### 4.3 Handler 实现

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

## 5. 数据库操作

### 5.1 使用 Prisma 进行数据操作

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

### 5.2 数据库操作的错误处理

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

## 6. 完整数据流示例

### 6.1 修改个人资料的完整流程

1. **用户填写表单**：在 `ProfileView` 组件中，用户填写个人资料表单
2. **客户端验证**：表单提交时，`react-hook-form` 使用 `zodResolver` 验证输入
3. **触发 Mutation**：验证通过后，调用 `updateProfileMutation.mutate(values)`
4. **tRPC 请求**：tRPC 客户端将请求发送到服务端
5. **服务端验证**：tRPC 服务端使用 `ZUpdateProfileInputSchema` 验证输入
6. **执行业务逻辑**：`updateProfileHandler` 处理业务逻辑，如检查用户名可用性、处理邮箱验证等
7. **数据库操作**：使用 `prisma.user.update()` 更新数据库
8. **返回结果**：handler 返回更新结果
9. **前端处理响应**：前端根据响应更新状态，显示成功或错误提示

### 6.2 修改可用性设置的流程

1. **用户修改日程**：在可用性设置页面，用户修改日程安排
2. **表单提交**：通过表单提交修改
3. **调用 tRPC Mutation**：使用 `trpc.viewer.availability.schedule.update.useMutation()`
4. **服务端处理**：`updateHandler` 调用 `ScheduleService` 中的 `updateSchedule` 方法
5. **数据库操作**：`ScheduleService` 使用 Prisma 更新数据库中的日程信息
6. **返回结果**：返回更新后的日程信息

## 7. 关键设计模式

### 7.1 分层架构

- **表示层**：React 组件，处理 UI 和用户交互
- **API 层**：tRPC routers 和 procedures，定义 API 端点
- **业务逻辑层**：Handler 函数和 Service 类，处理业务规则
- **数据访问层**：Prisma 操作，直接与数据库交互

### 7.2 双重验证

- **客户端验证**：使用 Zod 在前端进行快速验证，提供即时反馈
- **服务端验证**：同样使用 Zod 在服务端进行验证，确保数据安全性

### 7.3 统一错误处理

- 前端通过 `onError` 回调处理错误
- 服务端通过抛出 `TRPCError` 或 `HttpError` 传递错误信息
- 使用错误码（如 "email_already_used"）进行国际化错误提示

### 7.4 乐观更新与缓存失效

- 使用 `utils.viewer.me.invalidate()` 使相关查询缓存失效
- 在 mutation 成功后重新获取数据，确保 UI 与后端同步

## 8. 代码示例对比

### 8.1 个人资料更新 vs 日程更新

**个人资料更新**：
- 直接在 handler 中处理业务逻辑和数据库操作
- 涉及多个相关操作：用户表更新、次要邮箱处理、头像上传等
- 较复杂的业务逻辑，如邮箱验证、用户名检查等

**日程更新**：
- 使用 Service 层（`ScheduleService`）封装业务逻辑
- handler 只是简单地调用 Service 方法
- 更清晰的分离关注点

### 8.2 通用设置中的简单开关 vs 复杂表单

**简单开关（如动态预订）**：
```typescript
<SettingsToggle
  // ... 其他属性
  onCheckedChange={(checked) => {
    setIsAllowDynamicBookingChecked(checked);
    mutation.mutate({ allowDynamicBooking: checked });
  }}
  // ... 其他属性
/>
```
[general-view.tsx:326-337](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/general-view.tsx#L326-L337)

**复杂表单（如时区设置）**：
```typescript
<Form
  form={formMethods}
  handleSubmit={async (values) => {
    setIsUpdateBtnLoading(true);
    mutation.mutate({
      ...values,
      locale: values.locale.value,
      timeFormat: values.timeFormat.value,
      weekStart: values.weekStart.value,
    });
  }}>
  {/* 表单字段 */}
</Form>
```
[general-view.tsx:155-165](file:///h:/fz/solo-dogfeeding/code/72-cal.diy/apps/web/modules/settings/my-account/general-view.tsx#L155-L165)

## 9. 最佳实践总结

1. **使用类型安全**：全程使用 TypeScript，确保类型安全
2. **双重验证**：客户端和服务端都进行数据验证
3. **错误处理**：统一的错误处理机制，使用错误码进行国际化
4. **缓存管理**：合理使用 tRPC 的缓存失效机制
5. **业务逻辑分离**：复杂业务逻辑使用 Service 层封装
6. **小批量更新**：支持部分字段更新，减少数据传输
7. **乐观更新**：在适当情况下使用乐观更新提升用户体验

## 10. 关键文件索引

- **前端表单组件**：
  - `apps/web/modules/settings/my-account/profile-view.tsx` - 个人资料设置
  - `apps/web/modules/settings/my-account/general-view.tsx` - 通用设置
  - `apps/web/modules/settings/my-account/appearance-view.tsx` - 外观设置

- **tRPC Routers**：
  - `packages/trpc/server/routers/viewer/me/_router.tsx` - 用户相关 API
  - `packages/trpc/server/routers/viewer/availability/schedule/_router.ts` - 日程相关 API

- **tRPC Handlers**：
  - `packages/trpc/server/routers/viewer/me/updateProfile.handler.ts` - 更新个人资料
  - `packages/trpc/server/routers/viewer/availability/schedule/create.handler.ts` - 创建日程
  - `packages/trpc/server/routers/viewer/availability/schedule/update.handler.ts` - 更新日程

- **Schema 定义**：
  - `packages/trpc/server/routers/viewer/me/updateProfile.schema.ts` - 更新个人资料输入验证
  - `packages/trpc/server/routers/viewer/availability/schedule/create.schema.ts` - 创建日程输入验证
