# Cal.diy i18n 和 Timezone 分析文档

## 概述

本文档分析了 Cal.diy 代码库中 locale（语言环境）和 timezone（时区）如何影响以下核心功能：
- Slot（时间槽）展示
- Booking（预订）写入
- 邮件模板
- 服务端时间计算

同时也分析了前后端共享日期工具的协作机制、时区优先级、夏令时处理、跨日场景，并提供了端到端时序例子。

---

## 1. Locale 和 Timezone 来源优先级与冲突分支

### 1.1 时区优先级链

时区的确定遵循以下优先级链（从高到低）：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         时区优先级链                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  优先级 1: URL Query Parameter (cal.tz)                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  示例: https://cal.com/user/30min?cal.tz=Asia/Tokyo                │   │
│  │  存储位置: BookerStore.state.timezone                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  优先级 2: 访客手动选择的时区 (Booker UI)                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  用户在预订页面下拉框选择时区                                          │   │
│  │  存储位置: BookerStore.state.timezone                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  优先级 3: 登录用户的偏好设置 (timePreferences)                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  User 表中的 timeZone 字段                                           │   │
│  │  通过 useTimePreferences() Hook 获取                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  优先级 4: 活动类型默认时区 (EventType.schedule.timeZone)                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  EventType.schedule?.timeZone                                       │   │
│  │  如果活动关联了特定日程，使用该日程的时区                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  优先级 5: 组织者用户默认时区 (User.timeZone)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  活动组织者的时区设置                                                  │   │
│  │  User.timeZone 字段                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  兜底: UTC                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  normalizeTimezone(undefined) 返回 "UTC"                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心优先级代码

**代码位置**: `packages/features/bookings/Booker/utils/getBookerTimezone.ts:4-11`

```typescript
export const getBookerTimezone = ({
  storeTimezone,
  bookerUserPreferredTimezone,
}: {
  storeTimezone: string | null;
  bookerUserPreferredTimezone: string;
}) => {
  // BookerStore timezone is the one that is updated no matter what could be the reason of the update
  // e.g. timezone configured through cal.tz query param is available there but not in the preferences
  return storeTimezone ?? bookerUserPreferredTimezone;
};
```

**代码位置**: `packages/features/bookings/Booker/hooks/useBookerTime.ts:6-20`

```typescript
export const useBookerTime = () => {
  const [timezoneFromBookerStore] = useBookerStoreContext((state) => [state.timezone], shallow);
  const { timezone: timezoneFromTimePreferences, timeFormat } = useTimePreferences();
  const timezone = getBookerTimezone({
    storeTimezone: timezoneFromBookerStore,
    bookerUserPreferredTimezone: timezoneFromTimePreferences,
  });

  return {
    timezone,
    timeFormat,
    timezoneFromBookerStore,
    timezoneFromTimePreferences,
  };
};
```

### 1.3 Locale 优先级链

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Locale 优先级链                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  优先级 1: 请求头 Accept-Language                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  浏览器或客户端发送的 Accept-Language 头                              │   │
│  │  由 next-i18next 自动检测                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  优先级 2: URL 参数 (?hl=zh-CN 或 ?locale=fr)                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  可以通过 URL 参数强制指定语言                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  优先级 3: 登录用户的偏好设置 (User.locale)                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  User 表中的 locale 字段                                              │   │
│  │  用户设置页面配置的语言                                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  优先级 4: 活动组织者的语言偏好                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  活动组织者 User.locale                                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                          │
│                                    ▼                                          │
│  兜底: "en" (English)                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  代码中多处使用 ?? "en" 作为兜底                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.4 冲突处理策略

#### 场景 1: 访客选择 vs 用户偏好

**冲突情况**：访客（未登录）手动选择了时区，但活动组织者有默认时区

**处理策略**：
- **时间槽展示**：使用访客选择的时区（BookerStore 优先）
- **预订写入**：
  - 时间存储为 UTC
  - 参会者时区记录为访客选择的时区
  - 组织者时区保持不变

**代码位置**: `packages/features/bookings/lib/service/RegularBookingService.ts:1158-1170`

```typescript
// If the Organizer himself is rescheduling, the booker should be sent the communication in his timezone and locale.
const attendeeInfoOnReschedule =
  userReschedulingIsOwner && originalRescheduledBooking
    ? originalRescheduledBooking.attendees.find((attendee) => attendee.email === bookerEmail)
    : null;

const attendeeLanguage = attendeeInfoOnReschedule ? attendeeInfoOnReschedule.locale : language;
const attendeeTimezone = attendeeInfoOnReschedule ? attendeeInfoOnReschedule.timeZone : reqBody.timeZone;

const tAttendees = await getTranslation(attendeeLanguage ?? "en", "common");
```

#### 场景 2: 重排（Reschedule）时区继承

**冲突情况**：组织者重新预订，需要决定使用谁的时区

**处理策略**：

```typescript
// 规则: 如果是组织者自己重新预订，使用原始预订中参会者的时区和语言
// 原因: 确保参会者收到的沟通使用他们熟悉的时区和语言

const attendeeInfoOnReschedule =
  userReschedulingIsOwner && originalRescheduledBooking
    ? originalRescheduledBooking.attendees.find((attendee) => attendee.email === bookerEmail)
    : null;

// 优先级:
// 1. 如果有原始预订的参会者信息 → 使用原始的 locale 和 timeZone
// 2. 否则 → 使用当前请求中的 language 和 timeZone

const attendeeLanguage = attendeeInfoOnReschedule ? attendeeInfoOnReschedule.locale : language;
const attendeeTimezone = attendeeInfoOnReschedule ? attendeeInfoOnReschedule.timeZone : reqBody.timeZone;
```

