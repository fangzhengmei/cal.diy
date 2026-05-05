# Cal.diy iCal 日历订阅机制文档

## 1. 概述

Cal.diy 的日历订阅机制采用与传统 "独立 iCal feed URL" 不同的架构设计。系统主要通过**双向同步**和**写入外部日历**的方式，将用户的预约列表暴露给外部日历客户端（Google Calendar、Apple Calendar、Outlook 等）。

### 两类 ICS 链接的核心区别

在 Cal.diy 中，存在两类完全不同的 ICS 相关链接，它们的流向、用途和隐私边界截然不同：

| 维度 | ICS Feed 订阅链接（Cal.diy 作为客户端） | 单次预约 ICS 下载链接（Cal.diy 作为服务端） |
|------|------------------------------------------|---------------------------------------------|
| **流向** | 外部日历 → Cal.diy | Cal.diy → 用户 → 用户日历 |
| **用途** | 可用性检查（避免在繁忙时间创建预约） | 用户将单个预约添加到自己的日历 |
| **存储** | 加密存储在 Cal.diy 数据库 | 不存储，动态生成 |
| **鉴权** | 依赖外部 ICS URL 自身的访问控制 | 无鉴权，链接可自由分享 |
| **时效性** | 定期拉取，实时性取决于外部 ICS | 一次性，生成后即固定 |
| **隐私控制** | 依赖外部 ICS 内容本身的设置 | 部分隐私设置会影响 ICS 内容 |

---

## 2. ICS Feed 订阅链路（Cal.diy 作为客户端）

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ICS Feed 订阅链路（Cal.diy 作为客户端）                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐                                                        │
│  │  外部日历服务      │                                                        │
│  │  (Google/Outlook) │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                  │
│           │ 生成 ICS Feed URL                                                │
│           │ (用户在外部日历中操作)                                           │
│           ▼                                                                  │
│  ┌──────────────────┐                                                        │
│  │    ICS URL       │  例如：                                                  │
│  │  https://...     │  • Google: https://calendar.google.com/calendar/...   │
│  │  /calendar.ics   │  • Outlook: https://outlook.office365.com/ical/...   │
│  └────────┬─────────┘                                                        │
│           │                                                                  │
│           │ 用户复制到 Cal.diy                                                │
│           ▼                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                      Cal.diy 内部处理                                    │ │
│  │                                                                         │ │
│  │  Step 1: 前端输入                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │ Setup.tsx - 用户输入 ICS URL 列表                                  │   │ │
│  │  │  - 可以添加多个 ICS URL                                             │   │ │
│  │  │  - 提交到 /api/integrations/ics-feedcalendar/add                   │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │           │                                                             │ │
│  │           ▼                                                             │ │
│  │  Step 2: 后端校验                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │ IsICSUrlConstraint - URL 格式校验                                  │   │ │
│  │  │  • 协议：必须是 http: 或 https:                                    │   │ │
│  │  │  • 路径：必须以 .ics 结尾                                          │   │ │
│  │  │  • 必须是有效的 URL 格式                                            │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │           │                                                             │ │
│  │           ▼                                                             │ │
│  │  Step 3: 加密存储                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │ symmetricEncrypt({ urls, skipWriting })                           │   │ │
│  │  │  • 算法：AES-256                                                   │   │ │
│  │  │  • 密钥：CALENDSO_ENCRYPTION_KEY（32 字节）                        │   │ │
│  │  │  • 存储到 Credential.key 字段                                      │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │           │                                                             │ │
│  │           ▼                                                             │ │
│  │  Step 4: 连接性测试                                                      │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │ BuildCalendarService → listCalendars()                             │   │ │
│  │  │  • 尝试 fetch 每个 ICS URL                                          │   │ │
│  │  │  • 使用 ical.js 解析为 jCal 格式                                    │   │ │
│  │  │  • 验证是否能成功解析                                                │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │           │                                                             │ │
│  │           ▼                                                             │ │
│  │  Step 5: 读取使用                                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │ ICSFeedCalendarService.getAvailability()                           │   │ │
│  │  │  • 构造函数：symmetricDecrypt 解密 URL 列表                        │   │ │
│  │  │  • fetchCalendars()：HTTP GET 获取 ICS 内容                        │   │ │
│  │  │  • ical.js 解析 VEVENT 组件                                        │   │ │
│  │  │  • 提取 start/end 时间用于可用性检查                                │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                         │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 前端输入流程

**代码位置**：`apps/web/components/apps/ics-feedcalendar/Setup.tsx`

```tsx
export default function ICSFeedSetup() {
  const [urls, setUrls] = useState<string[]>([""]);
  
  return (
    <Form
      form={form}
      handleSubmit={async (_) => {
        // 提交到后端 API
        const res = await fetch("/api/integrations/ics-feedcalendar/add", {
          method: "POST",
          body: JSON.stringify({ urls }),
          headers: {
            "Content-Type": "application/json",
          },
        });
        const json = await res.json();
        if (!res.ok) {
          setErrorMessage(json?.message || t("something_went_wrong"));
        } else {
          router.push(json.url);
        }
      }}>
      {/* 动态添加多个 ICS URL 输入框 */}
      {urls.map((url, i) => (
        <div key={i} className="flex w-full items-center gap-2">
          <TextField
            required
            type="text"
            label={t("calendar_url")}
            value={url}
            onChange={(e) => {
              const newVal = e.target.value as string;
              setUrls((urls) => urls.map((x, ii) => (ii === i ? newVal : x)));
            }}
            placeholder="https://example.com/calendar.ics"
          />
          {i !== 0 ? (
            <button
              type="button"
              onClick={() => setUrls((urls) => urls.filter((_, ii) => i !== ii))}>
              <TrashIcon size={16} />
            </button>
          ) : null}
        </div>
      ))}
      
      {/* 添加更多 URL 按钮 */}
      <button
        type="button"
        onClick={() => {
          setUrls((urls) => urls.concat(""));
        }}>
        {t("add")} <PlusIcon className="inline" size={16} />
      </button>
    </Form>
  );
}
```

