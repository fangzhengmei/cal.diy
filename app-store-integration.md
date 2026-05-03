# App Store 集成插件接入核心预约流程技术文档

## 概述

本文档详细描述了 Cal.diy App Store 中的集成插件如何从注册到被核心预约流程触发的完整技术路径。以 Zoom 视频会议插件为例，深入分析插件的架构、注册机制、以及与预约流程的集成点。

---

## 1. 插件目录结构与文件组织

### 1.1 标准插件目录结构

每个 App Store 插件位于 `packages/app-store/[app-slug]/` 目录下，遵循统一的结构：

```
packages/app-store/
├── zoomvideo/                    # 插件目录名（通常与 appId 对应）
│   ├── _metadata.ts              # 插件元数据定义（必需）
│   ├── index.ts                   # 插件入口导出
│   ├── package.json               # 包配置
│   ├── zod.ts                     # Zod 验证模式
│   ├── lib/
│   │   ├── index.ts               # 库导出
│   │   └── VideoApiAdapter.ts     # 视频适配器实现（视频类插件）
│   ├── api/
│   │   ├── index.ts               # API 路由聚合
│   │   ├── add.ts                 # 添加/连接插件端点
│   │   └── callback.ts            # OAuth 回调端点
│   └── static/
│       ├── icon.svg               # 插件图标
│       └── zoom1.jpg              # 截图等静态资源
├── _appRegistry.ts                # 应用注册核心逻辑
├── appStoreMetaData.ts            # 元数据加载与规范化
├── video.adapters.generated.ts    # 自动生成的视频适配器映射
└── getVideoAdapters.ts            # 适配器获取工厂函数
```

### 1.2 关键文件说明

| 文件 | 用途 | 必需 |
|------|------|------|
| `_metadata.ts` | 定义插件的元数据（名称、类型、分类等） | 是 |
| `index.ts` | 插件入口，导出 api、lib、metadata 等 | 是 |
| `lib/VideoApiAdapter.ts` | 视频类插件核心适配器，实现会议创建/更新/删除 | 视频类必需 |
| `api/add.ts` | 处理用户连接插件的请求（OAuth 授权起始） | 是 |
| `api/callback.ts` | OAuth 授权回调处理 | OAuth 类必需 |
| `zod.ts` | 定义 API 响应和配置的验证模式 | 推荐 |

---

## 2. 插件注册机制

### 2.1 元数据定义 (`_metadata.ts`)

每个插件必须在 `_metadata.ts` 中定义其元数据，这是插件被系统识别的基础。

**Zoom 插件元数据示例** (`packages/app-store/zoomvideo/_metadata.ts:3-28`):

```typescript
import type { AppMeta } from "@calcom/types/App";

export const metadata = {
  linkType: "dynamic",                    // 链接类型：dynamic（动态生成）或 static（静态链接）
  name: "Zoom Video",                      // 插件显示名称
  description: "Zoom is a secure...",     // 插件描述
  type: "zoom_video",                      // 插件类型标识符
  categories: ["conferencing"],            // 分类：conferencing, calendar, payment, crm 等
  variant: "conferencing",                 // 变体类型
  logo: "icon.svg",                         // 图标文件名
  publisher: "Cal.diy",                    // 发布者
  url: "https://zoom.us/",                 // 官方网站
  category: "conferencing",                // 单一分类（已废弃，用 categories）
  slug: "zoom",                             // 插件 slug（用于路由和标识）
  title: "Zoom Video",                      // 标题（已废弃，用 name）
  email: "help@cal.com",                   // 联系邮箱
  appData: {
    location: {
      default: false,
      linkType: "dynamic",
      type: "integrations:zoom",           // 位置类型标识，用于预约时识别
      label: "Zoom Video",
    },
  },
  dirName: "zoomvideo",                     // 目录名
  isOAuth: true,                            // 是否使用 OAuth 授权
} as AppMeta;
```

### 2.2 元数据类型定义 (`AppMeta`)

**核心类型定义** (`packages/types/App.d.ts:52-171`):

```typescript
export interface App {
  type: `${string}_calendar` | `${string}_messaging` | 
        `${string}_payment` | `${string}_video` |
        `${string}_other` | `${string}_automation` |
        `${string}_analytics` | `${string}_crm` |
        `${string}_other_calendar`;
        
  variant: "calendar" | "payment" | "conferencing" | "video" | 
           "other" | "other_calendar" | "automation" | "crm";
           
  categories: AppCategories[];  // AppCategories 枚举：conferencing, calendar, payment, crm 等
  slug: string;                  // 唯一标识
  name: string;                  // 显示名称
  logo: string;                  // 图标路径
  
  appData?: {
    location?: EventLocationTypeFromAppMeta;  // 位置相关配置（视频类）
    tag?: Tag;                                  // 脚本标签（分析类）
  };
  
  isOAuth?: boolean;            // 是否 OAuth 应用
  extendsFeature?: "EventType" | "User";  // 扩展层级
  // ... 更多字段
}
```

### 2.3 自动生成的元数据与适配器映射

系统通过 `app-store-cli` 自动扫描所有插件目录，生成两个关键文件：

#### 2.3.1 `apps.metadata.generated.ts`

聚合所有插件的 `_metadata.ts` 导出，形成全局元数据注册表。

#### 2.3.2 `video.adapters.generated.ts` (`packages/app-store/video.adapters.generated.ts:1-21`)

**视频适配器动态导入映射**：

```typescript
export const VideoApiAdapterMap =
  process.env.NEXT_PUBLIC_IS_E2E === "1"
    ? {}
    : {
        dailyvideo: import("./dailyvideo/lib/VideoApiAdapter"),
        zoomvideo: import("./zoomvideo/lib/VideoApiAdapter"),
        // ... 其他视频插件
      };
```

**关键机制**：
- 使用动态 `import()` 实现按需加载
- 键名由插件目录名派生（`zoom_video` → `zoomvideo`）
- E2E 测试环境下返回空对象以避免外部依赖

### 2.4 元数据规范化 (`getNormalizedAppMetadata`)

**规范化处理** (`packages/app-store/getNormalizedAppMetadata.ts:12-28`):

```typescript
export const getNormalizedAppMetadata = (appMeta: ...) => {
  const dirName = "dirName" in appMeta ? appMeta.dirName : appMeta.slug;
  const metadata = {
    appData: null,
    dirName,
    __template: "",
    ...appMeta,
  } as ...;
  metadata.logo = getAppAssetFullPath(metadata.logo, {
    dirName,
    isTemplate: metadata.isTemplate,
  });
  return metadata;
};
```

**规范化内容**：
1. 补全 `dirName`（从 slug 或显式定义）
2. 设置默认 `appData: null`
3. 转换图标路径为完整 URL

### 2.5 应用注册与获取

**获取应用元数据** (`packages/app-store/_appRegistry.ts:19-37`):

```typescript
export async function getAppWithMetadata(app: { dirName: string } | { slug: string }) {
  let appMetadata: App | null;

  if ("dirName" in app) {
    appMetadata = appStoreMetadata[app.dirName as keyof typeof appStoreMetadata] as App;
  } else {
    const foundEntry = Object.entries(appStoreMetadata).find(([, meta]) => {
      return meta.slug === app.slug;
    });
    if (!foundEntry) return null;
    appMetadata = foundEntry[1] as App;
  }

  if (!appMetadata) return null;
  // 移除敏感字段（如 API keys）
  const { key, ...metadata } = appMetadata;
  return metadata;
}
```

---

## 3. 核心预约流程与插件触发路径

### 3.1 整体架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              预约请求入口                                      │
│  (Booking Handler / tRPC Router / API Endpoint)                              │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     RegularBookingService.handler()                          │
│  - 验证预约参数                                                               │
│  - 检查可用性                                                                 │
│  - 构建 CalendarEvent                                                         │
│  - 创建/更新 Booking 记录                                                     │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EventManager.create()                                │
│  - 分析 location 字段（如 "integrations:zoom"）                              │
│  - 分派到对应集成类型                                                          │
│  - 协调日历、视频、CRM 集成                                                    │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
            ┌─────────────────────┼─────────────────────┐
            ▼                     ▼                     ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  日历集成         │   │  视频集成         │   │  CRM 集成         │
│  CalendarManager │   │  videoClient     │   │  CrmManager      │
└──────────────────┘   └─────────┬────────┘   └──────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      getVideoAdapters(credentials)                           │
│  - 根据 credential.type 查找对应适配器                                        │
│  - 动态加载 VideoApiAdapter                                                    │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                   ZoomVideoApiAdapter.createMeeting()                         │
│  - 调用 Zoom API 创建会议                                                      │
│  - 返回 VideoCallData（id, url, password 等）                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 预约服务入口 (`RegularBookingService`)

**核心预约处理** (`packages/features/bookings/lib/service/RegularBookingService.ts`):

#### 3.2.1 构建 CalendarEvent

```typescript
// 约 1363-1450 行
let evt: BuiltCalendarEvent = new CalendarEventBuilder({
  bookerUrl,
  title: eventName,
  startTime: dayjs(reqBody.start).utc().format(),
  endTime: dayjs(reqBody.end).utc().format(),
  // ...
})
  .withLocation({
    location: platformBookingLocation ?? bookingLocation,  // 关键：位置信息
    conferenceCredentialId,
  })
  .withDestinationCalendar(...)
  .withIdentifiers({ iCalUID, iCalSequence })
  .build();
```

#### 3.2.2 初始化 EventManager

```typescript
// 约 1832-1836 行
const credentials = await refreshCredentials(allCredentials);
const apps = eventTypeAppMetadataOptionalSchema.parse(eventType?.metadata?.apps);
const eventManager =
  !isDryRun && !skipCalendarSyncTaskCreation
    ? new EventManager({ ...organizerUser, credentials }, apps)
    : buildDryRunEventManager();
```

#### 3.2.3 触发事件创建

