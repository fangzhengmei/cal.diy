# Cal.diy i18n 和 Timezone 分析文档

## 概述

本文档分析了 Cal.diy 代码库中 locale（语言环境）和 timezone（时区）如何影响以下核心功能：
- Slot（时间槽）展示
- Booking（预订）写入
- 邮件模板
- 服务端时间计算

同时也分析了前后端共享日期工具的协作机制。

---

## 1. Slot 展示如何受 locale 和 timezone 影响

### 1.1 核心逻辑

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

#### 时间槽生成流程

1. **获取起始时间**：
   ```typescript
   const startTimeWithMinNotice = dayjs.utc().add(minimumBookingNotice, "minute");
   ```
   从当前 UTC 时间加上最小预订通知时间

2. **时区转换**：
   ```typescript
   slotStartTime = slotStartTime.tz(timeZone);
   ```
   将时间转换到目标时区

3. **分钟对齐检查**：
   ```typescript
   if (slotStartTime.minute() % interval !== 0) {
     slotStartTime = getCorrectedSlotStartTime({...});
   }
   ```
   在目标时区检查分钟是否对齐

4. **时间槽边界处理**：
   ```typescript
   while (!slotStartTime.add(eventLength, "minutes").subtract(1, "second").utc().isAfter(range.end)) {
     // 生成时间槽
   }
   ```
   使用 UTC 时间进行边界比较

### 1.2 时区来源

时区通过 `getTimeZone(inviteeDate)` 函数获取，该函数从 `dayjs` 对象中提取时区信息。

**代码位置**: `packages/features/schedules/lib/slots.ts:255`

```typescript
const getSlots = ({
  inviteeDate,
  // ...其他参数
}: GetSlots): {
  // ...
}[] => {
  return buildSlotsWithDateRanges({
    // ...
    timeZone: getTimeZone(inviteeDate),
    // ...
  });
};
```

### 1.3 时区规范化

在 `packages/features/calendars/lib/timezone-conversion.ts` 中提供了时区规范化函数：

```typescript
/**
 * Validates and normalizes a timezone string to a valid IANA timezone.
 * Converts offset formats to Etc/GMT, falls back to UTC if invalid.
 */
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

**功能**：
- 验证时区字符串是否为有效的 IANA 时区
- 将偏移格式（如 GMT+5:30）转换为 Etc/GMT 格式
- 无效时区默认为 UTC

---

## 2. Booking 写入如何受 locale 和 timezone 影响

### 2.1 核心流程

Booking 写入的核心逻辑位于 `packages/features/bookings/lib/service/RegularBookingService.ts`。

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

#### 组织者和参会者时区处理

**代码位置**: `packages/features/bookings/lib/service/RegularBookingService.ts:1167-1176`

```typescript
const invitee: Invitee = [
  {
    email: bookerEmail,
    name: fullName,
    phoneNumber: bookerPhoneNumber,
    firstName: (typeof bookerName === "object" && bookerName.firstName) || "",
    lastName: (typeof bookerName === "object" && bookerName.lastName) || "",
    timeZone: attendeeTimezone,
    language: { translate: tAttendees, locale: attendeeLanguage ?? "en" },
  },
];
```

#### 组织者时区

**代码位置**: `packages/features/bookings/lib/service/RegularBookingService.ts:1369-1378`

```typescript
organizer: {
  id: organizerUser.id,
  name: organizerUser.name || "Nameless",
  email: organizerEmail,
  username: organizerUser.username || undefined,
  usernameInOrg: organizerOrganizationProfile?.username || undefined,
  timeZone: organizerUser.timeZone,
  language: { translate: tOrganizer, locale: organizerUser.locale ?? "en" },
  timeFormat: getTimeFormatStringFromUserTimeFormat(organizerUser.timeFormat),
},
```

#### 参会者时区和语言的确定

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

**规则**：
- 如果是组织者自己重新预订，使用原始预订中参会者的时区和语言
- 否则使用请求中提供的时区和语言
- 默认语言为 "en"

### 2.2 前端到后端的时间映射

前端将预订数据转换为 mutation 输入的逻辑位于 `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx`。

**代码位置**: `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx:35-94`

```typescript
export const mapBookingToMutationInput = ({
  values,
  event,
  date,
  duration,
  timeZone,
  language,
  // ...其他参数
}: BookingOptions): BookingCreateBody => {
  // ...
  return {
    ...values,
    user: username,
    start: dayjs(date).format(),
    end: dayjs(date)
      .add(duration || event.length, "minute")
      .format(),
    eventTypeId: event.id,
    eventTypeSlug: event.slug,
    timeZone: timeZone,
    language: language,
    // ...
  };
};
```

**关键点**：
- `start` 和 `end` 使用 `dayjs(date).format()` 格式化（带有时区信息的 ISO 格式）
- `timeZone` 字段显式传递
- `language` 字段显式传递

---

## 3. 邮件模板如何受 locale 和 timezone 影响

### 3.1 日期格式化工具

邮件模板中的日期格式化核心逻辑位于 `packages/emails/lib/utils/date-formatting.ts`。

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

  const getInviteeStart = (format: string) => {
    return getFormattedRecipientTime(calEvent.startTime, format);
  };

  const getInviteeEnd = (format: string) => {
    return getFormattedRecipientTime(calEvent.endTime, format);
  };

  return `${getInviteeStart(inviteeTimeFormat)} - ${getInviteeEnd(inviteeTimeFormat)}, ${t(
    getInviteeStart("dddd").toLowerCase()
  )}, ${t(getInviteeStart("MMMM").toLowerCase())} ${getInviteeStart("D, YYYY")}`;
}
```