### 2.3 后端 API 端点

Cal.diy 提供两个 API 端点用于添加 ICS Feed：

#### 2.3.1 Web 应用端点（Next.js API Route）

**代码位置**：`packages/app-store/ics-feedcalendar/api/add.ts`

```typescript
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method === "POST") {
    const { urls } = req.body;
    
    // 获取当前用户
    const user = await prisma.user.findFirstOrThrow({
      where: { id: req.session?.user?.id },
      select: { id: true, email: true },
    });

    // 加密存储
    const data = {
      type: appConfig.type,  // "ics_feed"
      key: symmetricEncrypt(
        JSON.stringify({ urls }), 
        process.env.CALENDSO_ENCRYPTION_KEY || ""
      ),
      userId: user.id,
      teamId: null,
      appId: appConfig.slug,
      invalid: false,
    };

    try {
      // 连接性测试：尝试 fetch 和解析
      const dav = BuildCalendarService({
        id: 0,
        ...data,
        user: { email: user.email },
        encryptedKey: null,
      });
      const listedCals = await dav.listCalendars();

      if (listedCals.length !== urls.length) {
        throw new Error(`Listed cals and URLs mismatch: ${listedCals.length} vs. ${urls.length}`);
      }

      // 保存到数据库
      await prisma.credential.create({ data });
    } catch (e) {
      logger.error("Could not add ICS feeds", e);
      return res.status(500).json({ message: "Could not add ICS feeds" });
    }

    return res.status(200).json({ 
      url: getInstalledAppPath({ variant: "calendar", slug: "ics-feed" }) 
    });
  }
}
```

#### 2.3.2 API v2 端点（NestJS）

**代码位置**：`apps/api/v2/src/platform/calendars/controllers/calendars.controller.ts:91-109`

```typescript
@Controller({ path: "/v2/calendars", version: API_VERSIONS_VALUES })
export class CalendarsController {
  @Post("/ics-feed/save")
  @UseGuards(ApiAuthGuard)
  @ApiHeader(API_KEY_OR_ACCESS_TOKEN_HEADER)
  @ApiOperation({ summary: "Save an ICS feed" })
  async createIcsFeed(
    @GetUser("id") userId: number,
    @GetUser("email") userEmail: string,
    @Body() body: CreateIcsFeedInputDto
  ): Promise<CreateIcsFeedOutputResponseDto> {
    return await this.icsFeedService.save(userId, userEmail, body.urls, body.readOnly);
  }

  @Get("/ics-feed/check")
  @UseGuards(ApiAuthGuard)
  @ApiOperation({ summary: "Check an ICS feed" })
  async checkIcsFeed(@GetUser("id") userId: number): Promise<ApiResponse> {
    return await this.icsFeedService.check(userId);
  }
}
```

### 2.4 URL 校验逻辑

**代码位置**：`apps/api/v2/src/platform/calendars/input/create-ics.input.ts`

```typescript
@ValidatorConstraint({ async: false })
export class IsICSUrlConstraint implements ValidatorConstraintInterface {
  validate(url: unknown) {
    if (typeof url !== "string") return false;

    // 检查是否是有效的 URL 且以 .ics 结尾
    try {
      const urlObject = new URL(url);
      return (
        (urlObject.protocol === "http:" || urlObject.protocol === "https:") &&
        urlObject.pathname.endsWith(".ics")
      );
    } catch (error) {
      return false;
    }
  }

  defaultMessage() {
    return "The URL must be a valid ICS URL (ending with .ics)";
  }
}

export class CreateIcsFeedInputDto {
  @ApiProperty({
    example: ["https://cal.com/ics/feed.ics", "http://cal.com/ics/feed.ics"],
    description: "An array of ICS URLs",
  })
  @IsArray()
  @ArrayNotEmpty()
  @IsNotEmpty({ each: true })
  @Validate(IsICSUrlConstraint, { each: true })  // 应用自定义校验器
  urls!: string[];

  @IsBoolean()
  @ApiPropertyOptional({
    example: false,
    description: "Whether to allowing writing to the calendar or not",
    default: true,
  })
  @IsOptional()
  readOnly?: boolean = true;
}
```

### 2.5 加密存储逻辑

**代码位置**：`apps/api/v2/src/platform/calendars/services/ics-feed.service.ts:25-84`

```typescript
@Injectable()
export class IcsFeedService implements ICSFeedCalendarApp {
  async save(
    userId: number,
    userEmail: string,
    urls: string[],
    readonly = true
  ): Promise<CreateIcsFeedOutputResponseDto> {
    // 准备存储数据
    const data = {
      type: ICS_CALENDAR_TYPE,  // "ics_feed"
      ICS_CALENDAR,
      key: symmetricEncrypt(
        JSON.stringify({ urls, skipWriting: readonly }),
        process.env.CALENDSO_ENCRYPTION_KEY || ""
      ),
      userId: userId,
      teamId: null,
      appId: ICS_CALENDAR,
      invalid: false,
      delegationCredentialId: null,
      encryptedKey: null,
    };

    try {
      // 连接性测试
      const dav = BuildIcsFeedCalendarService({
        id: 0,
        ...data,
        user: { email: userEmail },
      });

      const listedCals = await dav.listCalendars();

      if (listedCals.length !== urls.length) {
        throw new BadRequestException(
          `Listed cals and URLs mismatch: ${listedCals.length} vs. ${urls.length}`
        );
      }

      // 保存或更新凭证
      const credential = await this.credentialRepository.upsertUserAppCredential(
        ICS_CALENDAR_TYPE,
        data.key,
        userId
      );

      // 清除缓存
      await this.calendarsCacheService.deleteConnectedAndDestinationCalendarsCache(userId);

      return {
        status: SUCCESS_STATUS,
        data: {
          id: credential.id,
          type: credential.type,
          userId: credential.userId,
          teamId: credential.teamId,
          appId: credential.appId,
          invalid: credential.invalid,
        },
      };
    } catch (e) {
      this.logger.error("Could not add ICS feeds", e);
      throw new BadRequestException("Could not add ICS feeds, try using private ics feed.");
    }
  }
}
```