**决策流程图**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    重排时区/语言决策流程                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  开始                                                                        │
│    │                                                                         │
│    ▼                                                                         │
│  ┌─────────────────────┐                                                     │
│  │ 是否是重新预订?       │                                                     │
│  │ (rescheduleUid存在?) │                                                     │
│  └──────────┬──────────┘                                                     │
│             │                                                                 │
│     ┌───────┴───────┐                                                         │
│     │               │                                                         │
│     ▼               ▼                                                         │
│   是              否                                                          │
│     │               │                                                         │
│     ▼               │                                                         │
│  ┌──────────────────────────────────┐                                          │
│  │ userReschedulingIsOwner?         │                                          │
│  │ (当前用户是否是原始预订的组织者?)  │                                          │
│  └────────────┬─────────────────────┘                                          │
│               │                                                                  │
│      ┌────────┴────────┐                                                        │
│      │                 │                                                        │
│      ▼                 ▼                                                        │
│    是                否                                                         │
│      │                 │                                                        │
│      ▼                 │                                                        │
│  ┌─────────────────────────────────────────┐                                   │
│  │ 从 originalRescheduledBooking.attendees  │                                   │
│  │ 查找匹配 email 的参会者记录              │                                   │
│  └────────────┬────────────────────────────┘                                   │
│               │                                                                  │
│      ┌────────┴────────┐                                                        │
│      │                 │                                                        │
│      ▼                 ▼                                                        │
│   找到             未找到                                                       │
│      │                 │                                                        │
│      ▼                 ▼                                                        │
│  使用原始           使用请求中的                                               │
│  locale/timeZone   language/timeZone                                          │
│      │                 │                                                        │
│      └────────┬────────┘                                                        │
│               │                                                                  │
│               ▼                                                                  │
│          ┌────────┐                                                             │
│          │ 结束   │                                                             │
│          └────────┘                                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.5 时区规范化与验证

**代码位置**: `packages/features/calendars/lib/timezone-conversion.ts:28-43`

```typescript
export function normalizeTimezone(timezone: string | undefined): string {
  if (!timezone) return "UTC";

  try {
    Intl.DateTimeFormat(undefined, { timeZone: timezone });
    return timezone;
  } catch {
    const converted = convertOffsetToIanaTimezone(timezone);
    if (converted) {
      log.info(`Converted offset timezone to IANA format`, { original: timezone, converted });
      return converted;
    }
    log.warn(`Invalid timezone format, falling back to UTC`, { invalidTimezone: timezone });
    return "UTC";
  }
}
```

**处理规则**：
1. **无效时区兜底**：如果时区字符串无效，尝试转换为 IANA 格式
2. **偏移格式转换**：将 `GMT+5:30` 或 `UTC-8` 转换为 `Etc/GMT` 系列
3. **最终兜底**：无法转换时使用 `UTC`

**代码位置**: `packages/features/calendars/lib/timezone-conversion.ts:9-22`

```typescript
export function convertOffsetToIanaTimezone(timezone: string): string | null {
  const match = timezone.match(/^(?:GMT|UTC)([+-])(\d{1,2})(?::(\d{2}))?$/i);
  if (!match) return null;

  const [, sign, hoursStr, minutesStr] = match;
  const hours = parseInt(hoursStr, 10);
  const minutes = minutesStr ? parseInt(minutesStr, 10) : 0;

  // Etc/GMT only supports whole hours up to 14
  if (minutes !== 0 || hours > 14) return null;

  if (hours === 0) return "Etc/GMT";
  // 注意: Etc/GMT 使用反向符号
  // Etc/GMT+5 = UTC-5
  // Etc/GMT-8 = UTC+8
  return `Etc/GMT${sign === "+" ? "-" : "+"}${hours}`;
}
```

---

## 2. 夏令时切换处理策略

### 2.1 夏令时检测工具函数

**代码位置**: `packages/lib/dayjs/index.ts:194-236`

```typescript
/**
 * Verify if timeZone has Daylight Saving Time (DST).
 *
 * 北半球: DST 通常 3-4月开始，9-11月结束
 * 南半球: DST 通常 9-11月开始，3-4月结束
 */
export const timeZoneWithDST = (timeZone: string): boolean => {
  const jan = dayjs.tz(`${new Date().getFullYear()}-01-01T00:00:00`, timeZone);
  const jul = dayjs.tz(`${new Date().getFullYear()}-07-01T00:00:00`, timeZone);
  return jan.utcOffset() !== jul.utcOffset();
};

/**
 * Get DST difference.
 * 大多数地区 DST 是 1 小时
 * 例外: 澳大利亚 Lord Howe Island 是 30 分钟
 */
export const getDSTDifference = (timeZone: string): number => {
  const jan = dayjs.tz(`${new Date().getFullYear()}-01-01T00:00:00`, timeZone);
  const jul = dayjs.tz(`${new Date().getFullYear()}-07-01T00:00:00`, timeZone);
  return jul.utcOffset() - jan.utcOffset();
};

/**
 * Verifies if given time zone is in DST
 */
export const isInDST = (date: Dayjs) => {
  const timeZone = getTimeZone(date);
  return timeZoneWithDST(timeZone) && date.utcOffset() === getUTCOffsetInDST(timeZone);
};
```

### 2.2 工作时间计算中的夏令时处理

**代码位置**: `packages/features/schedules/lib/date-ranges.ts:58-74`

