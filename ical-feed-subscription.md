# Cal.diy iCal 日历订阅机制文档

## 1. 概述

Cal.diy 的日历订阅机制采用与传统 "独立 iCal feed URL" 不同的架构设计。系统主要通过**双向同步**和**写入外部日历**的方式，将用户的预约列表暴露给外部日历客户端（Google Calendar、Apple Calendar、Outlook 等）。

### 核心架构模式

```
┌─────────────────┐     写入事件      ┌──────────────────┐
│    Cal.diy      │ ────────────────► │  外部日历服务     │
│  (预约管理)      │                   │ (Google/Outlook) │
└────────┬────────┘                   └────────┬──────────┘
         │                                      │
         │ 订阅繁忙时间                          │ 用户查看
         ▼                                      ▼
┌─────────────────┐                   ┌──────────────────┐
│  可用性检查      │ ◄──────────────── │  日历客户端       │
│  (内部使用)      │   推送/拉取事件   │ (用户设备)        │
└─────────────────┘                   └──────────────────┘
```

**关键区别**：与传统日历应用（如 Google Calendar）提供独立的 iCal feed URL 不同，Cal.diy 采用**写入外部日历**的模式。用户需要在 Cal.diy 中连接他们的 Google/Outlook/Apple 日历，预约会直接写入这些外部日历，用户通过自己的日历客户端查看。

---

## 2. 预约暴露给外部日历的完整链路

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Cal.diy 预约暴露链路                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐                                                           │
│  │  用户创建预约  │                                                           │
│  └──────┬───────┘                                                           │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    Step 1: 确定目标日历                              │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │  │
│  │ │ 事件类型级别  │◄──►│  用户级别    │◄──►│  团队级别    │            │  │
│  │ │Destination  │    │Destination  │    │Destination  │            │  │
│  │ │Calendar     │    │Calendar     │    │Calendar     │            │  │
│  └───────────────┘    └─────────────┘    └─────────────┘            │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    Step 2: 生成 CalendarEvent                        │  │
│  │  • 组织者信息（name, email, timeZone）                               │  │
│  │  • 参与者列表                                                        │  │
│  │  • 时间（startTime, endTime, 时区处理）                              │  │
│  │  • 隐私设置（hideCalendarEventDetails, hideOrganizerEmail）         │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    Step 3: 根据日历类型选择处理方式                  │  │
│  │                                                                       │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │  │
│  │ │ Google Calendar │  │ Outlook/Office  │  │ Apple Calendar  │    │  │
│  │ │                 │  │   365           │  │   (CalDAV)      │    │  │
│  │ └────────┬────────┘  └────────┬────────┘  └────────┬────────┘    │  │
│  │          │                    │                    │              │  │
│  │          ▼                    ▼                    ▼              │  │
│  │  ┌─────────────────────────────────────────────────────────┐     │  │
│  │  │              调用对应 CalendarService.createEvent        │     │  │
│  │  │  • Google: Google Calendar API (calendar.events.insert) │     │  │
│  │  │  • Outlook: Microsoft Graph API (/me/calendar/events)  │     │  │
│  │  │  • Apple: CalDAV (PUT 请求到 .ics URL)                 │     │  │
│  │  └─────────────────────────────────────────────────────────┘     │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│         │                                                                   │
│         ▼                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    Step 4: 事件写入外部日历                         │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │  用户在日历客户端查看                                         │  │  │
│  │  │  • Google Calendar App/Web                                   │  │  │
│  │  │  • Outlook App/Web                                           │  │  │
│  │  │  • Apple Calendar (macOS/iOS)                               │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 目标日历选择机制

#### 2.2.1 数据模型

**DestinationCalendar 表**：
```prisma
model DestinationCalendar {
  id                     Int                   @id @default(autoincrement())
  integration            String           // 日历类型：google_calendar, office365_calendar, apple_calendar
  externalId             String           // 外部日历 ID
  primaryEmail           String?          // 主要邮箱
  userId                 Int?              @unique
  eventTypeId            Int?              @unique
  credentialId           Int?
  delegationCredentialId String?
  customCalendarReminder Int?
  
  // 关系
  user                   User?                 @relation(fields: [userId], references: [id])
  eventType              EventType?            @relation(fields: [eventTypeId], references: [id])
  credential             Credential?           @relation(fields: [credentialId], references: [id])
}
```

**SelectedCalendar 表**（用于可用性检查的日历选择）：
```prisma
model SelectedCalendar {
  id              String      @id @default(uuid())
  userId          Int
  integration     String      // 日历类型
  externalId      String      // 外部日历 ID
  credentialId    Int?
  eventTypeId     Int?        // 可选：事件类型级别
  googleChannelId String?     // Google Calendar 推送通道
  // ...
}
```

#### 2.2.2 目标日历优先级

Cal.diy 使用以下优先级确定预约应写入哪个日历：

```
优先级（从高到低）：

1. 事件类型级别 DestinationCalendar
   └── eventTypeId 有值时，使用该事件类型的目标日历

2. 用户级别 DestinationCalendar
   └── userId 有值时，使用用户默认目标日历

3. 团队级别（通过 DelegationCredential）
   └── 域委派凭据时，使用团队成员的日历
```

**代码位置**：`packages/features/calendars/lib/getConnectedDestinationCalendars.ts:295-338`