### 2.6 读取和使用逻辑

**代码位置**：`packages/app-store/ics-feedcalendar/lib/CalendarService.ts`

```typescript
class ICSFeedCalendarService implements Calendar {
  private urls: string[] = [];
  protected integrationName = "ics-feed_calendar";

  constructor(credential: CredentialPayload) {
    // 构造函数中解密 URL 列表
    const { urls } = JSON.parse(
      symmetricDecrypt(credential.key as string, CALENDSO_ENCRYPTION_KEY)
    );
    this.urls = urls;
  }

  // 只读操作：创建/更新/删除事件都会警告并返回空结果
  createEvent(_event: CalendarEvent, _credentialId: number): Promise<NewCalendarEventType> {
    console.warn("createEvent called on ICS (read-only) feed");
    return Promise.resolve({
      uid: _event.uid || "",
      type: this.integrationName,
      id: "",
      password: "",
      url: "",
      additionalInfo: { calWarnings: ["ICS feed is read-only"] },
    });
  }

  // 拉取 ICS 内容
  fetchCalendars = async (): Promise<{ url: string; vcalendar: ICAL.Component }[]> => {
    // 并行 fetch 所有 ICS URL
    const reqPromises = await Promise.allSettled(
      this.urls.map((x) => fetch(x).then((y) => [x, y]))
    );
    
    const reqs = reqPromises
      .filter((x) => x.status === "fulfilled")
      .map((x) => (x as PromiseFulfilledResult<[string, Response]>).value);
    
    // 解析响应文本
    const res = await Promise.all(reqs.map((x) => x[1].text().then((y) => [x[0], y])));
    
    // 使用 ical.js 解析
    return res
      .map((x) => {
        try {
          const jcalData = ICAL.parse(x[1]);
          return {
            url: x[0],
            vcalendar: new ICAL.Component(jcalData),
          };
        } catch (e) {
          console.error("Error parsing calendar object: ", e);
          return null;
        }
      })
      .filter((x) => x !== null) as { url: string; vcalendar: ICAL.Component }[];
  };

  // 获取可用性（核心用途）
  async getAvailability(params: GetAvailabilityParams): Promise<EventBusyDate[]> {
    const { dateFrom, dateTo, selectedCalendars } = params;
    
    // 拉取所有 ICS 日历
    const calendars = await this.fetchCalendars();
    
    const userId = this.getUserId(selectedCalendars);
    const userTimeZone = userId ? await this.getUserTimezoneFromDB(userId) : "Europe/London";
    const events: { start: string; end: string; title: string }[] = [];

    // 解析每个日历中的 VEVENT
    calendars.forEach(({ vcalendar }) => {
      const vevents = vcalendar.getAllSubcomponents("vevent");
      vevents.forEach((vevent) => {
        // 解析事件时间、时区、周期性等
        const event = new ICAL.Event(vevent);
        const title = String(vevent.getFirstPropertyValue("summary"));
        
        // ... 时区处理、周期性事件展开等
        
        // 添加到繁忙时间列表
        events.push({
          start: finalStartISO,
          end: finalEndISO,
          title,
        });
      });
    });

    return Promise.resolve(events);
  }

  // 列出日历（用于连接性测试）
  async listCalendars(): Promise<IntegrationCalendar[]> {
    const vcals = await this.fetchCalendars();

    return vcals.map(({ url, vcalendar }) => {
      const name: string = vcalendar.getFirstPropertyValue("x-wr-calname");
      return {
        name,
        readOnly: true,  // ICS Feed 始终只读
        externalId: url,  // 使用 URL 作为 externalId
        integration: this.integrationName,
      };
    });
  }
}
```

### 2.7 ICS Feed 的隐私边界

#### 2.7.1 Cal.diy 可见的内容

当用户添加一个 ICS Feed 到 Cal.diy 时，Cal.diy 可以看到：

| 内容 | 是否可见 | 说明 |
|------|---------|------|
| **事件时间** | ✅ 可见 | 用于可用性检查的核心数据 |
| **事件标题** | ✅ 可见 | 从 `SUMMARY` 属性获取 |
| **事件描述** | ✅ 可见 | 从 `DESCRIPTION` 属性获取 |
| **事件位置** | ✅ 可见 | 从 `LOCATION` 属性获取 |
| **参与者列表** | ✅ 可见 | 从 `ATTENDEE` 属性获取 |
| **CLASSIFICATION** | ✅ 可见 | 用于判断事件隐私级别 |

**注意**：Cal.diy 只能看到 ICS URL 中暴露的内容。如果外部日历服务在生成 ICS Feed 时设置了隐私选项（如只显示繁忙时间），Cal.diy 也只能看到这些信息。

#### 2.7.2 外部日历服务的可见性

当用户将 ICS Feed 添加到 Cal.diy 时，**外部日历服务完全不知道 Cal.diy 的存在**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ICS Feed 订阅的隐私边界                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  场景：用户 A 在 Google Calendar 中生成了一个私有 ICS Feed URL，并添加到 Cal.diy │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  Google Calendar 服务端                                                 │ │
│  │                                                                         │ │
│  │  对 Cal.diy 的可见性：                                                  │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  ✗ 不知道 Cal.diy 的存在                                           │   │ │
│  │  │  ✗ 不知道是谁在订阅这个 ICS Feed                                    │   │ │
│  │  │  ✗ 无法区分 Cal.diy 的请求和普通浏览器请求                          │   │ │
│  │  │  ✗ 只有 HTTP 请求日志显示有客户端访问了 ICS URL                     │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│           │                                                                  │
│           │ HTTP GET /calendar.ics                                          │
│           ▼                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  Cal.diy 服务端                                                         │ │
│  │                                                                         │ │
│  │  对外部日历的可见性：                                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  ✅ 只能看到 ICS URL 中暴露的内容                                   │   │ │
│  │  │  ✅ 无法修改外部日历的任何内容（只读）                                │   │ │
│  │  │  ✅ 无法访问外部日历的其他事件（不在 ICS Feed 中的）                  │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                         │ │
│  │  存储内容：                                                              │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  Credential 表：                                                    │   │ │
│  │  │  • type: "ics_feed"                                                 │   │ │
│  │  │  • key: AES-256 加密的 { urls, skipWriting }                       │   │ │
│  │  │  • userId: 所属用户 ID                                               │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 2.7.3 ICS Feed 在三大平台的可见性差异

