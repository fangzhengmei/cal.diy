# Cal.diy 幂等机制与补偿策略精确结论报告

> 按时序给出的精确结论，基于代码实证分析

---

## 一、核心发现摘要（先看结论）

### 关于幂等防重复

| 链路 | `idempotencyKey` 实际作用 | 真正生效的幂等机制 |
|------|---------------------------|-------------------|
| **主预约创建链路** | ❌ 硬编码为 `null`，无任何作用 | `uid`（数据库主键） |
| **外部同步链路** | ❌ 生成并传递，但**从未被查询** | `iCalUID` 解析 + `hasStartTimeChanged()` + `skipCalendarSyncTaskCreation` |

**关键结论**：`idempotencyKey` 字段**名存实亡**——有生成、有存储，但**无任何地方用于幂等检查**。

### 关于补偿与重试

| 场景 | 补偿机制 | 重试机制 |
|------|---------|---------|
| **建库成功，外部写入失败** | ❌ **无主动补偿** | ❌ **无主动重试** |
| **reschedule 场景** | ✅ 主动删除旧事件 | ❌ 无重试 |
| **cancel 场景** | ✅ 主动删除事件 | ❌ 无重试 |

**关键结论**：当数据库成功但外部日历写入失败时，系统**没有任何自动补偿或重试机制**。只有日志记录，依赖人工介入。

---

## 二、幂等防重复机制按时序分析

### 2.1 主预约创建链路的幂等机制时序

```
时序：访客提交预约表单 → API → RegularBookingService → 数据库

┌──────────────────────────────────────────────────────────────────────────────┐
│  主预约创建链路的幂等机制生效时序                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【时序 1】buildDryRunBooking() 阶段                                         │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: RegularBookingService.ts:204                                      │
│                                                                              │
│  idempotencyKey: null,  ← 硬编码为 null，从未被赋值                         │
│                                                                              │
│  状态：❌ 只是传递数据，无任何幂等作用                                       │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 2】生成 uid 阶段                                                      │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: RegularBookingService.ts:1267-1268                               │
│                                                                              │
│  const seed = `${organizerUser.username}:${dayjs(reqBody.start)            │
│               .utc().format()}:${Date.now()}`;                              │
│  const uid = translator.fromUUID(uuidv5(seed, uuidv5.URL));                │
│                                                                              │
│  分析：                                                                       │
│  - ✅ uid 是 Booking 表的唯一主键（数据库层面防重复）                        │
│  - ❌ 因包含 Date.now()，**每次请求生成不同值**                              │
│  - ⚠️ 无法防止业务层面的重复提交（如用户快速点击两次）                        │
│                                                                              │
│  状态：✅ 数据库层面生效，❌ 业务层面不生效                                   │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 3】生成 iCalUID 阶段                                                  │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: RegularBookingService.ts:1306-1309                               │
│                                                                              │
│  const iCalUID = getICalUID({                                                │
│    event: { iCalUID: originalRescheduledBooking?.iCalUID,                  │
│             uid: originalRescheduledBooking?.uid },                         │
│    uid,                                                                       │
│  });                                                                          │
│                                                                              │
│  格式：<uid>@cal.com                                                          │
│                                                                              │
│  分析：                                                                       │
│  - 用于外部日历标识事件                                                       │
│  - 主链路中**无任何基于 iCalUID 的重复检查**                                 │
│  - 只在外部同步链路中用于解析找到对应预约                                     │
│                                                                              │
│  状态：⚠️ 只是传递数据，主链路无幂等作用                                     │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 4】数据库存储阶段                                                     │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: prisma.$transaction 内部                                          │
│                                                                              │
│  Booking 表字段：                                                             │
│  - uid: 主键，唯一索引                                                        │
│  - iCalUID: 普通字段                                                          │
│  - idempotencyKey: 普通字段，值为 null                                       │
│                                                                              │
│  状态：✅ uid 作为主键生效，其他字段只是存储                                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 外部同步链路的幂等机制时序

```
时序：外部日历事件变更 → CalendarSyncService.handleEvents() → 处理

