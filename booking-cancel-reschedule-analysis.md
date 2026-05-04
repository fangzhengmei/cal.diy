# 预约取消与重排流程分析

## 一、取消预约流程

### 1.1 核心流程

取消预约的主流程在 `packages/features/bookings/lib/handleCancelBooking.ts` 中实现，主要步骤如下：

1. **权限与状态验证**
   - 检查预约是否已取消（`BookingStatus.CANCELLED`）
   - 验证用户是否有权取消该预约
   - 检查预约是否已结束

2. **座位预订特殊处理**
   - 如果是座位预订且提供了 `seatReferenceUid`，走 `cancelAttendeeSeat` 流程
   - 只取消单个参会人的座位，不影响其他参会人

3. **更新预订状态**
   - 将预订状态更新为 `CANCELLED`
   - 记录取消原因和取消人
   - 对于循环预约，可选择取消所有剩余预约

4. **退款处理**
   - 根据支付方式和退款政策处理退款
   - 详见 1.2 退款分支逻辑

5. **外部日历同步**
   - 通过 `EventManager.cancelEvent()` 删除外部日历事件
   - 详见 1.3 外部日历更新机制

6. **发送通知**
   - 发送取消通知邮件和短信给所有参会人
   - 详见 1.4 参会人通知流程

### 1.2 退款分支逻辑

退款处理的核心逻辑在 `packages/features/bookings/lib/payment/processPaymentRefund.ts` 中。

#### 触发条件

在 `handleCancelBooking.ts:396-414` 中：

```typescript
if (bookingToDelete.payment.some((payment) => payment.paymentOption === "ON_BOOKING")) {
  try {
    await processPaymentRefund({
      booking: bookingToDelete,
    });
  } catch (error) {
    log.error(`Error processing payment refund for booking ${bookingToDelete.uid}:`, error);
  }
} else if (bookingToDelete.payment.some((payment) => payment.paymentOption === "HOLD")) {
  try {
    await processNoShowFeeOnCancellation({
      booking: bookingToDelete,
      payments: bookingToDelete.payment,
      cancelledByUserId: userId,
    });
  } catch (error) {
    log.error(`Error processing no-show fee for booking ${bookingToDelete.uid}:`, error);
  }
}
```

#### 退款政策判断

在 `processPaymentRefund.ts:39-82` 中：

1. **检查是否有成功支付**
   ```typescript
   const successPayment = payment.find((p) => p.success);
   if (!successPayment) return;
   ```

2. **检查退款政策**
   ```typescript
   if (!appData?.refundPolicy || appData.refundPolicy === RefundPolicy.NEVER) return;
   ```

3. **检查退款期限（如果政策为 DAYS）**
   ```typescript
   if (refundPolicy === RefundPolicy.DAYS && refundDaysCount) {
     const refundDeadline = refundCountCalendarDays === true
       ? dayjs(startTime).subtract(refundDaysCount, "days")
       : dayjs(startTime).businessDaysSubtract(refundDaysCount);
     if (dayjs().isAfter(refundDeadline)) return;
   }
   ```

4. **执行退款**
   ```typescript
   await handlePaymentRefund(successPayment.id, paymentAppCredential);
   ```

#### 退款分支总结

| 支付选项 | 处理方式 | 退款政策影响 |
|---------|---------|-------------|
| `ON_BOOKING` | 调用 `processPaymentRefund()` | 受 `RefundPolicy` 约束 |
| `HOLD` | 调用 `processNoShowFeeOnCancellation()` | 处理预授权扣款 |

### 1.3 外部日历更新机制

外部日历更新通过 `EventManager` 类实现，位置在 `packages/features/bookings/lib/EventManager.ts`。

#### 取消时的日历更新

在 `handleCancelBooking.ts:454-463` 中：

```typescript
const eventManager = new EventManager(
  { ...bookingToDelete.user, credentials },
  bookingToDeleteEventTypeMetadata?.apps
);

await eventManager.cancelEvent(evt, bookingToDelete.references, isBookingInRecurringSeries);
```

#### `cancelEvent` 方法实现

在 `EventManager.ts:777-851` 中：

1. **遍历所有 booking references**
   ```typescript
   const allPromises: Promise<unknown>[] = [];
   for (const reference of bookingReferences) {
     const credential = getCredential({ id: reference, allCredentials: this.calendarCredentials });
     
     if (reference.type.includes("_calendar")) {
       const calendar = await getCalendar(credential, "booking");
       if (calendar) {
         allPromises.push(calendar.deleteEvent(reference.uid, reference.externalCalendarId));
       }
     }
     
     if (reference.type.includes("_video")) {
       allPromises.push(deleteMeeting(credential, reference));
     }
   }
   ```

