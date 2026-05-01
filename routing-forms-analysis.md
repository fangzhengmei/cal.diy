# Cal.diy 路由表单功能分析报告

## 一、功能概述

路由表单（Routing Forms）是 Cal.diy 中一个用于根据访客答案智能路由到不同预约入口的功能。该功能允许用户创建表单，设置条件路由规则，访客填写表单后，系统会根据其答案自动判断应该跳转到哪个预约页面。

**重要说明**：根据数据库迁移记录，路由表单的核心数据库表已于 2026-03-05 的迁移中被移除，但相关的查询参数处理逻辑（如 `routedTeamMemberIds`）仍然保留在代码库中，用于支持遗留场景和外部集成。

---

## 二、条件路由规则定义

### 2.1 数据结构

路由表单的定义存储在 `App_RoutingForms_Form` 表中，核心字段包括：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | TEXT | 表单唯一标识 |
| `name` | TEXT | 表单名称 |
| `description` | TEXT | 表单描述 |
| `fields` | JSONB | 表单字段定义（访客需要填写的问题） |
| `routes` | JSONB | **路由规则定义**（核心） |
| `settings` | JSONB | 表单设置 |
| `userId` | INTEGER | 创建者用户ID |
| `teamId` | INTEGER | 团队ID（支持团队级路由表单） |
| `disabled` | BOOLEAN | 是否禁用 |

### 2.2 路由规则结构

路由规则存储在 `routes` JSONB 字段中，每个规则定义了：

- **条件表达式**：判断访客答案是否满足该规则
- **目标动作**：满足条件后执行的动作
- **优先级**：多个规则的匹配顺序

#### 规则类型

根据 `headless-routing-to-booking-flow.md` 文档，路由规则主要支持以下动作类型：

1. **eventTypeRedirect**（事件类型重定向）
   - 跳转到指定的事件类型预约页面
   - 支持属性路由（Attribute Routing）来匹配特定团队成员

2. **外部 URL 重定向**
   - 跳转到外部 URL

3. **自定义页面**
   - 显示自定义消息或页面

### 2.3 条件表达式

条件表达式基于访客的表单答案进行判断，支持：

- **等于/不等于**：`fieldValue == "value"`
- **包含/不包含**：多选字段的包含判断
- **大于/小于**：数值字段的比较
- **逻辑组合**：AND/OR 组合多个条件

---

## 三、路由引擎判断逻辑

### 3.1 核心流程

根据 `headless-routing-to-booking-flow.md` 文档，路由引擎的工作流程如下：

```
1. 接收请求 → /router?formId={FORM_ID}&field1=value1&field2=value2
         ↓
2. 验证字段 → 检查字段类型和必填字段
         ↓
3. 匹配路由 → 按优先级依次检查路由规则
         ↓
4. 记录响应 → 保存 App_RoutingForms_FormResponse
         ↓
5. 执行动作 → 根据匹配的路由规则执行对应动作
         ↓
6. 发送通知 → 邮件通知 + Webhook 触发
```

### 3.2 路由匹配算法

路由引擎按以下顺序处理规则：

1. **按优先级排序**：高优先级规则先检查
2. **短路匹配**：找到第一个匹配的规则后立即停止
3. **回退机制**：如果没有规则匹配，使用默认路由或回退动作

### 3.3 属性路由（Attribute Routing）

对于 `eventTypeRedirect` 类型的路由，支持**属性路由**来进一步筛选团队成员：

**工作原理**：
1. 路由规则指定目标 `eventTypeId`
2. 同时定义成员属性匹配条件（如：`department == "Sales"`）
3. 路由引擎找出该事件类型下所有满足属性条件的团队成员
4. 将这些成员的 ID 作为 `routedTeamMemberIds` 传递到预约页面

**代码实现位置**：
- `apps/api/v2/src/lib/services/qualified-hosts.service.ts:64-70`

```typescript
} else if (routedTeamMemberIds.length > 0) {
  const routedMemberIdSet = new Set(routedTeamMemberIds);
  const routedHosts = allRRHosts.filter((h) => 
    routedMemberIdSet.has((h.user as { id: number }).id)
  );
  if (routedHosts.length > 0) {
    qualifiedRRHosts = routedHosts;
  }
}
```