┌──────────────────────────────────────────────────────────────────────────────┐
│  外部同步链路的幂等机制生效时序                                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【时序 1】筛选 Cal.com 创建的事件                                            │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: CalendarSyncService.ts:46-48                                      │
│                                                                              │
│  const calEvents = calendarSubscriptionEvents.filter((e) =>                 │
│    e.iCalUID?.toLowerCase()?.endsWith("@cal.com")                           │
│  );                                                                           │
│                                                                              │
│  效果：只处理 Cal.com 创建的事件，避免处理外部用户手动创建的事件              │
│                                                                              │
│  状态：✅ 幂等防护第一层生效                                                  │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 2】解析 iCalUID 找到对应预约                                          │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: CalendarSyncService.ts:82, 157                                    │
│                                                                              │
│  const [bookingUid] = event.iCalUID?.split("@") ?? [undefined];            │
│  if (!bookingUid) {                                                           │
│    log.debug("Unable to sync, booking UID not found in iCalUID");          │
│    return;  // 无法解析则跳过                                                 │
│  }                                                                            │
│                                                                              │
│  效果：通过 iCalUID 解析出 bookingUid，找到 Cal.diy 中对应的预约记录        │
│                                                                              │
│  状态：✅ 幂等防护第二层生效                                                  │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 3】检查时间是否真正变化                                                │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: CalendarSyncService.ts:190-196                                    │
│                                                                              │
│  if (!hasStartTimeChanged(booking, event)) {                                 │
│    log.debug("Skipping reschedule, start time has not changed",            │
│              { bookingUid });                                                 │
│    return;  // 时间未变化则跳过                                               │
│  }                                                                            │
│                                                                              │
│  hasStartTimeChanged 实现（CalendarSyncService.ts:351-357）：               │
│  export const hasStartTimeChanged = (                                        │
│    booking: BookingWithEventType,                                            │
│    event: CalendarSubscriptionEventItem                                      │
│  ): boolean => {                                                              │
│    if (!event.start) return false;                                           │
│    return event.start.getTime() !== booking.startTime.getTime();             │
│  };                                                                           │
│                                                                              │
│  效果：防止外部日历重复推送相同的时间变更，防止无限循环                        │
│                                                                              │
│  状态：✅ 幂等防护第三层生效                                                  │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 4】生成 idempotencyKey（❌ 无实际作用）                             │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: CalendarSyncService.ts:266-271                                    │
│                                                                              │
│  idempotencyKey: IdempotencyKeyService.generate({                           │
│    startTime: new Date(start),                                               │
│    endTime: new Date(end),                                                   │
│    userId: booking.userId ?? undefined,                                      │
│    reassignedById: null,                                                     │
│  }),                                                                          │
│                                                                              │
│  IdempotencyKeyService 实现（idempotencyKeyService.ts）：                   │
│  export class IdempotencyKeyService {                                        │
│    static generate({ startTime, endTime, userId, reassignedById }) {       │
│      return uuidv5(                                                          │
│        `${startTime.valueOf()}.${endTime.valueOf()}.${userId}              │
│         ${reassignedById ? `.${reassignedById}` : ""}`,                     │
│        uuidv5.URL                                                             │
│      );                                                                       │
│    }                                                                          │
│  }                                                                            │
│                                                                              │
│  分析：                                                                       │
│  - 生成 UUID v5 哈希，相同输入总是生成相同 key                                │
│  - ❌ **生成后传递给 createBooking，但从未被查询**                           │
│  - ❌ 搜索整个代码库，无任何 findFirst.*idempotencyKey                      │
│                                                                              │
│  状态：⚠️ 只是传递数据，无任何幂等作用                                       │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 5】设置 skipCalendarSyncTaskCreation                                  │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: CalendarSyncService.ts:209                                        │
│                                                                              │
│  bookingMeta: {                                                               │
│    // Skip calendar event creation to avoid infinite loops                  │
│    // (Google/Office365 → Cal.diy → Google/Office365 → ...)                │
│    skipCalendarSyncTaskCreation: true,                                       │
│    skipAvailabilityCheck: true,                                              │
│    skipEventLimitsCheck: true,                                               │
│  }                                                                            │
│                                                                              │
│  效果：使用空操作的 EventManager，防止 Google → Cal.diy → Google 的无限循环 │
│                                                                              │
│  代码位置: RegularBookingService.ts:1833-1836                               │
│  const eventManager =                                                         │
│    !isDryRun && !skipCalendarSyncTaskCreation                                │
│      ? new EventManager({ ...organizerUser, credentials }, apps)            │
│      : buildDryRunEventManager();  // 使用空操作的 EventManager              │
│                                                                              │
│  buildDryRunEventManager 实现：                                               │
│  const buildDryRunEventManager = () => {                                     │
│    return {                                                                   │
│      create: async () => ({ results: [], referencesToCreate: [] }),         │
│      reschedule: async () => ({ results: [], referencesToCreate: [] }),     │
│    };                                                                         │
│  };                                                                           │
│                                                                              │
│  状态：✅ 幂等防护第四层生效（防止无限循环）                                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 两条链路的幂等机制对比总览

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  幂等机制：哪一层生效？哪一层只是传递数据？                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【主预约创建链路】                                                           │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 机制               │ 层级          │ 状态          │ 代码位置          │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │ idempotencyKey     │ 数据字段      │ ❌ 仅传递      │ RegularBookingService:204 │
│  │ uid                │ 数据库主键    │ ✅ 生效        │ RegularBookingService:1267-1268 │
│  │ iCalUID            │ 数据字段      │ ⚠️ 仅传递      │ RegularBookingService:1306-1309 │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  【外部同步链路】                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 机制                      │ 层级          │ 状态          │ 代码位置    │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │ @cal.com 后缀筛选        │ 事件筛选      │ ✅ 生效        │ CalendarSyncService:46-48 │
│  │ iCalUID 解析 bookingUid  │ 预约定位      │ ✅ 生效        │ CalendarSyncService:82,157 │
│  │ hasStartTimeChanged()    │ 时间检查      │ ✅ 生效        │ CalendarSyncService:190-196 │
│  │ idempotencyKey           │ 数据字段      │ ❌ 仅传递      │ CalendarSyncService:266-271 │
│  │ skipCalendarSyncTaskCreation │ 循环防护  │ ✅ 生效        │ CalendarSyncService:209 │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 三、补偿与重试机制按时序分析

