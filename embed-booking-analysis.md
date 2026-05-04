# 外部网站嵌入预约组件协作机制分析

## 一、整体架构概览

外部网站嵌入 Cal.diy 预约组件时，涉及四个核心模块的协作：

1. **Embed 脚本** (`packages/embeds/embed-core/`) - 运行在外部网站（父页面）
2. **公开页面** (`apps/web/`) - 运行在 Cal.diy 域名下的 iframe 内容
3. **跨窗口消息通信** - postMessage 协议
4. **Booking API** - tRPC 端点处理预约业务

```
┌─────────────────────────────────────────────────────────────────┐
│                     外部网站 (example.com)                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Embed 脚本 (embed.ts)                       │   │
│  │  • Cal API: inline(), modal(), floatingButton()         │   │
│  │  • doInIframe(): 发送命令到 iframe                        │   │
│  │  • ActionManager: 监听 iframe 事件                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            │                                      │
│                            ▼ postMessage()                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              <iframe src="cal.com/xxx/embed">           │   │
│  │  ┌───────────────────────────────────────────────────┐  │   │
│  │  │           公开页面 (Cal.diy 域名)                  │  │   │
│  │  │  • embed-iframe.ts: 消息监听与处理                 │  │   │
│  │  │  • Booker.tsx: 预约表单组件                        │  │   │
│  │  │  • tRPC Client: 调用 Booking API                   │  │   │
│  │  └───────────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼ HTTP/HTTPS
┌─────────────────────────────────────────────────────────────────┐
│                    Cal.diy 后端服务                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Booking API (tRPC 路由)                      │   │
│  │  • publicViewer: 公开访问（无需认证）                      │   │
│  │    - getSchedule(): 获取可用时间段                         │   │
│  │    - reserveSlot(): 预留时间段                             │   │
│  │  • viewer: 需要认证（登录用户操作）                         │   │
│  │    - confirmHandler(): 确认预约                           │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、Embed 脚本详细分析

### 2.1 核心文件结构

```
packages/embeds/embed-core/
├── src/
│   ├── embed.ts              # 主入口，Cal API 实现
│   ├── embed-iframe.ts       # iframe 侧的消息处理
│   ├── constants.ts          # 常量定义
│   ├── EmbedElement.ts       # 自定义元素基类
│   ├── sdk-action-manager.ts # 事件管理器
│   ├── types.ts              # 类型定义
│   └── embed-iframe/
│       ├── lib/
│       │   ├── embedStore.ts # iframe 状态存储
│       │   └── utils.ts      # 工具函数
│       └── react-hooks.ts    # React Hooks
├── embed-iframe-init.ts      # iframe 初始化脚本
├── ModalBox/                 # 模态框组件
├── FloatingButton/           # 浮动按钮组件
└── Inline/                   # 内联组件
```

### 2.2 初始化流程

Embed 脚本通过以下方式加载到外部网站：

```html
<script>
  (function (C, A, L) {
    let p = function (a, ar) {
      a.q.push(ar);
    };
    let d = C.document;
    C.Cal = C.Cal || function () {
      let cal = C.Cal;
      let ar = arguments;
      if (!cal.loaded) {
        cal.ns = {};
        cal.q = cal.q || [];
        d.head.appendChild(d.createElement("script")).src = A;
        cal.loaded = true;
      }
      if (ar[0] === L) {
        const api = function () {
          p(api, arguments);
        };
        const namespace = ar[1];
        api.q = api.q || [];
        typeof namespace === "string"
          ? (cal.ns[namespace] = cal.ns[namespace] || api)
          : (p(cal, ar));
        return;
      }
      p(cal, ar);
    };
  })(window, "https://cal.com/embed/embed.js", "init");
  Cal("init", { origin: "https://cal.com" });
</script>
```

**关键要点**：
1. 使用队列模式（`cal.q`）存储指令，确保脚本异步加载也能正常工作
2. 支持命名空间（`cal.ns`），允许同一页面嵌入多个预约组件
3. `origin` 参数指定 Cal.diy 服务地址

### 2.3 Cal API 核心方法

#### 2.3.1 三种嵌入模式

Embed 脚本提供三种嵌入方式：

```typescript
// 1. 内联模式 - 直接嵌入到页面指定元素
Cal("inline", {
  calLink: "username/30min",
  elementOrSelector: "#booking-container",
  config: {
    layout: "month_view",
    theme: "light"
  }
});

// 2. 模态框模式 - 点击按钮弹出预约窗口
Cal("modal", {
  calLink: "username/30min",
  config: {
    layout: "month_view"
  }
});