2. **使用 `Promise.allSettled` 确保所有操作都能执行**
   ```typescript
   (await Promise.allSettled(allPromises)).some((result) => {
     if (result.status === "rejected") {
       log.warn("Error deleting calendar event or video meeting for booking", safeStringify({ error: result.reason }));
     }
   });
   ```

#### 关键点

- **跳过同步的情况**：当取消来自日历订阅 webhook 时（`skipCalendarSyncTaskCancellation = true`），避免无限循环
- **软错误处理**：删除操作失败不会阻止取消流程继续，只是记录警告日志
- **标记引用为删除**：无论是否成功删除外部事件，都会将 `bookingReferences` 标记为 `deleted: true`

### 1.4 参会人通知流程

取消预约时的通知发送在 `handleCancelBooking.ts:493-503` 中：

```typescript
try {
  if (!platformClientId || (platformClientId && arePlatformEmailsEnabled))
    await sendCancelledEmailsAndSMS(
      evt,
      { eventName: bookingToDelete?.eventType?.eventName },
      bookingToDelete?.eventType?.metadata as EventTypeMetadata
    );
} catch (error) {
  log.error("Error deleting event", error);
}
```

#### 单个座位取消的通知

在 `cancelAttendeeSeat.ts:138-147` 中：

```typescript
const tAttendees = await getTranslation(attendee.locale ?? "en", "common");

await sendCancelledSeatEmailsAndSMS(
  evt,
  {
    ...attendee,
    language: { translate: tAttendees, locale: attendee.locale ?? "en" },
  },
  eventTypeMetadata
);
```

#### 通知函数位置

- `sendCancelledEmailsAndSMS`：`packages/emails/email-manager.ts`
- `sendCancelledSeatEmailsAndSMS`：`packages/emails/email-manager.ts`

---

## 二、重排预约流程

### 2.1 座位预订（Seated Booking）重排

座位预订的重排逻辑在 `packages/features/bookings/lib/handleSeats/reschedule/` 目录下。

#### 重排入口

在 `rescheduleSeatedBooking.ts:20-150` 中：

```typescript
const rescheduleSeatedBooking = async (
  rescheduleSeatedBookingObject: RescheduleSeatedBookingObject,
  seatedBooking: SeatedBooking,
  resultBooking: HandleSeatsResultBooking | null,
  loggerWithEventDetails: ReturnType<typeof createLoggerWithEventDetails>
) => {
  // 检查新时间段是否已有预订
  const newTimeSlotBooking = await prisma.booking.findFirst({
    where: {
      startTime: dayjs(evt.startTime).toDate(),
      eventTypeId: eventType.id,
      status: BookingStatus.ACCEPTED,
    },
    select: { ... }
  });

  // 判断是组织者还是参会人重排
  if (!bookingSeat) {
    // 组织者重排
    resultBooking = await ownerRescheduleSeatedBooking(...);
  }

  const seatAttendee: SeatAttendee | null = bookingSeat?.attendee || null;
  if (seatAttendee) {
    // 参会人重排
    resultBooking = await attendeeRescheduleSeatedBooking(...);
  }
};
```

### 2.2 组织者重排（Owner Reschedule）

组织者重排的逻辑在 `ownerRescheduleSeatedBooking.ts` 中。

#### 两种情况

```typescript
const ownerRescheduleSeatedBooking = async (...) => {
  // 如果新时间段没有预订，更新当前预订到新日期
  if (!newTimeSlotBooking) {
    resultBooking = await moveSeatedBookingToNewTimeSlot(...);
  } else {
    // 如果新时间段已有预订，合并两个预订
    resultBooking = await combineTwoSeatedBookings(...);
  }
  return resultBooking;
};
```

#### 情况1：新时间段无预订（复用旧记录）

在 `moveSeatedBookingToNewTimeSlot.ts:45-126` 中：

```typescript
const moveSeatedBookingToNewTimeSlot = async (...) => {
  // 直接更新现有 booking 的时间
  const newBooking = await updateBooking({
    bookingId: seatedBooking.id,
    startTime: evt.startTime,
    endTime: evt.endTime,
    cancellationReason: rescheduleReason,
  });

  // 更新外部日历
  const updateManager = await eventManager.reschedule(copyEvent, rescheduleUid, newBooking.id);

  // 发送重排通知
  if (noEmail !== true && isConfirmedByDefault) {
    await sendRescheduledEmailsAndSMS(
      {
        ...copyEvent,
        additionalNotes,
        cancellationReason: `$RCH$${rescheduleReason ? rescheduleReason : ""}`,
      },
      eventType.metadata
    );
  }
};
```

**关键点**：
- **复用旧记录**：直接更新现有 booking 的 `startTime`、`endTime`、`cancellationReason`
- **不创建新记录**：保持预订 ID 不变
- **iCalUID 保持不变**：日历事件只更新时间，不创建新事件

#### 情况2：新时间段已有预订（合并预订）

在 `combineTwoSeatedBookings.ts:16-163` 中：

