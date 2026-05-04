# 公开预约页表单渲染、SEO 元数据与缓存策略协作分析

本文档基于代码证据，梳理公开预约页（`/[user]/[type]`）的渲染流程、元数据生成和缓存策略的可对账链路。

---

## 一、代码位置索引

| 模块 | 文件路径 |
|------|----------|
| 页面入口 | `apps/web/app/(booking-page-wrapper)/[user]/[type]/page.tsx` |
| SSR 数据获取 | `apps/web/server/lib/[user]/[type]/getServerSideProps.ts` |
| 公开事件查询 | `packages/features/eventtypes/lib/getPublicEvent.ts` |
| 元数据工具 | `apps/web/app/_utils.tsx` |
| Booker 包装器 | `apps/web/modules/bookings/components/BookerWebWrapper.tsx` |
| Booker 主组件 | `apps/web/modules/bookings/components/Booker.tsx` |
| 预订表单 | `apps/web/modules/bookings/components/BookEventForm/BookEventForm.tsx` |
| 表单字段 | `apps/web/modules/bookings/components/BookEventForm/BookingFields.tsx` |
| tRPC 配置 | `packages/trpc/react/trpc.ts` |
| Next.js 配置 | `apps/web/next.config.ts` |

---

## 二、渲染流程可对账链路

### 2.1 整体流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户请求 /[user]/[type]                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Next.js 服务端处理阶段                            │
│                                                                          │
│  1. 调用 generateMetadata() 生成 SEO 元数据                             │
│     └── 调用 getData() → getServerSideProps() → 数据库查询             │
│                                                                          │
│  2. 调用 ServerPage() 渲染页面组件                                       │
│     └── 调用 getData() → getServerSideProps() → 数据库查询             │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        客户端 Hydration 阶段                              │
│                                                                          │
│  1. users-type-public-view.tsx                                          │
│     └── 接收服务端传递的 eventData props                                 │
│         └── 渲染 BookerWebWrapper                                        │
│                                                                          │
│  2. BookerWebWrapper.tsx                                                 │
│     ├── 若 props.eventData 存在，直接使用（跳过客户端获取）             │
│     ├── 初始化 BookerStore                                               │
│     ├── useBookingForm() - 初始化表单                                   │
│     ├── useScheduleForEvent() - 获取可用时间段（tRPC）                 │
│     └── 渲染 Booker 组件                                                 │
│                                                                          │
│  3. Booker.tsx                                                           │
│     ├── 状态机管理: loading → selecting_date → selecting_time → booking│
│     └── 条件渲染 BookEventForm                                           │
│                                                                          │
│  4. BookEventForm.tsx                                                    │
│     └── 渲染 BookingFields + 提交按钮                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 链路 1：服务端数据获取（getServerSideProps）

#### 输入

| 输入项 | 来源 | 代码位置 |
|--------|------|----------|
| `context.params.user` | URL 路径参数 | `getServerSideProps.ts:314-316` |
| `context.params.type` | URL 路径参数 | `getServerSideProps.ts:314-316` |
| `context.query.rescheduleUid` | 查询参数 | `getServerSideProps.ts:117` |
| `context.query.bookingUid` | 查询参数 | `getServerSideProps.ts:117` |
| `context.query.orgRedirection` | 查询参数 | `getServerSideProps.ts:250` |

#### 处理流程

**Step 1：参数解析**
```typescript
// getServerSideProps.ts:307-310
const paramsSchema = z.object({
  type: z.string().transform((s) => slugify(s)),
  user: z.string().transform((s) => getUsernameList(s)),
});

// getServerSideProps.ts:314-318
export const getServerSideProps = async (context: GetServerSidePropsContext) => {
  const { user } = paramsSchema.parse(context.params);
  const isDynamicGroup = user.length > 1;
  return isDynamicGroup ? await getDynamicGroupPageProps(context) : await getUserPageProps(context);
};
```