当用户从外部日历服务生成 ICS Feed URL 时，不同平台的隐私设置不同：

| 平台 | ICS Feed 生成方式 | 默认隐私级别 | 可配置选项 |
|------|-----------------|-------------|-----------|
| **Google Calendar** | 日历设置 → "集成日历" → "秘密地址" | 取决于日历分享设置 | 可选择只显示"繁忙"或显示完整详情 |
| **Outlook/Office 365** | 设置 → "日历" → "共享日历" → "发布日历" | 可选择详细级别 | 可选择：仅繁忙时间、有限详情、完整详情 |
| **Apple Calendar** | 日历右键 → "共享日历" → "公共日历" | 公共日历完全公开 | 可设置密码保护 |

**重要提示**：ICS Feed 的隐私级别完全由**外部日历服务**控制，Cal.diy 只能看到外部服务暴露的内容。

---

## 3. 单次预约 ICS 下载链接（Cal.diy 作为服务端）

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              单次预约 ICS 下载链路（Cal.diy 作为服务端）                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────┐                                                        │
│  │    Cal.diy       │                                                        │
│  │  (预约管理)      │                                                        │
│  └────────┬─────────┘                                                        │
│           │                                                                  │
│           │ 用户查看预约详情页                                                │
│           ▼                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    预约详情页（bookings-single-view.tsx）                │ │
│  │                                                                         │ │
│  │  显示四种日历链接：                                                       │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  1. Google Calendar  - 深层链接到 eventedit 页面                   │   │ │
│  │  │  2. Microsoft Office - 深层链接到 outlook.office.com               │   │ │
│  │  │  3. Microsoft Outlook - 深层链接到 outlook.live.com               │   │ │
│  │  │  4. ICS - Data URI 格式，用于下载或直接打开                         │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│           │                                                                  │
│           │ 用户点击 ICS 链接                                                │
│           ▼                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    链接生成逻辑（getCalendarLinks.ts）                   │ │
│  │                                                                         │ │
│  │  buildICalLink() - 动态生成 ICS 内容：                                  │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  • 不存储到数据库，每次动态生成                                      │   │ │
│  │  │  • 使用 ics 库创建事件                                               │   │ │
│  │  │  • 格式：data:text/calendar,${encodeURIComponent(icsContent)}      │   │ │
│  │  │  • 不包含隐私控制（classification 等）                               │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│           │                                                                  │
│           │ 用户下载/打开 .ics 文件                                          │
│           ▼                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    用户日历客户端                                        │ │
│  │                                                                         │ │
│  │  用户可以：                                                              │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  • 直接打开 .ics 文件，添加到默认日历                               │   │ │
│  │  │  • 保存文件后手动导入到日历客户端                                    │   │ │
│  │  │  • 分享 .ics 文件给其他人                                           │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                         │ │
│  │  注意：一旦添加到用户日历，事件的可见性完全由外部日历服务控制！           │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 链接生成逻辑

**代码位置**：`packages/features/bookings/lib/getCalendarLinks.ts`

