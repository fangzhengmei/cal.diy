# Cal.diy 路由表单真实实现分析报告

> 分析日期：2026-05-02
> 分析范围：规则配置存放、匹配顺序、回退处理、预约页筛选逻辑
> 关键结论：路由表单核心功能已从数据库移除，但参数传递和主持人筛选机制保留

---

## 一、功能现状总览

### 1.1 核心发现

| 功能模块 | 状态 | 说明 |
|----------|------|------|
| 路由表单定义（App_RoutingForms_Form） | ❌ 已移除 | 数据库表于 2026-03-05 迁移中删除 |
| 表单响应存储 | ❌ 已移除 | App_RoutingForms_FormResponse 表已删除 |
| 路由规则匹配引擎 | ❌ 已移除 | 无实际代码实现 |
| Headless Router 端点 | ❌ 未找到 | 文档描述的 `/router?formId=...` 无对应代码 |
| routedTeamMemberIds 参数处理 | ✅ 已实现 | 从 URL 解析并传递到查询 |
| 主持人筛选逻辑 | ✅ 已实现 | QualifiedHostsService 中完整实现 |
| 回退处理机制 | ✅ 已实现 | 无匹配时使用所有主持人 |

### 1.2 关键迁移记录

**文件**: `packages/prisma/migrations/20260305043434_remove_routing_forms/migration.sql`

以下路由表单相关数据库表已被移除：
- `App_RoutingForms_Form` - 路由表单定义
- `App_RoutingForms_FormResponse` - 访客答案
- `App_RoutingForms_QueuedFormResponse` - 排队响应
- `App_RoutingForms_IncompleteBookingActions` - 未完成预约动作
- `RoutingTrace` / `PendingRoutingTrace` - 路由追踪
- `RoutingFormResponseDenormalized` - 非规范化响应
- `RoutingFormResponseField` - 响应字段
- `WorkflowsOnRoutingForms` - 工作流关联

以下枚举值已被移除：
- `AssignmentReasonEnum`: `ROUTING_FORM_ROUTING`, `ROUTING_FORM_ROUTING_FALLBACK`
- `WebhookTriggerEvents`: `ROUTING_FORM_FALLBACK_HIT`
- `WorkflowType`: `ROUTING_FORM`

---

## 二、规则配置存放位置与字段结构

### 2.1 已实现：查询参数定义

**文件**: `packages/trpc/server/routers/viewer/slots/types.ts:33-39`

```typescript
export const getScheduleSchemaObject = z.object({
  // ... 其他参数
  routedTeamMemberIds: z.array(z.number()).nullish(),
  skipContactOwner: z.boolean().nullish(),
  rrHostSubsetIds: z.array(z.number()).nullish(),
  queuedFormResponseId: z.string().nullish(),
  email: z.string().nullish(),
});
```

**字段说明**：

| 参数名 | 类型 | 用途 | 代码位置 |
|--------|------|------|----------|
| `routedTeamMemberIds` | `number[] \| null` | 路由目标团队成员 ID 列表 | `types.ts:33` |
| `skipContactOwner` | `boolean \| null` | 是否跳过联系负责人查找 | `types.ts:34` |
| `rrHostSubsetIds` | `number[] \| null` | 轮询主持人子集 ID（额外筛选） | `types.ts:35` |
| `queuedFormResponseId` | `string \| null` | 排队表单响应 ID（异步处理） | `types.ts:39` |
| `teamMemberEmail` | `string \| null` | 团队成员邮箱（联系负责人） | `types.ts:32` |

### 2.2 已实现：URL 查询参数解析

**文件**: `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts`

```typescript
export const getRoutedTeamMemberIdsFromSearchParams = (
  searchParams: URLSearchParams
) => {
  const routedTeamMemberIdsParam = searchParams.get(
    "cal.routedTeamMemberIds"
  );
  const routedTeamMemberIds =
    typeof routedTeamMemberIdsParam === "string"
      ? routedTeamMemberIdsParam
          .split(",")
          .filter(Boolean)
          .map((id) => parseInt(id, 10))
      : null;
  return routedTeamMemberIds;
};
```