```typescript
// 获取已选日历（用于可用性检查）
function getSelectedCalendars({
  user,
  eventTypeId,
}: {
  user: UserWithCalendars;
  eventTypeId: number | null;
}) {
  if (eventTypeId) {
    // 优先使用事件类型级别的选择
    return user.allSelectedCalendars.filter((calendar) => calendar.eventTypeId === eventTypeId);
  }
  // 使用用户级别的选择
  return user.userLevelSelectedCalendars;
}
```

### 2.3 预约创建完整流程

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         预约创建时序图                                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  用户          RegularBookingService      CalendarManager      外部日历 API  │
│   │                  │                          │                    │       │
│   │  createBooking   │                          │                    │       │
│   │─────────────────►│                          │                    │       │
│   │                  │                          │                    │       │
│   │                  │ buildCalendarEvent()    │                    │       │
│   │                  │  • 组织者信息           │                    │       │
│   │                  │  • 参与者列表           │                    │       │
│   │                  │  • 时间和时区           │                    │       │
│   │                  │  • 隐私设置             │                    │       │
│   │                  │                          │                    │       │
│   │                  │ getConnectedDestination  │                    │       │
│   │                  │ Calendars()              │                    │       │
│   │                  │─────────────────────────►│                    │       │
│   │                  │                          │                    │       │
│   │                  │◄─────────────────────────│                    │       │
│   │                  │  (destinationCalendar)   │                    │       │
│   │                  │                          │                    │       │
│   │                  │ createEvent()            │                    │       │
│   │                  │─────────────────────────►│                    │       │
│   │                  │                          │                    │       │
│   │                  │                          │ 根据 integration   │       │
│   │                  │                          │ 选择 CalendarService │     │
│   │                  │                          │                    │       │
│   │                  │                          │ GoogleCalendar     │       │
│   │                  │                          │ .createEvent()     │       │
│   │                  │                          │───────────────────►│       │
│   │                  │                          │                    │       │
│   │                  │                          │◄───────────────────│       │
│   │                  │◄─────────────────────────│ (创建结果)         │       │
│   │◄─────────────────│                          │                    │       │
│   │  (预约创建成功)   │                          │                    │       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**代码位置**：`packages/features/bookings/lib/service/RegularBookingService.ts`

---

## 3. 访问控制机制

### 3.1 三种认证方式对比

| 日历类型 | 认证方式 | 凭证存储 | 刷新机制 |
|---------|---------|---------|---------|
| **Google Calendar** | OAuth 2.0 | `Credential.key`（加密） | 自动刷新 access_token |
| **Outlook/Office 365** | OAuth 2.0 | `Credential.key`（加密） | 自动刷新 access_token |
| **Apple Calendar** | CalDAV Basic Auth | `Credential.key`（加密） | 应用专用密码，无需刷新 |
| **域委派（Enterprise）** | Service Account | `DelegationCredential` | 客户端凭证模式 |

### 3.2 OAuth 2.0 认证流程（Google/Outlook）

#### 3.2.1 流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OAuth 2.0 连接流程                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  用户                Cal.diy                OAuth Provider                  │
│   │                    │                          │                         │
│   │  点击连接日历       │                          │                         │
│   │───────────────────►│                          │                         │
│   │                    │                          │                         │
│   │                    │  构建授权 URL           │                         │
│   │                    │  • client_id            │                         │
│   │                    │  • redirect_uri         │                         │
│   │                    │  • scope (日历权限)     │                         │
│   │                    │                          │                         │
│   │◄───────────────────│  重定向到授权页面       │                         │
│   │                    │─────────────────────────►│                         │
│   │                    │                          │                         │
│   │  授权同意           │                          │                         │
│   │──────────────────────────────────────────────►│                         │
│   │                    │                          │                         │
│   │◄──────────────────────────────────────────────│  重定向回 Cal.diy      │
│   │                    │                          │  携带 authorization_code│
│   │                    │                          │                         │
│   │                    │  用 code 换取 token     │                         │
│   │                    │─────────────────────────►│                         │
│   │                    │                          │                         │
│   │                    │◄─────────────────────────│  access_token,         │
│   │                    │                          │  refresh_token          │
│   │                    │                          │                         │
│   │                    │  加密存储 token          │                         │
│   │                    │  symmetricEncrypt()      │                         │
│   │                    │                          │                         │
│   │                    │  保存到 Credential 表    │                         │
│   │                    │  • type: "google_calendar"│                        │
│   │                    │  • key: 加密的 token    │                         │
│   │                    │  • userId: 当前用户      │                         │
│   │                    │                          │                         │
│   │◄───────────────────│  连接成功               │                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.2.2 凭证加密存储

所有敏感凭证（包括 OAuth tokens 和 CalDAV 密码）都通过 AES-256 加密存储：

**代码位置**：`packages/lib/crypto.ts:1-41`

```typescript
const ALGORITHM = "aes256";
const INPUT_ENCODING = "utf8";
const OUTPUT_ENCODING = "hex";
const IV_LENGTH = 16;

export const symmetricEncrypt = function (text: string, key: string) {
  const _key = Buffer.from(key, "latin1");
  const iv = crypto.randomBytes(IV_LENGTH);
  const cipher = crypto.createCipheriv(ALGORITHM, _key, iv);
  let ciphered = cipher.update(text, INPUT_ENCODING, OUTPUT_ENCODING);
  ciphered += cipher.final(OUTPUT_ENCODING);
  return `${iv.toString(OUTPUT_ENCODING)}:${ciphered}`;
};

export const symmetricDecrypt = function (text: string, key: string) {
  const _key = Buffer.from(key, "latin1");
  const components = text.split(":");
  const iv_from_ciphertext = Buffer.from(components.shift() || "", OUTPUT_ENCODING);
  const decipher = crypto.createDecipheriv(ALGORITHM, _key, iv_from_ciphertext);
  let deciphered = decipher.update(components.join(":"), OUTPUT_ENCODING, INPUT_ENCODING);
  deciphered += decipher.final(INPUT_ENCODING);
  return deciphered;
};
```