```typescript
export const enum CalendarLinkType {
  GOOGLE_CALENDAR = "googleCalendar",
  MICROSOFT_OFFICE = "microsoftOffice",
  MICROSOFT_OUTLOOK = "microsoftOutlook",
  ICS = "ics",  // ICS Data URI
}

// 构建 ICS Data URI
const buildICalLink = ({
  startTime,
  endTime,
  title,
  description,
  location,
}: {
  startTime: Dayjs;
  endTime: Dayjs;
  title: string;
  description: string | null;
  location: string | null;
}) => {
  const durationInMinutes = endTime.diff(startTime, "minutes");

  // 使用 ics 库创建事件
  const iCalEvent = createEvent({
    start: [
      startTime.toDate().getUTCFullYear(),
      (startTime.toDate().getUTCMonth() as number) + 1,
      startTime.toDate().getUTCDate(),
      startTime.toDate().getUTCHours(),
      startTime.toDate().getUTCMinutes(),
    ],
    startInputType: "utc",  // 注意：使用 UTC 时间
    title,
    duration: { minutes: durationInMinutes },
    ...(description ? { description } : {}),
    ...(location ? { location } : {}),
  });

  if (iCalEvent.error) {
    throw iCalEvent.error;
  }

  // 注意：返回的是 Data URI，不是 HTTP URL！
  return `data:text/calendar,${encodeURIComponent(iCalEvent.value ? iCalEvent.value : false)}`;
};

// 构建 Google Calendar 深层链接
const buildGoogleCalendarLink = ({
  startTime,
  endTime,
  eventName,
  eventDescription,
  bookingLocation,
  recurringEvent,
}: {
  startTime: Dayjs;
  endTime: Dayjs;
  eventName: string;
  eventDescription: string | null;
  bookingLocation: string | null;
  recurringEvent: RecurringEventOrPrismaJsonObject;
}) => {
  const startTimeInUtcFormat = startTime.utc().format("YYYYMMDDTHHmmss[Z]");
  const endTimeInUtcFormat = endTime.utc().format("YYYYMMDDTHHmmss[Z]");
  const recurrence = recurringEvent ? encodeURIComponent(new RRule(recurringEvent).toString()) : "";

  const location = bookingLocation ? encodeURIComponent(bookingLocation) : "";
  const description = encodeURIComponent(eventDescription ?? "");

  // 深层链接到 Google Calendar 的 eventedit 页面
  const googleCalendarLink = `https://calendar.google.com/calendar/r/eventedit?dates=${startTimeInUtcFormat}/${endTimeInUtcFormat}&text=${eventName}&details=${description}${
    location ? `&location=${location}` : ""
  }${recurrence ? `&recur=${recurrence}` : ""}`;

  return googleCalendarLink;
};

// 导出所有类型的链接
export const getCalendarLinks = ({
  booking,
  eventType,
  t,
}: {
  booking: {
    startTime: Date;
    endTime: Date;
    location: string | null;
    title: string;
    responses: Prisma.JsonObject;
    metadata: Prisma.JsonObject | null;
  };
  eventType: {
    recurringEvent: RecurringEventOrPrismaJsonObject;
    description?: string | null;
    eventName?: string | null;
    isDynamic: boolean;
    length: number;
    team?: { name: string } | null;
    users: { name: string | null }[];
    title: string;
  };
  t: TFunction;
}) => {
  // ... 事件名称生成逻辑
  
  const startTime = dayjs(booking.startTime);
  const endTime = dayjs(booking.endTime);
  const videoCallUrl = bookingMetadataSchema.parse(booking?.metadata || {})?.videoCallUrl ?? null;

  // 生成四种类型的链接
  const googleCalendarLink = buildGoogleCalendarLink({
    startTime,
    endTime,
    eventName,
    eventDescription,
    bookingLocation: videoCallUrl ?? null,
    recurringEvent,
  });

  const microsoftOfficeLink = buildMicrosoftOfficeLink({
    startTime,
    endTime,
    eventName,
    eventDescription,
    bookingLocation: videoCallUrl,
  });

  const microsoftOutlookLink = buildMicrosoftOutlookLink({
    startTime,
    endTime,
    eventName,
    eventDescription,
    bookingLocation: videoCallUrl,
  });

  // 生成 ICS Data URI
  let icsFileLink = "";
  try {
    icsFileLink = buildICalLink({
      startTime,
      endTime,
      title: eventName,
      description: eventDescription ?? null,
      location: videoCallUrl ?? null,
    });
  } catch (error) {
    console.error("Error generating ICS file", error);
  }

  // 返回所有链接供前端展示
  return [
    {
      label: "Google Calendar",
      id: CalendarLinkType.GOOGLE_CALENDAR,
      link: googleCalendarLink,
    },
    {
      label: "Microsoft Office",
      id: CalendarLinkType.MICROSOFT_OFFICE,
      link: microsoftOfficeLink,
    },
    {
      label: "Microsoft Outlook",
      id: CalendarLinkType.MICROSOFT_OUTLOOK,
      link: microsoftOutlookLink,
    },
    {
      label: "ICS",
      id: CalendarLinkType.ICS,
      link: icsFileLink,
    },
  ];
};
```

### 3.3 邮件中的 ICS 附件

除了预约详情页的链接，Cal.diy 在发送通知邮件时也会附加 ICS 文件。这个版本的 ICS 包含更多隐私控制。

**代码位置**：`packages/emails/lib/generateIcsString.ts`

```typescript
const generateIcsString = ({
  event,
  status,
  partstat = "ACCEPTED",
  t,
}: {
  event: ICSCalendarEvent;
  status: EventStatus;
  partstat?: ParticipationStatus;
  t?: TFunction;
}): string | undefined => {
  const location = getVideoCallUrlFromCalEvent(event) || event.location;

  // 检查组织者邮箱是否豁免隐藏
  const isOrganizerExempt = ORGANIZER_EMAIL_EXEMPT_DOMAINS?.split(",")
    .filter((domain) => domain.trim() !== "")
    .some((domain) => event.organizer.email.toLowerCase().endsWith(domain.toLowerCase()));

  const icsEvent = createEvent({
    uid: event.iCalUID || event.uid!,
    sequence: event.iCalSequence || 0,
    start: toICalDateArray(event.startTime),
    end: toICalDateArray(event.endTime),
    startInputType: "utc",
    productId: "calcom/ics",
    title: event.title,
    description: getRichDescription(event, t),
    
    // 隐私控制 1：隐藏组织者邮箱
    organizer: {
      name: event.organizer.name,
      ...(event.hideOrganizerEmail && !isOrganizerExempt
        ? { email: "no-reply@cal.com" }  // 使用通用邮箱
        : { email: event.organizer.email }),
    },
    
    recurrenceRule: recurrenceRule,
    
    // 参与者列表
    attendees: [
      ...event.attendees.map((attendee: Person) => ({
        name: attendee.name,
        email: attendee.email,
        partstat,
        role: icsRole,
        rsvp: true,
      })),
      // ... 团队成员
    ],
    
    location: location ?? undefined,
    method: "REQUEST",
    status,
    
    // 隐私控制 2：隐藏事件详情
    ...(event.hideCalendarEventDetails ? { classification: "PRIVATE" } : {}),
    
    busyStatus: "BUSY",
  });
  
  if (icsEvent.error) {
    // 错误处理
    if (icsEvent.error.name === "ValidationError") {
      throw new ErrorWithCode(ErrorCode.BadRequest, icsEvent.error.message);
    }
    throw icsEvent.error;
  }
  
  return icsEvent.value;
};
```

### 3.4 两类 ICS 生成方式的对比

| 特性 | 预约详情页 ICS 链接 | 邮件附件 ICS |
|------|-------------------|-------------|
| **生成位置** | `getCalendarLinks.ts` | `generateIcsString.ts` |
| **隐私控制** | ❌ 无 | ✅ `classification: "PRIVATE"`、`hideOrganizerEmail` |
| **参与者列表** | ❌ 无 | ✅ 包含所有参与者 |
| **组织者信息** | 简化版 | 完整（或隐藏） |
| **事件状态** | ❌ 无 | ✅ `CONFIRMED`、`CANCELLED` 等 |
| **序列数** | ❌ 无 | ✅ `iCalSequence`（用于更新/取消） |
| **用途** | 用户手动添加到日历 | 邮件客户端自动识别 |

