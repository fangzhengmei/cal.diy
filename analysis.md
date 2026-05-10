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

## 9. 端到端调用链分析

### 9.1 预约功能端到端调用链

#### 9.1.1 预约创建完整调用链

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           预约创建端到端调用链                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  前端消费层                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ apps/web/modules/bookings/hooks/useBookings.ts:1-100                   │   │
│  │   ├── import type { BookerEvent, BookingResponse } from               │   │
│  │   │         "@calcom/features/bookings/types"                         │   │
│  │   ├── import { BookingStatus } from "@calcom/prisma/enums"            │   │
│  │   ├── 消费 BookerEvent 类型（Pick<PublicEvent, ...>）                 │   │
│  │   └── 消费 BookingResponse 类型                                       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  业务逻辑层                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/features/bookings/types.ts:1-102                              │   │
│  │   ├── export type { BookingCreateBody };                               │   │
│  │   ├── export type { BookingResponse };                                 │   │
│  │   └── import type { SchedulingType } from "@calcom/prisma/enums"       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  packages/features/bookings/lib/bookingCreateBodySchema.ts:1-100               │   │
│  │   ├── import { CreationSource } from "@calcom/prisma/enums"               │   │
│  │   ├── export const bookingCreateBodySchema = z.object({...})               │   │
│  │   └── 定义 Zod 验证 schema                                                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  packages/features/bookings/lib/handleNewBooking/createBooking.ts:1-200        │   │
│  │   ├── import prisma from "@calcom/prisma"                                   │   │
│  │   ├── import type { CreationSource } from "@calcom/prisma/enums"           │   │
│  │   ├── import { BookingStatus } from "@calcom/prisma/enums"                 │   │
│  │   ├── type CreateBookingParams = {...}                                      │   │
│  │   ├── const newBookingData: Prisma.BookingCreateInput = {...}               │   │
│  │   └── return prisma.$transaction(async (tx) => {                            │   │
│  │           const booking = await tx.booking.create(createBookingObj);        │   │
│  │       });                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  Prisma 类型层                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/prisma/schema.prisma:851-900 (Booking 模型)                     │   │
│  │   ├── model Booking {                                                   │   │
│  │   │   uid                    String                                     │   │
│  │   │   userPrimaryEmail       String?                                    │   │
│  │   │   eventTypeId            Int?                                       │   │
│  │   │   title                  String                                     │   │
│  │   │   startTime              DateTime                                   │   │
│  │   │   endTime                DateTime                                   │   │
│  │   │   status                 BookingStatus                              │   │
│  │   │   attendees              Attendee[]                                 │   │
│  │   │   location               String?                                    │   │
│  │   │   metadata               Json?                                      │   │
│  │   │   ...                                                                │   │
│  │   └── }                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  packages/prisma/generated/prisma/client/index.d.ts (自动生成)                  │   │
│  │   ├── export type BookingCreateInput = {...}                               │   │
│  │   ├── export type Booking = {...}                                          │   │
│  │   └── export const Prisma: {...}                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  数据库层                                                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/prisma/migrations/20210605225507_added_bookings/                │   │
│  │   ├── CREATE TABLE "Booking" (...)                                       │   │
│  │   └── CREATE TABLE "Attendee" (...)                                      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 9.1.2 API v2 预约创建调用链

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        API v2 预约创建调用链                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  API 输入层                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/platform/types/bookings/2024-08-13/inputs/create-booking.input.ts│  │
│  │   ├── export class CreateBookingInput_2024_08_13 {...}                  │   │
│  │   ├── start!: string                                                    │   │
│  │   ├── attendee!: CreateBookingAttendee                                  │   │
│  │   ├── eventTypeId?: number                                              │   │
│  │   ├── eventTypeSlug?: string                                            │   │
│  │   ├── lengthInMinutes?: number                                          │   │
│  │   └── 使用 class-validator 装饰器                                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  API 服务层                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ apps/api/v2/src/platform/bookings/2024-08-13/services/                   │   │
│  │   ├── 接收平台类型输入                                                   │   │
│  │   ├── 转换为内部服务类型                                                 │   │
│  │   └── 调用 BookingAttendeesService 等内部服务                            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  桥接层                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/platform/libraries/index.ts                                    │   │
│  │   ├── export { getBookingForReschedule } from                           │   │
│  │   │   "@calcom/features/bookings/lib/get-booking"                       │   │
│  │   ├── export type { BookingCreateBody, BookingResponse } from           │   │
│  │   │   "@calcom/features/bookings/types"                                 │   │
│  │   └── export { CreationSource, BookingStatus } from                     │   │
│  │       "@calcom/prisma/enums"                                            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  内部业务层（同 9.1.1）                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/features/bookings/...                                          │   │
│  │   ├── bookingCreateBodySchema.ts                                        │   │
│  │   ├── handleNewBooking/createBooking.ts                                 │   │
│  │   └── 使用 Prisma 类型                                                   │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  API 输出层                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts   │   │
│  │   ├── export class BookingAttendee {...}                                │   │
│  │   │   ├── name!: string                                                 │   │
│  │   │   ├── email!: string                                                │   │
│  │   │   ├── timeZone!: string                                             │   │
│  │   │   ├── language?: BookingLanguageType                                │   │
│  │   │   ├── absent!: boolean                                              │   │
│  │   │   └── phoneNumber?: string                                          │   │
│  │   ├── export class BookingOutput_2024_08_13 extends                     │   │
│  │   │   BaseBookingOutput_2024_08_13                                      │   │
│  │   └── 使用 class-transformer @Expose() 装饰器                            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 事件类型端到端调用链