// 3. 浮动按钮模式 - 右下角浮动预约按钮
Cal("floatingButton", {
  calLink: "username/30min",
  buttonText: "预约咨询",
  buttonColor: "#000000",
  buttonPosition: "bottom-right"
});
```

#### 2.3.2 Iframe 创建逻辑

在 `embed.ts:301-317` 中，`createIframe` 方法负责创建 iframe：

```typescript
createIframe({
  calLink,
  config = {},
  calOrigin,
}: {
  calLink: string;
  config?: PrefillAndIframeAttrsConfigWithGuestAndColorScheme;
  calOrigin: string | null;
}) {
  const iframe = (this.iframe = document.createElement("iframe"));
  iframe.className = "cal-embed";
  iframe.name = `cal-embed=${this.namespace}`;
  iframe.title = `Book a call`;

  this.loadInIframe({ calLink, config, calOrigin, iframe });
  return iframe;
}
```

**关键处理**：
1. **URL 构造** (`embed.ts:349-385`)：
   - 自动添加 `/embed` 后缀
   - 添加 `embed` 查询参数（命名空间）
   - 支持调试模式 `debug=true`

2. **样式处理** (`embed.ts:364`)：
   ```typescript
   iframe.style.visibility = "hidden"; // 初始隐藏，等待握手完成
   ```

### 2.4 命令发送与队列机制

#### 2.4.1 doInIframe 方法

```typescript
doInIframe(doInIframeArg: DoInIframeArg) {
  if (!this.iframeReady) {
    this.iframeDoQueue.push(doInIframeArg); // 加入队列
    return;
  }
  if (!this.iframe) {
    throw new Error("iframe doesn't exist");
  }
  if (this.iframe.contentWindow) {
    this.iframe.contentWindow.postMessage(
      { originator: "CAL", method: doInIframeArg.method, arg: doInIframeArg.arg },
      "*" // 目标源为 *，接收方需要验证
    );
  }
}
```

#### 2.4.2 支持的命令方法

在 `embed-iframe.ts:345-502` 中定义了 iframe 可接收的命令：

```typescript
export const methods = {
  // UI 配置：主题、样式、布局等
  ui: function style(uiConfig: UiConfig) { ... },
  
  // 父窗口确认已收到 iframeReady 信号
  parentKnowsIframeReady: (_unused: unknown) { ... },
  
  // 连接预渲染的页面（用于 prerender 优化）
  connect: async function connect({ config, params }) { ... },
  
  // 重新加载 initiated
  __reloadInitiated: function __reloadInitiated(_unused: unknown) { ... },
};
```

---

## 三、公开页面与 iframe 初始化

### 3.1 路由配置

在 `apps/web/next.config.ts` 中配置了 embed 相关路由：

```typescript
// embed.js 脚本路由
{
  source: "/embed.js",
  destination: "/embed/embed.js",
}

// 动态 embed 页面路由
{
  source: "/:path*/embed",
  destination: "/:path*/embed", // 实际指向对应页面的 embed 版本
}

// 组织级 embed 路由
{
  source: orgUserTypeEmbedRoutePath, // e.g., /:org/:user/:type/embed
  destination: `/org/${orgSlug}/:user/:type/embed`,
}
```

### 3.2 Iframe 初始化脚本

`embed-iframe-init.ts` 负责在 iframe 加载时检测 embed 模式：

```typescript
export default function EmbedInitIframe() {
  if (typeof window === "undefined" || window.isEmbed) {
    return;
  }

  const url = new URL(document.URL);
  const embedNameSpaceFromQueryParam = url.searchParams.get("embed");
  const hasEmbedPath = url.pathname.endsWith("/embed");
  const defaultNamespace = "";

  // 命名空间检测优先级：
  // 1. URL 查询参数 ?embed=xxx
  // 2. window.name (iframe 创建时设置，持久化)
  // 3. 路径后缀 /embed
  const embedNamespace =
    typeof embedNameSpaceFromQueryParam === "string"
      ? embedNameSpaceFromQueryParam
      : window.name.includes("cal-embed=")
      ? window.name.replace(/cal-embed=(.*)/, "$1").trim()
      : hasEmbedPath
      ? defaultNamespace
      : null;

  window.isEmbed = () => {
    return typeof embedNamespace == "string";
  };

  window.getEmbedNamespace = () => {
    return embedNamespace;
  };
  // ...
}
```

### 3.3 公开页面的 Booker 组件

`Booker.tsx` 是预约表单的核心组件，在 embed 模式下有特殊处理：

```typescript
const BookerComponent = ({ ... }) => {
  const searchParams = useCompatSearchParams();
  const isPlatformBookerEmbed = useIsPlatformBookerEmbed();
  
  // 使用 embed 相关的 hooks
  const embedUiConfig = useEmbedUiConfig();
  
  // 更新 embed 状态
  useEffect(() => {
    if (slotsQuery && bookerState) {
      updateEmbedBookerState({
        bookerState,
        slotsQuery,
      });
    }
  }, [bookerState, slotsQuery]);
  
  // ...
};
```

---

## 四、跨窗口消息通信协议

### 4.1 协议概述

通信基于 `postMessage` API，使用统一的消息格式：

```typescript
// 父窗口 -> Iframe (命令)
{
  originator: "CAL",
  method: "ui" | "parentKnowsIframeReady" | "connect",
  arg: { ... } // 方法参数
}

