# Webhook 系统架构与自动化工具接入分析

## 目录

1. [概述](#1-概述)
2. [订阅机制](#2-订阅机制)
3. [投递机制](#3-投递机制)
4. [重试机制](#4-重试机制)
5. [外部自动化工具接入](#5-外部自动化工具接入)
6. [关键文件位置](#6-关键文件位置)
7. [数据流图](#7-数据流图)

---

## 1. 概述

Cal.diy 的 Webhook 系统采用**生产者-消费者架构**，结合 Trigger.dev 任务队列实现可靠的事件通知机制。系统支持多种触发事件类型，并提供了与外部自动化工具（Zapier、Make）的集成能力。

### 核心设计原则

- **关注点分离**: 生产者负责入队，消费者负责处理
- **版本控制**: Payload 格式支持版本化
- **可靠性**: 内置重试机制和错误处理
- **安全性**: HMAC-SHA256 签名验证

---

## 2. 订阅机制

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
  active                Boolean                    @default(true)
  eventTriggers         WebhookTriggerEvents[]    // 触发事件数组
  appId                 String?                    // 关联应用 ID (zapier/make)
  secret                String?                    // 签名密钥
  platform              Boolean                    @default(false)  // 是否平台级
  time                  Int?                       // 定时触发时间
  timeUnit              TimeUnit?                  // 时间单位
  version               String                     @default("2021-10-20")
}
```

#### WebhookScheduledTriggers 表 (`packages/prisma/schema.prisma:1313`)

用于定时触发的 webhook（如 MEETING_STARTED, MEETING_ENDED）：

```prisma
model WebhookScheduledTriggers {
  id            Int       @id @default(autoincrement())
  subscriberUrl String
  payload       String    // 序列化的任务数据
  startAfter    DateTime  // 触发时间
  retryCount    Int       @default(0)  // 重试计数
  appId         String?
  webhookId     String?
  bookingId     Int?
}
```

### 2.2 触发事件类型

支持的 `WebhookTriggerEvents` 枚举 (`packages/prisma/schema.prisma:1120`)：

| 事件类型 | 描述 |
|---------|------|
| `BOOKING_CREATED` | 预订创建 |
| `BOOKING_CANCELLED` | 预订取消 |
| `BOOKING_RESCHEDULED` | 预订重新安排 |
| `BOOKING_REQUESTED` | 预订请求（待确认） |
| `BOOKING_REJECTED` | 预订被拒绝 |
| `BOOKING_PAYMENT_INITIATED` | 支付发起 |
| `BOOKING_PAID` | 支付完成 |
| `BOOKING_NO_SHOW_UPDATED` | 未到场状态更新 |
| `MEETING_STARTED` | 会议开始（定时触发） |
| `MEETING_ENDED` | 会议结束（定时触发） |
| `RECORDING_READY` | 录制就绪 |
| `RECORDING_TRANSCRIPTION_GENERATED` | 转录生成 |
| `FORM_SUBMITTED` | 表单提交 |
| `OOO_CREATED` | 外出状态创建 |
| `AFTER_HOSTS_CAL_VIDEO_NO_SHOW` | 主持人未到场 |
| `AFTER_GUESTS_CAL_VIDEO_NO_SHOW` | 嘉宾未到场 |
| `DELEGATION_CREDENTIAL_ERROR` | 委派凭证错误 |
| `WRONG_ASSIGNMENT_REPORT` | 错误分配报告 |

### 2.3 订阅者查询机制

`WebhookRepository.getSubscribers` (`packages/features/webhooks/lib/repository/WebhookRepository.ts:82`) 使用 **UNION ALL** 查询多个来源的订阅者，按优先级排序：

```typescript
// 优先级 1: Platform webhooks (平台级)
// 优先级 2: User-specific webhooks (用户级)
// 优先级 3: Event type webhooks (事件类型级)
// 优先级 4: Parent event type webhooks (父事件类型级)
// 优先级 5: Team webhooks (团队级)
// 优先级 6: OAuth client webhooks (OAuth 客户端级)
```

查询上下文 (`SubscriberContext`) 包含：
- `triggerEvent`: 触发事件类型
- `userId`: 用户 ID
- `eventTypeId`: 事件类型 ID
- `teamId`: 团队 ID（支持数组）
- `orgId`: 组织 ID
- `oAuthClientId`: OAuth 客户端 ID

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
  operationId: string;        // 操作追踪 ID
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

### 3.3 消费者 (Consumer)

**WebhookTaskConsumer** (`packages/features/webhooks/lib/service/WebhookTaskConsumer.ts`) 处理队列中的任务：

处理流程：

1. **获取 DataFetcher**: 根据 `triggerEvent` 选择对应的数据获取器
2. **获取订阅者**: 调用 `webhookRepository.getSubscribers()`
3. **获取事件数据**: 调用 `fetcher.fetchEventData()`
4. **构建 DTO**: 映射数据库数据到 `WebhookEventDTO`
5. **构建 Payload**: 通过 `PayloadBuilderFactory` 构建版本化 payload
6. **发送 Webhook**: 调用 `webhookService.processWebhooks()`

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

### 3.6 HTTP 投递

**sendPayload** (`packages/features/webhooks/lib/sendPayload.ts`) 处理实际的 HTTP 请求：

```typescript
const _sendPayload = async (
  secretKey: string | null,
  webhook: WebhookForPayload,
  body: string,
  contentType: "application/json" | "application/x-www-form-urlencoded"
) => {
  const response = await fetch(subscriberUrl, {
    method: "POST",
    headers: {
      "Content-Type": contentType,
      "X-Cal-Signature-256": createWebhookSignature({ secret: secretKey, body }),
      "X-Cal-Webhook-Version": version,
    },
    redirect: "manual",
    body,
  });

  return {
    ok: response.ok,
    status: response.status,
  };
};
```

**签名算法**:
```typescript
export const createWebhookSignature = (params: { secret?: string | null; body: string }) =>
  params.secret
    ? createHmac("sha256", params.secret).update(`${params.body}`).digest("hex")
    : "no-secret-provided";
```

---

## 4. 重试机制

### 4.1 Trigger.dev 配置

**webhookDeliveryTaskConfig** (`packages/features/webhooks/lib/tasker/trigger/config.ts`):

```typescript
export const webhookDeliveryTaskConfig: WebhookDeliveryTask = {
  machine: "small-1x",
  queue: webhookDeliveryQueue,
  retry: {
    maxAttempts: 3,                    // 最多重试 3 次
    factor: 2,                          // 指数退避因子
    minTimeoutInMs: 30000,             // 最小超时 30秒
    maxTimeoutInMs: 600000,            // 最大超时 10分钟
    randomize: true,                    // 随机化防止抖动
    outOfMemory: {                      // 内存不足时升级机器
      machine: "medium-1x",
    },
  },
};
```

### 4.2 队列配置

```typescript
export const webhookDeliveryQueue: Queue = queue({
  name: "webhook-delivery",
  concurrencyLimit: 25,  // 并发限制 25
});
```

### 4.3 错误处理

**WebhookService.processWebhooks** (`packages/features/webhooks/lib/service/WebhookService.ts:135`) 使用 `Promise.allSettled` 确保单个订阅者失败不会影响其他订阅者：

```typescript
// 使用 Promise.allSettled 防止单个 webhook 失败影响整个处理器
const results = await Promise.allSettled(promises);

// 记录摘要用于监控
const successCount = results.filter((result) => result.status === "fulfilled").length;
const failureCount = results.filter((result) => result.status === "rejected").length;
```

### 4.4 定时触发的重试

`WebhookScheduledTriggers` 表有 `retryCount` 字段用于跟踪定时触发的重试次数。

---

## 5. 外部自动化工具接入

### 5.1 支持的工具

| 工具 | appId | 集成位置 |
|------|-------|---------|
| **Zapier** | `"zapier"` | `packages/app-store/zapier/` |
| **Make** | `"make"` | `packages/app-store/make/` |

### 5.2 接入架构

```
┌─────────────────────────────────────────────────────────────┐
│                    外部自动化工具                              │
│  ┌──────────┐                    ┌──────────┐               │
│  │  Zapier  │                    │   Make   │               │
│  └────┬─────┘                    └────┬─────┘               │
└───────┼───────────────────────────────┼─────────────────────┘
        │                               │
        ▼                               ▼
┌─────────────────────────────────────────────────────────────┐
│                    API 端点层                                 │
│  ┌──────────────────────┐    ┌──────────────────────┐      │
│  │ /api/zapier/...      │    │ /api/make/...        │      │
│  │ - addSubscription    │    │ - addSubscription    │      │
│  │ - deleteSubscription │    │ - deleteSubscription │      │
│  │ - listBookings       │    │ - listBookings       │      │
│  │ - me                 │    │ - me                 │      │
│  └──────────┬───────────┘    └──────────┬───────────┘      │
└─────────────┼─────────────────────────────┼──────────────────┘
              │                             │
              ▼                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    核心订阅逻辑                               │
│              scheduleTrigger.ts (addSubscription)            │
│  - 创建 Webhook 记录 (appId = "zapier"/"make")              │
│  - 为现有预订安排定时触发                                      │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 订阅 API

#### 添加订阅

**Zapier**: `packages/app-store/zapier/api/subscriptions/addSubscription.ts`

```typescript
async function handler(req: NextApiRequest, res: NextApiResponse) {
  const { subscriberUrl, triggerEvent } = req.body;
  const { account, appApiKey } = await validateAccountOrApiKey(req, ["READ_BOOKING", "READ_PROFILE"]);
  
  const createAppSubscription = await addSubscription({
    appApiKey,
    account,
    triggerEvent: triggerEvent,
    subscriberUrl: subscriberUrl,
    appId: "zapier",  // 关键：标识为 Zapier 订阅
  });

  res.status(200).json(createAppSubscription);
}
```

**Make**: `packages/app-store/make/api/subscriptions/addSubscription.ts`

```typescript
async function handler(req: NextApiRequest, res: NextApiResponse) {
  const apiKey = req.query.apiKey as string;
  const validKey = await findValidApiKey(apiKey, "make");

  const { subscriberUrl, triggerEvent } = req.body;
  const createAppSubscription = await addSubscription({
    appApiKey: validKey,
    triggerEvent: triggerEvent,
    subscriberUrl: subscriberUrl,
    appId: "make",  // 关键：标识为 Make 订阅
  });

  res.status(200).json(createAppSubscription);
}
```

#### addSubscription 核心逻辑

`packages/features/webhooks/lib/scheduleTrigger.ts:30`

```typescript
export async function addSubscription({
  appApiKey,
  triggerEvent,
  subscriberUrl,
  appId,
  account,
}: {
  appApiKey?: ApiKey;
  triggerEvent: WebhookTriggerEvents;
  subscriberUrl: string;
  appId: string;
  account?: { id: number; name: string | null; isTeam: boolean } | null;
}) {
  const userId = appApiKey ? appApiKey.userId : account && !account.isTeam ? account.id : null;
  const teamId = appApiKey ? appApiKey.teamId : account && account.isTeam ? account.id : null;

  // 1. 创建 Webhook 记录
  const createSubscription = await prisma.webhook.create({
    data: {
      id: v4(),
      userId,
      teamId,
      eventTriggers: [triggerEvent],
      subscriberUrl,
      active: true,
      appId: appId,  // 设置 appId
    },
  });

  // 2. 对于定时触发事件 (MEETING_STARTED/MEETING_ENDED)
  // 为现有的预订安排触发
  if (
    triggerEvent === WebhookTriggerEvents.MEETING_ENDED ||
    triggerEvent === WebhookTriggerEvents.MEETING_STARTED
  ) {
    const bookings = await prisma.booking.findMany({ /* 查询现有预订 */ });
    
    for (const booking of bookings) {
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

### 5.4 特殊处理

#### 1. Zapier 自定义 Payload

`packages/features/webhooks/lib/sendPayload.ts:132`

当 `appId === "zapier"` 时，使用特殊的精简 payload 格式：

```typescript
function getZapierPayload(data: WithUTCOffsetType<EventPayloadType & { createdAt: string }>): string {
  const attendees = (data.attendees as (Person & UTCOffset)[]).map((attendee) => {
    return {
      name: attendee.name,
      email: attendee.email,
      timeZone: attendee.timeZone,
      utcOffset: attendee.utcOffset,
    };
  });

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

#### 2. 订阅过滤

`packages/features/webhooks/lib/repository/WebhookRepository.ts:43`

自动化工具的订阅在普通 webhook 列表中被过滤：

```typescript
const filterWebhooks = (webhook: { appId: string | null }): boolean => {
  const appIds = [
    "zapier",
    "make",
  ];
  return !appIds.some((appId: string) => webhook.appId === appId);
};
```

#### 3. 认证方式

**Zapier**: `packages/app-store/zapier/lib/validateAccountOrApiKey.ts`
- 支持 OAuth 账户认证
- 支持 API Key 认证
- 权限检查: `["READ_BOOKING", "READ_PROFILE"]`

**Make**: `packages/app-store/make/_utils/findValidApiKey.ts`
- API Key 认证
- 验证 appId 匹配

### 5.5 订阅管理 API 端点

| 端点 | 方法 | 功能 |
|------|------|------|
| `/api/subscriptions/add` | POST | 添加订阅 |
| `/api/subscriptions/delete` | DELETE | 删除订阅 |
| `/api/subscriptions/listBookings` | GET | 列出示例预订数据 |
| `/api/subscriptions/listOOOEntries` | GET | 列出示例外出状态 |
| `/api/subscriptions/me` | GET | 获取当前用户信息 |

---

## 6. 关键文件位置

### 6.1 核心 Webhook 功能

| 文件 | 功能 |
|------|------|
| `packages/features/webhooks/lib/service/WebhookTaskerProducerService.ts` | 生产者服务 |
| `packages/features/webhooks/lib/service/WebhookTaskConsumer.ts` | 消费者服务 |
| `packages/features/webhooks/lib/service/WebhookService.ts` | 核心服务（发送、调度） |
| `packages/features/webhooks/lib/service/WebhookNotificationHandler.ts` | 通知处理器 |
| `packages/features/webhooks/lib/repository/WebhookRepository.ts` | 数据访问层 |
| `packages/features/webhooks/lib/sendPayload.ts` | HTTP 投递实现 |
| `packages/features/webhooks/lib/scheduleTrigger.ts` | 定时触发逻辑 |

### 6.2 任务队列

| 文件 | 功能 |
|------|------|
| `packages/features/webhooks/lib/tasker/WebhookTriggerTasker.ts` | Trigger.dev 异步任务器 |
| `packages/features/webhooks/lib/tasker/WebhookSyncTasker.ts` | 同步任务器（测试用） |
| `packages/features/webhooks/lib/tasker/trigger/deliver-webhook.ts` | Trigger.dev 任务定义 |
| `packages/features/webhooks/lib/tasker/trigger/config.ts` | 重试和队列配置 |

### 6.3 数据获取器

| 文件 | 功能 |
|------|------|
| `packages/features/webhooks/lib/service/data-fetchers/BookingWebhookDataFetcher.ts` | 预订数据获取 |
| `packages/features/webhooks/lib/service/data-fetchers/PaymentWebhookDataFetcher.ts` | 支付数据获取 |
| `packages/features/webhooks/lib/service/data-fetchers/FormWebhookDataFetcher.ts` | 表单数据获取 |
| `packages/features/webhooks/lib/service/data-fetchers/RecordingWebhookDataFetcher.ts` | 录制数据获取 |
| `packages/features/webhooks/lib/service/data-fetchers/OOOWebhookDataFetcher.ts` | OOO 数据获取 |

### 6.4 自动化工具集成

| 文件 | 功能 |
|------|------|
| `packages/app-store/zapier/api/subscriptions/addSubscription.ts` | Zapier 添加订阅 |
| `packages/app-store/zapier/api/subscriptions/deleteSubscription.ts` | Zapier 删除订阅 |
| `packages/app-store/zapier/lib/validateAccountOrApiKey.ts` | Zapier 认证 |
| `packages/app-store/make/api/subscriptions/addSubscription.ts` | Make 添加订阅 |
| `packages/app-store/make/api/subscriptions/deleteSubscription.ts` | Make 删除订阅 |

### 6.5 数据模型

| 文件 | 功能 |
|------|------|
| `packages/prisma/schema.prisma` | Webhook 和 WebhookScheduledTriggers 模型 |

---

## 7. 数据流图

### 7.1 完整数据流

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           1. 事件发生                                      │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                  │
│  │ 预订创建    │    │ 预订取消    │    │ 表单提交    │                  │
│  │ Booking     │    │ Booking     │    │ Form        │                  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘                  │
└─────────┼──────────────────┼──────────────────┼─────────────────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        2. 生产者入队                                       │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │              WebhookTaskerProducerService                          │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ 1. 生成 operationId (UUID)                                  │  │   │
│  │  │ 2. 构建 WebhookTaskPayload (仅含 ID，无完整数据)            │  │   │
│  │  │ 3. 调用 WebhookTasker.deliverWebhook()                      │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
└─────────────────────────────┼─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        3. 任务队列                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                      Trigger.dev 队列                               │   │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │   │
│  │  │   Task 1    │    │   Task 2    │    │   Task 3    │          │   │
│  │  │  (待处理)   │    │  (处理中)   │    │  (已完成)   │          │   │
│  │  └─────────────┘    └─────────────┘    └─────────────┘          │   │
│  │                                                                  │   │
│  │  配置:                                                          │   │
│  │  - 队列名: webhook-delivery                                     │   │
│  │  - 并发: 25                                                      │   │
│  │  - 重试: 3次，指数退避 (30s ~ 10min)                            │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
└─────────────────────────────┼─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        4. 消费者处理                                       │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    WebhookTaskConsumer                             │   │
│  │                                                                  │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ 步骤 1: 获取 DataFetcher                                     │  │   │
│  │  │         根据 triggerEvent 选择对应的数据获取器                 │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                              │                                     │   │
│  │                              ▼                                     │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ 步骤 2: 获取订阅者                                           │  │   │
│  │  │         webhookRepository.getSubscribers()                   │  │   │
│  │  │         - Platform webhooks (优先级 1)                       │  │   │
│  │  │         - User webhooks (优先级 2)                           │  │   │
│  │  │         - EventType webhooks (优先级 3)                      │  │   │
│  │  │         - Team webhooks (优先级 5)                            │  │   │
│  │  │         - OAuth client webhooks (优先级 6)                    │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                              │                                     │   │
│  │                              ▼                                     │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ 步骤 3: 获取事件数据                                         │  │   │
│  │  │         fetcher.fetchEventData()                             │  │   │
│  │  │         - 从数据库查询完整数据                                 │  │   │
│  │  │         - 构建 CalendarEvent                                  │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                              │                                     │   │
│  │                              ▼                                     │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ 步骤 4: 构建 Payload                                         │  │   │
│  │  │         PayloadBuilderFactory.getBuilder()                   │  │   │
│  │  │         - 版本控制 (当前: 2021-10-20)                       │  │   │
│  │  │         - 支持自定义模板                                      │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                              │                                     │   │
│  │                              ▼                                     │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ 步骤 5: 发送 Webhook                                         │  │   │
│  │  │         webhookService.processWebhooks()                     │  │   │
│  │  │         - 对每个订阅者并行发送                                │  │   │
│  │  │         - Promise.allSettled 隔离失败                        │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
└─────────────────────────────┼─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        5. HTTP 投递                                        │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                         sendPayload                                │   │
│  │                                                                  │   │
│  │  请求:                                                           │   │
│  │  POST {subscriberUrl}                                            │   │
│  │                                                                  │   │
│  │  Headers:                                                        │   │
│  │  ┌────────────────────────────────────────────────────────────┐  │   │
│  │  │ Content-Type: application/json                              │  │   │
│  │  │ X-Cal-Signature-256: sha256(secret, body)                 │  │   │
│  │  │ X-Cal-Webhook-Version: 2021-10-20                          │  │   │
│  │  └────────────────────────────────────────────────────────────┘  │   │
│  │                                                                  │   │
│  │  特殊处理:                                                        │   │
│  │  - appId === "zapier": 使用精简的 Zapier payload 格式           │   │
│  │  - 支持自定义 payloadTemplate (Handlebars)                       │   │
│  └──────────────────────────┬───────────────────────────────────────┘   │
└─────────────────────────────┼─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        6. 订阅者接收                                        │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                  │
│  │  用户服务   │    │   Zapier    │    │    Make     │                  │
│  │  Webhook URL│    │   Webhook   │    │   Webhook   │                  │
│  └─────────────┘    └─────────────┘    └─────────────┘                  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 定时触发数据流 (MEETING_STARTED/MEETING_ENDED)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    自动化工具创建订阅                                       │
│  Zapier/Make 调用 addSubscription API                                     │
│  - triggerEvent: MEETING_STARTED 或 MEETING_ENDED                       │
│  - subscriberUrl: 工具的 webhook URL                                      │
│  - appId: "zapier" 或 "make"                                              │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    1. 创建 Webhook 记录                                    │
│  prisma.webhook.create({                                                  │
│    eventTriggers: [MEETING_STARTED],                                     │
│    subscriberUrl: "...",                                                  │
│    appId: "zapier",                                                       │
│    active: true                                                           │
│  })                                                                        │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    2. 为现有预订安排触发                                    │
│  查询 startTime > now() 且 status = ACCEPTED 的预订                       │
│  对每个预订:                                                                │
│    scheduleTrigger({                                                       │
│      booking,                                                              │
│      triggerEvent: MEETING_STARTED,                                       │
│      startAfter: booking.startTime  // 或 endTime                         │
│    })                                                                      │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    3. 创建 WebhookScheduledTriggers                        │
│  prisma.webhookScheduledTriggers.create({                                 │
│    subscriberUrl: "...",                                                  │
│    payload: JSON.stringify({ triggerEvent, bookingId, ... }),            │
│    startAfter: booking.startTime,  // 触发时间                            │
│    retryCount: 0,                                                          │
│    webhookId: webhook.id,                                                 │
│    bookingId: booking.id,                                                 │
│    appId: "zapier"                                                        │
│  })                                                                        │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    4. 定时任务执行                                          │
│  Cron 或任务调度器检查 startAfter <= now() 的记录                          │
│  调用 handleWebhookScheduledTriggers() 处理                               │
└──────────────────────────┬──────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    5. 发送 Webhook                                          │
│  同普通 webhook 投递流程                                                    │
│  - 构建 payload                                                            │
│  - 发送 POST 请求                                                           │
│  - 失败重试 (retryCount++)                                                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 版本控制

### 8.1 当前版本

- **版本号**: `2021-10-20`
- **默认版本**: `DEFAULT_WEBHOOK_VERSION = WebhookVersion.V_2021_10_20`

### 8.2 PayloadBuilderFactory

`packages/features/webhooks/lib/factory/versioned/PayloadBuilderFactory.ts` 管理不同版本的 payload builder：

```typescript
// 注册 builder
registry.register(WebhookVersion.V_2021_10_20, {
  [WebhookTriggerEvents.BOOKING_CREATED]: BookingPayloadBuilder,
  [WebhookTriggerEvents.BOOKING_CANCELLED]: BookingPayloadBuilder,
  // ... 其他事件类型
});

// 获取 builder
const builder = this.payloadBuilderFactory.getBuilder(version, triggerEvent);
const webhookPayload = builder.build(dto);
```

### 8.3 版本扩展

如需添加新版本：

1. 在 `WebhookVersion` 枚举中添加新版本
2. 创建对应版本的 `PayloadBuilder`
3. 在 `registry.ts` 中注册
4. 更新 `WEBHOOK_VERSION_LABELS` 和 `WEBHOOK_VERSION_DOCS`

---

## 9. 安全考虑

### 9.1 签名验证

- 算法: HMAC-SHA256
- Header: `X-Cal-Signature-256`
- 签名方式: `sha256(secret, request_body)`

订阅者应验证签名以确保请求来自 Cal.diy。

### 9.2 认证

- 普通 webhook: 使用 `secret` 字段
- 自动化工具 (Zapier/Make):
  - OAuth 2.0 认证
  - API Key 认证

### 9.3 权限检查

- 订阅管理需要 `READ_BOOKING`, `READ_PROFILE` 权限
- 团队 webhook 需要相应的团队权限

---

## 10. 监控与日志

### 10.1 日志记录

关键操作都有详细日志：

```typescript
// 生产者入队
this.log.debug("Queueing booking webhook task", {
  operationId,
  triggerEvent,
  bookingUid,
});

// 消费者处理
this.log.debug("Processing webhook delivery task", {
  operationId,
  taskId,
  triggerEvent,
});

// 发送结果
this.log.info(`Webhook processing completed for ${trigger}`, {
  totalSubscribers,
  successful: successCount,
  failed: failureCount,
});
```

### 10.2 错误日志

```typescript
this.log.error("Webhook failed", {
  error: result.message,
  trigger,
  webhookId: subscriber.id,
  statusCode: result.status,
});
```

---

## 11. 扩展指南

### 11.1 添加新的触发事件

1. 在 `WebhookTriggerEvents` 枚举中添加新事件
2. 创建或更新对应的 `DataFetcher`
3. 在 `PayloadBuilderFactory` 中注册 builder
4. 在 `WebhookTaskerProducerService` 中添加入队方法

### 11.2 集成新的自动化工具

参考 Zapier/Make 的集成方式：

1. 创建 `packages/app-store/{tool-name}/` 目录
2. 实现订阅管理 API (`addSubscription`, `deleteSubscription`)
3. 实现认证机制
4. 在 `filterWebhooks` 中添加新的 `appId`
5. 如需特殊 payload 格式，在 `sendPayload.ts` 中添加处理

---

## 附录

### A. 相关文件索引

| 模块 | 文件路径 |
|------|---------|
| 核心服务 | `packages/features/webhooks/lib/service/` |
| 数据访问 | `packages/features/webhooks/lib/repository/` |
| 任务队列 | `packages/features/webhooks/lib/tasker/` |
| Payload 构建 | `packages/features/webhooks/lib/factory/` |
| 数据获取器 | `packages/features/webhooks/lib/service/data-fetchers/` |
| Zapier 集成 | `packages/app-store/zapier/` |
| Make 集成 | `packages/app-store/make/` |
| 数据模型 | `packages/prisma/schema.prisma` |

### B. 关键接口

#### WebhookFeature Facade

```typescript
interface WebhookFeature {
  producer: IWebhookProducerService;      // 轻量级入队
  consumer: WebhookTaskConsumer;          // 消费者处理
  core: IWebhookService;                  // 核心服务
  booking: IBookingWebhookService;        // 预订事件
  recording: IRecordingWebhookService;    // 录制事件
  ooo: IOOOWebhookService;                // OOO 事件
  notifier: IWebhookNotifier;             // 通知器
  repository: IWebhookRepository;         // 数据访问
}
```

#### 使用方式

```typescript
import { getWebhookFeature } from "@calcom/features/webhooks/di";

const webhooks = getWebhookFeature();

// 入队 webhook (异步)
await webhooks.producer.queueBookingCreatedWebhook({
  bookingUid: booking.uid,
  eventTypeId: eventType.id,
});
```