**关键流程**：
1. 从 `attendee` 对象获取时区 (`timeZone`)、语言环境 (`locale`) 和翻译函数 (`translate`)
2. 使用 `dayjs(time).tz(timezone).locale(locale).format(format)` 进行格式化
3. 使用翻译函数翻译星期和月份名称

### 3.2 邮件组件中的时间展示

`WhenInfo` 组件负责在邮件中展示时间信息。

**代码位置**: `packages/emails/src/components/WhenInfo.tsx:34-74`

```typescript
export function WhenInfo(props: {
  calEvent: CalendarEvent;
  timeZone: string;
  t: TFunction;
  locale: string;
  timeFormat: TimeFormat;
}) {
  const { timeZone, t, calEvent: { recurringEvent } = {}, locale, timeFormat } = props;

  function getRecipientStart(format: string) {
    return dayjs(props.calEvent.startTime).tz(timeZone).locale(locale).format(format);
  }

  function getRecipientEnd(format: string) {
    return dayjs(props.calEvent.endTime).tz(timeZone).locale(locale).format(format);
  }

  // ...

  return (
    <div>
      <Info
        // ...
        description={
          <span data-testid="when">
            {recurringEvent?.count ? `${t("starting")} ` : ""}
            {getRecipientStart(`dddd, LL | ${timeFormat}`)} - {getRecipientEnd(timeFormat)}{" "}
            <span style={{ color: "#4B5563" }}>({timeZone})</span>
          </span>
        }
        // ...
      />
    </div>
  );
}
```

### 3.3 基础邮件模板

`BaseScheduledEmail` 组件是所有预订相关邮件的基础。

**代码位置**: `packages/emails/src/templates/BaseScheduledEmail.tsx:18-151`