---

## 四、完整链路分析

### 4.1 端到端流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           访客填写表单阶段                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. 访客访问路由表单页面                                                        │
│     URL: /forms/{formId} 或嵌入到外部页面                                       │
│                                                                              │
│  2. 访客填写表单字段                                                           │
│     - 文本输入、单选、多选、下拉等                                              │
│     - 支持条件显示字段（根据前面的答案显示不同问题）                               │
│                                                                              │
│  3. 访客提交表单                                                              │
│     → POST /api/forms/{formId}/submit                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                           路由引擎处理阶段                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  4. 后端接收表单数据                                                           │
│     - 验证字段类型和格式                                                        │
│     - 检查必填字段是否完整                                                       │
│                                                                              │
│  5. 加载路由表单配置                                                           │
│     → 从 App_RoutingForms_Form 表读取 fields 和 routes                        │
│                                                                              │
│  6. 路由规则匹配                                                              │
│     ┌─────────────────────────────────────────────────────────────────────┐│
│     │ for each route in form.routes (按优先级排序):                         ││
│     │   - 解析条件表达式                                                      ││
│     │   - 代入访客答案进行评估                                                 ││
│     │   - 如果匹配成功:                                                       ││
│     │       → 记录匹配的路由规则                                              ││
│     │       → break (短路匹配)                                               ││
│     └─────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  7. 保存表单响应                                                              │
│     → 插入 App_RoutingForms_FormResponse 表                                  │
│     → 包含 formId、visitorId、response(JSONB)                                │
│                                                                              │
│  8. 处理路由动作                                                              │
│     ┌─────────────────────────────────────────────────────────────────────┐│
│     │ 根据匹配的路由规则类型：                                                ││
│     │                                                                       ││
│     │ 类型 A: eventTypeRedirect                                            ││
│     │   - 获取目标 eventTypeId                                              ││
│     │   - 执行属性路由匹配（如果有）                                          ││
│     │   - 找出符合条件的团队成员 → routedTeamMemberIds                      ││
│     │   - 构建重定向 URL: /{user}/{eventSlug}                              ││
│     │                                                                       ││
│     │ 类型 B: 外部 URL 重定向                                               ││
│     │   - 直接使用配置的外部 URL                                             ││
│     │                                                                       ││
│     │ 类型 C: 无匹配（回退）                                                 ││
│     │   - 使用表单的 fallbackAction                                         ││
│     │   - 或事件类型的 redirectUrlOnNoRoutingFormResponse                  ││
│     └─────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  9. 触发通知和 Webhook                                                        │
│     - 发送邮件给需要通知的成员                                                 │
│     - 触发 FORM_SUBMITTED Webhook                                            │
│     - 如果是回退路由，触发 ROUTING_FORM_FALLBACK_HIT Webhook                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                           重定向到预约页面阶段                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  10. 返回重定向响应                                                           │
│      HTTP 302 / 303 重定向                                                    │
│      Location: /{user}/{eventSlug}?cal.routedTeamMemberIds=1,2,3           │
│                                                                              │
│      查询参数说明：                                                           │
│      - cal.routedTeamMemberIds: 逗号分隔的团队成员 ID 列表                     │
│      - cal.skipContactOwner: 是否跳过联系负责人查找                            │
│      - cal.queuedFormResponseId: 排队的表单响应 ID（异步处理时）                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                           预约页面处理阶段                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│  11. 预约页面加载                                                             │
│      → 解析 URL 查询参数                                                      │
│                                                                              │
│  12. 解析 routedTeamMemberIds                                                 │
│      代码位置: packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts│
│                                                                              │
│      ```typescript                                                           │
│      export const getRoutedTeamMemberIdsFromSearchParams = (                │
│        searchParams: URLSearchParams                                         │
│      ) => {                                                                   │
│        const routedTeamMemberIdsParam = searchParams.get(                   │
│          "cal.routedTeamMemberIds"                                           │
│        );                                                                     │
│        return typeof routedTeamMemberIdsParam === "string"                  │
│          ? routedTeamMemberIdsParam                                          │
│              .split(",")                                                      │
│              .filter(Boolean)                                                 │
│              .map((id) => parseInt(id, 10))                                  │
│          : null;                                                              │
│      };                                                                       │
│      ```                                                                      │
│                                                                              │
│  13. 筛选可用时段                                                             │
│      代码位置: apps/api/v2/src/lib/services/qualified-hosts.service.ts      │
│                                                                              │
│      流程：                                                                   │
│      ┌─────────────────────────────────────────────────────────────────────┐│
│      │ 1. 获取事件类型的所有主持人（hosts）                                    ││
│      │ 2. 如果有 routedTeamMemberIds：                                        ││
│      │    - 过滤出 ID 在 routedTeamMemberIds 中的主持人                       ││
│      │    - 这些是"合格的轮询主持人"（qualifiedRRHosts）                     ││
│      │ 3. 计算可用时段时，只考虑这些合格的主持人                                ││
│      │ 4. 如果 routedTeamMemberIds 为空或没有匹配：                           ││
│      │    - 使用所有主持人作为回退（allFallbackRRHosts）                      ││
│      └─────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  14. 访客选择时段并完成预订                                                    │
│      - 只看到 routedTeamMemberIds 中成员的可用时段                            │
│      - 预订时，AssignmentReason 记录为 ROUTING_FORM_ROUTING                  │
│      - 路由追踪记录到 RoutingTrace 表                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 关键代码路径