```typescript
const combineTwoSeatedBookings = async (...) => {
  // 处理参会人：移动或删除
  const attendeesToMove = [], attendeesToDelete = [];
  
  for (const attendee of seatedBooking.attendees) {
    if (newTimeSlotBooking.attendees.some((newBookingAttendee) => 
      newBookingAttendee.email === attendee.email
    )) {
      // 该参会人已在新预订中，删除旧记录
      attendeesToDelete.push(attendee.id);
    } else {
      // 移动参会人到新预订
      attendeesToMove.push({ id: attendee.id, seatReferenceId: attendee.bookingSeat?.id });
    }
  }

  // 检查座位是否足够
  if (!eventType.seatsPerTimeSlot ||
      attendeesToMove.length + newTimeSlotBooking.attendees.filter(
        (attendee) => attendee.bookingSeat
      ).length > eventType.seatsPerTimeSlot
  ) {
    throw new HttpError({ statusCode: 409, message: ErrorCode.NotEnoughAvailableSeats });
  }

  // 执行数据库操作
  await prisma.$transaction([
    ...moveAttendeeCalls, // 移动参会人
    prisma.attendee.deleteMany({ where: { id: { in: attendeesToDelete } } }), // 删除重复参会人
  ]);

  // 更新外部日历
  const updateManager = await eventManager.reschedule(copyEvent, rescheduleUid, newTimeSlotBooking.id);

  // 发送通知
  if (noEmail !== true && isConfirmedByDefault) {
    await sendRescheduledEmailsAndSMS(...);
  }

  // 标记旧预订为已取消
  await prisma.booking.update({
    where: { id: seatedBooking.id },
    data: { status: BookingStatus.CANCELLED },
  });
};
```

**关键点**：
- **复用新记录**：使用已存在的新时间段预订
- **标记旧记录为取消**：将旧 booking 状态更新为 `CANCELLED`
- **移动参会人**：更新参会人的 `bookingId` 指向新预订
- **检查座位容量**：确保合并后不超过座位限制

### 2.3 参会人重排（Attendee Reschedule）

参会人重排的逻辑在 `attendeeRescheduleSeatedBooking.ts:14-122` 中。

```typescript
const attendeeRescheduleSeatedBooking = async (...) => {
  // 首先从旧预订中移除该参会人
  if (originalBookingEvt && originalRescheduledBooking) {
    const filteredAttendees = originalRescheduledBooking?.attendees.filter(
      (attendee) => attendee.email !== bookerEmail
    );
    const deletedReference = await lastAttendeeDeleteBooking(
      originalRescheduledBooking,
      filteredAttendees,
      originalBookingEvt
    );

    if (!deletedReference) {
      await eventManager.updateCalendarAttendees(originalBookingEvt, originalRescheduledBooking);
    }
  }

  if (!newTimeSlotBooking) {
    // 情况1：新时间段没有预订
    // 删除旧参会人记录
    await prisma.attendee.delete({ where: { id: seatAttendee?.id } });
    
    // 不触发原预订的重排逻辑
    originalRescheduledBooking = null;

    // 发送通知
    await sendRescheduledSeatEmailAndSMS(evtWithVideoCallData, seatAttendee as Person, eventType.metadata);

    return null; // 需要通过 createNewSeat 流程创建新预订
  }

  // 情况2：新时间段已有预订
  if (seatAttendee?.id && bookingSeat?.id) {
    await prisma.$transaction([
      // 更新参会人的 bookingId
      prisma.attendee.update({
        where: { id: seatAttendee.id },
        data: { bookingId: newTimeSlotBooking.id },
      }),
      // 更新 bookingSeat 的 bookingId
      prisma.bookingSeat.update({
        where: { id: bookingSeat.id },
        data: { bookingId: newTimeSlotBooking.id },
      }),
    ]);
  }

  // 更新新预订的日历参会人
  const copyEvent = cloneDeep({ ...evt, iCalUID: newTimeSlotBooking.iCalUID });
  await eventManager.updateCalendarAttendees(copyEvent, newTimeSlotBooking);

  // 发送通知
  await sendRescheduledSeatEmailAndSMS(
    copyEventWithVideoCallData,
    seatAttendee as Person,
    eventType.metadata
  );

  // 检查旧预订是否还有其他参会人，没有则标记为取消
  const filteredAttendees = originalRescheduledBooking?.attendees.filter(
    (attendee) => attendee.email !== bookerEmail
  );
  await lastAttendeeDeleteBooking(originalRescheduledBooking, filteredAttendees, originalBookingEvt);
};
```

#### `lastAttendeeDeleteBooking` 逻辑

在 `lastAttendeeDeleteBooking.ts:1-70` 中：

