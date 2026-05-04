# 视频会议链接生成与外部 Provider 接入分析

## 1. 概述

Cal.diy 支持多种视频会议解决方案，包括内置的 Cal Video 以及与外部平台（如 Zoom、Google Meet、Microsoft Teams 等）的集成。本文档详细分析视频会议链接的生成机制、不同外部 Provider 的接入方式，以及会议链接的分发流程。

## 2. 核心架构

### 2.1 主要组件

| 组件 | 路径 | 职责 |
|------|------|------|
| EventManager | `packages/features/bookings/lib/EventManager.ts` | 协调日历事件和视频会议的创建、更新、删除 |
| videoClient | `packages/features/conferencing/lib/videoClient.ts` | 统一的视频会议客户端，封装 createMeeting/updateMeeting/deleteMeeting |
| VideoApiAdapter | `packages/types/VideoApiAdapter.d.ts` | 视频会议 Provider 的统一接口定义 |
| getVideoAdapters | `packages/app-store/getVideoAdapters.ts` | 根据凭据动态加载对应的 VideoApiAdapter |
| BookingReferenceRepository | 多处 | 管理预订引用（包括视频会议信息）的存储 |

### 2.2 数据流概览

```
预订创建
    ↓
EventManager.create()
    ↓
processLocation() - 处理位置类型
    ↓
isDedicatedIntegration() - 判断是否为专用视频集成
    ↓
createVideoEvent()
    ↓
getVideoCredentialByCalendarEvent() - 获取视频凭据
    ↓
videoClient.createMeeting()
    ↓
getVideoAdapters() - 加载对应 Provider 的 Adapter
    ↓
VideoApiAdapter.createMeeting() - 调用具体 Provider API
    ↓
返回 VideoCallData { id, password, url, type }
    ↓
存储到 BookingReference
    ↓
同步到日历事件 + 邮件通知
```

## 3. 视频会议链接生成机制

### 3.1 核心创建流程

视频会议链接的创建主要通过 `videoClient.ts` 中的 `createMeeting` 函数实现：

**文件**: `packages/features/conferencing/lib/videoClient.ts:29-103`

```typescript
const createMeeting = async (
  credential: CredentialPayload | CredentialForCalendarService,
  calEvent: CalendarEvent
) => {
  // 1. 验证凭据
  if (!credential || !credential.appId) {
    throw new Error("Credentials must be set!");
  }

  // 2. 获取对应的 VideoApiAdapter
  const videoAdapters = await getVideoAdapters([credential]);
  const [firstVideoAdapter] = videoAdapters;

  // 3. 检查 App 是否启用
  const enabledApp = await prisma.app.findUnique({
    where: { slug: credential.appId },
    select: { enabled: true },
  });

  if (!enabledApp?.enabled)
    throw `Location app ${credential.appId} is either disabled or not seeded at all`;

  // 4. 调用 Adapter 创建会议
  createdMeeting = await firstVideoAdapter?.createMeeting(calEvent);

  // 5. 返回结果（失败时回退到 Cal Video）
  return {
    appName: credential.appName || credential.appId || "",
    type: credential.type,
    uid,
    originalEvent: calEvent,
    success: true,
    createdEvent: createdMeeting,
    credentialId: credential.id,
  };
};
```

### 3.2 VideoApiAdapter 接口定义

**文件**: `packages/types/VideoApiAdapter.d.ts:12-52`

```typescript
export interface VideoCallData {
  type: string;      // 如 "zoom_video", "daily_video", "office365_video"
  id: string;        // 会议 ID
  password: string;  // 会议密码/令牌
  url: string;       // 会议加入链接
}

export type VideoApiAdapter = {
  createMeeting(event: CalendarEvent): Promise<VideoCallData>;
  updateMeeting(bookingRef: PartialReference, event: CalendarEvent): Promise<VideoCallData>;
  deleteMeeting(uid: string): Promise<unknown>;
  getAvailability(dateFrom?: string, dateTo?: string): Promise<EventBusyDate[]>;
  // 可选方法（主要用于 Cal Video）
  getRecordings?(roomName: string): Promise<GetRecordingsResponseSchema>;
  createInstantCalVideoRoom?(endTime: string): Promise<VideoCallData>;
  // ... 更多转录/录制相关方法
};
```

