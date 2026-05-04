# 外部网站嵌入预约组件协作机制分析

> **版本说明**：v3.0，统一接口认证边界表述，修复 `bookingsRouter.find` 匿名可用的发现

---

## 一、预约接口认证边界完整对照表

### 1.1 核心结论

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ⚠️ 关键发现：接口认证边界                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  🔓 匿名可用接口（publicProcedure）：                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  1. slotsRouter（独立端点：/api/trpc/slots/）                 │   │
│  │     • getSchedule()        - 获取可用时间段                   │   │
│  │     • reserveSlot()        - 预留时间段（设置 uid cookie）   │   │
│  │     • isAvailable()        - 检查时间段可用性                 │   │
│  │     • removeSelectedSlotMark() - 移除预留标记                │   │
│  │                                                                 │   │
│  │  2. bookingsRouter.find（端点：/api/trpc/bookings.find）     │   │
│  │     • find()               - 通过 bookingUid 查找预约         │   │
│  │                          ⚠️ 仅返回脱敏的基本信息              │   │
│  │                                                                 │   │
│  │  3. publicViewerRouter（独立端点：/api/trpc/public/）         │   │
│  │     • event()              - 获取公开事件信息                 │   │
│  │     • submitRating()       - 提交评价                         │   │
│  │     • markHostAsNoShow()   - 标记组织者未出现                 │   │
│  │     • countryCode()        - 获取国家代码                     │   │
│  │                                                                 │   │
│  │  4. 内部服务调用（非 tRPC 端点）                               │   │
│  │     • createBooking()      - 创建预约（业务层直接调用）       │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  🔒 需登录接口（authedProcedure）：                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  bookingsRouter（端点：/api/trpc/bookings.xxx）               │   │
│  │  • get()                  - 获取用户预约列表                   │   │
│  │  • confirm()              - 确认/拒绝预约                     │   │
│  │  • requestReschedule()    - 申请改期                         │   │
│  │  • editLocation()         - 编辑预约地点                      │   │
│  │  • addGuests()            - 添加嘉宾                           │   │
│  │  • getBookingAttendees()  - 获取预约嘉宾                      │   │
│  │  • getBookingDetails()    - 获取预约详情                      │   │
│  │  • getBookingHistory()    - 获取预约历史                      │   │
│  │  • reportBooking()        - 报告预约                          │   │
│  │  • reportWrongAssignment() - 报告错误分配                      │   │
│  │  • hasWrongAssignmentReport() - 检查错误分配报告              │   │
│  │  • getWrongAssignmentReports() - 获取错误分配报告列表         │   │
│  │  • updateWrongAssignmentReportStatus() - 更新报告状态         │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ⚠️ 注意事项：                                                        │
│  1. slotsRouter 虽然代码位置在 viewer/slots/ 目录下，              │
│     但使用的是 publicProcedure，是完全匿名的                        │
│  2. bookingsRouter.find 是 bookingsRouter 中唯一的匿名接口        │
│     其他 bookings 接口都需要登录                                    │
│  3. createBooking 不在 tRPC 路由中，是内部业务服务                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 完整接口认证对照表

