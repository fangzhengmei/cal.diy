# 公开预约页缓存命中与失效分析

本文档基于代码证据，梳理公开预约页（`/[user]/[type]`）在三类时序下的缓存行为，明确输入输出、命中条件、失效条件和回退路径。

---

## 一、代码位置速查

| 功能 | 文件路径 | 行号 |
|------|----------|------|
| tRPC 默认 staleTime | `packages/trpc/react/trpc.ts` | 107 |
| useSchedule 配置 | `apps/web/modules/schedules/hooks/useSchedule.ts` | 116-136 |
| 缓存常量定义 | `packages/lib/constants.ts` | 84-93, 243 |
| 缓存失效触发 | `apps/web/modules/bookings/components/Booker.tsx` | 260-266 |
| invalidate 实现 | `apps/web/modules/schedules/hooks/useSchedule.ts` | 191-193 |
| EventRepository 直接调用 | `packages/features/eventtypes/repositories/EventRepository.ts` | 12-24 |
| getPublicEvent 数据库查询 | `packages/features/eventtypes/lib/getPublicEvent.ts` | 430-436 |

---

## 二、缓存策略基础配置

### 2.1 服务端缓存配置

#### 已验证：服务端无缓存

**代码证据 1：EventRepository 直接调用**
```typescript
// EventRepository.ts:12-24
export class EventRepository {
  static async getPublicEvent(input: GetPublicEventInput, userId?: number) {
    // 无 unstable_cache 包装，直接调用
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

**代码证据 2：getPublicEvent 直接查库**
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

**代码证据 3：无 Next.js 缓存配置**
```typescript
// [user]/[type]/page.tsx 中无以下配置：
// - 无 export const revalidate
// - 无 export const dynamic
// - 无 export const fetchCache
```

**反例：其他模块使用了缓存**
```typescript
// travelSchedule.ts:9-22（作为对比，这个模块使用了缓存）
export const getTravelSchedule = unstable_cache(
  async (userId: number) => {
    return await TravelScheduleRepository.findTravelSchedulesByUserId(userId);
  },
  ["getTravelSchedule"],
  {
    revalidate: NEXTJS_CACHE_TTL,  // 3600 秒 = 1 小时
    tags: [CACHE_TAGS.TRAVEL_SCHEDULES],
  }
);
```

**结论**：
- ✅ 服务端 `getPublicEvent` 无缓存
- ✅ 每次请求都直接查询数据库
- ❌ 无 `unstable_cache` 包装
- ❌ 无 `revalidate` / `dynamic` / `fetchCache` 配置

### 2.2 客户端缓存配置

#### tRPC 默认配置

**代码证据**：
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

**说明**：
- `staleTime: 1000`：查询结果在 1 秒内视为新鲜
- 超过 1 秒后，数据变旧，但仍可使用（stale-while-revalidate）

#### useSchedule（时间段查询）配置

**代码证据**：
```typescript
// useSchedule.ts:116-136
const options = {
  trpc: {
    context: {
      skipBatch: true,
    },
  },
  // 窗口聚焦时重新获取
  refetchOnWindowFocus: true,
  // 5 分钟轮询一次
  refetchInterval: PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS * 1000,
  enabled:
    !skipGetSchedule &&
    Boolean(username) &&
    Boolean(month) &&
    Boolean(timezone) &&
    (Boolean(eventSlug) || Boolean(eventId) || eventId === 0) &&
    enabledProp,
};
```

#### 缓存常量定义

**代码证据**：
```typescript
// constants.ts:84-93
// 预留相关（时间段锁定）
export const PUBLIC_QUERY_RESERVATION_INTERVAL_SECONDS =
  parseInt(process.env.NEXT_PUBLIC_QUERY_RESERVATION_INTERVAL_SECONDS ?? "", 10) || 30;

export const PUBLIC_QUERY_RESERVATION_STALE_TIME_SECONDS =
  parseInt(process.env.NEXT_PUBLIC_QUERY_RESERVATION_STALE_TIME_SECONDS ?? "", 10) || 20;

// 可用时间段轮询间隔
export const PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS =
  parseInt(process.env.NEXT_PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS ?? "", 10) || 5 * 60;

