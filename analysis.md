# Cal.diy 数据库模型、迁移脚本与类型共享分析

## 1. 架构概览

Cal.diy 采用 Yarn/Turbo monorepo 架构，其数据层设计具有以下特点：

- **数据库**: PostgreSQL + Prisma ORM
- **类型系统**: 多层类型抽象，从数据库模型到 API DTO
- **迁移策略**: Prisma 迁移系统，版本化管理

## 2. 数据库模型分析

### 2.1 核心数据模型

#### EventType（事件类型）
位置：`packages/prisma/schema.prisma:156-306`

核心字段：
- `id`: 主键，自增
- `title`: 事件标题
- `slug`: URL 友好标识
- `description`: 描述
- `length`: 会议时长（分钟）
- `locations`: JSON 类型，存储会议位置
- `hosts`: 关联 Host 模型，支持多主持人
- `teamId`: 团队关联
- `userId`: 用户关联
- `periodType`: 周期类型（UNLIMITED, ROLLING, RANGE）
- `recurringEvent`: JSON，递归事件配置
- `seatsPerTimeSlot`: 座位数量
- `schedulingType`: 调度类型（ROUND_ROBIN, COLLECTIVE, MANAGED）

#### Booking（预约）
位置：`packages/prisma/schema.prisma:851-900`

核心字段：
- `id`: 主键，自增
- `uid`: 唯一标识符
- `userId`: 用户关联
- `eventTypeId`: 事件类型关联
- `title`: 预约标题
- `startTime`: 开始时间
- `endTime`: 结束时间
- `status`: 预约状态（ACCEPTED, PENDING, CANCELLED, REJECTED, AWAITING_HOST）
- `attendees`: 关联 Attendee 模型
- `references`: 关联 BookingReference 模型
- `metadata`: JSON，扩展字段
- `recurringEventId`: 递归事件关联

#### Host（主持人）
位置：`packages/prisma/schema.prisma:61-85`

- 连接 User 和 EventType 的中间表
- 支持多主持人配置
- 包含优先级、权重等调度相关字段

#### Attendee（参会者）
位置：`packages/prisma/schema.prisma:826-841`

- 存储参会者信息
- 关联 Booking 或 BookingSeat

### 2.2 模型关系图

```
User ─┬─── EventType ── Host
      │        │
      │        └── Booking ── Attendee
      │              │
      └── Membership ── Team
```

## 3. 迁移脚本分析

### 3.1 迁移策略

位置：`packages/prisma/migrations/`

Cal.diy 使用 Prisma 的迁移系统，遵循以下策略：

1. **版本化管理**: 每个迁移有唯一的时间戳前缀
2. **增量迁移**: 每次 schema 变更生成新的迁移文件
3. **SQL 跟踪**: 迁移文件包含原始 SQL，便于审查和回滚

### 3.2 关键迁移历史

#### 初始化迁移 (20210605225044_init)
- 创建基础表：EventType, Credential, User
- EventType 初始字段：id, title, slug, description, locations, length, hidden, userId

#### 预约功能迁移 (20210605225507_added_bookings)
- 创建 Booking, BookingReference, Attendee 表
- 建立外键关联

#### 字段变更示例
- `20240205185412_add_email_field_in_booking`: 在 Booking 表添加 `userPrimaryEmail` 字段
- `20220420230104_update_booking_id_constrain`: 修改外键约束为 ON DELETE CASCADE

### 3.3 迁移流程

```
schema.prisma 修改
    ↓
prisma migrate dev
    ↓
生成新的 migration.sql 文件
    ↓
应用到数据库
```

## 4. 类型共享机制

### 4.1 类型生成架构

Cal.diy 使用多层类型生成策略：

```
schema.prisma
    │
    ├── Prisma Client Generator
    │       → packages/prisma/generated/prisma/
    │       → 提供类型安全的数据库操作
    │
    ├── Zod Generator (zod-prisma-types)
    │       → packages/prisma/zod/
    │       → 生成验证 schema
    │
    ├── Kysely Generator (prisma-kysely)
    │       → packages/kysely/types.ts
    │       → 为 Kysely 查询构建器提供类型
    │
    └── Enum Generator
            → packages/prisma/enums/
            → 导出枚举类型
```

### 4.2 类型导出机制

#### Prisma 包导出
位置：`packages/prisma/index.ts`