### 3.3 动态 Adapter 加载机制

**文件**: `packages/app-store/getVideoAdapters.ts:11-45`

```typescript
export const getVideoAdapters = async (withCredentials: CredentialPayload[]): Promise<VideoApiAdapter[]> => {
  const videoAdapters: VideoApiAdapter[] = [];

  for (const cred of withCredentials) {
    // 转换凭据类型为 App 名称：zoom_video -> zoomvideo
    const appName = cred.type.split("_").join("");
    
    // 从自动生成的 VideoApiAdapterMap 中获取
    let videoAdapterImport = VideoApiAdapterMap[appName as keyof typeof VideoApiAdapterMap];

    // 回退机制：zoom_video -> zoom
    if (!videoAdapterImport) {
      const appTypeVariant = cred.type.substring(0, cred.type.lastIndexOf("_"));
      videoAdapterImport = VideoApiAdapterMap[appTypeVariant as keyof typeof VideoApiAdapterMap];
    }

    if (videoAdapterImport) {
      const videoAdapterModule = await videoAdapterImport;
      const makeVideoApiAdapter = videoAdapterModule.default as VideoApiAdapterFactory;
      const videoAdapter = makeVideoApiAdapter(cred);
      videoAdapters.push(videoAdapter);
    }
  }

  return videoAdapters;
};
```

**关键点**:
- `VideoApiAdapterMap` 是由 `app-store-cli` 自动生成的
- 命名约定：`zoom_video` 类型对应 `packages/app-store/zoomvideo/lib/VideoApiAdapter.ts`
- 支持两种命名映射：`zoomvideo` 和 `zoom`

### 3.4 会议信息存储

视频会议信息存储在 `BookingReference` 表中，包含以下关键字段：

| 字段 | 说明 |
|------|------|
| `type` | 引用类型，如 `zoom_video`, `daily_video`, `google_calendar` |
| `uid` | 外部服务的唯一标识符 |
| `meetingId` | 会议 ID |
| `meetingPassword` | 会议密码或访问令牌 |
| `meetingUrl` | 会议加入链接（核心字段） |
| `credentialId` | 关联的凭据 ID |
| `externalCalendarId` | 对于日历类型，关联的日历 ID |

**文件**: `packages/features/bookings/lib/EventManager.ts:388-410`

```typescript
const referencesToCreate = results.map((result) => {
  return {
    type: result.type,
    uid: createdEventObj ? createdEventObj.id : (result.createdEvent?.id?.toString() ?? ""),
    meetingId: createdEventObj ? createdEventObj.id : result.createdEvent?.id?.toString(),
    meetingPassword: createdEventObj ? createdEventObj.password : result.createdEvent?.password,
    meetingUrl: createdEventObj ? createdEventObj.onlineMeetingUrl : result.createdEvent?.url,
    externalCalendarId: isCalendarType ? result.externalId : undefined,
    ...getCredentialPayload(result),
  };
});
```

## 4. 外部会议 Provider 接入方式

Cal.diy 支持多种视频会议 Provider，它们的接入方式主要分为三类：

### 4.1 分类概览

| 类型 | Provider | 接入方式 | 依赖 |
|------|----------|----------|------|
| 内置 | Cal Video (Daily.co) | 全局 API Key，无需用户凭据 | `DAILY_API_KEY` 环境变量 |
| 直接 API 集成 | Zoom, Webex, Jitsi 等 | 独立 VideoApiAdapter，直接调用 Provider API | 用户 OAuth 凭据 |
| 日历联动 | Google Meet, Microsoft Teams | 通过日历 API 间接创建会议 | 日历集成 + 会议创建请求 |

### 4.2 内置方案：Cal Video (Daily.co)

Cal Video 是基于 Daily.co 的内置视频会议解决方案，无需用户单独配置。

**文件**: `packages/app-store/dailyvideo/lib/VideoApiAdapter.ts:239-519`