// Iframe -> 父窗口 (事件)
{
  originator: "CAL",
  type: "__iframeReady" | "linkReady" | "__dimensionChanged",
  data: { ... } // 事件数据
}
```

### 4.2 握手协议（Handshake）

详细的握手流程如下（参考 `embed-handshake.mermaid`）：

```
┌─────────────┐         ┌─────────────┐         ┌─────────────┐
│  父窗口     │         │ postMessage │         │   Iframe    │
│  (embed.ts) │         │  Transport  │         │(embed-iframe)│
└──────┬──────┘         └──────┬──────┘         └──────┬──────┘
       │                       │                       │
       │  PHASE 1: Iframe 创建                        │
       │                       │                       │
       ├─── 创建 iframe ────────│──────────────────────>│
       │   - 设置 src           │                       │
       │   - visibility: hidden │                       │
       │   - iframeReady=false  │                       │
       │                       │                       │ 开始加载页面
       │                       │                       │ body隐藏
       │                       │                       │
       │  PHASE 2: 初始化                           │
       │                       │                       │
       │                       │                       ├─── main() 执行
       │                       │                       │    - 解析 URL 参数
       │                       │                       │    - 初始化 embedStore
       │                       │                       │
       │  PHASE 3: __iframeReady 事件                │
       │                       │                       │
       │<── postMessage ────────│<── fire("__iframeReady") ──│
       │   {                    │                       │
       │     originator: "CAL", │                       │
       │     type: "__iframeReady",                    │
       │     data: { isPrerendering }                  │
       │   }                │                       │
       │                       │                       │
       ├─── iframeReady = true │                       │
       ├─── 显示 iframe (非 prerender)                 │
       │                       │                       │
       │  PHASE 4: parentKnowsIframeReady 响应        │
       │                       │                       │
       ├─── doInIframe() ──────│──────────────────────>│
       │   {                    │                       │
       │     method: "parentKnowsIframeReady"         │
       │   }                │                       │
       │                       │                       │
       │                       │                       ├─── 等待 isLinkReady()
       │                       │                       │    - 页面渲染完成
       │                       │                       │    - 插槽加载完成
       │                       │                       │
       │                       │                       ├─── makeBodyVisible()
       │                       │                       │    - renderState = "completed"
       │                       │                       │
       │<── postMessage ────────│<── fire("linkReady") ───────│
       │   (或 "linkPrerendered" 如果是 prerender)    │
       │                       │                       │
       │  PHASE 5: 队列刷新                            │
       │                       │                       │
       ├─── 处理 iframeDoQueue  │                       │
       │    中的所有命令          │                       │
       │                       │                       │
       │  握手完成！双向通信建立  │                       │
       │                       │                       │
```

### 4.3 消息协议架构

参考 `embed-message-protocol.mermaid`：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Parent Page (embed.ts)                        │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │   Cal API    │───>│ doInIframe() │───>│iframeDoQueue │     │
│  │ inline/modal │    │  命令发送者   │    │   命令缓冲    │     │
│  └──────────────┘    └──────────────┘    └──────┬───────┘     │
│         │                                          │              │
│         ▼                                          ▼              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              ActionManager (事件监听器)                    │   │
│  │  - on("linkReady", callback)                              │   │
│  │  - on("__dimensionChanged", callback)                    │   │
│  │  - on("bookingSuccessful", callback)                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼ postMessage()
┌─────────────────────────────────────────────────────────────────┐
│                    postMessage Transport                          │
│                                                                   │
│  父 -> Iframe:                                                    │
│  { originator: "CAL", method: "ui", arg: {...} }               │
│                                                                   │
│  Iframe -> 父:                                                    │
│  { originator: "CAL", type: "linkReady", data: {...} }         │
│                                                                   │
│  ⚠️  targetOrigin 使用 "*"，接收方通过 originator: "CAL" 验证    │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Iframe (embed-iframe.ts)                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │        window.addEventListener('message')                 │   │
│  │        消息监听器                                           │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                         │                                          │
│                         ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │        interfaceWithParent (方法注册表)                    │   │
│  │  {                                                        │   │
│  │    ui: (uiConfig) => {...},                              │   │
│  │    parentKnowsIframeReady: () => {...},                 │   │
│  │    connect: ({config, params}) => {...}                 │   │
│  │  }                                                        │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                         │                                          │
│                         ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              sdkActionManager (事件发射器)                 │   │
│  │  - fire("__iframeReady", data)                           │   │
│  │  - fire("linkReady", data)                               │   │
│  │  - fire("__dimensionChanged", {iframeHeight})           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                         │                                          │
│                         ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   embedStore (状态存储)                    │   │
│  │  - namespace: string | null                               │   │
│  │  - renderState: "inProgress" | "completed"               │   │
│  │  - styles: EmbedStyles                                    │   │
│  │  - viewId: number (用于追踪视图次数)                      │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 核心事件类型

#### 4.4.1 Iframe -> 父窗口事件

```typescript
// 系统事件（__ 前缀）
"__iframeReady"      // Iframe 加载完成，开始握手
"__dimensionChanged" // 内容尺寸变化，用于调整 iframe 大小
"__routeChanged"     // 路由变化
"__closeIframe"      // 关闭模态框
"__scrollByDistance" // 滚动请求
"__connectInitiated" // 连接预渲染页面开始
"__connectCompleted" // 连接预渲染页面完成