**解析规则**：
- 查询参数名：`cal.routedTeamMemberIds`
- 格式：逗号分隔的数字列表，如 `cal.routedTeamMemberIds=10,15,22`
- 返回值：`number[]` 或 `null`

### 2.3 仅文档描述：已移除的数据库结构

根据历史迁移文件 `packages/prisma/migrations/20220616072241_app_routing_forms/migration.sql`，原路由表单表结构为：

```sql
CREATE TABLE "App_RoutingForms_Form" (
    "id" TEXT NOT NULL,
    "description" TEXT,
    "routes" JSONB,           -- 路由规则定义（核心）
    "createdAt" TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP,
    "updatedAt" TIMESTAMP(3) NOT NULL,
    "name" TEXT NOT NULL,
    "fields" JSONB,           -- 表单字段定义
    "userId" INTEGER NOT NULL,
    "teamId" INTEGER,          -- 后加：支持团队级表单
    "settings" JSONB,          -- 后加：表单设置
    "disabled" BOOLEAN NOT NULL DEFAULT false,
    CONSTRAINT "App_RoutingForms_Form_pkey" PRIMARY KEY ("id")
);
```

**⚠️ 重要说明**：
- 此表结构**已不存在**于当前 `schema.prisma`
- `routes` 和 `fields` JSONB 字段的具体结构**无代码实现可验证**
- 所有基于此表的路由匹配逻辑**已被移除**

### 2.4 仅文档描述：Headless Router 端点

**文档来源**: `headless-routing-to-booking-flow.md`

文档描述的访问方式：
```
GET /router?formId={FORM_ID}&field1=value1&field2=value2
```

**实际状态**：
- ❌ 未在 `apps/web/pages/api/` 找到对应路由
- ❌ 未在 `apps/web/app/api/` 找到对应路由
- ❌ 无任何代码实现此端点

---

## 三、提交后的匹配顺序与终止条件

### 3.1 已实现：预约页面参数传递链路

**文件**: `apps/web/modules/schedules/hooks/useSchedule.ts:77-114`

```typescript
export const useSchedule = ({ /* 参数 */ }) => {
  const searchParams = useSearchParams();
  
  // 从 URL 解析 routedTeamMemberIds
  const routedTeamMemberIds = searchParams
    ? getRoutedTeamMemberIdsFromSearchParams(
        new URLSearchParams(searchParams.toString())
      )
    : null;
  
  const skipContactOwner = searchParams 
    ? searchParams.get("cal.skipContactOwner") === "true" 
    : false;
  
  const queuedFormResponseId = searchParams?.get("cal.queuedFormResponseId");
  const email = searchParams?.get("email");

  // 构建查询输入
  const input = {
    // ... 其他参数
    routedTeamMemberIds,
    skipContactOwner,
    ...(queuedFormResponseId ? { queuedFormResponseId } : {}),
    email,
    // ...
  };

  // 调用 tRPC 查询
  const schedule = trpc.viewer.slots.getSchedule.useQuery(input, options);
  // ...
};
```

**传递顺序**：
1. URL 查询参数 → `useSearchParams()`
2. `getRoutedTeamMemberIdsFromSearchParams()` 解析
3. 构建 `input` 对象传递给 `getSchedule`
4. tRPC 路由传递到后端处理

### 3.2 已实现：skipContactOwner 判断逻辑

**文件**: `packages/lib/bookings/routing/utils.ts:16-27`

```typescript
export function isRerouting({
  rescheduleUid,
  routedTeamMemberIds,
}: {
  rescheduleUid: string | null;
  routedTeamMemberIds: number[] | null;
}) {
  // 只有同时存在 rescheduleUid 和非空 routedTeamMemberIds 才算重新路由
  return !!rescheduleUid && !!routedTeamMemberIds?.length;
}

export function shouldIgnoreContactOwner({
  skipContactOwner,
  rescheduleUid,
  routedTeamMemberIds,
}: {
  skipContactOwner: boolean | null;
  rescheduleUid: string | null;
  routedTeamMemberIds: number[] | null;
}) {
  // 终止条件：显式跳过 OR 处于重新路由状态
  // 原因：重新路由时，联系负责人可能不在 routedTeamMemberIds 中
  return skipContactOwner || isRerouting({ rescheduleUid, routedTeamMemberIds });
}
```