| 接口名称 | 路由/端点 | Procedure | 认证要求 | 返回数据范围 |
|---------|----------|-----------|---------|-------------|
| **slots.getSchedule** | `/api/trpc/slots/getSchedule` | `publicProcedure` | 🔓 无需 | 可用时间段列表 |
| **slots.reserveSlot** | `/api/trpc/slots/reserveSlot` | `publicProcedure` | 🔓 无需 | `{ uid: string }` |
| **slots.isAvailable** | `/api/trpc/slots/isAvailable` | `publicProcedure` | 🔓 无需 | 可用性状态 |
| **slots.removeSelectedSlotMark** | `/api/trpc/slots/removeSelectedSlotMark` | `publicProcedure` | 🔓 无需 | 无 |
| **publicViewer.event** | `/api/trpc/public/event` | `publicProcedure` | 🔓 无需 | 公开事件信息 |
| **publicViewer.submitRating** | `/api/trpc/public/submitRating` | `publicProcedure` | 🔓 无需 | 评价结果 |
| **publicViewer.markHostAsNoShow** | `/api/trpc/public/markHostAsNoShow` | `publicProcedure` | 🔓 无需 | 操作结果 |
| **publicViewer.countryCode** | `/api/trpc/public/countryCode` | `publicProcedure` | 🔓 无需 | 国家代码 |
| **bookings.find** ⚠️ | `/api/trpc/bookings.find` | `publicProcedure` | 🔓 无需 | 仅 `{ id, uid, startTime, endTime, description, status, paid, eventTypeId }` |
| **createBooking** | 内部服务调用 | N/A | 🔓 无需 | Booking 对象 |
| --- | --- | --- | --- | --- |
| **bookings.get** | `/api/trpc/bookings.get` | `authedProcedure` | 🔒 需登录 | 用户完整预约列表 |
| **bookings.confirm** | `/api/trpc/bookings.confirm` | `authedProcedure` | 🔒 需登录 | 确认/拒绝结果 |
| **bookings.requestReschedule** | `/api/trpc/bookings.requestReschedule` | `authedProcedure` | 🔒 需登录 | 改期请求结果 |
| **bookings.editLocation** | `/api/trpc/bookings.editLocation` | `bookingsProcedure` (继承 authed) | 🔒 需登录 | 编辑结果 |
| **bookings.addGuests** | `/api/trpc/bookings.addGuests` | `authedProcedure` | 🔒 需登录 | 添加结果 |
| **bookings.getBookingAttendees** | `/api/trpc/bookings.getBookingAttendees` | `authedProcedure` | 🔒 需登录 | 嘉宾列表 |
| **bookings.getBookingDetails** | `/api/trpc/bookings.getBookingDetails` | `authedProcedure` | 🔒 需登录 | 预约详情 |
| **bookings.getBookingHistory** | `/api/trpc/bookings.getBookingHistory` | `authedProcedure` | 🔒 需登录 | 历史记录 |

### 1.3 关键代码验证

#### 验证 1：slotsRouter 使用 publicProcedure

**文件**: `packages/trpc/server/routers/viewer/slots/_router.tsx`

```typescript
// ⚠️ 虽然在 viewer/slots/ 目录下，但使用的是 publicProcedure！
import publicProcedure from "../../../procedures/publicProcedure";
import { router } from "../../../trpc";

export const slotsRouter = router({
  getSchedule: publicProcedure
    .input(ZGetScheduleInputSchema)
    .query(async ({ input, ctx }) => { /* ... */ }),
  
  reserveSlot: publicProcedure
    .input(ZReserveSlotInputSchema)
    .mutation(async ({ input, ctx }) => { /* ... */ }),
  
  isAvailable: publicProcedure
    .input(ZIsAvailableInputSchema)
    .query(async ({ input, ctx }) => { /* ... */ }),
  
  removeSelectedSlotMark: publicProcedure
    .input(ZRemoveSelectedSlotInputSchema)
    .mutation(async ({ input, ctx }) => { /* ... */ }),
});
```

#### 验证 2：bookingsRouter.find 使用 publicProcedure

**文件**: `packages/trpc/server/routers/viewer/bookings/_router.tsx`

```typescript
import publicProcedure from "../../../procedures/publicProcedure";
import authedProcedure from "../../../procedures/authedProcedure";
import { router } from "../../../trpc";
import { bookingsProcedure } from "./util";  // 继承自 authedProcedure

export const bookingsRouter = router({
  // ⚠️ 注意：find 是 bookingsRouter 中唯一使用 publicProcedure 的接口！
  find: publicProcedure.input(ZFindInputSchema).query(async ({ input, ctx }) => {
    const { getHandler } = await import("./find.handler");
    return getHandler({ ctx, input });
  }),
  
  // 以下所有接口都需要登录
  get: authedProcedure.input(ZGetInputSchema).query(async ({ input, ctx }) => { /* ... */ }),
  
  requestReschedule: authedProcedure.input(ZRequestRescheduleInputSchema).mutation(async ({ input, ctx }) => { /* ... */ }),
  
  editLocation: bookingsProcedure.input(ZEditLocationInputSchema).mutation(async ({ input, ctx }) => { /* ... */ }),
  
  addGuests: authedProcedure.input(ZAddGuestsInputSchema).mutation(async ({ input, ctx }) => { /* ... */ }),
  
  confirm: authedProcedure.input(ZConfirmInputSchema).mutation(async ({ input, ctx }) => { /* ... */ }),
  
  // ... 其他所有 bookings 接口都使用 authedProcedure
});
```

#### 验证 3：bookings.find 返回脱敏数据

**文件**: `packages/trpc/server/routers/viewer/bookings/find.handler.ts`

