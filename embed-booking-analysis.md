# 外部网站嵌入预约组件协作机制分析

> **版本说明**：v2.0，修正了 API 认证边界和 postMessage 校验机制的关键理解

---

## 一、整体架构概览

外部网站嵌入 Cal.diy 预约组件时，涉及四个核心模块的协作：

1. **Embed 脚本** (`packages/embeds/embed-core/`) - 运行在外部网站（父页面）
2. **公开页面** (`apps/web/`) - 运行在 Cal.diy 域名下的 iframe 内容
3. **跨窗口消息通信** - postMessage 协议（⚠️ 存在安全风险）
4. **Booking API** - tRPC 端点处理预约业务

```
┌─────────────────────────────────────────────────────────────────────┐
│                     外部网站 (example.com)                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Embed 脚本 (embed.ts)                          │   │
│  │  • Cal API: inline(), modal(), floatingButton()            │   │
│  │  • doInIframe(): 发送命令到 iframe (targetOrigin="*")      │   │
│  │  • ActionManager: 监听 iframe 事件 (仅校验 CAL: 前缀)      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                 │                                     │
│                                 ▼ postMessage()                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              <iframe src="cal.com/xxx/embed">               │   │
│  │  ┌───────────────────────────────────────────────────────┐  │   │
│  │  │           公开页面 (Cal.diy 域名)                      │  │   │
│  │  │  • embed-iframe.ts: 消息监听 (仅校验 originator: "CAL")│  │   │
│  │  │  • Booker.tsx: 预约表单组件                            │  │   │
│  │  │  • tRPC Client: 调用 Booking API                       │  │   │
│  │  └───────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼ HTTP/HTTPS
┌─────────────────────────────────────────────────────────────────────┐
│                    Cal.diy 后端服务                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              tRPC 路由结构                                    │   │
│  │                                                               │   │
│  │  🔓 匿名可用 (publicProcedure)                               │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  slotsRouter (/api/trpc/slots/)              ⚠️ 关键 │   │   │
│  │  │  • getSchedule()     - 获取可用时间段               │   │   │
│  │  │  • reserveSlot()     - 预留时间段（设置 uid cookie）│   │   │
│  │  │  • isAvailable()     - 检查可用性                   │   │   │
│  │  ├─────────────────────────────────────────────────────┤   │   │
│  │  │  publicViewerRouter (/api/trpc/public/)             │   │   │
│  │  │  • event()           - 获取公开事件信息             │   │   │
│  │  │  • submitRating()    - 提交评价                     │   │   │
│  │  │  • countryCode()     - 国家代码                     │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  │                                                               │   │
│  │  🔒 需登录 (authedProcedure)                                 │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │  viewerRouter (/api/trpc/)                           │   │   │
│  │  │  • bookings.confirm() - 确认/拒绝预约               │   │   │
│  │  │  • bookings.find()   - 查找预约                     │   │   │
│  │  │  • eventTypes.*      - 事件类型管理                 │   │   │
│  │  │  • calendars.*       - 日历管理                     │   │   │
│  │  │  • me.*              - 当前用户信息                 │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、关键纠正点

### ⚠️ 纠正 1：slotsRouter 是独立的匿名路由

**之前的错误理解**：
- 认为 `reserveSlot` 在 `viewerRouter` 下，需要认证
- 混淆了 `slotsRouter` 和 `viewerRouter` 的关系

**实际情况**：
```
┌─────────────────────────────────────────────────────────────────┐
│                    tRPC API 端点结构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  独立路由（独立的 API 端点文件）                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  /api/trpc/slots/[trpc].ts                               │   │
│  │  → slotsRouter                                           │   │
│  │  → 100% 使用 publicProcedure                              │   │
│  │  → 完全匿名，无需任何认证                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  /api/trpc/public/[trpc].ts                              │   │
│  │  → publicViewerRouter                                    │   │
│  │  → 使用 publicProcedure                                  │   │
│  │  → 完全匿名                                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  主路由                                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  /api/trpc/[trpc].ts                                     │   │
│  │  → appRouter → viewerRouter                              │   │
│  │  → 99% 使用 authedProcedure                               │   │
│  │  → 需要登录认证                                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ⚠️ 关键结论：                                                    │
│  - getSchedule、reserveSlot、isAvailable 都是完全匿名的          │
│  - 它们在独立的 slotsRouter 中，不依赖 viewerRouter              │
│  - 整个预约流程（除了确认/拒绝）都可以匿名完成                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### ⚠️ 纠正 2：postMessage 安全校验严重不足

**之前的错误理解**：
- 认为通过 `originator: "CAL"` 验证足够安全
- 没有详细分析两侧的校验逻辑