```typescript
const lastAttendeeDeleteBooking = async (
  originalRescheduledBooking,
  filteredAttendees,
  originalBookingEvt
) => {
  if (filteredAttendees?.length === 0) {
    // 没有其他参会人，标记预订为取消
    await prisma.booking.update({
      where: { id: originalRescheduledBooking.id },
      data: { status: BookingStatus.CANCELLED },
    });
    // 删除日历事件
    // ...
    return true;
  }
  return false;
};
```

### 2.4 外部日历更新机制（重排）

重排时的日历更新通过 `EventManager.reschedule()` 方法实现，位置在 `EventManager.ts:615-775`。

#### 核心逻辑

```typescript
public async reschedule(
  event: CalendarEvent,
  rescheduleUid: string,
  newBookingId?: number,
  changedOrganizer?: boolean,
  previousHostDestinationCalendar?: DestinationCalendar[] | null,
  isBookingRequestedReschedule?: boolean,
  skipDeleteEventsAndMeetings?: boolean
): Promise<CreateUpdateResult> {
  // 获取原预订信息
  const booking = await prisma.booking.findUnique({
    where: { uid: rescheduleUid },
    select: { ... }
  });

  // 判断是否需要重新创建事件
  const shouldRecreateEvent =
    !!changedOrganizer || isLocationChanged || !!isBookingRequestedReschedule || isDailyVideoRoomExpired;

  if (evt.requiresConfirmation) {
    // 需要确认的重排：先删除旧事件，等待确认后再创建
    if (!skipDeleteEventsAndMeetings) {
      await this.deleteEventsAndMeetings({ ... });
    }
  } else if (shouldRecreateEvent) {
    // 组织者变化或位置变化：删除旧事件，创建新事件
    if (!skipDeleteEventsAndMeetings) {
      await this.deleteEventsAndMeetings({ ... });
    }
    const createdEvent = await this.create(originalEvt);
    results.push(...createdEvent.results);
  } else {
    // 普通重排：更新现有事件
    const updatedEvents = await this.updateEvents(evt, booking, rescheduleUid);
    results.push(...updatedEvents.results);
  }
}
```

#### 重排时的日历更新策略

| 场景 | 策略 | 说明 |
|-----|------|------|
| 需要确认的重排 | 删除旧事件，等待确认 | 组织者需要确认后才创建新事件 |
| 组织者变化 | 删除旧事件，创建新事件 | 不同组织者的日历不同 |
| 位置变化 | 删除旧事件，创建新事件 | 视频会议链接可能变化 |
| 普通重排 | 更新现有事件 | 只修改时间，保持 iCalUID 不变 |

### 2.5 参会人通知流程（重排）

重排时的通知发送有两种情况：

#### 组织者重排通知

在 `moveSeatedBookingToNewTimeSlot.ts:111-122` 和 `combineTwoSeatedBookings.ts:137-148` 中：

```typescript
if (noEmail !== true && isConfirmedByDefault) {
  await sendRescheduledEmailsAndSMS(
    {
      ...copyEvent,
      additionalNotes,
      cancellationReason: `$RCH$${rescheduleReason ? rescheduleReason : ""}`,
    },
    eventType.metadata
  );
}
```

#### 参会人重排通知

在 `attendeeRescheduleSeatedBooking.ts:63` 和 `109-113` 中：

```typescript
await sendRescheduledSeatEmailAndSMS(
  evtWithVideoCallData,
  seatAttendee as Person,
  eventType.metadata
);
```

#### 特殊的取消原因前缀

注意到重排通知中的 `cancellationReason` 有特殊前缀 `$RCH$`：

```typescript
cancellationReason: `$RCH$${rescheduleReason ? rescheduleReason : ""}`
```

这个前缀用于在邮件模板中区分真正的取消和重排（重排是"先取消再创建"的逻辑）。

---

## 三、新记录生成 vs 旧记录复用

### 3.1 复用旧记录的情况

#### 情况1：组织者重排座位预订，新时间段无预订

**位置**：`packages/features/bookings/lib/handleSeats/reschedule/owner/moveSeatedBookingToNewTimeSlot.ts`

**操作**：
```typescript
// 直接更新现有 booking 的时间
const newBooking = await updateBooking({
  bookingId: seatedBooking.id,
  startTime: evt.startTime,
  endTime: evt.endTime,
  cancellationReason: rescheduleReason,
});
```

**复用策略**：
- **保持 booking ID 不变**：不创建新的 booking 记录
- **更新时间字段**：只修改 `startTime`、`endTime`、`cancellationReason`
- **保持 iCalUID 不变**：日历事件只更新时间，不创建新事件
- **保持参会人关联不变**：`attendee.bookingId` 保持不变

**原因**：
- 避免创建不必要的新记录
- 保持预订历史的连续性
- 减少数据库操作
- 日历事件只需更新时间，无需重新创建

#### 情况2：参会人重排座位预订，新时间段已有预订

