# 公开预约页表单渲染、SEO 元数据与缓存策略协作分析

## 一、整体架构概览

公开预约页采用 Next.js 13+ App Router 架构，结合服务端渲染（SSR）和客户端状态管理，实现高性能的预约体验。

### 核心文件结构

```
apps/web/
├── app/(booking-page-wrapper)/
│   ├── [user]/[type]/
│   │   ├── page.tsx          # 主页面入口，生成 SEO 元数据
│   │   └── embed/page.tsx    # 嵌入模式页面
│   ├── d/[link]/[slug]/
│   │   └── page.tsx          # 私密链接页面
│   └── layout.tsx            # 预约页布局
├── modules/
│   ├── bookings/
│   │   ├── components/
│   │   │   ├── BookerWebWrapper.tsx    # Web 包装器
│   │   │   ├── Booker.tsx               # 主 Booker 组件
│   │   │   └── BookEventForm/
│   │   │       ├── BookEventForm.tsx    # 预订表单
│   │   │       └── BookingFields.tsx    # 表单字段渲染
│   │   └── hooks/
│   │       ├── useBookings.ts            # 预订逻辑
│   │       ├── useSlots.ts               # 时间段管理
│   │       └── useVerifyEmail.ts         # 邮箱验证
└── users/
    └── views/
        └── users-type-public-view.tsx    # 公开页面视图
```

---

## 二、表单渲染机制

### 2.1 渲染流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        服务端渲染阶段                              │
│  [user]/[type]/page.tsx                                          │
│  ├── getServerSideProps() → 获取 eventData                       │
│  └── generateMetadata() → 生成 SEO 元数据                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        客户端渲染阶段                              │
│  users-type-public-view.tsx                                      │
│  └── BookerWebWrapper.tsx                                        │
│      ├── useInitializeBookerStore() → 初始化状态                │
│      ├── useEvent() → 获取事件详情                               │
│      ├── useBookingForm() → 初始化表单                          │
│      ├── useScheduleForEvent() → 获取可用时间段                 │
│      └── Booker.tsx → 渲染主组件                                 │
│          ├── 状态机: loading → selecting_date → selecting_time  │
│          │                                    → booking          │
│          └── BookEventForm.tsx → 渲染预订表单                   │
│              └── BookingFields.tsx → 渲染表单字段               │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件解析

#### BookerWebWrapper.tsx (`apps/web/modules/bookings/components/BookerWebWrapper.tsx`)

**职责**：作为 Web 端 Booker 的入口包装器，负责初始化状态和协调各个 hooks。

**关键功能**：
1. **数据获取策略**：
   - 服务端预取：`props.eventData` 来自 `getServerSideProps`
   - 客户端 fallback：使用 `useEvent()` hook 进行客户端获取
   ```typescript
   const clientFetchedEvent = useEvent({
     disabled: !!props.eventData,
     fromRedirectOfNonOrgLink: props.entity.fromRedirectOfNonOrgLink,
   });
   const event = props.eventData
     ? { data: props.eventData, isSuccess: true, isError: false, isPending: false }
     : clientFetchedEvent;
   ```

2. **状态初始化**：
   - `useInitializeBookerStore()` - 初始化全局状态
   - `useInitializeBookerStoreContext()` - 初始化 Context 状态

3. **表单管理**：
   ```typescript
   const bookerForm = useBookingForm({
     event: event.data,
     sessionEmail: session?.user.email,
     sessionName: session?.user.name,
     hasSession,
     extraOptions: routerQuery,
     prefillFormParams,
   });
   ```

4. **数据 hooks 协调**：
   - `useCalendars()` - 日历集成
   - `useVerifyEmail()` - 邮箱验证
   - `useSlots()` - 时间段选择
   - `useScheduleForEvent()` - 可用时间表
   - `useBookings()` - 预订提交逻辑

#### Booker.tsx (`apps/web/modules/bookings/components/Booker.tsx`)