**实际情况**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    postMessage 安全校验分析                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  父窗口 (embed.ts)                                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  发送消息 (doInIframe)                                        │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ this.iframe.contentWindow.postMessage(                │   │   │
│  │  │   { originator: "CAL", method: ..., arg: ... },      │   │   │
│  │  │   "*"  // ⚠️ targetOrigin = "*"，任何窗口都能接收    │   │   │
│  │  │ );                                                    │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │  ✅ 做了什么：添加 originator: "CAL" 标识                      │   │
│  │  ❌ 没做什么：                                                  │   │
│  │    - 没有验证 iframe.src 的域名                                │   │
│  │    - 使用 targetOrigin="*"，消息可能被任何 iframe 接收       │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  接收消息 (message listener)                                  │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ window.addEventListener("message", (e) => {            │   │   │
│  │  │   const fullType = e.data.fullType;                     │   │   │
│  │  │   const [cal, ns, type] = fullType.split(":");         │   │   │
│  │  │   if (cal !== "CAL") return;  // ✅ 检查前缀          │   │   │
│  │  │   // 执行对应 action...                                 │   │   │
│  │  │ });                                                    │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │  ✅ 做了什么：检查 fullType 是否以 "CAL:" 开头                 │   │
│  │  ❌ 没做什么：                                                  │   │
│  │    - 没有验证 e.origin（消息来源域名）                         │   │
│  │    - 没有验证 e.source（消息来源窗口）                         │   │
│  │    - 任何窗口都可以伪造 { fullType: "CAL::xxx", ... }       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Iframe (embed-iframe.ts)                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  发送消息 (messageParent)                                    │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ parent.postMessage(                                     │   │   │
│  │  │   { originator: "CAL", type: ..., data: ... },        │   │   │
│  │  │   "*"  // ⚠️ targetOrigin = "*"，任何父窗口都能接收   │   │   │
│  │  │ );                                                    │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │  ✅ 做了什么：添加 originator: "CAL" 标识                      │   │
│  │  ❌ 没做什么：                                                  │   │
│  │    - 没有验证 parent 是否是预期的外部网站域名                  │   │
│  │    - 使用 targetOrigin="*"，消息可能被任何父窗口接收         │   │
│  ├─────────────────────────────────────────────────────────────┤   │
│  │  接收消息 (message listener)                                  │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ window.addEventListener("message", (e) => {            │   │   │
│  │  │   const data = e.data;                                  │   │   │
│  │  │   if (data.originator === "CAL" &&                     │   │   │
│  │  │       typeof data.method === "string") {               │   │   │
│  │  │     interfaceWithParent[data.method]?.(data.arg);     │   │   │
│  │  │   }                                                     │   │   │
│  │  │ });                                                    │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │  ✅ 做了什么：检查 data.originator === "CAL"                  │   │
│  │  ❌ 没做什么：                                                  │   │
│  │    - 没有验证 e.origin（消息来源域名）                         │   │
│  │    - 没有验证 e.source（消息来源窗口）                         │   │
│  │    - 任何窗口都可以伪造 { originator: "CAL", method: ... }  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ⚠️ 安全风险总结：                                                  │
│  1. 两侧发送都使用 targetOrigin="*"                                 │
│  2. 两侧接收都只验证字符串标识（originator: "CAL" 或 CAL: 前缀）  │
│  3. 完全没有验证 e.origin 或 e.source                              │
│  4. 攻击者可以伪造消息格式，发送恶意命令                            │
│                                                                      │
│  🔒 缓解措施（依赖业务层）：                                         │
│  - 传递的都是 UI 配置、事件通知等非敏感数据                         │
│  - 关键操作（创建预约、确认预约）在服务端处理                       │
│  - uid cookie 有 SameSite=None 限制（仅 HTTPS）                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、API 认证边界详细分析

### 3.1 tRPC Procedure 分类

```typescript
// publicProcedure - 完全匿名，无需任何认证
// 文件: packages/trpc/server/procedures/publicProcedure.ts
const publicProcedure = tRPCContext.procedure
  .use(perfMiddleware)
  .use(errorConversionMiddleware);

// authedProcedure - 需要登录认证
// 文件: packages/trpc/server/procedures/authedProcedure.ts
const authedProcedure = procedure
  .use(perfMiddleware)
  .use(errorConversionMiddleware)
  .use(isAuthed);  // ⚠️ 关键：isAuthed middleware

// isAuthed middleware 实现
// 文件: packages/trpc/server/middlewares/sessionMiddleware.ts
export const isAuthed = middleware(async ({ ctx, next }) => {
  const { user, session } = await getUserSession(ctx);
  
  // ⚠️ 如果没有 user 或 session，抛出 UNAUTHORIZED
  if (!user || !session) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  
  return next({ ctx: { user, session } });
});
```

### 3.2 路由与 Procedure 映射表

| 路由 | API 端点 | 使用的 Procedure | 认证要求 |
|------|---------|-----------------|---------|
| **slotsRouter** | `/api/trpc/slots/` | `publicProcedure` | ❌ 无需 |
| **publicViewerRouter** | `/api/trpc/public/` | `publicProcedure` | ❌ 无需 |
| **viewerRouter** | `/api/trpc/` | `authedProcedure` | ✅ 需登录 |

### 3.3 slotsRouter 完整分析

**文件**: `packages/trpc/server/routers/viewer/slots/_router.tsx`

```typescript
// ⚠️ 注意：虽然路径在 viewer/slots/ 下，但使用的是 publicProcedure！
import publicProcedure from "../../../procedures/publicProcedure";
import { router } from "../../../trpc";

export const slotsRouter = router({
  // 获取可用时间段 - 完全匿名
  getSchedule: publicProcedure
    .input(ZGetScheduleInputSchema)
    .query(async ({ input, ctx }) => {
      const { getScheduleHandler } = await import("./getSchedule.handler");
      return getScheduleHandler({ ctx, input });
    }),
  
  // 预留时间段 - 完全匿名（设置 uid cookie）
  reserveSlot: publicProcedure
    .input(ZReserveSlotInputSchema)
    .mutation(async ({ input, ctx }) => {
      const { reserveSlotHandler } = await import("./reserveSlot.handler");
      return reserveSlotHandler({
        ctx: { ...ctx, req: ctx.req as NextApiRequest, res: ctx.res as NextApiResponse },
        input,
      });
    }),
  
  // 检查时间段是否可用 - 完全匿名
  isAvailable: publicProcedure
    .input(ZIsAvailableInputSchema)
    .output(ZIsAvailableOutputSchema)
    .query(async ({ input, ctx }) => {
      const { isAvailableHandler } = await import("./isAvailable.handler");
      return isAvailableHandler({
        ctx: { ...ctx, req: ctx.req as NextApiRequest },
        input,
      });
    }),
  
  // 移除预留标记 - 完全匿名
  removeSelectedSlotMark: publicProcedure
    .input(ZRemoveSelectedSlotInputSchema)
    .mutation(async ({ input, ctx }) => {
      const { req, prisma } = ctx;
      const uid = req?.cookies?.uid || input.uid;
      if (uid) {
        await prisma.selectedSlots.deleteMany({ where: { uid: { equals: uid } } });
      }
      return;
    }),
});
```