```typescript
// 约 2050-2073 行
} else if (isConfirmedByDefault) {
  const shouldSkipCalendarEvents = !areCalendarEventsEnabled || skipCalendarSyncTaskCreation;
  const createManager = await eventManager.create(evt, { 
    skipCalendarEvent: shouldSkipCalendarEvents 
  });
  if (evt.location) {
    booking.location = evt.location;
  }
  // ...
  results = createManager.results;
  referencesToCreate = createManager.referencesToCreate;
  videoCallUrl = evt.videoCallData?.url ? evt.videoCallData.url : null;
}
```

### 3.3 EventManager - 集成协调中心

**EventManager 职责** (`packages/features/bookings/lib/EventManager.ts:133-173`):

```typescript
export default class EventManager {
  calendarCredentials: CredentialForCalendarService[];
  videoCredentials: CredentialForCalendarService[];
  crmCredentials: CredentialForCalendarService[];
  appOptions?: Record<string, any>;

  constructor(user: EventManagerUser, eventTypeAppMetadata?: Record<string, any>) {
    const appCredentials = getApps(user.credentials, true).flatMap((app) =>
      app.credentials.map((creds) => ({ ...creds, appName: app.name }))
    );
    
    // 按类型分类凭据
    this.calendarCredentials = appCredentials
      .filter((cred) => cred.type.endsWith("_calendar") && !cred.type.includes("other_calendar"))
      .sort(latestCredentialFirst)
      .sort(delegatedCredentialFirst);

    this.videoCredentials = appCredentials
      .filter((cred) => cred.type.endsWith("_video") || cred.type.endsWith("_conferencing"))
      .sort(latestCredentialFirst);

    this.crmCredentials = appCredentials.filter(
      (cred) => cred.type.endsWith("_crm") || cred.type.endsWith("_other_calendar")
    );
    // ...
  }
}
```

**核心方法**：

| 方法 | 用途 | 触发场景 |
|------|------|----------|
| `create(event)` | 创建所有集成事件（日历、视频、CRM） | 新预约 |
| `reschedule(event, uid)` | 重新调度事件 | 改期 |
| `cancelEvent(event, references)` | 取消事件 | 取消预约 |
| `deleteEventsAndMeetings(...)` | 删除所有关联事件 | 清理 |

### 3.4 事件创建流程 (`EventManager.create`)

**位置检测与分派** (`packages/features/bookings/lib/EventManager.ts:287-416`):

```typescript
public async create(
  event: CalendarEvent,
  options?: { skipCalendarEvent?: boolean }
): Promise<CreateUpdateResult> {
  const { skipCalendarEvent = false } = options ?? {};
  const evt = processLocation(event);  // 处理位置信息

  // 回退到默认视频（如 Daily.co）
  if (!evt.location) {
    const calVideo = await prisma.app.findUnique({
      where: { slug: "daily-video" },
      select: { keys: true, enabled: true },
    });
    // ...
    if (calVideo?.enabled && calVideoKeys.success) 
      evt["location"] = "integrations:daily";
  }

  // 检测是否为专用视频集成位置
  const isDedicated = evt.location ? isDedicatedIntegration(evt.location) : null;

  const results: Array<EventResult<...>> = [];

  // 关键：如果是专用视频集成（如 integrations:zoom），创建视频会议
  if (isDedicated && !isMSTeamsWithOutlookCalendar) {
    const result = await this.createVideoEvent(evt);  // ← 触发视频插件

    if (result?.createdEvent) {
      evt.videoCallData = result.createdEvent;
      evt.location = result.originalEvent.location;
      result.type = result.createdEvent.type;
    }
    results.push(result);
  }

  // 创建日历事件（包含 videoCallData）
  if (!skipCalendarEvent) {
    results.push(...(await this.createAllCalendarEvents(clonedCalEvent)));
  }

  // 创建 CRM 事件
  const createdCRMEvents = skipCalendarEvent ? [] : await this.createAllCRMEvents(evt);
  results.push(...createdCRMEvents);

  // 构建 referencesToCreate（保存到数据库）
  const referencesToCreate = results.map((result) => {
    return {
      type: result.type,
      uid: createdEventObj ? createdEventObj.id : (result.createdEvent?.id?.toString() ?? ""),
      meetingId: createdEventObj ? createdEventObj.id : result.createdEvent?.id?.toString(),
      meetingPassword: createdEventObj ? createdEventObj.password : result.createdEvent?.password,
      meetingUrl: createdEventObj ? createdEventObj.onlineMeetingUrl : result.createdEvent?.url,
      // ...
    };
  });

  return { results, referencesToCreate };
}
```

### 3.5 视频事件创建 (`createVideoEvent`)

**获取对应凭据** (`packages/features/bookings/lib/EventManager.ts:1079-1088`):

```typescript
private getVideoCredentialByCalendarEvent(event: CalendarEvent): CredentialForCalendarService | undefined {
  if (!event.location) {
    return undefined;
  }

  // 从 location 提取集成名称："integrations:zoom" → "zoom"
  const integrationName = event.location.replace("integrations:", "");
  
  let videoCredential;
  if (event.conferenceCredentialId) {
    // 使用指定的凭据 ID
    videoCredential = this.videoCredentials.find(
      (credential) => credential.id === event.conferenceCredentialId
    );
  } else {
    // 根据类型查找：credential.type 包含 "zoom"
    videoCredential = this.videoCredentials.find((credential) =>
      credential.type.includes(integrationName)
    );
  }

  // 回退到 Daily.co
  if (!videoCredential) {
    videoCredential = { ...FAKE_DAILY_CREDENTIAL };
  }

  return videoCredential;
}
```

**创建视频会议** (`packages/features/bookings/lib/EventManager.ts:1079-1088`):

```typescript
private async createVideoEvent(event: CalendarEvent) {
  const credential = this.getVideoCredentialByCalendarEvent(event);
  if (credential) {
    return createMeeting(credential, event);  // ← 调用 videoClient
  } else {
    return Promise.reject(
      `No suitable credentials given for the requested integration name:${event.location}`
    );
  }
}
```

### 3.6 视频客户端 (`videoClient`)

**核心会议操作** (`packages/features/conferencing/lib/videoClient.ts:29-103`):

```typescript
const createMeeting = async (
  credential: CredentialPayload | CredentialForCalendarService,
  calEvent: CalendarEvent
) => {
  const uid: string = getUid(calEvent.uid);
  
  // 检查应用是否启用
  const enabledApp = await prisma.app.findUnique({
    where: { slug: credential.appId },
    select: { enabled: true },
  });

  if (!enabledApp?.enabled)
    throw `Location app ${credential.appId} is either disabled or not seeded at all`;

  // 关键：获取视频适配器
  const videoAdapters = await getVideoAdapters([credential]);
  const [firstVideoAdapter] = videoAdapters;
  
  let createdMeeting;
  let returnObject = {
    appName: credential.appName || credential.appId || "",
    type: credential.type,
    uid,
    originalEvent: calEvent,
    success: false,
    createdEvent: undefined,
    credentialId: credential.id,
  };

  try {
    // 调用适配器的 createMeeting 方法
    createdMeeting = await firstVideoAdapter?.createMeeting(calEvent);
    
    returnObject = { ...returnObject, createdEvent: createdMeeting, success: true };
  } catch (err) {
    // 错误处理：发送邮件通知，回退到 Cal Video
    await sendBrokenIntegrationEmail(calEvent, "video");
    const defaultMeeting = await createMeetingWithCalVideo(calEvent);
    if (defaultMeeting) {
      calEvent.location = DailyLocationType;
    }
    returnObject = { ...returnObject, originalEvent: calEvent, createdEvent: defaultMeeting };
  }

  return returnObject;
};
```

### 3.7 适配器获取工厂 (`getVideoAdapters`)

**动态加载适配器** (`packages/app-store/getVideoAdapters.ts:11-45`):

```typescript
export const getVideoAdapters = async (
  withCredentials: CredentialPayload[]
): Promise<VideoApiAdapter[]> => {
  const videoAdapters: VideoApiAdapter[] = [];

  for (const cred of withCredentials) {
    // 转换类型名："zoom_video" → "zoomvideo"
    const appName = cred.type.split("_").join("");
    
    let videoAdapterImport = VideoApiAdapterMap[appName as keyof typeof VideoApiAdapterMap];

    // 回退策略："zoom_video" → "zoom"
    if (!videoAdapterImport) {
      const appTypeVariant = cred.type.substring(0, cred.type.lastIndexOf("_"));
      videoAdapterImport = VideoApiAdapterMap[appTypeVariant as keyof typeof VideoApiAdapterMap];
    }

    if (!videoAdapterImport) {
      log.error(`Couldn't get adapter for ${appName}`);
      continue;
    }

    // 动态加载模块
    const videoAdapterModule = await videoAdapterImport;
    const makeVideoApiAdapter = videoAdapterModule.default as VideoApiAdapterFactory;

    if (makeVideoApiAdapter) {
      // 实例化适配器（传入凭据）
      const videoAdapter = makeVideoApiAdapter(cred);
      videoAdapters.push(videoAdapter);
    }
  }

  return videoAdapters;
};
```

**类型转换规则**：

| `credential.type` | 第一次转换 | 回退转换 | 适配器映射键 |
|-------------------|------------|----------|--------------|
| `zoom_video` | `zoomvideo` | `zoom` | `zoomvideo` |
| `google_video` | `googlevideo` | `google` | `googlevideo` |
| `jitsi_video` | `jitsivideo` | `jitsi` | `jitsivideo` |

---

## 4. 插件适配器实现深度分析

### 4.1 VideoApiAdapter 接口定义

**接口规范** (`packages/types/VideoApiAdapter.d.ts:12-50`):

```typescript
export interface VideoCallData {
  type: string;      // 如 "zoom_video"
  id: string;        // 会议 ID
  password: string;  // 会议密码
  url: string;       // 加入链接
}