**终止条件（停止联系负责人查找）**：
| 条件 | 说明 | 代码位置 |
|------|------|----------|
| `skipContactOwner === true` | URL 参数 `cal.skipContactOwner=true` | `utils.ts:26` |
| `isRerouting() === true` | 同时存在 `rescheduleUid` 和非空 `routedTeamMemberIds` | `utils.ts:13` |

### 3.3 仅文档描述：路由规则匹配引擎

**文档来源**: `headless-routing-to-booking-flow.md`

文档描述的匹配流程：
```
1. 验证字段类型和必填字段
2. 基于字段值选择路由（按优先级）
3. 记录表单响应
4. 如果是 eventTypeRedirect：
   - 识别匹配属性路由规则的团队成员
   - 重定向到预约页面，携带 routedTeamMemberIds
5. 发送邮件通知和触发 Webhook
```

**实际状态**：
- ❌ 无任何代码实现此匹配流程
- ❌ `routes` JSONB 字段的具体结构无定义
- ❌ 优先级排序、短路匹配逻辑不存在

### 3.4 仅文档描述：属性路由（Attribute Routing）

**文档来源**: `headless-routing-to-booking-flow.md`

文档描述的功能：
> "Identify the teamMembers matching that route's attribute routing rules."

即基于表单答案动态筛选符合属性条件的团队成员，如：
- 答案为"销售咨询" → 筛选 `department === "Sales"` 的成员
- 答案为"技术支持" → 筛选 `department === "Support"` 的成员

**实际状态**：
- ❌ 无代码实现基于表单答案的动态筛选
- ✅ 仅支持**直接传递** `routedTeamMemberIds` 参数（外部系统指定）
- 成员属性匹配逻辑不存在

---

## 四、无匹配或成员为空时的回退处理

### 4.1 已实现：QualifiedHostsService 回退逻辑

**文件**: `apps/api/v2/src/lib/services/qualified-hosts.service.ts:55-85`

```typescript
let qualifiedRRHosts = allRRHosts;

if (contactOwnerEmail) {
  // 优先匹配联系负责人
  const contactOwnerHost = allRRHosts.filter(
    (h) => (h.user as { email?: string }).email === contactOwnerEmail
  );
  if (contactOwnerHost.length > 0) {
    qualifiedRRHosts = contactOwnerHost;
  }
} else if (routedTeamMemberIds.length > 0) {
  // 使用 routedTeamMemberIds 筛选
  const routedMemberIdSet = new Set(routedTeamMemberIds);
  const routedHosts = allRRHosts.filter(
    (h) => routedMemberIdSet.has((h.user as { id: number }).id)
  );
  // ⚠️ 关键回退逻辑
  if (routedHosts.length > 0) {
    qualifiedRRHosts = routedHosts;
  }
  // ❌ 如果 routedHosts 为空，qualifiedRRHosts 保持为 allRRHosts
}

// 返回结果
return { 
  qualifiedRRHosts,      // 合格的轮询主持人
  allFallbackRRHosts: allRRHosts,  // 回退主持人（所有）
  fixedHosts 
};
```

**回退规则 1（routedTeamMemberIds 无匹配）**：

| 场景 | `routedHosts.length` | `qualifiedRRHosts` 值 |
|------|----------------------|----------------------|
| 完全匹配 | > 0 | `routedHosts`（仅匹配的成员） |
| 部分匹配 | > 0 | `routedHosts`（仅存在的成员） |
| **无匹配** | **= 0** | **`allRRHosts`（所有主持人）** |

**代码分析**：
- `qualifiedRRHosts` 初始化为 `allRRHosts`（所有轮询主持人）
- 只有当 `routedHosts.length > 0` 时才更新
- 如果 `routedTeamMemberIds` 中的 ID 都不在 `allRRHosts` 中，`qualifiedRRHosts` 保持为 `allRRHosts`

### 4.2 已实现：getSchedule 中的 2 周回退机制