### 3.4 预约流程 API 调用链

```
┌─────────────────────────────────────────────────────────────────────┐
│                    完整预约流程 API 调用链                           │
│                    （所有步骤都可匿名完成！）                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  访客视角（无需登录）                                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  Step 1: 加载公开事件信息                                      │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 端点: publicViewer.event()                              │   │   │
│  │  │ 路径: /api/trpc/public/event.get                        │   │   │
│  │  │ Procedure: publicProcedure                              │   │   │
│  │  │ 认证: ❌ 无需                                           │   │   │
│  │  │ 输入: { username, eventTypeSlug }                      │   │   │
│  │  │ 输出: 事件详情（时长、描述、位置等）                     │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                           ↓                                    │   │
│  │  Step 2: 获取可用时间段                                       │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 端点: slotsRouter.getSchedule()                        │   │   │
│  │  │ 路径: /api/trpc/slots/getSchedule                      │   │   │
│  │  │ Procedure: publicProcedure                              │   │   │
│  │  │ 认证: ❌ 无需                                           │   │   │
│  │  │ 输入: { eventTypeId, startTime, endTime, timezone }    │   │   │
│  │  │ 输出: 可用时间段列表                                     │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                           ↓                                    │   │
│  │  Step 3: 选择时间段并预留                                     │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 端点: slotsRouter.reserveSlot()                        │   │   │
│  │  │ 路径: /api/trpc/slots/reserveSlot                      │   │   │
│  │  │ Procedure: publicProcedure                              │   │   │
│  │  │ 认证: ❌ 无需                                           │   │   │
│  │  │ 输入: { slotUtcStartDate, slotUtcEndDate, eventTypeId }│  │   │
│  │  │ 输出: { uid: string }                                   │   │   │
│  │  │                                                         │   │   │
│  │  │ ⚠️ 关键：设置 uid Cookie                                │   │   │
│  │  │ Set-Cookie: uid=xxx; SameSite=None; Secure            │   │   │
│  │  │ 这个 cookie 用于追踪访客，关联后续操作                   │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                           ↓                                    │   │
│  │  Step 4: 填写表单并创建预约                                   │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 端点: 内部服务调用（非 tRPC 端点）                      │   │   │
│  │  │ 调用: createBooking()                                  │   │   │
│  │  │ 位置: packages/features/bookings/lib/handleNewBooking/ │   │   │
│  │  │ 认证: ❌ 无需                                           │   │   │
│  │  │ 追踪: 使用 uid cookie 关联预留的时间段                   │   │   │
│  │  │                                                         │   │   │
│  │  │ 输入: {                                                 │   │   │
│  │  │   uid: string,          // 来自 cookie                  │   │   │
│  │  │   name: string,         // 访客姓名                     │   │   │
│  │  │   email: string,        // 访客邮箱                     │   │   │
│  │  │   notes?: string,       // 备注                         │   │   │
│  │  │   startTime: Date,      // 预约时间                     │   │   │
│  │  │   endTime: Date,        // 结束时间                     │   │   │
│  │  │   eventTypeId: number,  // 事件类型                     │   │   │
│  │  │ }                                                       │   │   │
│  │  │                                                         │   │   │
│  │  │ 输出: Booking 对象                                      │   │   │
│  │  │ - 状态: ACCEPTED 或 PENDING（取决于配置）               │   │   │
│  │  │ - uid: 关联的访客标识                                   │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  │  ✅ 至此，访客的预约流程已全部完成！                           │   │
│  │  ✅ 整个过程无需登录，仅通过 uid cookie 追踪                   │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  组织者视角（需要登录）                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  Step 5: 确认/拒绝预约（如果需要）                             │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 端点: viewerRouter.bookings.confirm()                  │   │   │
│  │  │ 路径: /api/trpc/bookings.confirm                        │   │   │
│  │  │ Procedure: authedProcedure                              │   │   │
│  │  │ 认证: ✅ 需登录                                          │   │   │
│  │  │ 权限检查: 验证用户是否有权限操作该预约                   │   │   │
│  │  │                                                         │   │   │
│  │  │ 输入: { bookingId, confirmed: boolean }                │   │   │
│  │  │ 输出: { message, status }                               │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  │  ⚠️ 注意：许多事件类型默认自动确认，无需组织者手动确认        │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.5 认证边界总结表

| 操作 | 执行方 | API 端点 | Procedure | 认证要求 | 追踪方式 |
|------|-------|---------|-----------|---------|---------|
| 查看公开事件 | 访客 | `publicViewer.event` | `publicProcedure` | ❌ 无需 | 无 |
| 查看可用时间 | 访客 | `slots.getSchedule` | `publicProcedure` | ❌ 无需 | 无 |
| 预留时间段 | 访客 | `slots.reserveSlot` | `publicProcedure` | ❌ 无需 | uid Cookie |
| 检查可用性 | 访客 | `slots.isAvailable` | `publicProcedure` | ❌ 无需 | 无 |
| 移除预留 | 访客 | `slots.removeSelectedSlotMark` | `publicProcedure` | ❌ 无需 | uid Cookie |
| 创建预约 | 访客 | 内部服务调用 | N/A | ❌ 无需 | uid Cookie |
| 确认预约 | 组织者 | `viewer.bookings.confirm` | `authedProcedure` | ✅ 需登录 | Session |
| 拒绝预约 | 组织者 | `viewer.bookings.confirm` | `authedProcedure` | ✅ 需登录 | Session |
| 取消预约 | 双方 | `viewer.bookings.*` | `authedProcedure` | ✅ 需登录 | Session |
| 改期预约 | 双方 | `viewer.bookings.requestReschedule` | `authedProcedure` | ✅ 需登录 | Session |

---

## 四、postMessage 安全机制深度分析

### 4.1 完整消息格式

#### 4.1.1 父窗口 → Iframe（命令）

```typescript
// 发送方: embed.ts doInIframe()
// 接收方: embed-iframe.ts message listener
{
  originator: "CAL",      // 标识字段
  method: string,         // 命令类型: "ui" | "parentKnowsIframeReady" | "connect"
  arg: unknown            // 命令参数
}
```

#### 4.1.2 Iframe → 父窗口（事件）

```typescript
// 发送方: embed-iframe.ts messageParent()
// 接收方: embed.ts message listener
{
  originator: "CAL",      // 标识字段
  type: string,           // 事件类型: "__iframeReady" | "linkReady" | "bookingSuccessful" 等
  namespace: string,      // 命名空间
  fullType: string,       // 完整类型: "CAL:{namespace}:{type}"
  data: unknown           // 事件数据
}
```

### 4.2 发送端实现与校验

#### 4.2.1 父窗口发送（embed.ts:397-413）

```typescript
doInIframe(doInIframeArg: DoInIframeArg) {
  if (!this.iframeReady) {
    this.iframeDoQueue.push(doInIframeArg);
    return;
  }
  if (!this.iframe) {
    throw new Error("iframe doesn't exist. `createIframe` must be called before `doInIframe`");
  }
  if (this.iframe.contentWindow) {
    // TODO: Ensure that targetOrigin is as defined by user(and not *). 
    // Generally it would be cal.com but in case of self hosting it can be anything.
    // Maybe we can derive targetOrigin from __config.origin
    
    // ⚠️ 问题 1: targetOrigin = "*"
    // 这意味着任何 iframe（不仅仅是我们创建的那个）都能接收这条消息
    this.iframe.contentWindow.postMessage(
      { 
        originator: "CAL", 
        method: doInIframeArg.method, 
        arg: doInIframeArg.arg 
      },
      "*"  // ❌ 没有指定目标源
    );
  }
}
```

#### 4.2.2 Iframe 发送（embed-iframe.ts:512-520）

```typescript
const messageParent = (data: CustomEvent["detail"]) => {
  // ⚠️ 问题 1: targetOrigin = "*"
  // 这意味着任何父窗口（不仅仅是加载我们的那个）都能接收这条消息
  parent.postMessage(
    {
      originator: "CAL",
      ...data,
    },
    "*"  // ❌ 没有指定目标源
  );
};
```

### 4.3 接收端实现与校验

#### 4.3.1 Iframe 接收（embed-iframe.ts:559-568）

```typescript
window.addEventListener("message", (e) => {
  const data: Message = e.data;
  if (!data) {
    return;
  }
  const method: keyof typeof interfaceWithParent = data.method;
  
  // ⚠️ 唯一的校验：检查 originator 字段
  // ❌ 没有验证 e.origin（消息来源域名）
  // ❌ 没有验证 e.source（消息来源窗口）
  if (data.originator === "CAL" && typeof method === "string") {
    interfaceWithParent[method]?.(data.arg as never);
  }
});
```

#### 4.3.2 父窗口接收（embed.ts:1565-1582）

```typescript
window.addEventListener("message", (e) => {
  const detail = e.data;
  const fullType = detail.fullType;
  
  // ⚠️ 解析 fullType，检查是否以 "CAL:" 开头
  const parsedAction = SdkActionManager.parseAction(fullType);
  if (!parsedAction) {
    return;
  }

  // ❌ 没有验证 e.origin（消息来源域名）
  // ❌ 没有验证 e.source（消息来源窗口）
  // ❌ 任何窗口只要发送 { fullType: "CAL::xxx", ... } 就能触发
  
  const actionManager = Cal.actionsManagers[parsedAction.ns];
  // ...
  actionManager.fire(parsedAction.type, detail.data);
});