```typescript
function processWorkingHours(
  results: Record<number, DateRange>,
  {
    item,
    timeZone,
    dateFrom,
    dateTo,
    travelSchedules,
  }: {
    item: WorkingHours;
    timeZone: string;
    dateFrom: Dayjs;
    dateTo: Dayjs;
    travelSchedules: TravelSchedule[];
  }
) {
  // ...

  // it always has to be start of the day (midnight) even when DST changes
  const dateInTz = date.add(fromOffset - offset, "minutes").tz(adjustedTimezone);

  // ...

  let start = dateInTz
    .add(item.startTime.getUTCHours(), "hours")
    .add(item.startTime.getUTCMinutes(), "minutes");

  let end = dateInTz.add(item.endTime.getUTCHours(), "hours").add(item.endTime.getUTCMinutes(), "minutes");

  // DST 切换日的偏移调整
  const offsetBeginningOfDay = dayjs(start.format("YYYY-MM-DD hh:mm")).tz(adjustedTimezone).utcOffset();
  const offsetDiff = start.utcOffset() - offsetBeginningOfDay; // there will be 60 min offset on the day day of DST change

  start = start.add(offsetDiff, "minute");
  end = end.add(offsetDiff, "minute");

  // ...
}
```

**处理策略说明**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    夏令时切换日的时间计算问题                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  场景: 美国纽约 (America/New_York)                                          │
│  - 标准时间: UTC-5                                                          │
│  - 夏令时: UTC-4 (时钟拨快 1 小时)                                          │
│                                                                             │
│  假设用户工作时间设置为: 09:00 - 17:00                                      │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 问题场景: DST 开始日 (春季，时钟拨快)                                 │   │
│  │                                                                       │   │
│  │ UTC 时间线:                                                           │   │
│  │ 01:00 UTC ──► 02:00 UTC ──► 03:00 UTC ──► 04:00 UTC              │   │
│  │              │               │                                        │   │
│  │              ▼               ▼                                        │   │
│  │ 纽约本地:    20:00 EST      00:00 EDT (跳过 02:00)                │   │
│  │              (UTC-5)         (UTC-4)                                 │   │
│  │                                                                       │   │
│  │ 问题: 如果简单地按 UTC 偏移计算，                                      │   │
│  │       09:00 纽约时间可能对应错误的 UTC 时间                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 解决方案:                                                             │   │
│  │                                                                       │   │
│  │ 1. 计算日期开始时的偏移量 (offsetBeginningOfDay)                      │   │
│  │ 2. 计算当前时间的偏移量 (start.utcOffset())                          │   │
│  │ 3. 计算差值 (offsetDiff)                                              │   │
│  │ 4. 调整时间: start.add(offsetDiff, "minute")                         │   │
│  │                                                                       │   │
│  │ 这样确保: 无论 DST 是否切换，用户设置的 09:00 始终是当地时间 09:00   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 各链路中的夏令时一致性保证

| 链路 | 处理位置 | 一致性策略 |
|------|----------|------------|
| **Slot 计算** | `date-ranges.ts:processWorkingHours` | 使用 `offsetDiff` 调整 DST 切换日的时间 |
| **Booking 写入** | `RegularBookingService` | 时间存储为 UTC，时区信息分离存储 |
| **邮件展示** | `date-formatting.ts`, `WhenInfo.tsx` | 使用 IANA 时区转换，自动处理 DST |
| **前端展示** | `MeetingTimeInTimezones.tsx` | 使用 `dayjs.tz()` 自动处理 DST |

### 2.4 夏令时兜底策略

**策略 1: 使用 IANA 时区而非固定偏移**

```typescript
// 错误: 使用固定偏移，无法处理 DST
const timeZone = "UTC-5";

// 正确: 使用 IANA 时区，自动处理 DST
const timeZone = "America/New_York";
```

**策略 2: 无效时区的规范化**

```typescript
// normalizeTimezone 确保:
// 1. 验证时区是否有效
// 2. 无效时区兜底为 UTC
// 3. 偏移格式尝试转换为 Etc/GMT

const safeTimeZone = normalizeTimezone(userInputTimeZone);
```

**策略 3: 时间边界检查**

```typescript
// 在 date-ranges.ts 中处理 DST 切换日的边界问题

// 确保结束时间不会因为 DST 而异常
if (endResult.hour() === 23 && endResult.minute() === 59) {
  endResult = endResult.add(1, "minute");
}
```

---

## 3. 跨日场景处理策略

### 3.1 跨日检测工具函数

**代码位置**: `packages/lib/dayjs/index.ts:127-155`

```typescript
/**
 * Verifies given time is a day before in timezoneB.
 * 检查在 timezoneB 中，给定时间是否是前一天
 */
export const isPreviousDayInTimezone = (time: string, timezoneA: string, timezoneB: string) => {
  const timeInTimezoneA = formatTime(time, 24, timezoneA);
  const timeInTimezoneB = formatTime(time, 24, timezoneB);
  if (time === timeInTimezoneB) return false;

  // Eg timeInTimezoneA = 12:00 and timeInTimezoneB = 23:00
  const hoursTimezoneBIsLater = timeInTimezoneB.localeCompare(timeInTimezoneA) === 1;
  // If it is 23:00, does timezoneA come before or after timezoneB in GMT?
  const timezoneBIsEarlierTimezone = sortByTimezone(timezoneA, timezoneB) === 1;
  return hoursTimezoneBIsLater && timezoneBIsEarlierTimezone;
};

/**
 * Verifies given time is a day after in timezoneB.
 * 检查在 timezoneB 中，给定时间是否是后一天
 */
export const isNextDayInTimezone = (time: string, timezoneA: string, timezoneB: string) => {
  const timeInTimezoneA = formatTime(time, 24, timezoneA);
  const timeInTimezoneB = formatTime(time, 24, timezoneB);
  if (time === timeInTimezoneB) return false;

  // Eg timeInTimezoneA = 12:00 and timeInTimezoneB = 09:00
  const hoursTimezoneBIsEarlier = timeInTimezoneB.localeCompare(timeInTimezoneA) === -1;
  // If it is 09:00, does timezoneA come before or after timezoneB in GMT?
  const timezoneBIsLaterTimezone = sortByTimezone(timezoneA, timezoneB) === -1;
  return hoursTimezoneBIsEarlier && timezoneBIsLaterTimezone;
};
```