**Step 2：事件查询（直接查数据库，无缓存）**
```typescript
// getServerSideProps.ts:245-253 (getUserPageProps)
const eventData = await EventRepository.getPublicEvent(
  {
    username,
    eventSlug: slug,
    org,
    fromRedirectOfNonOrgLink: context.query.orgRedirection === "true",
  },
  session?.user?.id
);
```

```typescript
// EventRepository.ts:12-24
export class EventRepository {
  static async getPublicEvent(input: GetPublicEventInput, userId?: number) {
    // 直接调用 getPublicEvent，无缓存包装
    const event = await getPublicEvent(
      input.username,
      input.eventSlug,
      input.isTeamEvent,
      input.org,
      prisma,
      input.fromRedirectOfNonOrgLink,
      userId
    );
    return event;
  }
}
```

```typescript
// getPublicEvent.ts:430-436
let event = await prisma.eventType.findFirst({
  where: {
    slug: eventSlug,
    ...usersOrTeamQuery,
  },
  select: getPublicEventSelect(fetchAllUsers),
});
```

**Step 3：SEO 可索引性判断**
```typescript
// getServerSideProps.ts:261-265
const allowSEOIndexing = org
  ? user?.profile?.organization?.organizationSettings?.allowSEOIndexing
    ? user?.allowSEOIndexing
    : false
  : user?.allowSEOIndexing;
```

**Step 4：改期/座位预订处理**
```typescript
// getServerSideProps.ts:281-300
if (rescheduleUid) {
  const processRescheduleResult = await processReschedule({
    props,
    rescheduleUid,
    session,
    allowRescheduleForCancelledBooking,
  });
  if (processRescheduleResult) {
    return processRescheduleResult;  // 可能返回 redirect 或 notFound
  }
}
```

#### 输出

| 输出项 | 类型 | 代码位置 |
|--------|------|----------|
| `eventData` | `PublicEventType` | `getServerSideProps.ts:268` |
| `isSEOIndexable` | `boolean \| null` | `getServerSideProps.ts:275` |
| `isBrandingHidden` | `boolean` | `getServerSideProps.ts:271-274` |
| `rescheduleUid` | `string \| null` | `getServerSideProps.ts:277` |
| `bookingUid` | `string \| null` | `getServerSideProps.ts:276` |
| `redirect` | `{ destination: string, permanent: boolean }` | `getServerSideProps.ts:48-52` |
| `notFound` | `true` | `getServerSideProps.ts:144-146` |

#### 边界场景

| 场景 | 触发条件 | 处理方式 | 代码位置 |
|------|----------|----------|----------|
| 事件不存在 | `!eventData` | 返回 `notFound: true` | `getServerSideProps.ts:255-259` |
| 用户不存在 | `!user` | 返回 `notFound: true` | `getServerSideProps.ts:235-239` |
| 事件禁止改期 | `booking.eventType.disableRescheduling` | 重定向到 `/booking/[uid]` | `getServerSideProps.ts:46-52` |
| 已取消预订改期 | `booking.status === CANCELLED` | 需 `allowRescheduleForCancelledBooking=true` | `getServerSideProps.ts:59-63` |

---

### 2.3 链路 2：元数据生成（generateMetadata）

#### 输入

| 输入项 | 来源 | 代码位置 |
|--------|------|----------|
| `params` | URL 路径参数 | `page.tsx:36` |
| `searchParams` | URL 查询参数 | `page.tsx:36` |
| `headers()` | 请求头 | `page.tsx:37` |
| `cookies()` | Cookie | `page.tsx:37` |

#### 处理流程

**Step 1：构建 legacy 上下文**
```typescript
// page.tsx:37
const legacyCtx = buildLegacyCtx(await headers(), await cookies(), await params, await searchParams);
```

**Step 2：获取数据（与页面渲染共享相同逻辑）**
```typescript
// page.tsx:38
const props = await getData(legacyCtx);  // 调用 withAppDirSsr(getServerSideProps)
```