// SdkActionManager.parseAction 实现（sdk-action-manager.ts:314-327）
static parseAction(fullType: string) {
  if (!fullType) {
    return null;
  }
  // FIXME: Ensure that any action if it has :, it is properly encoded.
  const [cal, calNamespace, type] = fullType.split(":");
  if (cal !== "CAL") {
    return null;
  }
  return {
    ns: calNamespace,
    type,
  };
}
```

### 4.4 安全校验对照表

| 校验项 | 父窗口（发送） | Iframe（发送） | 父窗口（接收） | Iframe（接收） |
|-------|--------------|---------------|--------------|---------------|
| **targetOrigin 指定** | ❌ `*` | ❌ `*` | N/A | N/A |
| **验证 `e.origin`** | N/A | N/A | ❌ 未验证 | ❌ 未验证 |
| **验证 `e.source`** | N/A | N/A | ❌ 未验证 | ❌ 未验证 |
| **验证消息标识** | ✅ `originator: "CAL"` | ✅ `originator: "CAL"` | ✅ `CAL:` 前缀 | ✅ `originator: "CAL"` |

### 4.5 潜在攻击场景

#### 场景 1：恶意 iframe 接收消息

```
┌─────────────────────────────────────────────────────────────────┐
│  攻击场景：恶意 iframe 窃听消息                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  正常场景：                                                       │
│  ┌─────────────┐         ┌─────────────┐                        │
│  │  父窗口      │ ──────> │  Cal iframe │                        │
│  │ example.com │         │ cal.com     │                        │
│  │             │         │             │                        │
│  │ postMessage │         │ 接收命令    │                        │
│  │ target="*"  │         │             │                        │
│  └─────────────┘         └─────────────┘                        │
│                                                                   │
│  攻击场景：                                                       │
│  ┌─────────────┐         ┌─────────────┐                        │
│  │  父窗口      │ ──────> │  恶意 iframe │                        │
│  │ example.com │         │ evil.com    │                        │
│  │             │         │             │                        │
│  │ postMessage │         │ 也能接收！  │                        │
│  │ target="*"  │         │             │                        │
│  └─────────────┘         └─────────────┘                        │
│                                                                   │
│  风险：虽然消息内容不敏感（UI 配置等），但理论上可被窃听          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### 场景 2：伪造消息注入

