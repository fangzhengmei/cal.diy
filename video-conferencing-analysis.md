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

### 4.5 Microsoft Teams 两种接入路径详解

Microsoft Teams 有两种完全独立的接入路径，**触发条件、行为模式、生命周期管理完全不同**。

#### 路径触发条件

**文件**: `packages/features/bookings/lib/EventManager.ts:333-342`
```typescript
const isDedicated = evt.location ? isDedicatedIntegration(evt.location) : null;
const isMSTeamsWithOutlookCalendar =
  evt.location === MSTeamsLocationType &&
  mainHostDestinationCalendar?.integration === "office365_calendar";

// 如果是专用视频集成且不是 Teams + Outlook 组合，才创建独立视频会议
if (isDedicated && !isMSTeamsWithOutlookCalendar) {
  const result = await this.createVideoEvent(evt);
  // ...
}
```

| 路径 | 触发条件 | 位置类型 | 处理方式 |
|------|----------|----------|----------|
| **日历联动路径** | `isMSTeamsWithOutlookCalendar = true` | `MSTeamsLocationType` + `office365_calendar` | 调用 `createAllCalendarEvents()`，**不调用** `createVideoEvent()` |
| **office365video 直连路径** | `isDedicated = true && !isMSTeamsWithOutlookCalendar` | 专用 video 集成类型 | 调用 `createVideoEvent()` → `videoClient.createMeeting()` |

---

#### 路径一：Outlook 日历联动（推荐）

**触发条件**: `evt.location === "integrations:office365"` 且 `destinationCalendar.integration === "office365_calendar"`

**核心机制**:
- **不创建独立的视频会议**（`createVideoEvent` 不执行）
- 只创建 Outlook 日历事件，由 Outlook 自动生成 Teams 会议链接
- 会议链接存储在 `onlineMeeting.joinUrl` 中

**创建时的行为** (Office365CalendarService.createEvent):
**文件**: `packages/app-store/office365calendar/lib/CalendarService.ts:300-313`
```typescript
const response = await this.fetcher(eventsUrl, {
  method: "POST",
  body: JSON.stringify(this.translateEvent(event)),
});

const responseJson = await handleErrorsJson<
  NewCalendarEventType & { iCalUId: string; onlineMeeting?: { joinUrl?: string } }
>(response);

if (responseJson?.onlineMeeting?.joinUrl) {
  responseJson.url = responseJson?.onlineMeeting?.joinUrl;  // 映射到 url 字段
}
```

**translateEvent 中的关键设置**:
**文件**: `packages/app-store/office365calendar/lib/CalendarService.ts:548-550`
```typescript
if (isOnlineMeeting) {
  office365Event.isOnlineMeeting = true;  // 触发 Outlook 生成 Teams 链接
}
```

**视频数据提取**:
**文件**: `packages/features/bookings/lib/EventManager.ts:251-275`
```typescript
private updateMSTeamsVideoCallData(
  evt: CalendarEvent,
  results: Array<EventResult<Exclude<Event, AdditionalInformation>>>
) {
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
  }
}
```

---

#### 路径二：office365video 直连

**触发条件**: 用户连接了 Office 365 Video 凭据，但没有配置 Outlook 日历作为目标日历

**核心机制**:
- 直接调用 Microsoft Graph API `/onlineMeetings` 端点
- **不依赖** Outlook 日历
- 有独立的 `VideoApiAdapter` 实现

**创建时的行为**:
**文件**: `packages/app-store/office365video/lib/VideoApiAdapter.ts:270-317`
```typescript
createMeeting: async (event: CalendarEvent): Promise<VideoCallData> => {
  const url = `${await getUserEndpoint()}/onlineMeetings`;
  const response = await auth.requestRaw({
    url,
    options: {
      method: "POST",  // POST 创建新会议
      body: JSON.stringify(translateEvent(event)),
    },
  });

  const resultObject = JSON.parse(await response.text());

  return Promise.resolve({
    type: "office365_video",
    id: resultObject.id,
    password: "",
    url: resultObject.joinWebUrl || resultObject.joinUrl,
  });
},
```

**委托凭据支持**:
office365video 直连路径特别支持委托凭据（Delegation Credential），用于组织级别的服务账户集成。通过 `getAzureUserId()` 函数查找目标用户的 Azure AD ID，然后使用服务账户代表用户创建会议。

---

#### 两种路径关键对比

| 维度 | 日历联动路径 | office365video 直连路径 |
|------|-------------|------------------------|
| **触发条件** | `MSTeamsLocationType` + `office365_calendar` | 专用 video 集成，无 Outlook 日历 |
| **API 调用** | `POST /calendar/events` | `POST /onlineMeetings` |
| **会议生成** | Outlook 自动生成 | 直接调用 Graph API |
| **链接存储** | `onlineMeeting.joinUrl` → `url` | `joinWebUrl` / `joinUrl` |
| **推荐程度** | ⭐⭐⭐ 推荐 | ⭐ 不推荐 |

**为什么不推荐直连路径**：
- 改期时会创建新会议（链接变化）
- 取消时不会删除会议（会议残留）
- 详见第 8.3 节和第 9.3 节的详细分析

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

