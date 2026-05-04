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

## 2. 预约暴露给外部日历的方式

### 2.1 主要方式：写入外部日历

Cal.diy 不会提供一个独立的、用户可订阅的 iCal feed URL。相反，系统通过以下方式工作：

1. **用户连接外部日历**：用户在 Cal.diy 中连接他们的 Google Calendar、Outlook 或 Apple Calendar
2. **设置目标日历**：用户选择一个目标日历（Destination Calendar）用于写入预约
3. **预约创建时写入**：当新预约创建时，Cal.diy 自动将事件写入用户的目标日历

**代码位置**：
- 日历服务基类：`packages/lib/CalendarService.ts:427-504` (createEvent 方法)
- iCal 字符串生成：`packages/emails/lib/generateIcsString.ts:50-124`

### 2.2 辅助方式：单个预约 ICS 下载

对于单个预约，Cal.diy 提供 ICS 文件下载链接，用户可以手动添加到日历：

**代码位置**：`packages/features/bookings/lib/getCalendarLinks.ts:138-253`

```typescript
// 生成单个预约的日历链接
export const getCalendarLinks = ({ booking, eventType, t }) => {
  // 返回四种类型的链接：
  // 1. Google Calendar (deeplink)
  // 2. Microsoft Office (deeplink)
  // 3. Microsoft Outlook (deeplink)
  // 4. ICS 文件 (data URI 或下载链接)
};
```

## 3. 鉴权机制

### 3.1 加密存储

Cal.diy 使用 AES-256 加密算法安全存储敏感的日历凭证和外部 ICS feed URL。

**加密实现**：`packages/lib/crypto.ts:1-41`

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

### 3.2 加密密钥

- **环境变量**：`CALENDSO_ENCRYPTION_KEY`
- **密钥要求**：必须是 32 字节（256 位）
- **存储位置**：不存储在数据库中，通过环境变量注入

### 3.3 外部 ICS Feed 订阅的鉴权

当用户添加外部 ICS 日历 URL 时，系统会加密存储这些 URL：

**代码位置**：`apps/api/v2/src/platform/calendars/services/ics-feed.service.ts:25-84`

```typescript
async save(userId: number, userEmail: string, urls: string[], readonly = true) {
  const data = {
    type: ICS_CALENDAR_TYPE,
    ICS_CALENDAR,
    key: symmetricEncrypt(
      JSON.stringify({ urls, skipWriting: readonly }),
      process.env.CALENDSO_ENCRYPTION_KEY || ""
    ),
    userId: userId,
    // ...
  };
  // 存储到 Credential 表
}
```

**读取时解密**：`packages/app-store/ics-feedcalendar/lib/CalendarService.ts:43-50`

```typescript
class ICSFeedCalendarService implements Calendar {
  private urls: string[] = [];

  constructor(credential: CredentialPayload) {
    const { urls } = JSON.parse(
      symmetricDecrypt(credential.key as string, CALENDSO_ENCRYPTION_KEY)
    );
    this.urls = urls;
  }
}
```

### 3.4 数据模型

**Credential 表**（简化）：
```prisma
model Credential {
  id          Int      @id @default(autoincrement())
  type        String   // 如 "ics_feed"、"google_calendar"
  key         String   // 加密存储的敏感数据
  userId      Int?
  user        User?    @relation(fields: [userId], references: [id])
  // ...
}
```

## 4. 时区数据处理

### 4.1 时区存储

**用户时区**：
- **字段**：`User.timeZone`
- **默认值**：`"Europe/London"`
- **用途**：处理全天事件、生成 VTIMEZONE 组件

**代码位置**：`packages/prisma/schema.prisma:412`
```prisma
model User {
  // ...
  timeZone    String    @default("Europe/London")
  // ...
}
```

### 4.2 VTIMEZONE 组件生成

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

### 4.3 DST 转换计算

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

### 4.4 时区注入

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

### 4.5 读取外部日历时的时区处理

当从外部日历读取事件时，Cal.diy 会智能处理时区信息：

**代码位置**：`packages/app-store/ics-feedcalendar/lib/CalendarService.ts:164-201`

```typescript
// 时区优先级：
// 1. DTSTART 属性中的 TZID
// 2. 独立的 TZID 属性
// 3. UTC 标记 ("Z")
// 4. 事件本身的 timezone
// 5. 用户的默认时区

const tzid: string | undefined =
  tzidFromDtstart || vevent?.getFirstPropertyValue("tzid") || (isUTC ? "UTC" : timezone);

// 如果日历没有 VTIMEZONE 组件，自动添加
if (!vcalendar.getFirstSubcomponent("vtimezone")) {
  const timezoneToUse = tzid || userTimeZone;
  if (timezoneToUse) {
    try {
      const timezoneComp = new ICAL.Component("vtimezone");
      timezoneComp.addPropertyWithValue("tzid", timezoneToUse);
      // ... 添加标准时间和夏令时定义
      vcalendar.addSubcomponent(timezoneComp);
    } catch (e) {
      logger.warn("error in adding vtimezone", e);
    }
  }
}
```