```
┌─────────────────────────────────────────────────────────────────┐
│  攻击场景：伪造消息注入                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  正常流程：                                                       │
│  ┌─────────────┐         ┌─────────────┐                        │
│  │  Cal iframe  │ ──────> │  父窗口      │                        │
│  │ cal.com     │         │ example.com │                        │
│  │             │         │             │                        │
│  │ 发送事件     │         │ 处理事件    │                        │
│  │ {           │         │             │                        │
│  │   originator│         │             │                        │
│  │   : "CAL"   │         │             │                        │
│  │ }           │         │             │                        │
│  └─────────────┘         └─────────────┘                        │
│                                                                   │
│  攻击场景：                                                       │
│  ┌─────────────┐         ┌─────────────┐                        │
│  │  任意窗口    │ ──────> │  父窗口      │                        │
│  │ any.com     │         │ example.com │                        │
│  │             │         │             │                        │
│  │ 伪造消息     │         │ 也能处理！  │                        │
│  │ {           │         │             │                        │
│  │   originator│         │             │                        │
│  │   : "CAL",  │         │             │                        │
│  │   fullType: │         │             │                        │
│  │   "CAL::xx" │         │             │                        │
│  │ }           │         │             │                        │
│  └─────────────┘         └─────────────┘                        │
│                                                                   │
│  风险：                                                           │
│  - 任何窗口都可以伪造消息格式                                     │
│  - 可以触发 `linkReady`、`bookingSuccessful` 等事件              │
│  - 虽然这些事件通常只影响 UI，但可能导致状态混乱                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.6 现有缓解措施

虽然 postMessage 校验不足，但系统有多层防护：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    安全防护层次                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Layer 1: 消息内容限制                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  父窗口 → Iframe 的命令：                                      │   │
│  │  - "ui" - UI 配置（主题、布局等）                             │   │
│  │  - "parentKnowsIframeReady" - 握手确认                       │   │
│  │  - "connect" - 连接预渲染页面                                 │   │
│  │  ⚠️ 这些都是非敏感操作，不涉及业务逻辑                        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Layer 2: 关键操作在服务端                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  预约创建、确认等关键操作：                                    │   │
│  │  - 不在 postMessage 中传递                                    │   │
│  │  - 通过 tRPC API 直接调用服务端                               │   │
│  │  - 服务端有自己的验证逻辑                                     │   │
│  │  ⚠️ 即使 postMessage 被劫持，也无法直接创建预约              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Layer 3: uid Cookie 限制                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  追踪访客的 uid cookie：                                       │   │
│  │  - SameSite=None（仅 HTTPS）                                 │   │
│  │  - Secure（仅 HTTPS）                                        │   │
│  │  - HttpOnly（无法通过 JS 读取）                              │   │
│  │  ⚠️ 即使 postMessage 被劫持，攻击者也无法获取或设置 uid      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  Layer 4: 业务逻辑验证                                               │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  创建预约时：                                                  │   │
│  │  - 验证时间段是否仍可用                                       │   │
│  │  - 验证 uid 是否匹配预留记录                                  │   │
│  │  - 验证事件类型是否存在                                       │   │
│  │  ⚠️ 即使前端被篡改，服务端也会重新验证                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、完整调用链梳理

### 5.1 从嵌入到预约完成的完整流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    完整预约流程调用链                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  阶段 1: 脚本加载与初始化                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  1.1 外部网站加载 embed 脚本                                   │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ <!-- 外部网站 HTML -->                                  │   │   │
│  │  │ <script>                                                │   │   │
│  │  │   (function(C,A,L){ /* ... */ })(                      │   │   │
│  │  │     window,                                             │   │   │
│  │  │     "https://cal.com/embed/embed.js",                  │   │   │
│  │  │     "init"                                              │   │   │
│  │  │   );                                                    │   │   │
│  │  │   Cal("init", { origin: "https://cal.com" });         │   │   │
│  │  │ </script>                                               │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  │  1.2 初始化 Cal API                                            │   │
│  │  - 创建全局 Cal 对象                                          │   │
│  │  - 设置命名空间                                               │   │
│  │  - 准备队列存储延迟指令                                       │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  阶段 2: Iframe 创建与握手                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  2.1 创建 iframe                                              │   │
│  │  Cal("modal", { calLink: "username/30min" });               │   │
│  │  │                                                            │   │
│  │  ├─── createIframe()                                         │   │
│  │  │     ├─── 设置 src: "cal.com/username/30min/embed"       │   │
│  │  │     ├─── 设置 visibility: hidden                         │   │
│  │  │     └─── 添加到 DOM                                       │   │
│  │  │                                                            │   │
│  │  2.2 握手协议（⚠️ postMessage 无严格校验）                   │   │
│  │  │                                                            │   │
│  │  父窗口                    Iframe (cal.com)                  │   │
│  │  │                            │                               │   │
│  │  │                            ├─── main() 执行               │   │
│  │  │                            │    - 检测 embed 模式         │   │
│  │  │                            │    - 初始化 embedStore       │   │
│  │  │                            │                               │   │
│  │  │<─── postMessage ───────────│                               │   │
│  │  │   {                        │                               │   │
│  │  │     originator: "CAL",    │                               │   │
│  │  │     type: "__iframeReady",│                               │   │
│  │  │     data: { isPrerendering }                              │   │
│  │  │   }                        │                               │   │
│  │  │   targetOrigin: "*"       │                               │   │
│  │  │                            │                               │   │
│  │  ├─── iframeReady = true     │                               │   │
│  │  ├─── 显示 iframe            │                               │   │
│  │  │                            │                               │   │
│  │  ├─── postMessage ───────────>│                               │   │
│  │  │   {                        │                               │   │
│  │  │     originator: "CAL",    │                               │   │
│  │  │     method: "parentKnowsIframeReady"                     │   │
│  │  │   }                        │                               │   │
│  │  │   targetOrigin: "*"       │                               │   │
│  │  │                            │                               │   │
│  │  │                            ├─── 等待页面渲染               │   │
│  │  │                            ├─── 等待插槽加载               │   │
│  │  │                            │                               │   │
│  │  │<─── postMessage ───────────│                               │   │
│  │  │   {                        │                               │   │
│  │  │     originator: "CAL",    │                               │   │
│  │  │     fullType: "CAL::linkReady"                           │   │
│  │  │   }                        │                               │   │
│  │  │   targetOrigin: "*"       │                               │   │
│  │  │                            │                               │   │
│  │  ├─── 处理 iframeDoQueue     │                               │   │
│  │  └─── 用户可交互             │                               │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  阶段 3: 预约流程（全部匿名 API）                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  3.1 加载事件信息                                              │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 调用: publicViewer.event()                              │   │   │
│  │  │ 端点: /api/trpc/public/event.get                        │   │   │
│  │  │ Procedure: publicProcedure                              │   │   │
│  │  │ 认证: ❌ 无需                                           │   │   │
│  │  │                                                         │   │   │
│  │  │ 返回: 事件详情                                           │   │   │
│  │  │ - 时长、描述、位置                                       │   │   │
│  │  │ - 组织者信息（脱敏）                                     │   │   │
│  │  │ - 自定义字段配置                                         │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                           ↓                                    │   │
│  │  3.2 获取可用时间段                                           │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 调用: slots.getSchedule()                               │   │   │
│  │  │ 端点: /api/trpc/slots/getSchedule                      │   │   │
│  │  │ Procedure: publicProcedure                              │   │   │
│  │  │ 认证: ❌ 无需                                           │   │   │
│  │  │                                                         │   │   │
│  │  │ 输入: {                                                 │   │   │
│  │  │   eventTypeId: number,                                  │   │   │
│  │  │   startTime: ISODate,                                   │   │   │
│  │  │   endTime: ISODate,                                     │   │   │
│  │  │   timezone: string                                      │   │   │
│  │  │ }                                                       │   │   │
│  │  │                                                         │   │   │
│  │  │ 返回: 可用时间段列表                                     │   │   │
│  │  │ - 已被其他访客预留的槽会被排除                           │   │   │
│  │  │ - 考虑组织者的日程冲突                                   │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                           ↓                                    │   │
│  │  3.3 预留时间段                                               │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 调用: slots.reserveSlot()                               │   │   │
│  │  │ 端点: /api/trpc/slots/reserveSlot                      │   │   │
│  │  │ Procedure: publicProcedure                              │   │   │
│  │  │ 认证: ❌ 无需                                           │   │   │
│  │  │                                                         │   │   │
│  │  │ 输入: {                                                 │   │   │
│  │  │   slotUtcStartDate: ISODate,                           │   │   │
│  │  │   slotUtcEndDate: ISODate,                             │   │   │
│  │  │   eventTypeId: number                                   │   │   │
│  │  │ }                                                       │   │   │
│  │  │                                                         │   │   │
│  │  │ 处理:                                                   │   │   │
│  │  │ 1. 检查时间段是否仍可用                                  │   │   │
│  │  │ 2. 检查是否被其他人预留                                  │   │   │
│  │  │ 3. 在 selectedSlots 表创建记录                          │   │   │
│  │  │    - userId: 组织者 ID                                  │   │   │
│  │  │    - uid: 访客标识（新生成或从 cookie 读取）            │   │   │
│  │  │    - releaseAt: 过期时间（通常 15 分钟）               │   │   │
│  │  │                                                         │   │   │
│  │  │ 返回: { uid: string }                                   │   │   │
│  │  │                                                         │   │   │
│  │  │ ⚠️ 设置 Cookie:                                         │   │   │
│  │  │ Set-Cookie: uid=xxx;                                    │   │   │
│  │  │     SameSite=None;    // 允许跨域                       │   │   │
│  │  │     Secure;            // 仅 HTTPS                       │   │   │
│  │  │     HttpOnly;          // JS 不可读                      │   │   │
│  │  │     Path=/             │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                           ↓                                    │   │
│  │  3.4 填写表单并创建预约                                       │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 调用: createBooking() (内部服务，非 tRPC)               │   │   │
│  │  │ 位置: packages/features/bookings/lib/handleNewBooking/ │   │   │
│  │  │ 认证: ❌ 无需                                           │   │   │
│  │  │ 追踪: uid cookie（自动携带）                            │   │   │
│  │  │                                                         │   │   │
│  │  │ 输入: {                                                 │   │   │
│  │  │   uid: string,        // 从 cookie 读取                 │   │   │
│  │  │   name: string,       // 访客姓名                       │   │   │
│  │  │   email: string,      // 访客邮箱                       │   │   │
│  │  │   notes?: string,     // 备注                           │   │   │
│  │  │   customInputs?: {},  // 自定义字段                     │   │   │
│  │  │   startTime: Date,    // 预约时间                       │   │   │
│  │  │   endTime: Date,      // 结束时间                       │   │   │
│  │  │   eventTypeId: number // 事件类型                       │   │   │
│  │  │ }                                                       │   │   │
│  │  │                                                         │   │   │
│  │  │ 处理流程:                                               │   │   │
│  │  │ 1. 验证 uid 与预留的时间段匹配                          │   │   │
│  │  │ 2. 检查时间段是否仍可用（防止过期）                      │   │   │
│  │  │ 3. 构建 booking 数据                                    │   │   │
│  │  │ 4. 数据库事务：                                          │   │   │
│  │  │    a. 如果是改期，取消原预约                            │   │   │
│  │  │    b. 创建新 booking 记录                               │   │   │
│  │  │    c. 创建 attendee 记录                                │   │   │
│  │  │    d. 删除 selectedSlots 预留记录                      │   │   │
│  │  │ 5. 发送邮件通知                                         │   │   │
│  │  │                                                         │   │   │
│  │  │ 返回: Booking 对象                                      │   │   │
│  │  │ - id: 预约 ID                                           │   │   │
│  │  │ - uid: 访客标识                                         │   │   │
│  │  │ - status: ACCEPTED 或 PENDING                          │   │   │
│  │  │ - startTime/endTime: 预约时间                          │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                           ↓                                    │   │
│  │  3.5 通知父窗口                                               │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 调用: sdkActionManager.fire("bookingSuccessful", data) │   │   │
│  │  │                                                         │   │   │
│  │  │ 通过 postMessage 发送:                                  │   │   │
│  │  │ {                                                       │   │   │
│  │  │   originator: "CAL",                                   │   │   │
│  │  │   fullType: "CAL::bookingSuccessful",                  │   │   │
│  │  │   type: "bookingSuccessful",                           │   │   │
│  │  │   data: {                                               │   │   │
│  │  │     uid: string,           // 预约 UID                 │   │   │
│  │  │     title: string,         // 事件标题                 │   │   │
│  │  │     startTime: string,     // 开始时间                 │   │   │
│  │  │     endTime: string,       // 结束时间                 │   │   │
│  │  │     eventTypeId: number,   // 事件类型 ID              │   │   │
│  │  │     // ... 其他字段                                    │   │   │
│  │  │   }                                                     │   │   │
│  │  │ }                                                       │   │   │
│  │  │ targetOrigin: "*"                                       │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  │  ✅ 至此，访客预约流程完成！                                   │   │
│  │  ✅ 全部使用匿名 API，无需登录                                │   │
│  │  ✅ 通过 uid cookie 追踪访客身份                              │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  阶段 4: 组织者确认（可选，需登录）                                 │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  ⚠️ 注意：许多事件类型默认自动确认，无需此步骤                │   │
│  │                                                                 │   │
│  │  4.1 组织者登录 Cal.diy                                       │   │
│  │  4.2 查看待确认预约列表                                       │   │
│  │  4.3 确认或拒绝预约                                           │   │
│  │  ┌───────────────────────────────────────────────────────┐   │   │
│  │  │ 调用: viewer.bookings.confirm()                         │   │   │
│  │  │ 端点: /api/trpc/bookings.confirm                        │   │   │
│  │  │ Procedure: authedProcedure                              │   │   │
│  │  │ 认证: ✅ 需登录                                          │   │   │
│  │  │                                                         │   │   │
│  │  │ 权限检查:                                               │   │   │
│  │  │ - 验证用户是否有权限操作该预约                          │   │   │
│  │  │ - 检查用户是否是组织者或团队成员                        │   │   │
│  │  │                                                         │   │   │
│  │  │ 输入: { bookingId, confirmed: boolean }                │   │   │
│  │  │ 返回: { message, status }                               │   │   │
│  │  └───────────────────────────────────────────────────────┘   │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键数据流

```
┌─────────────────────────────────────────────────────────────────────┐
│                    关键数据流                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  uid 追踪流                                                          │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  1. reserveSlot 调用时：                                      │   │
│  │     - 检查 Cookie: uid                                        │   │
│  │     - 如果不存在，生成新的 UUID                               │   │
│  │     - 设置 Set-Cookie: uid=xxx; SameSite=None; Secure      │   │
│  │                                                                 │   │
│  │  2. 后续所有请求自动携带：                                     │   │
│  │     - Cookie: uid=xxx                                        │   │
│  │     - 浏览器自动处理，JS 无法读取（HttpOnly）                │   │
│  │                                                                 │   │
│  │  3. createBooking 时：                                        │   │
│  │     - 从 request.cookies 读取 uid                            │   │
│  │     - 用 uid 查找对应的 selectedSlots 记录                   │   │
│  │     - 验证时间段是否匹配                                       │   │
│  │     - 创建 booking 并关联 uid                                │   │
│  │     - 删除 selectedSlots 记录                                 │   │
│  │                                                                 │   │
│  │  4. uid 的作用：                                               │   │
│  │     - 关联预留时间段和预约                                    │   │
│  │     - 防止一个访客占用多个时间段                              │   │
│  │     - 追踪预约归属（访客侧）                                  │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  预约状态流转                                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  时间段状态：                                                  │   │
│  │  可用 ──> 预留（selectedSlots）──> 已预约（booking）        │   │
│  │           │                         │                         │   │
│  │           │ 15 分钟超时          │ 状态：ACCEPTED 或        │   │
│  │           │ 自动释放             │ PENDING（需确认）         │   │
│  │           ▼                         ▼                         │   │
│  │         可用                    完成/取消                    │   │
│  │                                                                 │   │
│  │  Booking 状态（取决于事件配置）：                              │   │
│  │  - ACCEPTED: 自动确认（默认）                                 │   │
│  │  - PENDING: 需组织者手动确认                                  │   │
│  │  - CANCELLED: 已取消                                          │   │
│  │  - REJECTED: 已拒绝                                           │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、关键文件索引