**职责**：主渲染组件，管理 Booker 的不同状态和 UI 布局。

**状态机管理**：
```typescript
// 状态流转
loading → selecting_date → selecting_time → booking
            ↑                    ↓
            └────── 取消/返回 ───┘
```

**关键状态逻辑**：
```typescript
useEffect(() => {
  if (event.isPending) return setBookerState("loading");
  if (!selectedDate) return setBookerState("selecting_date");
  if (!selectedTimeslot) return setBookerState("selecting_time");
  const isSkipConfirmStepSupported = layout !== BookerLayouts.WEEK_VIEW;
  if (selectedTimeslot && skipConfirmStep && isSkipConfirmStepSupported)
    return setBookerState("selecting_time");
  return setBookerState("booking");
}, [event.isPending, selectedDate, selectedTimeslot, setBookerState, skipConfirmStep, layout]);
```

**布局系统**：
- `BookerLayouts.MONTH_VIEW` - 月视图
- `BookerLayouts.WEEK_VIEW` - 周视图
- `BookerLayouts.COLUMN_VIEW` - 列视图

使用 `AnimatePresence` 和 `BookerSection` 组件实现条件渲染和动画过渡。

#### BookEventForm.tsx (`apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`)

**职责**：预订确认表单，处理用户输入和提交。

**核心功能**：
1. **表单状态管理**：
   - 使用 `react-hook-form` 管理表单状态
   - 与 `BookerStoreContext` 同步表单值
   ```typescript
   <Form
     onChange={() => {
       const values = bookingForm.getValues();
       setFormValues(values);  // 保存到 store，支持导航后恢复
     }}
     form={bookingForm}
     handleSubmit={onSubmit}
     noValidate>
   ```

2. **动态表单字段**：
   - 通过 `BookingFields` 组件渲染动态字段
   - 支持不同位置类型、定价字段等

3. **错误处理**：
   - 表单验证错误
   - 预订提交错误
   - 时间段不可用提示

#### BookingFields.tsx (`apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx`)

**职责**：渲染动态表单字段，处理字段逻辑。

**字段类型处理**：
1. **系统字段（SystemField）**：
   - `location` - 位置选择
   - `guests` - 嘉宾列表
   - `notes` - 备注
   - `rescheduleReason` - 改期原因
   - `smsReminderNumber` - SMS 提醒号码

2. **动态字段特性**：
   - **重新安排时的只读逻辑**：
     ```typescript
     const rescheduleReadOnly =
       (field.editable === "system" || field.editable === "system-but-optional") &&
       !!rescheduleUid && bookingData !== null;
     ```

   - **位置字段特殊处理**：
     ```typescript
     if (field.name === SystemField.Enum.location && field.type === "radioInput") {
       const options = getLocationOptionsForSelect(locations, t);
       // 动态填充位置选项
       field.options = options.filter(...);
     }
     ```

   - **电话字段同步**：
     ```typescript
     const syncPhoneFields = (locationValue: unknown) => {
       // 当用户选择电话位置时，自动同步到其他电话字段
       otherPhoneFieldNames.forEach((name) => {
         if (!targetTouched) {
           setValue(`responses.${name}`, phone, {...});
         }
       });
     };
     ```

3. **定价字段**：
   - 支持字段级定价（`getFieldWithDirectPricing`）
   - 支持选项级定价（`getFieldWithOptionLevelPrices`）
   - 动态渲染价格标签

### 2.3 useBookingForm Hook (`packages/features/bookings/Booker/hooks/useBookingForm.ts`)

**职责**：表单核心逻辑，包括验证、默认值和状态管理。

**Schema 验证**：
```typescript
const bookingFormSchema = z
  .object({
    responses: event
      ? getBookingResponsesSchema({
          bookingFields: event.bookingFields,
          view: rescheduleUid ? "reschedule" : "booking",
          translateFn: (key, options) => t(key, options ?? {}),
        })
      : z.object({}),
  })
  .passthrough();
```