## 5. 私有日历的隐私边界

### 5.1 隐私控制选项

Cal.diy 提供多层次的隐私控制，确保敏感信息不会通过日历事件泄露。

#### 5.1.1 隐藏日历事件详情

**字段**：`EventType.hideCalendarEventDetails`

**效果**：
- iCal 事件设置 `CLASSIFICATION:PRIVATE`
- 外部日历客户端可能会隐藏事件详情

**代码位置**：
- 类型定义：`packages/types/Calendar.d.ts:197`
- iCal 生成：`packages/emails/lib/generateIcsString.ts:75-114`

```typescript
const icsEvent = createEvent({
  // ...
  ...(event.hideCalendarEventDetails ? { classification: "PRIVATE" } : {}),
  busyStatus: "BUSY",
});
```

#### 5.1.2 隐藏组织者邮箱

**字段**：`EventType.hideOrganizerEmail`

**效果**：
- 使用 `no-reply@cal.com` 替代真实组织者邮箱
- 例外：豁免域名列表中的邮箱不会被隐藏

**代码位置**：`packages/emails/lib/generateIcsString.ts:84-88`

```typescript
const isOrganizerExempt = ORGANIZER_EMAIL_EXEMPT_DOMAINS?.split(",")
  .filter((domain) => domain.trim() !== "")
  .some((domain) => event.organizer.email.toLowerCase().endsWith(domain.toLowerCase()));

organizer: {
  name: event.organizer.name,
  ...(event.hideOrganizerEmail && !isOrganizerExempt
    ? { email: "no-reply@cal.com" }
    : { email: event.organizer.email }),
}
```

#### 5.1.3 隐藏日历备注

**字段**：`EventType.hideCalendarNotes`

**效果**：
- 控制是否在日历事件中显示备注信息
- 影响 `description` 字段的内容

### 5.2 iCal 隐私属性说明

#### CLASSIFICATION 属性

| 值 | 说明 |
|---|------|
| `PUBLIC` | 公开事件，所有详情可见 |
| `PRIVATE` | 私有事件，仅繁忙时间可见 |
| `CONFIDENTIAL` | 机密事件，最高保密级别 |

Cal.diy 使用 `PRIVATE` 当 `hideCalendarEventDetails` 启用时。

#### X-WR-CALNAME 属性

用于日历显示名称，不影响隐私，但可能暴露日历用途。

### 5.3 跨第三方客户端的隐私边界

#### 5.3.1 Cal.diy 可控范围

Cal.diy 可以控制写入外部日历的事件内容：

```
┌─────────────────────────────────────────────────────────────┐
│                    Cal.diy 可控边界                           │
│  ┌─────────────┐      写入       ┌─────────────────────┐    │
│  │  Cal.diy    │ ──────────────► │  外部日历服务        │    │
│  │  (事件生成)  │                 │  (Google/Outlook)   │    │
│  └─────────────┘                 └──────────┬──────────┘    │
│         ▲                                    │               │
│         │ 同步取消/修改                      │ 同步           │
│         │                                    ▼               │
│  ┌─────────────┐                 ┌─────────────────────┐    │
│  │  Cal.diy    │ ◄────────────── │  日历客户端          │    │
│  │  (状态更新)  │   仅 @cal.com   │  (用户设备)          │    │
│  └─────────────┘     事件         └─────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

#### 5.3.2 外部日历服务的行为

一旦事件写入外部日历，Cal.diy 无法控制：

1. **外部日历的分享设置**：用户可能在 Google Calendar/Outlook 中分享他们的日历
2. **第三方应用的访问**：其他应用可能通过 API 访问用户的日历
3. **日历搜索**：某些日历服务可能索引事件内容

#### 5.3.3 隐私保护最佳实践

Cal.diy 通过以下方式减轻风险：

| 机制 | 实现位置 | 说明 |
|------|---------|------|
| `CLASSIFICATION:PRIVATE` | `generateIcsString.ts:112` | 标记事件为私有 |
| 邮箱隐藏 | `generateIcsString.ts:84-88` | 使用 `no-reply@cal.com` |
| 详情隐藏 | 事件类型设置 | 用户可控制是否暴露详情 |
| iCalUID 命名空间 | `CalendarSyncService.ts:47-48` | 仅同步 `@cal.com` 结尾的事件 |

### 5.4 双向同步的隐私考虑

**代码位置**：`packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts:30-72`

```typescript
async handleEvents(selectedCalendar: SelectedCalendar, calendarSubscriptionEvents: CalendarSubscriptionEventItem[]) {
  // 只处理 Cal.diy 创建的事件
  const calEvents = calendarSubscriptionEvents.filter((e) =>
    e.iCalUID?.toLowerCase()?.endsWith("@cal.com")
  );
  
  // 同步逻辑：
  // - 取消预约：检查 booking.userId === calendarUserId
  // - 重新预约：同样验证权限
  
  // 权限检查
  if (booking.userId !== calendarUserId) {
    log.debug("Skipping sync, calendar owner is not the booking host", {
      bookingUid,
      calendarUserId,
      bookingUserId: booking.userId,
    });
    return;  // 无权操作，跳过
  }
}
```

**安全边界**：
- 只同步 `iCalUID` 以 `@cal.com` 结尾的事件
- 验证 `booking.userId === calendarUserId` 确保权限
- 外部日历中非 Cal.diy 创建的事件不会被修改

## 6. 架构流程图

### 6.1 预约创建流程

```
1. 用户创建预约
        │
        ▼