**加密密钥**：
- **环境变量**：`CALENDSO_ENCRYPTION_KEY`
- **要求**：必须是 32 字节（256 位）

### 3.3 CalDAV 认证（Apple Calendar）

Apple Calendar 使用 CalDAV 协议，需要应用专用密码：

**代码位置**：`packages/lib/CalendarService.ts:388-420`

```typescript
class BaseCalendarService implements Calendar {
  protected credential: CredentialPayload;
  protected integrationName: string;
  protected url: string;
  protected davAccount: DAVAccount | null = null;

  constructor(credential: CredentialPayload, integrationName: string, url: string) {
    this.credential = credential;
    this.integrationName = integrationName;
    this.url = url;
  }

  protected async getAccount(): Promise<DAVAccount> {
    if (this.davAccount) return this.davAccount;

    const { username, password } = JSON.parse(
      symmetricDecrypt(this.credential.key as string, CALENDSO_ENCRYPTION_KEY)
    );

    this.davAccount = await createAccount({
      server: this.url,
      credentials: {
        username: username,
        password: password,
      },
      authMethod: "Basic",
      defaultAccountType: "caldav",
    });

    return this.davAccount;
  }
}
```

### 3.4 域委派凭据（Enterprise）

对于企业用户，Cal.diy 支持域委派（Domain-Wide Delegation）：

**数据模型**：
```prisma
model DelegationCredential {
  id                  String   @id @default(uuid())
  type                String   // google_calendar, office365_calendar
  serviceAccountKey   Json     // 服务账号密钥（加密）
  delegatedToId       Int?     // 委派给的用户 ID
  // ...
}
```

**使用场景**：
- Google Workspace 域委派
- Microsoft 365 应用程序权限
- 无需每个用户单独授权

### 3.5 访问权限验证

#### 3.5.1 日历操作权限检查