**表单初始化**：
- 使用 `useInitialFormValues` 获取初始值
- 支持查询参数预填充（`prefillFormParams`）
- 支持会话用户信息预填充

---

## 三、SEO 元数据生成机制

### 3.1 生成流程

```
┌─────────────────────────────────────────────────────────────────┐
│  [user]/[type]/page.tsx                                          │
│                                                                  │
│  export const generateMetadata = async ({ params, searchParams })│
│  {                                                                │
│    1. 构建 legacy 上下文                                          │
│    2. 调用 getData() 获取 eventData (与 getServerSideProps 同源) │
│    3. 提取 isSEOIndexable、eventData、isBrandingHidden          │
│    4. 调用 generateMeetingMetadata() 生成元数据                  │
│    5. 覆盖 robots 指令                                            │
│  }                                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 核心实现

#### 页面级 generateMetadata (`apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx`)

```typescript
export const generateMetadata = async ({ params, searchParams }: PageProps): Promise<Metadata> => {
  const legacyCtx = buildLegacyCtx(await headers(), await cookies(), await params, await searchParams);
  const props = await getData(legacyCtx);  // 与页面数据同

源

  const { booking, isSEOIndexable = true, eventData, isBrandingHidden } = props;
  const rescheduleUid = booking?.uid;
  const profileName = eventData?.profile?.name ?? "";
  const title = eventData?.title ?? "";

  const meeting = {
    title,
    profile: { name: profileName, image: eventData?.profile.image },
    users: eventData?.subsetOfUsers.map(...) || [],
  };

  const metadata = await generateMeetingMetadata(
    meeting,
    (t) => `${rescheduleUid && !!booking ? t("reschedule") : ""} ${title} | ${profileName}`,
    (t) => `${rescheduleUid ? t("reschedule") : ""} ${title}`,
    isBrandingHidden,
    WEBAPP_URL,
    `/${decodedParams.user}/${decodedParams.type}`
  );

  return {
    ...metadata,
    robots: {
      follow: !(eventData?.hidden || !isSEOIndexable),
      index: !(eventData?.hidden || !isSEOIndexable),
    },
  };
};
```

#### generateMeetingMetadata (`apps/web/app/_utils.tsx`)

**职责**：生成会议/预约相关的 SEO 元数据。

```typescript
export const generateMeetingMetadata = async (
  meeting: MeetingImageProps,
  getTitle: (t: TFunction) => string,
  getDescription: (t: TFunction) => string,
  hideBranding?: boolean,
  origin?: string,
  pathname?: string
) => {
  const metadata = await _generateMetadataWithoutImage(
    getTitle, getDescription, hideBranding, origin, pathname
  );
  
  // 生成 OG 图片
  const image = SEO_IMG_OGIMG + (await constructMeetingImage(meeting));

  return {
    ...metadata,
    openGraph: {
      ...metadata.openGraph,
      images: [image],
    },
  };
};
```

#### _generateMetadataWithoutImage (`apps/web/app/_utils.tsx`)

**职责**：生成基础元数据（不含图片）。

```typescript
const _generateMetadataWithoutImage = async (
  getTitle, getDescription, hideBranding, origin, pathname
) => {
  const canonical = buildCanonical({ path: pathname, origin });
  const t = await getTranslate();

  const title = getTitle(t);
  const description = getDescription(t);
  const titleSuffix = `| ${APP_NAME}`;
  const displayedTitle = title.includes(titleSuffix) || hideBranding 
    ? title 
    : `${title} ${titleSuffix}`;

  return {
    title: title.length === 0 ? APP_NAME : displayedTitle,
    description,
    alternates: { canonical },
    openGraph: {
      description: truncateOnWord(description, 158),
      url: canonical,
      type: "website",
      siteName: APP_NAME,
      title: displayedTitle,
    },
    metadataBase,
  };
};
```

### 3.3 元数据字段解析

| 字段 | 来源 | 说明 |
|------|------|------|
| **title** | `eventData.title` + `profile.name` | 格式：`{事件标题} | {用户名}` |
| **description** | 事件描述或默认值 | 支持国际化 |
| **canonical** | 构建自 `WEBAPP_URL` + 路径 | 规范 URL |
| **openGraph.images** | `constructMeetingImage()` | 动态生成的 OG 图片 |
| **openGraph.url** | canonical URL | OG 链接 |
| **robots.index/follow** | `eventData.hidden` + `isSEOIndexable` | 控制搜索引擎索引 |

### 3.4 SEO 控制逻辑

**索引控制**：
```typescript
robots: {
  follow: !(eventData?.hidden || !isSEOIndexable),
  index: !(eventData?.hidden || !isSEOIndexable),
}
```

**isSEOIndexable 来源** (`apps/web/server/lib/[user]/[type]/getServerSideProps.ts`)：
```typescript
const allowSEOIndexing = org
  ? user?.profile?.organization?.organizationSettings?.allowSEOIndexing
    ? user?.allowSEOIndexing
    : false
  : user?.allowSEOIndexing;