```typescript
export const getHandler = async ({ ctx, input }: GetOptions) => {
  const { prisma } = ctx;
  const { bookingUid } = input;

  const booking = await prisma.booking.findUnique({
    where: { uid: bookingUid },
    // ⚠️ 只返回脱敏的基本信息
    select: {
      id: true,
      uid: true,
      startTime: true,
      endTime: true,
      description: true,
      status: true,
      paid: true,
      eventTypeId: true,
    },
  });

  // Don't leak anything private from the booking
  return { booking };
};
```

**对比**：`bookings.get` 返回完整数据（包含 attendees、eventType、user 等敏感信息）

---

## 二、完整预约流程接口调用链

### 2.1 访客视角（全流程匿名）

```
┌─────────────────────────────────────────────────────────────────────┐
│                    访客预约流程（全匿名）                            │
│                    无需 Cal.com 账号登录                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  阶段 1：查看事件信息                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用：publicViewer.event()                                    │   │
│  │  端点：/api/trpc/public/event                                  │   │
│  │  认证：🔓 publicProcedure                                      │   │
│  │                                                                 │   │
│  │  输入：{ username, eventTypeSlug }                            │   │
│  │  输出：事件详情（时长、描述、位置、组织者信息等）               │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           ↓                                          │
│  阶段 2：查看可用时间段                                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用：slots.getSchedule()                                     │   │
│  │  端点：/api/trpc/slots/getSchedule                             │   │
│  │  认证：🔓 publicProcedure                                      │   │
│  │                                                                 │   │
│  │  输入：{ eventTypeId, startTime, endTime, timezone }          │   │
│  │  输出：可用时间段列表                                          │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           ↓                                          │
│  阶段 3：预留时间段                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用：slots.reserveSlot()                                     │   │
│  │  端点：/api/trpc/slots/reserveSlot                             │   │
│  │  认证：🔓 publicProcedure                                      │   │
│  │                                                                 │   │
│  │  输入：{ slotUtcStartDate, slotUtcEndDate, eventTypeId }      │   │
│  │  输出：{ uid: string }                                         │   │
│  │                                                                 │   │
│  │  ⚠️ 关键操作：                                                  │   │
│  │  1. 在 selectedSlots 表创建预留记录                           │   │
│  │  2. 设置 uid Cookie（SameSite=None, Secure, HttpOnly）        │   │
│  │  3. 预留通常在 15 分钟后自动释放                              │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           ↓                                          │
│  阶段 4：创建预约（内部服务调用）                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用：createBooking()（内部服务，非 tRPC 端点）               │   │
│  │  位置：packages/features/bookings/lib/handleNewBooking/       │   │
│  │  认证：🔓 无需（通过 uid cookie 追踪）                        │   │
│  │                                                                 │   │
│  │  输入：{                                                        │   │
│  │    uid: string,        // 从 cookie 读取                      │   │
│  │    name: string,       // 访客姓名                            │   │
│  │    email: string,      // 访客邮箱                            │   │
│  │    startTime: Date,    // 预约时间                            │   │
│  │    endTime: Date,      // 结束时间                            │   │
│  │    eventTypeId: number // 事件类型                            │   │
│  │  }                                                             │   │
│  │                                                                 │   │
│  │  输出：Booking 对象                                            │   │
│  │                                                                 │   │
│  │  ⚠️ 关键操作：                                                  │   │
│  │  1. 验证 uid 与预留的时间段匹配                               │   │
│  │  2. 检查时间段是否仍可用                                      │   │
│  │  3. 数据库事务：                                               │   │
│  │     - 创建 Booking 记录                                        │   │
│  │     - 创建 Attendee 记录                                       │   │
│  │     - 删除 selectedSlots 预留记录                             │   │
│  │  4. 发送邮件通知                                               │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           ↓                                          │
│  阶段 5：查看预约详情（可选）                                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用：bookings.find()                                         │   │
│  │  端点：/api/trpc/bookings.find                                 │   │
│  │  认证：🔓 publicProcedure                                      │   │
│  │                                                                 │   │
│  │  输入：{ bookingUid: string }                                 │   │
│  │  输出：{ booking: { id, uid, startTime, endTime,             │   │
│  │                    description, status, paid, eventTypeId } } │   │
│  │                                                                 │   │
│  │  ⚠️ 注意：只返回脱敏的基本信息，不包含敏感信息                 │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ✅ 访客预约完成！全程无需登录                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 组织者视角（需登录）

```
┌─────────────────────────────────────────────────────────────────────┐
│                    组织者操作（需登录）                               │
│                    需要 Cal.com 账号登录                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  前置条件：登录成功，获得 Session                                    │
│                                                                      │
│  操作 1：查看预约列表                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用：bookings.get()                                          │   │
│  │  端点：/api/trpc/bookings.get                                  │   │
│  │  认证：🔒 authedProcedure                                      │   │
│  │                                                                 │   │
│  │  输入：filters, sort, limit, offset                            │   │
│  │  输出：完整的预约列表（包含 attendees、eventType、user 等）   │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           ↓                                          │
│  操作 2：查看预约详情                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用：bookings.getBookingDetails()                            │   │
│  │  端点：/api/trpc/bookings.getBookingDetails                    │   │
│  │  认证：🔒 authedProcedure                                      │   │
│  │                                                                 │   │
│  │  输入：{ uid: string }                                         │   │
│  │  输出：完整的预约详情                                          │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           ↓                                          │
│  操作 3：确认/拒绝预约（如需要）                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  调用：bookings.confirm()                                      │   │
│  │  端点：/api/trpc/bookings.confirm                              │   │
│  │  认证：🔒 authedProcedure                                      │   │
│  │                                                                 │   │
│  │  输入：{ bookingId, confirmed: boolean }                       │   │
│  │  输出：{ message, status }                                     │   │
│  │                                                                 │   │
│  │  ⚠️ 权限检查：                                                  │   │
│  │  - 验证用户是否有权限操作该预约                                │   │
│  │  - 检查用户是否是组织者或团队成员                              │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                           ↓                                          │
│  操作 4：改期/取消/编辑（如需要）                                   │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  bookings.requestReschedule() - 申请改期                      │   │
│  │ bookings.editLocation()        - 编辑地点                     │   │
│  │ bookings.addGuests()           - 添加嘉宾                     │   │
│  │                                                                 │   │
│  │  全部使用 🔒 authedProcedure 或 bookingsProcedure              │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、认证边界设计意图分析