**位置**：`packages/features/bookings/lib/handleSeats/reschedule/attendee/attendeeRescheduleSeatedBooking.ts`

**操作**：
```typescript
if (seatAttendee?.id && bookingSeat?.id) {
  await prisma.$transaction([
    // 更新参会人的 bookingId 指向新预订
    prisma.attendee.update({
      where: { id: seatAttendee.id },
      data: { bookingId: newTimeSlotBooking.id },
    }),
    // 更新 bookingSeat 的 bookingId
    prisma.bookingSeat.update({
      where: { id: bookingSeat.id },
      data: { bookingId: newTimeSlotBooking.id },
    }),
  ]);
}
```

**复用策略**：
- **复用新时间段的预订**：使用已存在的 `newTimeSlotBooking`
- **更新关联关系**：只修改 `attendee.bookingId` 和 `bookingSeat.bookingId`
- **不创建新记录**：不创建新的 booking、attendee 或 bookingSeat 记录

**原因**：
- 新时间段已有预订，避免重复创建
- 保持预订的原子性
- 参会人只是"移动"到另一个已存在的时间段

### 3.2 生成新记录/标记旧记录为取消的情况

#### 情况1：组织者重排座位预订，新时间段已有预订

**位置**：`packages/features/bookings/lib/handleSeats/reschedule/owner/combineTwoSeatedBookings.ts`

**操作**：
```typescript
// 1. 移动或删除参会人
for (const attendee of seatedBooking.attendees) {
  if (newTimeSlotBooking.attendees.some(
    (newBookingAttendee) => newBookingAttendee.email === attendee.email
  )) {
    attendeesToDelete.push(attendee.id); // 已存在，删除旧记录
  } else {
    attendeesToMove.push({ id: attendee.id, seatReferenceId: attendee.bookingSeat?.id }); // 移动到新预订
  }
}

// 2. 执行数据库操作
await prisma.$transaction([
  ...moveAttendeeCalls, // 移动参会人（更新 bookingId）
  prisma.attendee.deleteMany({ where: { id: { in: attendeesToDelete } } }), // 删除重复参会人
]);

// 3. 标记旧预订为取消
await prisma.booking.update({
  where: { id: seatedBooking.id },
  data: { status: BookingStatus.CANCELLED },
});

// 4. 可能需要创建新的 bookingSeat
prisma.attendee.update({
  data: {
    bookingSeat: {
      upsert: {
        create: { referenceUid: uuid(), bookingId: newTimeSlotBooking.id },
        update: { bookingId: newTimeSlotBooking.id },
      },
    },
  },
});
```

**策略**：
- **复用新预订**：使用已存在的 `newTimeSlotBooking`
- **标记旧预订为取消**：`seatedBooking.status = CANCELLED`
- **移动参会人**：更新 `attendee.bookingId`
- **可能创建新的 bookingSeat**：使用 `upsert` 确保座位记录存在

**原因**：
- 两个时间段都有预订，需要合并
- 保持新时间段的预订作为主记录
- 旧预订需要保留历史记录（标记为取消而非删除）

#### 情况2：参会人重排座位预订，新时间段无预订

**位置**：`packages/features/bookings/lib/handleSeats/reschedule/attendee/attendeeRescheduleSeatedBooking.ts`

**操作**：
```typescript
if (!newTimeSlotBooking) {
  // 删除旧参会人记录
  await prisma.attendee.delete({
    where: { id: seatAttendee?.id },
  });

  // 不触发原预订的重排逻辑
  originalRescheduledBooking = null;

  // 发送通知
  await sendRescheduledSeatEmailAndSMS(...);

  return null; // 返回 null，需要通过 createNewSeat 创建新预订
}
```

**策略**：
- **删除旧参会人**：从原预订中移除该参会人
- **检查旧预订状态**：如果没有其他参会人，标记为取消
- **返回 null**：触发 `createNewSeat` 流程创建新预订

**原因**：
- 新时间段没有预订，需要创建新的预订
- 旧预订可能还有其他参会人，不能直接取消
- 通过返回 null，让上层逻辑走创建新座位的流程

#### 情况3：普通预订重排（非座位预订）

**位置**：`packages/features/bookings/lib/service/RegularBookingService.ts`

**策略**（基于代码分析）：
- **创建新的 booking 记录**：使用新的 `uid`
- **标记旧记录为取消**：`originalRescheduledBooking.status = CANCELLED`
- **记录重排关系**：`fromReschedule` 字段指向旧预订
- **递增 iCalSequence**：`iCalSequence = originalRescheduledBooking.iCalSequence + 1`

**原因**：
- 保持预订历史的完整性
- 记录每次重排的轨迹
- 日历事件可以通过 iCalSequence 区分版本

### 3.3 决策流程图