### 6.1 Embed 核心

| 文件路径 | 职责 | 关键内容 |
|---------|------|---------|
| `packages/embeds/embed-core/src/embed.ts` | 父窗口 API 实现 | `doInIframe()` 发送消息，`targetOrigin="*"` |
| `packages/embeds/embed-core/src/embed-iframe.ts` | Iframe 消息处理 | `messageParent()` 发送事件，消息监听仅校验 `originator: "CAL"` |
| `packages/embeds/embed-core/src/sdk-action-manager.ts` | 事件管理器 | `parseAction()` 仅检查 `CAL:` 前缀 |

### 6.2 API 路由与认证

| 文件路径 | 职责 | 关键内容 |
|---------|------|---------|
| `packages/trpc/server/procedures/publicProcedure.ts` | 公开 procedure 定义 | 无认证 middleware |
| `packages/trpc/server/procedures/authedProcedure.ts` | 认证 procedure 定义 | 使用 `isAuthed` middleware |
| `packages/trpc/server/middlewares/sessionMiddleware.ts` | 认证 middleware | `isAuthed` 检查 session/user |
| `packages/trpc/server/routers/viewer/slots/_router.tsx` | 插槽路由 | ⚠️ 位于 viewer 目录但使用 `publicProcedure` |
| `packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts` | 预留时间段 | 设置 `uid` cookie，`SameSite=None` |
| `packages/trpc/server/routers/viewer/bookings/confirm.handler.ts` | 确认预约 | 使用 `authedProcedure`，有额外权限检查 |