#### 9.2.1 事件类型创建/更新调用链

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      事件类型端到端调用链                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  业务类型定义层                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/features/eventtypes/lib/types.ts:1-494                        │   │
│  │   ├── export type FormValues = {...}                                   │   │
│  │   │   ├── id: number                                                   │   │
│  │   │   ├── title: string                                                │   │
│  │   │   ├── slug: string                                                 │   │
│  │   │   ├── length: number                                               │   │
│  │   │   ├── schedulingType: SchedulingType | null                        │   │
│  │   │   ├── periodType: PeriodType                                       │   │
│  │   │   ├── hosts: Host[]                                                │   │
│  │   │   └── ...                                                          │   │
│  │   ├── export type EventTypeUpdateInput = {...}                         │   │
│  │   │   ├── id: number                                                   │   │
│  │   │   ├── title?: string                                               │   │
│  │   │   ├── slug?: string                                                │   │
│  │   │   ├── length?: number                                              │   │
│  │   │   ├── schedulingType?: SchedulingType | null                       │   │
│  │   │   └── ...                                                          │   │
│  │   └── import type {                                                    │   │
│  │           CancellationReasonRequirement,                               │   │
│  │           MembershipRole, PeriodType, SchedulingType                   │   │
│  │       } from "@calcom/prisma/enums"                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  服务层                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/features/eventtypes/service/EventTypeService.ts               │   │
│  │   ├── 使用 EventTypeUpdateInput 类型                                    │   │
│  │   └── 调用 Prisma 进行数据操作                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  仓库层                                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/features/eventtypes/repositories/eventTypeRepository.ts       │   │
│  │   ├── 使用 Prisma 类型                                                  │   │
│  │   └── prisma.eventType.create/update/select                             │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  Prisma 类型层                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/prisma/schema.prisma:156-306 (EventType 模型)                  │   │
│  │   ├── model EventType {                                                 │   │
│  │   │   id                Int                                             │   │
│  │   │   title             String                                          │   │
│  │   │   slug              String                                          │   │
│  │   │   description       String?                                         │   │
│  │   │   length            Int                                             │   │
│  │   │   schedulingType    SchedulingType?                                 │   │
│  │   │   periodType        PeriodType                                      │   │
│  │   │   hosts             Host[]                                          │   │
│  │   │   metadata          Json?                                           │   │
│  │   │   locations         Json?                                           │   │
│  │   │   bookingFields     Json?                                           │   │
│  │   │   recurringEvent    Json?                                           │   │
│  │   │   seatsPerTimeSlot  Int?                                            │   │
│  │   │   ...                                                               │   │
│  │   └── }                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  Host 关联模型                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ packages/prisma/schema.prisma:61-85 (Host 模型)                         │   │
│  │   ├── model Host {                                                      │   │
│  │   │   userId           Int                                              │   │
│  │   │   eventTypeId      Int                                              │   │
│  │   │   isFixed          Boolean                                          │   │
│  │   │   priority         Int?                                             │   │
│  │   │   weight           Int?                                             │   │
│  │   │   scheduleId       Int?                                             │   │
│  │   │   location         HostLocation?                                    │   │
│  │   └── }                                                                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 10. 字段映射关系表

### 10.1 Booking 字段完整映射