### 3.5 单次预约 ICS 链接的隐私边界

#### 3.5.1 链接类型和风险

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              单次预约 ICS 链接的隐私边界                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  链接类型 1：ICS Data URI（预约详情页）                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  格式：data:text/calendar,BEGIN:VCALENDAR...                          │   │
│  │                                                                         │   │
│  │  风险级别：⚠️ 中等                                                       │   │
│  │                                                                         │   │
│  │  特点：                                                                  │   │
│  │  • 完整的 ICS 内容编码在 URL 中                                         │   │
│  │  • 不依赖网络，可离线使用                                                │   │
│  │  • 可自由分享，链接包含所有信息                                          │   │
│  │  • 不包含隐私控制（classification 等）                                   │   │
│  │                                                                         │   │
│  │  可见信息：                                                              │   │
│  │  ✅ 事件时间、标题、描述、位置                                            │   │
│  │  ❌ 无参与者列表、无组织者邮箱隐藏                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  链接类型 2：邮件附件 ICS                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  格式：.ics 文件作为邮件附件                                             │   │
│  │                                                                         │   │
│  │  风险级别：✅ 较低（有隐私控制）                                         │   │
│  │                                                                         │   │
│  │  特点：                                                                  │   │
│  │  • 包含完整的隐私控制                                                    │   │
│  │  • `hideCalendarEventDetails` → `classification: "PRIVATE"`           │   │
│  │  • `hideOrganizerEmail` → 使用 `no-reply@cal.com`                    │   │
│  │  • 包含参与者列表、RSVP 状态                                             │   │
│  │                                                                         │   │
│  │  可见信息：                                                              │   │
│  │  ✅ 事件时间、标题（取决于隐私设置）                                      │   │
│  │  ✅ 参与者列表                                                           │   │
│  │  ⚠️ 组织者邮箱可能被隐藏                                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  链接类型 3：深层链接（Google/Outlook）                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  格式：https://calendar.google.com/calendar/r/eventedit?...           │   │
│  │                                                                         │   │
│  │  风险级别：⚠️ 中等                                                       │   │
│  │                                                                         │   │
│  │  特点：                                                                  │   │
│  │  • 链接本身包含所有信息（标题、描述、位置）                               │   │
│  │  • 跳转到外部日历的事件创建页面                                          │   │
│  │  • 外部日历可能会记录用户的操作                                          │   │
│  │                                                                         │   │
│  │  可见信息（在链接 URL 中）：                                              │   │
│  │  ✅ 事件时间（dates 参数）                                                │   │
│  │  ✅ 事件标题（text 参数）                                                │   │
│  │  ✅ 事件描述（details 参数）                                              │   │
│  │  ✅ 位置（location 参数）                                                │   │
│  │  ✅ 周期性规则（recur 参数）                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.5.2 在三大平台的可见性差异

当用户将 ICS 事件添加到外部日历后，事件的可见性完全由外部日历服务控制：

| 平台 | 添加方式 | 隐私控制 | 分享风险 |
|------|---------|---------|---------|
| **Google Calendar** | 导入 .ics 文件或使用深层链接 | 继承日历的默认可见性设置 | 用户可自由分享包含该事件的日历 |
| **Outlook** | 导入 .ics 文件或使用深层链接 | 继承日历的默认敏感度设置 | 用户可自由分享包含该事件的日历 |
| **Apple Calendar** | 双击 .ics 文件 | 继承日历的默认隐私设置 | 用户可自由分享包含该事件的日历 |

**关键提示**：
1. **一旦添加到外部日历，Cal.diy 完全失去控制**
2. **用户可以自由分享包含该事件的日历**
3. **外部日历服务可能会索引事件内容**
4. **深层链接中的 URL 参数是明文的，可能被浏览器历史记录、代理服务器等记录**

#### 3.5.3 深层链接的参数分析

以 Google Calendar 深层链接为例：

```
https://calendar.google.com/calendar/r/eventedit
  ?dates=20260505T100000Z/20260505T103000Z  ← 事件时间（UTC）
  &text=30min%20Meeting%20-%20John          ← 事件标题（URL 编码）
  &details=Join%20Zoom%3A%20https%3A%2F%2Fzoom.us%2Fj%2F123456  ← 描述（包含敏感链接）
  &location=https%3A%2F%2Fzoom.us%2Fj%2F123456  ← 位置
  &recur=FREQ%3DWEEKLY%3BCOUNT%3D4  ← 周期性规则
```

**风险点**：
- `text` 参数可能暴露参与者姓名
- `details` 参数可能暴露 Zoom 链接、会议密码等敏感信息
- `location` 参数可能暴露物理地址或虚拟会议链接
- 这些信息可能被浏览器历史、书签、分享链接等方式泄露

---

## 4. 两类 ICS 链接的完整对比

### 4.1 对比总表

| 维度 | ICS Feed 订阅链接（Cal.diy 作为客户端） | 单次预约 ICS 下载链接（Cal.diy 作为服务端） |
|------|------------------------------------------|---------------------------------------------|
| **数据流向** | 外部日历 → Cal.diy | Cal.diy → 用户 → 外部日历 |
| **主要用途** | 可用性检查（避免双订） | 用户添加单个预约到日历 |
| **存储方式** | 加密存储在 Cal.diy 数据库 | 不存储，动态生成 |
| **鉴权机制** | 依赖外部 ICS URL 的访问控制 | 无鉴权，链接可自由分享 |
| **有效期** | 长期，用户手动删除 | 一次性，生成后固定 |
| **内容范围** | 多个事件（整个日历） | 单个事件 |
| **隐私控制** | 依赖外部日历服务的 ICS 隐私设置 | 邮件附件有控制，详情页链接无控制 |
| **修改权限** | 只读，无法修改外部日历 | 用户添加后可自由修改 |
| **Cal.diy 控制力** | 中等（可控制是否存储 URL） | 极低（生成后完全失控） |