```
                    ┌─────────────────┐
                    │  发起重排请求   │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                              │
      ┌───────▼───────┐              ┌───────▼───────┐
      │  座位预订？   │              │  普通预订？    │
      └───────┬───────┘              └───────┬───────┘
              │                              │
    ┌─────────┴─────────┐                    │
    │                   │                    │
┌───▼───┐         ┌─────▼─────┐              │
│组织者 │         │  参会人   │              │
│重排  │         │  重排    │              │
└───┬───┘         └─────┬─────┘              │
    │                   │                    │
 ┌──▼──────────┐   ┌────▼──────────┐         │
 │新时间段有预订?│   │新时间段有预订? │         │
 └──┬──────────┘   └────┬──────────┘         │
    │                   │                    │
   ┌▼─┐               ┌──▼─┐              ┌──▼─┐
   │是 │               │是  │              │否 │
   └┬──┘               └┬───┘              └──┬─┘
    │                   │                     │
┌───▼──────────┐   ┌────▼─────────────┐  ┌───▼──────────┐
│复用新预订    │   │更新参会人bookingId │  │创建新预订    │
│标记旧预订为  │   │指向新预订         │  │标记旧预订为  │
│CANCELLED    │   │                   │  │CANCELLED    │
└─────────────┘   └───────────────────┘  └───────────────┘
    │
   ┌▼─┐
   │否 │
   └┬──┘
    │
┌───▼──────────┐
│复用旧预订    │
│更新时间字段  │
│不创建新记录  │
└─────────────┘
```

### 3.4 关键字段说明

| 字段名 | 位置 | 用途 | 复用/新建时的变化 |
|-------|------|------|-----------------|
| `id` | `booking` | 主键 | 复用时不变，新建时自增 |
| `uid` | `booking` | 业务唯一标识 | 复用时不变，新建时生成新 UUID |
| `iCalUID` | `booking` | 日历事件唯一标识 | 普通重排不变，组织者变化时新建 |
| `iCalSequence` | `booking` | 日历事件序列 | 每次重排递增 |
| `status` | `booking` | 预订状态 | 旧记录可能变为 CANCELLED |
| `fromReschedule` | `booking` | 重排来源 | 新记录指向旧记录 uid |
| `rescheduled` | `booking` | 是否重排过 | 重排后置为 true |
| `rescheduledBy` | `booking` | 重排人 | 记录重排操作人 |
| `cancellationReason` | `booking` | 取消/重排原因 | 重排时记录原因，带 $RCH$ 前缀 |
| `bookingId` | `attendee` | 关联预订 | 移动参会人时更新 |
| `referenceUid` | `bookingSeat` | 座位唯一标识 | 复用时不变 |

---

## 四、关键代码位置索引

### 4.1 取消预约

| 功能 | 文件路径 | 关键函数/方法 |
|-----|---------|-------------|
| 主流程 | `packages/features/bookings/lib/handleCancelBooking.ts` | `handler()` |
| 座位取消 | `packages/features/bookings/lib/handleSeats/cancel/cancelAttendeeSeat.ts` | `cancelAttendeeSeat()` |
| 退款处理 | `packages/features/bookings/lib/payment/processPaymentRefund.ts` | `processPaymentRefund()` |
| 退款政策 | `packages/lib/payment/types.ts` | `RefundPolicy` enum |
| 取消通知 | `packages/emails/email-manager.ts` | `sendCancelledEmailsAndSMS()` |
| 座位取消通知 | `packages/emails/email-manager.ts` | `sendCancelledSeatEmailsAndSMS()` |
| 日历取消 | `packages/features/bookings/lib/EventManager.ts` | `cancelEvent()` |

### 4.2 重排预约

| 功能 | 文件路径 | 关键函数/方法 |
|-----|---------|-------------|
| 座位重排入口 | `packages/features/bookings/lib/handleSeats/reschedule/rescheduleSeatedBooking.ts` | `rescheduleSeatedBooking()` |
| 组织者重排 | `packages/features/bookings/lib/handleSeats/reschedule/owner/ownerRescheduleSeatedBooking.ts` | `ownerRescheduleSeatedBooking()` |
| 移动到新时间段 | `packages/features/bookings/lib/handleSeats/reschedule/owner/moveSeatedBookingToNewTimeSlot.ts` | `moveSeatedBookingToNewTimeSlot()` |
| 合并预订 | `packages/features/bookings/lib/handleSeats/reschedule/owner/combineTwoSeatedBookings.ts` | `combineTwoSeatedBookings()` |
| 参会人重排 | `packages/features/bookings/lib/handleSeats/reschedule/attendee/attendeeRescheduleSeatedBooking.ts` | `attendeeRescheduleSeatedBooking()` |
| 最后参会人处理 | `packages/features/bookings/lib/handleSeats/lib/lastAttendeeDeleteBooking.ts` | `lastAttendeeDeleteBooking()` |
| 创建新座位 | `packages/features/bookings/lib/handleSeats/create/createNewSeat.ts` | `createNewSeat()` |
| 日历重排 | `packages/features/bookings/lib/EventManager.ts` | `reschedule()` |
| 日历更新参会人 | `packages/features/bookings/lib/EventManager.ts` | `updateCalendarAttendees()` |
| 重排通知 | `packages/emails/email-manager.ts` | `sendRescheduledEmailsAndSMS()` |
| 座位重排通知 | `packages/emails/email-manager.ts` | `sendRescheduledSeatEmailAndSMS()` |
| 普通预订服务 | `packages/features/bookings/lib/service/RegularBookingService.ts` | `handler()` |