| schema.prisma 字段 | Prisma 类型 | Zod schema | Features 类型 | Platform Types (API v2) | 备注 |
|-------------------|-------------|-----------|--------------|------------------------|------|
| `id Int` | `number` | `z.number()` | `number` | `id!: number` | 主键，自增 |
| `uid String @unique` | `string` | `z.string()` | `string` | `uid!: string` | 业务唯一标识 |
| `userId Int?` | `number \| null` | `z.number().nullable()` | `number \| null` | `hosts[].id!: number` | 组织者关联 |
| `eventTypeId Int?` | `number \| null` | `z.number().nullable()` | `number \| null` | `eventTypeId!: number` | 事件类型关联 |
| `userPrimaryEmail String?` | `string \| null` | `emailSchema` | `string \| null` | 无直接映射 | 组织者邮箱快照 |
| `title String` | `string` | `z.string()` | `string` | `title!: string` | 预约标题 |
| `description String?` | `string \| null` | `z.string().nullable()` | `string \| null` | `description!: string` | 预约描述 |
| `startTime DateTime` | `Date` | `coerceToDate` | `string (ISO)` | `start!: string (ISO)` | 开始时间 |
| `endTime DateTime` | `Date` | `coerceToDate` | `string (ISO)` | `end!: string (ISO)` | 结束时间 |
| `status BookingStatus` | `enum` | `z.nativeEnum(BookingStatus)` | `BookingStatus` | `status!: "cancelled" \| "accepted" \| "rejected" \| "pending"` | 预约状态 |
| `location String?` | `string \| null` | `eventTypeLocations` | `string \| null` | `location!: string` | 会议位置 |
| `metadata Json?` | `Prisma.JsonValue` | `bookingMetadataSchema` | `Record<string, string>` | `metadata?: Record<string, string>` | 扩展数据 |
| `smsReminderNumber String?` | `string \| null` | 无自动生成 | `string \| null` | 无直接映射 | SMS 提醒号码 |
| `attendees Attendee[]` | `Attendee[]` | `attendeeSchema[]` | `Attendee[]` | `attendees!: BookingAttendee[]` | 参会者列表 |
| `cancellationReason String?` | `string \| null` | 无自动生成 | `string \| null` | `cancellationReason?: string` | 取消原因 |
| `rejectionReason String?` | `string \| null` | 无自动生成 | `string \| null` | 无直接映射 | 拒绝原因 |
| `createdAt DateTime` | `Date` | 无自动生成 | `Date` | `createdAt!: string (ISO)` | 创建时间 |
| `updatedAt DateTime?` | `Date \| null` | 无自动生成 | `Date \| null` | `updatedAt!: string \| null` | 更新时间 |
| `noShowHost Boolean?` | `boolean \| null` | 无自动生成 | `boolean \| null` | `absentHost!: boolean` | 主持人缺席标记 |
| `rating Int?` | `number \| null` | 无自动生成 | `number \| null` | `rating?: number` | 评分 |
| `ratingFeedback String?` | `string \| null` | 无自动生成 | `string \| null` | 无直接映射 | 评分反馈 |
| `paid Boolean` | `boolean` | `z.boolean()` | `boolean` | 无直接映射 | 是否已支付 |
| `responses Json?` | `Prisma.JsonValue` | `bookingResponses` | `Record<string, unknown>` | `bookingFieldsResponses!: Record<string, unknown>` | 自定义字段响应 |
| `fromReschedule String?` | `string \| null` | 无自动生成 | `string \| null` | `rescheduledFromUid?: string` | 来源预约 UID |

### 10.2 Attendee 字段完整映射

| schema.prisma 字段 | Prisma 类型 | Zod schema | Features 类型 | Platform Types (API v2) | 备注 |
|-------------------|-------------|-----------|--------------|------------------------|------|
| `id Int` | `number` | `z.number()` | `number` | `id?: number` | 主键 |
| `email String` | `string` | `emailSchema` | `string` | `email!: string` | 邮箱 |
| `name String` | `string` | `z.string()` | `string` | `name!: string` | 姓名 |
| `timeZone String` | `string` | `timeZoneSchema` | `string` | `timeZone!: string` | 时区 |
| `phoneNumber String?` | `string \| null` | 无自动生成 | `string \| null` | `phoneNumber?: string` | 电话号码 |
| `locale String?` | `string \| null` | 无自动生成 | `string \| null` | `language?: BookingLanguageType` | 语言偏好 |
| `bookingId Int?` | `number \| null` | 无自动生成 | `number \| null` | `bookingId?: number` | 预约关联 |
| `noShow Boolean?` | `boolean \| null` | 无自动生成 | `boolean \| null` | `absent!: boolean` | 缺席标记 |

### 10.3 EventType 字段完整映射