### 6.3 预约业务逻辑

| 文件路径 | 职责 |
|---------|------|
| `packages/features/bookings/lib/handleNewBooking/createBooking.ts` | 创建预约核心逻辑 |
| `packages/features/bookings/lib/handleNewBooking/` | 完整预约流程 |
| `packages/features/selectedSlots/repositories/` | 时间段预留存储 |

---

## 七、总结与建议

### 7.1 核心结论

#### 关于 API 认证边界

```
✅ 正确理解：
┌─────────────────────────────────────────────────────────────────┐
│  1. slotsRouter（getSchedule, reserveSlot 等）是完全匿名的      │
│     - 虽然代码在 viewer/slots/ 目录下                           │
│     - 但使用的是 publicProcedure                                 │
│     - 通过独立的 /api/trpc/slots/ 端点暴露                      │
│                                                                   │
│  2. 整个访客预约流程可匿名完成                                    │
│     - 无需登录 Cal.com 账号                                      │
│     - 通过 uid cookie 追踪访客身份                               │
│     - uid cookie: SameSite=None, Secure, HttpOnly              │
│                                                                   │
│  3. 只有组织者操作需要登录                                        │
│     - confirm（确认/拒绝）                                      │
│     - cancel（取消）                                             │
│     - requestReschedule（改期）                                 │
│     - 这些使用 authedProcedure，通过 session 认证               │
└─────────────────────────────────────────────────────────────────┘
```