**代码位置**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:30-72`

```typescript
async handleEvents(selectedCalendar: SelectedCalendar, calendarSubscriptionEvents: CalendarSubscriptionEventItem[]) {
  // 只处理 Cal.diy 创建的事件（通过 iCalUID 命名空间识别）
  const calEvents = calendarSubscriptionEvents.filter((e) =>
    e.iCalUID?.toLowerCase()?.endsWith("@cal.com")
  );

  for (const event of calEvents) {
    // 查找对应的预约
    const booking = await this.bookingRepository.findByUid(event.iCalUID.split("@")[0]);
    if (!booking) continue;

    // 权限检查：确保日历所有者是预约的组织者
    if (booking.userId !== calendarUserId) {
      log.debug("Skipping sync, calendar owner is not the booking host", {
        bookingUid,
        calendarUserId,
        bookingUserId: booking.userId,
      });
      continue;  // 无权操作，跳过
    }

    // 执行同步操作（取消/重新预约）
    // ...
  }
}
```

#### 3.5.2 目标日历访问验证

**代码位置**：`packages/trpc/server/routers/viewer/calendars/setDestinationCalendar.handler.ts:67-89`

```typescript
export const setDestinationCalendarHandler = async ({ ctx, input }: SetDestinationCalendarOptions) => {
  const { user } = ctx;
  const { integration, externalId, eventTypeId } = input;

  // 获取用户所有连接的日历
  const credentials = await getUsersCredentialsIncludeServiceAccountKey(user);
  const calendarCredentials = getCalendarCredentials(credentials);
  const { connectedCalendars } = await getConnectedCalendars(
    calendarCredentials,
    user.userLevelSelectedCalendars
  );

  // 验证用户是否有权访问该日历
  const allCals = connectedCalendars.map((cal) => cal.calendars ?? []).flat();
  
  const firstConnectedCalendar = getFirstConnectedCalendar({
    connectedCalendars,
    matcher: (cal) =>
      cal.externalId === externalId && 
      cal.integration === integration && 
      cal.readOnly === false,  // 必须可写
  });

  if (!firstConnectedCalendar) {
    throw new TRPCError({ 
      code: "BAD_REQUEST", 
      message: `Could not find calendar ${input.externalId}` 
    });
  }

  // 验证事件类型权限（如果指定）
  if (eventTypeId) {
    if (!(await prisma.eventType.findFirst({
      where: { id: eventTypeId, userId: user.id },
    }))) {
      throw new TRPCError({
        code: "UNAUTHORIZED",
        message: `You don't have access to event type ${eventTypeId}`,
      });
    }
  }

  // 更新 DestinationCalendar
  await DestinationCalendarRepository.upsert({ ... });
};
```

---

## 4. Google / Apple / Outlook 可见性边界差异

### 4.1 三大平台对比总览

| 特性 | Google Calendar | Outlook/Office 365 | Apple Calendar (CalDAV) |
|------|-----------------|---------------------|-------------------------|
| **API 类型** | REST API | Microsoft Graph API | CalDAV (WebDAV) |
| **认证方式** | OAuth 2.0 | OAuth 2.0 | Basic Auth (应用专用密码) |
| **隐私控制** | `visibility` 属性 | `sensitivity` 属性 | `CLASSIFICATION` iCal 属性 |
| **繁忙状态** | `transparency` | `showAs` | `TRANSP` iCal 属性 |
| **参与者可见性** | `guestsCanSeeOtherGuests` | `hideAttendees` | 依赖日历服务实现 |
| **时区处理** | API 支持 IANA 时区 | Windows 时区 → IANA 转换 | 标准 iCal VTIMEZONE |

### 4.2 Google Calendar 实现细节

#### 4.2.1 事件创建

**代码位置**：`packages/app-store/googlecalendar/lib/CalendarService.ts:179-346`

```typescript
async createEvent(
  calEvent: CalendarServiceEvent,
  credentialId: number,
  externalCalendarId?: string
): Promise<NewCalendarEventType> {
  const payload: calendar_v3.Schema$Event = {
    summary: calEvent.title,
    description: calEvent.calendarDescription,
    start: {
      dateTime: calEvent.startTime,
      timeZone: calEvent.organizer.timeZone,  // IANA 时区
    },
    end: {
      dateTime: calEvent.endTime,
      timeZone: calEvent.organizer.timeZone,
    },
    attendees: this.getAttendees({ event: calEvent, hostExternalCalendarId: externalCalendarId }),
    reminders: this.getReminders(customReminderMinutes),
    guestsCanSeeOtherGuests: calEvent.seatsPerTimeSlot ? calEvent.seatsShowAttendees : true,
    iCalUID: calEvent.iCalUID,
  };

  // 隐私控制：hideCalendarEventDetails
  if (calEvent.hideCalendarEventDetails) {
    payload.visibility = "private";  // 关键！设置为私有
  }

  // 位置处理
  if (calEvent.location) {
    payload["location"] = getLocation({ ... });
  }

  // Google Meet 会议
  if (calEvent.conferenceData && calEvent.location === MeetLocationType) {
    payload["conferenceData"] = calEvent.conferenceData;
  }

  // 调用 API
  const calendar = await this.authedCalendar();
  const selectedCalendar = externalCalendarId ?? "primary";
  
  const eventResponse = await calendar.events.insert({
    calendarId: selectedCalendar,
    requestBody: payload,
    conferenceDataVersion: 1,
    sendUpdates: "none",
  });

  return eventResponse.data;
}
```

#### 4.2.2 Google Calendar 隐私属性说明

| visibility 值 | 说明 |
|--------------|------|
| `"default"` | 使用日历的默认设置 |
| `"public"` | 公开，所有能看到日历的人都能看到详情 |
| `"private"` | 私有，只有组织者能看到详情，其他人只看到"繁忙" |
| `"confidential"` | 机密（已废弃，等同于 private） |

#### 4.2.3 参与者可见性控制

```typescript
// Google Calendar 特有：控制参与者是否能看到其他参与者
guestsCanSeeOtherGuests: calEvent.seatsPerTimeSlot 
  ? calEvent.seatsShowAttendees  // 座位模式：根据设置
  : true,  // 普通模式：可见