```typescript
export const BaseScheduledEmail = (
  props: {
    calEvent: CalendarEvent;
    attendee: Person;
    timeZone: string;
    // ...
    t: TFunction;
    locale: string;
    timeFormat: TimeFormat | undefined;
    // ...
  }
) => {
  const { t, timeZone, locale, timeFormat: timeFormat_ } = props;
  const timeFormat = timeFormat_ ?? TimeFormat.TWELVE_HOUR;

  function getRecipientStart(format: string) {
    return dayjs(props.calEvent.startTime).tz(timeZone).format(format);
  }

  function getRecipientEnd(format: string) {
    return dayjs(props.calEvent.endTime).tz(timeZone).format(format);
  }

  // 邮件主题包含格式化的日期
  const subject = t(props.subject || "confirmed_event_type_subject", {
    eventType: props.calEvent.type,
    name: props.calEvent.team?.name || props.calEvent.organizer.name,
    date: `${getRecipientStart("h:mma")} - ${getRecipientEnd("h:mma")}, ${t(
      getRecipientStart("dddd").toLowerCase()
    )}, ${t(getRecipientStart("MMMM").toLowerCase())} ${getRecipientStart("D, YYYY")}`,
    interpolation: { escapeValue: false },
  });

  // ...

  return (
    <BaseEmailHtml
      // ...
      >
      {/* ... */}
      <WhenInfo timeFormat={timeFormat} calEvent={props.calEvent} t={t} timeZone={timeZone} locale={locale} />
      {/* ... */}
    </BaseEmailHtml>
  );
};
```

### 3.4 参会者邮件模板

`AttendeeScheduledEmail` 组件专门用于参会者的预订确认邮件。

**代码位置**: `packages/emails/src/templates/AttendeeScheduledEmail.tsx:5-20`

```typescript
export const AttendeeScheduledEmail = (
  props: {
    calEvent: CalendarEvent;
    attendee: Person;
  } & Partial<React.ComponentProps<typeof BaseScheduledEmail>>
) => {
  return (
    <BaseScheduledEmail
      locale={props.attendee.language.locale}
      timeZone={props.attendee.timeZone}
      t={props.attendee.language.translate}
      timeFormat={props.attendee?.timeFormat}
      {...props}
    />
  );
};
```

**关键点**：
- 使用参会者的 `locale` 作为语言环境
- 使用参会者的 `timeZone` 作为时区
- 使用参会者的 `translate` 函数进行翻译
- 使用参会者的 `timeFormat` 作为时间格式（12小时或24小时）

---

## 4. 服务端时间计算如何受 locale 和 timezone 影响

### 4.1 可用时间槽计算

服务端计算可用时间槽的核心逻辑位于 `packages/trpc/server/routers/viewer/slots/util.ts` 中的 `AvailableSlotsService` 类。

#### 时区处理流程

**代码位置**: `packages/trpc/server/routers/viewer/slots/util.ts:672-677`

```typescript
private getStartTime(startTimeInput: string, timeZone?: string, minimumBookingNotice?: number) {
  const startTimeMin = dayjs.utc().add(minimumBookingNotice || 1, "minutes");
  const startTime = timeZone === "Etc/GMT" ? dayjs.utc(startTimeInput) : dayjs(startTimeInput).tz(timeZone);

  return startTimeMin.isAfter(startTime) ? startTimeMin.tz(timeZone) : startTime;
}
```

**规则**：
- 如果时区是 `Etc/GMT`，使用 UTC 时间
- 否则将输入时间转换到目标时区
- 如果计算出的时间早于最小预订通知时间，则使用最小预订通知时间

#### 结束时间处理

**代码位置**: `packages/trpc/server/routers/viewer/slots/util.ts:949-950`

```typescript
const endTime =
  input.timeZone === "Etc/GMT" ? dayjs.utc(input.endTime) : dayjs(input.endTime).utc().tz(input.timeZone);
```

#### 日期范围过滤

**代码位置**: `packages/trpc/server/routers/viewer/slots/util.ts:274-313`

```typescript
private _filterSlotsByRequestedDateRange<
  T extends Record<string, { time: string; attendees?: number; bookingUid?: string }[]>,
>({
  slotsMappedToDate,
  startTime,
  endTime,
  timeZone,
}: {
  slotsMappedToDate: T;
  startTime: string;
  endTime: string;
  timeZone: string | undefined;
}): T {
  if (!timeZone) {
    return slotsMappedToDate;
  }
  const inputStartTime = dayjs(startTime).tz(timeZone);
  const inputEndTime = dayjs(endTime).tz(timeZone);

  // fr-CA uses YYYY-MM-DD format
  const formatter = new Intl.DateTimeFormat("fr-CA", {
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    timeZone: timeZone,
  });

  const allowedDates = new Set<string>();
  for (let d = inputStartTime.startOf("day"); !d.isAfter(inputEndTime, "day"); d = d.add(1, "day")) {
    allowedDates.add(formatter.format(d.toDate()));
  }

  // 过滤掉不在允许日期范围内的时间槽
  // ...
}
```