**文件**: `packages/trpc/server/routers/viewer/slots/util.ts:1001-1088`

```typescript
// 判断是否有回退主持人
const hasFallbackRRHosts =
  eligibleFallbackRRHosts.length > 0 && 
  eligibleFallbackRRHosts.length > eligibleQualifiedRRHosts.length;

// ... 计算 qualifiedRRHosts 的可用性 ...

if (hasFallbackRRHosts) {
  let diff = 0;
  if (startTime.isBefore(twoWeeksFromNow)) {
    // 检查未来 2 周内是否有可用时段
    diff = aggregatedAvailability.length > 0
      ? aggregatedAvailability[0].start.diff(twoWeeksFromNow, "day")
      : 1; // 没有可用时段 → diff = 1
  } else {
    // 如果开始时间不在 2 周内，检查是否有任何可用时段
    if (!aggregatedAvailability.length) {
      // 没有可用时段，检查前 2 周
      const firstTwoWeeksAvailabilities = await this.calculateHostsAndAvailabilities({
        hosts: [...eligibleQualifiedRRHosts, ...eligibleFixedHosts],
        startTime: dayjs(),
        endTime: twoWeeksFromNow,
        // ...
      });
      if (!getAggregatedAvailability(
        firstTwoWeeksAvailabilities.allUsersAvailability,
        eventType.schedulingType
      ).length) {
        // 前 2 周也没有 → diff = 1
        diff = 1;
      }
    }
  }

  // ⚠️ 触发回退
  if (diff > 0) {
    // 使用回退主持人重新计算可用性
    ({ allUsersAvailability, usersWithCredentials, currentSeats } =
      await this.calculateHostsAndAvailabilities({
        hosts: [...eligibleFallbackRRHosts, ...eligibleFixedHosts],
        // ^^^^ 使用 allFallbackRRHosts（所有主持人）
        startTime,
        endTime,
        // ...
      }));
    aggregatedAvailability = getAggregatedAvailability(
      allUsersAvailability, 
      eventType.schedulingType
    );
  }
}
```

**回退规则 2（2 周内无可用时段）**：

```
┌─────────────────────────────────────────────────────────────────┐
│  输入：qualifiedRRHosts（由 routedTeamMemberIds 筛选的主持人）   │
│         allFallbackRRHosts（所有轮询主持人）                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
              检查 qualifiedRRHosts 未来 2 周可用性
                              ↓
              ┌───────────────────────────────┐
              │  2 周内有可用时段？            │
              └───────────────────────────────┘
                    ↙              ↘
                  是                否
                    ↓                ↓
           使用 qualifiedRRHosts    触发回退
                                    使用 allFallbackRRHosts
                                    （所有轮询主持人）
```

### 4.3 已实现：所有主持人都被阻止时的回退

**文件**: `packages/trpc/server/routers/viewer/slots/util.ts:989-997`

```typescript
const allHosts = [...eligibleQualifiedRRHosts, ...eligibleFixedHosts];

// 如果所有主持人都被阻止（观看名单），返回空时段
if (allHosts.length === 0) {
  loggerWithEventDetails.info(
    "All hosts are blocked by watchlist, returning empty slots"
  );
  return {
    slots: {},
  };
}
```

**回退规则 3（所有主持人被阻止）**：
- 如果 `eligibleQualifiedRRHosts` + `eligibleFixedHosts` 为空
- 返回空的时段对象 `{ slots: {} }`
- 不使用其他回退机制

### 4.4 回退机制汇总

| 回退场景 | 触发条件 | 回退行为 | 代码位置 |
|----------|----------|----------|----------|
| routedTeamMemberIds 无匹配 | `routedHosts.length === 0` | 使用所有轮询主持人 | `qualified-hosts.service.ts:64-70` |
| 2 周内无可用时段 | `qualifiedRRHosts` 未来 2 周没空 | 使用所有轮询主持人 | `slots/util.ts:1073-1088` |
| 所有主持人被阻止 | `allHosts.length === 0` | 返回空时段 | `slots/util.ts:992-997` |

---

## 五、routedTeamMemberIds 传到预约页后的筛选逻辑