export type VideoApiAdapter =
  | {
      createMeeting(event: CalendarEvent): Promise<VideoCallData>;
      updateMeeting(bookingRef: PartialReference, event: CalendarEvent): Promise<VideoCallData>;
      deleteMeeting(uid: string): Promise<unknown>;
      getAvailability(dateFrom?: string, dateTo?: string): Promise<EventBusyDate[]>;
      
      // 可选方法
      getRecordings?(roomName: string): Promise<GetRecordingsResponseSchema>;
      createInstantCalVideoRoom?(endTime: string): Promise<VideoCallData>;
      // ...
    }
  | undefined;

// 适配器工厂函数签名
export type VideoApiAdapterFactory = (credential: CredentialPayload) => VideoApiAdapter;
```

### 4.2 Zoom 适配器实现详解

**适配器工厂** (`packages/app-store/zoomvideo/lib/VideoApiAdapter.ts:235-577`):

```typescript
const ZoomVideoApiAdapter = (credential: CredentialPayload): VideoApiAdapter => {
  const tokenResponse = getTokenObjectFromCredential(credential);

  // OAuth 管理器：处理 token 刷新、过期检测
  const fetchZoomApi = async (endpoint: string, options?: RequestInit) => {
    const auth = new OAuthManager({
      credentialSyncVariables: {
        APP_CREDENTIAL_SHARING_ENABLED,
        CREDENTIAL_SYNC_ENDPOINT,
        CREDENTIAL_SYNC_SECRET,
        CREDENTIAL_SYNC_SECRET_HEADER_NAME,
      },
      resourceOwner: {
        type: "user",
        id: credential.userId,
      },
      appSlug: metadata.slug,  // "zoom"
      currentTokenObject: tokenResponse,
      
      // 刷新 token 回调
      fetchNewTokenObject: async ({ refreshToken }) => {
        if (!refreshToken) return null;
        const clientCredentials = await getZoomAppKeys();
        const { client_id, client_secret } = clientCredentials;
        const authHeader = `Basic ${Buffer.from(`${client_id}:${client_secret}`).toString("base64")}`;
        return fetch("https://zoom.us/oauth/token", {
          method: "POST",
          headers: {
            Authorization: authHeader,
            "Content-Type": "application/x-www-form-urlencoded",
          },
          body: new URLSearchParams({
            refresh_token: refreshToken,
            grant_type: "refresh_token",
          }),
        });
      },
      
      // Token 有效性检测
      isTokenObjectUnusable: async function (response) {
        if (!response.ok) {
          let responseBody = await response.json();
          // Zoom 的 invalid_grant 错误表示 refresh token 失效
          if (responseBody.error === "invalid_grant") {
            return { reason: responseBody.error };
          }
        }
        return null;
      },
      
      // 访问 token 失效检测
      isAccessTokenUnusable: async function (response) {
        if (!response.ok) {
          let responseBody = await response.json();
          // Zoom 错误码 124 表示 access token 无效
          if (responseBody.code === 124) {
            return { reason: responseBody.message ?? "" };
          }
        }
        return null;
      },
      
      // 更新 token 到数据库
      updateTokenObject: async (newTokenObject) => {
        await prisma.credential.update({
          where: { id: credential.id },
          data: {
            key: newTokenObject as unknown as Prisma.InputJsonValue,
          },
        });
      },
    });

    const { json } = await auth.request({
      url: `https://api.zoom.us/v2/${endpoint}`,
      options: { method: "GET", ...options },
    });
    return json;
  };

  // 返回适配器对象
  return {
    getAvailability: async () => {
      const responseBody = await fetchZoomApi("users/me/meetings?type=scheduled&page_size=300");
      const data = zoomMeetingsSchema.parse(responseBody);
      return data.meetings.map((meeting) => ({
        start: meeting.start_time,
        end: new Date(new Date(meeting.start_time).getTime() + meeting.duration * 60000).toISOString(),
      }));
    },

    createMeeting: async (event: CalendarEvent): Promise<VideoCallData> => {
      try {
        const response = await fetchZoomApi("users/me/meetings", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify(await translateEvent(event)),
        });

        const result = zoomEventResultSchema.parse(response);

        if (result.id && result.join_url) {
          return {
            type: "zoom_video",
            id: result.id.toString(),
            password: result.password || "",
            url: result.join_url,
          };
        }
        throw new Error(`Failed to create meeting. Response is ${JSON.stringify(result)}`);
      } catch (err) {
        log.error("Zoom meeting creation failed", ...);
        throw new Error("Unexpected error");
      }
    },

    deleteMeeting: async (uid: string): Promise<void> => {
      await fetchZoomApi(`meetings/${uid}`, { method: "DELETE" });
      return Promise.resolve();
    },

    updateMeeting: async (bookingRef: PartialReference, event: CalendarEvent): Promise<VideoCallData> => {
      await fetchZoomApi(`meetings/${bookingRef.uid}`, {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(await translateEvent(event)),
      });

      const updatedMeeting = await fetchZoomApi(`meetings/${bookingRef.uid}`);
      const result = zoomEventResultSchema.parse(updatedMeeting);

      return {
        type: "zoom_video",
        id: result.id.toString(),
        password: result.password || "",
        url: result.join_url,
      };
    },
  };
};

export default ZoomVideoApiAdapter;
```

### 4.3 事件转换 (`translateEvent`)

**CalendarEvent → Zoom API 转换** (`packages/app-store/zoomvideo/lib/VideoApiAdapter.ts:251-349`):

```typescript
const translateEvent = async (event: CalendarEvent) => {
  const getRecurrence = ({ recurringEvent, startTime, attendees }: CalendarEvent) => {
    if (!recurringEvent) return;

    let recurrence: ZoomRecurrence;
    switch (recurringEvent.freq) {
      case Frequency.DAILY:
        recurrence = { type: 1 };
        break;
      case Frequency.WEEKLY:
        recurrence = {
          type: 2,
          weekly_days: dayjs(startTime).tz(attendees[0].timeZone).day() + 1,
        };
        break;
      case Frequency.MONTHLY:
        recurrence = {
          type: 3,
          monthly_day: dayjs(startTime).tz(attendees[0].timeZone).date(),
        };
        break;
      default:
        return;  // Zoom 不支持 YEARLY, HOURLY, MINUTELY
    }
    // ...
    return { recurrence };
  };

  const userSettings = await getUserSettings();
  const recurrence = getRecurrence(event);
  const waitingRoomEnabled = userSettings?.in_meeting?.waiting_room ?? false;
  const password = getCompliantPassword(defaultPassword, passwordRequirements);

  return {
    topic: event.title,
    type: 2,  // 2 = 预定会议
    start_time: dayjs(event.startTime).tz(event.organizer.timeZone).format("YYYY-MM-DDTHH:mm:ss"),
    duration: (new Date(event.endTime).getTime() - new Date(event.startTime).getTime()) / 60000,
    timezone: event.organizer.timeZone,
    agenda: truncateAgenda(event.description),
    ...(password && { password }),
    settings: {
      host_video: true,
      participant_video: true,
      join_before_host: !waitingRoomEnabled,
      mute_upon_entry: false,
      watermark: false,
      use_pmi: false,
      approval_type: 2,
      audio: "both",
      auto_recording: userSettings?.recording?.auto_recording || "none",
      enforce_login: false,
      registrants_email_notification: true,
      waiting_room: waitingRoomEnabled,
    },
    ...recurrence,
  };
};
```

---

## 5. 数据持久化与引用追踪

### 5.1 BookingReference 模型

当插件操作成功后，结果通过 `referencesToCreate` 保存到数据库的 `BookingReference` 表。

**引用数据结构** (`packages/features/bookings/lib/EventManager.ts:388-410`):

```typescript
const referencesToCreate = results.map((result) => {
  let createdEventObj: createdEventSchema | null = null;
  if (typeof result?.createdEvent === "string") {
    createdEventObj = createdEventSchema.parse(JSON.parse(result.createdEvent));
  }
  
  return {
    type: result.type,                              // 如 "zoom_video", "google_calendar"
    uid: createdEventObj ? createdEventObj.id : (result.createdEvent?.id?.toString() ?? ""),
    thirdPartyRecurringEventId: isCalendarType ? thirdPartyRecurringEventId : undefined,
    meetingId: createdEventObj ? createdEventObj.id : result.createdEvent?.id?.toString(),
    meetingPassword: createdEventObj ? createdEventObj.password : result.createdEvent?.password,
    meetingUrl: createdEventObj ? createdEventObj.onlineMeetingUrl : result.createdEvent?.url,
    externalCalendarId: isCalendarType ? result.externalId : undefined,
    credentialId: result.credentialId,
  };
});
```

**保存到数据库** (`packages/features/bookings/lib/service/RegularBookingService.ts:2444-2459`):

```typescript
try {
  if (!isDryRun) {
    await deps.prismaClient.booking.update({
      where: { uid: booking.uid },
      data: {
        location: evt.location,
        metadata: { ...(typeof booking.metadata === "object" && booking.metadata), ...metadata },
        references: {
          createMany: {
            data: referencesToCreate,  // ← 批量创建引用记录
          },
        },
      },
    });
  }
} catch (error) {
  tracingLogger.error("Error while creating booking references", JSON.stringify({ error }));
}
```

### 5.2 引用类型分类

| `type` 前缀 | 集成类型 | 示例 |
|-------------|----------|------|
| `*_video` | 视频会议 | `zoom_video`, `google_video`, `daily_video` |
| `*_calendar` | 日历同步 | `google_calendar`, `office365_calendar` |
| `*_crm` | CRM 集成 | `hubspot_crm`, `salesforce_crm` |
| `*_payment` | 支付集成 | `stripe_payment`, `paypal_payment` |

---

---

## 附录 A：凭据授权落库与预约选择完整链路

### A.1 概述：完整链路全景

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    完整链路：授权 → 落库 → 关联 → 预约 → 选择                      │
└───────────────────────────────────────────────────────────────────────────────────────────────┘

  [阶段 1: OAuth 授权与凭据落库]
  ┌─────────────┐     ┌─────────────┐     ┌─────────────────┐     ┌────────────────────────┐
  │  用户点击   │────>│  重定向到  │────>│  OAuth 回调     │────>│  创建 Credential 记录  │
  │ "添加 Zoom" │     │  Zoom 授权  │     │  换取 token     │     │  (type: zoom_video)    │
  └─────────────┘     └─────────────┘     └─────────────────┘     └─────────────┬──────────┘
                                                                                        │
  [阶段 2: 凭据与事件类型关联]                                                         │
  ┌─────────────┐     ┌──────────────────────────┐     ┌─────────────────────────┐ │
  │  用户设置   │────>│  更新 EventType.locations │────>│  记录 { type:           │ │
  │ 默认会议应用 │     │  保存 credentialId        │     │    "integrations:zoom", │ │
  │             │     │                         │     │    credentialId: 123 }   │ │
  └─────────────┘     └──────────────────────────┘     └─────────────────────────┘ │
                                                                                        │
  [阶段 3: 预约时凭据选择]                                                             │
  ┌─────────────┐     ┌──────────────────────────┐     ┌─────────────────────────┐ │
  │  发起预约   │────>│  解析 location 和        │────>│  三级分支选择凭据        │ │
  │ 请求        │     │  conferenceCredentialId  │     │  (ID匹配 → 类型匹配 → 兜底)│ │
  └─────────────┘     └──────────────────────────┘     └─────────────┬───────────┘ │
                                                                         │             │
                                                                         ▼             │
  [阶段 4: 凭据使用与追踪]                                                           │
  ┌─────────────────┐     ┌──────────────────────────┐     ┌──────────────────────┐ │
  │  加载适配器     │<────│  调用 VideoApiAdapter    │<────│  使用选中的凭据      │ │
  │  创建 Zoom 会议 │     │  保存 BookingReference   │     │                      │ │
  │  返回会议链接   │     │  (含 credentialId)       │     │                      │ │
  └─────────────────┘     └──────────────────────────┘     └──────────────────────┘ │
                                                                                        │
                                                                                        │
  ┌───────────────────────────────────────────────────────────────────────────────────┘
  │ 关键数据字段
  ├─ Credential.type: "zoom_video" (数据库凭据类型标识)
  ├─ EventType.locations: [{ type: "integrations:zoom", credentialId: 123 }]
  ├─ CalendarEvent.location: "integrations:zoom"
  ├─ CalendarEvent.conferenceCredentialId: 123 (可选，优先级最高)
  └─ BookingReference.credentialId: 123 (追踪哪个凭据被使用)
```