**关键点**：
- 使用 `Intl.DateTimeFormat` 并指定 `timeZone` 来格式化日期
- 使用 `fr-CA` 语言环境来获取 `YYYY-MM-DD` 格式
- 在目标时区中计算日期范围

### 4.2 时间边界检查

时间边界检查逻辑位于 `packages/lib/isOutOfBounds.tsx`。

**代码位置**: `packages/trpc/server/routers/viewer/slots/util.ts:1341-1366`

```typescript
const filteredSlots = slots.filter((slot) => {
  const isFutureLimitViolationForTheSlot = isTimeViolatingFutureLimit({
    time: slot.time,
    periodLimits,
  });

  let isOutOfBounds = false;
  try {
    isOutOfBounds = isTimeOutOfBounds({
      time: slot.time,
      minimumBookingNotice: eventType.minimumBookingNotice,
    });
  } catch (error) {
    if (error instanceof BookingDateInPastError) {
      throw new TRPCError({
        code: "BAD_REQUEST",
        message: error.message,
      });
    }
    throw error;
  }

  if (isFutureLimitViolationForTheSlot) {
    foundAFutureLimitViolation = true;
  }

  return (
    !isFutureLimitViolationForTheSlot &&
    !isOutOfBounds
  );
});
```

### 4.3 周期限制计算

**代码位置**: `packages/trpc/server/routers/viewer/slots/util.ts:1311-1320`

```typescript
const eventTimeZone =
  eventType.timeZone || eventType?.schedule?.timeZone || allUsersAvailability?.[0]?.timeZone;

const eventUtcOffset = getUTCOffsetByTimezone(eventTimeZone) ?? 0;
const bookerUtcOffset = input.timeZone ? (getUTCOffsetByTimezone(input.timeZone) ?? 0) : 0;
const periodLimits = calculatePeriodLimits({
  periodType: eventType.periodType,
  periodDays: eventType.periodDays,
  periodCountCalendarDays: eventType.periodCountCalendarDays,
  periodStartDate: eventType.periodStartDate,
  periodEndDate: eventType.periodEndDate,
  allDatesWithBookabilityStatusInBookerTz: allDatesWithBookabilityStatus,
  eventUtcOffset,
  bookerUtcOffset,
});
```

**关键点**：
- 计算事件时区的 UTC 偏移
- 计算预订者时区的 UTC 偏移
- 将两个偏移传递给 `calculatePeriodLimits` 函数

---

## 5. 前后端共享日期工具的协作机制

### 5.1 共享日期库

整个代码库使用 `@calcom/dayjs` 作为共享的日期处理库。

#### 时区工具函数

时区相关的工具函数位于 `packages/lib/timezone.ts`。

**代码位置**: `packages/lib/timezone.ts:1-30`