```typescript
export const prisma: PrismaClient;
export type { PrismaClient, PrismaTransaction };
export * from "./selects";
```

#### Platform Libraries 桥接层
位置：`packages/platform/libraries/index.ts`

这是 API v2 的类型桥接层，解决 API v2 无法直接导入 `@calcom/features` 等包的问题：

```typescript
export { getBookingForReschedule } from "@calcom/features/bookings/lib/get-booking";
export type { BookingCreateBody, BookingResponse } from "@calcom/features/bookings/types";
export { 
  AttributeType, CreationSource, MembershipRole, PeriodType, SchedulingType, TimeUnit, WebhookTriggerEvents 
} from "@calcom/prisma/enums";
export { bookingMetadataSchema, teamMetadataSchema, userMetadata } from "@calcom/prisma/zod-utils";
```

### 4.3 平台类型系统

位置：`packages/platform/types/`

为 API v2 提供版本化的 DTO 类型：

#### Bookings 类型
- `packages/platform/types/bookings/2024-04-15/` - 旧版本
- `packages/platform/types/bookings/2024-08-13/` - 新版本

#### EventTypes 类型
- `packages/platform/types/event-types/event-types_2024_06_14/`

特点：
- 使用 NestJS 的 class-validator 装饰器
- 版本化命名，支持 API 版本控制
- 包含输入（inputs）和输出（outputs）类型

### 4.4 类型使用示例

#### tRPC 路由层
位置：`packages/trpc/server/routers/viewer/bookings/get.handler.ts`

```typescript
import type { PrismaClient } from "@calcom/prisma";
import type { Booking } from "@calcom/prisma/client";
import { Prisma } from "@calcom/prisma/client";
import { BookingStatus, MembershipRole, SchedulingType } from "@calcom/prisma/enums";
import { EventTypeMetaDataSchema } from "@calcom/prisma/zod-utils";
```

#### 业务逻辑层
位置：`packages/features/bookings/lib/bookingCreateBodySchema.ts`

```typescript
import { CreationSource } from "@calcom/prisma/enums";

export const bookingCreateBodySchema = z.object({
  eventTypeId: z.number(),
  start: z.string(),
  end: z.string().optional(),
  timeZone: timeZoneSchema,
  creationSource: z.nativeEnum(CreationSource).optional(),
  // ...
});
```

#### API v2 层
位置：`apps/api/v2/src/platform/`

```typescript
import { MembershipRole } from "@calcom/platform-libraries";
import { UpdateScheduleInput_2024_04_15 } from "@calcom/platform-types";
```

## 5. 字段变更传播路径

### 5.1 完整传播流程

```
1. 修改 schema.prisma
        │
        ▼
2. 生成迁移脚本 (prisma migrate dev)
        │
        ▼
3. 生成类型 (prisma generate)
        │
        ├── Prisma Client 类型更新
        ├── Zod schema 更新
        ├── Kysely 类型更新
        └── Enums 更新
        │
        ▼
4. 类型检查器发现问题
        │
        ├── 编译错误（直接导入 Prisma 类型的文件）
        ├── 运行时验证错误（使用 Zod schema 的地方）
        └── Kysely 查询类型错误
        │
        ▼
5. 更新相关代码
        │
        ├── tRPC handlers
        ├── features 业务逻辑
        ├── platform-libraries 桥接层
        ├── platform-types DTO（如需）
        └── API v2 转换器
```

### 5.2 传播路径示例

假设在 Booking 模型添加新字段 `userPrimaryEmail`：

#### 步骤 1：修改 schema.prisma
```prisma
model Booking {
  // ...
  userPrimaryEmail String?
}
```

#### 步骤 2：生成迁移
```bash
yarn prisma migrate dev --name add_user_primary_email
```

生成文件：`packages/prisma/migrations/20240205185412_add_email_field_in_booking/migration.sql`

```sql
ALTER TABLE "Booking" ADD COLUMN "userPrimaryEmail" TEXT;
```

#### 步骤 3：生成类型
```bash
yarn prisma generate
```

更新文件：
- `packages/prisma/generated/prisma/client/index.d.ts` - PrismaClient 类型
- `packages/prisma/zod/bookingSchema.ts` - Zod 验证 schema
- `packages/kysely/types.ts` - Kysely 类型

#### 步骤 4：编译检查