| schema.prisma 字段 | Prisma 类型 | Zod schema | Features 类型 | Platform Types (API v2) | 备注 |
|-------------------|-------------|-----------|--------------|------------------------|------|
| `id Int` | `number` | `z.number()` | `number` | `eventType.id!: number` | 主键 |
| `title String` | `string` | `z.string().min(1)` | `string` | 无直接映射 | 标题 |
| `slug String` | `string` | `eventTypeSlug` | `string` | `eventType.slug!: string` | URL 标识 |
| `description String?` | `string \| null` | `z.string().nullable()` | `string \| null` | 无直接映射 | 描述 |
| `length Int` | `number` | `z.number().min(1)` | `number` | `duration!: number` | 时长（分钟） |
| `schedulingType SchedulingType?` | `enum` | `z.nativeEnum(SchedulingType)` | `SchedulingType \| null` | 无直接映射 | 调度类型 |
| `periodType PeriodType` | `enum` | `z.nativeEnum(PeriodType)` | `PeriodType` | 无直接映射 | 周期类型 |
| `locations Json?` | `Prisma.JsonValue` | `eventTypeLocations` | `EventLocation[]` | 无直接映射 | 位置配置 |
| `metadata Json?` | `Prisma.JsonValue` | `EventTypeMetaDataSchema` | `EventTypeMetadata` | 无直接映射 | 扩展元数据 |
| `bookingFields Json?` | `Prisma.JsonValue` | `eventTypeBookingFields` | `eventTypeBookingFields` | 无直接映射 | 预订字段配置 |
| `recurringEvent Json?` | `Prisma.JsonValue` | `recurringEventType` | `RecurringEvent \| null` | 无直接映射 | 递归事件配置 |
| `seatsPerTimeSlot Int?` | `number \| null` | `z.number().nullable()` | `number \| null` | 无直接映射 | 座位数 |
| `requiresConfirmation Boolean` | `boolean` | `z.boolean()` | `boolean` | 无直接映射 | 是否需要确认 |
| `minimumBookingNotice Int` | `number` | `z.number().min(0)` | `number` | 无直接映射 | 最小预订提前量 |
| `beforeEventBuffer Int` | `number` | `z.number()` | `number` | 无直接映射 | 会前缓冲 |
| `afterEventBuffer Int` | `number` | `z.number()` | `number` | 无直接映射 | 会后缓冲 |
| `hosts Host[]` | `Host[]` | `hostSchema[]` | `Host[]` | `hosts!: BookingHost[]` | 主持人列表 |

### 10.4 Host 字段映射

| schema.prisma 字段 | Prisma 类型 | Zod schema | Features 类型 | Platform Types (API v2) | 备注 |
|-------------------|-------------|-----------|--------------|------------------------|------|
| `userId Int` | `number` | `z.number()` | `number` | `hosts[].id!: number` | 用户关联 |
| `eventTypeId Int` | `number` | `z.number()` | `number` | 无直接映射 | 事件类型关联 |
| `isFixed Boolean` | `boolean` | `z.boolean()` | `boolean` | 无直接映射 | 是否固定主持人 |
| `priority Int?` | `number \| null` | `z.number().nullable()` | `number \| null` | 无直接映射 | 优先级 |
| `weight Int?` | `number \| null` | `z.number().nullable()` | `number \| null` | 无直接映射 | 权重 |
| `scheduleId Int?` | `number \| null` | `z.number().nullable()` | `number \| null` | 无直接映射 | 日程关联 |
| `location HostLocation?` | `HostLocation \| null` | `hostLocationSchema` | `HostLocation \| null` | 无直接映射 | 位置配置 |

## 11. 真实字段变更案例分析

### 11.1 案例：Attendee 表添加 phoneNumber 字段

#### 11.1.1 变更背景

**变更时间**：2024-04-08

**迁移脚本**：`packages/prisma/migrations/20240408155446_add_phone_number_in_attendee/migration.sql`

**变更内容**：
```sql
-- AlterTable
ALTER TABLE "Attendee" ADD COLUMN     "phoneNumber" TEXT;
```

#### 11.1.2 Schema 变更

**修改前**（schema.prisma:826-841）：
```prisma
model Attendee {
  id          Int          @id @default(autoincrement())
  email       String
  name        String
  timeZone    String
  locale      String?      @default("en")
  booking     Booking?     @relation(fields: [bookingId], references: [id], onDelete: Cascade)
  bookingId   Int?
  bookingSeat BookingSeat?
  noShow      Boolean?     @default(false)
}
```

**修改后**：
```prisma
model Attendee {
  id          Int          @id @default(autoincrement())
  email       String
  name        String
  timeZone    String
  phoneNumber String?      // 新增字段
  locale      String?      @default("en")
  booking     Booking?     @relation(fields: [bookingId], references: [id], onDelete: Cascade)
  bookingId   Int?
  bookingSeat BookingSeat?
  noShow      Boolean?     @default(false)
}
```

#### 11.1.3 受影响模块分析

##### 模块 1：Prisma 生成类型
- **文件**：`packages/prisma/generated/prisma/client/index.d.ts`（自动生成）
- **影响**：
  - `Attendee` 类型新增 `phoneNumber?: string | null`
  - `AttendeeCreateInput` 类型新增 `phoneNumber?: string`
  - `AttendeeUpdateInput` 类型新增 `phoneNumber?: string | Prisma.StringFieldUpdateOperationsInput`

##### 模块 2：业务逻辑层 - BookingAttendeesService
- **文件**：`packages/features/bookings/services/BookingAttendeesService.ts`

**类型定义变化**（第 32-40 行）：
```typescript
export type CreatedAttendee = {
  id: number;
  bookingId: number;
  email: string;
  name: string;
  timeZone: string;
  locale: string | null;
  phoneNumber: string | null;  // 新增字段
};
```

**数据映射变化**（第 105-111 行）：
```typescript
const newAttendeeDetails = validatedAttendees.map((a) => ({
  name: a.name || "",
  email: a.email,
  timeZone: a.timeZone || organizer.timeZone,
  locale: a.language || organizer.locale,
  phoneNumber: a.phoneNumber || null,  // 新增映射
}));
```