### 3.1 为什么这样设计？

```
┌─────────────────────────────────────────────────────────────────────┐
│                    认证边界设计意图                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  设计理念：访客预约零门槛，组织者操作高安全                          │
│                                                                      │
│  🔓 匿名接口的设计原因：                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  1. 访客不需要 Cal.com 账号                                   │   │
│  │     - 预约是访客向组织者发起的行为                             │   │
│  │     - 要求访客注册会大幅降低转化率                            │   │
│  │     - 通过 uid cookie 实现轻量级追踪                          │   │
│  │                                                                 │   │
│  │  2. 公开信息本身就是可访问的                                   │   │
│  │     - 事件信息、可用时间本来就是公开的                        │   │
│  │     - 任何人都可以通过公开链接访问                            │   │
│  │     - 不需要额外的认证层                                      │   │
│  │                                                                 │   │
│  │  3. bookings.find 的特殊设计                                   │   │
│  │     - 用于预约确认页面、取消链接等场景                         │   │
│  │     - 访客通过邮件中的链接查看预约                             │   │
│  │     - 只返回脱敏的基本信息，不暴露敏感数据                     │   │
│  │     - 通过 bookingUid 的随机性实现安全（难以猜测）            │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  🔒 需登录接口的设计原因：                                           │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  1. 保护隐私和安全                                            │   │
│  │     - 预约列表包含用户的敏感信息                              │   │
│  │     - 嘉宾邮件、联系方式等需要保护                            │   │
│  │                                                                 │   │
│  │  2. 权限控制                                                  │   │
│  │     - 确认/拒绝预约需要验证身份                               │   │
│  │     - 团队预约需要验证角色权限                               │   │
│  │                                                                 │   │
│  │  3. 审计追踪                                                  │   │
│  │     - 登录用户操作可被审计                                   │   │
│  │     - 可追溯谁做了什么操作                                   │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 安全防护措施

```
┌─────────────────────────────────────────────────────────────────────┐
│                    匿名接口的安全防护                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  防护层 1：数据脱敏                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │ bookings.find 返回：                                           │   │
│  │ ✅ 允许返回：id, uid, startTime, endTime, description,        │   │
│  │              status, paid, eventTypeId                        │   │
│  │ ❌ 不返回：attendees（嘉宾信息）, user（组织者信息）,          │   │
│  │         location（位置信息）, responses（表单响应）           │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  防护层 2：UID 追踪                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │ reserveSlot 设置的 Cookie：                                   │   │
│  │ • uid: 随机生成的 UUID（难以猜测）                            │   │
│  │ • SameSite=None: 允许跨域 iframe 访问                        │   │
│  │ • Secure: 仅在 HTTPS 下传输                                   │   │
│  │ • HttpOnly: 无法通过 JavaScript 读取                         │   │
│  │                                                                 │   │
│  │ 作用：                                                         │   │
│  │ - 关联预留时间段和创建预约                                    │   │
│  │ - 防止一个访客占用多个时间段                                  │   │
│  │ - 不用于身份认证，仅用于追踪                                  │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  防护层 3：时间段锁                                                  │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │ selectedSlots 表机制：                                         │   │
│  │ • 预留时间段时创建记录                                        │   │
│  │ • releaseAt 字段指定过期时间（通常 15 分钟）                 │   │
│  │ • 创建预约时自动删除预留记录                                  │   │
│  │                                                                 │   │
│  │ 作用：                                                         │   │
│  │ - 防止重复预订                                                │   │
│  │ - 防止恶意占用大量时间段                                      │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  防护层 4：业务层验证                                                │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │ createBooking 时的验证：                                      │   │
│  │ 1. 验证 uid 与预留记录匹配                                    │   │
│  │ 2. 验证时间段是否仍可用                                       │   │
│  │ 3. 验证事件类型是否存在                                       │   │
│  │ 4. 验证组织者是否在该时间段可用                               │   │
│  │                                                                 │   │
│  │ 不依赖前端状态，服务端做最终判断                              │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 四、postMessage 安全机制（保持原分析）