```

**条件说明**：
- 组织用户：需要组织设置允许 + 用户设置允许
- 个人用户：只需用户设置允许
- 事件隐藏（`eventData.hidden`）：强制不索引

---

## 四、缓存策略

### 4.1 缓存架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        缓存层级                                   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. HTTP 响应头缓存 (next.config.ts)                     │   │
│  │     - 静态资源: max-age=31536000, immutable            │   │
│  │     - 跨域资源: Cross-Origin-Resource-Policy            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  2. Next.js Data Cache (unstable_cache)                  │   │
│  │     - 服务端组件缓存                                       │   │
│  │     - 支持 TTL 和 cache tags                              │   │
│  │     - 自定义序列化 (superjson)                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  3. 客户端状态缓存                                         │   │
│  │     - React Query (tRPC) stale-while-revalidate         │   │
│  │     - localStorage (overlayCalendarSwitchDefault)        │   │
│  │     - BookerStore (Zustand) 表单状态                     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 HTTP 响应头缓存 (`apps/web/next.config.ts`)

**静态资源缓存**：
```typescript
{
  source: "/icons/sprite.svg(\\?v=[0-9a-zA-Z\\-\\.]+)?",
  headers: [
    {
      key: "Cache-Control",
      value: "public, max-age=31536000, immutable",  // 1年，不可变
    },
  ],
}
```

**安全相关头**：
```typescript
{
  source: "/:path*",
  headers: [
    { key: "X-Content-Type-Options", value: "nosniff" },
    { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  ],
}
```

**跨域资源**：
```typescript
{
  source: "/embed/embed.js",
  headers: [
    { key: "Cross-Origin-Resource-Policy", value: "cross-origin" },
  ],
}
```

### 4.3 Next.js Data Cache (unstable_cache)

#### 自定义缓存封装 (`packages/lib/unstable_cache/unstable_cache.ts`)

**问题**：Next.js 原生 `unstable_cache` 不支持复杂类型序列化。

**解决方案**：
```typescript
import { unstable_cache } from "next/cache";
import { parse, stringify } from "superjson";

export const cache = <T, P extends unknown[]>(
  fn: (...params: P) => Promise<T>,
  keys: Parameters<typeof unstable_cache>[1],
  opts: Parameters<typeof unstable_cache>[2]
) => {
  const wrap = async (params: unknown[]): Promise<string> => {
    const result = await fn(...(params as P));
    return stringify(result);  // 序列化
  };

  const cachedFn = unstable_cache(wrap, keys, opts);

  return async (...params: P): Promise<T> => {
    const result = await cachedFn(params);
    return parse(result);  // 反序列化
  };
};
```

#### 缓存使用示例

**旅行日程缓存** (`apps/web/app/cache/travelSchedule.ts`)：
```typescript
"use server";

import { revalidateTag } from "next/cache";
import { TravelScheduleRepository } from "@calcom/features/travelSchedule/repositories/TravelScheduleRepository";
import { NEXTJS_CACHE_TTL } from "@calcom/lib/constants";
import { unstable_cache } from "@calcom/lib/unstable_cache";

const CACHE_TAGS = {
  TRAVEL_SCHEDULES: "TravelRepository.findTravelSchedulesByUserId",
} as const;

export const getTravelSchedule = unstable_cache(
  async (userId: number) => {
    return await TravelScheduleRepository.findTravelSchedulesByUserId(userId);
  },
  ["getTravelSchedule"],  // 缓存键
  {
    revalidate: NEXTJS_CACHE_TTL,  // 3600 秒 = 1 小时
    tags: [CACHE_TAGS.TRAVEL_SCHEDULES],  // 缓存标签，用于失效
  }
);

export const revalidateTravelSchedules = async () => {
  revalidateTag(CACHE_TAGS.TRAVEL_SCHEDULES, "max");  // 按标签失效
};
```

**成员资格缓存** (`apps/web/app/cache/membership.ts`)：
```typescript
"use server";

import { MembershipRepository } from "@calcom/features/membership/repositories/MembershipRepository";
import { NEXTJS_CACHE_TTL } from "@calcom/lib/constants";
import { revalidateTag, unstable_cache } from "next/cache";

const CACHE_TAGS = {
  HAS_TEAM_PLAN: "MembershipRepository.hasAnyAcceptedMembershipByUserId",
} as const;

export const getCachedHasTeamPlan = unstable_cache(
  async (userId: number) => {
    const hasTeamPlan = await MembershipRepository.hasAnyAcceptedMembershipByUserId(userId);
    return { hasTeamPlan: !!hasTeamPlan };
  },
  ["getCachedHasTeamPlan"],
  {
    revalidate: NEXTJS_CACHE_TTL,
    tags: [CACHE_TAGS.HAS_TEAM_PLAN],
  }
);
```

#### 缓存配置常量 (`packages/lib/constants.ts`)

```typescript
// 缓存 TTL: 1 小时
export const NEXTJS_CACHE_TTL = 3600;

// 时间段查询相关缓存配置
export const PUBLIC_QUERY_RESERVATION_INTERVAL_SECONDS = 
  parseInt(process.env.NEXT_PUBLIC_QUERY_RESERVATION_INTERVAL_SECONDS ?? "", 10) || 30;

export const PUBLIC_QUERY_RESERVATION_STALE_TIME_SECONDS =
  parseInt(process.env.NEXT_PUBLIC_QUERY_RESERVATION_STALE_TIME_SECONDS ?? "", 10) || 20;

export const PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS =
  parseInt(process.env.NEXT_PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS ?? "", 10) || 5 * 60;
```

### 4.4 客户端缓存策略

#### React Query / tRPC 缓存

**可用时间段失效** (`apps/web/modules/bookings/components/Booker.tsx`)：
```typescript
onCancel={() => {
  setSelectedTimeslot(null);
  // 当用户取消预订时，可选地失效时间段缓存
  if (PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM) {
    schedule?.invalidate();  // 确保用户获得最新的可用时间段
  }
  // ...
}}
```

#### LocalStorage 缓存

**日历覆盖开关** (`apps/web/modules/bookings/components/BookerWebWrapper.tsx`)：
```typescript
const onOverlaySwitchStateChange = useCallback(
  (state: boolean) => {
    const url = new URL(window.location.href);
    if (state) {
      url.searchParams.set("overlayCalendar", "true");
      localStorage.setItem("overlayCalendarSwitchDefault", "true");  // 持久化
    } else {
      url.searchParams.delete("overlayCalendar");
      localStorage.removeItem("overlayCalendarSwitchDefault");
    }
    router.push(`${url.pathname}${url.search}`);
  },
  [router]
);
```

#### BookerStore 状态缓存

**表单值持久化** (`apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx`)：
```typescript
<Form
  onChange={() => {
    // 表单数据保存到 store，用户导航后返回时仍保留
    const values = bookingForm.getValues();
    setFormValues(values);
  }}
  // ...
>
```

---

## 五、三者协作机制

### 5.1 数据流向图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            首次请求流程                                    │
│                                                                          │
│  用户请求 /[user]/[type]                                                 │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  Next.js 服务端                                                   │   │
│  │                                                                   │   │
│  │  1. 调用 getServerSideProps()                                    │   │
│  │     ├── EventRepository.getPublicEvent() → 查询数据库           │   │
│  │     └── 返回 props: { eventData, isSEOIndexable, ... }         │   │
│  │                                                                   │   │
│  │  2. 调用 generateMetadata()                                      │   │
│  │     ├── 复用相同的 getData() 逻辑                                │   │
│  │     ├── 调用 generateMeetingMetadata()                          │   │
│  │     └── 生成: title, description, og:image, robots             │   │
│  │                                                                   │   │
│  │  3. 渲染 ServerPage                                              │   │
│  │     └── 传递 props 给客户端组件                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  客户端 (Hydration)                                              │   │
│  │                                                                   │   │
│  │  1. BookerWebWrapper 初始化                                      │   │
│  │     ├── 使用服务端预取的 eventData (无需重复请求)                │   │
│  │     ├── 初始化 BookerStore                                       │   │
│  │     └── 注册 hooks: useEvent, useBookingForm, etc.             │   │
│  │                                                                   │   │
│  │  2. 动态数据获取 (React Query/tRPC)                              │   │
│  │     ├── useScheduleForEvent() → 获取可用时间段                  │   │
│  │     ├── useSlots() → 获取具体时间段                              │   │
│  │     └── useCalendars() → 获取用户日历（如果有会话）             │   │
│  │                                                                   │   │
│  │  3. 表单渲染                                                      │   │
│  │     ├── useBookingForm → 初始化 react-hook-form                 │   │
│  │     └── BookingFields → 动态渲染字段                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 服务端数据共享机制

**核心设计**：`getServerSideProps` 和 `generateMetadata` 共享相同的数据源。

```typescript
// [user]/[type]/page.tsx

// 1. 定义数据获取函数
const getData: (ctx: ReturnType<typeof buildLegacyCtx>) => Promise<LegacyPageProps> =
  withAppDirSsr<LegacyPageProps>(getServerSideProps);

// 2. generateMetadata 使用相同的 getData
export const generateMetadata = async ({ params, searchParams }: PageProps): Promise<Metadata> => {
  const legacyCtx = buildLegacyCtx(await headers(), await cookies(), await params, await searchParams);
  const props = await getData(legacyCtx);  // 与页面数据同源
  // ...
};

// 3. ServerPage 也使用相同的 getData
const ServerPage = async ({ params, searchParams }: PageProps): Promise<JSX.Element> => {
  const legacyCtx = buildLegacyCtx(await headers(), await cookies(), await params, await searchParams);
  const props = await getData(legacyCtx);  // 同样的数据源
  // ...
};
```

**优势**：
1. **数据一致性**：SEO 元数据和页面内容使用相同的数据
2. **避免重复查询**：Next.js 会自动缓存相同参数的 `getData` 调用
3. **单一数据源**：便于维护和调试

### 5.3 状态流转与缓存交互

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        用户交互流程                                        │
│                                                                          │
│  初始状态: bookerState = "loading"                                      │
│       │                                                                  │
│       ▼ (event 加载完成)                                                 │
│  bookerState = "selecting_date"                                         │
│       │                                                                  │
│       ├─── 用户选择日期 ──────────────────────────────────────┐         │
│       │                                                        │         │
│       ▼                                                        ▼         │
│  bookerState = "selecting_time"                    useScheduleForEvent  │
│       │                                                (React Query)    │
│       │                                                        │         │
│       ├─── 用户选择时间段 ───────────────────────────────────┤         │
│       │                                                        │         │
│       ▼                                                        │         │
│  bookerState = "booking"                                      │         │
│       │                                                        │         │
│       ├─── 表单渲染 ──────────────────────────────────────────┤         │
│       │    ├── useBookingForm 初始化                         │         │
│       │    ├── BookingFields 动态渲染                        │         │
│       │    └── 表单值同步到 BookerStore                      │         │
│       │                                                        │         │
│       ├─── 用户点击取消 ──────────────────────────────────────┤         │
│       │                                                        │         │
│       ▼                                                        ▼         │
│  bookerState = "selecting_time"              schedule?.invalidate()    │
│       │                                              (可选)              │
│       │                                                        │         │
│       └─── 用户提交表单 ──────────────────────────────────────┘         │
│            │                                                             │
│            ▼                                                             │
│  useBookings.handleBookEvent()                                           │
│       │                                                                  │
│       ├── 调用 tRPC mutation                                             │
│       ├── 处理错误 (显示 Alert)                                          │
│       └── 成功时导航到确认页                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.4 缓存失效场景

| 场景 | 触发方式 | 失效范围 |
|------|----------|----------|
| **用户取消预订** | `schedule?.invalidate()` | 可用时间段缓存 |
| **旅行日程变更** | `revalidateTravelSchedules()` | 按 tag: TravelRepository.findTravelSchedulesByUserId |
| **成员资格变更** | `revalidateHasTeamPlan()` | 按 tag: MembershipRepository.hasAnyAcceptedMembershipByUserId |
| **新预订创建** | tRPC mutation 自动失效 | 相关查询缓存 |
| **页面重新部署** | Next.js 自动 | 所有 Data Cache |

### 5.5 SEO 与表单数据的关联

**动态元数据依赖**：
```typescript
// generateMetadata 中依赖的表单相关数据
const meeting = {
  title: eventData?.title ?? "",                    // 表单中显示的事件标题
  profile: { 
    name: eventData?.profile?.name ?? "",          // 组织者名称
    image: eventData?.profile.image                 // 组织者头像（用于 OG 图片）
  },
  users: eventData?.subsetOfUsers.map(...) || [],  // 参与者列表
};
```

**OG 图片生成**：
- 使用 `constructMeetingImage(meeting)` 动态生成图片
- 图片包含：事件标题、组织者名称、参与者头像等
- 图片 URL 被添加到 `openGraph.images`

**索引控制与表单可用性**：
- 如果 `eventData.hidden === true`，页面不被搜索引擎索引
- 这通常用于私密事件或测试事件
- 表单仍然可以通过直接链接访问

---

## 六、关键代码位置索引

### 6.1 表单渲染相关

| 功能 | 文件路径 | 关键函数/组件 |
|------|----------|---------------|
| 主页面入口 | `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx` | `ServerPage`, `generateMetadata` |
| 公开页面视图 | `apps/web/modules/users/views/users-type-public-view.tsx` | `Type` 组件 |
| Web 包装器 | `apps/web/modules/bookings/components/BookerWebWrapper.tsx` | `BookerWebWrapper`, 所有 hooks |
| 主 Booker 组件 | `apps/web/modules/bookings/components/Booker.tsx` | `Booker`, 状态机逻辑 |
| 预订表单 | `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx` | `BookEventForm` |
| 表单字段 | `apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx` | `BookingFields`, `syncPhoneFields` |
| 表单 Hook | `packages/features/bookings/Booker/hooks/useBookingForm.ts` | `useBookingForm` |

### 6.2 SEO 元数据相关

| 功能 | 文件路径 | 关键函数/组件 |
|------|----------|---------------|
| 元数据生成工具 | `apps/web/app/_utils.tsx` | `generateMeetingMetadata`, `_generateMetadata` |
| OG 图片构造 | `packages/lib/OgImages.ts` | `constructMeetingImage` |
| 规范 URL 构建 | `packages/lib/next-seo.config.ts` | `buildCanonical` |

### 6.3 缓存策略相关

| 功能 | 文件路径 | 关键函数/组件 |
|------|----------|---------------|
| HTTP 缓存头 | `apps/web/next.config.ts` | `headers()` 配置 |
| 自定义缓存封装 | `packages/lib/unstable_cache/unstable_cache.ts` | `cache` 函数 |
| 旅行日程缓存 | `apps/web/app/cache/travelSchedule.ts` | `getTravelSchedule`, `revalidateTravelSchedules` |
| 成员资格缓存 | `apps/web/app/cache/membership.ts` | `getCachedHasTeamPlan`, `revalidateHasTeamPlan` |
| 缓存常量 | `packages/lib/constants.ts` | `NEXTJS_CACHE_TTL` 等 |

---

## 七、性能优化建议

### 7.1 已实施的优化

1. **服务端预取**：`getServerSideProps` 在服务端获取 `eventData`，避免客户端首次请求
2. **数据共享**：`generateMetadata` 和页面组件共享 `getData` 调用
3. **表单状态持久化**：使用 `BookerStore` 保存表单值，支持导航后恢复
4. **条件渲染**：使用 `AnimatePresence` 和 `BookerSection` 实现按需渲染
5. **stale-while-revalidate**：React Query 默认策略，先显示缓存数据再更新

### 7.2 潜在优化点

1. **EventRepository 缓存**：
   - 当前 `getPublicEvent` 每次都查询数据库
   - 建议：为公开事件添加 `unstable_cache` 包装

2. **时间段查询优化**：
   - 当前 `useScheduleForEvent` 每次都请求
   - 建议：根据日期范围实现更细粒度的缓存

3. **OG 图片缓存**：
   - 当前 `constructMeetingImage` 每次都生成
   - 建议：添加 CDN 缓存或服务端缓存

4. **表单字段 Schema 缓存**：
   - `getBookingResponsesSchema` 每次都解析
   - 建议：按 `eventId` 缓存解析结果

---

## 八、安全考虑

### 8.1 缓存安全

1. **敏感数据不缓存**：
   - `credential.key` 字段绝不出现在 API 响应或查询中（遵循项目规则）
   - 用户隐私数据不使用 `unstable_cache` 缓存

2. **缓存标签命名**：
   - 使用 `Repository.methodName` 格式，便于追踪
   - 避免使用可能冲突的标签名

### 8.2 SEO 安全

1. **robots 指令**：
   - 私密事件强制 `noindex, nofollow`
   - 组织设置和用户设置双重检查

2. **元数据泄露**：
   - OG 图片不包含敏感信息
   - 描述字段经过 `truncateOnWord` 处理，避免过长

---

## 九、总结

公开预约页的表单渲染、SEO 元数据和缓存策略通过以下方式紧密协作：

1. **数据层面**：`getServerSideProps` 和 `generateMetadata` 共享相同的数据源，确保 SEO 元数据与页面内容一致。

2. **状态层面**：服务端预取的数据用于初始化客户端状态，减少了客户端请求，同时表单状态通过 `BookerStore` 持久化，提升用户体验。

3. **缓存层面**：多层缓存策略（HTTP 头 → Data Cache → 客户端状态）确保了性能，同时精细的缓存标签和失效机制保证了数据新鲜度。

4. **SEO 层面**：元数据生成与表单数据紧密关联，动态生成的标题、描述和 OG 图片提升了搜索引擎排名和社交媒体分享效果。

这种设计既保证了用户体验（快速加载、状态保留），又保证了可发现性（SEO 优化），同时通过分层缓存策略实现了高性能。