### 4.3 数据模型

| 模型 | 文件路径 | 关键字段 |
|-----|---------|---------|
| Booking | `packages/prisma/schema.prisma` | `id`, `uid`, `status`, `iCalUID`, `iCalSequence`, `fromReschedule`, `rescheduled`, `rescheduledBy`, `cancellationReason` |
| Attendee | `packages/prisma/schema.prisma` | `id`, `bookingId`, `email`, `name` |
| BookingSeat | `packages/prisma/schema.prisma` | `id`, `referenceUid`, `bookingId`, `attendeeId` |
| BookingReference | `packages/prisma/schema.prisma` | `id`, `uid`, `type`, `bookingId`, `deleted` |
| Payment | `packages/prisma/schema.prisma` | `id`, `amount`, `currency`, `success`, `paymentOption`, `appId` |

---

## 五、协作机制总结

### 5.1 取消预约时的协作

```
┌─────────────────────────────────────────────────────────────────┐
│                      取消预约流程                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  1. 权限验证    │
                    │  2. 状态检查    │
                    └────────┬────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
    ┌───────────────┐ ┌───────────────┐ ┌───────────────┐
    │ 座位预订?     │ │ 退款处理      │ │ 外部日历同步  │
    │ cancelAttendee│ │processPayment │ │EventManager.  │
    │Seat()         │ │Refund()       │ │cancelEvent()  │
    └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
            │                 │                 │
            │                 ▼                 │
            │         ┌───────────────┐         │
            │         │ 退款政策判断  │         │
            │         │ - RefundPolicy│         │
            │         │ - 退款期限    │         │
            │         └───────┬───────┘         │
            │                 │                 │
            └─────────────────┼─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  参会人通知      │
                    │ sendCancelled   │
                    │ EmailsAndSMS()  │
                    └─────────────────┘
```

### 5.2 重排预约时的协作

```
┌─────────────────────────────────────────────────────────────────┐
│                      重排预约流程                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  检查新时间段    │
                    │  是否有预订     │
                    └────────┬────────┘
                              │
            ┌─────────────────┴─────────────────┐
            │                                   │
            ▼                                   ▼
    ┌───────────────┐                   ┌───────────────┐
    │ 无预订        │                   │ 有预订        │
    │ (复用旧记录)  │                   │ (合并/移动)   │
    └───────┬───────┘                   └───────┬───────┘
            │                                   │
            ▼                                   ▼
    ┌───────────────┐                   ┌───────────────┐
    │ 更新booking   │                   │ 更新参会人    │
    │ 时间字段      │                   │ bookingId     │
    │ (不新建)      │                   │ 或标记为取消  │
    └───────┬───────┘                   └───────┬───────┘
            │                                   │
            └─────────────────┬─────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  外部日历同步    │
                    │ EventManager.   │
                    │ reschedule()    │
                    └────────┬────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  参会人通知      │
                    │ sendRescheduled │
                    │ EmailsAndSMS()  │
                    └─────────────────┘
```

### 5.3 退款、日历、通知的协作顺序

#### 取消预约时的执行顺序

```
1. 更新 booking.status = CANCELLED  (数据库)
         │
         ▼
2. 处理退款 (processPaymentRefund)
   - 检查 paymentOption (ON_BOOKING / HOLD)
   - 检查 RefundPolicy (NEVER / DAYS)
   - 检查退款期限
   - 执行退款 (handlePaymentRefund)
         │
         ▼
3. 外部日历同步 (EventManager.cancelEvent)
   - 删除日历事件
   - 删除视频会议
   - 标记 bookingReferences.deleted = true
         │
         ▼
4. 发送通知 (sendCancelledEmailsAndSMS)
   - 邮件
   - SMS
```

**关键点**：
- 数据库状态更新优先（确保数据一致性）
- 退款在日历同步之前（如果退款失败，至少预订状态已更新）
- 日历同步使用 `allSettled`（即使某个日历删除失败，不影响其他操作）
- 通知最后发送（确保所有状态已更新）

#### 重排预约时的执行顺序