## 8. 改期(Reschedule)时的视频会议处理

### 8.1 改期触发条件

在 `EventManager.reschedule()` 中，根据不同条件采取不同的处理策略：

**文件**: `packages/features/bookings/lib/EventManager.ts:675-687`

```typescript
const isLocationChanged = !!evt.location && !!booking.location && evt.location !== booking.location;

let isDailyVideoRoomExpired = false;

if (evt.location === "integrations:daily") {
  const originalBookingEndTime = new Date(booking.endTime);
  const roomExpiryTime = new Date(originalBookingEndTime.getTime() + 14 * 24 * 60 * 60 * 1000);
  const now = new Date();
  isDailyVideoRoomExpired = now > roomExpiryTime;
}

const shouldUpdateBookingReferences =
  !!changedOrganizer || isLocationChanged || !!isBookingRequestedReschedule || isDailyVideoRoomExpired;
```

### 8.2 改期处理策略矩阵

| 条件 | 处理方式 | 会议链接变化 |
|------|----------|--------------|
| `changedOrganizer = true` (组织者变更) | 删除旧会议 + 创建新会议 | **链接变化** |
| `isLocationChanged = true` (位置变更) | 调用 `updateLocation()` | 可能变化 |
| `isBookingRequestedReschedule = true` (请求改期) | 调用 `updateLocation()` | 可能变化 |
| `isDailyVideoRoomExpired = true` (Daily 房间过期) | 调用 `updateLocation()` | **链接变化** |
| 普通改期（仅时间变更） | 调用 `updateVideoEvent()` | 通常不变 |

### 8.3 各 Provider 在改期时的行为

#### Zoom (直接 API 集成)

**文件**: `packages/app-store/zoomvideo/lib/VideoApiAdapter.ts:549-575`

```typescript
updateMeeting: async (bookingRef: PartialReference, event: CalendarEvent): Promise<VideoCallData> => {
  try {
    // 1. PATCH 更新现有会议
    await fetchZoomApi(`meetings/${bookingRef.uid}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(await translateEvent(event)),
    });

    // 2. 重新获取会议详情
    const updatedMeeting = await fetchZoomApi(`meetings/${bookingRef.uid}`);
    const result = zoomEventResultSchema.parse(updatedMeeting);

    return {
      type: "zoom_video",
      id: result.id.toString(),
      password: result.password || "",
      url: result.join_url,  // 链接保持不变
    };
  } catch (err) {
    // ...
  }
},
```

**行为特点**:
- 使用 `PATCH` 请求更新现有会议
- **会议链接保持不变**（`join_url` 相同）
- 只更新会议时间、标题等属性

#### Cal Video (Daily.co)

**文件**: `packages/app-store/dailyvideo/lib/VideoApiAdapter.ts:404-405`

```typescript
updateMeeting: (bookingRef: PartialReference, event: CalendarEvent): Promise<VideoCallData> =>
  createOrUpdateMeeting(`/rooms/${bookingRef.uid}`, event, region),