```

### 4.3 Outlook/Office 365 实现细节

#### 4.3.1 事件创建

**代码位置**：`packages/app-store/office365calendar/lib/CalendarService.ts:290-319`

```typescript
async createEvent(event: CalendarServiceEvent, credentialId: number): Promise<NewCalendarEventType> {
  const mainHostDestinationCalendar = event.destinationCalendar
    ? (event.destinationCalendar.find((cal) => cal.credentialId === credentialId) ??
      event.destinationCalendar[0])
    : undefined;

  const eventsUrl = mainHostDestinationCalendar?.externalId
    ? `${await this.getUserEndpoint()}/calendars/${mainHostDestinationCalendar?.externalId}/events`
    : `${await this.getUserEndpoint()}/calendar/events`;

  const response = await this.fetcher(eventsUrl, {
    method: "POST",
    body: JSON.stringify(this.translateEvent(event)),
  });

  const responseJson = await handleErrorsJson<
    NewCalendarEventType & { iCalUId: string; onlineMeeting?: { joinUrl?: string } }
  >(response);

  return { ...responseJson, iCalUID: responseJson.iCalUId };
}
```

#### 4.3.2 事件翻译（隐私控制）

**代码位置**：`packages/app-store/office365calendar/lib/CalendarService.ts:473-560`

```typescript
private translateEvent = (event: CalendarServiceEvent, rescheduledEvent?: Event) => {
  const office365Event: Event = {
    subject: event.title,
    body: {
      contentType: isOnlineMeeting ? "html" : "text",
      content,
    },
    start: {
      dateTime: dayjs(event.startTime).tz(event.organizer.timeZone).format("YYYY-MM-DDTHH:mm:ss"),
      timeZone: event.organizer.timeZone,  // IANA 时区
    },
    end: {
      dateTime: dayjs(event.endTime).tz(event.organizer.timeZone).format("YYYY-MM-DDTHH:mm:ss"),
      timeZone: event.organizer.timeZone,
    },
    hideAttendees: !event.seatsPerTimeSlot ? false : !event.seatsShowAttendees,
    organizer: {
      emailAddress: {
        address: event.destinationCalendar
          ? (event.destinationCalendar.find((cal) => cal.userId === event.organizer.id)?.externalId ??
            event.organizer.email)
          : event.organizer.email,
        name: event.organizer.name,
      },
    },
    attendees: [/* ... */],
    location: event.location ? { displayName: getLocation(event) } : undefined,
  };

  // 隐私控制：hideCalendarEventDetails
  if (event.hideCalendarEventDetails) {
    office365Event.sensitivity = "private";  // 关键！设置为私有
  }

  // Teams 在线会议
  if (isOnlineMeeting) {
    office365Event.isOnlineMeeting = true;
    office365Event.onlineMeetingProvider = "teamsForBusiness";
  }

  return office365Event;
};
```

#### 4.3.3 Outlook 隐私属性说明

| sensitivity 值 | 说明 |
|----------------|------|
| `"normal"` | 普通，默认值 |
| `"personal"` | 个人 |
| `"private"` | 私有，详情隐藏 |
| `"confidential"` | 机密 |

#### 4.3.4 时区转换（Windows → IANA）

Outlook 使用 Windows 时区名称，需要转换为 IANA 格式：

**代码位置**：`packages/app-store/office365calendar/lib/CalendarService.ts:743-793`

```typescript
async getMainTimeZone(): Promise<string> {
  try {
    const response = await this.fetcher(`${await this.getUserEndpoint()}/mailboxSettings/timeZone`);
    const timezoneResponse = await handleErrorsJson<string | { value: string }>(response);

    // 新格式：{ "value": "Windows Timezone" }
    if (typeof timezoneResponse === "object" && timezoneResponse !== null && "value" in timezoneResponse) {
      const windowsTimezoneName = timezoneResponse.value;
      
      // Windows 时区 → IANA 时区转换
      try {
        const ianaTimezone = findIana(windowsTimezoneName);
        if (ianaTimezone && ianaTimezone.length > 0) {
          return ianaTimezone[0];
        }
      } catch (conversionError) {
        // 转换失败，使用原始值
      }
      return windowsTimezoneName;
    }

    // 旧格式：直接返回字符串
    if (typeof timezoneResponse === "string") {
      return timezoneResponse;
    }

    return "Europe/London";  // 默认值
  } catch (error) {
    throw error;
  }
}
```

### 4.4 Apple Calendar (CalDAV) 实现细节

#### 4.4.1 基类实现

**代码位置**：`packages/app-store/applecalendar/lib/CalendarService.ts:1-18`

```typescript
import BaseCalendarService from "@calcom/lib/CalendarService";
import type { Calendar } from "@calcom/types/Calendar";
import type { CredentialPayload } from "@calcom/types/Credential";