### 3.2 跨日场景示例

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         跨日场景示例                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  场景: 组织者在纽约 (America/New_York, UTC-5/UTC-4 DST)                    │
│        参会者在东京 (Asia/Tokyo, UTC+9)                                     │
│                                                                             │
│  预订时间: 纽约时间 2026-05-15 23:00 - 00:00 (次日)                        │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 时间转换:                                                             │   │
│  │                                                                       │   │
│  │ 纽约时间 (EDT, UTC-4):      2026-05-15 23:00 ──► 2026-05-16 00:00 │   │
│  │                              │                        │              │   │
│  │ UTC 时间:                  2026-05-16 03:00 ──► 2026-05-16 04:00 │   │
│  │                              │                        │              │   │
│  │ 东京时间 (JST, UTC+9):      2026-05-16 12:00 ──► 2026-05-16 13:00 │   │
│  │                                                                       │   │
│  │ 注意: 纽约的"5月15日晚11点"在东京是"5月16日中午12点"                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 各链路处理:                                                           │   │
│  │                                                                       │   │
│  │ 1. Slot 展示:                                                        │   │
│  │    - 纽约用户看到: 2026-05-15 23:00 - 00:00 (+1 天标记)            │   │
│  │    - 东京用户看到: 2026-05-16 12:00 - 13:00                        │   │
│  │                                                                       │   │
│  │ 2. Booking 写入:                                                      │   │
│  │    - 数据库存储: startTime='2026-05-16T03:00:00Z' (UTC)           │   │
│  │    - 组织者时区: America/New_York                                    │   │
│  │    - 参会者时区: Asia/Tokyo                                          │   │
│  │                                                                       │   │
│  │ 3. 邮件发送:                                                          │   │
│  │    - 组织者收到: "5月15日 23:00 - 00:00" (纽约时间)                │   │
│  │    - 参会者收到: "5月16日 12:00 - 13:00" (东京时间)                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 前端多时区展示组件

**代码位置**: `packages/ui/components/popover/MeetingTimeInTimezones.tsx:29-94`

```typescript
const MeetingTimeInTimezones = ({
  attendees,
  userTimezone,
  timeFormat,
  startTime,
  endTime,
}: MeetingTimeInTimezonesProps) => {
  if (!userTimezone || !attendees.length) return null;

  // If attendeeTimezone is unsupported, we fallback to host timezone.
  const attendeeTimezones = attendees.map((attendee) => {
    return isSupportedTimeZone(attendee.timeZone) ? attendee.timeZone : userTimezone;
  });
  const uniqueTimezones = [userTimezone, ...attendeeTimezones].filter(
    (value, index, self) => self.indexOf(value) === index
  );

  // Convert times to time in timezone, and then sort from earliest to latest time in timezone.
  const times = uniqueTimezones
    .map((timezone) => {
      const isPreviousDay = isPreviousDayInTimezone(startTime, userTimezone, timezone);
      const isNextDay = isNextDayInTimezone(startTime, userTimezone, timezone);
      return {
        startTime: formatTime(startTime, timeFormat, timezone),
        endTime: formatTime(endTime, timeFormat, timezone),
        timezone,
        isPreviousDay,
        isNextDay,
      };
    })
    .sort((timeA, timeB) => sortByTimezone(timeA.timezone, timeB.timezone));

  // 只有一个时区时不显示
  if (times.length === 1) return null;

  // 渲染多时区时间展示，包含 +1 / -1 天标记
  return (
    <Popover.Root>
      {/* ... */}
      <Popover.Content>
        {times.map((time) => (
          <span key={time.timezone}>
            <span>
              {time.startTime} - {time.endTime}
              {(time.isNextDay || time.isPreviousDay) && (
                <span>
                  {time.isNextDay ? "+1" : "-1"}
                </span>
              )}
            </span>
            <br />
            <span>{time.timezone}</span>
          </span>
        ))}
      </Popover.Content>
    </Popover.Root>
  );
};
```

### 3.4 邮件中的跨日处理

**代码位置**: `packages/emails/lib/utils/date-formatting.ts:5-26`

```typescript
export function getFormattedDate(calEvent: CalendarEvent, attendee: Person): string {
  const inviteeTimeFormat = calEvent.organizer.timeFormat || TimeFormat.TWELVE_HOUR;
  const timezone = attendee.timeZone;
  const locale = attendee.language.locale;
  const t = attendee.language.translate;

  const getFormattedRecipientTime = (time: string, format: string) => {
    return dayjs(time).tz(timezone).locale(locale).format(format);
  };

  // ...

  // 自动处理跨日
  // 如果 start 和 end 在不同日期，格式化会自然体现
  return `${getInviteeStart(inviteeTimeFormat)} - ${getInviteeEnd(inviteeTimeFormat)}, ${t(
    getInviteeStart("dddd").toLowerCase()
  )}, ${t(getInviteeStart("MMMM").toLowerCase())} ${getInviteeStart("D, YYYY")}`;
}
```

### 3.5 跨日场景的一致性保证

| 链路 | 处理方式 | 一致性策略 |
|------|----------|------------|
| **Slot 展示** | `getSlots()` 中使用 `dayjs.tz(timeZone)` | 按预订者时区展示 |
| **Booking 写入** | `dayjs().utc().format()` 存储 | 统一存储为 UTC |
| **邮件发送** | `dayjs(time).tz(attendee.timeZone)` | 按接收者时区展示 |
| **前端 UI** | `MeetingTimeInTimezones` 组件 | 多时区对比展示，+1/-1 标记 |

---

## 4. 端到端时序例子