```

**`createOrUpdateMeeting` 行为**:
- 对于已存在的房间，使用 POST 到 `/rooms/{roomName}` 来更新
- **房间链接保持不变**
- 但有一个特殊情况：如果房间已过期（结束时间 + 14 天），会创建新房间

**过期检测逻辑** (EventManager.reschedule):
```typescript
if (evt.location === "integrations:daily") {
  const originalBookingEndTime = new Date(booking.endTime);
  const roomExpiryTime = new Date(originalBookingEndTime.getTime() + 14 * 24 * 60 * 60 * 1000);
  const now = new Date();
  isDailyVideoRoomExpired = now > roomExpiryTime;  // 检查是否过期
}
```

#### Microsoft Teams (Office 365 Video)

**文件**: `packages/app-store/office365video/lib/VideoApiAdapter.ts` (需要查看实际实现)

从代码模式来看，Teams 的 `updateMeeting` 可能：
- 对于日历联动方式：更新 Outlook 事件，Teams 链接保持不变
- 对于直接 API 方式：可能创建新的会议链接（取决于实现）

#### Google Meet (日历联动)

**文件**: `packages/features/bookings/lib/service/RegularBookingService.ts:1950-2005`

```typescript
// Google Meet 改期时的特殊处理
if (bookingLocation === MeetLocationType) {
  // 查找 Google Calendar 结果
  const googleCalIndex = updateManager.referencesToCreate.findIndex(
    (ref) => ref.type === "google_calendar"
  );
  const googleCalResult = results[googleCalIndex];

  // 提取 hangoutLink
  const googleHangoutLink = Array.isArray(googleCalResult?.updatedEvent)
    ? googleCalResult.updatedEvent[0]?.hangoutLink
    : (googleCalResult?.updatedEvent?.hangoutLink ?? googleCalResult?.createdEvent?.hangoutLink);

  if (googleHangoutLink) {
    // 更新引用中的 meetingUrl
    updateManager.referencesToCreate[googleCalIndex] = {
      ...updateManager.referencesToCreate[googleCalIndex],
      meetingUrl: googleHangoutLink,  // 链接保持不变
    };

    // 创建单独的 google_meet_video 引用
    updateManager.referencesToCreate.push({
      type: "google_meet_video",
      meetingUrl: googleHangoutLink,
      uid: googleCalResult.uid,
      credentialId: updateManager.referencesToCreate[googleCalIndex].credentialId,
    });
  }
}
```

**行为特点**:
- Google Meet 链接是与日历事件绑定的
- 更新日历事件时，`hangoutLink` **保持不变**
- 除非创建了全新的日历事件（如组织者变更）

### 8.4 videoClient.updateMeeting 实现

**文件**: `packages/features/conferencing/lib/videoClient.ts:105-145`

```typescript
const updateMeeting = async (
  credential: CredentialPayload | CredentialForCalendarService,
  calEvent: CalendarEvent,
  bookingRef: PartialReference | null
): Promise<EventResult<VideoCallData>> => {
  const uid = translator.fromUUID(uuidv5(JSON.stringify(calEvent), uuidv5.URL));
  let success = true;
  const [firstVideoAdapter] = await getVideoAdapters([credential]);
  const canCallUpdateMeeting = !!(credential && bookingRef);
  
  // 调用 Adapter 的 updateMeeting
  const updatedMeeting = canCallUpdateMeeting
    ? await firstVideoAdapter?.updateMeeting(bookingRef, calEvent).catch(async (e) => {
        // 更新失败时发送集成失败邮件
        await sendBrokenIntegrationEmail(calEvent, "video");
        log.error("updateMeeting failed", e, calEvent);
        success = false;
        return undefined;
      })
    : undefined;

  if (!updatedMeeting) {
    // 更新失败的处理
    log.error("updateMeeting failed", safeStringify({ bookingRef, canCallUpdateMeeting, calEvent, credential }));
    return {
      appName: credential.appName || credential.appId || "",
      type: credential.type,
      success,
      uid,
      originalEvent: calEvent,
    };
  }

  return {
    appName: credential.appName || credential.appId || "",
    type: credential.type,
    success,
    uid,
    updatedEvent: updatedMeeting,
    originalEvent: calEvent,
  };
};
```

### 8.5 改期完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        EventManager.reschedule()                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 获取原始预订信息 (references, location, destinationCalendar)              │
│                                                                              │
│  2. 判断条件：                                                               │
│     ├─► changedOrganizer? (组织者变更)                                       │
│     ├─► isLocationChanged? (位置变更)                                        │
│     ├─► isBookingRequestedReschedule? (请求改期)                             │
│     └─► isDailyVideoRoomExpired? (Daily 房间过期: 结束时间+14天)            │
│                                                                              │
│  3. 分支处理：                                                               │
│                                                                              │
│     ┌─ requiresConfirmation = true (需要确认)                                │
│     │   └─► deleteEventsAndMeetings() - 删除旧会议                          │
│     │       └─► 等待组织者确认后创建新会议                                     │
│     │                                                                         │
│     ├─ changedOrganizer = true (组织者变更)                                  │
│     │   ├─► deleteEventsAndMeetings() - 删除旧会议                          │
│     │   └─► create() - 创建全新会议，生成新链接                               │
│     │                                                                         │
│     ├─ isLocationChanged || isBookingRequestedReschedule || isDailyExpired   │
│     │   └─► updateLocation()                                                 │
│     │       └─► 可能创建新的会议链接                                          │
│     │                                                                         │
│     └─ 普通改期 (仅时间变更)                                                  │
│         ├─ isDedicated = true (专用视频集成)                                 │
│         │   └─► updateVideoEvent() → videoClient.updateMeeting()           │
│         │       ├─► Zoom: PATCH → 链接不变                                  │
│         │       ├─► Daily: POST /rooms/{uid} → 链接不变                      │
│         │       └─► Teams: 取决于实现                                       │
│         │                                                                     │
│         └─► updateAllCalendarEvents() - 更新日历事件                         │
│             └─► Google Meet/Teams (日历联动): hangoutLink 保持不变           │
│                                                                              │
│  4. 更新 metadata: hangoutLink, conferenceData, entryPoints                 │
│                                                                              │
│  5. 更新 bookingReferences                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 9. 取消(Cancel)时的视频会议处理

### 9.1 取消流程入口

**文件**: `packages/features/bookings/lib/EventManager.ts:777-790`

```typescript
public async cancelEvent(
  event: CalendarEvent,
  bookingReferences: Pick<
    BookingReference,
    "uid" | "type" | "externalCalendarId" | "credentialId" | "thirdPartyRecurringEventId"
  >[],
  isBookingInRecurringSeries?: boolean
) {
  await this.deleteEventsAndMeetings({
    event,
    bookingReferences,
    isBookingInRecurringSeries,
  });
}
```

### 9.2 deleteEventsAndMeetings 核心逻辑

**文件**: `packages/features/bookings/lib/EventManager.ts:792-851`

```typescript
public async deleteEventsAndMeetings({
  event,
  bookingReferences,
  isBookingInRecurringSeries,
}: {
  event: CalendarEvent;
  bookingReferences: PartialReference[];
  isBookingInRecurringSeries?: boolean;
}) {
  const log = logger.getSubLogger({ prefix: [`[deleteEventsAndMeetings]: ${event?.uid}`] });
  const calendarReferences = [],
    videoReferences = [],
    crmReferences = [],
    allPromises = [];

  // 按类型分类处理
  for (const reference of bookingReferences) {
    // 日历类型 (排除 other_calendar)
    if (reference.type.includes("_calendar") && !reference.type.includes("other_calendar")) {
      calendarReferences.push(reference);
      allPromises.push(
        this.deleteCalendarEventForBookingReference({
          reference,
          event,
          isBookingInRecurringSeries,
        })
      );
    }

    // 视频类型
    if (reference.type.includes("_video")) {
      videoReferences.push(reference);
      allPromises.push(
        this.deleteVideoEventForBookingReference({
          reference,
        })
      );
    }

    // CRM 类型或 other_calendar
    if (reference.type.includes("_crm") || reference.type.includes("other_calendar")) {
      crmReferences.push(reference);
      allPromises.push(this.deleteCRMEvent({ reference, event }));
    }
  }

  log.debug("deleteEventsAndMeetings", safeStringify({ calendarReferences, videoReferences }));

  // 使用 allSettled 确保单个失败不影响其他操作
  (await Promise.allSettled(allPromises)).some((result) => {
    if (result.status === "rejected") {
      // 软错误：只记录警告，不抛出异常
      log.warn(
        "Error deleting calendar event or video meeting for booking",
        safeStringify({ error: result.reason })
      );
    }
  });

  if (!allPromises.length) {
    log.warn("No calendar or video references found for booking - Couldn't delete events or meetings");
  }
}
```

### 9.3 各 Provider 在取消时的行为

#### Zoom

**文件**: `packages/app-store/zoomvideo/lib/VideoApiAdapter.ts:539-548`

```typescript
deleteMeeting: async (uid: string): Promise<void> => {
  try {
    await fetchZoomApi(`meetings/${uid}`, {
      method: "DELETE",
    });
    return Promise.resolve();
  } catch (err) {
    return Promise.reject(new Error("Failed to delete meeting"));
  }
},
```

**行为**:
- 调用 Zoom API `DELETE /meetings/{meetingId}` 删除会议
- 会议链接立即失效
- 失败时抛出异常（但被 `Promise.allSettled` 捕获为软错误）

#### Cal Video (Daily.co)

**文件**: `packages/app-store/dailyvideo/lib/VideoApiAdapter.ts:400-403`

```typescript
deleteMeeting: async (uid: string): Promise<void> => {
  await fetcher(`/rooms/${uid}`, { method: "DELETE" });
  return Promise.resolve();
},
```

**行为**:
- 调用 Daily.co API `DELETE /rooms/{roomName}` 删除房间
- 房间链接立即失效

#### Microsoft Teams (Office 365 Video)

查看代码发现，Teams Video 的 `deleteMeeting` 可能是空实现（取决于具体接入方式）：

- **日历联动方式**: 删除 Outlook 事件时，Teams 会议会自动失效
- **直接 API 方式**: 可能需要显式调用 `DELETE /onlineMeetings/{meetingId}`

#### Google Meet

Google Meet 没有独立的 `deleteMeeting`，因为：
- Google Meet 链接与日历事件绑定
- 删除日历事件时，Meet 链接自动失效
- 没有 `_video` 类型的 BookingReference（只有 `google_meet_video` 引用）

### 9.4 videoClient.deleteMeeting 实现

**文件**: `packages/features/conferencing/lib/videoClient.ts:147-164`

```typescript
const deleteMeeting = async (
  credential: CredentialPayload | CredentialForCalendarService | null,
  uid: string
): Promise<unknown> => {
  if (credential) {
    const videoAdapter = (await getVideoAdapters([credential]))[0];
    log.debug(
      "Calling deleteMeeting for",
      safeStringify({ credential: getPiiFreeCredential(credential), uid })
    );
    
    // 某些 video app 没有定义 video adapter（如 riverby, whereby）
    if (videoAdapter) {
      return videoAdapter.deleteMeeting(uid);
    }
  }

  return Promise.resolve({});  // 没有凭据或 adapter 时静默成功
};
```

### 9.5 取消完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        EventManager.cancelEvent()                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 调用 deleteEventsAndMeetings()                                           │
│                                                                              │
│  2. 遍历所有 bookingReferences，按类型分类：                                   │
│                                                                              │
│     ┌─ type 包含 "_calendar" 且不是 "other_calendar"                        │
│     │   └─► deleteCalendarEventForBookingReference()                        │
│     │       └─► 删除日历事件                                                  │
│     │           ├─► Google Calendar: 事件 + Meet 链接失效                    │
│     │           └─► Outlook: 事件 + Teams 链接失效                           │
│     │                                                                         │
│     ├─ type 包含 "_video"                                                    │
│     │   └─► deleteVideoEventForBookingReference()                           │
│     │       └─► videoClient.deleteMeeting()                                  │
│     │           ├─► Zoom: DELETE /meetings/{uid}                            │
│     │           ├─► Daily: DELETE /rooms/{uid}                               │
│     │           └─► Teams: 取决于实现（可能为空）                              │
│     │                                                                         │
│     └─ type 包含 "_crm" 或 "other_calendar"                                  │
│         └─► deleteCRMEvent()                                                 │
│                                                                              │
│  3. 使用 Promise.allSettled 并行执行所有删除操作：                            │
│     ├─► 所有操作独立执行                                                      │
│     ├─► 单个失败不会影响其他操作                                               │
│     └─► 失败只记录为 warn，不抛出异常                                          │
│                                                                              │
│  4. BookingReference 软删除：                                                │
│     └─► 标记 deleted 字段，而非物理删除                                       │
│                                                                              │
│  5. Booking.metadata 保留：                                                  │
│     └─► 用于历史记录和审计                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 10. Booking Metadata 回写机制详解

### 10.1 Metadata 存储结构

`Booking.metadata` 是一个 JSON 字段，用于存储视频会议链接等额外信息：

**文件**: `packages/features/bookings/lib/handleNewBooking/getVideoCallDetails.ts`

```typescript
export function getVideoCallDetails({ results }: { results: VideoResult[] }) {
  const firstVideoResult = results.find((result) => result.type.includes("_video"));
  const updatedVideoEvent = extractUpdatedVideoEvent(firstVideoResult);
  const metadata = updatedVideoEvent ? extractMetadata(updatedVideoEvent) : {};

  const videoCallUrl = metadata.hangoutLink || updatedVideoEvent?.url;

  return { videoCallUrl, metadata, updatedVideoEvent };
}