// 缓存失效开关
export const PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM =
  process.env.NEXT_PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM === "1";

// constants.ts:243
export const NEXTJS_CACHE_TTL = 3600; // 1 hour（用于其他模块，非公开预约页）
```

**常量说明**：
| 常量 | 默认值 | 说明 |
|------|--------|------|
| `PUBLIC_QUERY_RESERVATION_INTERVAL_SECONDS` | 30 秒 | 时间段预留有效时长 |
| `PUBLIC_QUERY_RESERVATION_STALE_TIME_SECONDS` | 20 秒 | 预留查询缓存时间 |
| `PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS` | 5 分钟 | 可用时间段轮询间隔 |
| `PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM` | `false` | 预订表单取消时是否失效缓存 |
| `NEXTJS_CACHE_TTL` | 3600 秒 | 服务端 Data Cache TTL（用于其他模块） |

#### useEvent 配置

**代码证据**：
```typescript
// useEvent.ts:27-38
const event = trpc.viewer.public.event.useQuery(
  {
    username: username ?? "",
    eventSlug: eventSlug ?? "",
    isTeamEvent,
    org: org ?? null,
    fromRedirectOfNonOrgLink: props?.fromRedirectOfNonOrgLink,
  },
  {
    refetchOnWindowFocus: false,  // 窗口聚焦时不重新获取
    enabled: !props?.disabled && Boolean(username) && Boolean(eventSlug),
  }
);
```

**关键点**：
- `refetchOnWindowFocus: false`：窗口切换回来不自动刷新
- 当 `props.eventData` 存在时，`disabled: true`，`useEvent` 不会执行

---

## 三、三类时序分析

### 3.1 时序 1：首次请求

#### 输入

| 输入项 | 来源 | 说明 |
|--------|------|------|
| `params.user` | URL 路径 | 用户名（如 `alice`） |
| `params.type` | URL 路径 | 事件类型 slug（如 `30min`） |
| `searchParams.rescheduleUid` | 查询参数 | 改期预订 UID（可选） |
| `searchParams.bookingUid` | 查询参数 | 座位预订 UID（可选） |
| `searchParams.duration` | 查询参数 | 时长（可选，如 `60`） |

#### 服务端处理流程

**Step 1：generateMetadata 执行**

```
输入: params, searchParams, headers(), cookies()
     ↓
调用 getData(legacyCtx)
     ↓
withAppDirSsr(getServerSideProps)
     ↓
getUserPageProps / getDynamicGroupPageProps
     ↓
EventRepository.getPublicEvent(input, session?.user?.id)
     ↓
getPublicEvent(username, eventSlug, ...)
     ↓
prisma.eventType.findFirst(...)  ← 直接查询数据库
     ↓
返回 eventData
```

**Step 2：ServerPage 执行**

```
输入: params, searchParams, headers(), cookies()
     ↓
调用 getData(legacyCtx)  ← 同样的调用链
     ↓
prisma.eventType.findFirst(...)  ← 再次查询数据库
     ↓