### 5.1 完整调用链路

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. 访客访问预约页面                                                   │
│     URL: /team/sales-meeting?cal.routedTeamMemberIds=10,15,22      │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  2. 预约页面前端（useSchedule）                                        │
│     文件: apps/web/modules/schedules/hooks/useSchedule.ts             │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ searchParams.get("cal.routedTeamMemberIds")                  │  │
│     │ getRoutedTeamMemberIdsFromSearchParams() → [10, 15, 22]    │  │
│     │ 传递给 trpc.viewer.slots.getSchedule.query()                  │  │
│     └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  3. tRPC 路由处理                                                      │
│     文件: packages/trpc/server/routers/viewer/slots/util.ts          │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ _getAvailableSlots() 接收 input.routedTeamMemberIds          │  │
│     │ 调用 qualifiedHostsService.findQualifiedHostsWith...()       │  │
│     └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  4. 主持人筛选服务                                                      │
│     文件: apps/api/v2/src/lib/services/qualified-hosts.service.ts    │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ 从 eventType.hosts 获取所有轮询主持人（allRRHosts）           │  │
│     │ 用 routedTeamMemberIds 过滤 → qualifiedRRHosts               │  │
│     │ 返回 { qualifiedRRHosts, allFallbackRRHosts, fixedHosts }   │  │
│     └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  5. 可用性计算                                                          │
│     文件: packages/trpc/server/routers/viewer/slots/util.ts          │
│     ┌─────────────────────────────────────────────────────────────┐  │
│     │ calculateHostsAndAvailabilities()                             │  │
│     │ - 只查询 qualifiedRRHosts 的日历可用性                        │  │
│     │ - 2 周内无可用时段时，回退到 allFallbackRRHosts              │  │
│     └─────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────┐
│  6. 返回可用时段                                                       │
│     只显示 routedTeamMemberIds 中成员的可用时段（或回退后的所有成员）  │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 核心筛选代码

**文件**: `apps/api/v2/src/lib/services/qualified-hosts.service.ts:23-85`

```typescript
@Injectable()
export class QualifiedHostsService {
  async findQualifiedHostsWithDelegationCredentials(...args: unknown[]): Promise<{
    qualifiedRRHosts: QualifiedHost[];
    allFallbackRRHosts: QualifiedHost[];
    fixedHosts: QualifiedHost[];
  }> {
    const input = (args[0] ?? {}) as Record<string, unknown>;
    const eventType = (input.eventType ?? {}) as Record<string, unknown>;
    const contactOwnerEmail = input.contactOwnerEmail as string | null | undefined;
    const routedTeamMemberIds = (input.routedTeamMemberIds ?? []) as number[];

    const hosts = (eventType.hosts ?? []) as Host[];
    const users = (eventType.users ?? []) as User[];
    const schedulingType = eventType.schedulingType as string | null | undefined;

    if (hosts.length > 0) {
      const fixedHosts: QualifiedHost[] = [];
      const allRRHosts: QualifiedHost[] = [];

      // 分离固定主持人和轮询主持人
      for (const host of hosts) {
        const qualifiedHost: QualifiedHost = {
          user: host.user,
          isFixed: host.isFixed,
          groupId: host.groupId ?? null,
        };

        if (host.isFixed || schedulingType !== "ROUND_ROBIN") {
          // 固定主持人 或 非轮询事件 → 固定主持人
          fixedHosts.push(qualifiedHost);
        } else {
          // 轮询主持人
          allRRHosts.push(qualifiedHost);
        }
      }

      // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
      // 核心筛选逻辑
      // ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
      let qualifiedRRHosts = allRRHosts; // 默认所有轮询主持人

      if (contactOwnerEmail) {
        // 优先级 1：联系负责人邮箱匹配
        const contactOwnerHost = allRRHosts.filter(
          (h) => (h.user as { email?: string }).email === contactOwnerEmail
        );
        if (contactOwnerHost.length > 0) {
          qualifiedRRHosts = contactOwnerHost;
        }
      } else if (routedTeamMemberIds.length > 0) {
        // 优先级 2：routedTeamMemberIds 筛选
        const routedMemberIdSet = new Set(routedTeamMemberIds);
        const routedHosts = allRRHosts.filter(
          (h) => routedMemberIdSet.has((h.user as { id: number }).id)
        );
        // 只有匹配成功才更新（无匹配时保持 allRRHosts）
        if (routedHosts.length > 0) {
          qualifiedRRHosts = routedHosts;
        }
      }

      return { 
        qualifiedRRHosts,      // 筛选后的轮询主持人
        allFallbackRRHosts: allRRHosts,  // 回退用的所有轮询主持人
        fixedHosts             // 固定主持人（不受筛选影响）
      };
    }

    // 如果没有 hosts 配置，使用 users 作为固定主持人
    if (users.length > 0) {
      const fixedHosts = users.map((user) => ({
        user,
        isFixed: true as const,
        groupId: null,
      }));
      return { qualifiedRRHosts: [], allFallbackRRHosts: [], fixedHosts };
    }

    return { qualifiedRRHosts: [], allFallbackRRHosts: [], fixedHosts: [] };
  }
}
```