**返回值变化**（第 140-148 行）：
```typescript
return {
  id: createdAttendee.id,
  bookingId,
  email: createdAttendee.email,
  name: createdAttendee.name,
  timeZone: createdAttendee.timeZone,
  locale: createdAttendee.locale,
  phoneNumber: createdAttendee.phoneNumber,  // 新增返回
};
```

##### 模块 3：CalendarEventBuilder
- **文件**：`packages/features/CalendarEventBuilder.ts`

**类型定义变化**（第 51-57 行）：
```typescript
async function _buildPersonFromAttendee(
  attendee: Pick<Attendee, "locale" | "name" | "timeZone" | "email" | "phoneNumber"> & {  // 新增 phoneNumber
    bookingSeat: Pick<
      BookingSeat,
      "id" | "referenceUid" | "bookingId" | "metadata" | "data" | "attendeeId"
    > | null;
  }
) {
  return {
    name: attendee.name ?? "",
    email: attendee.email,
    timeZone: attendee.timeZone,
    language: { translate, locale: attendee.locale ?? "en" },
    phoneNumber: attendee.phoneNumber,  // 新增映射
    bookingSeat: attendee.bookingSeat,
  } satisfies Person;
}
```

##### 模块 4：API v2 服务层
- **文件**：`apps/api/v2/src/platform/bookings/2024-08-13/services/booking-attendees.service.ts`

**查询返回变化**（第 21-39 行）：
```typescript
async getBookingAttendees(bookingUid: string): Promise<BookingAttendeeWithId_2024_08_13[]> {
  const attendees = await this.bookingAttendeesService.getBookingAttendees(bookingUid);

  return attendees.map((attendee) =>
    plainToClass(
      BookingAttendeeWithId_2024_08_13,
      {
        id: attendee.id,
        name: attendee.name,
        email: attendee.email,
        displayEmail: this.getDisplayEmail(attendee.email),
        timeZone: attendee.timeZone,
        language: attendee.locale ?? undefined,
        absent: attendee.noShow ?? false,
        phoneNumber: attendee.phoneNumber ?? undefined,  // 新增字段映射
      },
      { strategy: "excludeAll" }
    )
  );
}
```

**添加参会者变化**（第 87-103 行）：
```typescript
const createdAttendee = await this.bookingAttendeesService.addAttendee({
  bookingId: booking.id,
  attendee: {
    email: input.email,
    name: input.name,
    timeZone: input.timeZone,
    phoneNumber: input.phoneNumber,  // 新增字段传递
    language: input.language,
  },
  user: {
    id: user.id,
    email: user.email,
    organizationId: user.organizationId,
    uuid: user.uuid,
  },
  emailsEnabled,
});
```

##### 模块 5：平台类型层
- **文件**：`packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts`

**BookingAttendee 类变更**（第 21-58 行）：
```typescript
export class BookingAttendee {
  @ApiProperty({ type: String, example: "John Doe" })
  @IsString()
  @Expose()
  name!: string;

  @ApiProperty({ type: String, example: "john@example.com" })
  @IsString()
  @Expose()
  email!: string;

  @ApiProperty({ type: String, example: "john@example.com", description: "Clean email for display purposes" })
  @IsString()
  @Expose()
  displayEmail!: string;

  @ApiProperty({ type: String, example: "America/New_York" })
  @IsTimeZone()
  @Expose()
  timeZone!: string;

  @ApiPropertyOptional({ enum: BookingLanguage, example: "en" })
  @IsEnum(BookingLanguage)
  @Expose()
  @IsOptional()
  language?: BookingLanguageType;

  @ApiProperty({ type: Boolean, example: false })
  @IsBoolean()
  @Expose()
  absent!: boolean;

  @ApiPropertyOptional({ type: String, example: "+1234567890" })  // 新增
  @IsString()
  @Expose()
  @IsOptional()
  phoneNumber?: string;  // 新增字段
}
```

##### 模块 6：平台输入类型
- **文件**：`packages/platform/types/bookings/2024-08-13/inputs/create-booking.input.ts`

**BaseBookingAttendee 类变更**（第 102-139 行）：
```typescript
export class BaseBookingAttendee {
  @ApiProperty({
    type: String,
    description: "The name of the attendee.",
    example: "John Doe",
  })
  @IsString()
  name!: string;

  @ApiProperty({
    type: String,
    description: "The time zone of the attendee.",
    example: "America/New_York",
  })
  @IsTimeZone()
  timeZone!: string;

  @ApiPropertyOptional({
    type: String,
    description: "The phone number of the attendee in international format.",
    example: "+919876543210",
  })
  @IsOptional()
  @Validate((value: string) => !value || isValidPhoneNumber(value), {  // 新增验证
    message: "Invalid phone number format. Please use international format.",
  })
  phoneNumber?: string;  // 新增字段

  @ApiPropertyOptional({
    enum: BookingLanguage,
    description: "The preferred language of the attendee. Used for booking confirmation.",
    example: BookingLanguage.it,
    default: BookingLanguage.en,
  })
  @IsEnum(BookingLanguage)
  @IsOptional()
  language?: BookingLanguageType;
}
```