**核心特点**:
1. **全局凭据**: 使用 `FAKE_DAILY_CREDENTIAL`（ID 为 0），不关联特定用户
2. **自动过期**: 会议室在会议结束后 14 天过期
3. **区域配置**: 可通过 `DAILY_VIDEO_REGION` 环境变量指定区域
4. **高级功能**: 支持云录制、转录（需要 Scale Plan 和团队订阅）

**创建会议流程**:
```typescript
const DailyVideoApiAdapter = (): VideoApiAdapter => {
  return {
    createMeeting: async (event: CalendarEvent): Promise<VideoCallData> => {
      // 1. 构建会议室配置
      const body = {
        privacy: "public",
        properties: {
          geo: region,  // 可选区域
          enable_prejoin_ui: true,
          enable_knocking: true,
          enable_screenshare: true,
          enable_chat: true,
          exp: exp,  // 过期时间
          enable_recording: enableRecording,  // 云录制
          enable_transcription_storage: isTranscriptionEnabled,  // 转录
        },
      };

      // 2. 调用 Daily.co API 创建房间
      const dailyEvent = await postToDailyAPI("/rooms", body);
      
      // 3. 创建主持人访问令牌
      const meetingToken = await postToDailyAPI("/meeting-tokens", {
        properties: {
          room_name: dailyEvent.name,
          exp: dailyEvent.config.exp,
          is_owner: true,
        },
      });

      // 4. 返回会议数据
      return {
        type: "daily_video",
        id: dailyEvent.name,
        password: meetingToken.token,  // 令牌作为密码
        url: dailyEvent.url,  // 格式: https://{domain}.daily.co/{roomName}
      };
    },
    // ... 其他方法
  };
};
```

**凭据定义**:
**文件**: `packages/app-store/dailyvideo/lib/VideoApiAdapter.ts:85-98`
```typescript
export const FAKE_DAILY_CREDENTIAL: CredentialForCalendarService & { invalid: boolean } = {
  id: 0,
  type: "daily_video",
  key: { apikey: process.env.DAILY_API_KEY },
  userId: 0,
  user: { email: "" },
  appId: "daily-video",
  invalid: false,
  teamId: null,
  encryptedKey: null,
  delegatedToId: null,
  delegatedTo: null,
  delegationCredentialId: null,
};
```

### 4.3 直接 API 集成：Zoom

Zoom 是最典型的直接 API 集成案例，通过 OAuth 2.0 授权后直接调用 Zoom API。

**文件**: `packages/app-store/zoomvideo/lib/VideoApiAdapter.ts:235-577`

**核心特点**:
1. **OAuth 2.0**: 使用 Zoom OAuth 流程获取访问令牌
2. **令牌管理**: `OAuthManager` 自动处理令牌刷新
3. **密码合规**: 检查并满足 Zoom 用户的密码复杂度要求
4. **会议设置**: 支持等候室、自动录制等高级设置

**创建会议流程**:
```typescript
const ZoomVideoApiAdapter = (credential: CredentialPayload): VideoApiAdapter => {
  return {
    createMeeting: async (event: CalendarEvent): Promise<VideoCallData> => {
      // 1. 转换事件为 Zoom API 格式
      const translatedEvent = await translateEvent(event);
      // 包含: topic, type, start_time, duration, timezone, agenda, settings 等

      // 2. 通过 OAuthManager 调用 Zoom API（自动处理令牌刷新）
      const response = await fetchZoomApi("users/me/meetings", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(translatedEvent),
      });

      // 3. 解析响应
      const result = zoomEventResultSchema.parse(response);

      // 4. 返回会议数据
      return {
        type: "zoom_video",
        id: result.id.toString(),
        password: result.password || "",
        url: result.join_url,  // 格式: https://zoom.us/j/{meetingId}
      };
    },
    // ... 其他方法
  };
};
```