```typescript
import type { ITimezoneOption } from "react-timezone-select";
import dayjs from "@calcom/dayjs";
import isProblematicTimezone from "./isProblematicTimezone";

export type Timezones = { label: string; timezone: string }[];

// 时区搜索过滤
const searchTextFilter = (tzOption: Timezones[number], searchText: string) => {
  return searchText && tzOption.label.toLowerCase().includes(searchText.toLowerCase());
};

export const filterBySearchText = (searchText: string, timezones: Timezones) => {
  return timezones.filter((tzOption) => searchTextFilter(tzOption, searchText));
};

// 时区下拉菜单处理
export const addTimezonesToDropdown = (timezones: Timezones) => {
  return Object.fromEntries(
    timezones
      .filter(({ timezone }) => {
        return timezone !== null && !isProblematicTimezone(timezone);
      })
      .map(({ label, timezone }) => [timezone, label])
  );
};

// 时区选项标签格式化
const formatOffset = (offset: string) =>
  offset.replace(/^([-+])(0)(\d):00$/, (_, sign, _zero, hour) => `${sign}${hour}:00`);

export const handleOptionLabel = (option: ITimezoneOption, timezones: Timezones) => {
  const offsetUnit = option.label.split(/[-+]/)[0].substring(1);
  const cityName = option.label.split(") ")[1];

  const timezoneValue = ` ${offsetUnit} ${formatOffset(dayjs.tz(undefined, option.value).format("Z"))}`;
  return timezones.length > 0
    ? `${cityName}${timezoneValue}`
    : `${option.value.replace(/_/g, " ")}${timezoneValue}`;
};
```

### 5.2 前端 Booker 时间处理

前端 Booker 组件的时间处理位于 `packages/features/bookings/Booker/hooks/useBookerTime.ts`。

**代码位置**: `packages/features/bookings/Booker/hooks/useBookerTime.ts:1-20`

```typescript
import { useBookerStoreContext } from "@calcom/features/bookings/Booker/BookerStoreProvider";
import { getBookerTimezone } from "@calcom/features/bookings/Booker/utils/getBookerTimezone";
import { useTimePreferences } from "@calcom/features/bookings/lib/timePreferences";
import { shallow } from "zustand/shallow";

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

**时区优先级**：
1. 首先尝试从 Booker Store 获取时区
2. 然后尝试从用户时间偏好获取时区
3. 使用 `getBookerTimezone` 函数确定最终时区

### 5.3 多时区时间展示

前端组件 `MeetingTimeInTimezones` 用于展示不同时区的会议时间。

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

  // 渲染多时区时间展示
  // ...
};
```

**使用的工具函数**（来自 `@calcom/lib/dayjs`）：
- `isSupportedTimeZone`: 检查时区是否支持
- `isPreviousDayInTimezone`: 检查在目标时区是否是前一天
- `isNextDayInTimezone`: 检查在目标时区是否是后一天
- `formatTime`: 格式化时间
- `sortByTimezone`: 按时区排序

### 5.4 i18n 服务端处理

服务端 i18n 处理位于 `packages/trpc/server/routers/viewer/i18n/i18n.handler.ts`。

**代码位置**: `packages/trpc/server/routers/viewer/i18n/i18n.handler.ts:1-18`

```typescript
import i18nConfig from "@calcom/i18n/next-i18next.config";
import type { I18nInputSchema } from "./i18n.schema";

type I18nOptions = {
  input: I18nInputSchema;
};

export const i18nHandler = async ({ input }: I18nOptions) => {
  const { locale } = input;
  const { serverSideTranslations } = await import("next-i18next/serverSideTranslations");
  const i18n = await serverSideTranslations(locale, ["common", "vital"], i18nConfig);

  return {
    i18n,
    locale,
  };
};

export default i18nHandler;
```

**功能**：
- 使用 `next-i18next` 的 `serverSideTranslations` 函数
- 加载指定 locale 的翻译文件（`common` 和 `vital` 命名空间）
- 返回翻译数据和 locale 信息

### 5.5 前端 i18n Hook

前端使用 `useLocale` Hook 来获取国际化上下文。

**代码位置**: `packages/platform/atoms/lib/useLocale.ts:1-26`

```typescript
import type { TFunction, i18n } from "i18next";
import { useTranslation } from "react-i18next";

import { useAtomsContext } from "@calcom/atoms/hooks/useAtomsContext";

type useLocaleReturnType = {
  i18n: i18n;
  t: TFunction;
  isLocaleReady: boolean;
};

export const useLocale = (
  namespace: Parameters<typeof useTranslation>[0] = "common"
): useLocaleReturnType => {
  const context = useAtomsContext();
  const { i18n, t } = useTranslation(namespace);
  const isLocaleReady = Object.keys(i18n).length > 0;
  if (context?.clientId) {
    return { i18n: context.i18n, t: context.t, isLocaleReady: true } as unknown as useLocaleReturnType;
  }
  return {
    i18n,
    t,
    isLocaleReady,
  };
};
```