---

### A.2 阶段 1：OAuth 授权与凭据落库

#### A.2.1 授权流程全景

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│    用户      │         │   Cal.diy    │         │    Zoom      │         │   Database   │
│ (浏览器)     │         │   (后端)     │         │   (OAuth)    │         │  (Credential)│
└──────┬───────┘         └──────┬───────┘         └──────┬───────┘         └──────┬───────┘
       │                         │                         │                         │
       │  点击 "添加 Zoom"       │                         │                         │
       │────────────────────────>│                         │                         │
       │                         │                         │                         │
       │                         │  GET /api/integrations/zoomvideo/add             │
       │                         │─────────────────────────│                         │
       │                         │                         │                         │
       │                         │  302 重定向到 Zoom 授权页                         │
       │<────────────────────────│                         │                         │
       │                         │                         │                         │
       │  跳转至 Zoom 登录/授权页面                        │                         │
       │─────────────────────────>                         │                         │
       │                         │                         │                         │
       │                         │                         │  用户授权同意          │
       │                         │                         │                         │
       │                         │  302 回调到 callback    │                         │
       │<────────────────────────|─────────────────────────|                         │
       │                         │                         │                         │
       │                         │  POST /oauth/token      │                         │
       │                         │  (code → access_token)  │                         │
       │                         │─────────────────────────>│                         │
       │                         │                         │                         │
       │                         │<─────────────────────────│                         │
       │                         │  { access_token,        │                         │
       │                         │    refresh_token,       │                         │
       │                         │    expires_in }          │                         │
       │                         │                         │                         │
       │                         │  DELETE old credentials  │                         │
       │                         │  (防重复)                │                         │
       │                         │──────────────────────────────────────────────────>│
       │                         │                         │                         │
       │                         │  CREATE new credential  │                         │
       │                         │  (type: zoom_video)     │                         │
       │                         │  (key: token数据)       │                         │
       │                         │──────────────────────────────────────────────────>│
       │                         │                         │                         │
       │                         │  302 重定向到已安装应用页                          │
       │<────────────────────────│                         │                         │
```

#### A.2.2 授权入口：`api/add.ts`

**代码位置**: `packages/app-store/zoomvideo/api/add.ts:12-39`

```typescript
async function handler(req: NextApiRequest) {
  // 验证用户登录
  await prisma.user.findFirstOrThrow({
    where: { id: req.session?.user?.id },
    select: { id: true },
  });

  // 获取 Zoom 应用配置（client_id）
  const { client_id } = await getZoomAppKeys();
  
  // 编码 OAuth state（用于防止 CSRF，可携带 teamId 等参数）
  const state = encodeOAuthState(req);

  // 构建 OAuth 授权 URL
  const params = {
    response_type: "code",
    client_id,
    redirect_uri: `${WEBAPP_URL_FOR_OAUTH}/api/integrations/zoomvideo/callback`,
    state,
  };
  const query = stringify(params);
  
  // 重定向到 Zoom 授权页面
  const url = `https://zoom.us/oauth/authorize?${query}`;
  return { url };
}
```

**关键设计**：
- `encodeOAuthState`: 编码状态参数，包含 `teamId`（用于团队凭据）、`returnTo`（授权后跳转地址）
- `redirect_uri`: 必须与 Zoom 应用配置的回调 URL 完全匹配

#### A.2.3 OAuth 回调与凭据创建：`api/callback.ts`

**代码位置**: `packages/app-store/zoomvideo/api/callback.ts:12-82`

```typescript
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const state = decodeOAuthState(req);
  const { code } = req.query;
  const { client_id, client_secret } = await getZoomAppKeys();

  // ========== 步骤 1: 用 code 换取 token ==========
  const redirectUri = encodeURI(`${WEBAPP_URL_FOR_OAUTH}/api/integrations/zoomvideo/callback`);
  const authHeader = `Basic ${Buffer.from(`${client_id}:${client_secret}`).toString("base64")}`;
  
  const result = await fetch(
    `https://zoom.us/oauth/token?grant_type=authorization_code&code=${code}&redirect_uri=${redirectUri}`,
    {
      method: "POST",
      headers: { Authorization: authHeader },
    }
  );

  if (result.status !== 200) {
    res.status(400).json({ message: "Zoom API error" });
    return;
  }

  const responseBody = await result.json();
  if (responseBody.error) {
    res.status(400).json({ message: responseBody.error });
    return;
  }

  // ========== 步骤 2: 处理 token 格式 ==========
  // 计算过期时间（转换为绝对时间戳）
  responseBody.expiry_date = Math.round(Date.now() + responseBody.expires_in * 1000);
  delete responseBody.expires_in;  // 移除相对时间，保存绝对时间

  // ========== 步骤 3: 清理旧凭据（防止重复） ==========
  const userId = req.session?.user.id;
  
  /**
   * 设计意图：
   * - 同一用户同一类型的凭据只保留一份
   * - 避免预约时不知道用哪个凭据的问题
   * - 重新授权时自动替换旧凭据
   */
  const existingCredentialZoomVideo = await prisma.credential.findMany({
    select: { id: true },
    where: {
      type: "zoom_video",      // 按类型查找
      userId: req.session?.user.id,
      appId: "zoom",
    },
  });

  // 删除旧凭据
  const credentialIdsToDelete = existingCredentialZoomVideo.map((item) => item.id);
  if (credentialIdsToDelete.length > 0) {
    await prisma.credential.deleteMany({ 
      where: { id: { in: credentialIdsToDelete }, userId } 
    });
  }

  // ========== 步骤 4: 创建新凭据 ==========
  await createOAuthAppCredential(
    { appId: "zoom", type: "zoom_video" }, 
    responseBody,  // { access_token, refresh_token, expiry_date }
    req
  );

  // ========== 步骤 5: 重定向回应用页面 ==========
  res.redirect(
    getSafeRedirectUrl(state?.returnTo) ?? 
    getInstalledAppPath({ variant: "conferencing", slug: "zoom" })
  );
}
```

**Token 存储格式** (`Credential.key`):
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUz...",
  "refresh_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUz...",
  "expiry_date": 1746403200000,
  "scope": "meeting:write,user:read"
}
```

#### A.2.4 凭据落库核心：`createOAuthAppCredential`

**代码位置**: `packages/app-store/_utils/oauth/createOAuthAppCredential.ts:18-52`

```typescript
const createOAuthAppCredential = async (
  appData: { type: string; appId: string },
  key: unknown,
  req: NextApiRequest
) => {
  const userId = req.session?.user.id;
  if (!userId) {
    throw new HttpError({ statusCode: 401, message: "You must be logged in" });
  }

  // 从 OAuth state 解析团队 ID（如果是团队安装）
  const state = decodeOAuthState(req);

  // ========== 分支：团队凭据 vs 用户凭据 ==========
  if (state?.teamId) {
    // 验证用户是否是该团队管理员
    await throwIfNotHaveAdminAccessToTeam({ 
      teamId: state?.teamId ?? null, 
      userId 
    });

    // 创建团队凭据（没有 userId，有 teamId）
    return await prisma.credential.create({
      data: {
        type: appData.type,    // "zoom_video"
        key: key || {},         // token 数据
        teamId: state.teamId,   // 团队 ID
        appId: appData.appId,   // "zoom"
      },
    });
  }

  // 创建用户凭据（有 userId）
  return await prisma.credential.create({
    data: {
      type: appData.type,
      key: key || {},
      userId,
      appId: appData.appId,
    },
  });
};
```