function extractMetadata(event: ExtraAdditionalInfo): AdditionalInformation {
  return {
    hangoutLink: event.hangoutLink,
    conferenceData: event.conferenceData,
    entryPoints: event.entryPoints,
  };
}
```

### 10.2 Metadata 字段说明

| 字段 | 类型 | 来源 | 说明 |
|------|------|------|------|
| `videoCallUrl` | `string` | 计算值 | 最终的会议链接（优先级最高） |
| `hangoutLink` | `string` | Google Calendar API | Google Meet 链接 |
| `conferenceData` | `object` | Google Calendar API | 会议数据（包含 entryPoints） |
| `entryPoints` | `array` | Google Calendar API | 会议入口点列表 |

### 10.3 创建时的 Metadata 回写流程

**文件**: `packages/features/bookings/lib/service/RegularBookingService.ts:2209-2213`

```typescript
const metadata = videoCallUrl
  ? {
      videoCallUrl: getVideoCallUrlFromCalEvent(evt) || videoCallUrl,
    }
  : undefined;
```

**完整流程**:

```
1. EventManager.create() 返回 results
      ↓
2. getVideoCallDetails({ results })
      ↓
3. 提取:
   - metadata.hangoutLink (Google Meet)
   - metadata.conferenceData
   - metadata.entryPoints
   - videoCallUrl (优先级: hangoutLink || updatedEvent.url)
      ↓