### 4.1 场景描述

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         跨时区预订场景                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  参与者:                                                                     │
│  - 组织者: Alice (纽约, America/New_York, 语言: en, 时间格式: 12小时)      │
│  - 参会者: Bob (东京, Asia/Tokyo, 语言: ja, 时间格式: 24小时)              │
│                                                                             │
│  事件:                                                                       │
│  - 活动类型: 30分钟会议                                                      │
│  - 活动时区: 使用组织者默认时区 (America/New_York)                          │
│                                                                             │
│  时间点:                                                                     │
│  - 当前 UTC 时间: 2026-05-15T12:00:00Z                                     │
│  - Bob 选择的时间: 纽约时间 2026-05-15 23:00 (即东京时间 2026-05-16 12:00) │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 完整时序图

```
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              端到端时序: Bob 预订 Alice 的会议                                        │
├──────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                  │
│  Bob (东京)                        前端                     API 服务                数据库         │
│  Asia/Tokyo, ja                                                                                  │
│       │                              │                         │                       │            │
│       │  1. 打开预订页面             │                         │                       │            │
│       │─────────────────────────────►│                         │                       │            │
│       │                              │                         │                       │            │
│       │                              │  2. 获取可用时间槽       │                       │            │
│       │                              │────────────────────────►│                       │            │
│       │                              │                         │                       │            │
│       │                              │                         │  3. 计算 Alice 的可用性 │            │
│       │                              │                         │  - Alice 时区: America/New_York │
│       │                              │                         │  - 当前 UTC: 2026-05-15T12:00Z  │
│       │                              │                         │  - Alice 工作时间: 09:00-17:00 ET │
│       │                              │                         │                       │            │
│       │                              │                         │  4. 转换到 Bob 的时区   │            │
│       │                              │                         │  - 纽约 23:00 = 东京次日 12:00    │
│       │                              │                         │                       │            │
│       │                              │  5. 返回可用时间槽       │                       │            │
│       │                              │◄────────────────────────│                       │            │
│       │                              │                         │                       │            │
│       │  6. 选择时间槽               │                         │                       │            │
│       │  (显示为东京时间 5月16日 12:00)│                         │                       │            │
│       │─────────────────────────────►│                         │                       │            │
│       │                              │                         │                       │            │
│       │                              │  7. 提交预订请求         │                       │            │
│       │                              │  - start: "2026-05-15T23:00:00-04:00" │            │
│       │                              │  - timeZone: "Asia/Tokyo" │                       │            │
│       │                              │  - language: "ja"       │                       │            │
│       │                              │────────────────────────►│                       │            │
│       │                              │                         │                       │            │
│       │                              │                         │  8. 处理预订            │                       │
│       │                              │                         │  - 转换 start 为 UTC:   │                       │
│       │                              │                         │    dayjs("2026-05-15T23:00:00-04:00") │    │
│       │                              │                         │    .utc().format()    │                       │
│       │                              │                         │    = "2026-05-16T03:00:00Z" │           │
│       │                              │                         │                       │            │
│       │                              │                         │  9. 写入数据库          │                       │
│       │                              │                         │  - Booking:            │                       │
│       │                              │                         │    startTime: 2026-05-16T03:00:00Z │    │
│       │                              │                         │    endTime: 2026-05-16T03:30:00Z   │    │
│       │                              │                         │  - Attendee:           │                       │
│       │                              │                         │    timeZone: Asia/Tokyo │                       │
│       │                              │                         │    locale: ja          │                       │
│       │                              │                         │  - Organizer (Alice):  │                       │
│       │                              │                         │    timeZone: America/New_York │            │
│       │                              │                         │    locale: en          │                       │
│       │                              │                         │──────────────────────►│            │
│       │                              │                         │                       │            │
│       │                              │                         │  10. 发送邮件          │                       │
│       │                              │                         │                       │            │
│       │                              │                         │  ┌─────────────────┐   │                       │
│       │                              │                         │  │ 给 Alice 的邮件   │   │                       │
│       │                              │                         │  │ - 时区: America/New_York │               │
│       │                              │                         │  │ - 语言: en       │   │                       │
│       │                              │                         │  │ - 时间显示:       │   │                       │
│       │                              │                         │  │   "May 15, 11:00 PM - 11:30 PM" │     │
│       │                              │                         │  │   "Friday"        │   │                       │
│       │                              │                         │  └─────────────────┘   │                       │
│       │                              │                         │                       │            │
│       │                              │                         │  ┌─────────────────┐   │                       │
│       │                              │                         │  │ 给 Bob 的邮件     │   │                       │
│       │                              │                         │  │ - 时区: Asia/Tokyo │   │                       │
│       │                              │                         │  │ - 语言: ja        │   │                       │
│       │                              │                         │  │ - 时间显示:       │   │                       │
│       │                              │                         │  │   "5月16日 12:00 - 12:30" │           │
│       │                              │                         │  │   "土曜日"        │   │                       │
│       │                              │                         │  └─────────────────┘   │                       │
│       │                              │                         │                       │            │
│       │                              │  11. 返回预订确认        │                       │            │
│       │                              │◄────────────────────────│                       │            │
│       │                              │                         │                       │            │
│       │  12. 显示确认页面            │                         │                       │            │
│       │  (东京时间 5月16日 12:00)    │                         │                       │            │
│       │◄─────────────────────────────│                         │                       │            │
│       │                              │                         │                       │            │
└───────┴──────────────────────────────┴─────────────────────────┴───────────────────────┘
```

### 4.3 关键时间点转换表

| 时间表示 | 纽约时间 (Alice) | UTC | 东京时间 (Bob) |
|----------|------------------|-----|-----------------|
| **Bob 选择的时间** | 2026-05-15 23:00 EDT | 2026-05-16 03:00 Z | 2026-05-16 12:00 JST |
| **前端发送** | `2026-05-15T23:00:00-04:00` | - | - |
| **数据库存储** | - | `2026-05-16T03:00:00Z` | - |
| **Alice 邮件显示** | May 15, 11:00 PM - 11:30 PM | - | - |
| **Bob 邮件显示** | - | - | 5月16日 12:00 - 12:30 |