### 5.3 筛选优先级

| 优先级 | 筛选条件 | 触发条件 | 代码位置 |
|--------|----------|----------|----------|
| 1 | 联系负责人邮箱 | `contactOwnerEmail` 存在且匹配 | `qualified-hosts.service.ts:57-63` |
| 2 | routedTeamMemberIds | `routedTeamMemberIds.length > 0` 且匹配 | `qualified-hosts.service.ts:64-70` |
| 3 | 默认（所有） | 以上条件都不满足 | `qualified-hosts.service.ts:55` |

**优先级判断逻辑**：
```
if (contactOwnerEmail) {
  // 优先使用联系负责人
} else if (routedTeamMemberIds.length > 0) {
  // 其次使用 routedTeamMemberIds
}
// 都不满足时，qualifiedRRHosts 保持为 allRRHosts
```

### 5.4 筛选结果使用

**文件**: `packages/trpc/server/routers/viewer/slots/util.ts:964-997`

```typescript
// 调用筛选服务
const { qualifiedRRHosts, allFallbackRRHosts, fixedHosts } =
  await this.dependencies.qualifiedHostsService.findQualifiedHostsWithDelegationCredentials({
    eventType,
    rescheduleUid: input.rescheduleUid ?? null,
    routedTeamMemberIds,
    contactOwnerEmail,
    rrHostSubsetIds: input.rrHostSubsetIds ?? undefined,
  });

// 过滤被阻止的主持人
const { eligibleHosts: eligibleQualifiedRRHosts } = await filterBlockedHosts(
  qualifiedRRHosts,
  organizationId
);
const { eligibleHosts: eligibleFixedHosts } = await filterBlockedHosts(fixedHosts, organizationId);
const { eligibleHosts: eligibleFallbackRRHosts } = allFallbackRRHosts
  ? await filterBlockedHosts(allFallbackRRHosts, organizationId)
  : { eligibleHosts: [] };

// 构建实际查询的主持人列表
const allHosts = [...eligibleQualifiedRRHosts, ...eligibleFixedHosts];

// 所有主持人都被阻止时返回空
if (allHosts.length === 0) {
  return { slots: {} };
}

// 计算可用性（只查询 allHosts）
let { allUsersAvailability, usersWithCredentials, currentSeats } =
  await this.calculateHostsAndAvailabilities({
    hosts: allHosts,  // <-- 只查询筛选后的主持人
    // ...
  });
```

### 5.5 预约页面使用示例

**文件**: `apps/web/modules/schedules/hooks/useSchedule.ts:77-83`

```typescript
const searchParams = useSearchParams();
const routedTeamMemberIds = searchParams
  ? getRoutedTeamMemberIdsFromSearchParams(
      new URLSearchParams(searchParams.toString())
    )
  : null;
```

**支持的 URL 参数**：