### 4.1 完整消息格式

#### 父窗口 → Iframe（命令）

```typescript
// 发送方: embed.ts doInIframe()
// 接收方: embed-iframe.ts message listener
{
  originator: "CAL",      // 标识字段
  method: string,         // 命令类型: "ui" | "parentKnowsIframeReady" | "connect"
  arg: unknown            // 命令参数
}
```

#### Iframe → 父窗口（事件）

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

### 4.2 安全校验对照表

| 校验项 | 父窗口（发送） | Iframe（发送） | 父窗口（接收） | Iframe（接收） |
|-------|--------------|---------------|--------------|---------------|
| **targetOrigin 指定** | ❌ `*` | ❌ `*` | N/A | N/A |
| **验证 `e.origin`** | N/A | N/A | ❌ 未验证 | ❌ 未验证 |
| **验证 `e.source`** | N/A | N/A | ❌ 未验证 | ❌ 未验证 |
| **验证消息标识** | ✅ `originator: "CAL"` | ✅ `originator: "CAL"` | ✅ `CAL:` 前缀 | ✅ `originator: "CAL"` |

### 4.3 实际风险评估

```
┌─────────────────────────────────────────────────────────────────────┐
│                    postMessage 风险评估                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ⚠️ 理论风险：                                                      │
│  1. 任何窗口都可以伪造 { originator: "CAL", ... } 格式的消息      │
│  2. 发送时使用 targetOrigin="*"，消息可能被任意 iframe 接收        │
│  3. 接收时不验证 e.origin，任意窗口都可以发送消息                   │
│                                                                      │
│  ✅ 实际风险较低的原因：                                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                                                                 │   │
│  │  1. 可执行的命令有限                                           │   │
│  │     父窗口 → Iframe 的命令：                                   │   │
│  │     • "ui" - 更新 UI 配置（主题、布局等）                     │   │
│  │     • "parentKnowsIframeReady" - 握手确认                    │   │
│  │     • "connect" - 连接预渲染页面                              │   │
│  │     ⚠️ 这些都是 UI 层面的操作，不涉及业务逻辑                 │   │
│  │                                                                 │   │
│  │  2. 事件通知不影响业务逻辑                                     │   │
│  │     Iframe → 父窗口的事件：                                   │   │
│  │     • "linkReady" - 链接准备就绪                              │   │
│  │     • "bookingSuccessful" - 预约成功                          │   │
│  │     • "__dimensionChanged" - 尺寸变化                         │   │
│  │     ⚠️ 这些是通知性质的事件，用于更新 UI 状态                  │   │
│  │                                                                 │   │
│  │  3. 关键操作在服务端处理                                       │   │
│  │     创建预约、确认预约等关键操作：                            │   │
│  │     • 不在 postMessage 中传递                                 │   │
│  │     • 通过 tRPC API 直接调用服务端                            │   │
│  │     • 服务端有独立的验证逻辑                                  │   │
│  │                                                                 │   │
│  │  4. uid cookie 有 HttpOnly 保护                               │   │
│  │     • 无法通过 JavaScript 读取                                │   │
│  │     • 即使 postMessage 被劫持，也无法获取 uid                 │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  🔒 结论：postMessage 的安全风险在可控范围内                        │
│       - 主要用于 UI 交互和状态通知                                  │
│       - 不涉及敏感数据或关键业务操作                                │
│       - 关键操作依赖服务端验证                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、完整架构概览（修正版）

```
┌─────────────────────────────────────────────────────────────────────┐
│                    外部网站嵌入预约组件架构                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    外部网站 (example.com)                    │   │
│  │  ┌───────────────────────────────────────────────────────┐  │   │
│  │  │              Embed 脚本 (embed.ts)                    │  │   │
│  │  │  • Cal API: inline(), modal(), floatingButton()      │  │   │
│  │  │  • doInIframe(): 发送命令到 iframe                    │  │   │
│  │  │  ⚠️ targetOrigin="*"，无来源验证                      │  │   │
│  │  └───────────────────────────────────────────────────────┘  │   │
│  │                            │                                  │   │
│  │                            ▼ postMessage()                    │   │
│  │  ┌───────────────────────────────────────────────────────┐  │   │
│  │  │       <iframe src="cal.com/username/30min/embed">    │  │   │
│  │  │  ┌─────────────────────────────────────────────────┐  │  │   │
│  │  │  │           公开页面 (Cal.com 域名)               │  │  │   │
│  │  │  │  • embed-iframe.ts: 消息监听                    │  │  │   │
│  │  │  │    ⚠️ 仅校验 originator: "CAL"                  │  │  │   │
│  │  │  │  • tRPC Client: 调用 Booking API                │  │  │   │
│  │  │  └─────────────────────────────────────────────────┘  │  │   │
│  │  └───────────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                            │                                         │
│                            ▼ HTTP/HTTPS                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Cal.com 后端服务                          │   │
│  │                                                                 │   │
│  │  🔓 匿名 API 端点（publicProcedure）                         │   │
│  │  ┌───────────────────────────────────────────────────────┐  │   │
│  │  │  /api/trpc/slots/                                       │  │   │
│  │  │  • slots.getSchedule()     - 获取可用时间段            │  │   │
│  │  │  • slots.reserveSlot()     - 预留时间段               │  │   │
│  │  │  • slots.isAvailable()     - 检查可用性               │  │   │
│  │  │  • slots.removeSelectedSlotMark() - 移除预留           │  │   │
│  │  ├───────────────────────────────────────────────────────┤  │   │
│  │  │  /api/trpc/public/                                      │  │   │
│  │  │  • publicViewer.event()   - 获取公开事件信息          │  │   │
│  │  │  • publicViewer.submitRating() - 提交评价             │  │   │
│  │  ├───────────────────────────────────────────────────────┤  │   │
│  │  │  /api/trpc/bookings.find                               │  │   │
│  │  │  • bookings.find()        - 通过 uid 查找预约         │  │   │
│  │  │                         ⚠️ 仅返回脱敏数据              │  │   │
│  │  └───────────────────────────────────────────────────────┘  │   │
│  │                                                                 │   │
│  │  🔒 需登录 API 端点（authedProcedure）                       │   │
│  │  ┌───────────────────────────────────────────────────────┐  │   │
│  │  │  /api/trpc/bookings.xxx                                 │  │   │
│  │  │  • bookings.get()           - 获取预约列表             │  │   │
│  │  │  • bookings.confirm()       - 确认/拒绝预约           │  │   │
│  │  │  • bookings.requestReschedule() - 申请改期            │  │   │
│  │  │  • bookings.getBookingDetails() - 获取预约详情        │  │   │
│  │  │  • ... 其他 bookings 接口                              │  │   │
│  │  └───────────────────────────────────────────────────────┘  │   │
│  │                                                                 │   │
│  │  🔓 内部服务调用（非 tRPC 端点）                              │   │
│  │  ┌───────────────────────────────────────────────────────┐  │   │
│  │  │  createBooking()          - 创建预约                   │  │   │
│  │  │  位置: packages/features/bookings/lib/handleNewBooking/│  │   │
│  │  │  ⚠️ 业务层直接调用，通过 uid cookie 追踪               │  │   │
│  │  └───────────────────────────────────────────────────────┘  │   │
│  │                                                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 六、关键文件索引