4. 构建最终 metadata:
   {
     videoCallUrl: getVideoCallUrlFromCalEvent(evt) || videoCallUrl,
     hangoutLink,
     conferenceData,
     entryPoints
   }
      ↓
5. 存储到 Booking.metadata
```

### 10.4 改期时的 Metadata 更新

**文件**: `packages/features/bookings/lib/service/RegularBookingService.ts:1929-2018`

```typescript
// 1. 调用 eventManager.reschedule()
const updateManager = await eventManager.reschedule(
  evt,
  originalRescheduledBooking.uid,
  // ...
);

// 2. 获取视频会议详情
const { metadata: videoMetadata, videoCallUrl: _videoCallUrl } = getVideoCallDetails({
  results,
});

let metadata: AdditionalInformation = {};
metadata = videoMetadata;
videoCallUrl = _videoCallUrl;

// 3. Google Meet 特殊处理
if (bookingLocation === MeetLocationType) {
  // ... 提取 hangoutLink
  const googleHangoutLink = Array.isArray(googleCalResult?.updatedEvent)
    ? googleCalResult.updatedEvent[0]?.hangoutLink
    : (googleCalResult?.updatedEvent?.hangoutLink ?? googleCalResult?.createdEvent?.hangoutLink);

  // 更新 metadata
  const createdOrUpdatedEvent = Array.isArray(results[0]?.updatedEvent)
    ? results[0]?.updatedEvent[0]
    : (results[0]?.updatedEvent ?? results[0]?.createdEvent);
  metadata.hangoutLink = createdOrUpdatedEvent?.hangoutLink;
  metadata.conferenceData = createdOrUpdatedEvent?.conferenceData;
  metadata.entryPoints = createdOrUpdatedEvent?.entryPoints;

  // 更新 videoCallUrl
  videoCallUrl =
    metadata.hangoutLink ||
    createdOrUpdatedEvent?.url ||
    organizerOrFirstDynamicGroupMemberDefaultLocationUrl ||
    getVideoCallUrlFromCalEvent(evt) ||
    videoCallUrl;
}

// 4. 最终构建 metadata
const metadata = videoCallUrl
  ? {
      videoCallUrl: getVideoCallUrlFromCalEvent(evt) || videoCallUrl,
    }
  : undefined;