#### 4.2.1 Headless Router 入口

根据文档，Headless Router 的访问路径为：
```
GET /router?formId={FORM_ID}&field1=value1&field2=value2
```

这种方式支持通过 URL 参数直接传递表单答案，适合外部系统集成。

#### 4.2.2 查询参数解析

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

#### 4.2.3 合格主持人筛选

**文件**: `apps/api/v2/src/lib/services/qualified-hosts.service.ts:55-70`

```typescript
let qualifiedRRHosts = allRRHosts;

if (contactOwnerEmail) {
  // 如果有联系负责人邮箱，优先匹配
  const contactOwnerHost = allRRHosts.filter(
    (h) => (h.user as { email?: string }).email === contactOwnerEmail
  );
  if (contactOwnerHost.length > 0) {
    qualifiedRRHosts = contactOwnerHost;
  }
} else if (routedTeamMemberIds.length > 0) {
  // 使用路由表单传递的团队成员ID筛选
  const routedMemberIdSet = new Set(routedTeamMemberIds);
  const routedHosts = allRRHosts.filter((h) => 
    routedMemberIdSet.has((h.user as { id: number }).id)
  );
  if (routedHosts.length > 0) {
    qualifiedRRHosts = routedHosts;
  }
}
```

#### 4.2.4 预约页面使用

**文件**: `apps/web/modules/schedules/hooks/useSchedule.ts:77-79`

```typescript
const routedTeamMemberIds = searchParams
  ? getRoutedTeamMemberIdsFromSearchParams(
      new URLSearchParams(searchParams.toString())
    )
  : null;
```

然后传递给 `getSchedule` 查询：
```typescript
const input = {
  // ... 其他参数
  routedTeamMemberIds,
  // ...
};
```

### 4.3 数据库表关系

```
┌─────────────────────────────┐
│   App_RoutingForms_Form     │
├─────────────────────────────┤
│ id (PK)                     │
│ name                        │
│ fields (JSONB)  ◄───────────┼────── 表单字段定义
│ routes (JSONB)  ◄───────────┼────── 路由规则定义
│ settings (JSONB)            │
│ userId (FK)                 │
│ teamId (FK)                 │
│ disabled                    │
└─────────────────────────────┘
              │
              │ 1:N
              ▼
┌─────────────────────────────┐
│ App_RoutingForms_FormResponse│
├─────────────────────────────┤
│ id (PK)                     │
│ formId (FK) ────────────────┼──► 关联的表单
│ formFillerId                │
│ response (JSONB) ◄──────────┼── 访客答案
│ routedToBookingUid (FK)     │
└─────────────────────────────┘
              │
              │ 1:1
              ▼
┌─────────────────────────────┐
│      RoutingTrace           │
├─────────────────────────────┤
│ id (PK)                     │
│ formResponseId (FK)         │
│ queuedFormResponseId (FK)   │
│ bookingUid (FK)             │
│ assignmentReasonId (FK)     │
│ trace (JSONB)  ◄────────────┼── 路由决策详情
└─────────────────────────────┘
```