// 业务事件
"linkReady"          // 链接准备就绪（可交互）
"linkPrerendered"    // 预渲染完成
"linkFailed"         // 链接加载失败
"bookingSuccessful"  // 预约成功
"bookingFailed"      // 预约失败
```

#### 4.4.2 父窗口 -> Iframe 命令

```typescript
"ui"                    // 更新 UI 配置（主题、样式等）
"parentKnowsIframeReady" // 父窗口确认收到 iframeReady
"connect"               // 连接预渲染的页面
"__reloadInitiated"     // 重新加载 initiated
```

### 4.5 消息监听器实现

在 `embed-iframe.ts:559-568` 中：

```typescript
window.addEventListener("message", (e) => {
  const data: Message = e.data;
  if (!data) {
    return;
  }
  const method: keyof typeof interfaceWithParent = data.method;
  // 验证消息来源：必须有 originator: "CAL"
  if (data.originator === "CAL" && typeof method === "string") {
    interfaceWithParent[method]?.(data.arg as never);
  }
});
```

**安全注意**：虽然 `postMessage` 使用 `targetOrigin: "*"`，但通过 `originator: "CAL"` 字段验证消息身份，防止恶意消息。

---

## 五、Booking API 与认证边界

### 5.1 API 架构分层

Cal.diy 使用 tRPC 提供 API，分为两个主要路由：

```
┌─────────────────────────────────────────────────────────────────┐
│                        tRPC 路由架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  publicViewerRouter                       │   │
│  │  （无需认证，公开访问）                                    │   │
│  │                                                           │   │
│  │  • event.handler.ts      - 获取公开事件信息               │   │
│  │  • getSchedule.handler.ts - 获取可用时间段               │   │
│  │  • reserveSlot.handler.ts - 预留时间段                   │   │
│  │  • timezones/            - 时区相关                       │   │
│  │  • countryCode.handler.ts- 国家代码                       │   │
│  │  • submitRating.handler.ts- 提交评价                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    viewerRouter                           │   │
│  │  （需要认证，登录用户操作）                                │   │
│  │                                                           │   │
│  │  • bookings/confirm.handler.ts - 确认/拒绝预约           │   │
│  │  • eventTypes/            - 事件类型管理                  │   │
│  │  • calendars/             - 日历管理                      │   │
│  │  • me/                    - 当前用户信息                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 路由注册

#### 5.2.1 publicViewer 路由

`apps/web/pages/api/trpc/public/[trpc].ts`：

```typescript
import { createNextApiHandler } from "@calcom/trpc/server/createNextApiHandler";
import { publicViewerRouter } from "@calcom/trpc/server/routers/publicViewer/_router";

export default createNextApiHandler(publicViewerRouter, true);
```

#### 5.2.2 主路由

`packages/trpc/server/routers/_app.ts`：

```typescript
export const appRouter = router({
  viewer: viewerRouter, // 需要认证
});
```

### 5.3 公开 API 详细分析

#### 5.3.1 publicProcedure 定义

`packages/trpc/server/procedures/publicProcedure.ts`：

```typescript
import { errorConversionMiddleware } from "../middlewares/errorConversionMiddleware";
import perfMiddleware from "../middlewares/perfMiddleware";
import { tRPCContext } from "../trpc";

// 无需认证的 procedure
const publicProcedure = tRPCContext.procedure
  .use(perfMiddleware)
  .use(errorConversionMiddleware);

export default publicProcedure;
```

#### 5.3.2 获取公开事件

`packages/trpc/server/routers/publicViewer/event.handler.ts`：

```typescript
import { EventRepository } from "@calcom/features/eventtypes/repositories/EventRepository";

export const eventHandler = async ({ input, userId }: EventHandlerOptions) => {
  // 直接调用 Repository，无需权限检查（事件是公开的）
  return await EventRepository.getPublicEvent(input, userId);
};
```

#### 5.3.3 预留时间段（关键 API）

`packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts`：

```typescript
export const reserveSlotHandler = async ({ ctx, input }: ReserveSlotOptions) => {
  const { prisma, req, res } = ctx;
  // 使用 uid cookie 追踪用户（无需登录）
  const uid = req?.cookies?.uid || uuid();

  const { slotUtcStartDate, slotUtcEndDate, eventTypeId, _isDryRun } = input;
  const releaseAt = dayjs.utc().add(parseInt(MINUTES_TO_BOOK), "minutes").format();
  
  // ... 检查时间段是否可用

  if (eventType && shouldReserveSlot && !reservedBySomeoneElse && !_isDryRun) {
    await Promise.all(
      eventType.users.map((user) =>
        prisma.selectedSlots.upsert({
          where: { 
            selectedSlotUnique: { 
              userId: user.id, 
              slotUtcStartDate, 
              slotUtcEndDate, 
              uid 
            } 
          },
          update: {
            slotUtcStartDate,
            slotUtcEndDate,
            releaseAt,
            eventTypeId,
          },
          create: {
            userId: user.id,
            eventTypeId,
            slotUtcStartDate,
            slotUtcEndDate,
            uid,
            releaseAt,
            isSeat: eventType.seatsPerTimeSlot !== null,
          },
        })
      )
    );
  }

  // ⚠️ 跨域 Cookie 关键处理
  // 对于第三方 iframe 嵌入场景，需要 SameSite=None 和 Secure
  const useSecureCookies = WEBAPP_URL.startsWith("https://");
  res?.setHeader(
    "Set-Cookie",
    serialize("uid", uid, {
      path: "/",
      sameSite: useSecureCookies ? "none" : "lax",
      secure: useSecureCookies,
    })
  );

  return { uid: uid };
};
```

**关键点分析**：
1. **无认证设计**：预留时间段不需要用户登录，使用 `uid` cookie 追踪
2. **跨域 Cookie**：
   - HTTPS 环境：`SameSite=None` + `Secure`
   - HTTP 开发环境：`SameSite=Lax`
3. **锁机制**：使用 `selectedSlots` 表实现时间段预留
4. **自动释放**：`releaseAt` 字段指定预留过期时间