#### A.2.5 Credential 表数据结构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Credential 表                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  字段              │  类型      │  示例值                                │
├─────────────────────────────────────────────────────────────────────────┤
│  id                │  Int       │  123                                   │
│  type              │  String    │  "zoom_video"                          │
│  key               │  Json      │  { access_token, refresh_token, ... } │
│  userId            │  Int?      │  1 (用户凭据) 或 null (团队凭据)       │
│  teamId            │  Int?      │  null (用户凭据) 或 5 (团队凭据)       │
│  appId             │  String?   │  "zoom"                                │
│  invalid           │  Boolean   │  false                                 │
└─────────────────────────────────────────────────────────────────────────┘

  关键关联关系：
  ├─ type: 用于按类型筛选凭据（如 `type.endsWith("_video")`）
  ├─ userId/teamId: 区分用户凭据 vs 团队凭据
  ├─ appId: 对应 App 表的 slug（用于检查应用是否启用）
  └─ key: 存储敏感的 token 数据（加密存储在实际部署中）
```

---

### A.3 阶段 2：凭据与事件类型关联

#### A.3.1 关联方式：两种路径

凭据创建后，需要与事件类型（EventType）关联才能在预约时使用。有两种关联方式：

```
方式 1：设置为默认会议应用（批量关联所有事件类型）
        用户操作 ──> setDefaultConferencingApp() ──> 更新所有 EventType.locations

方式 2：在事件类型设置中单独选择（单个事件类型）
        用户在编辑事件类型时 ──> 选择 Zoom ──> 更新单个 EventType.locations
```

#### A.3.2 设置默认会议应用：`setDefaultConferencingApp`

**代码位置**: `packages/app-store/_utils/setDefaultConferencingApp.ts:8-57`

```typescript
const setDefaultConferencingApp = async (userId: number, appSlug: string) => {
  // ========== 步骤 1: 获取用户所有事件类型 ==========
  const eventTypes = await getBulkUserEventTypes(userId);
  const eventTypeIds = eventTypes.eventTypes.map((item) => item.id);

  // ========== 步骤 2: 查找应用元数据，获取 location.type ==========
  const foundApp = getAppFromSlug(appSlug);
  const appType = foundApp?.appData?.location?.type;  // "integrations:zoom"

  if (!appType) return;

  // ========== 步骤 3: 查找用户的对应凭据 ==========
  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: { 
      metadata: true, 
      credentials: true  // 获取所有凭据
    },
  });

  // 找到对应的 credentialId（按 appSlug 匹配）
  const credentialId = user?.credentials.find((item) => item.appId == appSlug)?.id;

  // ========== 步骤 4: 更新用户默认设置（元数据） ==========
  const currentMetadata = userMetadata.parse(user?.metadata);
  await prisma.user.update({
    where: { id: userId },
    data: {
      metadata: {
        ...currentMetadata,
        defaultConferencingApp: {
          appSlug,  // "zoom"
        },
      },
    },
  });

  // ========== 步骤 5: 批量更新所有事件类型的 locations ==========
  await prisma.eventType.updateMany({
    where: {
      id: { in: eventTypeIds },
      userId,
    },
    data: {
      // 核心：将 location 和 credentialId 绑定
      locations: [
        { 
          type: appType,        // "integrations:zoom"
          credentialId           // 123（凭据 ID）
        }
      ] as LocationObject[],
    },
  });
};
```

#### A.3.3 EventType.locations 数据结构

**存储在 `EventType.locations` 字段（Json 类型）**:

```json
[
  {
    "type": "integrations:zoom",
    "credentialId": 123,
    "teamName": null,
    "displayLocationPublicly": false
  }
]
```

**关键关联字段**：
| 字段 | 用途 | 示例值 |
|------|------|--------|
| `type` | 标识集成类型，对应 `metadata.appData.location.type` | `"integrations:zoom"` |
| `credentialId` | 对应 `Credential.id`，预约时用于精确匹配凭据 | `123` |
| `teamName` | 团队凭据时显示团队名称 | `"Engineering Team"` |
| `displayLocationPublicly` | 是否在预约页面显示位置详情 | `true` |

#### A.3.4 前端位置选择组件

**代码位置**: `apps/web/modules/event-types/components/locations/Locations.tsx`

```typescript
// 用户选择 Zoom 作为位置时的处理
const handleLocationSelect = (e: TPrefillLocation) => {
  const newLocationType = e.value;  // "integrations:zoom"
  const canAppendLocation = !validLocations.find(
    (location) => location.type === newLocationType
  );

  if (canAppendLocation) {
    append({
      type: newLocationType,
      // 关键：保存 credentialId
      ...(e.credentialId && {
        credentialId: e.credentialId,
        teamName: e.teamName ?? undefined,
      }),
    });
  }
};
```

---

### A.4 阶段 3：预约时凭据选择（三级分支逻辑）

#### A.4.1 整体流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                        预约时凭据选择：三级分支决策树                                     │
└─────────────────────────────────────────────────────────────────────────────────────────┘

  预约请求
      │
      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  RegularBookingService.handler()                                                         │
│  ├─ 从 reqBody 获取 location（如 "integrations:zoom"）                                  │
│  ├─ 从 eventType.locations 解析 credentialId                                             │
│  └─ 构建 CalendarEvent { location, conferenceCredentialId }                             │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  EventManager.getVideoCredentialByCalendarEvent(event)                                   │
│                                                                                            │
│  输入参数：                                                                                │
│  ├─ event.location: "integrations:zoom"                                                  │
│  └─ event.conferenceCredentialId: 123 (可选)                                            │
│                                                                                            │
│  可用凭据池 (this.videoCredentials):                                                      │
│  ├─ [0] { id: 123, type: "zoom_video", userId: 1, ... }                                │
│  ├─ [1] { id: 456, type: "zoom_video", userId: 2, ... }  ← 团队成员的凭据              │
│  └─ [2] { id: 789, type: "google_video", userId: 1, ... }                              │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  ========== 分支 1：优先使用 conferenceCredentialId（精确 ID 匹配） ==========          │
│  if (event.conferenceCredentialId) {                                                    │
│      return this.videoCredentials.find(                                                 │
│          (credential) => credential.id === event.conferenceCredentialId                │
│      );                                                                                  │
│  }                                                                                        │
│                                                                                            │
│  匹配逻辑：credential.id === 123                                                          │
│  结果：找到 { id: 123, type: "zoom_video", ... }                                        │
│                                                                                            │
│  为什么优先？                                                                               │
│  ├─ 更可靠：ID 是唯一的，不会有歧义                                                       │
│  ├─ 支持团队：可以精确指定使用哪个团队成员的凭据                                           │
│  └─ 避免"类型包含"带来的问题（见下方潜在 bug 说明）                                      │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │ 未找到（credentialId 无效或凭据已删除）
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  ========== 分支 2：回退 - 按类型模糊匹配 ==========                                    │
│  else {                                                                                   │
│      const integrationName = event.location.replace("integrations:", "");  // "zoom"  │
│      return this.videoCredentials.find(                                                  │
│          (credential) => credential.type.includes(integrationName)                      │
│      );                                                                                   │
│  }                                                                                         │
│                                                                                             │
│  匹配逻辑：credential.type.includes("zoom")                                              │
│  检查：                                                                                     │
│  ├─ "zoom_video".includes("zoom")  →  ✓ true                                            │
│  ├─ "google_video".includes("zoom") →  ✗ false                                          │
│  └─ "zoom_teams".includes("zoom")   →  ⚠ true (潜在问题！)                              │
│                                                                                             │
│  ⚠️  潜在 Bug 风险：                                                                        │
│  如果有两个应用："zoom_video" 和 "zoom_teams"，模糊匹配可能选错凭据！                    │
└───────────────────────────────────────────┬─────────────────────────────────────────────┘
                                            │ 未找到（用户没有安装该集成）
                                            ▼
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  ========== 分支 3：兜底 - 使用 Daily.co 虚拟凭据 ==========                            │
│  if (!videoCredential) {                                                                 │
│      log.warn("Falling back to daily video integration");                               │
│      videoCredential = { ...FAKE_DAILY_CREDENTIAL };                                   │
│  }                                                                                         │
│                                                                                             │
│  FAKE_DAILY_CREDENTIAL 是什么？                                                            │
│  ├─ 不是数据库中的真实凭据                                                                 │
│  ├─ 硬编码的 "凭据" 用于访问 Daily.co API                                                  │
│  ├─ Daily.co 是 Cal.diy 内置的视频会议服务                                                 │
│  └─ 不需要用户单独授权，始终可用                                                           │
│                                                                                             │
│  设计意图：                                                                                  │
│  - 即使视频集成出错，预约也不能失败                                                        │
│  - 至少提供一个可用的会议链接                                                              │
│  - 后续可以发送邮件让用户重新授权                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

#### A.4.2 完整代码解析

**代码位置**: `packages/features/bookings/lib/EventManager.ts:1036-1069`

```typescript
private getVideoCredentialByCalendarEvent(
  event: CalendarEvent
): CredentialForCalendarService | undefined {
  
  // ========== 前置检查：没有位置就没有视频集成 ==========
  if (!event.location) {
    return undefined;
  }

  /**
   * @fixme 注释中的潜在 Bug 说明
   * 
   * 问题：Google Meet 的保存方式
   * - 位置类型："integrations:google:meet"
   * - 提取：integrationName = "google:meet"
   * - 凭据类型："google_video" (不是 "google:meet_video")
   * 
   * 匹配失败："google_video".includes("google:meet") → false
   * 
   * 但为什么实际能工作？因为 Google Meet 有特殊处理：
   * - Google Meet 不是独立的集成
   * - 它是 Google Calendar 集成的"副产品"
   * - 创建日历事件时自动获得 hangoutLink
   */
  // 从 location 提取集成名称
  // "integrations:zoom" → "zoom"
  // "integrations:google:meet" → "google:meet" (注意这里有特殊处理)
  const integrationName = event.location.replace("integrations:", "");
  
  let videoCredential;

  // ========== 分支 1：优先使用 conferenceCredentialId（精确匹配） ==========
  if (event.conferenceCredentialId) {
    // 使用 === 精确匹配 ID
    videoCredential = this.videoCredentials.find(
      (credential) => credential.id === event.conferenceCredentialId
    );
  } 

  // ========== 分支 2：回退 - 按类型模糊匹配 ==========
  else {
    // 使用 includes 模糊匹配
    // "zoom_video".includes("zoom") → true
    videoCredential = this.videoCredentials.find(
      (credential: CredentialForCalendarService) =>
        credential.type.includes(integrationName)
    );
    
    // 记录警告：这是降级路径，不如精确匹配可靠
    log.warn(
      `Could not find conferenceCredentialId for event with location: ${event.location}, ` +
      `trying to use last added video credential`
    );
  }

  // ========== 分支 3：兜底 - 使用 Daily.co ==========
  if (!videoCredential) {
    log.warn(
      `Falling back to "daily" video integration for event with location: ${event.location} ` +
      `because credential is missing for the app`
    );
    // FAKE_DAILY_CREDENTIAL 是硬编码的虚拟凭据
    videoCredential = { ...FAKE_DAILY_CREDENTIAL };
  }

  return videoCredential;
}
```

#### A.4.3 分支决策表

| 场景 | `conferenceCredentialId` | `videoCredentials` 匹配结果 | 使用凭据 | 可靠性 |
|------|---------------------------|------------------------------|----------|--------|
| **正常场景** | 123 | `{id:123, type:"zoom_video"}` 匹配 | ID 为 123 的凭据 | **高** |
| **凭据被删除** | 123 | 无匹配（ID 123 不存在） | 按类型模糊匹配 | **中** |
| **无 credentialId** | undefined | `{id:123, type:"zoom_video"}` | 类型包含 "zoom" 的凭据 | **低** |
| **用户未安装 Zoom** | undefined | 无类型匹配 | FAKE_DAILY_CREDENTIAL | **最低** |

#### A.4.4 conferenceCredentialId 来源链路

让我详细追踪 `conferenceCredentialId` 是如何从 `EventType.locations` 传递到 `CalendarEvent` 的：

```
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                         conferenceCredentialId 传递链路                                      │
└────────────────────────────────────────────────────────────────────────────────────────────┘

  数据库
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ EventType.locations = [{"type":"integrations:zoom","credentialId":123}]             │
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  RegularBookingService.handler()
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ // 输入：eventType.locations (从数据库加载)                                             │
  │ const eventType = await prisma.eventType.findUnique({                                  │
  │     where: { id: eventTypeId },                                                         │
  │     select: { ..., locations: true }                                                    │
  │ });                                                                                      │
  │                                                                                           │
  │ // locationBodyString 来自哪里？                                                         │
  │ // - 通常是 eventType.locations 中配置的默认位置                                         │
  │ // - 或者是预约请求中覆盖的位置                                                           │
  │ const locationBodyString = reqBody.location ??                                          │
  │   (eventType.locations?.[0]?.type || "");  // "integrations:zoom"                     │
  │                                                                                           │
  │ // ========== 关键转换：getLocationValueForDB ==========                                │
  │ const { bookingLocation, conferenceCredentialId: eventTypeCredentialId } =             │
  │     getLocationValueForDB(                                                               │
  │         locationBodyString,     // "integrations:zoom"                                 │
  │         eventType.locations      // [{type:"integrations:zoom", credentialId:123}]   │
  │     );                                                                                   │
  │                                                                                           │
  │ // 结果：                                                                                  │
  │ // bookingLocation = "integrations:zoom" (或实际链接，如果是静态类型)                   │
  │ // conferenceCredentialId = 123                                                          │
  │                                                                                           │
  │ // ========== 构建 CalendarEvent ==========                                              │
  │ const conferenceCredentialId = eventTypeCredentialId;                                   │
  │                                                                                           │
  │ let evt = new CalendarEventBuilder({...})                                                │
  │     .withLocation({                                                                       │
  │         location: platformBookingLocation ?? bookingLocation,                           │
  │         conferenceCredentialId,  // 123 传递进去                                        │
  │     })                                                                                    │
  │     .build();                                                                             │
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  CalendarEvent
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ {                                                                                         │
  │   location: "integrations:zoom",                                                         │
  │   conferenceCredentialId: 123,  // 关键：被 EventManager 使用                          │
  │   startTime: "2026-05-05T10:00:00Z",                                                   │
  │   // ...                                                                                  │
  │ }                                                                                         │
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  EventManager
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ private getVideoCredentialByCalendarEvent(event) {                                      │
  │     // 现在可以使用 event.conferenceCredentialId = 123                                  │
  │     if (event.conferenceCredentialId) {                                                  │
  │         // 分支 1：精确匹配                                                              │
  │         return this.videoCredentials.find(c => c.id === 123);                          │
  │     }                                                                                     │
  │     // ...                                                                                │
  │ }                                                                                         │
  └────────────────────────────────────────────────────────────────────────────────────────┘