类型检查器会在以下位置发现变更：

1. **直接导入 `Booking` 类型的文件**
   ```typescript
   // packages/trpc/server/routers/viewer/bookings/types.ts
   import type { Booking } from "@calcom/prisma/client";
   
   // 如果使用 Pick<Booking, '...'> 且不包含新字段，可能无影响
   // 如果解构 Booking 对象，新字段会出现
   ```

2. **使用 Prisma 查询的地方**
   ```typescript
   const booking = await prisma.booking.findUnique({
     select: { id: true, userPrimaryEmail: true } // 现在可以选择新字段
   });
   ```

3. **Zod 验证 schema**
   - 如果使用自动生成的 Zod schema，会自动包含新字段
   - 如果手动编写 schema，需要手动更新

### 5.3 影响范围分析

| 变更类型 | 影响范围 | 需要手动更新 |
|---------|---------|-------------|
| 新增可选字段 | 低 | 仅使用该字段的代码 |
| 新增必填字段 | 高 | 创建记录的所有地方 |
| 删除字段 | 高 | 使用该字段的所有地方 |
| 修改字段类型 | 高 | 使用该字段的所有地方 |
| 重命名字段 | 高 | 使用该字段的所有地方 |
| 修改枚举值 | 中 | 使用该枚举的所有地方 |

## 6. Monorepo 耦合边界分析

### 6.1 耦合层级

#### 层级 1：核心耦合（Prisma 包）
**位置**: `packages/prisma/`

**耦合特点**:
- 所有业务包都依赖 Prisma 类型
- 任何 schema 变更都会影响所有依赖方
- 这是"必要耦合"，无法避免

**依赖路径**:
```
@calcom/prisma
    │
    ├── @calcom/trpc
    ├── @calcom/features
    ├── @calcom/ui
    ├── @calcom/types
    └── @calcom/platform-libraries
```

#### 层级 2：业务逻辑耦合（Features 包）
**位置**: `packages/features/`

**耦合特点**:
- 依赖 Prisma 类型进行数据操作
- 包含业务规则和验证逻辑
- 被 tRPC 和 API v2 复用

**关键模块**:
- `@calcom/features/bookings` - 预约逻辑
- `@calcom/features/eventtypes` - 事件类型逻辑
- `@calcom/features/calendars` - 日历集成

#### 层级 3：API 层耦合
**位置**: `packages/trpc/`, `apps/api/v2/`

**耦合特点**:
- **tRPC 层**: 直接依赖 Prisma 和 features
- **API v2**: 通过 platform-libraries 间接依赖，降低耦合

### 6.2 解耦策略

#### 策略 1：Platform Libraries 桥接层
**位置**: `packages/platform/libraries/index.ts`

**目的**: 为 API v2 提供稳定的类型导出点

**优势**:
- API v2 不直接依赖 features 包
- 可以控制导出的 API 表面
- 支持 API 版本控制

**限制**:
- 需要手动维护导出列表
- 类型同步需要人工操作

#### 策略 2：版本化平台类型
**位置**: `packages/platform/types/`

**目的**: 为 API v2 提供独立于内部模型的 DTO

**优势**:
- API 契约不直接绑定数据库模型
- 支持版本化演进
- 可以在不改变数据库的情况下修改 API

**实现**:
```typescript
// packages/platform/types/bookings/2024-04-15/
export class GetBookingsInput_2024_04_15 {
  // 使用 class-validator 装饰器
}

// packages/platform/types/bookings/2024-08-13/
export class GetBookingsInput_2024_08_13 {
  // 新版本，可能有字段变更
}
```

#### 策略 3：转换器模式
**位置**: `apps/api/v2/src/platform/event-types/event-types_2024_06_14/transformers/`

**目的**: 在内部模型和 API DTO 之间进行转换

**示例**:
```typescript
// internal-to-api/recurrence.ts
import { Frequency } from "@calcom/platform-enums";
import type { Recurrence_2024_06_14 } from "@calcom/platform-types";

export function transformRecurringEvent(internal: InternalRecurrence): Recurrence_2024_06_14 {
  // 转换逻辑
}
```

### 6.3 耦合风险分析

#### 高风险区域

1. **直接导入 Prisma 类型的文件众多**
   - 位置：`packages/ui/`, `packages/types/`, `packages/trpc/`, `packages/features/`
   - 风险：schema 变更导致大规模编译错误