返回 eventData
```

#### 客户端处理流程

**Step 1：接收服务端 props**

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

**Step 2：useEvent 跳过执行**

```typescript
// BookerWebWrapper.tsx:39-50
const clientFetchedEvent = useEvent({
  disabled: !!props.eventData,  // 有服务端数据时，disabled = true
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

**Step 3：useSchedule 首次执行**

```typescript
// useSchedule.ts:150-154
const schedule = trpc.viewer.slots.getSchedule.useQuery(input, {
  ...options,
  enabled: options.enabled && !isCallingApiV2Slots,
});
```

#### 输出

| 输出项 | 数据来源 | 缓存状态 |
|--------|----------|----------|
| `eventData` | 服务端数据库查询 | 无缓存，每次都查 |
| `schedule` | 客户端 tRPC 查询 | 首次请求，无缓存 |
| `bookerState` | 客户端状态机 | `loading` → `selecting_date` |

#### 边界场景

| 场景 | 触发条件 | 行为 |
|------|----------|------|
| 用户不存在 | `getUsersInOrgContext()` 返回空 | 服务端返回 `notFound: true` |
| 事件不存在 | `EventRepository.getPublicEvent()` 返回 null | 服务端返回 `notFound: true` |
| 事件禁止改期 | `booking.eventType.disableRescheduling === true` | 服务端重定向到 `/booking/[uid]` |

#### 代码证据链

| 环节 | 代码位置 | 证据 |
|------|----------|------|
| 服务端查库 | `getPublicEvent.ts:430-436` | `prisma.eventType.findFirst(...)` |
| 无缓存包装 | `EventRepository.ts:12-24` | 无 `unstable_cache` |
| useEvent 跳过 | `BookerWebWrapper.tsx:39-50` | `disabled: !!props.eventData` |
| useSchedule 执行 | `useSchedule.ts:150-154` | `trpc.viewer.slots.getSchedule.useQuery` |

---

### 3.2 时序 2：短时间重复访问

#### 场景定义

本时序包含两种情况：
1. **场景 A**：页面内操作（不刷新页面，如切换月份、选择不同日期）
2. **场景 B**：刷新页面（新的 HTTP 请求）

#### 场景 A：页面内操作（1 秒内）

**输入**：
- 同一个浏览器 tab
- 用户操作：切换月份、选择日期等
- 时间距离上次请求 < 1 秒

**缓存命中条件**：
```
条件 1: 距离上次 useSchedule 查询 < 1 秒 (staleTime: 1000)
     AND
条件 2: 查询参数相同 (username, eventSlug, month, startTime, endTime, timezone, etc.)
     AND
条件 3: 未到 refetchInterval (5 分钟)
     AND
条件 4: 未触发 refetchOnWindowFocus (窗口未失焦)
```

**处理流程**：

```
用户操作（切换月份）
     ↓
useSchedule 查询参数变化（month 变了）
     ↓
新的 query key，无缓存，重新查询
     ↓
或者：

用户操作（选择同一天的不同时间段）
     ↓
useSchedule 查询参数相同（startTime/endTime 未变）
     ↓
距离上次查询 < 1 秒
     ↓
✅ 命中 React Query 缓存
     ↓
直接返回缓存数据
```

**代码证据**：
```typescript
// trpc.ts:107
staleTime: 1000,  // 1 秒内视为新鲜

// useSchedule.ts:127
refetchInterval: PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS * 1000,  // 5 分钟
```

**输出**：
- **命中时返回缓存数据，未命中时重新查询

#### 场景 B：刷新页面（新的 HTTP 请求）

**输入**：
- 用户点击浏览器刷新按钮
- 或在地址栏按 Enter
- 新的 HTTP 请求

**缓存命中条件**：
```
无服务端缓存命中条件：❌ 无条件命中（服务端无缓存）

客户端缓存命中条件：❌ 无条件命中（页面刷新，React Query 缓存清空）
```

**处理流程**：

```
新的 HTTP 请求到达服务端
     ↓
Next.js 重新执行 generateMetadata
     ↓
getData(legacyCtx) → getServerSideProps → 数据库查询  ← 无缓存
     ↓
Next.js 重新执行 ServerPage
     ↓
getData(legacyCtx) → getServerSideProps → 数据库查询  ← 无缓存
     ↓
返回新的 HTML 到客户端
     ↓
客户端重新初始化
     ↓
React Query 缓存被清空  ← 页面刷新
     ↓
useSchedule 重新查询  ← 无缓存
```

**代码证据**：
```typescript
// 服务端无缓存配置的证据：
// - 无 export const revalidate
// - 无 export const dynamic
// - 无 unstable_cache 包装

// 客户端页面刷新后，React Query 缓存清空是浏览器默认行为
```

#### 两类场景对比

| 维度 | 场景 A：页面内操作 | 场景 B：刷新页面 |
|------|-------------------|------------------|
| 服务端请求 | ❌ 无 | ❌ 无 |
| 客户端缓存 | ✅ 1 秒内可能命中 | ❌ 缓存清空 |
| 数据库查询 | 仅查询参数变化时 | 每次都查询 |
| 代码证据 | `trpc.ts:107` | 页面刷新行为 |

#### 边界场景

| 场景 | 触发条件 | 行为 |
|------|----------|------|
| 超过 1 秒 | 距离上次查询 > 1 秒 | 数据变旧，可能触发重新验证 |
| 超过 5 分钟 | 距离上次查询 > 5 分钟 | `refetchInterval` 触发重新查询 |
| 窗口失焦后返回 | 用户切换 tab 后返回 | `refetchOnWindowFocus: true` 触发重新查询 |

---

### 3.3 时序 3：用户操作触发刷新

#### 场景定义

用户在预订表单中点击"取消"按钮，返回时间段选择页面。

#### 输入

| 输入项 | 来源 | 说明 |
|--------|------|------|
| `selectedTimeslot` | 客户端状态 | 用户选中的时间段 |
| `PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM` | 环境变量 | 缓存失效开关 |

#### 处理流程

**Step 1：用户点击"取消"按钮"

**代码证据**：
```typescript
// Booker.tsx:260-274
onCancel={() => {
  setSelectedTimeslot(null);
  // Temporarily allow disabling it, till we are sure that it doesn't cause any significant load on the system
  if (PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM) {
    // Ensures that user has latest available slots when they want to re-choose from the slots
    schedule?.invalidate();
  }
  if (seatedEventData.bookingUid) {
    setSeatedEventData({
      ...seatedEventData,
      bookingUid: undefined,
      attendees: undefined,
    });
  }
}}
```

**Step 2：invalidate 实现

**代码证据**：
```typescript
// useSchedule.ts:186-194
return {
  ...schedule,
  /**
   * Invalidates the request and resends it regardless of any other configuration including staleTime
   */
  invalidate: () => {
    return utils.viewer.slots.getSchedule.invalidate(input);
  },
};
```

#### 失效条件

```
条件 1: PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM === "1"
     AND
条件 2: schedule 存在 (schedule?.invalidate())
```

#### 回退路径

```
如果条件 1 不满足（环境变量未设置）：
     ↓
schedule?.invalidate() 不会被调用
     ↓
继续使用缓存数据（staleTime 内）
     ↓
或等待 5 分钟轮询更新
```

#### 输出

| 环境变量状态 | 行为 | 数据新鲜度 |
|-------------|------|-----------|
| `NEXT_PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM === "1" | 强制重新查询 | ✅ 新鲜 |
| 环境变量未设置 | 使用缓存 | ⚠️ 可能过时 |

#### 边界场景

| 场景 | 触发条件 | 行为 |
|------|----------|------|
| 环境变量开启 | `NEXT_PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM === "1" | `schedule.invalidate()` 被调用 |
| 环境变量关闭 | 环境变量未设置或为其他值 | `schedule.invalidate()` 不被调用 |
| 取消后立即选择 | 用户取消后立即选择新时间段 | 缓存数据可能已过时（如果环境变量关闭） |

#### 代码证据链

| 环节 | 代码位置 | 证据 |
|------|----------|------|
| 失效触发 | `Booker.tsx:260-266` | `if (PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM) { schedule?.invalidate(); }` |
| 环境变量定义 | `constants.ts:92-93` | `process.env.NEXT_PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM === "1"` |
| invalidate 实现 | `useSchedule.ts:191-193` | `utils.viewer.slots.getSchedule.invalidate(input)` |

---

## 四、缓存行为汇总

### 4.1 缓存命中/失效决策树

```
                    ┌─────────────────────────────────────────────────────────┐
│                    公开预约页缓存决策树                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
              ┌───────────────────────────────┐
              │  这是服务端请求吗？      │
              │  (首次请求/刷新页面)     │
              └───────────────────────────────┘
                    │                 │
                   是                 否
                    │                 │
                    ▼                 ▼
         ┌────────────────┐    ┌──────────────────────┐
         │ 服务端处理 │    │ 客户端页面内操作    │
         │            │    │                    │
         │ ❌ 无缓存  │    │ 检查 React Query  │
         │ 每次查库  │    │ 缓存状态            │
         └────────────────┘    └──────────────────────┘
                                       │
                                       ▼
                        ┌───────────────────────────────┐
                        │  距离上次查询 < 1 秒？     │
                        │  (staleTime: 1000ms)    │
                        └───────────────────────────────┘
                              │                 │
                             是                 否
                              │                 │
                              ▼                 ▼
                     ┌────────────┐    ┌──────────────────┐
                     │ ✅ 命中缓存 │    │ ⚠️ 数据变旧       │
                     │ 直接返回   │    │ 可能重新验证      │
                     └────────────┘    └──────────────────┘
                                       │
                                       ▼
                            ┌──────────────────────────┐
                            │ 用户点击"取消"按钮？      │
                            └──────────────────────────┘
                                  │           │
                                 是           否
                                  │           │
                                  ▼           ▼
                       ┌──────────────────┐    │
                       │ 检查环境变量    │    │
                       │ INVALIDATE_... │    │
                       │ ON_BOOKING_FORM │    │
                       └──────────────────┘    │
                            │         │          │
                           是         否         │
                            │         │          │
                            ▼         ▼          │
                     ┌──────────┐  ┌──────────┐   │
                     │ 强制   │  │ 使用   │   │
                     │ 失效   │  │ 缓存   │   │
                     └──────────┘  └──────────┘   │
```

### 4.2 三类时序对比表

| 维度 | 时序 1：首次请求 | 时序 2A：页面内操作（<1秒） | 时序 2B：刷新页面 | 时序 3：用户取消操作 |
|------|-----------------|------------------------------|------------------|---------------------|
| **服务端缓存 | ❌ 无 | ❌ 无（服务端不参与） | ❌ 无，每次查库 | ❌ 无（服务端不参与） |
| **客户端缓存** | ❌ 首次请求，无缓存 | ✅ 可能命中（<1秒） | ❌ 缓存清空 | ⚠️ 取决于环境变量 |
| **数据库查询** | 服务端 + 客户端 | 仅查询参数变化时 | 服务端 + 客户端 | 环境变量开启时：客户端 |
| **失效触发** | 无 | 无 | 页面刷新 | `schedule.invalidate()` |
| **回退路径** | 无 | 无 | 无 | 使用缓存数据 |

### 4.3 关键常量速查

| 常量 | 默认值 | 影响 |
|------|--------|------|
| `staleTime` | 1000 ms | 1 秒内视为新鲜 |
| `refetchInterval` | 5 分钟 | 定时轮询更新 |
| `refetchOnWindowFocus` | `true` (useSchedule) / `false` (useEvent) | 窗口聚焦行为 |
| `PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM` | `false` | 取消时是否失效缓存 |

---

## 五、已删除的推断

以下结论**不在本文档中**，因为缺乏直接代码证据：

1. ❌ "Next.js 会自动缓存相同参数的 `getData` 调用"
   - 代码中无显式缓存配置
   - `getServerSideProps` 每次都执行数据库查询
   - 无 `revalidate` / `dynamic` / `fetchCache` 配置

2. ❌ "公开预约页使用了 Next.js Data Cache"
   - 只有 `travelSchedule.ts` 和 `membership.ts` 使用了 `unstable_cache`
   - `getPublicEvent` 无缓存包装

3. ❌ "多层次缓存策略"
   - 实际上只有：
     - 客户端 React Query 缓存（1 秒）
     - 静态资源 HTTP 缓存（1 年，非页面）
   - 服务端无缓存

4. ❌ "表单值通过 BookerStore 持久化"
   - 表单值同步到 store 的代码存在
   - 但"持久化"暗示跨页面或刷新后保留，此行为未验证

---

## 六、附录：环境变量配置参考

| 环境变量 | 类型 | 默认值 | 说明 |
|----------|------|--------|------|
| `NEXT_PUBLIC_QUERY_RESERVATION_INTERVAL_SECONDS` | number | 30 | 时间段预留有效时长（秒） |
| `NEXT_PUBLIC_QUERY_RESERVATION_STALE_TIME_SECONDS` | number | 20 | 预留查询缓存时间（秒） |
| `NEXT_PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS` | number | 300 | 可用时间段轮询间隔（秒） |
| `NEXT_PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM` | string | 未设置 | 预订表单取消时是否失效缓存（"1" 表示开启） |

**配置示例**：
```bash
# 开启预订取消时缓存失效
NEXT_PUBLIC_INVALIDATE_AVAILABLE_SLOTS_ON_BOOKING_FORM=1

# 缩短轮询间隔为 1 分钟
NEXT_PUBLIC_QUERY_AVAILABLE_SLOTS_INTERVAL_SECONDS=60
```