### 5.4 预约创建流程

#### 5.4.1 创建 Booking

`packages/features/bookings/lib/handleNewBooking/createBooking.ts`：

```typescript
const _createBooking = async ({
  uid,
  reqBody,
  eventType,
  input,
  evt,
  originalRescheduledBooking,
  rescheduledBy,
  creationSource,
  tracking,
}: CreateBookingParams & { rescheduledBy: string | undefined }) => {
  updateEventDetails(evt, originalRescheduledBooking);

  const bookingAndAssociatedData = buildNewBookingData({
    uid,
    rescheduledBy,
    reqBody,
    eventType,
    input,
    evt,
    originalRescheduledBooking,
    creationSource,
    tracking,
  });

  return await saveBooking(
    bookingAndAssociatedData,
    originalRescheduledBooking,
    eventType.paymentAppData,
    eventType.organizerUser
  );
};

async function saveBooking(...) {
  return prisma.$transaction(async (tx) => {
    // 如果是改期，先取消原预约
    if (originalBookingUpdateDataForCancellation) {
      await tx.booking.update(originalBookingUpdateDataForCancellation);
    }

    // 创建新预约
    const booking = await tx.booking.create(createBookingObj);

    return { ...booking, userUuid: booking.user?.uuid ?? null };
  });
}
```

#### 5.4.2 确认预约（需要认证）

`packages/trpc/server/routers/viewer/bookings/confirm.handler.ts`：

```typescript
export const confirmHandler = async ({ ctx, input }: ConfirmOptions) => {
  const { user } = ctx; // 从认证上下文获取当前用户

  const booking = await prisma.booking.findUniqueOrThrow({
    where: { id: bookingId },
    select: { ... },
  });

  // 权限检查：用户必须有权限确认该预约
  const bookingAccessService = new BookingAccessService(prisma);
  const isUserAuthorizedToConfirmBooking = 
    await bookingAccessService.doesUserIdHaveAccessToBooking({
      userId: ctx.user.id,
      bookingId: bookingId,
    });

  if (!isUserAuthorizedToConfirmBooking) {
    throw new TRPCError({
      code: "UNAUTHORIZED",
      message: "User is not authorized to confirm this booking",
    });
  }

  // 确认逻辑...
  if (confirmed) {
    await handleConfirmation({
      user: { ...user, credentials: allCredentials },
      evt,
      recurringEventId,
      prisma,
      bookingId,
      booking,
      emailsEnabled,
      platformClientParams,
      traceContext,
    });
  } else {
    // 拒绝逻辑...
  }

  return { message, status };
};
```

### 5.5 认证边界总结

| 操作 | API 路由 | 认证要求 | 追踪方式 |
|------|---------|---------|---------|
| 查看公开事件 | publicViewer.event | 无需 | 无 |
| 查看可用时间 | publicViewer.getSchedule | 无需 | 无 |
| 预留时间段 | viewer.slots.reserveSlot | 无需 | uid Cookie |
| 创建预约 | 内部服务调用 | 无需 | uid Cookie |
| 确认预约 | viewer.bookings.confirm | 需要登录 | Session |
| 取消预约 | viewer.bookings.cancel | 需要登录 | Session |
| 改期预约 | viewer.bookings.requestReschedule | 需要登录 | Session |

---

## 六、跨域处理机制

### 6.1 跨域场景分析

嵌入预约组件涉及三个层次的跨域：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        跨域场景层次                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  层次 1: 脚本加载跨域                                                │
│  ┌──────────────┐         ┌──────────────────────┐                 │
│  │ example.com  │ ──────> │ cal.com/embed.js     │                 │
│  │  (外部网站)   │  <script>  (Embed 脚本)       │                 │
│  └──────────────┘         └──────────────────────┘                 │
│  解决方案: CORS 头部 + script 标签天然支持跨域                      │
│                                                                      │
│  层次 2: iframe 父子通信跨域                                         │
│  ┌──────────────────────────────────────────────────────┐          │
│  │ 父窗口: example.com                                   │          │
│  │   ┌──────────────────────────────────────────────┐   │          │
│  │   │ <iframe src="https://cal.com/xxx/embed">    │   │          │
│  │   │                                              │   │          │
│  │   │   子窗口: cal.com (不同域名)                  │   │          │
│  │   └──────────────────────────────────────────────┘   │          │
│  └──────────────────────────────────────────────────────┘          │
│  解决方案: postMessage API (targetOrigin="*")                       │
│                                                                      │
│  层次 3: API 请求跨域（iframe 内）                                   │
│  ┌──────────────┐         ┌──────────────────────┐                 │
│  │ cal.com      │ ──────> │ api.cal.com 或      │                 │
│  │  (iframe 内) │         │ cal.com/api/trpc/... │                 │
│  └──────────────┘         └──────────────────────┘                 │
│  解决方案: 同域名请求，无跨域问题；Cookie 需特殊处理                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.2 PostMessage 跨域通信

#### 6.2.1 发送端（父窗口）

`embed.ts:397-413`：

```typescript
doInIframe(doInIframeArg: DoInIframeArg) {
  // ...
  if (this.iframe.contentWindow) {
    // ⚠️ targetOrigin 使用 "*"，允许发送到任何域名
    // 这是设计使然：embed 脚本可能嵌入到任意外部网站
    this.iframe.contentWindow.postMessage(
      { originator: "CAL", method: doInIframeArg.method, arg: doInIframeArg.arg },
      "*"
    );
  }
}
```