class AppleCalendarService extends BaseCalendarService {
  constructor(credential: CredentialPayload) {
    super(credential, "apple_calendar", "https://caldav.icloud.com");
  }
}
```

#### 4.4.2 CalDAV 事件创建（BaseCalendarService）

**代码位置**：`packages/lib/CalendarService.ts:427-504`

```typescript
async createEvent(event: CalendarServiceEvent): Promise<NewCalendarEventType> {
  const account = await this.getAccount();
  const calendars = await fetchCalendars({ account });

  // 查找目标日历
  const calendar = calendars.find((cal) => cal.url === event.destinationCalendar?.[0]?.externalId);
  
  // 生成 iCal 字符串
  const { error, value: icsString } = createEvent({
    uid: uuidv4(),
    startOutputType: "local",
    endOutputType: "local",
    title: event.title,
    description: event.calendarDescription,
    location: event.location,
    start: convertDate(event.startTime),
    end: convertDate(event.endTime),
    duration: getDuration(event.startTime, event.endTime),
    organizer: {
      name: event.organizer.name,
      email: event.organizer.email,
    },
    attendees: event.attendees.map((attendee) => ({
      name: attendee.name,
      email: attendee.email,
      partstat: "ACCEPTED",
      role: "REQ-PARTICIPANT",
    })),
    status: "CONFIRMED",
    busyStatus: "BUSY",
    classification: event.hideCalendarEventDetails ? "PRIVATE" : "PUBLIC",  // 隐私控制
    method: "PUBLISH",
  });

  if (error) {
    throw error;
  }

  // 时区注入：添加 VTIMEZONE 组件
  let icsStringWithTimezone = icsString;
  if (event.organizer.timeZone) {
    icsStringWithTimezone = injectVTimezone(
      icsString,
      event.organizer.timeZone,
      event.startTime,
      event.endTime
    );
  }

  // 防止 CalDAV 服务器发送重复邀请邮件
  const icsStringProcessed = injectScheduleAgent(icsStringWithTimezone);

  // CalDAV PUT 请求
  const result = await createCalendarObject({
    calendar: calendar || calendars[0],
    filename: `${uuidv4()}.ics`,
    iCalString: icsStringProcessed,
  });

  return {
    uid: result?.etag || "",
    id: result?.url || "",
    type: this.integrationName,
  };
}
```

#### 4.4.3 iCal 隐私属性

Apple Calendar 使用标准 iCal 属性：

| iCal 属性 | 值 | 说明 |
|-----------|---|------|
| `CLASSIFICATION` | `PUBLIC` | 公开事件 |
| `CLASSIFICATION` | `PRIVATE` | 私有事件（hideCalendarEventDetails=true） |
| `CLASSIFICATION` | `CONFIDENTIAL` | 机密事件 |
| `TRANSP` | `OPAQUE` | 占用时间（显示为繁忙） |
| `TRANSP` | `TRANSPARENT` | 不占用时间（空闲） |

### 4.5 三大平台隐私控制对比表

| 控制维度 | Google Calendar | Outlook/Office 365 | Apple Calendar |
|---------|-----------------|---------------------|----------------|
| **事件详情可见性** | `visibility: "private"` | `sensitivity: "private"` | `CLASSIFICATION:PRIVATE` |
| **繁忙状态** | `transparency` (API) | `showAs` (API) | `TRANSP:OPAQUE` |
| **参与者可见性** | `guestsCanSeeOtherGuests` | `hideAttendees` | 依赖服务器实现 |
| **组织者邮箱** | `attendees[].organizer: true` | `organizer.emailAddress` | `ORGANIZER` iCal 属性 |
| **备注隐藏** | `description` 字段控制 | `body.content` 字段控制 | `DESCRIPTION` iCal 属性 |

### 4.6 平台特定行为差异

#### 4.6.1 事件修改/删除权限

| 平台 | Cal.diy 能否修改/删除事件 | 条件 |
|------|---------------------------|------|
| **Google Calendar** | ✅ 可以 | 事件由当前凭证创建，或有编辑权限 |
| **Outlook** | ✅ 可以 | 事件由当前凭证创建 |
| **Apple Calendar** | ✅ 可以 | 通过 CalDAV URL 访问 |

**关键限制**：
- 所有平台都**只能修改 Cal.diy 创建的事件**（通过 `iCalUID` 以 `@cal.com` 结尾识别）
- 用户在外部日历中手动创建的事件不会被 Cal.diy 同步或修改

#### 4.6.2 分享日历的可见性

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      分享日历的隐私边界                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  场景：用户 A 在 Google Calendar 中分享了自己的日历给用户 B                  │
│                                                                             │
│  ┌──────────────────┐              ┌──────────────────┐                    │
│  │    用户 A         │              │    用户 B         │                    │
│  │  (日历所有者)     │              │  (被分享者)       │                    │
│  └────────┬─────────┘              └────────┬─────────┘                    │
│           │                                  │                              │
│           │ 分享日历（权限：仅查看繁忙）      │                              │
│           │─────────────────────────────────►│                              │
│           │                                  │                              │
│           │  Cal.diy 写入事件                │                              │
│           ▼                                  │                              │
│  ┌──────────────────────────────────────┐    │                              │
│  │  外部日历服务（Google/Outlook）        │    │                              │
│  │                                      │    │                              │
│  │  事件 A（hideCalendarEventDetails=true） │    │                              │
│  │  • visibility: "private"            │    │                              │
│  │  • 所有者：用户 A                     │    │                              │
│  │                                      │    │                              │
│  │  事件 B（hideCalendarEventDetails=false）│    │                              │
│  │  • visibility: "default"            │    │                              │
│  │  • 所有者：用户 A                     │    │                              │
│  └──────────────────────────────────────┘    │                              │
│           │                                  │                              │
│           │ 用户 B 查看分享的日历            │                              │
│           ▼                                  ▼                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    用户 B 看到的内容：                                 │  │
│  │                                                                       │  │
│  │  事件 A（hideCalendarEventDetails=true）：                              │  │
│  │  ├── 时间：10:00 - 11:00 ✓ 可见                                       │  │
│  │  ├── 标题："Busy" ✗ 隐藏（显示为"繁忙"）                               │  │
│  │  ├── 详情：✗ 隐藏                                                       │  │
│  │  └── 参与者：✗ 隐藏                                                     │  │
│  │                                                                       │  │
│  │  事件 B（hideCalendarEventDetails=false）：                             │  │
│  │  ├── 时间：14:00 - 15:00 ✓ 可见                                       │  │
│  │  ├── 标题："30min Meeting - John" ✓ 可见                               │  │
│  │  ├── 详情：✓ 可见（包含 Zoom 链接、备注等）                             │  │
│  │  └── 参与者：✓ 可见（取决于 guestsCanSeeOtherGuests）                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 时区数据处理

### 5.1 时区存储

**用户时区**：
- **字段**：`User.timeZone`
- **默认值**：`"Europe/London"`
- **用途**：处理全天事件、生成 VTIMEZONE 组件

### 5.2 VTIMEZONE 组件生成

iCal 规范要求，当事件使用本地时间（非 UTC）时，必须包含 VTIMEZONE 组件。Cal.diy 会自动生成完整的时区定义。

**代码位置**：`packages/lib/CalendarService.ts:232-356`

```typescript
const buildVTimezone = (timezone: string, eventStart: string): string => {
  const eventYear = dayjs(eventStart).tz(timezone).year();
  
  // 计算夏令时转换时刻
  const winterMoment = dayjs.tz(`${eventYear}-01-15T12:00:00`, timezone);
  const summerMoment = dayjs.tz(`${eventYear}-07-15T12:00:00`, timezone);
  
  const standardOffset = formatOffset(winterMoment);  // 如 "+0800"
  const daylightOffset = formatOffset(summerMoment);  // 如 "+0900"
  const hasDST = standardOffset !== daylightOffset;
  
  // 构建 VTIMEZONE 组件
  const lines: string[] = ["BEGIN:VTIMEZONE", `TZID:${timezone}`];
  
  if (hasDST) {
    // 计算 DST 转换时刻（二分查找）
    const springTransition = findDSTTransition(timezone, eventYear, 0, 6);
    const fallTransition = findDSTTransition(timezone, eventYear, 6, 11);
    
    // 添加 DAYLIGHT 组件
    lines.push(
      "BEGIN:DAYLIGHT",
      `TZOFFSETFROM:${trueStandardOffset}`,
      `TZOFFSETTO:${trueDaylightOffset}`,
      "TZNAME:DST",
      `DTSTART:${dtstart}`,
      `RRULE:FREQ=YEARLY;BYMONTH=${bymonth};BYDAY=${byday}`,
      "END:DAYLIGHT"
    );
    
    // 添加 STANDARD 组件
    lines.push(
      "BEGIN:STANDARD",
      // ...
      "END:STANDARD"
    );
  } else {
    // 无夏令时的时区
    lines.push(
      "BEGIN:STANDARD",
      `TZOFFSETFROM:${standardOffset}`,
      `TZOFFSETTO:${standardOffset}`,
      `TZNAME:${timezone}`,
      "DTSTART:19700101T000000",
      "END:STANDARD"
    );
  }
  
  lines.push("END:VTIMEZONE");
  return lines.join("\r\n");
};
```

### 5.3 DST 转换计算

**二分查找算法**：`packages/lib/CalendarService.ts:179-207`

```typescript
const findDSTTransition = (
  timezone: string,
  year: number,
  fromMonth: number,
  toMonth: number
): ReturnType<typeof dayjs> | null => {
  let low = dayjs.utc(new Date(year, fromMonth, 1)).valueOf();
  let high = dayjs.utc(new Date(year, toMonth, 1)).valueOf();
  const lowOffset = dayjs(low).tz(timezone).utcOffset();
  const highOffset = dayjs(high).tz(timezone).utcOffset();

  if (lowOffset === highOffset) return null;  // 无时区变化

  // 二分查找精确的转换时刻
  while (high - low > 60 * 1000) {
    const mid = Math.floor((low + high) / 2);
    const midOffset = dayjs(mid).tz(timezone).utcOffset();
    if (midOffset === lowOffset) {
      low = mid;
    } else {
      high = mid;
    }
  }
  // ...
};
```

### 5.4 时区注入

**代码位置**：`packages/lib/CalendarService.ts:361-383`

```typescript
const injectVTimezone = (
  iCalString: string,
  timezone: string,
  startTime: string,
  endTime: string
): string => {
  // 1. 生成 VTIMEZONE 组件
  const vtimezone = buildVTimezone(timezone, startTime);
  
  // 2. 将 UTC 时间转换为本地时间格式
  const localStart = formatLocalDateTime(startTime, timezone);
  const localEnd = formatLocalDateTime(endTime, timezone);
  
  // 3. 替换 DTSTART/DTEND 并注入 VTIMEZONE
  let result = iCalString
    .replace(/^DTSTART:[^\r\n]+/m, `DTSTART;TZID=${timezone}:${localStart}`)
    .replace(/^DTEND:[^\r\n]+/m, `DTEND;TZID=${timezone}:${localEnd}`);
  
  result = result.replace(/^BEGIN:VEVENT/m, `${vtimezone}\r\nBEGIN:VEVENT`);
  
  return result;
};
```

---

## 6. 数据模型总结

### 6.1 核心表关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           核心数据模型                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────┐         ┌──────────────────┐         ┌──────────────┐  │
│  │    User      │         │    Credential    │         │  EventType   │  │
│  │              │         │                  │         │              │  │
│  │ id: Int      │◄────────│ userId: Int?     │────────►│ userId: Int  │  │
│  │ timeZone:    │         │ type: String     │         │              │  │
│  │   String     │         │ key: String      │         │ hideCalendar │  │
│  │ email: String│         │ (加密存储)        │         │ EventDetails │  │
│  └──────┬───────┘         └────────┬─────────┘         │ hideOrganizer│  │
│         │                          │                   │ Email        │  │
│         │                          │                   │ hideCalendar │  │
│         │                          │                   │ Notes        │  │
│         │                          │                   └──────┬───────┘  │
│         │                          │                          │          │
│         │                          ▼                          │          │
│         │              ┌──────────────────┐                   │          │
│         │              │DestinationCalendar│                   │          │
│         │              │                  │                   │          │
│         └─────────────►│ userId: Int?    │◄──────────────────┘          │
│                        │ eventTypeId: Int?│                              │
│                        │ integration:     │                              │
│                        │   String         │                              │
│                        │ externalId:      │                              │
│                        │   String         │                              │
│                        │ credentialId:    │                              │
│                        │   Int?           │                              │
│                        └────────┬─────────┘                              │
│                                 │                                        │
│                                 ▼                                        │
│                        ┌──────────────────┐                              │
│                        │ SelectedCalendar │                              │
│                        │                  │                              │
│                        │ userId: Int     │                              │
│                        │ eventTypeId: Int?│                              │
│                        │ integration:     │                              │
│                        │   String         │                              │
│                        │ externalId:      │                              │
│                        │   String         │                              │
│                        └──────────────────┘                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 表字段说明

#### Credential 表（日历凭证）
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | Int | 主键 |
| `type` | String | 日历类型：`google_calendar`, `office365_calendar`, `apple_calendar`, `ics_feed` |
| `key` | String | 加密的敏感数据（OAuth tokens、密码等） |
| `userId` | Int? | 所属用户 |
| `appId` | Int? | 关联的应用 |

#### DestinationCalendar 表（目标日历选择）
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | Int | 主键 |
| `integration` | String | 日历类型 |
| `externalId` | String | 外部日历 ID |
| `userId` | Int? | 用户级别（唯一） |
| `eventTypeId` | Int? | 事件类型级别（唯一） |
| `credentialId` | Int? | 关联的凭证 |
| `primaryEmail` | String? | 主要邮箱 |

#### SelectedCalendar 表（可用性检查日历）
| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | String | UUID 主键 |
| `userId` | Int | 所属用户 |
| `eventTypeId` | Int? | 事件类型级别（可选） |
| `integration` | String | 日历类型 |
| `externalId` | String | 外部日历 ID |
| `credentialId` | Int? | 关联的凭证 |
| `googleChannelId` | String? | Google Calendar 推送通道 |

---

## 7. 安全边界与最佳实践

### 7.1 隐私保护机制

| 机制 | 实现位置 | 说明 |
|------|---------|------|
| `hideCalendarEventDetails` | EventType 模型 | 设置 `visibility/private` 或 `CLASSIFICATION:PRIVATE` |
| `hideOrganizerEmail` | EventType 模型 | 使用 `no-reply@cal.com` 替代真实邮箱 |
| `hideCalendarNotes` | EventType 模型 | 隐藏事件备注 |
| `guestsCanSeeOtherGuests` | Google Calendar API | 控制参与者互见 |
| `hideAttendees` | Outlook Graph API | 隐藏参与者列表 |

### 7.2 访问控制边界

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          访问控制边界                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Cal.diy 可控范围：                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ✓ 写入外部日历时设置隐私属性                                           │  │
│  │  ✓ 加密存储所有敏感凭证                                                 │  │
│  │  ✓ 只同步 iCalUID 以 @cal.com 结尾的事件                                │  │
│  │  ✓ 验证 booking.userId === calendarUserId                              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  外部日历服务可控范围（Cal.diy 无法控制）：                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ✗ 用户在 Google/Outlook 中的日历分享设置                               │  │
│  │  ✗ 其他应用通过 API 访问用户日历的权限                                   │  │
│  │  ✗ 日历服务的索引和搜索功能                                              │  │
│  │  ✗ 用户手动修改的事件                                                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  建议的安全措施：                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  1. 启用 hideCalendarEventDetails 保护敏感预约                          │  │
│  │  2. 启用 hideOrganizerEmail 保护组织者隐私                               │  │
│  │  3. 不在外部日历中分享包含 Cal.diy 预约的日历                            │  │
│  │  4. 定期审查已连接的日历应用                                             │  │
│  │  5. 使用应用专用密码（Apple Calendar）                                   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 安全注意事项

1. **加密密钥管理**：`CALENDSO_ENCRYPTION_KEY` 应使用强随机值，定期轮换
2. **凭证安全**：即使加密，`Credential.key` 字段仍包含敏感信息，访问需严格控制
3. **隐私设置传播**：修改 `hideCalendarEventDetails` 不会追溯更新已写入外部日历的事件
4. **外部日历风险**：用户的外部日历分享设置不受 Cal.diy 控制，应在 UI 中提示
5. **iCalUID 命名空间**：依赖 `@cal.com` 后缀识别同步事件，确保不与外部事件冲突
6. **域委派凭据**：Enterprise 功能应限制访问，服务账号密钥需严格保护

---

## 8. 关键代码文件索引

| 功能 | 文件路径 | 说明 |
|------|---------|------|
| 加密/解密 | `packages/lib/crypto.ts` | AES-256 实现 |
| iCal 生成 | `packages/emails/lib/generateIcsString.ts` | 预约事件 iCal 生成 |
| 日历服务基类 | `packages/lib/CalendarService.ts` | CalDAV、VTIMEZONE、时区处理 |
| Google Calendar | `packages/app-store/googlecalendar/lib/CalendarService.ts` | Google API 集成 |
| Outlook/Office 365 | `packages/app-store/office365calendar/lib/CalendarService.ts` | Graph API 集成 |
| Apple Calendar | `packages/app-store/applecalendar/lib/CalendarService.ts` | CalDAV 集成 |
| ICS Feed 客户端 | `packages/app-store/ics-feedcalendar/lib/CalendarService.ts` | 外部 ICS 订阅 |
| 日历同步 | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts` | 双向同步逻辑 |
| 日历链接生成 | `packages/features/bookings/lib/getCalendarLinks.ts` | 单个预约链接 |
| 目标日历管理 | `packages/features/calendars/repositories/DestinationCalendarRepository.ts` | DestinationCalendar 操作 |
| 日历管理 | `packages/features/calendars/lib/CalendarManager.ts` | 事件创建、更新、删除 |
| 数据库模型 | `packages/prisma/schema.prisma` | User, Credential, EventType, DestinationCalendar, SelectedCalendar |

---

## 9. 环境变量

| 变量名 | 用途 | 要求 |
|--------|------|------|
| `CALENDSO_ENCRYPTION_KEY` | AES-256 加密密钥 | 32 字节（256 位） |
| `ORGANIZER_EMAIL_EXEMPT_DOMAINS` | 豁免邮箱隐藏的域名列表 | 逗号分隔 |

---

*文档基于代码分析生成，最后更新：2026-05-05*