### 3.1 建库成功但外部写入失败的时序

```
时序：数据库事务提交 → 外部日历写入 → 失败处理

┌──────────────────────────────────────────────────────────────────────────────┐
│  建库成功 + 外部写入失败的时序分析                                            │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【时序 1】数据库事务（✅ 原子性）                                           │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: RegularBookingService.createBooking() 内部                        │
│                                                                              │
│  prisma.$transaction(async (tx) => {                                        │
│    // 如果是 reschedule，更新原预约为 CANCELLED                              │
│    await tx.booking.update(originalBookingUpdateDataForCancellation);      │
│                                                                              │
│    // 创建新预约记录                                                          │
│    const booking = await tx.booking.create(createBookingObj);               │
│    // ├── Booking 表 (status = ACCEPTED)                                    │
│    // ├── Attendee 表 (createMany)                                          │
│    // └── BookingReference 表 (此时为空)                                     │
│                                                                              │
│    return booking;                                                            │
│  });                                                                          │
│                                                                              │
│  状态：✅ 事务提交成功，数据已持久化                                          │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 2】外部日历写入（❌ 非事务性）                                       │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: RegularBookingService.ts:1833-1836 之后                         │
│                                                                              │
│  const createManager = await eventManager.create(evt);                      │
│  // 内部调用：                                                                │
│  //   - createAllCalendarEvents() → Google/Outlook/Apple 日历              │
│  //   - createAllVideoMeetings() → Daily/Zoom/Teams 视频会议               │
│  //   - createAllCRMEvents() → Salesforce/HubSpot CRM                      │
│                                                                              │
│  返回: {                                                                      │
│    results: [                                                                │
│      { type: "google_calendar", success: false, error: {...} },            │
│      { type: "daily_video", success: false, error: {...} },                │
│      { type: "hubspot_crm", success: true, uid: "abc123", ... },          │
│    ],                                                                         │
│    referencesToCreate: [  // 只包含成功的                                     │
│      { type: "hubspot_crm", uid: "abc123", ... }                           │
│    ]                                                                          │
│  }                                                                            │
│                                                                              │
│  状态：❌ 部分失败或全部失败                                                 │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【时序 3】失败检查（⚠️ 关键决策点）                                         │
│  ────────────────────────────────────────────────────────────────────────  │
│  代码位置: RegularBookingService.ts:2064-2074                               │
│                                                                              │
│  if (results.length > 0 && results.every((res) => !res.success)) {         │
│    // 全部失败                                                                │
│    const error = {                                                            │
│      errorCode: "BookingCreatingMeetingFailed",                              │
│      message: "Booking failed",                                              │
│    };                                                                         │
│                                                                              │
│    tracingLogger.error(                                                       │
│      `EventManager.create failure in some of the integrations`,             │
│      safeStringify({ error, results })                                       │
│    );                                                                         │
│    // ════════════════════════════════════════════════════════════════     │
│    // ⚠️ 关键：这里只有日志记录，没有其他操作                                │
│    //    - 没有回滚数据库操作                                                 │
│    //    - 没有补偿机制                                                       │
│    //    - 没有重试逻辑                                                       │
│    //    - 预约状态保持 ACCEPTED                                              │
│    //    - 用户/组织者没有通知                                                │
│    // ════════════════════════════════════════════════════════════════     │
│  } else {                                                                     │
│    // 部分成功或全部成功                                                      │
│    // 保存成功的 referencesToCreate                                           │
│    // 继续执行后续流程（邮件、Webhook等）                                     │
│  }                                                                            │
│                                                                              │
│  状态：❌ 只记录日志，无任何补偿或重试                                       │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 补偿机制的触发条件与时序

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  补偿机制：什么时候触发？谁负责？                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【场景 1】建库成功，外部写入失败 ❌ 无补偿                                   │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  触发条件：                                                                    │
│  - 数据库事务已提交 ✅                                                        │
│  - eventManager.create() 部分或全部失败 ❌                                   │
│                                                                              │
│  实际处理：                                                                    │
│  - ✅ 记录错误日志（tracingLogger.error）                                     │
│  - ❌ 不回滚数据库                                                             │
│  - ❌ 不删除已创建的成功引用                                                   │
│  - ❌ 不重试                                                                   │
│  - ❌ 不通知用户                                                               │
│                                                                              │
│  责任边界：                                                                    │
│  - 数据库层：事务已提交，不负责外部一致性                                     │
│  - 业务服务层：只记录日志，不承担补偿责任                                     │
│  - 外部集成层：只返回结果，不承担重试责任                                     │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【场景 2】reschedule 场景 ✅ 有补偿                                         │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  触发条件：                                                                    │
│  - 用户重新安排预约                                                           │
│  - skipCalendarSyncTaskCreation = false                                      │
│                                                                              │
│  补偿动作时序：                                                                │
│  代码位置: RegularBookingService.ts:1897-1902                                │
│                                                                              │
│  【时序 A】先删除旧事件（补偿动作）                                           │
│  if (!skipCalendarSyncTaskCreation) {                                        │
│    await originalHostEventManager.deleteEventsAndMeetings({                 │
│      event: deletionEvent,                                                    │
│      bookingReferences: originalRescheduledBooking.references,               │
│    });                                                                        │
│  }                                                                            │
│                                                                              │
│  deleteEventsAndMeetings 内部（EventManager.ts:836-846）：                  │
│  // 使用 Promise.allSettled 确保部分失败不影响整体                           │
│  (await Promise.allSettled(allPromises)).some((result) => {                 │
│    if (result.status === "rejected") {                                       │
│      // 只记录警告日志，不抛出异常                                             │
│      log.warn("Error deleting calendar event or video meeting", ...);       │
│    }                                                                          │
│  });                                                                          │
│                                                                              │
│  【时序 B】再创建新预约和新事件                                                │
│  prisma.$transaction({ /* 创建新预约 */ });                                  │
│  eventManager.create(evt);  // 创建新外部事件                                 │
│                                                                              │
│  状态：✅ 有主动补偿，但补偿本身无重试                                         │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【场景 3】cancel 场景 ✅ 有补偿                                             │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  触发条件：用户取消预约                                                        │
│                                                                              │
│  补偿动作：调用 deleteEventsAndMeetings 删除外部事件                         │
│                                                                              │
│  同样使用 Promise.allSettled，部分失败只记录日志                              │
│                                                                              │
│  状态：✅ 有主动补偿，但补偿本身无重试                                         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 重试机制的实际情况

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  重试机制：有没有？什么时候触发？                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  【证据 1】无主动重试机制                                                     │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  搜索整个代码库：                                                              │
│  - 无 `retry(` 函数调用                                                       │
│  - 无 `for (let i = 0; i < maxRetries; i++)` 模式                           │
│  - 无 `while (!success && retries < maxRetries)` 模式                        │
│  - 无任何重试库（如 p-retry）的使用                                           │
│                                                                              │
│  结论：❌ 无主动重试机制                                                     │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【证据 2】只有凭证刷新机制（不是操作重试）                                   │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  代码位置: RegularBookingService.ts:1830-1831                                │
│                                                                              │
│  // After polling videoBusyTimes, credentials might have been changed      │
│  // due to refreshment, so query them again.                                │
│  const credentials = await refreshCredentials(allCredentials);               │
│                                                                              │
│  分析：                                                                       │
│  - 这是**凭证刷新**，不是操作重试                                             │
│  - 如果 OAuth token 过期，会刷新后重新获取凭证                               │
│  - 但如果外部 API 调用本身失败（如超时、限流），**不会重试**                  │
│                                                                              │
│  结论：✅ 有凭证刷新，❌ 无操作重试                                           │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  【证据 3】EventManager 内部无重试                                            │
│  ────────────────────────────────────────────────────────────────────────  │
│                                                                              │
│  EventManager.create() 内部调用：                                             │
│  - createAllCalendarEvents()                                                  │
│  - createAllVideoMeetings()                                                   │
│  - createAllCRMEvents()                                                       │
│                                                                              │
│  这些函数内部：                                                               │
│  - 直接调用外部 API（如 googleCalendar.createEvent）                         │
│  - 无 try-catch 重试逻辑                                                      │
│  - 失败直接抛出或返回错误状态                                                 │
│                                                                              │
│  结论：❌ 无内部重试机制                                                     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 责任边界分析

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  责任边界：谁负责什么？失败后谁来处理？                                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 【数据库层】Prisma / PostgreSQL                                          │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │ 责任范围：                                                                │ │
│  │ - Booking/Attendee/BookingReference 表的 CRUD 操作                      │ │
│  │ - 本地事务的原子性保证                                                    │ │
│  │                                                                          │ │
│  │ 失败处理：                                                                │ │
│  │ - 事务内操作失败 → 自动回滚 ✅                                            │ │
│  │                                                                          │ │
│  │ 责任边界：只关心本地数据一致性，不关心外部服务                            │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 【业务服务层】RegularBookingService                                      │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │ 责任范围：                                                                │ │
│  │ - 协调数据库操作和外部服务调用                                           │ │
│  │ - 业务流程编排                                                            │ │
│  │                                                                          │ │
│  │ 失败处理（实际行为）：                                                    │ │
│  │ - 检查 EventManager 结果                                                 │ │
│  │ - 只记录日志 ❌                                                           │ │
│  │ - 不回滚数据库 ❌                                                         │ │
│  │ - 不补偿 ❌                                                               │ │
│  │ - 不重试 ❌                                                               │ │
│  │                                                                          │ │
│  │ 责任边界：**实际上没有承担"最终一致性"责任**                              │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 【外部集成层】EventManager                                               │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │ 责任范围：                                                                │ │
│  │ - 调用外部日历/视频/CRM API                                               │ │
│  │ - 返回各集成的成功/失败状态                                               │ │
│  │                                                                          │ │
│  │ 失败处理：                                                                │ │
│  │ - 使用 Promise.allSettled，部分失败不影响整体                           │ │
│  │ - 不重试 ❌                                                               │ │
│  │ - 不补偿 ❌                                                               │ │
│  │ - 只返回结果 ❌                                                           │ │
│  │                                                                          │ │
│  │ 责任边界：只负责调用和返回结果，不负责一致性                              │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │ 【用户层】人工介入                                                        │ │
│  ├───────────────────────────────────────────────────────────────────────┤ │
│  │ 实际责任承担者：                                                          │ │
│  │ - 运维人员查看监控/日志                                                   │ │
│  │ - 用户反馈"日历中没有事件"                                                │ │
│  │ - 手动重新安排或取消预约                                                  │ │
│  │                                                                          │ │
│  │ 责任边界：**系统设计上依赖人工处理不一致**                                │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 四、关键代码位置索引

### 4.1 幂等机制相关代码位置

| 机制 | 文件路径 | 行号 | 说明 |
|------|---------|------|------|
| 主链路 idempotencyKey 硬编码 null | `packages/features/bookings/lib/service/RegularBookingService.ts` | 204 | 无实际作用 |
| 主链路 uid 生成 | `packages/features/bookings/lib/service/RegularBookingService.ts` | 1267-1268 | 包含 Date.now() |
| 主链路 iCalUID 生成 | `packages/features/bookings/lib/service/RegularBookingService.ts` | 1306-1309 | 格式 `<uid>@cal.com` |
| 外部链路 @cal.com 筛选 | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts` | 46-48 | 只处理 Cal.com 事件 |
| 外部链路 iCalUID 解析 | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts` | 82, 157 | 找到对应预约 |
| 外部链路 hasStartTimeChanged | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts` | 190-196 | 时间检查 |
| 外部链路 idempotencyKey 生成 | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts` | 266-271 | 生成但不查询 |
| 外部链路 skipCalendarSyncTaskCreation | `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts` | 209 | 防止无限循环 |
| IdempotencyKeyService 实现 | `packages/lib/idempotencyKey/idempotencyKeyService.ts` | 全部 | 生成但不查询 |
| buildDryRunEventManager | `packages/features/bookings/lib/service/RegularBookingService.ts` | 1833-1836 | 空操作实现 |

### 4.2 补偿与重试相关代码位置

| 机制 | 文件路径 | 行号 | 说明 |
|------|---------|------|------|
| 外部写入失败检查 | `packages/features/bookings/lib/service/RegularBookingService.ts` | 2064-2074 | 只记录日志 |
| reschedule 删除旧事件 | `packages/features/bookings/lib/service/RegularBookingService.ts` | 1897-1902 | 补偿动作 |
| deleteEventsAndMeetings 实现 | `packages/features/bookings/lib/EventManager.ts` | 792-850 | 使用 allSettled |
| 凭证刷新 | `packages/features/bookings/lib/service/RegularBookingService.ts` | 1830-1831 | 不是操作重试 |

---

## 五、最终结论摘要

### 关于幂等防重复

**按时序结论**：

| 时序点 | 主预约创建链路 | 外部同步链路 |
|--------|--------------|-------------|
| 数据生成阶段 | `idempotencyKey: null`（硬编码） | `idempotencyKey: 生成 UUID v5` |
| 数据传递阶段 | 传递给数据库存储 | 传递给 `createBooking` |
| 幂等检查阶段 | ❌ **无任何查询检查** | ❌ **无任何查询检查** |
| 实际生效机制 | `uid`（数据库主键） | `iCalUID` 解析 + `hasStartTimeChanged` + `skipCalendarSyncTaskCreation` |

**精确结论**：
1. `idempotencyKey` 字段**名存实亡**——有生成、有存储，但**无任何地方用于幂等检查**
2. 主预约创建链路的实际幂等机制是 `uid`（数据库主键），但因包含 `Date.now()`，**无法防止业务层面重复提交**
3. 外部同步链路的实际幂等机制是**三层防护**：
   - Layer 1: `iCalUID` 解析找到对应预约
   - Layer 2: `hasStartTimeChanged()` 检查时间是否真正变化
   - Layer 3: `skipCalendarSyncTaskCreation` 防止无限循环

### 关于补偿与重试

**按时序结论**：

| 时序点 | 建库成功 + 外部写入失败场景 | reschedule/cancel 场景 |
|--------|---------------------------|-----------------------|
| 数据库事务 | ✅ 已提交，数据持久化 | ✅ 已提交 |
| 外部调用 | ❌ 部分或全部失败 | ✅ 先删除旧事件（补偿） |
| 失败检查 | ⚠️ 只记录日志 | ⚠️ 部分失败只记录日志 |
| 补偿机制 | ❌ **无主动补偿** | ✅ **有主动补偿**（删除旧事件） |
| 重试机制 | ❌ **无主动重试** | ❌ **无重试**（补偿本身不重试） |
| 预约状态 | ❌ 保持 ACCEPTED | ✅ 正常更新 |

**精确结论**：
1. **建库成功但外部写入失败后，无任何主动补偿或重试**
   - 只有日志记录
   - 预约状态保持 `ACCEPTED`
   - 用户/组织者无通知
   - 依赖人工介入

2. **只有 reschedule/cancel 场景有主动补偿**
   - 补偿动作：`deleteEventsAndMeetings()` 删除旧事件
   - 补偿本身使用 `Promise.allSettled`，部分失败只记录日志
   - 补偿本身无重试机制

3. **责任边界**：
   - 数据库层：只负责本地事务原子性
   - 业务服务层：只记录日志，不承担最终一致性责任
   - 外部集成层：只负责调用和返回结果
   - **实际责任承担者：用户/运维人员（人工介入）**

---

## 六、设计意图与风险评估

### 设计意图推测

1. **最终一致性策略**：
   - 系统采用"本地事务优先，外部调用尽力而为"的策略
   - 预约状态（ACCEPTED）基于数据库成功，而非外部服务成功
   - 这是一种"可用性优先"的设计

2. **幂等机制的历史遗留**：
   - `idempotencyKey` 字段可能是早期设计或其他场景（如 ManagedEventReassignment）使用
   - 在主预约创建和外部同步链路中，实际上没有实现完整的幂等检查

3. **补偿机制的场景限制**：
   - 补偿只在 reschedule/cancel 场景有意义（删除旧事件）
   - 新建场景的补偿（如删除已创建的预约）可能会造成用户困惑

### 风险评估

| 风险场景 | 概率 | 影响 | 当前缓解措施 |
|---------|------|------|-------------|
| 用户重复提交创建两个预约 | 中 | 中 | 无（依赖前端防抖） |
| 外部日历写入失败用户不知情 | 低 | 高 | 日志记录，依赖人工 |
| reschedule 时删除旧事件失败 | 低 | 中 | 日志记录 |
| 无限循环同步 | 低 | 高 | `skipCalendarSyncTaskCreation` |

---

**报告生成时间**：2025-07-01  
**分析依据**：代码实证分析（非文档推测）  
**相关文件**：
- `packages/features/bookings/lib/service/RegularBookingService.ts`
- `packages/features/calendar-subscription/lib/sync/CalendarSyncService.ts`
- `packages/features/bookings/lib/EventManager.ts`
- `packages/lib/idempotencyKey/idempotencyKeyService.ts`