2. **共享 Zod 验证 schema**
   - 位置：`packages/prisma/zod-utils.ts`
   - 风险：验证逻辑变更影响所有使用方

3. **枚举类型全局使用**
   - 位置：`packages/prisma/enums/`
   - 风险：枚举值变更影响所有使用方

#### 中风险区域

1. **Platform Libraries 导出**
   - 位置：`packages/platform/libraries/index.ts`
   - 风险：忘记导出新类型或函数

2. **API v2 转换器**
   - 位置：`apps/api/v2/src/platform/*/transformers/`
   - 风险：字段变更后转换器未同步更新

#### 低风险区域

1. **版本化平台类型**
   - 位置：`packages/platform/types/`
   - 优势：版本隔离，旧版本不受影响

### 6.4 耦合边界图示

```
┌─────────────────────────────────────────────────────────┐
│                    API 边界                             │
│  ┌──────────────┐    ┌────────────────────────────┐     │
│  │   API v2     │◄───│  platform-libraries (桥接) │     │
│  │  (NestJS)    │    │  platform-types (DTO)     │     │
│  └──────────────┘    └──────────────┬─────────────┘     │
└─────────────────────────────────────┼───────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────┐
│                 业务逻辑边界                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │              @calcom/features                   │   │
│  │  bookings, eventtypes, calendars, webhooks...  │   │
│  └──────────────────────┬──────────────────────────┘   │
│                         │                              │
│  ┌─────────────────────────────────────────────────┐   │
│  │              @calcom/trpc                       │   │
│  │        routers, handlers, schemas               │   │
│  └──────────────────────┬──────────────────────────┘   │
└─────────────────────────┼──────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                  数据层边界                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │              @calcom/prisma                     │   │
│  │  schema.prisma, PrismaClient, Zod, Kysely       │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              PostgreSQL Database                │   │
│  │  migrations, tables, indexes                    │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## 7. 预约和事件类型功能的类型支撑

### 7.1 事件类型创建流程

#### 类型参与点

1. **输入验证**
   - 使用 Zod schema：`@calcom/prisma/zod-utils.ts`
   - 平台类型：`packages/platform/types/event-types/event-types_2024_06_14/inputs/`

2. **业务逻辑**
   - 位置：`packages/features/eventtypes/`
   - 使用 Prisma 类型进行数据操作

3. **数据持久化**
   - Prisma Client 类型安全查询
   - `prisma.eventType.create()`

### 7.2 预约创建流程

#### 类型参与点

1. **API 输入**
   - tRPC: `packages/trpc/server/routers/viewer/bookings/`
   - API v2: `packages/platform/types/bookings/`

2. **验证 schema**
   - 位置：`packages/features/bookings/lib/bookingCreateBodySchema.ts`
   - 使用：`z.nativeEnum(CreationSource)` 等

3. **数据操作**
   - Prisma 类型：`Booking`, `Attendee`, `BookingReference`
   - 关联查询：使用 `select` 而非 `include`

4. **输出转换**
   - tRPC: 直接返回 Prisma 类型的子集
   - API v2: 通过转换器转换为平台类型

## 8. 总结与建议

### 8.1 当前架构优势

1. **类型安全**: 从数据库到 API 全链路类型安全
2. **代码复用**: features 包被 tRPC 和 API v2 复用
3. **解耦尝试**: platform-libraries 和版本化类型提供了解耦手段
4. **迁移可控**: Prisma 迁移系统提供版本化数据库变更

### 8.2 当前架构挑战

1. **核心耦合强**: Prisma schema 变更影响整个系统
2. **手动同步多**: platform-libraries 和转换器需要人工维护
3. **类型爆炸**: 多层类型抽象增加理解成本
4. **API 版本化复杂**: 需要维护多个版本的类型和转换器

### 8.3 改进建议

1. **增加自动化测试**
   - 添加类型兼容性测试
   - 确保字段变更后所有路径都被覆盖

2. **完善文档**
   - 记录类型变更的传播路径
   - 提供字段变更的检查清单

3. **考虑进一步解耦**
   - 为核心业务概念定义独立的领域类型
   - 减少对 Prisma 生成类型的直接依赖

4. **增强版本控制**
   - 明确平台类型的版本策略
   - 建立类型变更的 deprecation 流程