**事件转换逻辑**:
**文件**: `packages/app-store/zoomvideo/lib/VideoApiAdapter.ts:251-349`
```typescript
const translateEvent = async (event: CalendarEvent) => {
  // 获取用户设置（密码要求、等候室等）
  const userSettings = await getUserSettings();
  
  // 处理重复会议
  const recurrence = getRecurrence(event);
  
  // 密码合规检查
  const passwordRequirements = userSettings?.schedule_meeting?.meeting_password_requirement;
  const password = getCompliantPassword(defaultPassword, passwordRequirements);

  return {
    topic: event.title,
    type: 2,  //  scheduled meeting
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
      waiting_room: waitingRoomEnabled,
    },
    ...recurrence,
  };
};
```

**OAuth 管理**:
`OAuthManager` 是通用的 OAuth 2.0 令牌管理组件，负责：
- 自动检测令牌过期
- 使用 refresh_token 刷新访问令牌
- 更新数据库中的凭据
- 处理令牌失效场景

### 4.4 日历联动：Google Meet

Google Meet 不提供独立的会议创建 API，而是通过 Google Calendar API 的 `conferenceData` 功能间接创建。

**核心逻辑**:
**文件**: `packages/features/bookings/lib/EventManager.ts:63-79`
```typescript
export const getLocationRequestFromIntegration = (location: string) => {
  const eventLocationType = getLocationFromApp(location);
  if (eventLocationType) {
    const requestId = uuidv5(location, uuidv5.URL);

    return {
      conferenceData: {
        createRequest: {
          requestId: requestId,  // 幂等请求 ID
        },
      },
      location,
    };
  }
  return null;
};
```

**处理流程**（在 EventManager.create 中）:
**文件**: `packages/features/bookings/lib/EventManager.ts:96-109`
```typescript
export const processLocation = (event: CalendarEvent): CalendarEvent => {
  // 如果位置是集成类型（如 integrations:google:meet）
  if (event.location?.includes("integration")) {
    const maybeLocationRequestObject = getLocationRequestFromIntegration(event.location);
    // 将 conferenceData.createRequest 合并到事件中
    event = merge(event, maybeLocationRequestObject);
  }
  return event;
};
```

**位置类型定义**:
**文件**: `packages/app-store/constants.ts`（引用自 locations.ts）
```typescript
export const MeetLocationType = "integrations:google:meet";
```

**特殊回退逻辑**:
**文件**: `packages/features/bookings/lib/EventManager.ts:317-331`
```typescript
// 如果选择了 Google Meet 但没有连接 Google Calendar，回退到 Cal Video
if (evt.location === MeetLocationType && mainHostDestinationCalendar?.integration !== "google_calendar") {
  const [googleCalendarCredential] = this.calendarCredentials.filter(
    (cred) => cred.type === "google_calendar"
  );
  if (!isDelegationCredential({ credentialId: googleCalendarCredential?.id })) {
    evt["location"] = "integrations:daily";
    evt["conferenceCredentialId"] = undefined;
  }
}
```

**关键点**:
1. Google Meet 链接是在创建 Google Calendar 事件时，通过 `conferenceData.createRequest` 自动生成的
2. 日历事件创建后，响应中会包含 `hangoutLink`（Meet 链接）
3. 必须先连接 Google Calendar 才能使用 Google Meet
4. 没有 Google Calendar 时自动回退到 Cal Video

### 4.5 日历联动：Microsoft Teams

Microsoft Teams 支持两种接入方式：
1. **通过 Outlook 日历联动**（推荐）- 与 Google Meet 类似
2. **直接 Graph API 调用** - 独立的 Online Meetings API

#### 方式一：Outlook 日历联动

**文件**: `packages/features/bookings/lib/EventManager.ts:334-336`
```typescript
const isMSTeamsWithOutlookCalendar =
  evt.location === MSTeamsLocationType &&
  mainHostDestinationCalendar?.integration === "office365_calendar";
```

**条件判断**:
**文件**: `packages/features/bookings/lib/EventManager.ts:341-342`
```typescript
// 如果是专用视频集成且不是 Teams + Outlook 组合，才创建独立视频会议
if (isDedicated && !isMSTeamsWithOutlookCalendar) {
  const result = await this.createVideoEvent(evt);
  // ...
}
```

