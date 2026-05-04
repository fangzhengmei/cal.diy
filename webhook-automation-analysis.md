# Webhook 系统架构与自动化工具接入分析

## 目录

1. [概述](#1-概述)
2. [订阅机制 - 完整生命周期](#2-订阅机制---完整生命周期)
3. [投递机制](#3-投递机制)
4. [重试机制 - 失败终止与状态回写](#4-重试机制---失败终止与状态回写)
5. [外部自动化工具接入 - Zapier vs Make 对比](#5-外部自动化工具接入---zapier-vs-make-对比)
6. [关键文件位置](#6-关键文件位置)
7. [数据流图](#7-数据流图)
8. [版本控制](#8-版本控制)
9. [安全考虑](#9-安全考虑)
10. [监控与日志](#10-监控与日志)
11. [扩展指南](#11-扩展指南)

---

## 1. 概述

Cal.diy 的 Webhook 系统采用**生产者-消费者架构**，结合 Trigger.dev 任务队列实现可靠的事件通知机制。系统支持多种触发事件类型，并提供了与外部自动化工具（Zapier、Make）的集成能力。

### 核心设计原则

- **关注点分离**: 生产者负责入队，消费者负责处理
- **版本控制**: Payload 格式支持版本化
- **可靠性**: 内置重试机制和错误处理
- **安全性**: HMAC-SHA256 签名验证，SSRF 保护

---

## 2. 订阅机制 - 完整生命周期

### 2.1 数据模型

#### Webhook 表结构 (`packages/prisma/schema.prisma:1142`)

```prisma
model Webhook {
  id                    String                     @id @unique
  userId                Int?
  teamId                Int?
  eventTypeId           Int?
  platformOAuthClientId String?
  subscriberUrl         String                    // 接收 webhook 的 URL
  payloadTemplate       String?                    // 自定义 payload 模板
  createdAt             DateTime                   @default(now())
  active                Boolean                    @default(true)  // 启用状态
  eventTriggers         WebhookTriggerEvents[]    // 触发事件数组
  appId                 String?                    // 关联应用 ID (zapier/make)
  secret                String?                    // 签名密钥
  platform              Boolean                    @default(false)  // 是否平台级
  time                  Int?                       // 定时触发时间 (no-show)
  timeUnit              TimeUnit?                  // 时间单位
  version               String                     @default("2021-10-20")
  
  // 关系
  user                  User?                      @relation(fields: [userId], references: [id], onDelete: Cascade)
  team                  Team?                      @relation(fields: [teamId], references: [id], onDelete: Cascade)
  eventType             EventType?                 @relation(fields: [eventTypeId], references: [id], onDelete: Cascade)
  scheduledTriggers     WebhookScheduledTriggers[]
  
  @@unique([userId, subscriberUrl], name: "courseIdentifier")
  @@unique([platformOAuthClientId, subscriberUrl], name: "oauthclientwebhook")
  @@index([active])
}
```

#### WebhookScheduledTriggers 表 (`packages/prisma/schema.prisma:1313`)

用于定时触发的 webhook（如 MEETING_STARTED, MEETING_ENDED）：

```prisma
model WebhookScheduledTriggers {
  id            Int       @id @default(autoincrement())
  jobName       String?   // 已废弃，使用 webhook 和 booking 关系
  subscriberUrl String
  payload       String    // 序列化的任务数据
  startAfter    DateTime  // 触发时间
  retryCount    Int       @default(0)  // 重试计数
  createdAt     DateTime? @default(now())
  appId         String?
  webhookId     String?
  webhook       Webhook?  @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  bookingId     Int?
  booking       Booking?  @relation(fields: [bookingId], references: [id], onDelete: Cascade)
}
```

### 2.2 触发事件类型

支持的 `WebhookTriggerEvents` 枚举 (`packages/prisma/schema.prisma:1120`)：

| 事件类型 | 描述 | 触发方式 |
|---------|------|---------|
| `BOOKING_CREATED` | 预订创建 | 即时 |
| `BOOKING_CANCELLED` | 预订取消 | 即时 |
| `BOOKING_RESCHEDULED` | 预订重新安排 | 即时 |
| `BOOKING_REQUESTED` | 预订请求（待确认） | 即时 |
| `BOOKING_REJECTED` | 预订被拒绝 | 即时 |
| `BOOKING_PAYMENT_INITIATED` | 支付发起 | 即时 |
| `BOOKING_PAID` | 支付完成 | 即时 |
| `BOOKING_NO_SHOW_UPDATED` | 未到场状态更新 | 即时 |
| `MEETING_STARTED` | 会议开始 | 定时 |
| `MEETING_ENDED` | 会议结束 | 定时 |
| `RECORDING_READY` | 录制就绪 | 即时 |
| `RECORDING_TRANSCRIPTION_GENERATED` | 转录生成 | 即时 |
| `FORM_SUBMITTED` | 表单提交 | 即时 |
| `OOO_CREATED` | 外出状态创建 | 即时 |
| `AFTER_HOSTS_CAL_VIDEO_NO_SHOW` | 主持人未到场 | 定时 |
| `AFTER_GUESTS_CAL_VIDEO_NO_SHOW` | 嘉宾未到场 | 定时 |
| `DELEGATION_CREDENTIAL_ERROR` | 委派凭证错误 | 即时 |
| `WRONG_ASSIGNMENT_REPORT` | 错误分配报告 | 即时 |

### 2.3 订阅生命周期管理

#### 阶段 1: 创建订阅

**入口**: `packages/trpc/server/routers/viewer/webhook/create.handler.ts`

完整创建流程：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         1. 输入验证                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ SSRF 保护检查 (validateUrlForSSRFSync)                              │ │
│  │ - 禁止内网 IP (127.0.0.1, 192.168.x.x, 10.x.x.x, 172.16-31.x.x)  │ │
│  │ - 禁止 localhost                                                      │ │
│  │ - 禁止私有/保留地址范围                                                │ │
│  │ - URL 格式验证 (z.string().url())                                    │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         2. 权限检查                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ Platform 级 webhook: 需要 ADMIN 角色                                 │ │
│  │ Managed EventType: 检查 lockedFields.webhooks                        │ │
│  │ 普通订阅: 需要用户认证                                                 │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         3. 数据库创建                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ prisma.webhook.create({                                              │ │
│  │   id: v4(),                                                           │ │
│  │   subscriberUrl: input.subscriberUrl,                                │ │
│  │   eventTriggers: input.eventTriggers,                                │ │
│  │   active: input.active,                                               │ │
│  │   payloadTemplate: input.payloadTemplate,                            │ │
│  │   secret: input.secret,                                               │ │
│  │   time: input.time,                                                   │ │
│  │   timeUnit: input.timeUnit,                                           │ │
│  │   version: input.version,                                             │ │
│  │   appId: input.appId,                                                 │ │
│  │   // 关联                                                              │ │
│  │   user: input.eventTypeId ? undefined : { connect: { id: user.id } },│ │
│  │   eventTypeId: input.eventTypeId,                                     │ │
│  │   teamId: input.teamId,                                               │ │
│  │ })                                                                     │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    4. 定时触发安排 (如适用)                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ updateTriggerForExistingBookings(                                    │ │
│  │   newWebhook,                                                         │ │
│  │   [],                          // 旧触发事件 (空，因为是新建)         │ │
│  │   newWebhook.eventTriggers    // 新触发事件                          │ │
│  │ )                                                                      │ │
│  │                                                                        │ │
│  │ 对于 MEETING_STARTED / MEETING_ENDED:                                │ │
│  │ - 查询现有预订 (startTime > now(), status = ACCEPTED)                │ │
│  │ - 为每个预订创建 WebhookScheduledTriggers 记录                        │ │
│  │ - 计算 startAfter = booking.startTime (或 endTime)                   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

**核心代码** (`create.handler.ts`):

```typescript
export const createHandler = async ({ ctx, input }: CreateOptions) => {
  const { user } = ctx;

  // 1. SSRF 保护
  const validation = validateUrlForSSRFSync(input.subscriberUrl);
  if (!validation.isValid) {
    throw new TRPCError({
      code: "BAD_REQUEST",
      message: `Webhook URL is not allowed: ${validation.error}`,
    });
  }

  // 2. 权限检查
  if (input.platform && user.role !== "ADMIN") {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }

  // Managed EventType 锁定检查
  if (input.eventTypeId) {
    const parentManagedEvt = await prisma.eventType.findFirst({
      where: { id: input.eventTypeId, parentId: { not: null } },
      select: { parentId: true, metadata: true },
    });

    if (parentManagedEvt?.parentId) {
      const isLocked = !EventTypeMetaDataSchema.parse(parentManagedEvt.metadata)
        ?.managedEventConfig?.unlockedFields?.webhooks;
      if (isLocked) {
        throw new TRPCError({ code: "UNAUTHORIZED" });
      }
    }
  }

  // 3. 构建数据
  const webhookData: Prisma.WebhookCreateInput = {
    id: v4(),
    ...inputWithoutWebhookId,
  };
  
  if (!input.platform && !input.eventTypeId) {
    webhookData.user = { connect: { id: user.id } };
  }

  // 4. 数据库创建
  let newWebhook: Webhook;
  try {
    newWebhook = await prisma.webhook.create({ data: webhookData });
  } catch (error) {
    throw new TRPCError({ code: "INTERNAL_SERVER_ERROR", message: "Failed to create webhook" });
  }

  // 5. 安排定时触发 (对于 MEETING_STARTED/ENDED)
  await updateTriggerForExistingBookings(newWebhook, [], newWebhook.eventTriggers);

  return newWebhook;
};
```

---

#### 阶段 2: 更新订阅

**入口**: `packages/trpc/server/routers/viewer/webhook/edit.handler.ts`

更新流程：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         1. 查找现有订阅                                     │
│  prisma.webhook.findUnique({ where: { id: input.id } })                  │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         2. 条件验证                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ - Platform webhook: 需要 ADMIN 角色                                  │ │
│  │ - URL 变更: 重新进行 SSRF 检查                                        │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         3. 数据库更新                                      │
│  prisma.webhook.update({                                                  │
│    where: { id },                                                         │
│    data: {                                                                 │
│      subscriberUrl, eventTriggers, active,                                │
│      payloadTemplate, secret, time, timeUnit, version                    │
│    }                                                                       │
│  })                                                                        │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    4. 状态变更处理 (active 标志)                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 情况 A: active = true (启用)                                          │ │
│  │ ─────────────────────────────────────                                  │ │
│  │ updateTriggerForExistingBookings(                                     │ │
│  │   webhook,                                                             │ │
│  │   activeTriggersBefore,  // 旧触发事件 (webhook.active 时的)         │ │
│  │   updatedEventTriggers    // 新触发事件                                │ │
│  │ )                                                                       │ │
│  │                                                                         │ │
│  │ - 计算 addedEventTriggers: 新增的定时触发事件                          │ │
│  │ - 计算 removedEventTriggers: 移除的定时触发事件                        │ │
│  │ - 为新增事件创建 WebhookScheduledTriggers                              │ │
│  │ - 为移除事件删除 WebhookScheduledTriggers                              │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 情况 B: active = false (禁用，且之前为 active)                         │ │
│  │ ────────────────────────────────────────────────                      │ │
│  │ 1. 取消 NoShow 定时任务                                                │ │
│  │    cancelNoShowTasksForBooking({ webhook: {...} })                    │ │
│  │                                                                         │ │
│  │ 2. 删除 WebhookScheduledTriggers 记录                                  │ │
│  │    deleteWebhookScheduledTriggers({ webhookId: webhook.id })          │ │
│  │                                                                         │ │
│  │ 3. 任务队列清理 (通过任务系统)                                          │ │
│  │    - 对于 Trigger.dev 任务: 任务执行时检查 webhook.active             │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

**核心代码** (`edit.handler.ts`):

```typescript
export const editHandler = async ({ input, ctx }: EditOptions) => {
  const { id, ...data } = input;

  // 1. 查找现有订阅
  const webhook = await prisma.webhook.findUnique({ where: { id } });
  if (!webhook) return null;

  // 2. URL 变更时重新验证
  if (data.subscriberUrl && data.subscriberUrl !== webhook.subscriberUrl) {
    const validation = validateUrlForSSRFSync(data.subscriberUrl);
    if (!validation.isValid) {
      throw new TRPCError({
        code: "BAD_REQUEST",
        message: `Webhook URL is not allowed: ${validation.error}`,
      });
    }
  }

  // 3. Platform 权限检查
  if (webhook.platform) {
    const { user } = ctx;
    if (user?.role !== "ADMIN") {
      throw new TRPCError({ code: "UNAUTHORIZED" });
    }
  }

  // 4. 更新数据库
  const updatedWebhook = await prisma.webhook.update({
    where: { id },
    data: {
      ...data,
      time: data.time ?? null,
      timeUnit: data.timeUnit ?? null,
    },
  });

  // 5. 状态变更处理
  if (data.active) {
    // 启用: 更新定时触发
    const activeTriggersBefore = webhook.active ? webhook.eventTriggers : [];
    await updateTriggerForExistingBookings(webhook, activeTriggersBefore, updatedWebhook.eventTriggers);
  } else if (!data.active && webhook.active) {
    // 禁用: 清理定时任务
    await cancelNoShowTasksForBooking({
      webhook: {
        id: webhook.id,
        userId: webhook.userId,
        teamId: webhook.teamId,
        eventTypeId: webhook.eventTypeId,
      },
    });
    await deleteWebhookScheduledTriggers({ webhookId: webhook.id });
  }

  return updatedWebhook;
};
```

---

#### 阶段 3: 删除订阅

**入口**: `packages/trpc/server/routers/viewer/webhook/delete.handler.ts`

删除流程：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         1. 权限过滤                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 构建 WHERE 条件:                                                      │ │
│  │ - 有 eventTypeId: AND { eventTypeId }                                │ │
│  │ - ADMIN: OR [{ platform: true }, { userId }]                         │ │
│  │ - 普通用户: AND { userId }                                            │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         2. 查找并删除                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 1. prisma.webhook.findFirst({ where })                               │ │
│  │ 2. 找到后: prisma.webhook.delete({ where: { id } })                  │ │
│  │    - ON DELETE CASCADE 自动删除 WebhookScheduledTriggers            │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    3. 清理定时触发 (显式调用)                              │
│  updateTriggerForExistingBookings(                                       │
│    webhookToDelete,                                                       │
│    webhookToDelete.eventTriggers,  // 旧触发事件                         │
│    []                                    // 新触发事件 (空)                │
│  )                                                                         │
│                                                                           │
│  效果:                                                                     │
│  - 计算 removedEventTriggers = 所有旧事件                                 │
│  - 删除相关的 WebhookScheduledTriggers 记录                               │
│  - 取消 NoShow 任务 (通过 tasker.cancelWithReference)                   │
└──────────────────────────────────────────────────────────────────────────┘
```

**核心代码** (`delete.handler.ts`):

```typescript
export const deleteHandler = async ({ ctx, input }: DeleteOptions) => {
  const { id } = input;

  // 1. 构建权限过滤条件
  const where: Prisma.WebhookWhereInput = { AND: [{ id: id }] };
  
  if (Array.isArray(where.AND)) {
    if (input.eventTypeId) {
      where.AND.push({ eventTypeId: input.eventTypeId });
    } else if (ctx.user.role === "ADMIN") {
      where.AND.push({ OR: [{ platform: true }, { userId: ctx.user.id }] });
    } else {
      where.AND.push({ userId: ctx.user.id });
    }
  }

  // 2. 查找并删除
  const webhookToDelete = await prisma.webhook.findFirst({ where });

  if (webhookToDelete) {
    // 数据库删除 (CASCADE 自动清理 WebhookScheduledTriggers)
    await prisma.webhook.delete({ where: { id: webhookToDelete.id } });

    // 3. 显式清理定时触发安排
    await updateTriggerForExistingBookings(
      webhookToDelete,
      webhookToDelete.eventTriggers,  // 旧事件
      []                               // 新事件 (空)
    );
  }

  return { id };
};
```

---

#### 阶段 4: 失效处理 (主动/被动)

##### 4.1 主动失效 (用户操作)

通过 `edit` 接口设置 `active = false`:

```typescript
// 处理流程 (edit.handler.ts:66-76)
else if (!data.active && webhook.active) {
  // 1. 取消 NoShow 定时任务
  await cancelNoShowTasksForBooking({
    webhook: {
      id: webhook.id,
      userId: webhook.userId,
      teamId: webhook.teamId,
      eventTypeId: webhook.eventTypeId,
    },
  });
  
  // 2. 删除 WebhookScheduledTriggers 记录
  await deleteWebhookScheduledTriggers({ webhookId: webhook.id });
}
```

##### 4.2 被动失效 (级联删除)

通过数据库 `ON DELETE CASCADE`:

| 关联表 | 删除触发 | 清理内容 |
|--------|---------|---------|
| `User` | 用户删除 | `userId` 关联的 webhook 被删除 |
| `Team` | 团队删除 | `teamId` 关联的 webhook 被删除 |
| `EventType` | 事件类型删除 | `eventTypeId` 关联的 webhook 被删除 |
| `PlatformOAuthClient` | OAuth 客户端删除 | `platformOAuthClientId` 关联的 webhook 被删除 |

**级联关系** (`schema.prisma`):

```prisma
model Webhook {
  user        User?        @relation(fields: [userId], references: [id], onDelete: Cascade)
  team        Team?        @relation(fields: [teamId], references: [id], onDelete: Cascade)
  eventType   EventType?   @relation(fields: [eventTypeId], references: [id], onDelete: Cascade)
  // ...
}

model WebhookScheduledTriggers {
  webhook     Webhook?     @relation(fields: [webhookId], references: [id], onDelete: Cascade)
  booking     Booking?     @relation(fields: [bookingId], references: [id], onDelete: Cascade)
  // ...
}
```

##### 4.3 查询时过滤

`WebhookRepository.getSubscribers` 只查询 `active = true` 的订阅：

```typescript
// packages/features/webhooks/lib/repository/WebhookRepository.ts:82
// 所有 UNION 分支都包含:
WHERE active = true 
  AND ...
```

---

### 2.4 订阅者查询机制

`WebhookRepository.getSubscribers` (`packages/features/webhooks/lib/repository/WebhookRepository.ts:82`) 使用 **UNION ALL** 查询多个来源的订阅者，按优先级排序：

```typescript
// 优先级 1: Platform webhooks (平台级，最高优先级)
// 条件: platform = true

// 优先级 2: User-specific webhooks (用户级)
// 条件: userId = ? AND platform = false

// 优先级 3: Event type webhooks (事件类型级)
// 条件: eventTypeId = ? AND platform = false

// 优先级 4: Parent event type webhooks (父事件类型级)
// 条件: eventTypeId = managedParentEventTypeId AND platform = false

// 优先级 5: Team webhooks (团队级)
// 条件: teamId IN (teamIds) AND platform = false

// 优先级 6: OAuth client webhooks (OAuth 客户端级)
// 条件: platformOAuthClientId = ? AND platform = false
```

**SQL 实现**:

```sql
-- 使用 UNION ALL 而非 OR，每个分支可使用独立索引
SELECT id, "subscriberUrl", ..., 1 as priority FROM "Webhook"
WHERE active = true AND platform = true AND ? = ANY("eventTriggers")

UNION ALL

SELECT id, "subscriberUrl", ..., 2 as priority FROM "Webhook"
WHERE active = true AND ? IS NOT NULL AND "userId" = ? 
  AND ? = ANY("eventTriggers") AND platform = false

-- ... 其他分支

ORDER BY priority, id
```

查询上下文 (`SubscriberContext`) 包含：
- `triggerEvent`: 触发事件类型
- `userId`: 用户 ID
- `eventTypeId`: 事件类型 ID
- `teamId`: 团队 ID（支持数组）
- `orgId`: 组织 ID
- `oAuthClientId`: OAuth 客户端 ID

---

### 2.5 订阅生命周期状态图

```
                        ┌─────────────────────────────────────────────────┐
                        │              新订阅创建                           │
                        │  prisma.webhook.create()                         │
                        │  → active: true (默认)                           │
                        └─────────────────────┬───────────────────────────┘
                                              │
                                              ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                         ACTIVE (正常状态)                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐│
│  │ 操作:                                                                   ││
│  │ - 事件触发时被查询 (active = true)                                      ││
│  │ - 可通过 edit 更新配置                                                   ││
│  │ - 定时触发事件会创建 WebhookScheduledTriggers                           ││
│  └───────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
   ┌────────────────┐ ┌────────────────┐ ┌────────────────┐
   │   用户禁用     │ │   级联删除     │ │   预订结束     │
   │  active=false  │ │ ON DELETE CAS. │ │(定时触发清理)  │
   └───────┬────────┘ └───────┬────────┘ └───────┬────────┘
           │                   │                   │
           │                   │                   │
           ▼                   ▼                   ▼
   ┌─────────────────────────────────────────────────────────┐
   │                    INACTIVE / DELETED                    │
   │                                                           │
   │  INACTIVE (active=false):                                │
   │  - 不再被 getSubscribers 查询                            │
   │  - 已清理 WebhookScheduledTriggers                       │
   │  - 已取消 NoShow 定时任务                                │
   │                                                           │
   │  DELETED:                                                │
   │  - 数据库记录已删除                                       │
   │  - 所有关联记录通过 CASCADE 清理                          │
   └─────────────────────────────────────────────────────────┘
```

---

## 3. 投递机制

### 3.1 架构概览

系统采用**生产者-消费者模式**，通过 Trigger.dev 任务队列实现异步处理：

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────┐
│   事件源        │ ───> │    Producer      │ ───> │  Trigger.dev    │
│ (Booking/Form)  │      │ (轻量级入队)     │      │    任务队列     │
└─────────────────┘      └──────────────────┘      └────────┬────────┘
                                                              │
                              ┌───────────────────────────────┘
                              ▼
                    ┌──────────────────┐
                    │    Consumer      │
                    │  (重量级处理)     │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │DataFetcher│  │Payload   │  │HTTP      │
        │ 获取数据  │  │Builder   │  │Delivery  │
        └──────────┘  │构建payload│  │发送请求  │
                      └──────────┘  └──────────┘
```

### 3.2 生产者 (Producer)

**WebhookTaskerProducerService** (`packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts`) 是轻量级生产者：

- **依赖**: 仅 `WebhookTasker` 和 `Logger`
- **职责**: 构建最小化 payload 并入队
- **设计原则**: 不进行数据库查询或 heavy lifting

核心方法：

```typescript
// 预订相关
queueBookingCreatedWebhook()
queueBookingCancelledWebhook()
queueBookingRescheduledWebhook()
queueBookingRequestedWebhook()
queueBookingRejectedWebhook()
queueBookingNoShowUpdatedWebhook()

// 支付相关
queueBookingPaymentInitiatedWebhook()
queueBookingPaidWebhook()

// 其他
queueFormSubmittedWebhook()
queueRecordingReadyWebhook()
queueOOOCreatedWebhook()
```

**WebhookTaskPayload** 结构 (`packages/features/webhooks/lib/types/webhookTask.ts`)：

```typescript
{
  operationId: string;        // 操作追踪 ID (UUID)
  triggerEvent: WebhookTriggerEvents;
  timestamp: string;          // ISO 时间戳
  metadata?: Record<string, unknown>;
  
  // 根据事件类型的不同字段
  bookingUid?: string;        // 预订 UID
  eventTypeId?: number;
  teamId?: number | null;
  userId?: number;
  orgId?: number;
  oAuthClientId?: string | null;
  // ... 其他事件类型的特定字段
}
```

**入队流程** (`WebhookTaskerProducerService.ts:222`):

```typescript
private async queueTask(operationId: string, taskPayload: WebhookTaskPayload): Promise<void> {
  try {
    const result = await this.deps.webhookTasker.deliverWebhook(taskPayload);
    this.log.debug("Webhook delivery task queued", { operationId, taskId: result.taskId });
  } catch (error) {
    this.log.error("Failed to queue webhook delivery task", {
      operationId,
      error: error instanceof Error ? error.message : String(error),
    });
    throw error;  // 异常向上传播，由调用方处理
  }
}
```

### 3.3 消费者 (Consumer)

**WebhookTaskConsumer** (`packages/features/webhooks/lib/service/WebhookTaskConsumer.ts`) 处理队列中的任务：

处理流程：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    1. 获取 DataFetcher                                    │
│  this.dataFetchers.find((fetcher) => fetcher.canHandle(triggerEvent))    │
│  - BookingWebhookDataFetcher: BOOKING_* 事件                             │
│  - PaymentWebhookDataFetcher: 支付相关事件                                │
│  - FormWebhookDataFetcher: FORM_SUBMITTED                                │
│  - RecordingWebhookDataFetcher: 录制相关事件                              │
│  - OOOWebhookDataFetcher: OOO_CREATED                                     │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    2. 获取订阅者上下文                                      │
│  subscriberContext = fetcher.getSubscriberContext(payload)               │
│  包含: userId, eventTypeId, teamId, orgId, oAuthClientId                 │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    3. 查询订阅者                                           │
│  subscribers = await webhookRepository.getSubscribers(subscriberContext)  │
│  - 使用 UNION ALL 查询多个来源                                            │
│  - 只返回 active = true 的订阅                                            │
│  - 按优先级去重                                                            │
│                                                                           │
│  如果 subscribers.length === 0: 直接返回，无需后续处理                    │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    4. 获取事件数据                                         │
│  eventData = await fetcher.fetchEventData(payload)                       │
│  - 从数据库查询完整数据                                                    │
│  - 构建 CalendarEvent (用于 email/sms 相同的 payload 格式)               │
│  - 包含 booking, eventType, calendarEvent                                 │
│                                                                           │
│  如果 eventData 为 null: 返回 (数据不存在或已删除)                        │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    5. 构建 DTO 和 Payload                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ dto = this.buildDTO(eventData, payload)                              │ │
│  │ → 映射到 WebhookEventDTO 结构                                         │ │
│  │                                                                       │ │
│  │ builder = payloadBuilderFactory.getBuilder(version, triggerEvent)   │ │
│  │ → 获取版本化的 payload builder                                         │ │
│  │                                                                       │ │
│  │ webhookPayload = builder.build(dto)                                   │ │
│  │ → 构建最终的 webhook payload                                           │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    6. 发送到所有订阅者                                     │
│  await webhookService.processWebhooks(trigger, webhookPayload, subscribers) │
│  → 见 3.6 节详细流程                                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

**核心代码** (`WebhookTaskConsumer.ts:33`):

```typescript
async processWebhookTask(payload: WebhookTaskPayload, taskId: string): Promise<void> {
  this.log.debug("Processing webhook delivery task", {
    operationId: payload.operationId,
    taskId,
    triggerEvent: payload.triggerEvent,
  });

  try {
    // 1. 获取 DataFetcher
    const fetcher = this.getDataFetcher(payload.triggerEvent);
    if (!fetcher) {
      this.log.error("No data fetcher found for trigger event", {
        operationId: payload.operationId,
        triggerEvent: payload.triggerEvent,
      });
      throw new Error(`No data fetcher registered for trigger event: ${payload.triggerEvent}`);
    }

    // 2. 获取订阅者
    const subscriberContext = fetcher.getSubscriberContext(payload);
    const subscribers = await this.webhookRepository.getSubscribers(subscriberContext);
    
    if (subscribers.length === 0) {
      this.log.debug("No webhook subscribers found", { operationId: payload.operationId });
      return;
    }

    // 3. 获取事件数据
    const eventData = await fetcher.fetchEventData(payload);
    if (!eventData) {
      this.log.warn("Event data not found", {
        operationId: payload.operationId,
        triggerEvent: payload.triggerEvent,
      });
      return;
    }

    // 4. 发送 webhooks
    await this.sendWebhooksToSubscribers(subscribers, eventData, payload);

    this.log.debug("Webhook delivery task completed", {
      operationId: payload.operationId,
      subscriberCount: subscribers.length,
    });
  } catch (error) {
    this.log.error("Failed to process webhook delivery task", {
      operationId: payload.operationId,
      taskId,
      error: error instanceof Error ? error.message : String(error),
    });
    throw error;  // 抛出异常，触发 Trigger.dev 重试
  }
}
```

### 3.4 DataFetchers

每个事件类型有对应的 DataFetcher：

| DataFetcher | 处理事件 | 位置 |
|------------|---------|------|
| `BookingWebhookDataFetcher` | BOOKING_* 事件 | `packages/features/webhooks/lib/service/data-fetchers/BookingWebhookDataFetcher.ts` |
| `PaymentWebhookDataFetcher` | 支付相关事件 | `packages/features/webhooks/lib/service/data-fetchers/PaymentWebhookDataFetcher.ts` |
| `FormWebhookDataFetcher` | FORM_SUBMITTED | `packages/features/webhooks/lib/service/data-fetchers/FormWebhookDataFetcher.ts` |
| `RecordingWebhookDataFetcher` | 录制相关事件 | `packages/features/webhooks/lib/service/data-fetchers/RecordingWebhookDataFetcher.ts` |
| `OOOWebhookDataFetcher` | OOO_CREATED | `packages/features/webhooks/lib/service/data-fetchers/OOOWebhookDataFetcher.ts` |

### 3.5 Tasker 实现

系统支持两种执行模式：

#### 1. WebhookTriggerTasker (生产环境)

```typescript
// packages/features/webhooks/lib/tasker/WebhookTriggerTasker.ts
class WebhookTriggerTasker implements IWebhookTasker {
  async deliverWebhook(payload: WebhookTaskPayload): Promise<WebhookDeliveryResult> {
    const { deliverWebhook } = await import("./trigger/deliver-webhook");
    const handle = await deliverWebhook.trigger(payload);
    return { taskId: handle.id };
  }
}
```

#### 2. WebhookSyncTasker (E2E 测试)

```typescript
// packages/features/webhooks/lib/tasker/WebhookSyncTasker.ts
class WebhookSyncTasker implements IWebhookTasker {
  async deliverWebhook(payload: WebhookTaskPayload): Promise<WebhookDeliveryResult> {
    const taskId = `sync_${nanoid(10)}`;
    await this.deps.webhookTaskConsumer.processWebhookTask(payload, taskId);
    return { taskId };
  }
}
```

### 3.6 HTTP 投递与订阅者隔离

**WebhookService.processWebhooks** (`packages/features/webhooks/lib/service/WebhookService.ts:135`) 处理实际的投递：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    遍历所有订阅者                                          │
│  for (const subscriber of subscribers) { ... }                           │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    对每个订阅者并行发送                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ Promise.allSettled([                                                  │ │
│  │   sendWebhook(trigger, payload, subscriber1),                         │ │
│  │   sendWebhook(trigger, payload, subscriber2),                         │ │
│  │   ...                                                                  │ │
│  │ ])                                                                     │ │
│  │                                                                       │ │
│  │ 关键点: 使用 allSettled 而非 all，确保单个订阅者失败不影响其他订阅者   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    sendWebhook 内部逻辑                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ if (process.env.TASKER_ENABLE_WEBHOOKS === "1") {                   │ │
│  │   // 调度到任务队列 (用于旧版任务系统)                                │ │
│  │   return await scheduleWebhook(trigger, payload, subscriber);        │ │
│  │ } else {                                                              │ │
│  │   // 直接发送 (当前默认行为)                                           │ │
│  │   return await sendWebhookDirectly(trigger, payload, subscriber);    │ │
│  │ }                                                                      │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    sendWebhookDirectly 实现                                │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 1. 构建 body:                                                         │ │
│  │    JSON.stringify({                                                    │ │
│  │      triggerEvent: trigger,                                            │ │
│  │      createdAt: payload.createdAt,                                     │ │
│  │      payload: payload.payload,                                          │ │
│  │    })                                                                  │ │
│  │                                                                       │ │
│  │ 2. 构建签名:                                                          │ │
│  │    createHmac("sha256", subscriber.secret).update(body).digest("hex") │ │
│  │                                                                       │ │
│  │ 3. 发送 POST 请求:                                                    │ │
│  │    fetch(subscriberUrl, {                                             │ │
│  │      method: "POST",                                                  │ │
│  │      headers: {                                                        │ │
│  │        "Content-Type": contentType,                                   │ │
│  │        "X-Cal-Signature-256": signature,                              │ │
│  │        "X-Cal-Webhook-Version": subscriber.version,                   │ │
│  │      },                                                                │ │
│  │      redirect: "manual",  // 禁止重定向                               │ │
│  │      body,                                                             │ │
│  │    })                                                                  │ │
│  │                                                                       │ │
│  │ 4. 返回结果:                                                          │ │
│  │    { ok: response.ok, status: response.status }                       │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    结果统计与日志                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ const successCount = results.filter(r => r.status === "fulfilled").length │
│  │ const failureCount = results.filter(r => r.status === "rejected").length │
│  │                                                                       │ │
│  │ this.log.info(`Webhook processing completed for ${trigger}`, {       │ │
│  │   totalSubscribers: subscribers.length,                               │ │
│  │   successful: successCount,                                           │ │
│  │   failed: failureCount,                                               │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 记录每个失败的订阅者                                                │ │
│  │ results.forEach((result, index) => {                                  │ │
│  │   if (result.status === "rejected") {                                 │ │
│  │     this.log.error(`Webhook processing failed for subscriber`, {      │ │
│  │       trigger, webhookId, subscriberUrl, error: result.reason         │ │
│  │     });                                                                │ │
│  │   }                                                                    │ │
│  │ });                                                                    │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

**核心代码** (`WebhookService.ts:135`):

```typescript
async processWebhooks(
  trigger: WebhookTriggerEvents,
  payload: WebhookPayload,
  subscribers: WebhookSubscriber[]
): Promise<void> {
  if (subscribers.length === 0) {
    this.log.debug("No subscribers to process for trigger:", { trigger });
    return;
  }

  // 对每个订阅者并行发送
  const promises = subscribers.map(async (subscriber) => {
    try {
      this.log.debug("Processing webhook", {
        trigger,
        webhookId: subscriber.id,
        subscriberUrl: subscriber.subscriberUrl,
      });

      const result = await this.sendWebhook(trigger, payload, subscriber);

      if (result.ok) {
        this.log.debug(`Webhook sent successfully`, {
          trigger,
          webhookId: subscriber.id,
          statusCode: result.status,
        });
      } else {
        this.log.error(`Webhook failed`, {
          error: result.message,
          trigger,
          webhookId: subscriber.id,
          statusCode: result.status,
        });
      }
    } catch (err) {
      this.log.error("Error sending webhook", {
        error: err instanceof Error ? err.message : String(err),
        trigger,
        webhookId: subscriber.id,
      });
      // Re-throw 确保 Promise.allSettled 捕获失败
      throw err;
    }
  });

  // 关键点: 使用 allSettled 隔离单个订阅者的失败
  const results = await Promise.allSettled(promises);

  // 统计
  const successCount = results.filter((result) => result.status === "fulfilled").length;
  const failureCount = results.filter((result) => result.status === "rejected").length;

  this.log.info(`Webhook processing completed for ${trigger}`, {
    totalSubscribers: subscribers.length,
    successful: successCount,
    failed: failureCount,
  });

  // 详细记录每个失败
  results.forEach((result, index) => {
    if (result.status === "rejected") {
      const subscriber = subscribers[index];
      this.log.error(`Webhook processing failed for subscriber`, {
        trigger,
        webhookId: subscriber?.id,
        subscriberUrl: subscriber?.subscriberUrl,
        error: result.reason,
      });
    }
  });
}
```

---

## 4. 重试机制 - 失败终止与状态回写

### 4.1 两种重试机制

Cal.diy 的 webhook 系统存在**两层重试机制**：

| 层级 | 机制 | 触发条件 | 重试次数 |
|------|------|---------|---------|
| **L1: Trigger.dev 任务层** | Trigger.dev 内置重试 | 任务抛出异常 | 3 次 |
| **L2: 应用层 (processWebhooks)** | Promise.allSettled 隔离 | HTTP 响应失败 | 0 次 (仅日志记录) |

### 4.2 L1: Trigger.dev 任务重试配置

**配置位置**: `packages/features/webhooks/lib/tasker/trigger/config.ts`

```typescript
export const webhookDeliveryTaskConfig: WebhookDeliveryTask = {
  machine: "small-1x",
  queue: webhookDeliveryQueue,
  retry: {
    maxAttempts: 3,                    // 最大尝试次数 (包含首次)
    factor: 2,                          // 指数退避因子
    minTimeoutInMs: 30000,             // 最小超时 30秒
    maxTimeoutInMs: 600000,            // 最大超时 10分钟
    randomize: true,                    // 随机化防止抖动
    outOfMemory: {                      // 内存不足时升级机器
      machine: "medium-1x",
    },
  },
};

// 队列配置
export const webhookDeliveryQueue: Queue = queue({
  name: "webhook-delivery",
  concurrencyLimit: 25,  // 并发限制 25
});
```

### 4.3 重试触发条件与终止条件

#### 触发重试的条件

**Trigger.dev 任务重试的触发条件是**: 任务执行过程中抛出**未捕获的异常**。

**可能触发重试的场景**:

```typescript
// 场景 1: DataFetcher 抛出异常 (数据库查询失败)
const eventData = await fetcher.fetchEventData(payload);
// 如果数据库连接失败，抛出异常 → 触发重试

// 场景 2: 构建 payload 时抛出异常 (数据格式错误)
const dto = this.buildDTO(eventData, payload);
if (!dto) {
  this.log.warn("Failed to build DTO for webhook", {...});
  return;  // 注意: 返回不会触发重试
}
// 但如果 buildDTO 内部抛出异常 → 触发重试

// 场景 3: 所有订阅者都失败 (processWebhooks 内部捕获)
// 注意: processWebhooks 使用 allSettled，不会抛出异常
// 除非所有订阅者都失败且代码有特殊处理

// 场景 4: 任务队列本身的问题 (Trigger.dev 内部错误)
// 网络分区、服务不可用等 → Trigger.dev 自动重试
```

#### 终止重试的条件

| 条件 | 说明 |
|------|------|
| **maxAttempts 耗尽** | 达到 3 次尝试后不再重试 |
| **任务成功完成** | 没有抛出异常，正常返回 |
| **不可恢复的错误** | 某些错误类型标记为不重试 (需配置) |

**重试时间线 (指数退避)**:

```
尝试 1 (首次): t=0s
  ↓ 失败，抛出异常
  
等待: 30s * 2^0 * random(0.5-1.5) ≈ 15-45s
  ↓
  
尝试 2: t≈30s
  ↓ 失败，抛出异常
  
等待: 30s * 2^1 * random(0.5-1.5) ≈ 30-90s
  ↓
  
尝试 3: t≈90s
  ↓ 失败，抛出异常
  
等待: 30s * 2^2 * random(0.5-1.5) ≈ 60-180s
  ↓
  
尝试 4 (maxAttempts=3，实际不会执行):
  ↓ 终止，任务标记为 FAILED
```

### 4.4 状态回写与日志闭环

#### 日志记录体系

系统实现了**完整的日志闭环**，从入队到处理完成都有追踪：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    生产者入队日志                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ this.log.debug("Queueing booking webhook task", {                    │ │
│  │   operationId,                                                         │ │
│  │   triggerEvent: WebhookTriggerEvents.BOOKING_CREATED,                │ │
│  │   bookingUid: params.bookingUid,                                       │ │
│  │   eventTypeId: params.eventTypeId,                                     │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 入队成功                                                            │ │
│  │ this.log.debug("Webhook delivery task queued", {                      │ │
│  │   operationId, taskId: result.taskId                                  │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 入队失败                                                            │ │
│  │ this.log.error("Failed to queue webhook delivery task", {             │ │
│  │   operationId, error: error.message                                   │ │
│  │ });                                                                    │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    消费者处理日志                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ // 任务开始                                                            │ │
│  │ this.log.debug("Processing webhook delivery task", {                 │ │
│  │   operationId: payload.operationId,                                   │ │
│  │   taskId,                                                              │ │
│  │   triggerEvent: payload.triggerEvent,                                 │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 找到订阅者                                                          │ │
│  │ this.log.debug(`Found ${subscribers.length} webhook subscriber(s)`, {│ │
│  │   operationId: payload.operationId                                    │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 数据获取失败 (不会触发重试，仅记录警告)                              │ │
│  │ this.log.warn("Event data not found", {                               │ │
│  │   operationId: payload.operationId,                                   │ │
│  │   triggerEvent: payload.triggerEvent,                                 │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 任务完成                                                            │ │
│  │ this.log.debug("Webhook delivery task completed", {                   │ │
│  │   operationId: payload.operationId,                                   │ │
│  │   subscriberCount: subscribers.length,                                 │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 任务失败 (会触发重试)                                               │ │
│  │ this.log.error("Failed to process webhook delivery task", {          │ │
│  │   operationId: payload.operationId,                                   │ │
│  │   taskId,                                                              │ │
│  │   error: error.message,                                                │ │
│  │ });                                                                    │ │
│  │ throw error;  // 抛出异常，触发 Trigger.dev 重试                      │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    HTTP 投递日志                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ // 发送前                                                              │ │
│  │ this.log.debug("Processing webhook", {                                │ │
│  │   trigger, webhookId: subscriber.id, subscriberUrl                    │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 发送成功                                                            │ │
│  │ this.log.debug(`Webhook sent successfully`, {                         │ │
│  │   trigger, webhookId: subscriber.id, statusCode: result.status        │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 发送失败 (HTTP 非 2xx，不会触发重试，仅记录错误)                    │ │
│  │ this.log.error(`Webhook failed`, {                                    │ │
│  │   error: result.message,                                               │ │
│  │   trigger, webhookId: subscriber.id, statusCode: result.status        │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 异常失败 (网络错误、超时等，会抛出异常 → 触发重试)                   │ │
│  │ this.log.error("Error sending webhook", {                             │ │
│  │   error: err.message, trigger, webhookId: subscriber.id               │ │
│  │ });                                                                    │ │
│  │ throw err;  // 抛出，触发重试                                          │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    汇总统计日志                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ this.log.info(`Webhook processing completed for ${trigger}`, {        │ │
│  │   totalSubscribers: subscribers.length,                               │ │
│  │   successful: successCount,                                           │ │
│  │   failed: failureCount,                                               │ │
│  │ });                                                                    │ │
│  │                                                                       │ │
│  │ // 详细记录每个失败的订阅者                                            │ │
│  │ results.forEach((result, index) => {                                  │ │
│  │   if (result.status === "rejected") {                                 │ │
│  │     this.log.error(`Webhook processing failed for subscriber`, {       │ │
│  │       trigger, webhookId: subscriber?.id,                             │ │
│  │       subscriberUrl: subscriber?.subscriberUrl,                        │ │
│  │       error: result.reason,                                            │ │
│  │     });                                                                │ │
│  │   }                                                                    │ │
│  │ });                                                                    │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.5 失败场景与处理策略

#### 场景分类与处理

| 失败场景 | 层级 | 触发重试? | 处理策略 |
|---------|------|----------|---------|
| **网络超时** (fetch timeout) | L2 | ✅ 是 | 抛出异常 → Trigger.dev 重试 |
| **DNS 解析失败** | L2 | ✅ 是 | 抛出异常 → Trigger.dev 重试 |
| **连接拒绝** | L2 | ✅ 是 | 抛出异常 → Trigger.dev 重试 |
| **HTTP 5xx** (服务端错误) | L2 | ❌ 否 | 仅记录错误日志，不重试 |
| **HTTP 4xx** (客户端错误) | L2 | ❌ 否 | 仅记录错误日志，不重试 |
| **数据库查询失败** | L1 | ✅ 是 | 抛出异常 → Trigger.dev 重试 |
| **订阅者不存在** | L1 | ❌ 否 | return 静默返回，不重试 |
| **数据不存在** | L1 | ❌ 否 | return 静默返回，不重试 |
| **Payload 构建失败** | L1 | ✅ 是 | 抛出异常 → Trigger.dev 重试 |

#### 关键代码分析

**HTTP 响应失败不会触发重试** (`WebhookService.ts:75`):

```typescript
private async sendWebhookDirectly(...): Promise<WebhookDeliveryResult> {
  // ... 构建请求 ...
  
  const response = await fetch(subscriberUrl, { /* ... */ });
  const responseText = await response.text();

  // 关键点: 无论 HTTP 状态码是什么，都返回结果，不抛出异常
  return {
    ok: response.ok,           // 2xx 为 true，其他为 false
    status: response.status,   // HTTP 状态码
    message: responseText || undefined,
    duration: 0,
    subscriberUrl: subscriberUrl,
    webhookId: subscriber.id,
  };
}
```

**调用方仅记录日志** (`WebhookService.ts:153`):

```typescript
const result = await this.sendWebhook(trigger, payload, subscriber);

if (result.ok) {
  this.log.debug(`Webhook sent successfully`, {...});
} else {
  // 仅记录错误，不抛出异常 → 不触发重试
  this.log.error(`Webhook failed`, {
    error: result.message,
    trigger,
    webhookId: subscriber.id,
    statusCode: result.status,
  });
}
```

**但网络错误会抛出异常** (`sendPayload.ts:312`):

```typescript
const _sendPayload = async (...) => {
  // fetch 本身在网络错误时会抛出异常
  const response = await fetch(subscriberUrl, { /* ... */ });
  // 如果网络超时、DNS 失败等，fetch 会抛出异常
  
  return { ok: response.ok, status: response.status };
};
```

### 4.6 重试终止后的状态

当 Trigger.dev 任务达到 `maxAttempts` 后：

```
任务状态流转:

PENDING (待处理)
    ↓
RUNNING (处理中)
    ↓
FAILED (首次失败)
    ↓
DELAYED (等待重试)
    ↓
RUNNING (重试 1)
    ↓
FAILED (失败)
    ↓
DELAYED (等待重试)
    ↓
RUNNING (重试 2)
    ↓
FAILED (最终失败，maxAttempts 耗尽)
    ↓
TERMINAL_STATE (不再重试)
```

**此时的状态**:

1. **Trigger.dev 任务标记为 FAILED**
2. **日志中会有多次失败记录** (每次尝试都有日志)
3. **数据库中没有持久化的任务状态** (当前实现)
4. **没有自动通知机制** (需要手动查看日志或 Trigger.dev 仪表板)

### 4.7 定时触发的重试 (WebhookScheduledTriggers)

对于 `MEETING_STARTED` / `MEETING_ENDED` 等定时触发事件：

**处理流程** (`handleWebhookScheduledTriggers.ts`):

```typescript
export async function handleWebhookScheduledTriggers(prisma: PrismaClient) {
  // 1. 清理过期任务 (超过 1 天的)
  await prisma.webhookScheduledTriggers.deleteMany({
    where: { startAfter: { lte: dayjs().subtract(1, "day").toDate() } },
  });

  // 2. 获取到期任务
  const jobsToRun = await prisma.webhookScheduledTriggers.findMany({
    where: { startAfter: { lte: dayjs().toDate() } },
    select: { id, jobName, payload, subscriberUrl, webhook: { select: { secret, version } } },
  });

  // 3. 执行任务
  const fetchPromises: Promise<Response | void>[] = [];
  
  for (const job of jobsToRun) {
    // 构建请求
    const headers = {
      "Content-Type": "...",
      "X-Cal-Webhook-Version": webhook?.version ?? DEFAULT_WEBHOOK_VERSION,
    };
    if (webhook) {
      headers["X-Cal-Signature-256"] = createWebhookSignature({...});
    }

    // 发送请求 (catch 内部错误)
    fetchPromises.push(
      fetch(job.subscriberUrl, {
        method: "POST",
        body: job.payload,
        headers,
        redirect: "manual",
      }).catch((error) => {
        console.error(`Webhook trigger for subscriber url ${job.subscriberUrl} failed with error: ${error}`);
      })
    );

    // 关键点: 无论成功失败，立即删除任务
    await prisma.webhookScheduledTriggers.delete({
      where: { id: job.id },
    });
  }

  // 不等待结果
  Promise.allSettled(fetchPromises);
}
```

**定时触发的重试特点**:

| 特性 | 说明 |
|------|------|
| **重试机制** | ❌ 无内置重试 |
| **retryCount 字段** | ⚠️ 存在但未使用 |
| **失败处理** | 仅 console.error 记录 |
| **任务清理** | 执行后立即删除，无论成功失败 |
| **过期清理** | 超过 1 天的任务被删除 |

### 4.8 重试机制总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         重试机制架构                                       │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                        即时事件 (BOOKING_CREATED 等)                  │ │
│  │                                                                       │ │
│  │   事件源 ──> Producer ──> Trigger.dev 队列 ──> Consumer             │ │
│  │                                 │                                     │ │
│  │                                 ▼                                     │ │
│  │                         重试配置:                                      │ │
│  │                         - maxAttempts: 3                              │ │
│  │                         - 触发条件: 抛出未捕获异常                    │ │
│  │                         - 指数退避: 30s, 60s, 120s (带随机)        │ │
│  │                                                                       │ │
│  │                         不会重试的情况:                               │ │
│  │                         - HTTP 4xx/5xx 响应 (仅记录)                 │ │
│  │                         - 数据不存在 (return 静默返回)                │ │
│  │                         - 订阅者为空 (return 静默返回)                │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                        定时事件 (MEETING_STARTED 等)                  │ │
│  │                                                                       │ │
│  │   创建订阅时 ──> WebhookScheduledTriggers ──> Cron 执行             │ │
│  │                                 │                                     │ │
│  │                                 ▼                                     │ │
│  │                         重试配置:                                      │ │
│  │                         - ❌ 无内置重试机制                            │ │
│  │                         - retryCount 字段存在但未使用                  │ │
│  │                         - 失败后立即删除任务记录                       │ │
│  │                         - 仅 console.error 记录错误                    │ │
│  │                                                                       │ │
│  │                         清理策略:                                      │ │
│  │                         - 执行后立即删除                               │ │
│  │                         - 超过 1 天的任务被删除                       │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 外部自动化工具接入 - Zapier vs Make 对比

### 5.1 架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    外部自动化工具接入架构                                   │
│                                                                           │
│  ┌───────────────────────────────┐    ┌───────────────────────────────┐ │
│  │           Zapier              │    │            Make                 │ │
│  │  ┌─────────────────────────┐  │    │  ┌─────────────────────────┐  │ │
│  │  │ 鉴权方式:               │  │    │  │ 鉴权方式:               │  │ │
│  │  │ - OAuth 2.0            │  │    │  │ - 仅 API Key             │  │ │
│  │  │ - API Key (可选)       │  │    │  │ - 无 OAuth 支持         │  │ │
│  │  │                        │  │    │  │                         │  │ │
│  │  └─────────────────────────┘  │    │  └─────────────────────────┘  │ │
│  │                               │    │                               │  │
│  │  ┌─────────────────────────┐  │    │  ┌─────────────────────────┐  │ │
│  │  │ 订阅管理:               │  │    │  │ 订阅管理:               │  │ │
│  │  │ - validateAccountOrApi- │  │    │  │ - findValidApiKey()    │  │ │
│  │  │   Key() 双模式处理      │  │    │  │ - 直接调用               │  │ │
│  │  │                        │  │    │  │                         │  │ │
│  │  └─────────────────────────┘  │    │  └─────────────────────────┘  │ │
│  │                               │    │                               │  │
│  │  ┌─────────────────────────┐  │    │  ┌─────────────────────────┐  │ │
│  │  │ 投递链路:               │  │    │  │ 投递链路:               │  │ │
│  │  │ - getZapierPayload()   │  │    │  │ - 标准 payload 格式    │  │ │
│  │  │   自定义精简格式        │  │    │  │ - 无特殊处理           │  │ │
│  │  │                        │  │    │  │                         │  │ │
│  │  └─────────────────────────┘  │    │  └─────────────────────────┘  │ │
│  └───────────────────────────────┘    └───────────────────────────────┘ │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### 5.2 鉴权方式差异

#### 5.2.1 Zapier: OAuth 2.0 + API Key 双模式

Zapier 支持两种鉴权方式，通过 `validateAccountOrApiKey` 函数统一处理：

**核心代码** (`packages/app-store/zapier/lib/validateAccountOrApiKey.ts`):

```typescript
export async function validateAccountOrApiKey(req: NextApiRequest, requiredScopes: string[] = []) {
  const apiKey = req.query.apiKey as string;

  if (!apiKey) {
    // OAuth 2.0 模式: 从 Authorization header 获取 Bearer token
    const token = req.headers.authorization?.split(" ")[1] || "";
    const authorizedAccount = await isAuthorized(token, requiredScopes);
    if (!authorizedAccount) throw new HttpError({ statusCode: 401, message: "Unauthorized" });
    return { account: authorizedAccount, appApiKey: undefined };
  }

  // API Key 模式: 从 query 参数获取
  const validKey = await findValidApiKey(apiKey, "zapier");
  if (!validKey) throw new HttpError({ statusCode: 401, message: "API key not valid" });
  return { account: null, appApiKey: validKey };
}
```

**鉴权流程**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Zapier 鉴权流程                                        │
│                                                                           │
│  1. 检查 req.query.apiKey 是否存在                                         │
│       │                                                                   │
│       ├── 不存在 → OAuth 2.0 模式                                        │
│       │        │                                                          │
│       │        ▼                                                          │
│       │   从 Authorization header 获取 Bearer token                       │
│       │        │                                                          │
│       │        ▼                                                          │
│       │   isAuthorized(token, requiredScopes)                            │
│       │        │                                                          │
│       │        ├── 成功 → return { account, appApiKey: undefined }       │
│       │        │                                                          │
│       │        └── 失败 → throw HttpError(401)                          │
│       │                                                                   │
│       └── 存在 → API Key 模式                                             │
│                │                                                          │
│                ▼                                                          │
│           findValidApiKey(apiKey, "zapier")                               │
│                │                                                          │
│                ├── 成功 → return { account: null, appApiKey }            │
│                │                                                          │
│                └── 失败 → throw HttpError(401)                          │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

#### 5.2.2 Make: 仅 API Key 模式

Make 只支持 API Key 鉴权，不支持 OAuth 2.0：

**核心代码** (`packages/app-store/make/api/subscriptions/addSubscription.ts`):

```typescript
async function handler(req: NextApiRequest, res: NextApiResponse) {
  const apiKey = req.query.apiKey as string;

  // 必须提供 apiKey，无 OAuth 回退
  if (!apiKey) {
    return res.status(401).json({ message: "No API key provided" });
  }

  // 直接验证 API Key
  const validKey = await findValidApiKey(apiKey, "make");

  if (!validKey) {
    return res.status(401).json({ message: "API key not valid" });
  }

  // 继续处理订阅...
}
```

**鉴权流程**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Make 鉴权流程                                          │
│                                                                           │
│  1. 检查 req.query.apiKey 是否存在                                         │
│       │                                                                   │
│       ├── 不存在 → 直接返回 401 (无 OAuth 回退)                          │
│       │        │                                                          │
│       │        ▼                                                          │
│       │   return res.status(401).json({                                  │
│       │     message: "No API key provided"                                │
│       │   })                                                              │
│       │                                                                   │
│       └── 存在 → 验证 API Key                                             │
│                │                                                          │
│                ▼                                                          │
│           findValidApiKey(apiKey, "make")                                 │
│                │                                                          │
│                ├── 成功 → 继续处理订阅                                    │
│                │                                                          │
│                └── 失败 → return res.status(401).json({                  │
│                            message: "API key not valid"                   │
│                          })                                               │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

#### 5.2.3 OAuth 2.0 Token 验证机制 (Zapier 独有)

Zapier 的 OAuth 2.0 模式使用 `isAuthorized` 函数验证 JWT token：

**核心代码** (`packages/features/auth/lib/oAuthAuthorization.ts`):

```typescript
export default async function isAuthorized(token: string, requiredScopes: string[] = []) {
  let decodedToken: OAuthTokenPayload;
  try {
    decodedToken = jwt.verify(token, process.env.CALENDSO_ENCRYPTION_KEY || "") as OAuthTokenPayload;
  } catch {
    return null;
  }

  if (!decodedToken) return null;
  
  // 检查所需 scope
  const hasAllRequiredScopes = requiredScopes.every((scope) => decodedToken.scope.includes(scope));

  // 验证 token 类型
  if (!hasAllRequiredScopes || decodedToken.token_type !== "Access Token") {
    return null;
  }

  // 用户级 token
  if (decodedToken.userId) {
    const user = await prisma.user.findUnique({
      where: { id: decodedToken.userId },
      select: { id: true, username: true },
    });
    if (!user) return null;
    return { id: user.id, name: user.username, isTeam: false };
  }

  // 团队级 token
  if (decodedToken.teamId) {
    const team = await prisma.team.findUnique({
      where: { id: decodedToken.teamId },
      select: { id: true, name: true },
    });
    if (!team) return null;
    return { ...team, isTeam: true };
  }

  return null;
}
```

**OAuth Token 验证流程**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OAuth 2.0 Token 验证流程 (Zapier 独有)                │
│                                                                           │
│  1. 从 Authorization: Bearer <token> 获取 token                          │
│            │                                                              │
│            ▼                                                              │
│  2. JWT 验证 (jwt.verify)                                                 │
│            │                                                              │
│            ├── 失败 (签名错误/过期) → return null                        │
│            │                                                              │
│            ▼ (成功)                                                       │
│  3. 检查 token_type === "Access Token"                                    │
│            │                                                              │
│            ├── 不匹配 → return null                                       │
│            │                                                              │
│            ▼ (匹配)                                                       │
│  4. 检查 requiredScopes (如 ["READ_BOOKING", "READ_PROFILE"])            │
│            │                                                              │
│            ├── 缺少任一 scope → return null                               │
│            │                                                              │
│            ▼ (全部满足)                                                   │
│  5. 判断 token 类型                                                        │
│            │                                                              │
│            ├── userId 存在 → 用户级 token                                │
│            │        │                                                     │
│            │        ▼                                                     │
│            │   查询 prisma.user.findUnique({ id: userId })              │
│            │        │                                                     │
│            │        ├── 不存在 → return null                              │
│            │        │                                                     │
│            │        └── 存在 → return { id, name, isTeam: false }       │
│            │                                                              │
│            └── teamId 存在 → 团队级 token                                │
│                     │                                                      │
│                     ▼                                                      │
│                查询 prisma.team.findUnique({ id: teamId })               │
│                     │                                                      │
│                     ├── 不存在 → return null                               │
│                     │                                                      │
│                     └── 存在 → return { id, name, isTeam: true }         │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

**OAuth Token 与 API Key 鉴权结果差异**:

| 鉴权方式 | 返回值结构 | 使用场景 |
|---------|-----------|---------|
| **OAuth 2.0** | `{ account: { id, name, isTeam }, appApiKey: undefined }` | 支持用户/团队区分，可传递给 `addSubscription` 的 `account` 参数 |
| **API Key** | `{ account: null, appApiKey: ApiKey }` | 直接使用 `appApiKey.userId` 或 `appApiKey.teamId` |

**Token Payload 结构** (`OAuthTokenPayload`):
```typescript
{
  userId?: number;      // 用户级 token
  teamId?: number;      // 团队级 token
  scope: string[];      // 权限范围 (如 ["READ_BOOKING", "READ_PROFILE"])
  token_type: string;   // 必须为 "Access Token"
  // ... 其他 JWT 标准字段 (exp, iat 等)
}
```

#### 5.2.4 API Key 验证机制 (共享)

两种工具使用相同的 `findValidApiKey` 函数验证 API Key：

**核心代码** (`packages/app-store/_utils/findValidApiKey.ts`):

```typescript
function hashAPIKey(apiKey: string): string {
  return createHash("sha256").update(apiKey).digest("hex");
}

export async function findValidApiKey(apiKey: string, appId: string): Promise<ApiKey | null> {
  const hashedKey = hashAPIKey(apiKey);

  return prisma.apiKey.findFirst({
    where: {
      hashedKey,
      appId,  // 验证 appId 匹配 (zapier 或 make)
      OR: [{ expiresAt: null }, { expiresAt: { gte: new Date() } }],  // 未过期
    },
  });
}
```

**验证条件**:
1. **SHA256 哈希匹配**: 存储的是哈希值，不是明文
2. **appId 匹配**: 验证 API Key 是否属于该应用
3. **未过期**: `expiresAt` 为 null 或大于当前时间

#### 5.2.5 鉴权方式对比总结

| 特性 | Zapier | Make |
|------|--------|------|
| **OAuth 2.0 支持** | ✅ 支持 (Authorization header) | ❌ 不支持 |
| **API Key 支持** | ✅ 支持 (query 参数) | ✅ 支持 (query 参数) |
| **鉴权函数** | `validateAccountOrApiKey()` | 直接调用 `findValidApiKey()` |
| **OAuth 回退** | ✅ 无 apiKey 时使用 OAuth | ❌ 无 apiKey 时直接 401 |
| **appId 验证** | `appId: "zapier"` | `appId: "make"` |

---

### 5.3 订阅管理接口差异

#### 5.3.1 接口概览

两种工具的订阅管理接口位于：

| 工具 | 接口路径 |
|------|---------|
| **Zapier** | `packages/app-store/zapier/api/subscriptions/` |
| **Make** | `packages/app-store/make/api/subscriptions/` |

**可用接口**:

| 接口 | Zapier | Make |
|------|--------|------|
| `addSubscription` (POST) | ✅ | ✅ |
| `deleteSubscription` (DELETE) | ✅ | ✅ |
| `listBookings` (GET) | ✅ | ✅ |
| `me` (GET) | ✅ | ✅ |
| `listOOOEntries` (GET) | ✅ | ❌ |

**注意**: Zapier 有额外的 `listOOOEntries` 接口，用于获取外出状态列表。

#### 5.3.2 底层实现复用

两种工具**共享相同的底层实现** (`packages/features/webhooks/lib/scheduleTrigger.ts`):

- **`addSubscription`**: 创建新订阅
- **`deleteSubscription`**: 删除订阅
- **`scheduleTrigger`**: 为定时事件创建触发记录
- **`deleteWebhookScheduledTriggers`**: 删除定时触发记录

#### 5.3.3 添加订阅接口对比

**Zapier - addSubscription**:

```typescript
// packages/app-store/zapier/api/subscriptions/addSubscription.ts
async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { subscriberUrl, triggerEvent } = req.body;
  
  // 使用双模式鉴权
  const { account, appApiKey } = await validateAccountOrApiKey(req, ["READ_BOOKING", "READ_PROFILE"]);
  
  // 调用底层实现
  const createAppSubscription = await addSubscription({
    appApiKey,
    account,           // OAuth 模式时提供
    triggerEvent,
    subscriberUrl,
    appId: "zapier",
  });

  // ...
}
```

**Make - addSubscription**:

```typescript
// packages/app-store/make/api/subscriptions/addSubscription.ts
async function handler(req: NextApiRequest, res: NextApiResponse) {
  const apiKey = req.query.apiKey as string;
  
  // 仅 API Key 模式，无 OAuth
  if (!apiKey) {
    return res.status(401).json({ message: "No API key provided" });
  }

  const validKey = await findValidApiKey(apiKey, "make");
  
  if (!validKey) {
    return res.status(401).json({ message: "API key not valid" });
  }

  const { subscriberUrl, triggerEvent } = req.body;

  // 调用底层实现 (无 account 参数)
  const createAppSubscription = await addSubscription({
    appApiKey: validKey,
    // account: undefined (不支持 OAuth)
    triggerEvent,
    subscriberUrl,
    appId: "make",
  });

  // ...
}
```

#### 5.3.4 删除订阅接口对比

**Zapier - deleteSubscription**:

```typescript
// packages/app-store/zapier/api/subscriptions/deleteSubscription.ts
async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { id } = querySchema.parse(req.query);

  // 双模式鉴权
  const { account, appApiKey } = await validateAccountOrApiKey(req, ["READ_BOOKING", "READ_PROFILE"]);

  const deleteEventSubscription = await deleteSubscription({
    appApiKey,
    account,
    webhookId: id,
    appId: "zapier",
  });

  // ...
}
```

**Make - deleteSubscription**:

```typescript
// packages/app-store/make/api/subscriptions/deleteSubscription.ts
async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { apiKey, id } = querySchema.parse(req.query);

  // 仅 API Key 模式
  if (!apiKey) {
    return res.status(401).json({ message: "No API key provided" });
  }

  const validKey = await findValidApiKey(apiKey, "make");

  if (!validKey) {
    return res.status(401).json({ message: "API key not valid" });
  }

  const deleteEventSubscription = await deleteSubscription({
    appApiKey: validKey,
    // account: undefined
    webhookId: id,
    appId: "make",
  });

  // ...
}
```

#### 5.3.5 底层 addSubscription 实现

两种工具共享相同的底层实现：

**核心代码** (`packages/features/webhooks/lib/scheduleTrigger.ts:30`):

```typescript
export async function addSubscription({
  appApiKey,
  triggerEvent,
  subscriberUrl,
  appId,
  account,  // OAuth 模式时提供 (Zapier 特有)
}: {
  appApiKey?: ApiKey;
  triggerEvent: WebhookTriggerEvents;
  subscriberUrl: string;
  appId: string;
  account?: {
    id: number;
    name: string | null;
    isTeam: boolean;
  } | null;
}) {
  // 确定 userId 和 teamId
  const userId = appApiKey ? appApiKey.userId : account && !account.isTeam ? account.id : null;
  const teamId = appApiKey ? appApiKey.teamId : account && account.isTeam ? account.id : null;

  // 创建 Webhook 记录
  const createSubscription = await prisma.webhook.create({
    data: {
      id: v4(),
      userId,
      teamId,
      eventTriggers: [triggerEvent],
      subscriberUrl,
      active: true,
      appId: appId,  // "zapier" 或 "make"
    },
  });

  // 对于定时事件 (MEETING_STARTED/ENDED)，为现有预订创建触发记录
  if (
    triggerEvent === WebhookTriggerEvents.MEETING_ENDED ||
    triggerEvent === WebhookTriggerEvents.MEETING_STARTED
  ) {
    // 查询现有预订
    const bookings = await prisma.booking.findMany({
      where: {
        ...where,
        startTime: { gte: new Date() },
        status: BookingStatus.ACCEPTED,
      },
      // ...
    });

    // 为每个预订创建触发记录
    for (const booking of bookingsWithCalEventResponses) {
      scheduleTrigger({
        booking,
        subscriberUrl: createSubscription.subscriberUrl,
        subscriber: {
          id: createSubscription.id,
          appId: createSubscription.appId,
        },
        triggerEvent,
      });
    }
  }

  return createSubscription;
}
```

#### 5.3.6 订阅管理接口对比总结

| 特性 | Zapier | Make |
|------|--------|------|
| **底层实现** | 共享 `addSubscription` / `deleteSubscription` | 共享相同实现 |
| **鉴权方式** | `validateAccountOrApiKey()` (双模式) | `findValidApiKey()` (仅 API Key) |
| **account 参数** | ✅ 支持 (OAuth 模式) | ❌ 不支持 |
| **listOOOEntries 接口** | ✅ 支持 | ❌ 不支持 |
| **appId 标识** | `"zapier"` | `"make"` |

---

### 5.4 投递接入链路差异

#### 5.4.1 相同的投递链路

两种工具使用**完全相同的投递链路**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    共享的投递链路                                         │
│                                                                           │
│  事件触发                                                                  │
│      │                                                                    │
│      ▼                                                                    │
│  WebhookTaskerProducerService.queueXxxWebhook()                          │
│      │                                                                    │
│      ▼                                                                    │
│  Trigger.dev 任务队列                                                     │
│      │                                                                    │
│      ▼                                                                    │
│  WebhookTaskConsumer.processWebhookTask()                                 │
│      │                                                                    │
│      ▼                                                                    │
│  DataFetcher.fetchEventData()                                             │
│      │                                                                    │
│      ▼                                                                    │
│  PayloadBuilder.build()                                                   │
│      │                                                                    │
│      ▼                                                                    │
│  sendPayload()  ← 此处开始有差异                                          │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

#### 5.4.2 差异点: Payload 格式

**Zapier 有自定义的精简格式**，Make 使用标准格式。

**核心代码** (`packages/features/webhooks/lib/sendPayload.ts:232`):

```typescript
/* Zapier id is hardcoded in the DB, we send the raw data for this case  */
if (isEventPayload(data)) {
  data.description = data.description || data.additionalNotes;
  if (appId === "zapier") {
    // Zapier: 使用自定义精简格式
    body = getZapierPayload({ ...data, createdAt });
  }
}

// 其他情况 (包括 Make): 使用标准格式
if (body === undefined) {
  if (
    template &&
    (isOOOEntryPayload(data) || isEventPayload(data) || isNoShowPayload(data))
  ) {
    // 自定义模板
    body = applyTemplate(template, { ...data, triggerEvent, createdAt }, contentType);
  } else {
    // 标准格式
    body = JSON.stringify({
      triggerEvent: triggerEvent,
      createdAt: createdAt,
      payload: data,
    });
  }
}
```

#### 5.4.3 Zapier 自定义 Payload 格式

**核心代码** (`packages/features/webhooks/lib/sendPayload.ts:132`):

```typescript
function getZapierPayload(data: WithUTCOffsetType<EventPayloadType & { createdAt: string }>): string {
  // 精简 attendees
  const attendees = (data.attendees as (Person & UTCOffset)[]).map((attendee) => {
    return {
      name: attendee.name,
      email: attendee.email,
      timeZone: attendee.timeZone,
      utcOffset: attendee.utcOffset,
    };
  });

  // 获取可读的位置信息
  const t = data.organizer.language.translate;
  const location = getHumanReadableLocationValue(data.location || "", t);

  // 构建精简格式
  const body = {
    uid: data.uid,
    title: data.title,
    description: data.description,
    customInputs: data.customInputs,
    responses: data.responses,
    userFieldsResponses: data.userFieldsResponses,
    startTime: data.startTime,
    endTime: data.endTime,
    location: location,
    status: data.status,
    cancellationReason: data.cancellationReason,
    user: {
      username: data.organizer.username,
      usernameInOrg: data.organizer.usernameInOrg,
      name: data.organizer.name,
      email: data.organizer.email,
      timeZone: data.organizer.timeZone,
      utcOffset: data.organizer.utcOffset,
      locale: data.organizer.locale,
    },
    eventType: {
      title: data.eventTitle,
      description: data.eventDescription,
      requiresConfirmation: data.requiresConfirmation,
      price: data.price,
      currency: data.currency,
      length: data.length,
    },
    attendees: attendees,
    createdAt: data.createdAt,
    metadata: {
      videoCallUrl: data.metadata?.videoCallUrl,
    },
  };
  
  return JSON.stringify(body);
}
```

#### 5.4.4 标准 Payload 格式 (Make 使用)

```typescript
// 标准格式 (sendPayload.ts:247)
body = JSON.stringify({
  triggerEvent: triggerEvent,    // 如 "BOOKING_CREATED"
  createdAt: createdAt,           // ISO 时间戳
  payload: data,                  // 完整的 EventPayloadType
});
```

#### 5.4.5 Payload 格式对比

**Zapier 格式示例**:

```json
{
  "uid": "booking-uid-123",
  "title": "Meeting with John",
  "description": "Discuss project",
  "startTime": "2024-01-15T10:00:00.000Z",
  "endTime": "2024-01-15T10:30:00.000Z",
  "location": "Zoom",
  "status": "ACCEPTED",
  "user": {
    "username": "jane",
    "name": "Jane Doe",
    "email": "jane@example.com",
    "timeZone": "America/New_York",
    "utcOffset": -300
  },
  "attendees": [
    {
      "name": "John Smith",
      "email": "john@example.com",
      "timeZone": "Europe/London",
      "utcOffset": 0
    }
  ],
  "createdAt": "2024-01-14T15:30:00.000Z"
}
```

**Make 格式示例 (标准格式)**:

```json
{
  "triggerEvent": "BOOKING_CREATED",
  "createdAt": "2024-01-14T15:30:00.000Z",
  "payload": {
    "uid": "booking-uid-123",
    "title": "Meeting with John",
    "description": "Discuss project",
    "startTime": "2024-01-15T10:00:00.000Z",
    "endTime": "2024-01-15T10:30:00.000Z",
    "location": "Zoom",
    "status": "ACCEPTED",
    "organizer": {
      "username": "jane",
      "name": "Jane Doe",
      "email": "jane@example.com",
      "timeZone": "America/New_York"
    },
    "attendees": [
      {
        "name": "John Smith",
        "email": "john@example.com",
        "timeZone": "Europe/London",
        "locale": "en",
        "language": { /* ... */ }
      }
    ],
    "eventType": {
      "title": "30 Minute Meeting",
      "description": "A 30 minute meeting",
      "requiresConfirmation": false,
      "price": 0,
      "currency": "USD",
      "length": 30
    },
    "metadata": {
      "videoCallUrl": "https://zoom.us/j/123456"
    },
    "responses": { /* ... */ },
    "customInputs": { /* ... */ },
    "cancellationReason": null,
    "bookingId": 123
  }
}
```

#### 5.4.6 投递链路对比总结

| 特性 | Zapier | Make |
|------|--------|------|
| **投递链路** | 完全相同 | 完全相同 |
| **Producer** | `WebhookTaskerProducerService` | 相同 |
| **Consumer** | `WebhookTaskConsumer` | 相同 |
| **签名验证** | HMAC-SHA256 (`X-Cal-Signature-256`) | 相同 |
| **版本头** | `X-Cal-Webhook-Version` | 相同 |
| **Payload 格式** | 自定义精简格式 (`getZapierPayload`) | 标准格式 |
| **格式特点** | 扁平化、关键字段、`user` 而非 `organizer` | 完整嵌套、`triggerEvent` 外层、`organizer` 字段 |

---

### 5.5 完整对比总结

#### 5.5.1 对比总表

| 维度 | 特性 | Zapier | Make |
|------|------|--------|------|
| **鉴权方式** | OAuth 2.0 | ✅ 支持 | ❌ 不支持 |
| | API Key | ✅ 支持 (query 参数) | ✅ 支持 (query 参数) |
| | 鉴权函数 | `validateAccountOrApiKey()` | `findValidApiKey()` |
| | OAuth 回退 | ✅ 无 apiKey 时使用 OAuth | ❌ 直接 401 |
| | | | |
| **订阅管理** | 底层实现 | 共享 `addSubscription` / `deleteSubscription` | 相同 |
| | account 参数 | ✅ 支持 (OAuth 模式) | ❌ 不支持 |
| | listOOOEntries | ✅ 支持 | ❌ 不支持 |
| | appId | `"zapier"` | `"make"` |
| | | | |
| **投递链路** | Producer/Consumer | 完全相同 | 完全相同 |
| | 签名验证 | HMAC-SHA256 | 相同 |
| | 版本头 | `X-Cal-Webhook-Version` | 相同 |
| | Payload 格式 | 自定义精简格式 | 标准格式 |
| | 格式特点 | 扁平化、`user` 字段 | 嵌套、`triggerEvent` 外层、`organizer` 字段 |

#### 5.5.2 架构差异图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Zapier vs Make 架构差异                                │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                         Zapier 架构                                    │ │
│  │                                                                       │ │
│  │   ┌─────────────┐     ┌─────────────┐     ┌─────────────────────┐  │ │
│  │   │ OAuth 2.0   │     │  API Key    │     │                     │  │ │
│  │   │ (Bearer     │     │ (query)     │     │ validateAccount-    │  │ │
│  │   │ token)      │     │             │     │ OrApiKey()          │  │ │
│  │   └──────┬──────┘     └──────┬──────┘     │ (双模式统一处理)     │  │ │
│  │          │                   │            └──────────┬──────────┘  │ │
│  │          └─────────┬─────────┘                       │              │ │
│  │                    ▼                                 ▼              │ │
│  │         ┌─────────────────────────────────────────────────────┐    │ │
│  │         │              addSubscription / deleteSubscription      │    │ │
│  │         │              (scheduleTrigger.ts 共享实现)             │    │ │
│  │         └──────────────────────────┬──────────────────────────┘    │ │
│  │                                    │                                 │ │
│  │                                    ▼                                 │ │
│  │         ┌─────────────────────────────────────────────────────┐    │ │
│  │         │              投递链路 (共享)                          │    │ │
│  │         │  Producer → Trigger.dev → Consumer → sendPayload    │    │ │
│  │         └──────────────────────────┬──────────────────────────┘    │ │
│  │                                    │                                 │ │
│  │                                    ▼                                 │ │
│  │         ┌─────────────────────────────────────────────────────┐    │ │
│  │         │              getZapierPayload()                      │    │ │
│  │         │              (自定义精简格式)                         │    │ │
│  │         └─────────────────────────────────────────────────────┘    │ │
│  │                                                                       │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                          Make 架构                                    │ │
│  │                                                                       │ │
│  │   ┌─────────────┐                                                     │ │
│  │   │  API Key    │                                                     │ │
│  │   │ (query)     │                                                     │ │
│  │   └──────┬──────┘                                                     │ │
│  │          │                                                            │ │
│  │          ▼                                                            │ │
│  │   ┌─────────────────┐     ┌─────────────────────┐                   │ │
│  │   │ findValidApiKey │     │                     │                   │ │
│  │   │ (仅 API Key)    │     │ ❌ 无 OAuth 支持   │                   │ │
│  │   └────────┬────────┘     └─────────────────────┘                   │ │
│  │          │                                                             │ │
│  │          ▼                                                             │ │
│  │         ┌─────────────────────────────────────────────────────┐    │ │
│  │         │              addSubscription / deleteSubscription      │    │ │
│  │         │              (scheduleTrigger.ts 共享实现)             │    │ │
│  │         └──────────────────────────┬──────────────────────────┘    │ │
│  │                                    │                                 │ │
│  │                                    ▼                                 │ │
│  │         ┌─────────────────────────────────────────────────────┐    │ │
│  │         │              投递链路 (共享)                          │    │ │
│  │         │  Producer → Trigger.dev → Consumer → sendPayload    │    │ │
│  │         └──────────────────────────┬──────────────────────────┘    │ │
│  │                                    │                                 │ │
│  │                                    ▼                                 │ │
│  │         ┌─────────────────────────────────────────────────────┐    │ │
│  │         │              标准 Payload 格式                       │    │ │
│  │         │  { triggerEvent, createdAt, payload }                │    │ │
│  │         └─────────────────────────────────────────────────────┘    │ │
│  │                                                                       │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

#### 5.5.3 关键代码位置汇总

| 功能 | Zapier | Make | 共享 |
|------|--------|------|------|
| 鉴权函数 | `zapier/lib/validateAccountOrApiKey.ts` | - | `_utils/findValidApiKey.ts` |
| 添加订阅 | `zapier/api/subscriptions/addSubscription.ts` | `make/api/subscriptions/addSubscription.ts` | `webhooks/lib/scheduleTrigger.ts` |
| 删除订阅 | `zapier/api/subscriptions/deleteSubscription.ts` | `make/api/subscriptions/deleteSubscription.ts` | `webhooks/lib/scheduleTrigger.ts` |
| Payload 格式 | - | - | `webhooks/lib/sendPayload.ts` (getZapierPayload) |

---

## 6. 关键文件位置

### 6.1 核心 Webhook 系统

| 功能 | 文件路径 |
|------|---------|
| Webhook 模型 | `packages/prisma/schema.prisma:1142` |
| WebhookScheduledTriggers 模型 | `packages/prisma/schema.prisma:1313` |
| 生产者服务 | `packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts` |
| 消费者服务 | `packages/features/webhooks/lib/service/WebhookTaskConsumer.ts` |
| Webhook 服务 | `packages/features/webhooks/lib/service/WebhookService.ts` |
| Payload 发送 | `packages/features/webhooks/lib/sendPayload.ts` |
| 定时触发处理 | `packages/features/webhooks/lib/handleWebhookScheduledTriggers.ts` |
| 订阅管理 | `packages/features/webhooks/lib/scheduleTrigger.ts` |
| Trigger.dev 配置 | `packages/features/webhooks/lib/tasker/trigger/config.ts` |

### 6.2 自动化工具集成

| 功能 | Zapier | Make |
|------|--------|------|
| 鉴权函数 | `packages/app-store/zapier/lib/validateAccountOrApiKey.ts` | 无 (直接使用 findValidApiKey) |
| 添加订阅 | `packages/app-store/zapier/api/subscriptions/addSubscription.ts` | `packages/app-store/make/api/subscriptions/addSubscription.ts` |
| 删除订阅 | `packages/app-store/zapier/api/subscriptions/deleteSubscription.ts` | `packages/app-store/make/api/subscriptions/deleteSubscription.ts` |
| API Key 验证 | `packages/app-store/_utils/findValidApiKey.ts` | 相同 |

---

## 7. 数据流图

### 7.1 即时事件数据流 (BOOKING_CREATED 等)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    即时事件数据流                                          │
│                                                                           │
│  ┌─────────────┐                                                         │
│  │   事件源    │  (预订创建/取消/重新安排等)                              │
│  │  Booking/  │                                                         │
│  │   Form/    │                                                         │
│  │  Recording  │                                                         │
│  └──────┬──────┘                                                         │
│         │                                                                │
│         ▼                                                                │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              WebhookTaskerProducerService                          │  │
│  │  - 轻量级入队                                                        │  │
│  │  - 构建最小化 payload (仅 bookingUid 等关键字段)                    │  │
│  │  - 调用 webhookTasker.deliverWebhook()                             │  │
│  └──────────────────────────────┬────────────────────────────────────┘  │
│                                 │                                        │
│                                 ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    Trigger.dev 任务队列                             │  │
│  │  - 队列名: webhook-delivery                                         │  │
│  │  - 并发限制: 25                                                      │  │
│  │  - 重试配置: maxAttempts=3, 指数退避                               │  │
│  └──────────────────────────────┬────────────────────────────────────┘  │
│                                 │                                        │
│                                 ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                   WebhookTaskConsumer                               │  │
│  │  1. 获取 DataFetcher (根据 triggerEvent)                            │  │
│  │  2. 获取订阅者上下文                                                 │  │
│  │  3. 查询订阅者 (webhookRepository.getSubscribers)                  │  │
│  │  4. 获取事件数据 (fetcher.fetchEventData)                           │  │
│  │  5. 构建 DTO 和 Payload                                              │  │
│  │  6. 调用 webhookService.processWebhooks()                           │  │
│  └──────────────────────────────┬────────────────────────────────────┘  │
│                                 │                                        │
│                                 ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    WebhookService.processWebhooks()                 │  │
│  │  - 对每个订阅者并行发送 (Promise.allSettled)                        │  │
│  │  - 单个订阅者失败不影响其他订阅者                                    │  │
│  │  - 统计成功/失败数量                                                 │  │
│  │  - 记录详细日志                                                      │  │
│  └──────────────────────────────┬────────────────────────────────────┘  │
│                                 │                                        │
│                                 ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                       sendPayload()                                  │  │
│  │  - 构建 body (Zapier: getZapierPayload, 其他: 标准格式)           │  │
│  │  - 计算 HMAC-SHA256 签名                                            │  │
│  │  - 发送 POST 请求                                                   │  │
│  │  - 返回 { ok, status }                                              │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

### 7.2 定时事件数据流 (MEETING_STARTED 等)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    定时事件数据流                                          │
│                                                                           │
│  ┌─────────────┐                                                         │
│  │ 创建订阅时 │                                                         │
│  │  (add-    │                                                         │
│  │  Subscription) │                                                     │
│  └──────┬──────┘                                                         │
│         │                                                                │
│         ▼                                                                │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              scheduleTrigger() (scheduleTrigger.ts)                 │  │
│  │  - 为每个现有预订创建 WebhookScheduledTriggers 记录                 │  │
│  │  - 计算 startAfter = booking.startTime 或 endTime                   │  │
│  │  - 序列化 payload 到数据库                                           │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                 │                                        │
│                                 ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              WebhookScheduledTriggers 表                           │  │
│  │  - subscriberUrl: 目标 URL                                          │  │
│  │  - payload: 序列化的任务数据                                        │  │
│  │  - startAfter: 触发时间                                             │  │
│  │  - webhookId: 关联的 webhook                                        │  │
│  │  - bookingId: 关联的预订                                            │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                 │                                        │
│                                 ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              Cron 定时任务 (每小时/每分钟)                          │  │
│  │  - 调用 handleWebhookScheduledTriggers()                            │  │
│  └──────────────────────────────┬────────────────────────────────────┘  │
│                                 │                                        │
│                                 ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │         handleWebhookScheduledTriggers()                           │  │
│  │  1. 清理过期任务 (超过 1 天)                                        │  │
│  │  2. 查询到期任务 (startAfter <= now)                                │  │
│  │  3. 构建请求 (签名、版本头)                                          │  │
│  │  4. 发送 fetch 请求 (Promise.allSettled)                           │  │
│  │  5. **立即删除任务记录** (无论成功失败)                             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│  ⚠️ 注意: 定时触发无重试机制，失败后立即删除任务记录                      │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 版本控制

### 8.1 Webhook 版本

**版本字段**: `Webhook.version` (默认: `"2021-10-20"`)

**HTTP 头**: `X-Cal-Webhook-Version`

### 8.2 Payload 版本化

通过 `PayloadBuilderFactory` 管理不同版本的 payload 格式：

```typescript
// 获取版本化的 builder
builder = payloadBuilderFactory.getBuilder(version, triggerEvent)

// 构建 payload
webhookPayload = builder.build(dto)
```

---

## 9. 安全考虑

### 9.1 SSRF 保护

**位置**: `packages/trpc/server/routers/viewer/webhook/create.handler.ts`

```typescript
// 创建订阅时验证
const validation = validateUrlForSSRFSync(input.subscriberUrl);
if (!validation.isValid) {
  throw new TRPCError({
    code: "BAD_REQUEST",
    message: `Webhook URL is not allowed: ${validation.error}`,
  });
}
```

**禁止的地址**:
- 内网 IP: `127.0.0.1`, `192.168.x.x`, `10.x.x.x`, `172.16-31.x.x`
- `localhost`
- 私有/保留地址范围

### 9.2 HMAC-SHA256 签名验证

**位置**: `packages/features/webhooks/lib/sendPayload.ts`

```typescript
// 构建签名
export const createWebhookSignature = (params: { secret?: string | null; body: string }) =>
  params.secret
    ? createHmac("sha256", params.secret).update(`${params.body}`).digest("hex")
    : "no-secret-provided";

// HTTP 头
headers: {
  "X-Cal-Signature-256": signature,
}
```

**验证方应该**:
1. 从 `X-Cal-Signature-256` 获取签名
2. 使用相同的 secret 和 body 计算 HMAC-SHA256
3. 比较签名是否匹配

### 9.3 重定向保护

**位置**: `packages/features/webhooks/lib/sendPayload.ts:312`

```typescript
fetch(subscriberUrl, {
  method: "POST",
  headers: { /* ... */ },
  redirect: "manual",  // 禁止重定向
  body,
})
```

**原因**: 防止 SSRF 通过重定向访问内网

### 9.4 API Key 安全

**位置**: `packages/app-store/_utils/findValidApiKey.ts`

```typescript
function hashAPIKey(apiKey: string): string {
  return createHash("sha256").update(apiKey).digest("hex");
}
```

**安全措施**:
- 存储 SHA256 哈希，不是明文
- 验证时哈希比较
- 检查 `expiresAt` 过期时间
- 检查 `appId` 匹配

---

## 10. 监控与日志

### 10.1 日志体系

系统实现了**完整的日志闭环**，使用结构化日志：

```typescript
// 生产者入队
this.log.debug("Queueing booking webhook task", { operationId, triggerEvent, bookingUid });
this.log.debug("Webhook delivery task queued", { operationId, taskId });

// 消费者处理
this.log.debug("Processing webhook delivery task", { operationId, taskId, triggerEvent });
this.log.debug(`Found ${subscribers.length} webhook subscriber(s)`, { operationId });
this.log.warn("Event data not found", { operationId, triggerEvent });

// 投递结果
this.log.debug("Webhook sent successfully", { trigger, webhookId, statusCode });
this.log.error("Webhook failed", { error, trigger, webhookId, statusCode });

// 汇总统计
this.log.info(`Webhook processing completed for ${trigger}`, {
  totalSubscribers,
  successful: successCount,
  failed: failureCount,
});
```

### 10.2 日志级别

| 级别 | 使用场景 |
|------|---------|
| `debug` | 详细追踪信息 (入队、处理、发送) |
| `info` | 汇总统计 (处理完成) |
| `warn` | 可恢复的问题 (数据不存在) |
| `error` | 失败情况 (HTTP 非 2xx、异常) |

### 10.3 日志字段

每个日志条目包含：
- `operationId`: 操作追踪 ID (UUID)
- `taskId`: Trigger.dev 任务 ID
- `triggerEvent`: 触发事件类型
- `webhookId`: Webhook ID
- `subscriberUrl`: 目标 URL
- `statusCode`: HTTP 状态码
- `error`: 错误信息

---

## 11. 扩展指南

### 11.1 添加新的自动化工具集成

#### 步骤 1: 创建应用目录

```
packages/app-store/{tool-name}/
├── _metadata.ts (或 config.json)
├── lib/
│   └── validateAccountOrApiKey.ts (如需要)
└── api/
    └── subscriptions/
        ├── addSubscription.ts
        ├── deleteSubscription.ts
        ├── listBookings.ts
        └── me.ts
```

#### 步骤 2: 实现鉴权

参考 Zapier 或 Make 的实现：

```typescript
// 简单版本 (仅 API Key)
async function handler(req: NextApiRequest, res: NextApiResponse) {
  const apiKey = req.query.apiKey as string;
  
  if (!apiKey) {
    return res.status(401).json({ message: "No API key provided" });
  }

  const validKey = await findValidApiKey(apiKey, "your-tool-name");
  
  if (!validKey) {
    return res.status(401).json({ message: "API key not valid" });
  }

  // 继续处理...
}
```

#### 步骤 3: 实现订阅管理接口

复用 `scheduleTrigger.ts` 中的底层实现：

```typescript
import { addSubscription, deleteSubscription } from "@calcom/features/webhooks/lib/scheduleTrigger";

// 添加订阅
const createAppSubscription = await addSubscription({
  appApiKey: validKey,
  triggerEvent,
  subscriberUrl,
  appId: "your-tool-name",
});

// 删除订阅
const deleteEventSubscription = await deleteSubscription({
  appApiKey: validKey,
  webhookId: id,
  appId: "your-tool-name",
});
```

#### 步骤 4: (可选) 自定义 Payload 格式

修改 `sendPayload.ts` 中的 `getZapierPayload` 逻辑：

```typescript
// 在 sendPayload.ts 中添加自定义格式
if (appId === "your-tool-name") {
  body = getYourToolPayload({ ...data, createdAt });
}
```

### 11.2 添加新的触发事件类型

#### 步骤 1: 添加枚举值

在 `packages/prisma/schema.prisma` 中：

```prisma
enum WebhookTriggerEvents {
  // 现有事件...
  NEW_EVENT_TYPE
}
```

#### 步骤 2: 创建 DataFetcher

在 `packages/features/webhooks/lib/service/data-fetchers/` 中：

```typescript
export class NewEventWebhookDataFetcher implements IWebhookDataFetcher {
  canHandle(triggerEvent: WebhookTriggerEvents): boolean {
    return triggerEvent === WebhookTriggerEvents.NEW_EVENT_TYPE;
  }

  getSubscriberContext(payload: WebhookTaskPayload): SubscriberContext {
    // 返回订阅者查询上下文
  }

  async fetchEventData(payload: WebhookTaskPayload): Promise<WebhookEventData | null> {
    // 从数据库获取完整数据
  }
}
```

#### 步骤 3: 注册到 Consumer

在 `WebhookTaskConsumer` 中：

```typescript
constructor(/* ... */) {
  this.dataFetchers = [
    // 现有 fetchers...
    new NewEventWebhookDataFetcher(),
  ];
}
```

#### 步骤 4: 添加生产者方法

在 `WebhookTaskerProducerService` 中：

```typescript
async queueNewEventWebhook(params: NewEventParams): Promise<void> {
  const operationId = v4();
  
  this.log.debug("Queueing new event webhook task", {
    operationId,
    triggerEvent: WebhookTriggerEvents.NEW_EVENT_TYPE,
    // ... 其他参数
  });

  await this.queueTask(operationId, {
    operationId,
    triggerEvent: WebhookTriggerEvents.NEW_EVENT_TYPE,
    timestamp: new Date().toISOString(),
    // ... 其他 payload 字段
  });
}
```

---

## 附录: 关键代码索引

### Webhook 核心

| 功能 | 文件 | 行号 |
|------|------|------|
| 创建订阅 | `trpc/server/routers/viewer/webhook/create.handler.ts` | - |
| 更新订阅 | `trpc/server/routers/viewer/webhook/edit.handler.ts` | - |
| 删除订阅 | `trpc/server/routers/viewer/webhook/delete.handler.ts` | - |
| 生产者服务 | `features/webhooks/lib/service/WebhookTaskerProducerService.ts` | - |
| 消费者服务 | `features/webhooks/lib/service/WebhookTaskConsumer.ts` | 33 |
| 投递服务 | `features/webhooks/lib/service/WebhookService.ts` | 135 |
| Payload 发送 | `features/webhooks/lib/sendPayload.ts` | 217, 312 |
| Trigger 配置 | `features/webhooks/lib/tasker/trigger/config.ts` | - |
| 定时触发处理 | `features/webhooks/lib/handleWebhookScheduledTriggers.ts` | - |
| 订阅管理 | `features/webhooks/lib/scheduleTrigger.ts` | 30, 132 |

### 自动化工具

| 功能 | Zapier | Make |
|------|--------|------|
| 鉴权函数 | `app-store/zapier/lib/validateAccountOrApiKey.ts` | - |
| 添加订阅 | `app-store/zapier/api/subscriptions/addSubscription.ts` | `app-store/make/api/subscriptions/addSubscription.ts` |
| 删除订阅 | `app-store/zapier/api/subscriptions/deleteSubscription.ts` | `app-store/make/api/subscriptions/deleteSubscription.ts` |
| API Key 验证 | `app-store/_utils/findValidApiKey.ts` | 相同 |

---

**文档版本**: 1.0  
**最后更新**: 2024年  
**分析范围**: Cal.diy Webhook 系统及 Zapier/Make 自动化工具集成