### 4.4 代码执行流程详解

#### 步骤 1: 前端时间选择

**代码位置**: `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`

```typescript
// Bob 在东京选择了 "5月16日 12:00" (东京时间)
// 但前端实际存储的是带有时区偏移的 ISO 格式

const mapBookingToMutationInput = ({
  date, // "2026-05-15T23:00:00-04:00" (纽约时间的 ISO 表示)
  duration,
  timeZone, // "Asia/Tokyo" (Bob 选择的时区)
  language, // "ja" (Bob 的语言)
}: BookingOptions) => {
  return {
    start: dayjs(date).format(), // "2026-05-15T23:00:00-04:00"
    end: dayjs(date).add(duration, "minute").format(), // "2026-05-15T23:30:00-04:00"
    timeZone: timeZone, // "Asia/Tokyo"
    language: language, // "ja"
    // ...
  };
};
```

#### 步骤 2: 服务端时间处理

**代码位置**: `packages/features/bookings/lib/service/RegularBookingService.ts`

```typescript
// 服务端收到请求
const handler = (input: BookingHandlerInput, ...) => {
  const { reqBody } = input;
  // reqBody.start = "2026-05-15T23:00:00-04:00"
  // reqBody.timeZone = "Asia/Tokyo"
  // reqBody.language = "ja"

  // 转换为 UTC 存储
  let evt: BuiltCalendarEvent = new CalendarEventBuilder({
    // ...
    startTime: dayjs(reqBody.start).utc().format(),
    // "2026-05-16T03:00:00Z"
    endTime: dayjs(reqBody.end).utc().format(),
    // "2026-05-16T03:30:00Z"
    // ...
  });

  // 组织者信息
  const organizer: Person = {
    timeZone: organizerUser.timeZone, // "America/New_York"
    language: { 
      translate: tOrganizer, 
      locale: organizerUser.locale ?? "en" // "en"
    },
    timeFormat: getTimeFormatStringFromUserTimeFormat(organizerUser.timeFormat), // 12小时
  };

  // 参会者信息
  const invitee: Invitee = [
    {
      timeZone: reqBody.timeZone, // "Asia/Tokyo"
      language: { 
        translate: tAttendees, 
        locale: reqBody.language ?? "en" // "ja"
      },
    },
  ];
};
```

#### 步骤 3: 邮件发送

**代码位置**: `packages/emails/lib/utils/date-formatting.ts`

```typescript
// 给 Alice 的邮件 (组织者)
export function getFormattedDate(calEvent: CalendarEvent, attendee: Person): string {
  // calEvent.startTime = "2026-05-16T03:00:00Z" (UTC)
  // attendee.timeZone = "America/New_York"
  // attendee.language.locale = "en"
  // attendee.language.translate = t (英文翻译函数)
  // calEvent.organizer.timeFormat = "h:mma" (12小时)

  const getFormattedRecipientTime = (time: string, format: string) => {
    return dayjs(time)           // "2026-05-16T03:00:00Z"
      .tz("America/New_York")     // 转换为纽约时间: 2026-05-15 23:00 EDT
      .locale("en")              // 设置英文 locale
      .format(format);
  };

  // 结果:
  // getInviteeStart("h:mma") = "11:00pm"
  // getInviteeEnd("h:mma") = "11:30pm"
  // t("friday") = "Friday"
  // t("may") = "May"

  return "11:00pm - 11:30pm, Friday, May 15, 2026";
}
```

```typescript
// 给 Bob 的邮件 (参会者)
export function getFormattedDate(calEvent: CalendarEvent, attendee: Person): string {
  // calEvent.startTime = "2026-05-16T03:00:00Z" (UTC)
  // attendee.timeZone = "Asia/Tokyo"
  // attendee.language.locale = "ja"
  // attendee.language.translate = t (日文翻译函数)
  // timeFormat = "HH:mm" (24小时)

  const getFormattedRecipientTime = (time: string, format: string) => {
    return dayjs(time)           // "2026-05-16T03:00:00Z"
      .tz("Asia/Tokyo")          // 转换为东京时间: 2026-05-16 12:00 JST
      .locale("ja")              // 设置日文 locale
      .format(format);
  };

  // 结果:
  // getInviteeStart("HH:mm") = "12:00"
  // getInviteeEnd("HH:mm") = "12:30"
  // t("friday") = "金曜日"
  // t("may") = "5月"

  return "12:00 - 12:30, 土曜日, 5月 16, 2026";
}
```

---

## 5. 排障指南

### 5.1 常见问题诊断

#### 问题 1: 时间显示不正确

**可能原因**：
1. 时区选择错误
2. DST 切换问题
3. 前端/后端时区不一致

**诊断步骤**：

```typescript
// 1. 检查时区字符串是否有效
import { normalizeTimezone, isSupportedTimeZone } from "@calcom/lib/dayjs";

const userTimeZone = "America/New_York";
console.log("Is supported:", isSupportedTimeZone(userTimeZone));
console.log("Normalized:", normalizeTimezone(userTimeZone));

// 2. 检查时间转换
import dayjs from "@calcom/dayjs";

const utcTime = "2026-05-16T03:00:00Z";
const nyTime = dayjs(utcTime).tz("America/New_York");
const tokyoTime = dayjs(utcTime).tz("Asia/Tokyo");

console.log("UTC:", utcTime);
console.log("New York:", nyTime.format());
console.log("Tokyo:", tokyoTime.format());

// 3. 检查是否在 DST
console.log("New York in DST:", isInDST(nyTime));
console.log("DST difference:", getDSTDifference("America/New_York"));
```

#### 问题 2: 跨日场景显示错误

**可能原因**：
1. `isNextDayInTimezone`/`isPreviousDayInTimezone` 判断错误
2. 时区排序问题