```
1. 数据库操作 (新建/更新/标记为取消)
   - 复用旧记录：更新 startTime, endTime
   - 合并预订：移动参会人，标记旧记录为 CANCELLED
   - 新建记录：创建新 booking
         │
         ▼
2. 外部日历同步 (EventManager.reschedule)
   - 需要确认：删除旧事件，等待确认
   - 组织者变化：删除旧事件，创建新事件
   - 普通重排：更新现有事件 (iCalUID 不变)
         │
         ▼
3. 发送通知 (sendRescheduledEmailsAndSMS)
   - 邮件
   - SMS
   - cancellationReason 带 $RCH$ 前缀
```

**关键点**：
- 数据库操作使用事务（确保原子性）
- 日历同步根据场景选择不同策略
- 通知中的 `cancellationReason` 带 `$RCH$` 前缀，用于区分真正的取消和重排

### 5.4 新记录生成 vs 旧记录复用决策表

| 场景 | 操作类型 | Booking 记录 | Attendee 记录 | BookingSeat 记录 | iCalUID |
|-----|---------|-------------|---------------|-----------------|---------|
| 组织者重排座位，新时间段无预订 | 复用 | 更新时间字段 | 不变 | 不变 | 不变 |
| 组织者重排座位，新时间段有预订 | 合并 | 旧标记为 CANCELLED，复用新的 | 移动（更新 bookingId） | 移动或新建 | 新的不变 |
| 参会人重排座位，新时间段无预订 | 新建/删除 | 旧可能标记为 CANCELLED，需新建 | 删除旧的，创建新的 | 删除旧的，创建新的 | 新建时生成 |
| 参会人重排座位，新时间段有预订 | 移动 | 旧可能标记为 CANCELLED，复用新的 | 更新 bookingId | 更新 bookingId | 新的不变 |
| 普通预订重排 | 新建 | 旧标记为 CANCELLED，创建新的 | 复制或新建 | 新建 | 可能变化 |

---

## 六、注意事项与边界情况

### 6.1 循环预约（Recurring Booking）

- **取消时**：可选择取消单个预约或所有剩余预约
- **重排时**：单个预约重排会从循环系列中分离出来
- **日历同步**：循环事件的处理逻辑不同，需要更新 `thirdPartyRecurringEventId`

### 6.2 未确认预约（Pending Booking）

- **取消时**：日历事件可能不存在，删除操作会静默失败
- **重排时**：`requiresConfirmation` 为 true 时，先删除旧事件，等待确认后再创建

### 6.3 平台托管用户（Platform Managed User）

- **通知控制**：通过 `platformClientId` 和 `arePlatformEmailsEnabled` 控制是否发送邮件
- **日历同步**：可能有不同的同步策略

### 6.4 日历订阅 Webhook 触发的取消

- **跳过同步**：`skipCalendarSyncTaskCancellation = true` 时，不执行 `EventManager.cancelEvent()`
- **原因**：避免无限循环（Google/Office365 → Cal.diy → Google/Office365）
- **数据一致性**：仍然标记 `bookingReferences.deleted = true`

### 6.5 座位预订的特殊情况

- **最后一个参会人**：`lastAttendeeDeleteBooking` 会检查是否还有其他参会人，没有则标记预订为取消
- **座位容量检查**：合并预订时会检查 `seatsPerTimeSlot`，超过则抛出 `NotEnoughAvailableSeats` 错误
- **重复参会人**：合并时如果参会人已存在于新预订中，会删除旧的参会人记录

---

## 七、总结

### 7.1 核心设计原则

1. **数据一致性优先**：数据库操作优先于外部操作
2. **容错设计**：外部操作（日历、邮件）失败不阻止核心流程
3. **历史可追溯**：重排时标记旧记录为取消而非删除
4. **场景化处理**：不同场景（组织者/参会人、座位/普通预订）有不同策略

### 7.2 关键决策点

1. **复用 vs 新建**：
   - 组织者重排且新时间段无预订 → 复用
   - 参会人重排且新时间段有预订 → 复用新的
   - 其他情况 → 新建或标记旧记录为取消

2. **日历同步策略**：
   - 普通重排 → 更新现有事件
   - 组织者变化/位置变化 → 删除旧事件，创建新事件
   - 需要确认 → 删除旧事件，等待确认

3. **退款触发条件**：
   - `paymentOption = ON_BOOKING` → 检查退款政策
   - `paymentOption = HOLD` → 处理预授权
   - `RefundPolicy = NEVER` → 不退款
   - `RefundPolicy = DAYS` → 检查退款期限

### 7.3 代码优化建议

1. **事务边界**：部分数据库操作分散在不同函数中，建议统一事务边界
2. **错误处理**：外部操作失败的重试机制可以增强
3. **代码复用**：取消和重排的通知逻辑有相似之处，可以进一步抽象
4. **日志记录**：关键决策点（复用/新建、退款判断）的日志可以更详细

---

*文档生成时间：2026-05-04*
*基于代码版本：当前工作目录*