**功能**：
- 使用 `react-i18next` 的 `useTranslation` Hook
- 支持从 Atoms Context 覆盖 i18n 上下文（用于平台模式）
- 返回 i18n 实例、翻译函数和就绪状态

---

## 6. 数据流总结

### 6.1 时区和 Locale 数据流

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
│  │  - organizer.language.locale: organizerUser.locale                  │   │
│  │  - attendee.language.locale: reqBody.language                       │   │
│  │  - organizer.timeFormat: getTimeFormatStringFromUserTimeFormat()    │   │
│  │  - attendee.timeFormat: (如果有的话)                                 │   │
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
│  - (时区信息存储在 User 表和 Attendee 表中)                                 │
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

### 6.2 邮件发送时的时区转换

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         邮件发送流程                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 从数据库读取 Booking                                                     │
│     - startTime: UTC 时间戳                                                 │
│     - endTime: UTC 时间戳                                                   │
│                                                                             │
│  2. 构建 CalendarEvent 对象                                                 │
│     - calEvent.startTime: UTC 时间字符串                                    │
│     - calEvent.endTime: UTC 时间字符串                                      │
│     - calEvent.organizer.timeZone: 组织者时区                               │
│     - calEvent.organizer.language.locale: 组织者语言                       │
│     - calEvent.organizer.timeFormat: 组织者时间格式                         │
│                                                                             │
│  3. 确定参会者信息                                                           │
│     - attendee.timeZone: 参会者时区                                         │
│     - attendee.language.locale: 参会者语言                                 │
│     - attendee.language.translate: 翻译函数                                 │
│     - attendee.timeFormat: 参会者时间格式                                   │
│                                                                             │
│  4. 邮件模板渲染                                                             │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │  BaseScheduledEmail / AttendeeScheduledEmail                    │   │
│     │                                                                   │   │
│     │  输入参数:                                                        │   │
│     │  - locale: attendee.language.locale                             │   │
│     │  - timeZone: attendee.timeZone                                   │   │
│     │  - t: attendee.language.translate                                │   │
│     │  - timeFormat: attendee.timeFormat                               │   │
│     │                                                                   │   │
│     │  日期格式化 (WhenInfo 组件):                                      │   │
│     │  dayjs(calEvent.startTime)                                       │   │
│     │    .tz(timeZone)         ← 转换到参会者时区                       │   │
│     │    .locale(locale)      ← 设置语言环境                           │   │
│     │    .format(format)       ← 格式化输出                             │   │
│     │                                                                   │   │
│     │  翻译处理:                                                        │   │
│     │  t(getRecipientStart("dddd").toLowerCase())  ← 翻译星期          │   │
│     │  t(getRecipientStart("MMMM").toLowerCase())  ← 翻译月份          │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 关键设计原则

### 7.1 时间存储原则

1. **统一存储为 UTC**：所有时间在数据库中存储为 UTC 时间戳
2. **时区信息分离存储**：时区信息存储在 User 表和 Attendee 表中
3. **显示时转换**：在前端展示和邮件发送时，根据用户时区转换为本地时间

### 7.2 语言处理原则

1. **Locale 存储**：用户的语言偏好存储在 User 表的 `locale` 字段
2. **翻译文件**：使用 `next-i18next` 管理翻译文件
3. **动态切换**：前端可以通过 `i18n.changeLanguage()` 动态切换语言
4. **邮件翻译**：邮件使用接收者的 locale 进行翻译

### 7.3 时区处理原则

1. **时区规范化**：使用 `normalizeTimezone` 函数验证和规范化时区
2. **半小时时区支持**：在时间槽计算时特别处理半小时偏移时区（如 Asia/Kolkata）
3. **多时区展示**：UI 组件支持显示不同时区的时间
4. **Etc/GMT 特殊处理**：对 `Etc/GMT` 时区使用 UTC 时间