### 6.1 认证相关文件

| 文件路径 | 职责 | 关键内容 |
|---------|------|---------|
| `packages/trpc/server/procedures/publicProcedure.ts` | 公开 procedure 定义 | 无认证 middleware |
| `packages/trpc/server/procedures/authedProcedure.ts` | 认证 procedure 定义 | 使用 `isAuthed` middleware |
| `packages/trpc/server/middlewares/sessionMiddleware.ts` | 认证 middleware | `isAuthed` 检查 session/user |

### 6.2 路由定义文件

| 文件路径 | 职责 | 关键内容 |
|---------|------|---------|
| `packages/trpc/server/routers/viewer/slots/_router.tsx` | 插槽路由 | ⚠️ 使用 `publicProcedure`，完全匿名 |
| `packages/trpc/server/routers/publicViewer/_router.tsx` | 公开路由 | `publicProcedure`，匿名访问 |
| `packages/trpc/server/routers/viewer/bookings/_router.tsx` | 预约路由 | ⚠️ `find` 匿名，其他需登录 |

### 6.3 Handler 实现文件

| 文件路径 | 职责 | 认证要求 |
|---------|------|---------|
| `packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts` | 预留时间段 | 🔓 无需 |
| `packages/trpc/server/routers/viewer/bookings/find.handler.ts` | 查找预约 | 🔓 无需（脱敏） |
| `packages/trpc/server/routers/viewer/bookings/get.handler.ts` | 获取预约列表 | 🔒 需登录 |
| `packages/trpc/server/routers/viewer/bookings/confirm.handler.ts` | 确认预约 | 🔒 需登录 |
| `packages/features/bookings/lib/handleNewBooking/createBooking.ts` | 创建预约核心逻辑 | 🔓 无需（uid 追踪） |