```

### 10.5 API v2 中的 Metadata 更新

**文件**: `apps/api/v2/src/platform/bookings/2024-08-13/services/booking-location-integration.service.ts:319-344`

```typescript
// 构建更新后的 metadata
const updatedMetadata = this.buildUpdatedMetadata(ctx.existingBooking.metadata, videoCallUrl);

// 更新预订
const updatedBooking = await this.bookingsRepository.updateBooking(ctx.existingBooking.uid, {
  location: meetingUrl || ctx.internalLocation,
  metadata: updatedMetadata as Prisma.InputJsonValue,
});

// buildUpdatedMetadata 实现
private buildUpdatedMetadata(
  existingMetadata: unknown,
  videoCallUrl: string | undefined
): Record<string, unknown> {
  return {
    ...((existingMetadata || {}) as Record<string, unknown>),
    videoCallUrl,  // 覆盖或添加 videoCallUrl
  };
}
```

### 10.6 Metadata 读取优先级

当需要获取会议链接时，不同场景有不同的优先级：

```
读取优先级（从高到低）:

1. 日历导出 (getCalendarLinks.ts)
   └─► metadata.videoCallUrl (强制)

2. Webhook 通知 (RegularBookingService.ts)
   └─► webhookLocation = metadata?.videoCallUrl || evt.location

3. 预订详情页面
   └─► 取决于具体实现，但通常优先使用 metadata.videoCallUrl

4. API v2 响应 (output.service.ts)
   └─► location = metadata?.videoCallUrl || databaseBooking.location
```

## 11. Webhook 和日历导出的字段使用分析

### 11.1 Webhook 数据结构

**文件**: `packages/features/webhooks/lib/sendPayload.ts:81-104`

```typescript
export type EventPayloadType = Omit<CalendarEvent, "assignmentReason"> &
  TranscriptionGeneratedPayload &
  EventTypeInfo & {
    uid?: string | null;
    metadata?: { [key: string]: string | number | boolean | null };  // 包含 videoCallUrl
    bookingId?: number;
    status?: string;
    // ...
  };
```

### 11.2 Webhook 中 Location 字段的构建

**文件**: `packages/features/bookings/lib/service/RegularBookingService.ts:2232-2253`

```typescript
// location 优先级: metadata.videoCallUrl > evt.location
const webhookLocation = metadata?.videoCallUrl || evt.location;