##### 模块 7：前端消费层
- **文件**：`apps/web/lib/booking.ts`

**参会者比较逻辑**（第 205-208 行）：
```typescript
(a.phoneNumber && a.phoneNumber === seatAttendee?.attendee?.phoneNumber)  // 新增比较条件
```

##### 模块 8：测试工具
- **文件**：`packages/testing/src/lib/bookingScenario/bookingScenario.ts`

**测试数据构造**（第 2186 行）：
```typescript
attendee: Omit<Attendee, "bookingId" | "phoneNumber" | "email" | "noShow"> & {  // 排除新字段
```

#### 11.1.4 耦合边界分析

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│               phoneNumber 字段变更的耦合边界分析                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  强耦合层（直接依赖 Prisma 类型）                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ ✅ packages/features/bookings/services/BookingAttendeesService.ts       │   │
│  │    ├── 使用 Prisma.Attendee 类型                                        │   │
│  │    ├── CreatedAttendee 类型需要包含新字段                                │   │
│  │    └── 数据映射逻辑需要更新                                              │   │
│  │                                                                         │   │
│  │ ✅ packages/features/CalendarEventBuilder.ts                            │   │
│  │    ├── Pick<Attendee, "phoneNumber"> 显式引用                           │   │
│  │    └── Person 类型需要包含新字段                                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  中耦合层（通过桥接层）                                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ ⚠️ apps/api/v2/src/platform/bookings/2024-08-13/services/              │   │
│  │    ├── 通过 BookingAttendeesService 间接使用                            │   │
│  │    ├── 需要更新转换器逻辑                                                │   │
│  │    └── 需要更新 DTO 映射                                                │   │
│  │                                                                         │   │
│  │ ⚠️ packages/platform/types/bookings/2024-08-13/                        │   │
│  │    ├── 需要更新 DTO 类定义                                              │   │
│  │    ├── 需要添加 class-validator 装饰器                                  │   │
│  │    └── 需要添加 Swagger API 文档                                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  弱耦合层（未直接依赖类型）                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ ✳️ apps/web/lib/booking.ts                                              │   │
│  │    ├── 仅在运行时比较属性值                                              │   │
│  │    ├── TypeScript 可能不会报错                                           │   │
│  │    └── 需要人工检查更新                                                  │   │
│  │                                                                         │   │
│  │ ✳️ packages/testing/src/lib/bookingScenario/bookingScenario.ts          │   │
│  │    ├── 使用 Omit 排除新字段                                             │   │
│  │    ├── 测试数据构造需要更新                                              │   │
│  │    └── 可能导致测试失败                                                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                    ↓                                                             │
│  无影响层（解耦成功）                                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │ ✅ 旧版本 API v2 (2024-04-15)                                           │   │
│  │    ├── 版本隔离，不受影响                                                │   │
│  │    └── 旧 DTO 类保持不变                                                 │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 11.1.5 变更检查清单

| 层级 | 需检查/更新的文件 | 是否需要 | 说明 |
|-----|-------------------|---------|------|
| 数据库 | `schema.prisma` | ✅ 必须 | 添加字段定义 |
| 数据库 | `migrations/*.sql` | ✅ 必须 | 生成迁移脚本 |
| 类型 | `prisma generate` | ✅ 必须 | 重新生成 Prisma 类型 |
| 业务 | `BookingAttendeesService.ts` | ✅ 必须 | 更新 CreatedAttendee 类型 |
| 业务 | `CalendarEventBuilder.ts` | ✅ 必须 | 更新 Pick 类型和返回值 |
| 平台输入 | `create-booking.input.ts` | ⚠️ 可选 | 如果 API 需要接收该字段 |
| 平台输出 | `booking.output.ts` | ⚠️ 可选 | 如果 API 需要返回该字段 |
| API v2 | `booking-attendees.service.ts` | ⚠️ 可选 | 更新转换器逻辑 |
| 前端 | `apps/web/*` | ⚠️ 可选 | 如果前端需要使用该字段 |
| 测试 | `bookingScenario.ts` | ⚠️ 可选 | 更新测试数据构造 |

#### 11.1.6 耦合度评估

| 模块 | 耦合类型 | 影响程度 | 解耦建议 |
|-----|---------|---------|---------|
| Prisma 生成类型 | 结构耦合 | 高 | 无法避免，必要耦合 |
| BookingAttendeesService | 类型耦合 | 高 | 考虑定义领域类型 |
| CalendarEventBuilder | 类型耦合 | 高 | 使用 Pick 减少影响面 |
| API v2 转换器 | 逻辑耦合 | 中 | 转换器模式已提供一定解耦 |
| 平台类型 DTO | 契约耦合 | 中 | 版本化管理降低风险 |
| 前端消费层 | 隐式耦合 | 低 | 类型安全检查可能遗漏 |
| 旧版本 API v2 | 无耦合 | 无 | 版本隔离成功 |