**诊断步骤**：

```typescript
// 检查跨日判断
import { isNextDayInTimezone, isPreviousDayInTimezone } from "@calcom/lib/dayjs";

const time = "2026-05-16T03:00:00Z";
const nyTz = "America/New_York";
const tokyoTz = "Asia/Tokyo";

// 从纽约看东京时间
console.log("Is Tokyo next day from NY?", isNextDayInTimezone(time, nyTz, tokyoTz));
console.log("Is Tokyo previous day from NY?", isPreviousDayInTimezone(time, nyTz, tokyoTz));

// 从东京看纽约时间
console.log("Is NY next day from Tokyo?", isNextDayInTimezone(time, tokyoTz, nyTz));
console.log("Is NY previous day from Tokyo?", isPreviousDayInTimezone(time, tokyoTz, nyTz));
```

#### 问题 3: 邮件时间与预期不符

**可能原因**：
1. 参会者时区记录错误
2. 组织者时区与参会者时区混淆
3. locale 设置错误

**诊断步骤**：

```typescript
// 检查数据库中的时区信息
// Booking 表: startTime, endTime (UTC)
// Attendee 表: timeZone, locale
// User 表: timeZone, locale, timeFormat

// 检查邮件渲染时的参数
const emailParams = {
  calEvent: {
    startTime: "2026-05-16T03:00:00Z", // UTC
    organizer: {
      timeZone: "America/New_York",
      locale: "en",
      timeFormat: 12,
    },
  },
  attendee: {
    timeZone: "Asia/Tokyo",  // 检查这个值
    language: {
      locale: "ja",           // 检查这个值
      translate: t,
    },
    timeFormat: 24,
  },
};

// 手动验证转换
const formattedForAttendee = getFormattedDate(
  emailParams.calEvent,
  emailParams.attendee
);
console.log("Formatted for attendee:", formattedForAttendee);
```

### 5.2 日志排查要点

**关键日志位置**：

1. **时区规范化日志**
   ```typescript
   // packages/features/calendars/lib/timezone-conversion.ts
   log.info(`Converted offset timezone to IANA format`, { original: timezone, converted });
   log.warn(`Invalid timezone format, falling back to UTC`, { invalidTimezone: timezone });
   ```

2. **预订服务日志**
   ```typescript
   // packages/features/bookings/lib/service/RegularBookingService.ts
   tracingLogger.info("locationBodyString", locationBodyString);
   tracingLogger.info("event type locations", eventType.locations);
   ```

3. **时间槽计算日志**
   ```typescript
   // packages/trpc/server/routers/viewer/slots/util.ts
   log.info("[CACHE HIT] Available slots", { cacheKey });
   log.info("[CACHE MISS] Available slots", { cacheKey, ttl });
   ```

### 5.3 兜底策略验证

| 场景 | 预期行为 | 验证方法 |
|------|----------|----------|
| **无效时区** | 兜底为 UTC | `normalizeTimezone("Invalid/Timezone")` → `"UTC"` |
| **无效 locale** | 兜底为 "en" | `getTranslation(undefined, "common")` 使用英文 |
| **DST 切换日** | 正确计算工作时间 | 检查 `date-ranges.ts` 中的 `offsetDiff` 处理 |
| **跨日预订** | 按接收者时区显示 | 验证邮件中的日期是否正确 |
| **重排继承** | 使用原始预订的时区 | 检查 `attendeeInfoOnReschedule` 逻辑 |

---

## 6. 原有内容回顾

### 6.1 Slot 展示如何受 locale 和 timezone 影响

#### 核心逻辑

Slot 展示的核心逻辑位于 `packages/features/schedules/lib/slots.ts` 中的 `getSlots` 函数。

#### 时区转换关键点

**代码位置**: `packages/features/schedules/lib/slots.ts:138`

```typescript
// Convert to target timezone BEFORE checking if rounding is needed
// This ensures we check minute alignment in the local timezone, not UTC
// This prevents issues with half-hour offset timezones like Asia/Kolkata (GMT+5:30)
slotStartTime = slotStartTime.tz(timeZone);
```

**重要说明**：
- 时间槽计算时，会先将时间转换到目标时区，然后再检查分钟对齐
- 这样可以避免半小时偏移时区（如 Asia/Kolkata GMT+5:30）的问题

### 6.2 Booking 写入如何受 locale 和 timezone 影响

#### 时间存储格式

**关键原则**：所有时间在数据库中存储为 UTC 时间。

**代码位置**: `packages/features/bookings/lib/service/RegularBookingService.ts:1366-1367`

```typescript
let evt: BuiltCalendarEvent = new CalendarEventBuilder({
  // ...
  startTime: dayjs(reqBody.start).utc().format(),
  endTime: dayjs(reqBody.end).utc().format(),
  // ...
});
```

### 6.3 邮件模板如何受 locale 和 timezone 影响

#### 日期格式化工具

**代码位置**: `packages/emails/lib/utils/date-formatting.ts:5-26`

```typescript
export function getFormattedDate(calEvent: CalendarEvent, attendee: Person): string {
  const inviteeTimeFormat = calEvent.organizer.timeFormat || TimeFormat.TWELVE_HOUR;
  const timezone = attendee.timeZone;
  const locale = attendee.language.locale;
  const t = attendee.language.translate;

  const getFormattedRecipientTime = (time: string, format: string) => {
    return dayjs(time).tz(timezone).locale(locale).format(format);
  };
  // ...
}
```

### 6.4 服务端时间计算如何受 locale 和 timezone 影响

#### 可用时间槽计算

**代码位置**: `packages/trpc/server/routers/viewer/slots/util.ts:672-677`