#### 6.2.2 接收端（Iframe）

`embed-iframe.ts:559-568`：

```typescript
window.addEventListener("message", (e) => {
  const data: Message = e.data;
  if (!data) return;
  
  // 安全验证：只处理 originator: "CAL" 的消息
  // 这防止了恶意页面通过 postMessage 发送伪造命令
  if (data.originator === "CAL" && typeof data.method === "string") {
    interfaceWithParent[data.method]?.(data.arg as never);
  }
});
```

**安全策略总结**：
- **发送方**：使用 `targetOrigin="*"`，因为无法预知外部网站域名
- **接收方**：通过 `originator: "CAL"` 字段验证消息身份
- **补充**：重要操作应结合服务器端验证

### 6.3 Cookie 跨域处理

#### 6.3.1 关键代码

`reserveSlot.handler.ts:114-122`：

```typescript
const useSecureCookies = WEBAPP_URL.startsWith("https://");
res?.setHeader(
  "Set-Cookie",
  serialize("uid", uid, {
    path: "/",
    // 跨域 iframe 场景需要 SameSite=None
    sameSite: useSecureCookies ? "none" : "lax",
    // SameSite=None 必须配合 Secure
    secure: useSecureCookies,
  })
);
```

#### 6.3.2 Cookie 策略说明

| 环境 | SameSite | Secure | 说明 |
|------|---------|--------|------|
| HTTPS 生产 | `None` | `true` | 允许第三方 iframe 访问 |
| HTTP 开发 | `Lax` | `false` | 本地开发不需要 Secure |

#### 6.3.3 为什么需要这样配置？

```
场景：用户在 example.com 嵌入 cal.com 的预约组件

1. 用户访问 https://example.com
2. 页面创建 iframe 指向 https://cal.com/user/30min/embed
3. iframe 内的页面请求 cal.com 的 API（如 reserveSlot）
4. 服务器设置 Cookie: uid=xxx; SameSite=None; Secure
5. 后续请求（如创建预约）会携带这个 uid Cookie

如果不设置 SameSite=None：
- 浏览器会阻止第三方 Cookie
- 每次请求都会生成新的 uid
- 无法关联预留的时间段和创建的预约
```

### 6.4 CORS 配置

Embed 脚本通过 `<script>` 标签加载，天然支持跨域。API 请求在 iframe 内发送到同源域名，因此不需要 CORS 配置。

---

## 七、预渲染（Prerender）优化机制

### 7.1 预渲染概述

为了提升用户体验，Cal.diy 支持预渲染预约页面，在用户点击预约按钮前提前加载 iframe。

### 7.2 预渲染 API

```typescript
// 预渲染预约页面
Cal("prerender", {
  calLink: "username/30min",
  type: "modal", // "modal" 或 "floatingButton"
  options: {
    slotsStaleTimeMs: 60000,        // 插槽过期时间（默认 1 分钟）
    iframeForceReloadThresholdMs: 900000, // iframe 强制重载时间（默认 15 分钟）
  }
});

// 或者使用 preload
Cal("preload", {
  calLink: "username/30min",
  type: "modal",
  options: {
    prerenderIframe: true, // 设置为 true 时等同于 prerender
  }
});
```

### 7.3 预渲染流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      预渲染流程                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. 调用 prerender 指令                                          │
│     Cal("prerender", { calLink: "xxx", type: "modal" })       │
│                              │                                    │
│                              ▼                                    │
│  2. 创建隐藏的 iframe（visibility: hidden）                      │
│     - 设置 isPrerendering = true                                 │
│     - iframe 状态: "prerendering"                                │
│                              │                                    │
│                              ▼                                    │
│  3. Iframe 内部执行                                              │
│     - 加载页面                                                    │
│     - 获取事件信息                                                │
│     - （可选）获取可用时间段                                       │
│     - 触发 __iframeReady 事件                                     │
│                              │                                    │
│                              ▼                                    │
│  4. 握手完成，触发 linkPrerendered 事件（而非 linkReady）       │
│     - iframe 保持隐藏                                             │
│     - 等待用户操作                                                 │
│                              │                                    │
│                              ▼                                    │
│  5. 用户点击预约按钮                                              │
│     - 调用 connect 方法                                           │
│     - 传递最新的配置参数                                           │
│                              │                                    │
│                              ▼                                    │
│  6. 连接预渲染页面                                                │
│     - 更新 URL 查询参数                                           │
│     - （可选）重新获取插槽（如果过期）                             │
│     - 触发 linkReady 事件                                         │
│                              │                                    │
│                              ▼                                    │
│  7. 显示 iframe，用户可直接交互                                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 7.4 关键常量

`constants.ts`：

```typescript
// 插槽过期时间 - 1 分钟
// 超过此时间，打开模态框时会重新获取插槽
export const EMBED_MODAL_IFRAME_SLOT_STALE_TIME = 60 * 1000;

// iframe 强制重载时间 - 15 分钟
// 超过此时间，会完全重新加载 iframe
export const EMBED_MODAL_IFRAME_FORCE_RELOAD_THRESHOLD_MS = 15 * 60 * 1000;

// 预渲染防抖动阈值 - 1 分钟
// 防止短时间内重复预渲染相同链接
export const EMBED_MODAL_PRERENDER_PREVENT_THRESHOLD_MS = 1 * 60 * 1000;
```