### 4.2 隐私边界对比图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    两类 ICS 链接的隐私边界对比                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    ICS Feed 订阅链路                                     │ │
│  │                                                                         │ │
│  │  外部日历服务 ──────► Cal.diy ──────► 可用性检查                        │ │
│  │       │                    │                                           │ │
│  │       │ ICS URL            │ AES-256 加密存储                          │ │
│  │       │                    │                                           │ │
│  │  Cal.diy 可见的内容：                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  ✅ 事件时间、标题、描述、位置、参与者                               │   │ │
│  │  │  ⚠️ 可见程度取决于外部 ICS 的隐私设置                                │   │ │
│  │  │  ✅ 无法修改外部日历的任何内容                                       │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                         │ │
│  │  外部日历服务可见的内容：                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  ❌ 不知道 Cal.diy 的存在                                           │   │ │
│  │  │  ❌ 不知道是谁在订阅                                                 │   │ │
│  │  │  ✅ 只有 HTTP 请求日志                                               │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                    单次预约 ICS 下载链路                                 │ │
│  │                                                                         │ │
│  │  Cal.diy ──────► 用户 ──────► 外部日历服务 ──────► 用户日历客户端      │ │
│  │    │              │              │                    │               │ │
│  │    │ ICS Data URI │ .ics 文件    │ 导入到日历         │ 查看/分享     │ │
│  │    │ 或深层链接    │              │                    │               │ │
│  │                                                                         │ │
│  │  Cal.diy 可见的内容：                                                   │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  ✅ 生成时知道事件内容                                             │   │ │
│  │  │  ❌ 不知道用户是否添加到日历                                        │   │ │
│  │  │  ❌ 不知道用户是否分享链接                                          │   │ │
│  │  │  ❌ 无法控制添加后的事件                                             │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                         │ │
│  │  外部日历服务可见的内容：                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  ✅ 事件的完整内容（如果用户导入）                                   │   │ │
│  │  │  ✅ 用户的日历分享设置可能让事件更广泛可见                           │   │ │
│  │  │  ✅ 可能索引事件内容用于搜索                                         │   │ │
│  │  │  ⚠️ 深层链接的 URL 参数可能被日志记录                                │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 预约暴露给外部日历的完整链路（写入模式）

### 5.1 整体架构图

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

### 5.2 目标日历选择机制

#### 5.2.1 数据模型

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

#### 5.2.2 目标日历优先级

```
优先级（从高到低）：

1. 事件类型级别 DestinationCalendar
   └── eventTypeId 有值时，使用该事件类型的目标日历

2. 用户级别 DestinationCalendar
   └── userId 有值时，使用用户默认目标日历

3. 团队级别（通过 DelegationCredential）
   └── 域委派凭据时，使用团队成员的日历
```

---

## 6. 访问控制机制

### 6.1 三种认证方式对比

| 日历类型 | 认证方式 | 凭证存储 | 刷新机制 |
|---------|---------|---------|---------|
| **Google Calendar** | OAuth 2.0 | `Credential.key`（加密） | 自动刷新 access_token |
| **Outlook/Office 365** | OAuth 2.0 | `Credential.key`（加密） | 自动刷新 access_token |
| **Apple Calendar** | CalDAV Basic Auth | `Credential.key`（加密） | 应用专用密码，无需刷新 |
| **ICS Feed** | 无（依赖外部 URL） | `Credential.key`（加密存储 URL） | 无 |
| **域委派（Enterprise）** | Service Account | `DelegationCredential` | 客户端凭证模式 |

### 6.2 凭证加密存储

所有敏感凭证（包括 OAuth tokens、CalDAV 密码和 ICS URLs）都通过 AES-256 加密存储：

**代码位置**：`packages/lib/crypto.ts`

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

---

## 7. Google / Apple / Outlook 可见性边界差异

### 7.1 三大平台对比总览

| 特性 | Google Calendar | Outlook/Office 365 | Apple Calendar (CalDAV) |
|------|-----------------|---------------------|-------------------------|
| **API 类型** | REST API | Microsoft Graph API | CalDAV (WebDAV) |
| **认证方式** | OAuth 2.0 | OAuth 2.0 | Basic Auth (应用专用密码) |
| **隐私控制** | `visibility` 属性 | `sensitivity` 属性 | `CLASSIFICATION` iCal 属性 |
| **繁忙状态** | `transparency` | `showAs` | `TRANSP` iCal 属性 |
| **参与者可见性** | `guestsCanSeeOtherGuests` | `hideAttendees` | 依赖日历服务实现 |
| **ICS Feed 支持** | 支持私有地址 | 支持发布日历 | 支持公共日历 |

### 7.2 ICS Feed 在三大平台的生成方式

| 平台 | ICS Feed 生成位置 | 默认隐私级别 | 可配置选项 |
|------|------------------|-------------|-----------|
| **Google Calendar** | 日历设置 → 集成日历 → 秘密地址 | 取决于日历分享 | 可选择只显示繁忙时间 |
| **Outlook** | 设置 → 日历 → 共享日历 → 发布日历 | 可选择详细级别 | 仅繁忙/有限详情/完整详情 |
| **Apple Calendar** | 日历右键 → 共享日历 → 公共日历 | 完全公开 | 可设置密码保护 |

### 7.3 单次预约 ICS 在三大平台的行为

| 平台 | 导入方式 | 隐私继承 | Cal.diy 控制力 |
|------|---------|---------|---------------|
| **Google Calendar** | 导入 .ics 或深层链接 | 继承日历默认可见性 | ❌ 无（导入后失控） |
| **Outlook** | 导入 .ics 或深层链接 | 继承日历默认敏感度 | ❌ 无（导入后失控） |
| **Apple Calendar** | 双击 .ics 文件 | 继承日历默认隐私 | ❌ 无（导入后失控） |

---

## 8. 安全边界与最佳实践

### 8.1 两类链接的安全建议