```typescript
private getStartTime(startTimeInput: string, timeZone?: string, minimumBookingNotice?: number) {
  const startTimeMin = dayjs.utc().add(minimumBookingNotice || 1, "minutes");
  const startTime = timeZone === "Etc/GMT" ? dayjs.utc(startTimeInput) : dayjs(startTimeInput).tz(timeZone);

  return startTimeMin.isAfter(startTime) ? startTimeMin.tz(timeZone) : startTime;
}
```

### 6.5 前后端共享日期工具的协作机制

#### 共享日期库

整个代码库使用 `@calcom/dayjs` 作为共享的日期处理库。

#### 时区工具函数

**代码位置**: `packages/lib/dayjs/index.ts`

```typescript
// 时区检测
export const isSupportedTimeZone = (timeZone: string) => { /* ... */ };
export const timeZoneWithDST = (timeZone: string): boolean => { /* ... */ };
export const isInDST = (date: Dayjs) => { /* ... */ };

// 跨日检测
export const isNextDayInTimezone = (time: string, timezoneA: string, timezoneB: string) => { /* ... */ };
export const isPreviousDayInTimezone = (time: string, timezoneA: string, timezoneB: string) => { /* ... */ };

// 时间格式化
export const formatTime = (date: string | Date | Dayjs, timeFormat?: number | null, timeZone?: string | null) => { /* ... */ };
```

### 6.6 数据流总结

#### 时区和 Locale 数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              前端层                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────┐     ┌─────────────────────────────────────────────┐   │
│  │  用户选择时区     │────▶│  useBookerTime Hook                        │   │
│  │  (timeZone)     │     │  - 从 Booker Store 获取时区                  │   │
│  └─────────────────┘     │  - 从用户偏好获取时区                        │   │
│                          │  - 返回最终 timezone 和 timeFormat          │   │
│                          └─────────────────────────────────────────────┘   │
│                                              │                               │
│                                              ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  mapBookingToMutationInput                                           │   │
│  │  - start: dayjs(date).format()  (带时区的 ISO 格式)                 │   │
│  │  - end: dayjs(date).add(duration).format()                          │   │
│  │  - timeZone: 显式传递时区                                             │   │
│  │  - language: 显式传递语言                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                              │                               │
└──────────────────────────────────────────────┼──────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API 层                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  RegularBookingService                                                │   │
│  │                                                                       │   │
│  │  时间存储:                                                            │   │
│  │  - startTime: dayjs(reqBody.start).utc().format()  →  UTC          │   │
│  │  - endTime: dayjs(reqBody.end).utc().format()      →  UTC          │   │
│  │                                                                       │   │
│  │  时区信息保留:                                                        │   │
│  │  - organizer.timeZone: organizerUser.timeZone                        │   │
│  │  - attendee.timeZone: reqBody.timeZone (或原始预订信息)              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                              │                               │
└──────────────────────────────────────────────┼──────────────────────────────┘
                                               │
                                               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据库层                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Booking 表:                                                                │
│  - startTime: TIMESTAMP (UTC)                                              │
│  - endTime: TIMESTAMP (UTC)                                                │
│                                                                             │
│  User 表:                                                                   │
│  - timeZone: string (用户设置的时区)                                        │
│  - locale: string (用户设置的语言)                                          │
│  - timeFormat: int (12 或 24 小时制)                                       │
│                                                                             │
│  Attendee 表:                                                               │
│  - timeZone: string (参会者的时区)                                          │
│  - locale: string (参会者的语言)                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.7 关键设计原则

#### 时间存储原则

1. **统一存储为 UTC**：所有时间在数据库中存储为 UTC 时间戳
2. **时区信息分离存储**：时区信息存储在 User 表和 Attendee 表中
3. **显示时转换**：在前端展示和邮件发送时，根据用户时区转换为本地时间

#### 语言处理原则

1. **Locale 存储**：用户的语言偏好存储在 User 表的 `locale` 字段
2. **翻译文件**：使用 `next-i18next` 管理翻译文件
3. **动态切换**：前端可以通过 `i18n.changeLanguage()` 动态切换语言
4. **邮件翻译**：邮件使用接收者的 locale 进行翻译

#### 时区处理原则

1. **时区规范化**：使用 `normalizeTimezone` 函数验证和规范化时区
2. **半小时时区支持**：在时间槽计算时特别处理半小时偏移时区（如 Asia/Kolkata）
3. **多时区展示**：UI 组件支持显示不同时区的时间
4. **Etc/GMT 特殊处理**：对 `Etc/GMT` 时区使用 UTC 时间

### 6.8 关键文件索引

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 时间槽生成 | `packages/features/schedules/lib/slots.ts` | 核心时间槽计算逻辑 |
| 时区转换 | `packages/features/calendars/lib/timezone-conversion.ts` | 时区规范化和转换 |
| 时区优先级 | `packages/features/bookings/Booker/utils/getBookerTimezone.ts` | Booker 时区选择逻辑 |
| 夏令时检测 | `packages/lib/dayjs/index.ts` | DST 相关工具函数 |
| 跨日检测 | `packages/lib/dayjs/index.ts` | `isNextDayInTimezone` 等函数 |
| 工作时间 DST 处理 | `packages/features/schedules/lib/date-ranges.ts` | DST 切换日的时间调整 |
| 预订服务 | `packages/features/bookings/lib/service/RegularBookingService.ts` | 预订创建核心逻辑 |
| 邮件日期格式化 | `packages/emails/lib/utils/date-formatting.ts` | 邮件中的日期格式化 |
| 多时区展示组件 | `packages/ui/components/popover/MeetingTimeInTimezones.tsx` | 多时区时间对比展示 |
| 前端 Booker 时间 | `packages/features/bookings/Booker/hooks/useBookerTime.ts` | 前端时区获取 Hook |

---

*文档创建时间: 2026-05-04*
*最后更新: 2026-05-04 (添加时区优先级、夏令时、跨日场景、端到端例子)*