```

#### A.4.5 核心转换函数：`getLocationValueForDB`

**代码位置**: `packages/app-store/locations.ts:412-441`

```typescript
export const getLocationValueForDB = (
  bookingLocationTypeOrValue: EventLocationType["type"],  // "integrations:zoom"
  eventLocations: LocationObject[]                        // 来自 eventType.locations
): { bookingLocation: string; conferenceCredentialId?: number } => {
  
  let bookingLocation = bookingLocationTypeOrValue;
  let conferenceCredentialId: number | undefined;

  // ========== 遍历 eventType.locations 找匹配 ==========
  eventLocations.forEach((location) => {
    // 匹配条件：location.type === "integrations:zoom"
    if (location.type === bookingLocationTypeOrValue) {
      
      const eventLocationType = getEventLocationType(bookingLocationTypeOrValue);
      
      // ========== 提取 credentialId（关键！） ==========
      conferenceCredentialId = location.credentialId;  // 123
      
      if (!eventLocationType) {
        return;
      }
      
      // ========== 处理静态 vs 动态链接 ==========
      // 动态链接（如 Zoom）：保持 type 字符串，不保存实际链接
      // 静态链接（如自定义链接）：保存实际 URL
      if (!eventLocationType.default && eventLocationType.linkType === "dynamic") {
        // 动态链接类型：bookingLocation 保持为 type
        // 会议链接会在预约时通过 API 动态生成
        return;
      }

      // 静态链接类型：获取实际值
      bookingLocation = location[eventLocationType.defaultValueVariable] || bookingLocation;
    }
  });

  // ========== 兜底：如果没有位置，使用 Daily.co ==========
  if (bookingLocation.trim().length === 0) {
    bookingLocation = DailyLocationType;  // "integrations:daily"
  }

  return { bookingLocation, conferenceCredentialId };
};
```

#### A.4.6 动态 vs 静态链接类型

| 类型 | `linkType` | 存储值 | 示例 |
|------|------------|--------|------|
| **动态链接** | `"dynamic"` | 存储 `type` | `"integrations:zoom"`（预约时生成链接） |
| **静态链接** | `"static"` | 存储实际 URL | `"https://zoom.us/j/123456"` |

**为什么要区分？**
- **动态链接**：每次预约都调用 Zoom API 创建新会议，生成新链接
- **静态链接**：用户提前提供固定链接，所有预约都用同一个链接

---

### A.5 阶段 4：凭据使用与追踪

#### A.5.1 凭据使用流程

```
┌────────────────────────────────────────────────────────────────────────────────────────────┐
│                              凭据使用完整流程                                                 │
└────────────────────────────────────────────────────────────────────────────────────────────┘

  EventManager.createVideoEvent(event)
      │
      ▼
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ // 步骤 1：选择凭据                                                                      │
  │ const credential = this.getVideoCredentialByCalendarEvent(event);                      │
  │ // credential = { id: 123, type: "zoom_video", key: {...}, userId: 1, appId: "zoom" }│
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ // 步骤 2：调用 videoClient                                                              │
  │ return createMeeting(credential, event);                                                │
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  videoClient.createMeeting()
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ // 步骤 3：检查应用是否启用                                                              │
  │ const enabledApp = await prisma.app.findUnique({                                        │
  │     where: { slug: credential.appId },  // "zoom"                                       │
  │     select: { enabled: true }                                                           │
  │ });                                                                                      │
  │                                                                                           │
  │ if (!enabledApp?.enabled)                                                                │
  │     throw `Location app ${credential.appId} is disabled`;                               │
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ // 步骤 4：获取适配器                                                                     │
  │ const videoAdapters = await getVideoAdapters([credential]);                            │
  │ // 内部：                                                                                 │
  │ // - credential.type = "zoom_video"                                                      │
  │ // - appName = "zoomvideo" (去掉下划线)                                                 │
  │ // - 动态加载 import("./zoomvideo/lib/VideoApiAdapter")                                │
  │ // - 调用 ZoomVideoApiAdapter(credential) 创建实例                                       │
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ // 步骤 5：创建会议                                                                      │
  │ const [firstVideoAdapter] = videoAdapters;                                              │
  │ const createdMeeting = await firstVideoAdapter?.createMeeting(calEvent);               │
  │                                                                                           │
  │ // 返回：                                                                                 │
  │ // {                                                                                      │
  │ //   id: "1234567890",                                                                   │
  │ //   url: "https://zoom.us/j/1234567890?pwd=abc123",                                   │
  │ //   password: "abc123",                                                                 │
  │ //   type: "zoom_video"                                                                  │
  │ // }                                                                                      │
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ // 步骤 6：构建返回对象（包含 credentialId）                                            │
  │ return {                                                                                 │
  │     appName: credential.appName || credential.appId || "",                              │
  │     type: credential.type,          // "zoom_video"                                     │
  │     uid,                                                                                  │
  │     originalEvent: calEvent,                                                             │
  │     success: true,                                                                       │
  │     createdEvent: createdMeeting,                                                        │
  │     credentialId: credential.id,   // 123（关键：用于追踪）                             │
  │ };                                                                                       │
  └───────────────────────────────────────────┬────────────────────────────────────────────┘
                                              │
                                              ▼
  EventManager.create()
  ┌────────────────────────────────────────────────────────────────────────────────────────┐
  │ // 步骤 7：构建 referencesToCreate                                                       │
  │ const referencesToCreate = results.map((result) => {                                    │
  │     return {                                                                              │
  │         type: result.type,                    // "zoom_video"                            │
  │         uid: result.createdEvent?.id,         // "1234567890"                          │
  │         meetingId: result.createdEvent?.id,   // "1234567890"                          │
  │         meetingPassword: result.createdEvent?.password,  // "abc123"                   │
  │         meetingUrl: result.createdEvent?.url, // "https://zoom.us/j/..."                │
  │         credentialId: result.credentialId,    // 123（追踪哪个凭据创建的）              │
  │     };                                                                                    │
  │ });                                                                                       │
  │                                                                                           │
  │ // 保存到 BookingReference 表                                                            │
  └────────────────────────────────────────────────────────────────────────────────────────┘