**Step 3：提取元数据相关字段**
```typescript
// page.tsx:40-52
const { booking, isSEOIndexable = true, eventData, isBrandingHidden } = props;
const rescheduleUid = booking?.uid;
const profileName = eventData?.profile?.name ?? "";
const title = eventData?.title ?? "";

const meeting = {
  title,
  profile: { name: profileName, image: eventData?.profile.image },
  users: eventData?.subsetOfUsers.map((user) => ({
    name: `${user.name}`,
    username: `${user.username}`,
  })) || [],
};
```

**Step 4：调用元数据生成工具**
```typescript
// page.tsx:54-61
const metadata = await generateMeetingMetadata(
  meeting,
  (t) => `${rescheduleUid && !!booking ? t("reschedule") : ""} ${title} | ${profileName}`,
  (t) => `${rescheduleUid ? t("reschedule") : ""} ${title}`,
  isBrandingHidden,
  WEBAPP_URL,
  `/${decodedParams.user}/${decodedParams.type}`
);
```

**Step 5：覆盖 robots 指令**
```typescript
// page.tsx:63-69
return {
  ...metadata,
  robots: {
    follow: !(eventData?.hidden || !isSEOIndexable),
    index: !(eventData?.hidden || !isSEOIndexable),
  },
};
```

#### generateMeetingMetadata 内部流程