#### 8.1.1 ICS Feed 订阅链接（用户添加外部 ICS 到 Cal.diy）

| 风险点 | 建议 |
|-------|------|
| ICS URL 可能包含敏感信息 | 只从受信任的来源添加 ICS Feed |
| URL 可能被泄露 | 定期轮换外部日历的 ICS Feed URL |
| Cal.diy 存储加密后的 URL | 确保 `CALENDSO_ENCRYPTION_KEY` 足够强 |
| ICS 内容可能被解析 | 了解外部日历服务的 ICS 隐私设置 |

#### 8.1.2 单次预约 ICS 下载链接（Cal.diy 生成）

| 风险点 | 建议 |
|-------|------|
| Data URI 包含完整信息 | 避免通过不安全渠道分享链接 |
| 深层链接参数明文 | 注意浏览器历史记录可能泄露 |
| 导入后完全失控 | 启用 `hideCalendarEventDetails` 保护敏感预约 |
| 邮件附件可能被转发 | 启用 `hideOrganizerEmail` 保护组织者隐私 |

### 8.2 隐私保护机制总览

| 机制 | 适用场景 | 实现位置 |
|------|---------|---------|
| `hideCalendarEventDetails` | 写入外部日历、邮件附件 | 设置 `visibility/private` 或 `CLASSIFICATION:PRIVATE` |
| `hideOrganizerEmail` | 邮件附件 | 使用 `no-reply@cal.com` 替代真实邮箱 |
| `hideCalendarNotes` | 事件描述 | 隐藏事件备注 |
| `symmetricEncrypt` | 所有敏感凭证存储 | AES-256 加密 |
| `IsICSUrlConstraint` | ICS Feed URL 输入 | 协议和路径格式校验 |

### 8.3 访问控制边界

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          访问控制边界                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Cal.diy 可控范围：                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ✓ 写入外部日历时设置隐私属性                                           │  │
│  │  ✓ 加密存储所有敏感凭证（包括 ICS URLs）                                │  │
│  │  ✓ ICS Feed URL 格式校验                                                │  │
│  │  ✓ 只同步 iCalUID 以 @cal.com 结尾的事件                                │  │
│  │  ✓ 验证 booking.userId === calendarUserId                              │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  Cal.diy 不可控范围：                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  ✗ 用户在外部日历中分享包含 Cal.diy 预约的日历                          │  │
│  │  ✗ 用户分享单次预约的 ICS 链接或文件                                    │  │
│  │  ✗ 浏览器历史记录中的深层链接参数                                        │  │
│  │  ✗ 外部日历服务的索引和搜索功能                                          │  │
│  │  ✗ 用户手动修改导入后的事件                                              │  │
│  │  ✗ 外部 ICS Feed 的隐私设置（由外部服务控制）                            │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 关键代码文件索引

| 功能 | 文件路径 | 说明 |
|------|---------|------|
| 加密/解密 | `packages/lib/crypto.ts` | AES-256 实现 |
| ICS Feed 前端 Setup | `apps/web/components/apps/ics-feedcalendar/Setup.tsx` | 用户输入 ICS URL |
| ICS Feed 后端 API | `packages/app-store/ics-feedcalendar/api/add.ts` | Web 应用端点 |
| ICS Feed API v2 | `apps/api/v2/src/platform/calendars/controllers/calendars.controller.ts` | NestJS 端点 |
| ICS Feed URL 校验 | `apps/api/v2/src/platform/calendars/input/create-ics.input.ts` | URL 格式校验器 |
| ICS Feed 服务 | `apps/api/v2/src/platform/calendars/services/ics-feed.service.ts` | 保存/检查逻辑 |
| ICS Feed CalendarService | `packages/app-store/ics-feedcalendar/lib/CalendarService.ts` | 读取和解析 ICS |
| 单次预约 ICS 链接 | `packages/features/bookings/lib/getCalendarLinks.ts` | Data URI 和深层链接 |
| 邮件附件 ICS | `packages/emails/lib/generateIcsString.ts` | 带隐私控制的 ICS |
| 日历服务基类 | `packages/lib/CalendarService.ts` | CalDAV、VTIMEZONE、时区处理 |
| Google Calendar | `packages/app-store/googlecalendar/lib/CalendarService.ts` | Google API 集成 |
| Outlook/Office 365 | `packages/app-store/office365calendar/lib/CalendarService.ts` | Graph API 集成 |
| Apple Calendar | `packages/app-store/applecalendar/lib/CalendarService.ts` | CalDAV 集成 |

---

## 10. 环境变量

| 变量名 | 用途 | 要求 |
|--------|------|------|
| `CALENDSO_ENCRYPTION_KEY` | AES-256 加密密钥 | 32 字节（256 位） |
| `ORGANIZER_EMAIL_EXEMPT_DOMAINS` | 豁免邮箱隐藏的域名列表 | 逗号分隔 |

---

## 11. 总结

### 11.1 三类日历暴露方式的对比

| 方式 | 数据流向 | Cal.diy 角色 | 控制力 | 隐私风险 |
|------|---------|-------------|--------|---------|
| **写入外部日历** | Cal.diy → 外部日历 | 服务端（写入者） | 中等 | 依赖外部日历设置 |
| **ICS Feed 订阅** | 外部日历 → Cal.diy | 客户端（读取者） | 中等 | 依赖外部 ICS 设置 |
| **单次预约 ICS** | Cal.diy → 用户 | 服务端（生成者） | 极低 | 生成后完全失控 |

### 11.2 关键安全原则

1. **加密存储**：所有敏感凭证（包括 ICS URLs）都使用 AES-256 加密
2. **输入校验**：ICS Feed URL 必须通过协议和路径格式校验
3. **隐私默认**：敏感预约应启用 `hideCalendarEventDetails`
4. **边界意识**：明确 Cal.diy 的可控范围和不可控范围
5. **用户教育**：提醒用户单次预约 ICS 链接的分享风险

---

*文档基于代码分析生成，最后更新：2026-05-05*