```

#### A.5.2 BookingReference 数据结构

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                          BookingReference 表                                                  │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│  字段               │  类型      │  示例值                                                   │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│  id                 │  Int       │  456                                                      │
│  bookingId          │  Int       │  789                                                      │
│  type               │  String    │  "zoom_video"                                             │
│  uid                │  String    │  "1234567890"  (Zoom 会议 ID)                           │
│  meetingId          │  String?   │  "1234567890"                                            │
│  meetingPassword    │  String?   │  "abc123"                                                 │
│  meetingUrl         │  String?   │  "https://zoom.us/j/1234567890?pwd=abc123"             │
│  credentialId       │  Int?      │  123  (对应 Credential.id)                                │
│  deleted            │  Boolean   │  false                                                    │
└─────────────────────────────────────────────────────────────────────────────────────────────┘

  关键关联：
  ├─ BookingReference.credentialId ──> Credential.id
  ├─ BookingReference.type ──> Credential.type (用于识别集成类型)
  └─ BookingReference.uid ──> Zoom 会议 ID (用于后续更新/删除会议)
```

#### A.5.3 为什么要保存 credentialId？

| 场景 | 用途 |
|------|------|
| **改期预约** | 找到原凭据，调用 `updateMeeting` 更新会议时间 |
| **取消预约** | 找到原凭据，调用 `deleteMeeting` 删除 Zoom 会议 |
| **故障排查** | 知道是哪个凭据出了问题，帮助用户重新授权 |
| **审计追踪** | 记录哪些凭据被哪些预约使用 |

---

### A.6 关键数据结构关联图

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     完整数据关联图                                                     │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────┐
  │      User        │
  │  id: 1           │
  └────────┬─────────┘
           │
           │ 1:N
           ▼
  ┌──────────────────┐         ┌──────────────────────────────────────────────────────────┐
  │    Credential    │         │  数据示例                                                 │
  ├──────────────────┤         ├──────────────────────────────────────────────────────────┤
  │ id: 123          │         │  {                                                       │
  │ type: "zoom_vid..│◄────────│    type: "zoom_video",                                  │
  │ userId: 1        │         │    key: { access_token, refresh_token, expiry_date },  │
  │ appId: "zoom"    │         │    userId: 1,                                            │
  │ key: {...}       │         │    appId: "zoom"                                         │
  └────────┬─────────┘         │  }                                                       │
           │                   └──────────────────────────────────────────────────────────┘
           │
           │ 被 EventType.locations 引用
           │
           ▼
  ┌──────────────────┐         ┌──────────────────────────────────────────────────────────┐
  │    EventType     │         │  数据示例 (locations 字段)                               │
  ├──────────────────┤         ├──────────────────────────────────────────────────────────┤
  │ id: 500          │         │  [                                                       │
  │ userId: 1        │         │    {                                                     │
  │ locations: [..]  │◄────────│      type: "integrations:zoom",                        │
  └────────┬─────────┘         │      credentialId: 123  ◄──── 引用 Credential.id       │
           │                   │    }                                                     │
           │                   │  ]                                                       │
           │                   └──────────────────────────────────────────────────────────┘
           │ 决定预约时的默认位置
           │
           ▼
  ┌──────────────────┐         ┌──────────────────────────────────────────────────────────┐
  │  CalendarEvent   │         │  数据示例                                                 │
  ├──────────────────┤         ├──────────────────────────────────────────────────────────┤
  │ location: "in..  │◄────────│  {                                                       │
  │ conferenceCre..  │         │    location: "integrations:zoom",                      │
  │ startTime: ".."  │         │    conferenceCredentialId: 123,  ◄── 从 EventType 来   │
  └────────┬─────────┘         │    startTime: "2026-05-05T10:00:00Z"                  │
           │                   │  }                                                       │
           │                   └──────────────────────────────────────────────────────────┘
           │ 被 EventManager 用于选择凭据
           │
           ▼
  ┌──────────────────┐
  │  EventManager    │  使用 conferenceCredentialId 或 location 从 videoCredentials 中
  └────────┬─────────┘  选择对应的 Credential
           │
           │ 创建会议后生成引用
           │
           ▼
  ┌──────────────────┐         ┌──────────────────────────────────────────────────────────┐
  │ BookingReference │         │  数据示例                                                 │
  ├──────────────────┤         ├──────────────────────────────────────────────────────────┤
  │ id: 999          │         │  {                                                       │
  │ bookingId: 789   │         │    type: "zoom_video",                                  │
  │ type: "zoom_vid..│         │    uid: "1234567890",  (Zoom 会议 ID)                 │
  │ uid: "123456.."  │         │    meetingUrl: "https://zoom.us/j/...",                │
  │ credentialId: 123│◄────────│    credentialId: 123,  ◄── 回溯到哪个凭据创建的         │
  │ meetingUrl: "h.. │         │    meetingPassword: "abc123"                            │
  └──────────────────┘         │  }                                                       │
                               └──────────────────────────────────────────────────────────┘
```

---

### A.7 完整时序图

```
┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐     ┌────────────┐
│   User     │     │   Frontend │     │   Backend  │     │   OAuth    │     │  Database  │
│            │     │            │     │            │     │   Provider │     │            │
└─────┬──────┘     └─────┬──────┘     └─────┬──────┘     └─────┬──────┘     └─────┬──────┘
      │                   │                   │                   │                   │
      │                   │                   │                   │                   │
      │ [阶段 1: OAuth 授权]                                                 │
      │                   │                   │                   │                   │
      │ 点击"添加 Zoom"  │                   │                   │                   │
      │──────────────────>│                   │                   │                   │
      │                   │                   │                   │                   │
      │                   │ GET /api/zoomvideo/add              │                   │
      │                   │──────────────────>│                   │                   │
      │                   │                   │                   │                   │
      │                   │                   │ 构建 OAuth URL    │                   │
      │                   │                   │                   │                   │
      │                   │ 302 重定向到授权页                   │                   │
      │<──────────────────│                   │                   │                   │
      │                   │                   │                   │                   │
      │ 重定向到 Zoom 登录页面                │                   │                   │
      │───────────────────────────────────────>│                   │                   │
      │                   │                   │                   │                   │
      │                   │                   │                   │ 用户授权同意      │
      │                   │                   │                   │                   │
      │                   │ 302 回调到 callback│                   │                   │
      │<──────────────────|───────────────────│                   │                   │
      │                   │                   │                   │                   │
      │                   │                   │ POST /token (换取 token)              │
      │                   │                   │──────────────────>│                   │
      │                   │                   │                   │                   │
      │                   │                   │<──────────────────│                   │
      │                   │                   │ { access_token,.. }                   │
      │                   │                   │                   │                   │
      │                   │                   │ DELETE old credentials              │
      │                   │                   │──────────────────────────────────────>│
      │                   │                   │                   │                   │
      │                   │                   │ CREATE new Credential                │
      │                   │                   │ (type: zoom_video)                   │
      │                   │                   │──────────────────────────────────────>│
      │                   │                   │                   │                   │
      │                   │ 302 重定向到已安装应用页              │                   │
      │<──────────────────│                   │                   │                   │
      │                   │                   │                   │                   │
      │ [阶段 2: 关联事件类型]                                                │
      │                   │                   │                   │                   │
      │ 设置 Zoom 为默认 │                   │                   │                   │
      │──────────────────>│                   │                   │                   │
      │                   │                   │                   │                   │
      │                   │ POST 设置默认会议应用               │                   │
      │                   │──────────────────>│                   │                   │
      │                   │                   │                   │                   │
      │                   │                   │ UPDATE EventType.locations           │
      │                   │                   │ {type:"integrations:zoom",           │
      │                   │                   │  credentialId:123}                    │
      │                   │                   │──────────────────────────────────────>│
      │                   │                   │                   │                   │
      │ [阶段 3: 预约时凭据选择]                                              │
      │                   │                   │                   │                   │
      │ 发起预约请求      │                   │                   │                   │
      │──────────────────>│                   │                   │                   │
      │                   │                   │                   │                   │
      │                   │ POST /book        │                   │                   │
      │                   │──────────────────>│                   │                   │
      │                   │                   │                   │                   │
      │                   │                   │ 加载 EventType.locations              │
      │                   │                   │──────────────────────────────────────>│
      │                   │                   │                   │                   │
      │                   │                   │<──────────────────────────────────────│
      │                   │                   │ [{type:"integrations:zoom",         │
      │                   │                   │   credentialId:123}]                  │
      │                   │                   │                   │                   │
      │                   │                   │ getLocationValueForDB()              │
      │                   │                   │ 提取 conferenceCredentialId=123      │
      │                   │                   │                   │                   │
      │                   │                   │ EventManager.getVideoCredential...() │
      │                   │                   │ 分支 1: ID 精确匹配 (123)          │
      │                   │                   │                   │                   │
      │                   │                   │ 加载用户凭据                        │
      │                   │                   │──────────────────────────────────────>│
      │                   │                   │                   │                   │
      │                   │                   │<──────────────────────────────────────│
      │                   │                   │ Credential {id:123, type:zoom_video}│
      │                   │                   │                   │                   │
      │ [阶段 4: 凭据使用与追踪]                                              │
      │                   │                   │                   │                   │
      │                   │                   │ getVideoAdapters([credential])      │
      │                   │                   │ 动态加载 ZoomVideoApiAdapter         │
      │                   │                   │                   │                   │
      │                   │                   │ adapter.createMeeting(event)         │
      │                   │                   │                   │                   │
      │                   │                   │ POST Zoom API 创建会议               │
      │                   │                   │──────────────────>│                   │
      │                   │                   │                   │                   │
      │                   │                   │<──────────────────│                   │
      │                   │                   │ {id, url, password}                  │
      │                   │                   │                   │                   │
      │                   │                   │ CREATE BookingReference               │
      │                   │                   │ {type:zoom_video,                    │
      │                   │                   │  credentialId:123,                   │
      │                   │                   │  meetingUrl:"https://..."}            │
      │                   │                   │──────────────────────────────────────>│
      │                   │                   │                   │                   │
      │                   │ 返回预约确认（含会议链接）           │                   │
      │                   │<──────────────────│                   │                   │
      │<──────────────────│                   │                   │                   │