---

## 五、核心数据结构详解

### 5.1 表单字段定义（fields JSONB）

```json
[
  {
    "id": "field_1",
    "type": "text",
    "label": "您的姓名",
    "required": true,
    "placeholder": "请输入姓名"
  },
  {
    "id": "field_2",
    "type": "select",
    "label": "咨询类型",
    "required": true,
    "options": [
      { "value": "sales", "label": "销售咨询" },
      { "value": "support", "label": "技术支持" },
      { "value": "other", "label": "其他" }
    ]
  },
  {
    "id": "field_3",
    "type": "multiselect",
    "label": "感兴趣的产品",
    "required": false,
    "options": [
      { "value": "product_a", "label": "产品 A" },
      { "value": "product_b", "label": "产品 B" }
    ],
    "showConditions": [
      { "fieldId": "field_2", "operator": "equals", "value": "sales" }
    ]
  }
]
```

### 5.2 路由规则定义（routes JSONB）

```json
[
  {
    "id": "route_1",
    "name": "销售咨询路由",
    "priority": 1,
    "condition": {
      "type": "field_match",
      "fieldId": "field_2",
      "operator": "equals",
      "value": "sales"
    },
    "action": {
      "type": "eventTypeRedirect",
      "eventTypeId": 123,
      "attributeRouting": {
        "conditions": [
          {
            "attribute": "department",
            "operator": "equals",
            "value": "Sales"
          }
        ]
      }
    }
  },
  {
    "id": "route_2",
    "name": "技术支持路由",
    "priority": 2,
    "condition": {
      "type": "field_match",
      "fieldId": "field_2",
      "operator": "equals",
      "value": "support"
    },
    "action": {
      "type": "eventTypeRedirect",
      "eventTypeId": 456,
      "attributeRouting": {
        "conditions": [
          {
            "attribute": "department",
            "operator": "equals",
            "value": "Support"
          }
        ]
      }
    }
  },
  {
    "id": "route_default",
    "name": "默认路由",
    "priority": 100,
    "condition": {
      "type": "always_true"
    },
    "action": {
      "type": "eventTypeRedirect",
      "eventTypeId": 789
    }
  }
]
```

### 5.3 路由追踪记录（trace JSONB）