### 7.5 Connect 方法

`embed-iframe.ts:449-497`：

```typescript
connect: async function connect({
  config,
  params,
}: {
  config: PrefillAndIframeAttrsConfig;
  params: Record<string, string | string[]>;
}) {
  sdkActionManager?.fire("__connectInitiated", {});
  
  const {
    iframeAttrs: _1,
    "cal.embed.noSlotsFetchOnConnect": noSlotsFetchOnConnect,
    ...queryParamsFromConfig
  } = config;
  
  // 重置高度追踪
  embedStore.providedCorrectHeightToParent = false;

  if (noSlotsFetchOnConnect !== "true") {
    // 增加 connectVersion，强制重新获取插槽
    embedStore.connectVersion = embedStore.connectVersion + 1;
  }

  // 确保 URL 参数正确
  await connectPreloadedEmbed({
    toBeThereParams: {
      ...params,
      ...queryParamsFromConfig,
      "cal.embed.connectVersion": connectVersion.toString(),
    },
    toRemoveParams: ["preload", "prerender", "cal.skipSlotsFetch"],
  });
}
```

---

## 八、完整预约流程时序图

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 外部网站  │    │ Embed脚本 │    │ Iframe   │    │tRPC Client│   │  后端    │
│(example.com)  │ (embed.ts)│    │(cal.com) │    │          │    │ (API)    │
└────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘
     │               │               │               │               │
     │  用户访问页面  │               │               │               │
     │──────────────>│               │               │               │
     │               │               │               │               │
     │  1. 初始化 Embed               │               │               │
     │               │               │               │               │
     │  Cal("init",                  │               │               │
     │    {origin: "cal.com"})      │               │               │
     │──────────────>│               │               │               │
     │               │               │               │               │
     │  2. 创建 iframe               │               │               │
     │               │               │               │               │
     │  Cal("modal",                 │               │               │
     │    {calLink: "u/30min"})     │               │               │
     │──────────────>│               │               │               │
     │               │               │               │               │
     │               │── 创建 iframe │               │               │
     │               │── src="cal.com/u/30min/embed"│               │
     │               │               │               │               │
     │               │──────────────>│               │               │
     │               │               │               │               │
     │               │               │── 加载页面    │               │
     │               │               │               │               │
     │  3. 握手协议  │               │               │               │
     │               │               │               │               │
     │               │<── postMessage ───────────────│               │
     │               │   __iframeReady               │               │
     │               │               │               │               │
     │               │── iframeReady = true          │               │
     │               │               │               │               │
     │               │── postMessage ───────────────>│               │
     │               │   parentKnowsIframeReady      │               │
     │               │               │               │               │
     │               │<── postMessage ───────────────│               │
     │               │   linkReady                   │               │
     │               │               │               │               │
     │               │── 显示 iframe │               │               │
     │               │               │               │               │
     │  4. 用户选择时间              │               │               │
     │               │               │               │               │
     │<──────────────│<──────────────│               │               │
     │  用户看到日历  │               │               │               │
     │               │               │               │               │
     │               │               │── 获取事件信息│               │
     │               │               │──────────────>│               │
     │               │               │               │── publicViewer.event
     │               │               │               │──────────────>│
     │               │               │               │               │
     │               │               │               │<──────────────│
     │               │               │<──────────────│               │
     │               │               │               │               │
     │               │               │── 获取可用时间│               │
     │               │               │──────────────>│               │
     │               │               │               │── publicViewer.getSchedule
     │               │               │               │──────────────>│
     │               │               │               │               │
     │               │               │               │<──────────────│
     │               │               │<──────────────│               │
     │               │               │               │               │
     │  用户选择时间段              │               │               │
     │──────────────>│               │               │               │
     │               │               │               │               │
     │  5. 预留时间段               │               │               │
     │               │               │               │               │
     │               │               │──────────────>│               │
     │               │               │               │── viewer.slots.reserveSlot
     │               │               │               │──────────────>│
     │               │               │               │               │
     │               │               │               │   设置 Cookie │
     │               │               │               │   uid=xxx;   │
     │               │               │               │   SameSite=None│
     │               │               │               │   Secure      │
     │               │               │               │               │
     │               │               │               │<──────────────│
     │               │               │<──────────────│               │
     │               │               │               │               │
     │  6. 用户填写表单              │               │               │
     │               │               │               │               │
     │──────────────>│               │               │               │
     │  提交预约     │               │               │               │
     │──────────────>│               │               │               │
     │               │               │               │               │
     │  7. 创建预约                 │               │               │
     │               │               │               │               │
     │               │               │──────────────>│               │
     │               │               │               │── 内部服务调用 │
     │               │               │               │   createBooking │
     │               │               │               │──────────────>│
     │               │               │               │               │
     │               │               │               │   使用 uid    │
     │               │               │               │   关联预留槽  │
     │               │               │               │               │
     │               │               │               │<──────────────│
     │               │               │               │               │
     │               │               │<── postMessage ───────────────│
     │               │               │   bookingSuccessful           │
     │               │               │               │               │
     │               │<── postMessage ───────────────│               │
     │               │   bookingSuccessful           │               │
     │               │               │               │               │
     │<──────────────│               │               │               │
     │  预约成功！   │               │               │               │
     │               │               │               │               │
     │  （可选）确认预约             │               │               │
     │  （需要组织者登录）           │               │               │
     │               │               │               │               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 九、安全考虑与最佳实践

