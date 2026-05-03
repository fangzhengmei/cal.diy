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