| 参数名 | 格式示例 | 用途 | 代码位置 |
|--------|----------|------|----------|
| `cal.routedTeamMemberIds` | `10,15,22` | 路由目标成员 ID | `getRoutedTeamMemberIdsFromSearchParams.ts` |
| `cal.skipContactOwner` | `true` | 跳过联系负责人查找 | `useSchedule.ts:80` |
| `cal.queuedFormResponseId` | `abc123` | 排队响应 ID（异步） | `useSchedule.ts:82` |
| `cal.email` | `user@example.com` | 访客邮箱（用于日志） | `useSchedule.ts:83` |

---

## 六、功能现状总结

### 6.1 已实现的功能（有代码支持）

| 功能模块 | 代码位置 | 状态 |
|----------|----------|------|
| URL 参数解析 | `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts` | ✅ 完整 |
| 参数类型定义 | `packages/trpc/server/routers/viewer/slots/types.ts:33-39` | ✅ 完整 |
| 预约页面参数传递 | `apps/web/modules/schedules/hooks/useSchedule.ts:77-114` | ✅ 完整 |
| skipContactOwner 判断 | `packages/lib/bookings/routing/utils.ts` | ✅ 完整 |
| 主持人筛选服务 | `apps/api/v2/src/lib/services/qualified-hosts.service.ts` | ✅ 完整 |
| 2 周回退机制 | `packages/trpc/server/routers/viewer/slots/util.ts:1001-1088` | ✅ 完整 |
| 无匹配回退（使用所有主持人） | `qualified-hosts.service.ts:64-70` | ✅ 完整 |

### 6.2 仅文档描述/已移除的功能

| 功能模块 | 文档来源 | 状态 |
|----------|----------|------|
| 路由表单定义表 | 迁移历史 | ❌ 已移除（2026-03-05） |
| 表单响应存储 | 迁移历史 | ❌ 已移除 |
| 路由规则匹配引擎 | `headless-routing-to-booking-flow.md` | ❌ 无代码实现 |
| Headless Router 端点 | `headless-routing-to-booking-flow.md` | ❌ 无代码实现 |
| 属性路由（基于答案筛选成员） | `headless-routing-to-booking-flow.md` | ❌ 无代码实现 |
| 邮件通知和 Webhook | `headless-routing-to-booking-flow.md` | ❌ 枚举值已移除 |

### 6.3 当前可用的使用方式

由于路由表单核心功能已移除，当前只能通过以下方式使用参数传递机制：

**方式 1：直接 URL 参数传递**
```
https://cal.example.com/team/meeting?cal.routedTeamMemberIds=10,15,22&cal.skipContactOwner=true
```

**方式 2：嵌入代码传递**
```javascript
// Embed SDK 中传递
Cal("init", {
  orgSlug: "acme",
  cal.routedTeamMemberIds: "10,15,22",
  // ...
});
```

**方式 3：API 直接调用**
```typescript
// tRPC 调用
await trpc.viewer.slots.getSchedule.query({
  eventTypeSlug: "meeting",
  usernameList: ["team"],
  startTime: "2026-05-03T00:00:00Z",
  endTime: "2026-06-03T00:00:00Z",
  timeZone: "UTC",
  routedTeamMemberIds: [10, 15, 22],
  skipContactOwner: true,
});
```

### 6.4 关键代码索引

| 文件路径 | 功能说明 | 行号 |
|----------|----------|------|
| `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts` | 从 URL 解析 routedTeamMemberIds | 完整文件 |
| `packages/lib/bookings/routing/utils.ts` | isRerouting / shouldIgnoreContactOwner | 1-27 |
| `packages/trpc/server/routers/viewer/slots/types.ts` | 参数类型定义 | 33-39 |
| `apps/web/modules/schedules/hooks/useSchedule.ts` | 预约页面参数传递 | 77-114 |
| `apps/api/v2/src/lib/services/qualified-hosts.service.ts` | 主持人筛选核心逻辑 | 23-85 |
| `packages/trpc/server/routers/viewer/slots/util.ts` | 2 周回退机制 | 1001-1088 |
| `packages/trpc/server/routers/viewer/slots/util.ts` | 无匹配回退逻辑 | 953-971 |

---

## 七、与文档描述的差异对比

### 7.1 文档描述 vs 实际实现