#### 11.1.7 变更影响范围总结

**直接影响（编译错误）**：
1. `packages/features/bookings/services/BookingAttendeesService.ts` - CreatedAttendee 类型
2. `packages/features/CalendarEventBuilder.ts` - Pick 类型和返回值

**间接影响（需要手动检查）**：
1. `packages/platform/types/bookings/2024-08-13/` - DTO 类
2. `apps/api/v2/src/platform/bookings/2024-08-13/services/` - 转换器
3. `apps/web/lib/booking.ts` - 运行时属性访问
4. `packages/testing/` - 测试数据

**无影响（版本隔离）**：
1. `packages/platform/types/bookings/2024-04-15/` - 旧版本 API

### 11.2 字段变更传播路径完整图示

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                      字段变更传播路径完整流程图                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                            第一步：Schema 变更                               │   │
│  │  packages/prisma/schema.prisma                                               │   │
│  │  ├── model Attendee { phoneNumber String? }  ← 添加字段                     │   │
│  │  └─────────────────────────────────────────────────────────────────────────┘   │
│  │                                    ↓                                            │
│  │  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │  │                           第二步：生成迁移脚本                               ││
│  │  │  yarn prisma migrate dev --name add_phone_number_in_attendee               ││
│  │  │  packages/prisma/migrations/20240408155446_add_phone_number_in_attendee/    ││
│  │  │  └── migration.sql: ALTER TABLE "Attendee" ADD COLUMN "phoneNumber" TEXT;  ││
│  │  └──────────────────────────────────────────────────────────────────────────────┘│
│  │                                    ↓                                            │
│  │  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │  │                           第三步：生成类型                                   ││
│  │  │  yarn prisma generate                                                      ││
│  │  │  自动更新文件：                                                            ││
│  │  │  ├── packages/prisma/generated/prisma/client/index.d.ts                    ││
│  │  │  │   ├── Attendee 类型：新增 phoneNumber?: string | null                   ││
│  │  │  │   ├── AttendeeCreateInput：新增 phoneNumber?: string                   ││
│  │  │  │   └── AttendeeUpdateInput：新增 phoneNumber?: ...                      ││
│  │  │  ├── packages/prisma/zod/attendeeSchema.ts (如配置了 zod 生成器)          ││
│  │  │  └── packages/kysely/types.ts (如配置了 kysely 生成器)                     ││
│  │  └──────────────────────────────────────────────────────────────────────────────┘│
│  │                                    ↓                                            │
│  │  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │  │                           第四步：类型检查发现问题                           ││
│  │  │  yarn type-check:ci --force                                                ││
│  │  │  发现编译错误的文件：                                                       ││
│  │  │  ├── packages/features/bookings/services/BookingAttendeesService.ts        ││
│  │  │  │   └── CreatedAttendee 类型定义不匹配                                     ││
│  │  │  └── packages/features/CalendarEventBuilder.ts                            ││
│  │  │      └── Pick<Attendee, ...> 缺少 phoneNumber                             ││
│  │  └──────────────────────────────────────────────────────────────────────────────┘│
│  │                                    ↓                                            │
│  │  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │  │                           第五步：修复业务层                                 ││
│  │  │  1. BookingAttendeesService.ts:                                            ││
│  │  │     ├── CreatedAttendee 类型添加 phoneNumber: string | null               ││
│  │  │     ├── newAttendeeDetails 映射添加 phoneNumber                           ││
│  │  │     └── 返回值添加 phoneNumber                                             ││
│  │  │                                                                           ││
│  │  │  2. CalendarEventBuilder.ts:                                               ││
│  │  │     ├── Pick 类型添加 "phoneNumber"                                       ││
│  │  │     └── Person 类型添加 phoneNumber                                       ││
│  │  └──────────────────────────────────────────────────────────────────────────────┘│
│  │                                    ↓                                            │
│  │  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │  │                           第六步：更新 API 层（如需要）                       ││
│  │  │  1. 平台输入类型 (create-booking.input.ts):                                ││
│  │  │     ├── BaseBookingAttendee 添加 phoneNumber 字段                          ││
│  │  │     └── 添加电话号码验证 (isValidPhoneNumber)                              ││
│  │  │                                                                           ││
│  │  │  2. 平台输出类型 (booking.output.ts):                                      ││
│  │  │     ├── BookingAttendee 类添加 phoneNumber 属性                           ││
│  │  │     ├── 添加 @ApiPropertyOptional 装饰器                                  ││
│  │  │     └── 添加 @Expose() 装饰器                                             ││
│  │  │                                                                           ││
│  │  │  3. API v2 服务层 (booking-attendees.service.ts):                         ││
│  │  │     ├── getBookingAttendees 映射添加 phoneNumber                         ││
│  │  │     ├── getBookingAttendee 映射添加 phoneNumber                          ││
│  │  │     └── addAttendee 传递 phoneNumber                                      ││
│  │  └──────────────────────────────────────────────────────────────────────────────┘│
│  │                                    ↓                                            │
│  │  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │  │                           第七步：更新前端（如需要）                         ││
│  │  │  apps/web/lib/booking.ts:                                                  ││
│  │  │  └── 参会者比较逻辑添加 phoneNumber 比较条件                                 ││
│  │  └──────────────────────────────────────────────────────────────────────────────┘│
│  │                                    ↓                                            │
│  │  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │  │                           第八步：更新测试（如需要）                         ││
│  │  │  packages/testing/src/lib/bookingScenario/bookingScenario.ts:              ││
│  │  │  └── Omit 类型排除 phoneNumber 字段                                        ││
│  │  └──────────────────────────────────────────────────────────────────────────────┘│
│  │                                    ↓                                            │
│  │  ┌──────────────────────────────────────────────────────────────────────────────┐│
│  │  │                           第九步：验证变更                                   ││
│  │  │  yarn type-check:ci --force                                                ││
│  │  │  yarn biome check --write .                                                ││
│  │  │  TZ=UTC yarn test                                                          ││
│  │  └──────────────────────────────────────────────────────────────────────────────┘│
│  │                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