```

---

## 6. 完整调用时序图

### 6.1 新预约视频会议创建时序

```
┌──────┐     ┌────────────────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
│ Client │     │ RegularBookingService │    │ EventManager │    │ videoClient │    │ ZoomAdapter │
└──┬───┘     └──────────┬─────────┘    └──────┬───────┘    └──────┬──────┘    └──────┬───────┘
   │                    │                     │                   │                  │
   │  POST /book        │                     │                   │                  │
   │───────────────────>│                     │                   │                  │
   │                    │                     │                   │                  │
   │                    │ new EventManager()  │                   │                  │
   │                    │ with credentials ───>│                   │                  │
   │                    │                     │                   │                  │
   │                    │ eventManager.create()│                   │                  │
   │                    │ with CalendarEvent  ─>│                   │                  │
   │                    │                     │                   │                  │
   │                    │                     │ processLocation() │                  │
   │                    │                     │ location = "integrations:zoom"       │
   │                    │                     │                   │                  │
   │                    │                     │ isDedicated = true│                  │
   │                    │                     │                   │                  │
   │                    │                     │ createVideoEvent()│                  │
   │                    │                     │──────────────────>│                  │
   │                    │                     │                   │                  │
   │                    │                     │                   │ getVideoAdapters()│
   │                    │                     │                   │ with credential  ─>│
   │                    │                     │                   │                  │
   │                    │                     │                   │                  │ OAuthManager.init()
   │                    │                     │                   │                  │
   │                    │                     │                   │ createMeeting()  │
   │                    │                     │                   │ with CalendarEvent ─>
   │                    │                     │                   │                  │
   │                    │                     │                   │                  │ POST /users/me/meetings
   │                    │                     │                   │                  │ to Zoom API
   │                    │                     │                   │                  │
   │                    │                     │                   │                  │<── VideoCallData
   │                    │                     │                   │                  │  {id, url, password}
   │                    │                     │                   │                  │
   │                    │                     │<──────────────────│                  │
   │                    │                     │    EventResult    │                  │
   │                    │                     │                   │                  │
   │                    │                     │ createAllCalendarEvents()           │
   │                    │                     │ with videoCallData │                  │
   │                    │                     │                   │                  │
   │                    │                     │ build referencesToCreate            │
   │                    │                     │ with meetingUrl/id/password         │
   │                    │<────────────────────│                   │                  │
   │                    │    CreateUpdateResult                   │                  │
   │                    │                     │                   │                  │
   │                    │ prisma.booking.update()                  │                  │
   │                    │ with references.createMany              │                  │
   │                    │                     │                   │                  │
   │<───────────────────│                     │                   │                  │
   │   BookingResponse  │                     │                   │                  │
   │   with videoCallUrl│                     │                   │                  │
```

---

## 7. 关键代码路径索引

### 7.1 插件注册层

| 功能 | 文件路径 | 关键行 |
|------|----------|--------|
| 元数据定义 | `packages/app-store/zoomvideo/_metadata.ts` | 3-28 |
| 元数据类型 | `packages/types/App.d.ts` | 52-171 |
| 适配器映射生成 | `packages/app-store/video.adapters.generated.ts` | 1-21 |
| 元数据规范化 | `packages/app-store/getNormalizedAppMetadata.ts` | 12-28 |
| 应用注册表 | `packages/app-store/_appRegistry.ts` | 19-37 |

### 7.2 预约流程层

| 功能 | 文件路径 | 关键行 |
|------|----------|--------|
| 预约服务入口 | `packages/features/bookings/lib/service/RegularBookingService.ts` | 1703-2170 |
| EventManager 构造 | `packages/features/bookings/lib/EventManager.ts` | 133-173 |
| 事件创建流程 | `packages/features/bookings/lib/EventManager.ts` | 287-416 |
| 位置检测 | `packages/features/bookings/lib/EventManager.ts` | 96-109 |
| 视频凭据获取 | `packages/features/bookings/lib/EventManager.ts` | 1036-1069 |

### 7.3 适配器层

| 功能 | 文件路径 | 关键行 |
|------|----------|--------|
| 适配器工厂 | `packages/app-store/getVideoAdapters.ts` | 11-45 |
| 视频客户端 | `packages/features/conferencing/lib/videoClient.ts` | 29-103 |
| Zoom 适配器 | `packages/app-store/zoomvideo/lib/VideoApiAdapter.ts` | 235-577 |
| 适配器接口 | `packages/types/VideoApiAdapter.d.ts` | 12-50 |
| OAuth 管理器 | `packages/app-store/_utils/oauth/OAuthManager.ts` | - |

---

## 8. 插件开发指南

### 8.1 创建新视频插件步骤

1. **创建目录结构**
   ```bash
   mkdir -p packages/app-store/myvideo/{lib,api,static}
   ```

2. **定义元数据** (`_metadata.ts`)
   ```typescript
   import type { AppMeta } from "@calcom/types/App";
   
   export const metadata = {
     name: "My Video",
     type: "my_video",
     categories: ["conferencing"],
     variant: "conferencing",
     slug: "myvideo",
     dirName: "myvideo",
     logo: "icon.svg",
     publisher: "My Company",
     url: "https://myvideo.com",
     email: "support@myvideo.com",
     isOAuth: true,
     appData: {
       location: {
         linkType: "dynamic",
         type: "integrations:myvideo",
         label: "My Video",
       },
     },
   } as AppMeta;
   ```

3. **实现适配器** (`lib/VideoApiAdapter.ts`)
   ```typescript
   import type { VideoApiAdapter, VideoApiAdapterFactory } from "@calcom/types/VideoApiAdapter";
   
   const MyVideoAdapter: VideoApiAdapterFactory = (credential) => {
     return {
       createMeeting: async (event) => {
         // 调用 API 创建会议
         return { type: "my_video", id: "...", url: "...", password: "" };
       },
       updateMeeting: async (bookingRef, event) => {
         // 更新会议
         return { type: "my_video", id: "...", url: "...", password: "" };
       },
       deleteMeeting: async (uid) => {
         // 删除会议
         return {};
       },
       getAvailability: async () => {
         // 获取忙闲时间
         return [];
       },
     };
   };
   
   export default MyVideoAdapter;
   ```

4. **导出模块** (`index.ts`)
   ```typescript
   export * as api from "./api";
   export * as lib from "./lib";
   export { metadata } from "./_metadata";
   ```

5. **实现 OAuth 流程** (`api/add.ts`, `api/callback.ts`)

6. **重新生成适配器映射**
   ```bash
   yarn app-store:build
   ```

### 8.2 插件类型约定

| 类型后缀 (`variant`) | 用途 | 必需接口 |
|----------------------|------|----------|
| `conferencing` / `video` | 视频会议 | `VideoApiAdapter` |
| `calendar` | 日历同步 | `Calendar` 接口 |
| `payment` | 支付处理 | `PaymentService` 接口 |
| `crm` | CRM 集成 | `CrmService` 接口 |
| `analytics` | 分析追踪 | 脚本注入 (tag) |
| `automation` | 自动化工作流 | Webhook 触发 |

---

## 9. 总结

### 9.1 核心设计亮点

1. **基于类型的动态分派**
   - 通过 `credential.type` 和 `event.location` 双重匹配
   - 适配器工厂模式实现按需加载

2. **统一的 EventManager 协调**
   - 集中管理日历、视频、CRM 等多种集成
   - 统一的 `referencesToCreate` 数据结构

3. **OAuth 令牌自动管理**
   - `OAuthManager` 封装令牌刷新逻辑
   - 自动检测 token 失效并更新

4. **优雅的错误降级**
   - 视频会议创建失败时自动回退到 Daily.co
   - 集成错误不阻塞主预约流程

### 9.2 数据流摘要

```
预约请求
    │
    ▼
┌─────────────────┐
│ 构建 CalendarEvent │
│  - location: "integrations:zoom"
│  - conferenceCredentialId
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  EventManager   │
│  - 按类型分类凭据
│  - 检测专用集成位置
│  - 调用对应适配器
└────────┬────────┘
         │
         ▼
┌─────────────────────────┐
│  videoClient.createMeeting │
│  - 获取适配器
│  - 调用 Zoom API
│  - 返回 VideoCallData
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│  保存 BookingReference  │
│  - type: "zoom_video"
│  - meetingUrl, meetingId
│  - credentialId
└─────────────────────────┘
```

### 9.3 关键文件速查表

| 职责 | 文件 |
|------|------|
| 插件元数据 | `packages/app-store/*/_metadata.ts` |
| 适配器映射 | `packages/app-store/video.adapters.generated.ts` |
| 预约服务 | `packages/features/bookings/lib/service/RegularBookingService.ts` |
| 集成协调 | `packages/features/bookings/lib/EventManager.ts` |
| 视频客户端 | `packages/features/conferencing/lib/videoClient.ts` |
| 适配器工厂 | `packages/app-store/getVideoAdapters.ts` |
| 类型定义 | `packages/types/VideoApiAdapter.d.ts`, `packages/types/App.d.ts` |