| 文档描述（headless-routing-to-booking-flow.md） | 实际实现状态 | 差异说明 |
|--------------------------------------------------|--------------|----------|
| `GET /router?formId={FORM_ID}&field1=value1...` | ❌ 不存在 | 无此端点 |
| 验证字段类型和必填字段 | ❌ 不存在 | 无代码实现 |
| 基于字段值选择路由（按优先级） | ❌ 不存在 | 无规则匹配引擎 |
| 记录表单响应 | ❌ 已移除 | 表已删除 |
| **eventTypeRedirect + 属性路由** | ⚠️ 部分实现 | 只支持直接传递 ID，不支持基于答案的属性匹配 |
| 重定向到预约页面 + `routedTeamMemberIds` | ✅ 部分实现 | 参数传递支持，但路由引擎不存在 |
| 发送邮件通知 | ❌ 已移除 | 枚举值已删除 |
| 触发 Webhook | ❌ 已移除 | `ROUTING_FORM_FALLBACK_HIT` 枚举已删除 |

### 7.2 关键差异点

**1. 路由规则定义**
- 文档：`routes` JSONB 字段存储规则，支持条件表达式、优先级、动作类型
- 实际：表已删除，无任何规则定义和匹配逻辑

**2. 属性路由（Attribute Routing）**
- 文档：基于表单答案动态匹配成员属性（如 `department === "Sales"`）
- 实际：只支持外部直接传递 `routedTeamMemberIds`，无动态计算

**3. Headless Router 端点**
- 文档：`/router?formId=...` 入口，接收表单答案
- 实际：无此端点，代码库中找不到对应实现

**4. 回退处理**
- 文档：`redirectUrlOnNoRoutingFormResponse`、`fallbackAction`
- 实际：
  - `redirectUrlOnNoRoutingFormResponse` 字段已删除
  - `fallbackAction` 相关代码已删除
  - 仅存的回退：无匹配时使用所有主持人、2 周无空时使用所有主持人

---

## 八、结论与建议

### 8.1 核心结论

1. **路由表单功能已从产品中移除**
   - 所有相关数据库表已于 2026-03-05 删除
   - 路由规则匹配引擎、Headless Router 端点无代码实现
   - `headless-routing-to-booking-flow.md` 描述的是**已移除的功能**

2. **参数传递和筛选机制保留**
   - `routedTeamMemberIds`、`skipContactOwner` 等参数的解析和传递完整实现
   - `QualifiedHostsService` 中的主持人筛选逻辑完整可用
   - 2 周回退机制、无匹配回退逻辑完整实现

3. **文档与代码不一致**
   - `headless-routing-to-booking-flow.md` 描述的是旧功能
   - 该文档未随功能移除而更新或删除

### 8.2 如果需要恢复路由表单功能

如果需要重新实现路由表单功能，建议：

1. **数据库层**
   - 恢复 `App_RoutingForms_Form`、`App_RoutingForms_FormResponse` 等表
   - 明确定义 `routes` JSONB 的 schema（使用 Zod 或 JSON Schema）

2. **路由引擎**
   - 实现规则解析器（解析 `routes` JSON）
   - 实现条件匹配引擎（支持 AND/OR、比较操作符）
   - 实现属性路由（基于答案动态计算 `routedTeamMemberIds`）

3. **API 端点**
   - 实现 `GET /router?formId=...&field1=value1` 端点
   - 实现 `POST /forms/{formId}/submit` 端点

4. **文档更新**
   - 更新或删除 `headless-routing-to-booking-flow.md`
   - 添加新功能的技术文档

### 8.3 建议的代码清理

当前代码库中存在以下不一致，建议清理：

1. **`queuedFormResponseId` 参数**
   - 在 `types.ts:39` 中定义
   - 在 `slots/util.ts:902` 中解构但**从未使用**
   - 建议：移除或实现其用途

2. **`headless-routing-to-booking-flow.md`**
   - 描述的是已移除的功能
   - 建议：删除或标记为"已废弃"

3. **`packages/lib/bookings/routing/` 目录**
   - 只包含 `utils.ts` 和 `utils.test.ts`
   - 功能有限且与路由表单核心功能无关
   - 建议：保留但重命名，或合并到相关模块