## 12. 字段变更操作指南

### 12.1 标准操作流程

#### 阶段 1：Schema 设计与迁移

```bash
# 1. 修改 schema.prisma
# 添加/修改/删除字段

# 2. 生成迁移脚本
yarn prisma migrate dev --name <migration-name>

# 3. 审查生成的 SQL
# packages/prisma/migrations/<timestamp>_<name>/migration.sql

# 4. 生成类型
yarn prisma generate
```

#### 阶段 2：类型检查与修复

```bash
# 1. 运行类型检查
yarn type-check:ci --force

# 2. 分析编译错误
# - 查看受影响的文件
# - 判断是类型错误还是逻辑错误

# 3. 修复类型定义
# - 更新 Features 包中的类型
# - 更新 DTO 类型（如需要）
```

#### 阶段 3：业务逻辑更新

1. **检查使用 Prisma 类型的文件**
   ```typescript
   // 搜索模式
   import type { Attendee } from "@calcom/prisma/client";
   import { Prisma } from "@calcom/prisma/client";
   Pick<Attendee, "...">
   ```

2. **检查使用枚举的文件**
   ```typescript
   import { BookingStatus } from "@calcom/prisma/enums";
   ```

3. **检查使用 Zod schema 的文件**
   ```typescript
   import { bookingMetadataSchema } from "@calcom/prisma/zod-utils";
   ```

#### 阶段 4：API 层更新（如需要）

1. **判断是否需要更新 API**
   - 新字段是否需要通过 API 接收/返回？
   - 是否会影响 API 契约？

2. **如需要，更新平台类型**
   ```typescript
   // packages/platform/types/*/inputs/*.ts
   // packages/platform/types/*/outputs/*.ts
   ```

3. **更新转换器**
   ```typescript
   // apps/api/v2/src/platform/*/transformers/
   ```

#### 阶段 5：测试与验证

```bash
# 1. 运行类型检查
yarn type-check:ci --force

# 2. 运行 lint
yarn biome check --write .

# 3. 运行相关测试
TZ=UTC yarn test --filter="<affected-package>"

# 4. 如需要，运行 E2E 测试
```

### 12.2 不同变更类型的影响评估

| 变更类型 | 编译错误风险 | 运行时错误风险 | 建议操作 |
|---------|------------|---------------|---------|
| 新增可选字段 | 低 | 低 | 仅更新需要使用该字段的模块 |
| 新增必填字段 | 高 | 高 | 必须更新所有创建记录的地方 |
| 删除字段 | 高 | 高 | 必须全局搜索并移除所有引用 |
| 修改字段类型 | 高 | 高 | 必须更新所有使用该字段的地方 |
| 重命名字段 | 高 | 高 | 等同于删除+新增，需要额外处理 |
| 修改默认值 | 低 | 中 | 检查业务逻辑是否依赖默认值 |
| 添加索引 | 低 | 低 | 无代码变更，仅迁移 |
| 修改枚举值 | 中 | 中 | 检查所有使用该枚举的 switch 语句 |

### 12.3 风险缓解策略

1. **字段变更前**
   - 使用 `yarn type-check:ci --force` 确认基线状态
   - 备份数据库（生产环境）
   - 准备回滚迁移脚本

2. **字段变更中**
   - 小步提交，每次只变更一个字段
   - 频繁运行类型检查
   - 使用 `select` 而非 `include` 减少影响面

3. **字段变更后**
   - 运行完整测试套件
   - 检查 API 文档是否需要更新
   - 更新变更日志（CHANGELOG）