2. 生成 CalendarEvent 对象
   ├── 组织者信息（name, email, timeZone）
   ├── 参与者列表
   ├── 时间（startTime, endTime）
   └── 隐私设置（hideCalendarEventDetails, hideOrganizerEmail）
        │
        ▼
3. 生成 iCal 字符串（generateIcsString）
   ├── 应用时区（injectVTimezone）
   ├── 应用隐私设置（classification: PRIVATE）
   └── 生成 VTIMEZONE 组件
        │
        ▼
4. 写入目标日历（CalendarService.createEvent）
   ├── Google Calendar API
   ├── Outlook Graph API
   └── CalDAV（Apple Calendar）
        │
        ▼
5. 用户在日历客户端查看
   ├── 事件显示为 "Busy"
   └── 详情根据 CLASSIFICATION 显示
```

### 6.2 外部日历订阅流程

```
1. 用户添加外部 ICS URL
        │
        ▼
2. 加密存储 URL
   ├── symmetricEncrypt({ urls, skipWriting })
   └── 存入 Credential.key
        │
        ▼
3. 定期拉取/订阅推送
   ├── ICS Feed: 定期 HTTP GET
   ├── Google Calendar: Webhook 推送
   └── Outlook: Webhook 推送
        │
        ▼
4. 解析事件（ical.js）
   ├── 解析 VTIMEZONE
   ├── 处理 recurring events
   └── 应用用户时区
        │
        ▼
5. 用于可用性检查
   └── getAvailability() 合并繁忙时间
```

## 7. 关键代码文件索引

| 功能 | 文件路径 | 说明 |
|------|---------|------|
| 加密/解密 | `packages/lib/crypto.ts` | AES-256 实现 |
| iCal 生成 | `packages/emails/lib/generateIcsString.ts` | 预约事件 iCal 生成 |
| 日历服务基类 | `packages/lib/CalendarService.ts` | VTIMEZONE、时区处理 |
| ICS Feed 客户端 | `packages/app-store/ics-feedcalendar/lib/CalendarService.ts` | 外部 ICS 订阅 |
| 日历同步 | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts` | 双向同步逻辑 |
| 日历链接生成 | `packages/features/bookings/lib/getCalendarLinks.ts` | 单个预约链接 |
| 数据库模型 | `packages/prisma/schema.prisma` | User, Credential, EventType |

## 8. 环境变量

| 变量名 | 用途 | 要求 |
|--------|------|------|
| `CALENDSO_ENCRYPTION_KEY` | AES-256 加密密钥 | 32 字节（256 位） |
| `ORGANIZER_EMAIL_EXEMPT_DOMAINS` | 豁免邮箱隐藏的域名列表 | 逗号分隔 |

## 9. 安全注意事项

1. **加密密钥管理**：`CALENDSO_ENCRYPTION_KEY` 应使用强随机值，定期轮换
2. **凭证安全**：即使加密，`Credential.key` 字段仍包含敏感信息，访问需严格控制
3. **隐私设置传播**：修改 `hideCalendarEventDetails` 不会追溯更新已写入外部日历的事件
4. **外部日历风险**：用户的外部日历分享设置不受 Cal.diy 控制，应在 UI 中提示
5. **iCalUID 命名空间**：依赖 `@cal.com` 后缀识别同步事件，确保不与外部事件冲突

---

*文档基于代码分析生成，最后更新：2026-05-05*