### 7.4 前后端协作原则

1. **共享日期库**：前后端都使用 `@calcom/dayjs`
2. **统一格式**：使用 ISO 8601 格式传递时间（带时区信息）
3. **服务端验证**：服务端对所有时间进行验证和边界检查
4. **类型安全**：使用 TypeScript 确保类型一致性

---

## 8. 关键文件索引

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 时间槽生成 | `packages/features/schedules/lib/slots.ts` | 核心时间槽计算逻辑 |
| 时区转换 | `packages/features/calendars/lib/timezone-conversion.ts` | 时区规范化和转换 |
| 预订服务 | `packages/features/bookings/lib/service/RegularBookingService.ts` | 预订创建核心逻辑 |
| 前端时间映射 | `packages/features/bookings/lib/client/booking-event-form/booking-to-mutation-input-mapper.tsx` | 前端到后端的数据映射 |
| 邮件日期格式化 | `packages/emails/lib/utils/date-formatting.ts` | 邮件中的日期格式化 |
| 邮件时间组件 | `packages/emails/src/components/WhenInfo.tsx` | 邮件中的时间展示组件 |
| 基础邮件模板 | `packages/emails/src/templates/BaseScheduledEmail.tsx` | 预订邮件基础模板 |
| 可用时间槽计算 | `packages/trpc/server/routers/viewer/slots/util.ts` | 服务端时间槽计算 |
| 时区工具 | `packages/lib/timezone.ts` | 时区相关工具函数 |
| 前端 Booker 时间 | `packages/features/bookings/Booker/hooks/useBookerTime.ts` | 前端时区获取 Hook |
| 多时区展示 | `packages/ui/components/popover/MeetingTimeInTimezones.tsx` | 多时区时间展示组件 |
| 服务端 i18n | `packages/trpc/server/routers/viewer/i18n/i18n.handler.ts` | 服务端 i18n 处理 |
| 前端 i18n Hook | `packages/platform/atoms/lib/useLocale.ts` | 前端 i18n Hook |

---

## 9. 常见问题和注意事项

### 9.1 时区问题

1. **半小时时区**：如 Asia/Kolkata (GMT+5:30)，需要在时间槽计算时特别处理
2. **Etc/GMT 系列**：Etc/GMT+5 实际上是 UTC-5（符号相反）
3. **夏令时**：使用 IANA 时区（如 America/New_York）而非固定偏移，因为 IANA 时区包含夏令时信息

### 9.2 时间格式问题

1. **12小时 vs 24小时**：用户可以选择时间格式，需要在展示时尊重用户偏好
2. **日期格式**：不同 locale 有不同的日期格式习惯

### 9.3 跨时区预订

1. **组织者和参会者时区不同**：邮件会分别使用各自的时区显示时间
2. **时间槽展示**：时间槽按预订者的时区显示
3. **数据库存储**：始终使用 UTC，避免歧义

### 9.4 i18n 最佳实践

1. **所有 UI 字符串都要翻译**：不要硬编码英文文本
2. **使用翻译键**：使用语义化的键名，如 `confirmed_event_type_subject`
3. **插值处理**：使用 `t(key, { variable: value })` 进行变量插值
4. **复数形式**：利用 i18next 的复数支持

---

## 10. 测试建议

### 10.1 时区测试

1. 测试不同时区的用户预订
2. 测试半小时偏移时区（如 Asia/Kolkata）
3. 测试跨日期的预订（如在一个时区是 23:00，在另一个时区是第二天 02:00）
4. 测试夏令时切换期间的预订

### 10.2 i18n 测试

1. 测试不同语言的界面展示
2. 测试 RTL（右到左）语言（如阿拉伯语、希伯来语）
3. 测试日期格式在不同语言中的显示
4. 测试邮件内容的翻译

---

*文档创建时间: 2026-05-04*
*最后更新: 2026-05-04*