#### 关于 postMessage 安全

```
⚠️ 需要注意：
┌─────────────────────────────────────────────────────────────────┐
│  1. 两侧发送都使用 targetOrigin="*"                              │
│     - 理论上任何窗口都能接收消息                                  │
│     - 但消息内容都是 UI 配置或事件通知，不敏感                   │
│                                                                   │
│  2. 两侧接收都只验证字符串标识                                    │
│     - Iframe 接收: 检查 data.originator === "CAL"              │
│     - 父窗口接收: 检查 fullType 以 "CAL:" 开头                  │
│     - ❌ 都没有验证 e.origin 或 e.source                        │
│                                                                   │
│  3. 实际风险较低                                                  │
│     - 可执行的命令有限（ui, parentKnowsIframeReady, connect）  │
│     - 关键操作（创建预约）不在 postMessage 中处理               │
│     - 服务端有独立的验证逻辑                                     │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 安全建议

| 风险项 | 当前状态 | 建议改进 |
|-------|---------|---------|
| `targetOrigin="*"` | 发送时使用 | 改为从 `__config.origin` 动态获取 |
| 不验证 `e.origin` | 接收时未验证 | 验证消息来源是否是预期域名 |
| 不验证 `e.source` | 接收时未验证 | 验证消息来源窗口 |
| uid cookie 有效期 | 无明确过期 | 设置合理的 Max-Age |

### 7.3 设计亮点

```
┌─────────────────────────────────────────────────────────────────────┐
│                    优秀的设计决策                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 匿名预约设计                                                     │
│     - 访客无需注册账号即可完成预约                                  │
│     - 降低预约门槛，提升转化率                                      │
│     - 通过 uid cookie 实现轻量级追踪                                │
│                                                                      │
│  2. 路由分层设计                                                     │
│     - slotsRouter 独立暴露，不依赖 viewerRouter                    │
│     - 明确区分公开操作和认证操作                                    │
│     - 便于安全审计和权限控制                                        │
│                                                                      │
│  3. uid cookie 策略                                                 │
│     - SameSite=None 允许跨域 iframe 访问                           │
│     - Secure 确保仅在 HTTPS 下传输                                 │
│     - HttpOnly 防止 XSS 窃取                                       │
│     - 平衡了安全性和用户体验                                        │
│                                                                      │
│  4. 业务层验证                                                       │
│     - 创建预约时重新验证时间段可用性                                │
│     - 验证 uid 与预留记录的匹配                                    │
│     - 不依赖前端状态，服务端做最终判断                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```