---

## 七、总结

### 7.1 核心结论（统一口径）

```
┌─────────────────────────────────────────────────────────────────────┐
│                    核心结论摘要                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  关于 API 认证边界：                                                 │
│  ✅ 访客预约全流程匿名，无需 Cal.com 账号                          │
│  ✅ slotsRouter 完全匿名（虽然在 viewer/slots/ 目录下）           │
│  ✅ bookingsRouter.find 是唯一的匿名预约查询接口                   │
│  ✅ bookingsRouter.find 只返回脱敏数据，不暴露敏感信息             │
│  ✅ 其他 bookings 接口都需要登录                                    │
│  ✅ createBooking 是内部服务调用，通过 uid cookie 追踪             │
│                                                                      │
│  关于 postMessage 安全：                                            │
│  ⚠️ 两侧都使用 targetOrigin="*"                                    │
│  ⚠️ 两侧都只验证字符串标识（originator: "CAL" 或 CAL: 前缀）      │
│  ⚠️ 两侧都不验证 e.origin 或 e.source                              │
│  ✅ 但实际风险较低：                                                │
│     - 可执行的命令有限（都是 UI 配置）                             │
│     - 关键操作不在 postMessage 中处理                              │
│     - uid cookie 有 HttpOnly 保护                                  │
│     - 服务端有独立验证逻辑                                          │
│                                                                      │
│  设计理念：                                                          │
│  🎯 访客预约零门槛，组织者操作高安全                               │
│  🎯 公开信息公开访问，敏感信息需认证                               │
│  🎯 前端 UI 松耦合，服务端验证强耦合                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 预约流程接口速查表

| 步骤 | 接口 | 认证 | 关键数据 |
|------|------|------|---------|
| 1. 查看事件 | `publicViewer.event()` | 🔓 无需 | 事件详情 |
| 2. 查看时间 | `slots.getSchedule()` | 🔓 无需 | 可用时间段 |
| 3. 预留时间 | `slots.reserveSlot()` | 🔓 无需 | uid cookie |
| 4. 创建预约 | `createBooking()` | 🔓 无需 | Booking 对象 |
| 5. 查看预约 | `bookings.find()` | 🔓 无需 | 脱敏基本信息 |
| - 确认预约 | `bookings.confirm()` | 🔒 需登录 | 确认结果 |
| - 查看列表 | `bookings.get()` | 🔒 需登录 | 完整预约列表 |