```typescript
// _utils.tsx:125-149
export const generateMeetingMetadata = async (
  meeting: MeetingImageProps,
  getTitle, getDescription, hideBranding, origin, pathname
) => {
  // Step 1: 生成基础元数据（不含图片）
  const metadata = await _generateMetadataWithoutImage(
    getTitle, getDescription, hideBranding, origin, pathname
  );
  
  // Step 2: 构建 OG 图片 URL
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

```typescript
// _utils.tsx:21-51
const _generateMetadataWithoutImage = async (...) => {
  const canonical = buildCanonical({ path: pathname, origin });
  const t = await getTranslate();

  const title = getTitle(t);
  const description = getDescription(t);
  const displayedTitle = hideBranding ? title : `${title} | ${APP_NAME}`;

  return {
    title: displayedTitle,
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

#### 输出

| 输出字段 | 数据源 | 代码位置 |
|----------|--------|----------|
| `title` | `eventData.title` + `eventData.profile.name` | `page.tsx:56` |
| `description` | `eventData.title`（改期时加 "reschedule"） | `page.tsx:57` |
| `alternates.canonical` | `WEBAPP_URL` + 路径 | `_utils.tsx:29` |
| `openGraph.title` | 同 `title` | `_utils.tsx:47` |
| `openGraph.description` | 截断为 158 字符 | `_utils.tsx:43` |
| `openGraph.url` | canonical URL | `_utils.tsx:44` |
| `openGraph.images` | `constructMeetingImage(meeting)` | `_utils.tsx:140` |
| `robots.index` | `!(eventData.hidden \|\| !isSEOIndexable)` | `page.tsx:67` |
| `robots.follow` | 同 `index` | `page.tsx:66` |

#### 边界场景

| 场景 | 触发条件 | 输出结果 | 代码位置 |
|------|----------|----------|----------|
| 隐藏事件 | `eventData.hidden === true` | `robots: { index: false, follow: false }` | `page.tsx:65-68` |
| SEO 禁用 | `isSEOIndexable === false` | `robots: { index: false, follow: false }` | `page.tsx:65-68` |
| 改期场景 | `rescheduleUid` 存在 | 标题前缀加 "Reschedule" | `page.tsx:56-57` |
| 品牌隐藏 | `isBrandingHidden === true` | 标题不加 `\| ${APP_NAME}` | `_utils.tsx:35` |

---

### 2.4 链路 3：客户端数据获取与表单渲染

#### 输入（服务端传递的 props）

| 输入项 | 类型 | 代码位置 |
|--------|------|----------|
| `eventData` | `PublicEventType` | `users-type-public-view.tsx:27` |
| `user` | `string` | `users-type-public-view.tsx:27` |
| `slug` | `string` | `users-type-public-view.tsx:27` |
| `booking` | `GetBookingType \| undefined` | `users-type-public-view.tsx:27` |
| `isBrandingHidden` | `boolean` | `users-type-public-view.tsx:27` |
| `orgBannerUrl` | `string \| null` | `users-type-public-view.tsx:27` |

#### 处理流程

**Step 1：服务端数据直接使用（跳过客户端获取）**
```typescript
// BookerWebWrapper.tsx:39-50
const clientFetchedEvent = useEvent({
  disabled: !!props.eventData,  // 有服务端数据时禁用
  fromRedirectOfNonOrgLink: props.entity.fromRedirectOfNonOrgLink,
});

const event = props.eventData
  ? {
      data: props.eventData,
      isSuccess: true,
      isError: false,
      isPending: false,
    }
  : clientFetchedEvent;  // fallback 到客户端获取
```

**Step 2：useEvent Hook（备用路径，当无服务端数据时使用）**
```typescript
// useEvent.ts:21-46
export const useEvent = (props?: { fromRedirectOfNonOrgLink?: boolean; disabled?: boolean }) => {
  const [username, eventSlug, isTeamEvent, org] = useBookerStoreContext(...);

  const event = trpc.viewer.public.event.useQuery(
    {
      username: username ?? "",
      eventSlug: eventSlug ?? "",
      isTeamEvent,
      org: org ?? null,
      fromRedirectOfNonOrgLink: props?.fromRedirectOfNonOrgLink,
    },
    {
      refetchOnWindowFocus: false,
      enabled: !props?.disabled && Boolean(username) && Boolean(eventSlug),
    }
  );
  // ...
};
```

**Step 3：表单初始化**
```typescript
// BookerWebWrapper.tsx:114-122
const bookerForm = useBookingForm({
  event: event.data,
  sessionEmail: session?.user.email,
  sessionUsername: session?.user.username,
  sessionName: session?.user.name,
  hasSession,
  extraOptions: routerQuery,
  prefillFormParams,
});
```

**Step 4：可用时间段获取（客户端 tRPC）**
```typescript
// BookerWebWrapper.tsx:139-153
const schedule = useScheduleForEvent({
  eventId: props.entity.eventTypeId ?? event.data?.id,
  username: props.username,
  // ...
});
```

**Step 5：Booker 状态机**
```typescript
// Booker.tsx:221-229
useEffect(() => {
  if (event.isPending) return setBookerState("loading");
  if (!selectedDate) return setBookerState("selecting_date");
  if (!selectedTimeslot) return setBookerState("selecting_time");
  const isSkipConfirmStepSupported = layout !== BookerLayouts.WEEK_VIEW;
  if (selectedTimeslot && skipConfirmStep && isSkipConfirmStepSupported)
    return setBookerState("selecting_time");
  return setBookerState("booking");
}, [...]);
```

**Step 6：表单渲染（仅当 bookerState === "booking"）**
```typescript
// Booker.tsx:250-293
const EventBooker = useMemo(() => {
  if (bookerState !== "booking") {
    return null;
  }

  return (
    <BookEventForm
      key={key}
      timeslot={selectedTimeslot}
      bookingForm={bookingForm}
      eventQuery={event}
      // ...
    />
  );
}, [...]);
```

#### 输出

| 输出项 | 类型 | 说明 |
|--------|------|------|
| `bookerState` | `'loading' \| 'selecting_date' \| 'selecting_time' \| 'booking'` | 当前状态 |
| `bookingForm` | `UseFormReturn` | react-hook-form 实例 |
| `formEmail` | `string` | 表单中的邮箱值 |
| `formName` | `string` | 表单中的姓名值 |
| `schedule` | `{ data, isPending, invalidate, ... }` | 可用时间表数据 |

#### 边界场景

| 场景 | 触发条件 | 处理方式 | 代码位置 |
|------|----------|----------|----------|
| 服务端无数据 | `!props.eventData` | 使用 `useEvent` 客户端获取 | `BookerWebWrapper.tsx:39-50` |
| 无选中日期 | `!selectedDate` | 状态设为 `selecting_date` | `Booker.tsx:223` |
| 无选中时间段 | `!selectedTimeslot` | 状态设为 `selecting_time` | `Booker.tsx:224` |
| 时间段不可用 | `unavailableTimeSlots.includes(timeslot)` | 显示警告，禁用提交 | `BookEventForm.tsx:153-174` |

---

## 三、缓存策略可对账链路

### 3.1 缓存使用情况总览

| 缓存类型 | 适用范围 | 是否使用 | 代码位置 |
|----------|----------|----------|----------|
| **服务端 Data Cache** (`unstable_cache`) | `getPublicEvent` | ❌ 未使用 | 见下方验证 |
| **客户端 React Query 缓存** | tRPC queries | ✅ 使用 | `trpc.ts:107` |
| **HTTP 响应头缓存** | 静态资源 | ✅ 使用 | `next.config.ts:448-456` |
| **HTTP 响应头缓存** | API/页面 | ❌ 未配置 | 无相关配置 |

### 3.2 验证：getPublicEvent 未使用缓存

**证据 1：EventRepository 直接调用**
```typescript
// EventRepository.ts:12-24
export class EventRepository {
  static async getPublicEvent(input: GetPublicEventInput, userId?: number) {
    // 无 unstable_cache 包装，直接调用
    const event = await getPublicEvent(...);
    return event;
  }
}
```

**证据 2：getPublicEvent 直接查库**
```typescript
// getPublicEvent.ts:430-436
let event = await prisma.eventType.findFirst({
  where: {
    slug: eventSlug,
    ...usersOrTeamQuery,
  },
  select: getPublicEventSelect(fetchAllUsers),
});
// 无缓存逻辑
```

**证据 3：getServerSideProps 无缓存配置**
```typescript
// getServerSideProps.ts:245-253
const eventData = await EventRepository.getPublicEvent(...);
// 每次请求都执行，无缓存包装
```

**证据 4：page.tsx 无缓存配置**
```typescript
// 检查 [user]/[type]/page.tsx
// 无 export const revalidate
// 无 export const dynamic
// 无 export const fetchCache
```

**对比：其他模块使用了 unstable_cache**
```typescript
// travelSchedule.ts:9-22（作为对比，这个模块使用了缓存）
export const getTravelSchedule = unstable_cache(
  async (userId: number) => {
    return await TravelScheduleRepository.findTravelSchedulesByUserId(userId);
  },
  ["getTravelSchedule"],
  {
    revalidate: NEXTJS_CACHE_TTL,  // 3600 秒
    tags: [CACHE_TAGS.TRAVEL_SCHEDULES],
  }
);
```

### 3.3 客户端 React Query 缓存

#### 配置

```typescript
// trpc.ts:100-122
queryClientConfig: {
  defaultOptions: {
    queries: {
      staleTime: 1000,  // 1 秒后变旧
      retry(failureCount, _err) {
        // 重试逻辑
      },
    },
  },
}
```

#### useEvent 覆盖配置

```typescript
// useEvent.ts:27-38
const event = trpc.viewer.public.event.useQuery(
  { ... },
  {
    refetchOnWindowFocus: false,  // 窗口聚焦时不重新获取
    enabled: !props?.disabled && Boolean(username) && Boolean(eventSlug),
  }
);
```

#### 缓存行为

| 配置项 | 值 | 行为 |
|--------|-----|------|
| `staleTime` | `1000` ms | 查询结果在 1 秒内视为新鲜 |
| `refetchOnWindowFocus` | `false` | 窗口切换回来不自动刷新 |
| `enabled` | 条件 | 有 `username` 和 `eventSlug` 时才执行 |

### 3.4 HTTP 静态资源缓存

```typescript
// next.config.ts:448-456
{
  source: "/icons/sprite.svg(\\?v=[0-9a-zA-Z\\-\\.]+)?",
  headers: [
    {
      key: "Cache-Control",
      value: "public, max-age=31536000, immutable",  // 1 年，不可变
    },
  ],
}
```

**注意**：此配置仅适用于 `/icons/sprite.svg`，不适用于页面或 API 路由。

---

## 四、三者协作关系（基于代码证据）

### 4.1 数据流图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           首次请求                                        │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
┌───────────────┐         ┌───────────────┐         ┌───────────────┐
│ generateMeta- │         │  ServerPage   │         │               │
│    data()     │         │   (页面渲染)   │         │               │
└───────┬───────┘         └───────┬───────┘         │               │
        │                         │                   │               │
        ▼                         ▼                   │               │
┌─────────────────────────────────────────────┐       │               │
│         getData(legacyCtx)                   │       │               │
│  └── withAppDirSsr(getServerSideProps)      │       │               │
│      └── EventRepository.getPublicEvent()    │       │               │
│          └── getPublicEvent()                │       │               │
│              └── prisma.eventType.findFirst()│      │               │
│                  (每次都查数据库，无缓存)     │       │               │
└─────────────────────────────────────────────┘       │               │
        │                         │                   │               │
        ▼                         ▼                   │               │
┌───────────────┐         ┌───────────────┐         │               │
│ 返回 Metadata │         │ 返回 Page     │         │               │
│ (含 robots)   │         │ (含 eventData)│         │               │
└───────────────┘         └───────┬───────┘         │               │
                                    │                   │               │
                                    ▼                   │               │
                            ┌───────────────┐         │               │
                            │  客户端 HTML  │         │               │
                            │  (含 eventData│         │               │
                            │   序列化数据)  │         │               │
                            └───────┬───────┘         │               │
                                    │                   │               │
                                    ▼                   ▼               │
                            ┌─────────────────────────────────────┐    │
                            │         客户端 Hydration             │    │
                            │                                     │    │
                            │  BookerWebWrapper:                  │    │
                            │  ├── 使用 props.eventData           │    │
                            │  │   (跳过 useEvent 客户端获取)     │    │
                            │  ├── useBookingForm() 初始化表单   │    │
                            │  └── useScheduleForEvent()         │    │
                            │      └── tRPC query (staleTime:   │    │
                            │          1000ms 客户端缓存)        │    │
                            └─────────────────────────────────────┘    │
                                                                         │
┌─────────────────────────────────────────────────────────────────────────┤
│                           后续请求（1 秒内）                              │
└─────────────────────────────────────────────────────────────────────────┤
                                    │
                                    ▼
                            ┌───────────────┐
                            │  服务端处理    │
                            │  (每次都查库)  │
                            │  (无服务端缓存)│
                            └───────┬───────┘
                                    │
                                    ▼
                            ┌───────────────┐
                            │  客户端 Hydra- │
                            │    tion       │
                            │               │
                            │ useSchedule-  │
                            │ ForEvent():   │
                            │ 若 1 秒内    │
                            │ 使用 React   │
                            │ Query 缓存    │
                            └───────────────┘
```

### 4.2 关键协作点

#### 协作点 1：服务端数据共享

**代码证据**：
```typescript
// page.tsx:15-16
const getData: (ctx: ReturnType<typeof buildLegacyCtx>) => Promise<LegacyPageProps> =
  withAppDirSsr<LegacyPageProps>(getServerSideProps);

// page.tsx:18-20 (ServerPage)
const legacyCtx = buildLegacyCtx(await headers(), await cookies(), await params, await searchParams);
const props = await getData(legacyCtx);

// page.tsx:37-38 (generateMetadata)
const legacyCtx = buildLegacyCtx(await headers(), await cookies(), await params, await searchParams);
const props = await getData(legacyCtx);
```

**事实**：
- `generateMetadata` 和 `ServerPage` 都调用 `getData(legacyCtx)`
- 两者都执行相同的 `getServerSideProps` → `getPublicEvent` → 数据库查询
- **无显式缓存或数据共享代码**，依赖 Next.js 内部行为

#### 协作点 2：服务端 → 客户端数据传递

**代码证据**：
```typescript
// users-type-public-view.tsx:27-33
function Type({ slug, user, isEmbed, booking, isBrandingHidden, eventData, orgBannerUrl }: PageProps) {
  return (
    <Booker
      username={user}
      eventSlug={slug}
      bookingData={booking}
      hideBranding={isBrandingHidden}
      eventData={eventData}  // 传递给 Booker
      // ...
    />
  );
}
```

```typescript
// BookerWebWrapper.tsx:39-50
const clientFetchedEvent = useEvent({
  disabled: !!props.eventData,  // 有服务端数据时禁用
  // ...
});

const event = props.eventData
  ? { data: props.eventData, isSuccess: true, isError: false, isPending: false }
  : clientFetchedEvent;
```

**事实**：
- 服务端获取的 `eventData` 通过 props 传递到客户端
- 客户端 `useEvent` hook 检测到 `props.eventData` 存在时，`disabled: true`，跳过客户端获取
- 这确保了**服务端和客户端使用相同的 eventData**，避免 hydration mismatch

#### 协作点 3：SEO 元数据与表单数据的关联

**代码证据**：
```typescript
// page.tsx:40-52 (generateMetadata)
const { booking, isSEOIndexable = true, eventData, isBrandingHidden } = props;

const meeting = {
  title: eventData?.title ?? "",
  profile: { 
    name: eventData?.profile?.name ?? "", 
    image: eventData?.profile.image 
  },
  users: eventData?.subsetOfUsers.map(...) || [],
};

// robots 控制
return {
  ...metadata,
  robots: {
    follow: !(eventData?.hidden || !isSEOIndexable),
    index: !(eventData?.hidden || !isSEOIndexable),
  },
};
```

**关联字段**：

| 元数据字段 | 表单相关数据 | 说明 |
|------------|--------------|------|
| `title` | `eventData.title` | 表单页面标题 |
| `openGraph.images` | `eventData.profile.image`, `eventData.title` | 社交分享图片 |
| `robots.index` | `eventData.hidden`, `isSEOIndexable` | 控制搜索引擎索引 |
| `canonical` | URL 路径 | 规范链接 |

---

## 五、边界场景汇总

### 5.1 服务端边界场景

| 场景 | 触发条件 | 行为 | 代码位置 |
|------|----------|------|----------|
| 用户不存在 | `getUsersInOrgContext()` 返回空 | `notFound: true` | `getServerSideProps.ts:235-239` |
| 事件不存在 | `EventRepository.getPublicEvent()` 返回 null | `notFound: true` | `getServerSideProps.ts:255-259` |
| 事件禁止改期 | `booking.eventType.disableRescheduling === true` | 重定向到 `/booking/[uid]` | `getServerSideProps.ts:46-52` |
| 已取消预订改期 | `booking.status === CANCELLED` | 需 `allowRescheduleForCancelledBooking=true` | `getServerSideProps.ts:59-63` |

### 5.2 SEO 边界场景

| 场景 | 触发条件 | `robots.index` | `robots.follow` |
|------|----------|-----------------|-----------------|
| 正常事件 | `eventData.hidden = false`, `isSEOIndexable = true` | `true` | `true` |
| 隐藏事件 | `eventData.hidden = true` | `false` | `false` |
| SEO 禁用 | `isSEOIndexable = false` | `false` | `false` |

### 5.3 客户端边界场景

| 场景 | 触发条件 | 行为 |
|------|----------|------|
| 无服务端数据 | `!props.eventData` | 使用 `useEvent` 客户端获取 |
| 无选中日期 | `!selectedDate` | `bookerState = 'selecting_date'` |
| 无选中时间段 | `!selectedTimeslot` | `bookerState = 'selecting_time'` |
| 时间段不可用 | `unavailableTimeSlots.includes(timeslot)` | 显示警告，禁用提交按钮 |

---

## 六、关键结论（基于代码证据）

### 6.1 已证实的结论

1. **服务端数据获取无缓存**
   - `getPublicEvent` 每次都直接查询数据库
   - 无 `unstable_cache` 包装
   - 无 `revalidate` 或 `dynamic` 配置

2. **客户端 tRPC 有短缓存**
   - `staleTime: 1000` ms（1 秒）
   - `refetchOnWindowFocus: false`

3. **服务端与客户端数据一致**
   - 服务端 `eventData` 通过 props 传递
   - 客户端 `useEvent` 检测到 `props.eventData` 时禁用自身
   - 避免 hydration mismatch

4. **元数据生成与页面渲染共享数据源**
   - 两者都调用 `getData(legacyCtx)`
   - 但无显式缓存代码，依赖 Next.js 内部行为

### 6.2 已删除的推断（无代码证据）

以下结论**不在本文档中**，因为缺乏直接代码证据：

1. ❌ "Next.js 会自动缓存相同参数的 `getData` 调用"
   - 无显式缓存配置
   - `getServerSideProps` 每次都执行数据库查询

2. ❌ "公开预约页使用了 Next.js Data Cache"
   - 只有 `travelSchedule.ts` 和 `membership.ts` 使用了 `unstable_cache`
   - `getPublicEvent` 无缓存包装

3. ❌ "表单值通过 BookerStore 持久化"
   - 表单值同步到 store 的代码存在（`BookEventForm.tsx:116-122`）
   - 但"持久化"暗示跨页面或刷新后保留，此行为需验证

4. ❌ "多层次缓存策略"
   - 实际上只有：
     - 客户端 React Query 缓存（1 秒）
     - 静态资源 HTTP 缓存（1 年）
   - 服务端无缓存

---

## 七、代码引用速查

### 7.1 渲染流程

| 功能 | 文件 | 行号 |
|------|------|------|
| getServerSideProps 入口 | `getServerSideProps.ts` | 314-318 |
| getUserPageProps 主逻辑 | `getServerSideProps.ts` | 212-305 |
| EventRepository.getPublicEvent | `EventRepository.ts` | 12-24 |
| getPublicEvent 数据库查询 | `getPublicEvent.ts` | 430-436 |
| BookerWebWrapper 事件数据选择 | `BookerWebWrapper.tsx` | 39-50 |
| useEvent hook | `useEvent.ts` | 21-46 |
| Booker 状态机 | `Booker.tsx` | 221-229 |

### 7.2 元数据生成

| 功能 | 文件 | 行号 |
|------|------|------|
| generateMetadata 入口 | `page.tsx` | 36-70 |
| generateMeetingMetadata | `_utils.tsx` | 125-149 |
| _generateMetadataWithoutImage | `_utils.tsx` | 21-51 |
| robots 指令覆盖 | `page.tsx` | 63-69 |
| isSEOIndexable 判断 | `getServerSideProps.ts` | 261-265 |

### 7.3 缓存策略

| 功能 | 文件 | 行号 |
|------|------|------|
| tRPC 默认 staleTime | `trpc.ts` | 107 |
| useEvent refetchOnWindowFocus | `useEvent.ts` | 36 |
| 静态资源 Cache-Control | `next.config.ts` | 448-456 |
| travelSchedule 缓存示例（对比） | `travelSchedule.ts` | 9-22 |