```json
{
  "formId": "form_abc123",
  "formResponseId": 456,
  "visitorAnswers": {
    "field_1": "张三",
    "field_2": "sales",
    "field_3": ["product_a"]
  },
  "routingDecision": {
    "matchedRouteId": "route_1",
    "matchedRouteName": "销售咨询路由",
    "actionType": "eventTypeRedirect",
    "eventTypeId": 123,
    "routedTeamMemberIds": [10, 15, 22],
    "attributeRoutingMatches": [
      { "userId": 10, "attributes": { "department": "Sales" } },
      { "userId": 15, "attributes": { "department": "Sales" } },
      { "userId": 22, "attributes": { "department": "Sales" } }
    ]
  },
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## 六、功能现状与遗留代码

### 6.1 数据库迁移历史

根据 `packages/prisma/migrations/20260305043434_remove_routing_forms/migration.sql`，路由表单相关表已于 2026-03-05 被移除：

**被移除的表**：
- `App_RoutingForms_Form`
- `App_RoutingForms_FormResponse`
- `App_RoutingForms_QueuedFormResponse`
- `App_RoutingForms_IncompleteBookingActions`
- `RoutingTrace`
- `PendingRoutingTrace`
- `RoutingFormResponseDenormalized`
- `RoutingFormResponseField`
- `WorkflowsOnRoutingForms`

**被移除的枚举值**：
- `AssignmentReasonEnum`: `ROUTING_FORM_ROUTING`, `ROUTING_FORM_ROUTING_FALLBACK`
- `WebhookTriggerEvents`: `ROUTING_FORM_FALLBACK_HIT`
- `WorkflowType`: `ROUTING_FORM`

### 6.2 保留的代码

虽然数据库表被移除，但以下相关代码仍然保留：

**查询参数处理**：
- `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts`
- 支持 `cal.routedTeamMemberIds`, `cal.skipContactOwner`, `cal.queuedFormResponseId` 等参数

**主持人筛选逻辑**：
- `apps/api/v2/src/lib/services/qualified-hosts.service.ts`
- 仍然支持 `routedTeamMemberIds` 参数用于筛选主持人

**预约页面集成**：
- `apps/web/modules/schedules/hooks/useSchedule.ts`
- 仍然从 URL 读取并传递 `routedTeamMemberIds`

### 6.3 可能的用途

保留这些代码的可能原因：

1. **外部系统集成**：支持通过 API 直接传递 `routedTeamMemberIds` 来控制预约路由
2. **Salesforce 等 CRM 集成**：从 CRM 传递负责人信息
3. **向后兼容**：支持已有的嵌入代码和 API 调用
4. **Headless 模式**：支持 `headless-routing-to-booking-flow.md` 中描述的无头路由模式

---

## 七、关键文件索引

| 文件路径 | 功能说明 |
|----------|----------|
| `headless-routing-to-booking-flow.md` | 无头路由到预订流程文档 |
| `packages/lib/bookings/getRoutedTeamMemberIdsFromSearchParams.ts` | 从 URL 解析路由团队成员 ID |
| `apps/api/v2/src/lib/services/qualified-hosts.service.ts` | 合格主持人筛选服务（核心路由逻辑） |
| `apps/web/modules/schedules/hooks/useSchedule.ts` | 预约页面使用路由参数 |
| `packages/trpc/server/routers/viewer/slots/util.ts` | 时段查询中的路由参数处理 |
| `packages/trpc/server/routers/viewer/slots/types.ts` | 路由参数类型定义 |

---

## 八、总结

### 8.1 核心机制

Cal.diy 的路由表单功能通过以下核心机制实现访客答案到预约入口的映射：

1. **规则定义**：在 `routes` JSONB 字段中定义条件-动作对
2. **路由引擎**：按优先级匹配规则，短路执行第一个匹配的动作
3. **属性路由**：对于团队事件，支持基于成员属性的二次筛选
4. **参数传递**：通过 `cal.routedTeamMemberIds` URL 参数将路由结果传递到预约页面
5. **主持人筛选**：在预约页面使用 `QualifiedHostsService` 只显示路由目标成员的可用时段

### 8.2 完整链路

```
访客填写表单 → 提交答案 → 路由引擎匹配规则 → 计算 routedTeamMemberIds
                                                         ↓
                                              重定向到预约页面
                                                         ↓
                                         解析 cal.routedTeamMemberIds
                                                         ↓
                                         QualifiedHostsService 筛选主持人
                                                         ↓
                                              只显示匹配成员的时段
                                                         ↓
                                              访客选择时段完成预订
```

### 8.3 当前状态

- **数据库层**：路由表单核心表已于 2026-03-05 移除
- **代码层**：`routedTeamMemberIds` 相关的处理逻辑仍然保留
- **用途**：支持外部集成、API 调用和无头路由模式

### 8.4 扩展点

如果需要重新启用或增强路由表单功能，可以考虑：

1. **恢复数据库表**：基于迁移文件恢复 `App_RoutingForms_Form` 等表
2. **重构路由引擎**：使用现代的规则引擎库
3. **可视化规则编辑器**：提供拖拽式的路由规则配置界面
4. **增强属性路由**：支持更复杂的成员匹配逻辑
5. **A/B 测试路由**：支持分流测试不同路由策略的效果