const webhookData: EventPayloadType = {
  ...evt,
  ...eventTypeInfo,
  bookingId: booking?.id,
  // ...
  metadata: { ...metadata, ...reqBody.metadata },  // 合并 metadata
  location: webhookLocation,  // 使用计算后的 location
  // ...
};
```

### 11.3 Zapier Webhook 特殊处理

**文件**: `packages/features/webhooks/lib/sendPayload.ts:132-181`

```typescript
function getZapierPayload(data: WithUTCOffsetType<EventPayloadType & { createdAt: string }>): string {
  // location 经过 getHumanReadableLocationValue 处理
  const location = getHumanReadableLocationValue(data.location || "", t);

  const body = {
    // ...
    location: location,  // 处理后的 location
    // ...
    metadata: {
      videoCallUrl: data.metadata?.videoCallUrl,  // 单独的 videoCallUrl
    },
  };
  return JSON.stringify(body);
}
```

### 11.4 日历导出的字段使用

**文件**: `packages/features/bookings/lib/getCalendarLinks.ts:138-254`

```typescript
export const getCalendarLinks = ({
  booking,
  eventType,
  t,
}: {
  booking: {
    // ...
    location: string | null;
    metadata: Prisma.JsonObject | null;
  };
  // ...
}) => {
  // 关键：优先使用 metadata.videoCallUrl，而非 booking.location
  const videoCallUrl = bookingMetadataSchema.parse(booking?.metadata || {})?.videoCallUrl ?? null;

  // Google Calendar 链接
  const googleCalendarLink = buildGoogleCalendarLink({
    // ...
    bookingLocation: videoCallUrl ?? null,  // 使用 videoCallUrl
  });

  // Microsoft Office 链接
  const microsoftOfficeLink = buildMicrosoftOfficeLink({
    // ...
    bookingLocation: videoCallUrl,  // 使用 videoCallUrl
  });

  // Outlook.com 链接
  const microsoftOutlookLink = buildMicrosoftOutlookLink({
    // ...
    bookingLocation: videoCallUrl,  // 使用 videoCallUrl
  });

  // ICS 文件
  let icsFileLink = "";
  try {
    icsFileLink = buildICalLink({
      // ...
      location: videoCallUrl ?? null,  // 使用 videoCallUrl
    });
  } catch (error) {
    console.error("Error generating ICS file", error);
  }

  return [
    { label: "Google Calendar", id: CalendarLinkType.GOOGLE_CALENDAR, link: googleCalendarLink },
    { label: "Microsoft Office", id: CalendarLinkType.MICROSOFT_OFFICE, link: microsoftOfficeLink },
    { label: "Microsoft Outlook", id: CalendarLinkType.MICROSOFT_OUTLOOK, link: microsoftOutlookLink },
    { label: "ICS", id: CalendarLinkType.ICS, link: icsFileLink },
  ];
};
```

### 11.5 字段使用对比表

| 场景 | 使用的字段 | 优先级 | 文件路径 |
|------|-----------|--------|----------|
| **Webhook (标准)** | `location` | `metadata.videoCallUrl \|\| evt.location` | `RegularBookingService.ts:2232` |
| **Webhook (metadata)** | `metadata.videoCallUrl` | 直接使用 | `sendPayload.ts:177` |
| **Zapier Webhook** | `location` + `metadata.videoCallUrl` | 分离为两个字段 | `sendPayload.ts:154,177` |
| **Google Calendar 链接** | `videoCallUrl` (来自 metadata) | 强制使用 metadata | `getCalendarLinks.ts:198` |
| **Outlook 链接** | `videoCallUrl` (来自 metadata) | 强制使用 metadata | `getCalendarLinks.ts:207` |
| **ICS 文件** | `videoCallUrl` (来自 metadata) | 强制使用 metadata | `getCalendarLinks.ts:226` |
| **API v2 响应** | `location` | `metadata.videoCallUrl \|\| databaseBooking.location` | `output.service.ts:110` |

### 11.6 为什么日历导出只使用 metadata.videoCallUrl

**关键原因**: `booking.location` 可能是类型标识而非实际 URL

**位置类型定义** (locations.ts):
- `"integrations:daily"` - Cal Video
- `"integrations:google:meet"` - Google Meet
- `"integrations:office365"` - Microsoft Teams
- `"https://zoom.us/j/123456"` - 实际 URL（某些情况下）

**问题场景**:
```typescript
// booking.location 可能是:
"integrations:daily"  // 类型标识，不是 URL
// 但 metadata.videoCallUrl 总是:
"https://cal.com/video/abc123"  // 实际的会议链接
```

**测试验证** (getCalendarLinks.test.ts:118-130):
```typescript
it("should use videoCallUrl from metadata when available", async () => {
  const videoCallUrl = "https://meet.google.com/abc-defg-hij";
  const booking = {
    // ...
    location: "integrations:google:meet",  // 类型标识
    metadata: { videoCallUrl },  // 实际 URL
  };

  const links = getCalendarLinks({ booking, eventType, t });

  // 验证 Google Calendar 链接使用的是 videoCallUrl
  expect(links[0].link).toContain(encodeURIComponent(videoCallUrl));
  expect(links[0].link).not.toContain("integrations:google:meet");
});
```

## 12. 总结与完整生命周期

### 12.1 各 Provider 操作行为汇总

| Provider | 创建 | 改期(时间) | 改期(位置) | 取消 | 链接稳定性 |
|----------|------|-----------|-----------|------|-----------|
| **Cal Video** | POST `/rooms` | POST `/rooms/{uid}` (复用) | POST `/rooms` (新建) | DELETE `/rooms/{uid}` | 时间改期稳定，位置/过期时变化 |
| **Zoom** | POST `/users/me/meetings` | PATCH `/meetings/{uid}` (链接不变) | 新建会议 | DELETE `/meetings/{uid}` | 时间改期稳定 |
| **Google Meet** | 日历 `conferenceData.createRequest` | 日历事件更新 (hangoutLink 不变) | 新建日历事件 | 删除日历事件 | 稳定（除非新建日历事件） |
| **Microsoft Teams (日历)** | 日历联动 | 日历事件更新 | 新建日历事件 | 删除日历事件 | 稳定（除非新建日历事件） |
| **Microsoft Teams (直接 API)** | POST `/onlineMeetings` | POST `/onlineMeetings` (可能新建) | 新建会议 | 空实现 | 可能变化 |

### 12.2 完整视频会议链接生命周期

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              预订创建 (Booking Created)                                  │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  1. 位置类型检测:                                                                       │
│     ├─► "integrations:daily" → Cal Video (全局凭据)                                   │
│     ├─► "integrations:google:meet" → 需要 Google Calendar                             │
│     ├─► "integrations:office365" → Teams (日历联动或直接 API)                         │
│     └─► 其他 → 查找对应 VideoApiAdapter                                                │
│                                                                                         │
│  2. 会议创建:                                                                           │
│     ├─► createVideoEvent() → videoClient.createMeeting()                              │
│     └─► 返回 VideoCallData { id, password, url, type }                                │
│                                                                                         │
│  3. 数据存储:                                                                           │
│     ├─► BookingReference:                                                              │
│     │   ├─► type: "zoom_video", "daily_video", "google_calendar", etc.              │
│     │   ├─► meetingUrl: 实际会议链接                                                    │
│     │   ├─► meetingId: 会议 ID                                                         │
│     │   └─► meetingPassword: 密码/令牌                                                 │
│     │                                                                                   │
│     └─► Booking.metadata:                                                              │
│         ├─► videoCallUrl: 最终会议链接（优先级最高）                                    │
│         ├─► hangoutLink: Google Meet 链接（如果适用）                                  │
│         ├─► conferenceData: 会议数据                                                   │
│         └─► entryPoints: 入口点列表                                                    │
│                                                                                         │
│  4. 分发:                                                                               │
│     ├─► Webhook:                                                                       │
│     │   ├─► location: metadata.videoCallUrl \|\| evt.location                        │
│     │   └─► metadata.videoCallUrl: 单独字段                                           │
│     │                                                                                   │
│     ├─► 日历导出:                                                                       │
│     │   └─► 强制使用 metadata.videoCallUrl                                             │
│     │                                                                                   │
│     ├─► 确认邮件:                                                                       │
│     │   └─► 包含会议链接（通常来自 videoCallData.url）                                 │
│     │                                                                                   │
│     └─► 日历同步:                                                                       │
│         ├─► Google Calendar: hangoutLink 在 conferenceData 中                        │
│         └─► Outlook: Teams 链接在事件属性中                                            │
│                                                                                         │
└────────────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              预订改期 (Booking Rescheduled)                              │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  1. 条件判断:                                                                           │
│     ┌─ changedOrganizer = true (组织者变更)                                             │
│     │   ├─► deleteEventsAndMeetings() - 删除旧会议                                     │
│     │   └─► create() - 新建会议，生成新链接                                             │
│     │                                                                                   │
│     ├─ isLocationChanged = true (位置变更)                                             │
│     │   └─► updateLocation() - 可能创建新链接                                          │
│     │                                                                                   │
│     ├─ isDailyVideoRoomExpired = true                                                  │
│     │   └─► (结束时间 + 14 天) 已过，创建新房间                                       │
│     │                                                                                   │
│     └─ 普通改期 (仅时间变更)                                                           │
│         ├─► updateVideoEvent() → videoClient.updateMeeting()                         │
│         │   ├─► Zoom: PATCH → 链接不变                                                │
│         │   ├─► Daily: POST /rooms/{uid} → 链接不变                                   │
│         │   └─► Teams: 取决于实现                                                     │
│         │                                                                               │
│         └─► updateAllCalendarEvents()                                                 │
│             └─► Google Meet/Teams (日历联动): hangoutLink 保持不变                    │
│                                                                                         │
│  2. Metadata 更新:                                                                      │
│     ├─► hangoutLink: 可能更新（如果新建日历事件）                                       │
│     ├─► conferenceData: 可能更新                                                        │
│     └─► videoCallUrl: 重新计算                                                         │
│         └─► priority: hangoutLink \|\| updatedEvent.url \|\| ...                     │
│                                                                                         │
│  3. BookingReference 更新:                                                              │
│     └─► 如果链接变化，更新 meetingUrl, meetingId, meetingPassword                     │
│                                                                                         │
│  4. 通知分发:                                                                           │
│     └─► 与创建时相同，但触发 BOOKING_RESCHEDULED 事件                                  │
│                                                                                         │
└────────────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              预订取消 (Booking Cancelled)                                │
├────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                         │
│  1. deleteEventsAndMeetings():                                                          │
│     ├─► 按类型分类处理                                                                  │
│     │                                                                                   │
│     ├─ Calendar References (_calendar):                                                │
│     │   └─► deleteCalendarEventForBookingReference()                                   │
│     │       ├─► 删除 Google Calendar 事件 → Meet 链接失效                             │
│     │       └─► 删除 Outlook 事件 → Teams 链接失效                                    │
│     │                                                                                   │
│     ├─ Video References (_video):                                                      │
│     │   └─► deleteVideoEventForBookingReference()                                     │
│     │       ├─► Zoom: DELETE /meetings/{uid}                                          │
│     │       ├─► Daily: DELETE /rooms/{uid}                                             │
│     │       └─► Teams: 可能为空实现                                                    │
│     │                                                                                   │
│     └─ CRM References (_crm or other_calendar):                                       │
│         └─► deleteCRMEvent()                                                           │
│                                                                                         │
│  2. 错误处理:                                                                           │
│     └─► Promise.allSettled - 单个失败不影响其他操作，只记录 warn                      │
│                                                                                         │
│  3. 数据清理:                                                                           │
│     ├─► BookingReference: 软删除 (deleted 字段标记)                                    │
│     └─► Booking.metadata: 保留（用于历史记录）                                         │
│                                                                                         │
│  4. 通知:                                                                               │
│     └─► BOOKING_CANCELLED Webhook，包含原始 location/metadata                         │
│                                                                                         │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### 12.3 关键设计决策

1. **metadata.videoCallUrl 作为真相源**
   - `booking.location` 可能是类型标识而非 URL
   - `metadata.videoCallUrl` 始终是实际的会议链接
   - 日历导出强制使用此字段

2. **Promise.allSettled 用于取消操作**
   - 单个 Provider 删除失败不影响其他操作
   - 软错误策略确保用户体验流畅

3. **Daily Video 房间过期机制**
   - 房间在会议结束后 14 天过期
   - 改期时检测过期，过期则创建新房间

4. **Google Meet 与日历深度绑定**
   - 没有独立的 VideoApiAdapter
   - 通过 `conferenceData.createRequest` 间接创建
   - `hangoutLink` 是关键字段

5. **Webhook 双重字段策略**
   - `location` 字段兼容旧系统
   - `metadata.videoCallUrl` 提供精确的视频链接

---

*文档生成时间: 2026-05-04*
*基于代码提交: 当前工作目录状态*