**视频数据更新**:
**文件**: `packages/features/bookings/lib/EventManager.ts:251-275`
```typescript
private updateMSTeamsVideoCallData(
  evt: CalendarEvent,
  results: Array<EventResult<Exclude<Event, AdditionalInformation>>>
) {
  // 查找成功创建的 Office 365 日历事件，其中包含 Teams 链接
  const office365CalendarWithTeams = results.find(
    (result) => result.type === "office365_calendar" && result.success && result.createdEvent?.url
  );
  if (office365CalendarWithTeams) {
    evt.videoCallData = {
      type: "office365_video",
      id: office365CalendarWithTeams.createdEvent?.id,
      password: "",
      url: office365CalendarWithTeams.createdEvent?.url,
    };
    // ...
  }
}
```

#### 方式二：直接 Graph API 调用

当用户连接了 Office 365 Video 但没有 Outlook 日历时，使用独立的 VideoApiAdapter。

**文件**: `packages/app-store/office365video/lib/VideoApiAdapter.ts:40-319`

```typescript
const TeamsVideoApiAdapter = (credential: CredentialForCalendarServiceWithTenantId): VideoApiAdapter => {
  return {
    createMeeting: async (event: CalendarEvent): Promise<VideoCallData> => {
      // 1. 构建 Graph API 端点
      const url = `${await getUserEndpoint()}/onlineMeetings`;
      
      // 2. 转换事件格式
      const body = {
        startDateTime: event.startTime,
        endDateTime: event.endTime,
        subject: event.title,
      };

      // 3. 调用 Microsoft Graph API
      const response = await auth.requestRaw({
        url,
        options: {
          method: "POST",
          body: JSON.stringify(body),
        },
      });

      // 4. 解析响应
      const resultObject = JSON.parse(await response.text());

      return {
        type: "office365_video",
        id: resultObject.id,
        password: "",
        url: resultObject.joinWebUrl || resultObject.joinUrl,
      };
    },
    // ... 其他方法
  };
};
```