### 9.1 安全机制总结

| 安全层面 | 实现方式 | 说明 |
|---------|---------|------|
| 消息验证 | `originator: "CAL"` | 防止恶意 postMessage |
| Cookie 安全 | `SameSite=None` + `Secure` | 仅在 HTTPS 下允许第三方访问 |
| 时间段锁 | `selectedSlots` 表 + `uid` | 防止重复预订 |
| 权限检查 | `BookingAccessService` | 确认/取消需要验证权限 |
| API 分层 | publicViewer vs viewer | 明确区分公开和认证操作 |

### 9.2 潜在风险与缓解

#### 风险 1: postMessage 目标源为 "*"

**风险**：理论上任何窗口都可以接收消息

**缓解**：
- 消息包含 `originator: "CAL"` 标识
- 敏感操作不在 postMessage 中传递
- 关键操作依赖服务器端验证

#### 风险 2: 第三方 Cookie 依赖

**风险**：部分浏览器可能阻止 `SameSite=None` 的 Cookie

**缓解**：
- 使用 uid 作为关联标识，而非 session
- 预留和创建在同一会话中完成
- 提供备用方案（如直接跳转预约页面）

#### 风险 3: XSS 攻击

**风险**：embed 脚本可能被注入恶意代码

**缓解**：
- 使用 `textContent` 而非 `innerHTML`
- 对用户输入进行验证（`validate` 函数）
- 自定义元素（Custom Elements）沙箱化

### 9.3 验证机制

`embed.ts:108-143` 中的输入验证：

```typescript
function validate(data: Record<string, unknown>, schema: ValidationSchema) {
  function checkType(value: unknown, expectedType: ValidationSchemaPropType) {
    if (typeof expectedType === "string") {
      return typeof value == expectedType;
    } else {
      return value instanceof expectedType;
    }
  }

  // 必填检查
  if (schema.required && isUndefined(data)) {
    throw new Error("Argument is required");
  }

  // 类型检查
  for (const [prop, propSchema] of Object.entries(schema.props || {})) {
    if (propSchema.required && isUndefined(data[prop])) {
      throw new Error(`"${prop}" is required`);
    }
    // 类型验证...
  }
}
```

---

## 十、关键文件索引

### 10.1 Embed 核心

| 文件路径 | 职责 |
|---------|------|
| `packages/embeds/embed-core/src/embed.ts` | 父窗口 API 实现、iframe 管理 |
| `packages/embeds/embed-core/src/embed-iframe.ts` | iframe 消息监听、事件发射 |
| `packages/embeds/embed-core/src/constants.ts` | 超时时间、阈值常量 |
| `packages/embeds/embed-core/src/embed-iframe/lib/embedStore.ts` | iframe 状态存储 |
| `packages/embeds/embed-core/embed-iframe-init.ts` | iframe embed 模式检测 |

### 10.2 消息协议

| 文件路径 | 职责 |
|---------|------|
| `packages/embeds/embed-core/src/sdk-action-manager.ts` | 事件管理器（SDK 侧） |
| `packages/embeds/embed-core/src/sdk-event.ts` | 事件发射器（iframe 侧） |
| `packages/embeds/embed-message-protocol.mermaid` | 消息协议流程图 |
| `packages/embeds/embed-handshake.mermaid` | 握手协议时序图 |

### 10.3 Booking API

| 文件路径 | 职责 |
|---------|------|
| `packages/trpc/server/routers/publicViewer/_router.ts` | 公开 API 路由 |
| `packages/trpc/server/routers/publicViewer/event.handler.ts` | 获取公开事件 |
| `packages/trpc/server/routers/viewer/slots/reserveSlot.handler.ts` | 预留时间段 |
| `packages/trpc/server/routers/viewer/bookings/confirm.handler.ts` | 确认预约 |
| `packages/features/bookings/lib/handleNewBooking/createBooking.ts` | 创建预约核心逻辑 |

### 10.4 路由配置

| 文件路径 | 职责 |
|---------|------|
| `apps/web/next.config.ts` | embed 路由重写规则 |
| `apps/web/pages/api/trpc/public/[trpc].ts` | publicViewer API 入口 |
| `apps/web/pages/api/trpc/[trpc].ts` | 主 API 入口 |

---

## 十一、总结

### 11.1 核心设计理念

1. **无状态前端**：嵌入组件不依赖外部网站的状态
2. **分层 API**：明确区分公开操作和认证操作
3. **安全通信**：postMessage + 消息验证 + Cookie 策略
4. **性能优化**：预渲染机制、队列缓冲、增量更新

### 11.2 跨域解决方案

| 问题 | 解决方案 |
|------|---------|
| 脚本加载跨域 | script 标签天然支持 |
| 父子窗口通信 | postMessage + originator 验证 |
| 第三方 Cookie | SameSite=None + Secure（HTTPS） |
| API 请求跨域 | iframe 内同源请求 |

### 11.3 认证边界

- **访客操作**（查看时间、预留、创建预约）：使用 `uid` Cookie 追踪，无需登录
- **主人操作**（确认、取消、改期）：需要 Session 认证

这种设计允许访客无需登录即可完成预约，同时保护了主人的敏感操作。