**用户端点确定**:
**文件**: `packages/app-store/office365video/lib/VideoApiAdapter.ts:218-223`
```typescript
async function getUserEndpoint(): Promise<string> {
  const azureUserId = await getAzureUserId(credential);
  return azureUserId
    ? `https://graph.microsoft.com/v1.0/users/${azureUserId}`
    : "https://graph.microsoft.com/v1.0/me";
}
```

**委托凭据支持**:
Teams Video 特别支持委托凭据（Delegation Credential），用于组织级别的服务账户集成。

### 4.6 其他支持的 Provider

除了上述主要 Provider，Cal.diy 还支持以下视频会议服务，均通过独立的 `VideoApiAdapter` 实现：

| Provider | 类型标识 | Adapter 路径 |
|----------|----------|---------------|
| Webex | `webex_video` | `packages/app-store/webex/lib/VideoApiAdapter.ts` |
| Jitsi | `jitsi_video` | `packages/app-store/jitsivideo/lib/VideoApiAdapter.ts` |
| Tandem | `tandem_video` | `packages/app-store/tandemvideo/lib/VideoApiAdapter.ts` |
| Sylaps | `sylaps_video` | `packages/app-store/sylapsvideo/lib/VideoApiAdapter.ts` |
| Shimmer | `shimmer_video` | `packages/app-store/shimmervideo/lib/VideoApiAdapter.ts` |
| Nextcloud Talk | `nextcloudtalk_video` | `packages/app-store/nextcloudtalk/lib/VideoApiAdapter.ts` |
| Lyra | `lyra_video` | `packages/app-store/lyra/lib/VideoApiAdapter.ts` |
| Jelly | `jelly_video` | `packages/app-store/jelly/lib/VideoApiAdapter.ts` |
| Huddle01 | `huddle01_video` | `packages/app-store/huddle01video/lib/VideoApiAdapter.ts` |

## 5. 会议链接分发机制

### 5.1 分发渠道概览

会议链接通过以下渠道分发给组织者和参与者：

| 渠道 | 触发时机 | 接收者 | 内容 |
|------|----------|--------|------|
| 日历事件 | 预订创建/更新 | 所有参与者 | 会议链接在 `location` 或 `conferenceData` 中 |
| 确认邮件 | 预订确认后 | 组织者 + 参与者 | 邮件正文中包含会议链接 |
| 预订详情页面 | 访问预订页面 | 组织者 + 参与者（通过链接） | 页面显示会议信息 |
| Webhook 通知 | 预订事件触发 | 外部系统 | `location` 字段包含会议链接 |
| iCalendar 附件 | 邮件附件 | 所有参与者 | `.ics` 文件中包含会议信息 |

### 5.2 日历同步

**文件**: `packages/features/bookings/lib/EventManager.ts:364-369`
```typescript
// 克隆事件以避免被日历库修改
const clonedCalEvent = cloneDeep(event);
// 创建包含视频会议数据的日历事件
if (!skipCalendarEvent) {
  results.push(...(await this.createAllCalendarEvents(clonedCalEvent)));
}
```

**视频数据注入**:
在事件创建后，`videoCallData` 被注入到日历事件中：
**文件**: `packages/features/bookings/lib/EventManager.ts:345-358`
```typescript
if (result?.createdEvent) {
  evt.videoCallData = result.createdEvent;  // 包含 id, password, url, type
  evt.location = result.originalEvent.location;  // 可能是 URL 或类型标识
  result.type = result.createdEvent.type;
  // 更新 responses 中的位置数据（用于 webhook）
  if (evt.location && evt.responses) {
    evt.responses["location"] = {
      ...(evt.responses["location"] ?? {}),
      value: {
        optionValue: "",
        value: evt.location,
      },
    };
  }
}
```

### 5.3 位置更新服务（API v2）

在 API v2 中，有专门的服务处理预订位置的更新：

**文件**: `apps/api/v2/src/platform/bookings/2024-08-13/services/booking-location-integration.service.ts:66-110`

```typescript
async handleIntegrationLocationUpdate(
  existingBooking: BookingForLocationUpdate,
  inputLocation: { type: "integration"; integration: Integration_2024_08_13 },
  user: ApiAuthGuardUser,
  existingBookingHost: { organizationId: number | null } | null
): Promise<BookingLocationResponse> {
  // 根据 integration slug 选择处理方式
  switch (integrationSlug) {
    case "google-meet":
      return this.handleGoogleMeetLocation(ctx);
    case "office365-video":
      return this.handleMSTeamsLocation(ctx);
    case "cal-video":
      return this.handleCalVideoLocation(ctx);
    default:
      // 其他集成（Zoom, Webex 等）使用 VideoApiAdapter
      return this.handleVideoApiIntegration(ctx);
  }
}
```

**Google Meet 处理**:
**文件**: `apps/api/v2/src/platform/bookings/2024-08-13/services/booking-location-integration.service.ts:112-127`
```typescript
private async handleGoogleMeetLocation(ctx: IntegrationHandlerContext): Promise<BookingLocationResponse> {
  // 检查是否有 Google Calendar 连接
  const hasGoogleCalendar = ctx.booking.references.some(
    (ref) => ref.type.includes("google_calendar") && !ref.deleted
  );

  if (!hasGoogleCalendar) {
    // 没有 Google Calendar 时回退到 Cal Video
    return this.handleCalVideoLocation({
      ...ctx,
      integrationSlug: "cal-video",
      internalLocation: "integrations:daily",
    });
  }

  return this.handleCalendarBasedIntegration(ctx, "google_calendar");
}
```

**日历联动的核心逻辑**:
**文件**: `apps/api/v2/src/platform/bookings/2024-08-13/services/booking-location-integration.service.ts:245-311`
```typescript
private async handleCalendarBasedIntegration(
  ctx: IntegrationHandlerContext,
  requiredCalendarType: string
): Promise<BookingLocationResponse> {
  // 1. 查找现有的日历引用
  const calendarReference = ctx.booking.references.find(
    (ref) => ref.type.includes(requiredCalendarType) && !ref.deleted
  );

  // 2. 获取日历凭据
  const calendarCredential = await this.credentialService.getCredentialForReference(
    calendarReference,
    ctx.booking.user?.credentials || []
  );

  // 3. 构建事件（对于 Google Meet，添加 conferenceData.createRequest）
  const evt = await this.calendarSyncService.buildCalEventFromBookingData(
    ctx.booking,
    ctx.internalLocation,
    null
  );

  if (ctx.integrationSlug === "google-meet") {
    evt.conferenceData = {
      createRequest: {
        requestId: `${ctx.booking.uid}-meet`,
      },
    };
  }

  // 4. 更新日历事件（触发会议链接生成）
  const updateResult = await updateEvent(
    calendarCredential,
    evt,
    calendarReference.uid,
    calendarReference.externalCalendarId
  );

  // 5. 提取会议链接
  let meetingUrl: string | undefined;
  if (updateResult.updatedEvent) {
    const updatedEvent = Array.isArray(updateResult.updatedEvent)
      ? updateResult.updatedEvent[0]
      : updateResult.updatedEvent;
    // Google Meet: hangoutLink, Teams: url
    meetingUrl = updatedEvent?.hangoutLink || updatedEvent?.url;
  }

  // 6. 更新预订
  const updatedBooking = await this.bookingsRepository.updateBooking(ctx.existingBooking.uid, {
    location: meetingUrl || ctx.internalLocation,
    metadata: updatedMetadata as Prisma.InputJsonValue,
  });

  // 7. 同步到其他日历并发送通知
  await this.emitLocationChangeEvents(ctx, bookingLocation, evt);

  return this.bookingsService.getBooking(updatedBooking.uid, ctx.user);
}
```

### 5.4 Webhook 通知中的位置信息

**示例**: `BOOKING_CREATED` webhook payload
```json
{
  "triggerEvent": "BOOKING_CREATED",
  "payload": {
    "title": "30 Minute Meeting",
    "startTime": "2024-01-15T10:00:00.000Z",
    "endTime": "2024-01-15T10:30:00.000Z",
    "location": "https://cal.com/video/abc123",  // 会议链接
    "organizer": { ... },
    "attendees": [ ... ],
    "uid": "booking-uid-123",
    "metadata": {
      "videoCallUrl": "https://cal.com/video/abc123"  // 元数据中也有
    }
  }
}
```

### 5.5 元数据存储

会议链接同时存储在 `Booking.metadata` 中：

**文件**: `apps/api/v2/src/platform/bookings/2024-08-13/services/booking-location-integration.service.ts:336-344`
```typescript
private buildUpdatedMetadata(
  existingMetadata: unknown,
  videoCallUrl: string | undefined
): Record<string, unknown> {
  return {
    ...((existingMetadata || {}) as Record<string, unknown>),
    videoCallUrl,  // 在 metadata 中存储视频链接
  };
}
```

## 6. 错误处理与回退机制

### 6.1 视频会议创建失败回退

当首选视频会议 Provider 创建失败时，系统会自动回退到 Cal Video：

**文件**: `packages/features/conferencing/lib/videoClient.ts:86-100`
```typescript
try {
  createdMeeting = await firstVideoAdapter?.createMeeting(calEvent);
  returnObject = { ...returnObject, createdEvent: createdMeeting, success: true };
} catch (err) {
  // 发送集成失败邮件通知
  await sendBrokenIntegrationEmail(calEvent, "video");
  log.error("createMeeting failed", safeStringify(err), ...);
  
  // 回退到 Cal Video
  const defaultMeeting = await createMeetingWithCalVideo(calEvent);
  if (defaultMeeting) {
    calEvent.location = DailyLocationType;
  }
  returnObject = { ...returnObject, originalEvent: calEvent, createdEvent: defaultMeeting };
}
```

### 6.2 Google Meet 无日历连接回退

如前文所述，当选择 Google Meet 但没有连接 Google Calendar 时：

**文件**: `packages/features/bookings/lib/EventManager.ts:317-331`
```typescript
if (evt.location === MeetLocationType && mainHostDestinationCalendar?.integration !== "google_calendar") {
  // 检查是否有委托凭据（特殊情况）
  if (!isDelegationCredential({ credentialId: googleCalendarCredential?.id })) {
    evt["location"] = "integrations:daily";  // 切换到 Cal Video
    evt["conferenceCredentialId"] = undefined;
  }
}
```

### 6.3 凭据查找回退

当找不到指定的视频凭据时：

**文件**: `packages/features/bookings/lib/EventManager.ts:1036-1069`
```typescript
private getVideoCredentialByCalendarEvent(event: CalendarEvent): CredentialForCalendarService | undefined {
  if (!event.location) return undefined;

  const integrationName = event.location.replace("integrations:", "");
  
  // 1. 首选：使用 conferenceCredentialId
  if (event.conferenceCredentialId) {
    videoCredential = this.videoCredentials.find(
      (credential) => credential.id === event.conferenceCredentialId
    );
  } else {
    // 2. 次选：按类型查找最新凭据
    videoCredential = this.videoCredentials.find((credential) =>
      credential.type.includes(integrationName)
    );
  }

  // 3. 最终回退：Cal Video
  if (!videoCredential) {
    videoCredential = { ...FAKE_DAILY_CREDENTIAL };
  }

  return videoCredential;
}
```

## 7. 关键代码文件索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| 事件管理核心 | `packages/features/bookings/lib/EventManager.ts` | 全文 |
| 视频客户端 | `packages/features/conferencing/lib/videoClient.ts` | 29-103 (createMeeting) |
| Adapter 接口 | `packages/types/VideoApiAdapter.d.ts` | 12-52 |
| 动态加载 | `packages/app-store/getVideoAdapters.ts` | 11-45 |
| Cal Video | `packages/app-store/dailyvideo/lib/VideoApiAdapter.ts` | 239-519 |
| Zoom | `packages/app-store/zoomvideo/lib/VideoApiAdapter.ts` | 235-577 |
| Teams | `packages/app-store/office365video/lib/VideoApiAdapter.ts` | 40-319 |
| 位置定义 | `packages/app-store/locations.ts` | 全文 |
| API v2 位置更新 | `apps/api/v2/src/platform/bookings/2024-08-13/services/booking-location-integration.service.ts` | 66-311 |

## 8. 总结

### 8.1 架构设计亮点

1. **统一接口抽象**: `VideoApiAdapter` 接口使所有 Provider 遵循相同的操作模式
2. **动态加载**: `VideoApiAdapterMap` 和 `getVideoAdapters` 实现了 Provider 的即插即用
3. **多层回退**: 从凭据查找、Provider 选择到会议创建，均有完善的回退机制
4. **日历联动**: Google Meet 和 Teams 通过日历 API 间接创建会议，简化了集成复杂度
5. **OAuth 管理**: `OAuthManager` 统一处理令牌刷新和失效，各 Provider 无需重复实现

### 8.2 Provider 接入方式对比

| 维度 | Cal Video | Zoom | Google Meet | Microsoft Teams |
|------|-----------|------|-------------|-----------------|
| 用户配置 | 无需 | 需要 OAuth | 需要日历连接 | 日历或 OAuth |
| API 调用 | 直接 | 直接 | 间接（日历） | 混合 |
| 链接生成时机 | 预订时 | 预订时 | 日历事件创建时 | 预订时或日历创建时 |
| 回退支持 | 作为回退目标 | 回退到 Cal Video | 回退到 Cal Video | 回退到 Cal Video |
| 令牌管理 | 全局 API Key | OAuthManager | 日历 OAuth | OAuthManager |

### 8.3 会议链接生命周期

```
1. 预订创建
      ↓
2. EventManager 检测位置类型
      ↓
3. 获取对应 Provider 的 VideoApiAdapter
      ↓
4. 调用 Provider API 创建会议
      ↓
5. 生成 VideoCallData { id, password, url, type }
      ↓
6. 存储到 BookingReference (meetingUrl, meetingId, meetingPassword)
      ↓
7. 同步到：
   - 日历事件（location/conferenceData）
   - 预订 metadata (videoCallUrl)
   - Webhook 通知
   - 确认邮件
      ↓
8. 参与者通过以下方式访问：
   - 日历事件中的链接
   - 邮件中的链接
   - 预订详情页面
```

---

*文档生成时间: 2026-05-04*
*基于代码提交: 当前工作目录状态*
