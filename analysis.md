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

Cal.diy 使用多层类型生成策略。根据 `packages/prisma/schema.prisma` 配置：

```prisma
generator client {
  provider        = "prisma-client-js"
  output          = "./generated/prisma"
  ...
}

generator kysely {
  provider = "prisma-kysely"
  output   = "../kysely/types.ts"
  ...
}

generator zod {
  provider         = "zod-prisma-types"
  output           = "./zod"
  ...
}

generator enums {
  provider = "prisma-enum-generator"
  output   = "./enums"
}
```

**重要说明**：
- 上述 `output` 配置指向的目录/文件在磁盘上**不存在**
- 它们是 `yarn prisma generate` 时动态生成的
- **请勿在可核对映射中直接引用这些路径**

**实际存在的生成器实现文件**：
- `packages/prisma/enum-generator.ts` - 枚举生成器源码

### 4.2 实际存在的类型导出文件

#### Prisma 包导出
位置：`packages/prisma/index.ts`

```typescript
import { PrismaClient, type Prisma } from "./generated/prisma/client";
// ...
export const prisma: PrismaClient = ...;
export type { PrismaClient, PrismaTransaction };
export * from "./selects";
```

位置：`packages/prisma/client/index.ts`

```typescript
export * from "../generated/prisma/client";
```

**说明**：这两个文件实际存在，但它们引用的 `./generated/prisma/client` 是动态生成的。

#### Prisma Selects 导出
位置：`packages/prisma/selects/`

**实际存在的文件**：
- `packages/prisma/selects/index.ts` - 导出入口
- `packages/prisma/selects/booking.ts` - Booking 相关 select 配置
- `packages/prisma/selects/event-types.ts` - EventType 相关 select 配置
- `packages/prisma/selects/user.ts` - User 相关 select 配置
- `packages/prisma/selects/credential.ts` - Credential 相关 select 配置
- `packages/prisma/selects/payment.ts` - Payment 相关 select 配置
- `packages/prisma/selects/app.ts` - App 相关 select 配置

#### Prisma Zod Utils 导出
位置：`packages/prisma/zod-utils.ts`

```typescript
import z, { ZodNullable, ZodObject, ZodOptional } from "zod";
import type { Prisma } from "./client";
import { EventTypeCustomInputType } from "./enums";

export const bookingMetadataSchema = ...;
export const teamMetadataSchema = ...;
export const userMetadata = ...;
export const emailRegex = ...;
export const emailRegexSchema = ...;
export enum Frequency { ... }
export enum BookerLayouts { ... }
```

#### Platform Libraries 桥接层
位置：`packages/platform/libraries/index.ts`

这是 API v2 的类型桥接层，解决 API v2 无法直接导入 `@calcom/features` 等包的问题：

```typescript
import type { Prisma } from "@calcom/prisma/client";
import { credentialForCalendarServiceSelect } from "@calcom/prisma/selects/credential";
import { paymentDataSelect } from "@calcom/prisma/selects/payment";

export {
  AttributeType,
  CreationSource,
  MembershipRole,
  PeriodType,
  SchedulingType,
  TimeUnit,
  WebhookTriggerEvents,
} from "@calcom/prisma/enums";

export type {
  BookingCreateBody,
  BookingResponse,
} from "@calcom/features/bookings/types";

export {
  bookingMetadataSchema,
  teamMetadataSchema,
  userMetadata,
} from "@calcom/prisma/zod-utils";

export { credentialForCalendarServiceSelect };
export { paymentDataSelect };
```

**说明**：
- `from "@calcom/prisma/enums"` 导入的是动态生成的枚举类型
- 但 `packages/platform/libraries/index.ts` 文件本身是实际存在的

### 4.3 平台类型系统

位置：`packages/platform/types/`

为 API v2 提供版本化的 DTO 类型：

#### Bookings 类型
- `packages/platform/types/bookings/2024-04-15/` - 旧版本
- `packages/platform/types/bookings/2024-08-13/` - 新版本

**实际存在的文件结构**：
```
packages/platform/types/bookings/
├── 2024-04-15/
│   ├── inputs/
│   │   └── index.ts
│   └── index.ts
├── 2024-08-13/
│   ├── inputs/
│   │   ├── validators/
│   │   │   └── validate-metadata.ts
│   │   ├── add-attendee.input.ts
│   │   ├── add-guests.input.ts
│   │   ├── cancel-booking-input.pipe.ts
│   │   ├── cancel-booking.input.ts
│   │   ├── create-booking-input.pipe.ts
│   │   ├── create-booking.input.ts
│   │   ├── decline-booking.input.ts
│   │   ├── get-bookings.input.ts
│   │   ├── index.ts
│   │   ├── language.ts
│   │   ├── location.input.ts
│   │   ├── mark-absent.input.ts
│   │   ├── reassign-to-user.input.ts
│   │   ├── reschedule-booking-input.pipe.ts
│   │   ├── reschedule-booking.input.ts
│   │   └── update-location.input.ts
│   ├── outputs/
│   │   ├── booking.output.ts
│   │   ├── get-booking-recordings.output.ts
│   │   ├── get-booking-transcripts.output.ts
│   │   ├── get-booking-video-sessions.output.ts
│   │   ├── get-booking.output.ts
│   │   ├── get-bookings.output.ts
│   │   └── index.ts
│   └── index.ts
└── index.ts
```

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

## 5. 可核对字段映射关系表

### 5.1 实际存在的文件列表（可核对）

**以下文件/路径在磁盘上实际存在，可以直接核对：**

| 文件/路径 | 类型 | 说明 |
|----------|------|------|
| `packages/prisma/schema.prisma` | 源文件 | Prisma Schema 定义 |
| `packages/prisma/migrations/*/migration.sql` | 源文件 | 数据库迁移脚本 |
| `packages/prisma/index.ts` | 源文件 | Prisma 包导出入口 |
| `packages/prisma/client/index.ts` | 源文件 | Prisma Client 转发入口 |
| `packages/prisma/selects/index.ts` | 源文件 | Selects 导出入口 |
| `packages/prisma/selects/booking.ts` | 源文件 | Booking 相关 select 配置 |
| `packages/prisma/selects/event-types.ts` | 源文件 | EventType 相关 select 配置 |
| `packages/prisma/selects/user.ts` | 源文件 | User 相关 select 配置 |
| `packages/prisma/selects/credential.ts` | 源文件 | Credential 相关 select 配置 |
| `packages/prisma/selects/payment.ts` | 源文件 | Payment 相关 select 配置 |
| `packages/prisma/selects/app.ts` | 源文件 | App 相关 select 配置 |
| `packages/prisma/zod-utils.ts` | 源文件 | 手动维护的 Zod schema |
| `packages/prisma/enum-generator.ts` | 源文件 | 枚举生成器实现 |
| `packages/platform/libraries/index.ts` | 源文件 | API v2 桥接层 |
| `packages/platform/types/bookings/2024-08-13/*` | 源文件 | 平台类型 DTO |
| `packages/features/bookings/types.ts` | 源文件 | 业务类型定义 |
| `packages/features/bookings/lib/bookingCreateBodySchema.ts` | 源文件 | 预约创建验证 schema |
| `packages/features/eventtypes/lib/types.ts` | 源文件 | 事件类型定义 |
| `packages/trpc/server/routers/viewer/bookings/get.handler.ts` | 源文件 | tRPC 预约查询路由 |
| `packages/features/bookings/lib/handleNewBooking/createBooking.ts` | 源文件 | 预约创建核心逻辑 |

### 5.2 动态生成的文件（不可直接核对）

**以下路径在磁盘上不存在，是 `prisma generate` 时动态生成的：**

| 动态生成路径 | 生成器 | 引用方式 | 说明 |
|-------------|--------|---------|------|
| `packages/prisma/generated/prisma/*` | `prisma-client-js` | `import { PrismaClient } from "@calcom/prisma/client"` | Prisma Client 类型和运行时 |
| `packages/prisma/zod/*` | `zod-prisma-types` | 无直接引用 | 自动生成的 Zod schema（项目不直接使用） |
| `packages/prisma/enums/*` | `prisma-enum-generator` | `import { BookingStatus } from "@calcom/prisma/enums"` | 枚举类型 |
| `packages/kysely/types.ts` | `prisma-kysely` | `import type { DB } from "@calcom/kysely"` | Kysely 类型 |

**验证方法**：
- 枚举类型：可通过 `packages/prisma/schema.prisma` 中的 `enum` 定义核对
- Prisma Client 类型：可通过实际使用的代码核对（如 `Prisma.BookingSelect`）
- 项目主要使用手动维护的 `packages/prisma/zod-utils.ts` 而非自动生成的 Zod schema

### 5.3 Booking 字段映射（可核对版本）

#### 从 Schema 到 Selects 的映射

**Schema 定义**: `packages/prisma/schema.prisma:851-900`

**Selects 定义**: `packages/prisma/selects/booking.ts:1-130`

| Schema 字段 | bookingMinimalSelect | bookingDetailsSelect | bookingWithUserAndEventDetailsSelect |
|------------|---------------------|---------------------|-------------------------------------|
| `id` | ✅ (第4行) | ❌ | ✅ (第41行) |
| `uid` | ❌ | ✅ (第17行) | ✅ (第37行) |
| `title` | ✅ (第5行) | ❌ | ✅ (第32行) |
| `description` | ✅ (第7行) | ❌ | ✅ (第33行) |
| `userPrimaryEmail` | ✅ (第6行) | ❌ | ✅ (第36行) |
| `startTime` | ✅ (第9行) | ❌ | ✅ (第34行) |
| `endTime` | ✅ (第10行) | ❌ | ✅ (第35行) |
| `attendees` | ✅ (第11行) | ❌ | ✅ (带 select, 第61-68行) |
| `metadata` | ✅ (第12行) | ❌ | ✅ (第45行) |
| `createdAt` | ✅ (第13行) | ❌ | ❌ |
| `customInputs` | ✅ (第8行) | ❌ | ❌ |
| `rescheduled` | ❌ | ✅ (第18行) | ❌ |
| `fromReschedule` | ❌ | ✅ (第19行) | ❌ |
| `iCalUID` | ❌ | ❌ | ✅ (第38行) |
| `iCalSequence` | ❌ | ❌ | ✅ (第39行) |
| `eventTypeId` | ❌ | ❌ | ✅ (第40行) |
| `userId` | ❌ | ❌ | ✅ (第42行) |
| `location` | ❌ | ❌ | ✅ (第43行) |

#### 从 Selects 到业务消费的映射

**Selects 导出**: `packages/prisma/selects/index.ts:1-5`

```typescript
export { safeAppSelect } from "./app";
export * from "./booking";
export { safeCredentialSelect } from "./credential";
export * from "./event-types";
export * from "./user";
```

**Prisma 包导出**: `packages/prisma/index.ts:112-113`

```typescript
export default prisma;
export * from "./selects";
```

### 5.4 EventType 字段映射（可核对版本）

#### Schema 定义: `packages/prisma/schema.prisma:156-306`

**Selects 定义**: `packages/prisma/selects/event-types.ts:1-128`

| Schema 字段 | baseEventTypeSelect | bookEventTypeSelect | availiblityPageEventTypeSelect |
|------------|--------------------|--------------------|-------------------------------|
| `id` | ✅ (第4行) | ✅ (第23行) | ✅ (第74行) |
| `title` | ✅ (第5行) | ✅ (第24行) | ✅ (第75行) |
| `description` | ✅ (第6行) | ✅ (第26行) | ✅ (第77行) |
| `slug` | ✅ (第10行) | ✅ (第25行) | ✅ (第99行) |
| `length` | ✅ (第7行) | ✅ (第27行) | ✅ (第78行) |
| `schedulingType` | ✅ (第8行) | ❌ | ✅ (第88行) |
| `periodType` | ❌ | ✅ (第30行) | ✅ (第82行) |
| `recurringEvent` | ✅ (第9行) | ✅ (第34行) | ✅ (第89行) |
| `metadata` | ❌ | ✅ (第40行) | ✅ (第104行) |
| `locations` | ❌ | ✅ (第28行) | ✅ (第87行) |
| `bookingFields` | ❌ | ✅ (第47行) | ❌ |
| `seatsPerTimeSlot` | ✅ (第19行) | ✅ (第46行) | ✅ (第106行) |
| `requiresConfirmation` | ✅ (第16行) | ✅ (第37行) | ✅ (第90行) |
| `minimumBookingNotice` | ❌ | ❌ | ✅ (第100行) |

## 6. 逐跳证据链：Booking.status 字段

**目的**：展示 `Booking.status` 字段从 schema 到业务消费的完整可核对路径

### 第 1 跳：Schema 定义

**文件**: `packages/prisma/schema.prisma:843-848`

```prisma
enum BookingStatus {
  ACCEPTED
  PENDING
  CANCELLED
  REJECTED
  AWAITING_HOST
}
```

**文件**: `packages/prisma/schema.prisma:851-900` (Booking 模型)

```prisma
model Booking {
  // ...
  status                 BookingStatus
  // ...
}
```

### 第 2 跳：迁移脚本

**文件**: `packages/prisma/migrations/20210904162403_add_booking_status_enum/migration.sql`

```sql
-- CreateEnum
CREATE TYPE "BookingStatus" AS ENUM ('cancelled', 'accepted', 'rejected', 'pending');

-- AlterTable
ALTER TABLE "Booking" ADD COLUMN     "status" "BookingStatus" NOT NULL DEFAULT E'accepted';
```

### 第 3 跳：枚举生成器配置与实现

**生成器配置**: `packages/prisma/schema.prisma:52-55`

```prisma
generator enums {
  provider = "prisma-enum-generator"
  output   = "./enums"
}
```

**生成器实现**: `packages/prisma/enum-generator.ts:1-37`

```typescript
generatorHandler({
  onManifest() {
    return {
      defaultOutput: "./enums/index.ts",
      prettyName: "Prisma Enum Generator",
    };
  },
  async onGenerate(options) {
    const enums = options.dmmf.datamodel.enums;
    // 生成 TypeScript enum 代码
    const output = enums.map((e) => {
      let enumString = `export const ${e.name} = {\n`;
      e.values.forEach(({ name: value }) => {
        enumString += `  ${value}: "${value}",\n`;
      });
      // ...
    });
  },
});
```

### 第 4 跳：tRPC 路由消费

**文件**: `packages/trpc/server/routers/viewer/bookings/get.handler.ts:13`

```typescript
import { BookingStatus, MembershipRole, SchedulingType } from "@calcom/prisma/enums";
```

**文件**: `packages/trpc/server/routers/viewer/bookings/get.handler.ts:118`

```typescript
const fallbackRoles: MembershipRole[] = [MembershipRole.ADMIN, MembershipRole.OWNER];
```

### 第 5 跳：业务逻辑消费

**文件**: `packages/features/bookings/lib/handleNewBooking/createBooking.ts:6-7`

```typescript
import type { CreationSource } from "@calcom/prisma/enums";
import { BookingStatus } from "@calcom/prisma/enums";
```

### 第 6 跳：平台类型桥接

**文件**: `packages/platform/libraries/index.ts:24-32`

```typescript
export {
  AttributeType,
  CreationSource,
  MembershipRole,
  PeriodType,
  SchedulingType,
  TimeUnit,
  WebhookTriggerEvents,
} from "@calcom/prisma/enums";
```

**注意**: `BookingStatus` 未通过 platform-libraries 导出，API v2 使用字符串字面量解耦

### 第 7 跳：API v2 消费（字符串字面量）

**文件**: `packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts`

```typescript
export class BaseBookingOutput_2024_08_13 {
  @ApiProperty({
    enum: ["cancelled", "accepted", "rejected", "pending"],
    description: "The status of the booking.",
    example: "accepted",
  })
  @IsString()
  @IsIn(["cancelled", "accepted", "rejected", "pending"])
  @Expose()
  status!: "cancelled" | "accepted" | "rejected" | "pending";
}
```

**说明**: API v2 使用字符串字面量类型，而非直接导入 Prisma 枚举，实现了解耦

### Booking.status 证据链总结

```
Schema 定义 (packages/prisma/schema.prisma:843-848)
    ↓
迁移脚本 (packages/prisma/migrations/20210904162403_add_booking_status_enum/migration.sql)
    ↓
枚举生成器 (packages/prisma/enum-generator.ts)
    ↓
tRPC 路由 (packages/trpc/server/routers/viewer/bookings/get.handler.ts:13)
    ↓
业务逻辑 (packages/features/bookings/lib/handleNewBooking/createBooking.ts:7)
    ↓
平台桥接 (packages/platform/libraries/index.ts:24-32) [BookingStatus 未导出]
    ↓
API v2 (packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts) [字符串字面量]
```

## 6A. BookingStatus 演进链路：从数据库到消费端的完整映射

### 6A.1 演进阶段概览

| 阶段 | 时间戳 | 事件 | 数据库值 | Schema 定义 |
|-----|--------|------|---------|------------|
| 1 | 2021-09-04 | 初始创建枚举 | `'cancelled', 'accepted', 'rejected', 'pending'` (小写) | 未知（后续添加 @map） |
| 2 | 2022-06-04 | 数据迁移（布尔字段 → 枚举） | 不变 | 不变 |
| 3 | - | 添加 @map 映射 | 不变（小写） | `CANCELLED @map("cancelled")` |
| 4 | 2023-12-13 | 新增 AWAITING_HOST | 新增 `'awaiting_host'` | `AWAITING_HOST @map("awaiting_host")` |

---

### 6A.2 阶段 1：初始创建枚举（2021-09-04）

**迁移文件**: `packages/prisma/migrations/20210904162403_add_booking_status_enum/migration.sql`

```sql
-- CreateEnum
CREATE TYPE "BookingStatus" AS ENUM ('cancelled', 'accepted', 'rejected', 'pending');

-- AlterTable
ALTER TABLE "Booking" ADD COLUMN     "status" "BookingStatus" NOT NULL DEFAULT E'accepted';
```

**关键特征**:
- 数据库枚举值为**小写字符串**
- 4 个状态值：`'cancelled'`, `'accepted'`, `'rejected'`, `'pending'`
- 默认值为 `'accepted'`

---

### 6A.3 阶段 2：数据迁移（2022-06-04）

**迁移文件**: `packages/prisma/migrations/20220604144700_fixes_booking_status/migration.sql`

```sql
-- Set BookingStatus.PENDING
UPDATE "Booking" SET "status" = 'pending' WHERE "confirmed" = false AND "rejected" = false AND "rescheduled" IS NOT true;

-- Set BookingStatus.REJECTED
UPDATE "Booking" SET "status" = 'rejected' WHERE "confirmed" = false AND "rejected" = true AND "rescheduled" IS NOT true;

-- Set BookingStatus.CANCELLED
UPDATE "Booking" SET "status" = 'cancelled' WHERE "confirmed" = false AND "rejected" = false AND "rescheduled" IS true;

-- Set BookingStatus.ACCEPTED
UPDATE "Booking" SET "status" = 'accepted' WHERE "confirmed" = true AND "rejected" = false AND "rescheduled" IS NOT true;
```

**迁移逻辑**:
| 旧字段条件 | 新 status 值 |
|-----------|-------------|
| `confirmed=false AND rejected=false AND rescheduled<>true` | `'pending'` |
| `confirmed=false AND rejected=true AND rescheduled<>true` | `'rejected'` |
| `confirmed=false AND rejected=false AND rescheduled=true` | `'cancelled'` |
| `confirmed=true AND rejected=false AND rescheduled<>true` | `'accepted'` |

---

### 6A.4 阶段 3：Schema @map 映射（大小写转换）

**当前 Schema 定义**: `packages/prisma/schema.prisma:843-849`

```prisma
enum BookingStatus {
  CANCELLED     @map("cancelled")
  ACCEPTED      @map("accepted")
  REJECTED      @map("rejected")
  PENDING       @map("pending")
  AWAITING_HOST @map("awaiting_host")
}
```

**@map 指令的作用**:
- **TypeScript 代码中**: 使用大写枚举成员名 `BookingStatus.CANCELLED`
- **数据库中**: 存储小写值 `'cancelled'`
- **Prisma 自动处理转换**: 开发者无需手动转换

---

### 6A.5 阶段 4：新增 AWAITING_HOST（2023-12-13）

**迁移文件**: `packages/prisma/migrations/20231213153230_add_instant_meeting/migration.sql`

```sql
-- AlterEnum
ALTER TYPE "BookingStatus" ADD VALUE 'awaiting_host';

-- AlterEnum
ALTER TYPE "WebhookTriggerEvents" ADD VALUE 'INSTANT_MEETING';

-- AlterTable
ALTER TABLE "EventType" ADD COLUMN     "isInstantEvent" BOOLEAN NOT NULL DEFAULT false;

-- CreateTable
CREATE TABLE "InstantMeetingToken" ( ... );
```

**关键信息**:
- PostgreSQL 枚举新增值为 `'awaiting_host'`（小写）
- 与即时会议（Instant Meeting）功能关联
- Schema 中对应: `AWAITING_HOST @map("awaiting_host")`

---

### 6A.6 类型生成影响分析

#### 枚举生成器输出

**生成器配置**: `packages/prisma/schema.prisma:52-55`

```prisma
generator enums {
  provider = "prisma-enum-generator"
  output   = "./enums"
}
```

**生成器实现**: `packages/prisma/enum-generator.ts:1-37`

```typescript
generatorHandler({
  onManifest() {
    return {
      defaultOutput: "./enums/index.ts",
      prettyName: "Prisma Enum Generator",
    };
  },
  async onGenerate(options) {
    const enums = options.dmmf.datamodel.enums;
    const output = enums.map((e) => {
      let enumString = `export const ${e.name} = {\n`;
      e.values.forEach(({ name: value }) => {
        enumString += `  ${value}: "${value}",\n`;  // 使用 schema 中的成员名（大写）
      });
      enumString += `} as const;\n\n`;
      enumString += `export type ${e.name} = (typeof ${e.name})[keyof typeof ${e.name}];\n`;
      return enumString;
    });
    // ... 写入文件
  },
});
```

**生成的类型（概念）**:
```typescript
export const BookingStatus = {
  CANCELLED: "CANCELLED",      // 大写字符串值
  ACCEPTED: "ACCEPTED",
  REJECTED: "REJECTED",
  PENDING: "PENDING",
  AWAITING_HOST: "AWAITING_HOST",
} as const;

export type BookingStatus = (typeof BookingStatus)[keyof typeof BookingStatus];
// 即: "CANCELLED" | "ACCEPTED" | "REJECTED" | "PENDING" | "AWAITING_HOST"
```

**注意**: 
- 生成的 TypeScript 枚举使用**大写**字符串值
- 但数据库中存储的是**小写**值
- Prisma Client 在读写时自动通过 `@map` 进行转换

---

### 6A.7 消费端映射规则

#### 规则 1：Prisma ORM 自动转换（内部使用）

**场景**: 使用 Prisma Client 进行数据库操作

**代码**: `packages/features/bookings/lib/handleNewBooking/createBooking.ts:6-7`

```typescript
import { BookingStatus } from "@calcom/prisma/enums";

// 使用大写枚举
const status = BookingStatus.ACCEPTED;  // "ACCEPTED"

// Prisma 自动转换为数据库小写 'accepted'
await prisma.booking.update({
  where: { id },
  data: { status: BookingStatus.ACCEPTED }
});
```

**转换机制**: Prisma Client 根据 `@map` 自动转换

---

#### 规则 2：Kysely 原始 SQL 查询需手动转换

**场景**: 使用 Kysely 进行复杂查询

**文件**: `packages/trpc/server/routers/viewer/bookings/get.handler.ts:483-500`

```typescript
eb
  .cast<BookingStatus>(
    eb
      .case()
      .when("Booking.status", "=", "cancelled")      // 数据库小写
      .then(BookingStatus.CANCELLED)                  // 代码大写
      .when("Booking.status", "=", "accepted")       // 数据库小写
      .then(BookingStatus.ACCEPTED)                   // 代码大写
      .when("Booking.status", "=", "rejected")       // 数据库小写
      .then(BookingStatus.REJECTED)                   // 代码大写
      .when("Booking.status", "=", "pending")        // 数据库小写
      .then(BookingStatus.PENDING)                    // 代码大写
      .when("Booking.status", "=", "awaiting_host")  // 数据库小写
      .then(BookingStatus.AWAITING_HOST)              // 代码大写
      .else(BookingStatus.PENDING)
      .end(),
    "varchar"
  )
  .as("status"),
```

**映射表**:
| 数据库值（Kysely 条件） | TypeScript 枚举值 |
|----------------------|-----------------|
| `"cancelled"` | `BookingStatus.CANCELLED` |
| `"accepted"` | `BookingStatus.ACCEPTED` |
| `"rejected"` | `BookingStatus.REJECTED` |
| `"pending"` | `BookingStatus.PENDING` |
| `"awaiting_host"` | `BookingStatus.AWAITING_HOST` |

**关键点**: Kysely 是原始 SQL 工具，**不了解 Prisma 的 @map 映射**，必须手动转换

---

#### 规则 3：业务逻辑中使用大写枚举

**文件**: `packages/trpc/server/routers/viewer/bookings/reportBooking.handler.ts:69-73`

```typescript
import { BookingStatus } from "@calcom/prisma/enums";

const isUpcoming =
  (booking.status === BookingStatus.ACCEPTED ||      // 大写
    booking.status === BookingStatus.PENDING ||       // 大写
    booking.status === BookingStatus.AWAITING_HOST) && // 大写
    new Date(booking.startTime) > new Date();
```

---

#### 规则 4：API v2 2024-04-15：大写字符串（与 Prisma 枚举一致）

**文件**: `apps/api/v2/src/platform/bookings/2024-04-15/outputs/get-bookings.output.ts:19-27`

```typescript
const Status = {
  CANCELLED: "CANCELLED",      // 大写
  REJECTED: "REJECTED",        // 大写
  ACCEPTED: "ACCEPTED",        // 大写
  PENDING: "PENDING",          // 大写
  AWAITING_HOST: "AWAITING_HOST",  // 大写 ✅ 包含
} as const;

export type Status = (typeof Status)[keyof typeof Status];
```

**使用**: `packages/api/v2/src/platform/bookings/2024-04-15/outputs/get-bookings.output.ts:222-224`

```typescript
@IsEnum(Status)
@ApiProperty({ enum: Status, type: String })
status!: Status;  // "CANCELLED" | "REJECTED" | "ACCEPTED" | "PENDING" | "AWAITING_HOST"
```

**一致性**: 与 Prisma 生成的枚举值完全一致（大写）

---

#### 规则 5：API v2 2024-08-13：小写字符串（与数据库一致）

**文件**: `packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts:158-161`

```typescript
@ApiProperty({ enum: ["cancelled", "accepted", "rejected", "pending"], example: "accepted" })
@IsEnum(["cancelled", "accepted", "rejected", "pending"])
@Expose()
status!: "cancelled" | "accepted" | "rejected" | "pending";  // 小写
```

**关键差异**:
1. 使用**小写**字符串（与数据库值一致）
2. **缺少 AWAITING_HOST** ❌
3. 枚举值硬编码在装饰器中：`["cancelled", "accepted", "rejected", "pending"]`

---

### 6A.8 大小写与向后兼容边界

#### 三层表示系统

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    三层 BookingStatus 表示系统                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  数据库层（PostgreSQL）                                                  │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  ENUM 值（存储在磁盘）                                               │  │
│  │  'cancelled' | 'accepted' | 'rejected' | 'pending' | 'awaiting_host'│  │
│  │  始终小写                                                           │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│                              │ Prisma @map 自动转换                      │
│                              ▼                                          │
│  TypeScript 代码层                                                        │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Prisma 枚举（编译时类型）                                           │  │
│  │  BookingStatus.CANCELLED    → "CANCELLED"（大写字符串）              │  │
│  │  BookingStatus.ACCEPTED     → "ACCEPTED"                           │  │
│  │  BookingStatus.REJECTED     → "REJECTED"                           │  │
│  │  BookingStatus.PENDING      → "PENDING"                            │  │
│  │  BookingStatus.AWAITING_HOST → "AWAITING_HOST"                      │  │
│  │  始终大写                                                           │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│                              │ API 层手动转换                            │
│                              ▼                                          │
│  API 契约层                                                               │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  API v2 2024-04-15（大写）                                          │  │
│  │  "CANCELLED" | "ACCEPTED" | "REJECTED" | "PENDING" | "AWAITING_HOST"│  │
│  │  与 Prisma 枚举一致，包含 AWAITING_HOST ✅                           │  │
│  │                                                                     │  │
│  │  API v2 2024-08-13（小写）                                          │  │
│  │  "cancelled" | "accepted" | "rejected" | "pending"                 │  │
│  │  与数据库一致，**缺少 AWAITING_HOST** ❌                             │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

#### 向后兼容边界分析

| 维度 | 2024-04-15 | 2024-08-13 | 兼容性说明 |
|-----|-----------|-----------|-----------|
| **大小写** | 大写字符串 | 小写字符串 | **不兼容**：客户端需要调整大小写 |
| **AWAITING_HOST** | ✅ 支持 | ❌ 不支持 | **不兼容**：即时会议状态丢失 |
| **枚举值来源** | 常量对象 `Status` | 硬编码数组 | 硬编码难以同步 |
| **与 Prisma 同步** | ✅ 一致 | ❌ 不一致 | 需要额外转换层 |

**问题 1：大小写变更**

```typescript
// API v2 2024-04-15 响应
{
  "status": "ACCEPTED"  // 大写
}

// API v2 2024-08-13 响应
{
  "status": "accepted"  // 小写
}
```

**影响**: 客户端比较逻辑 `status === "ACCEPTED"` 在升级后会失败

---

**问题 2：AWAITING_HOST 缺失**

```typescript
// 数据库状态
const dbStatus = 'awaiting_host';  // 实际存在

// API v2 2024-04-15 能正确返回
"status": "AWAITING_HOST"

// API v2 2024-08-13 验证失败
@IsEnum(["cancelled", "accepted", "rejected", "pending"])  // 无 awaiting_host
// 可能导致：
// - 序列化时过滤掉该字段
// - 验证错误
// - 前端无法识别即时会议状态
```

---

**问题 3：Kysely 映射维护负担**

**文件**: `packages/trpc/server/routers/viewer/bookings/get.handler.ts:486-496`

```typescript
.when("Booking.status", "=", "cancelled")
.then(BookingStatus.CANCELLED)
.when("Booking.status", "=", "accepted")
.then(BookingStatus.ACCEPTED)
.when("Booking.status", "=", "rejected")
.then(BookingStatus.REJECTED)
.when("Booking.status", "=", "pending")
.then(BookingStatus.PENDING)
.when("Booking.status", "=", "awaiting_host")  // 必须手动添加
.then(BookingStatus.AWAITING_HOST)             // 必须手动添加
```

**风险**:
- 每次新增枚举值都需要更新所有 Kysely 查询
- 遗漏会导致 `.else(BookingStatus.PENDING)` 将新状态错误映射为 PENDING

---

### 6A.9 完整演进时间线

```
2021-09-04
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  创建 BookingStatus 枚举                                    │
│  数据库值: 'cancelled', 'accepted', 'rejected', 'pending'  │
│  4 个状态，全小写                                            │
└─────────────────────────────────────────────────────────────┘
    │
    │ 2022-06-04
    ▼
┌─────────────────────────────────────────────────────────────┐
│  数据迁移: confirmed/rejected/rescheduled → status          │
│  数据库值不变                                                │
└─────────────────────────────────────────────────────────────┘
    │
    │ (时间未知)
    ▼
┌─────────────────────────────────────────────────────────────┐
│  添加 @map 映射                                              │
│  Schema: CANCELLED @map("cancelled")                        │
│  Prisma 自动处理大小写转换                                    │
│  TypeScript: BookingStatus.CANCELLED ("CANCELLED")          │
│  数据库: 'cancelled'                                        │
└─────────────────────────────────────────────────────────────┘
    │
    │ 2023-12-13
    ▼
┌─────────────────────────────────────────────────────────────┐
│  新增 AWAITING_HOST                                          │
│  数据库: ALTER TYPE ADD VALUE 'awaiting_host'               │
│  Schema: AWAITING_HOST @map("awaiting_host")                │
│  TypeScript: BookingStatus.AWAITING_HOST                    │
└─────────────────────────────────────────────────────────────┘
    │
    │ 2024-04-15
    ▼
┌─────────────────────────────────────────────────────────────┐
│  API v2 2024-04-15 发布                                      │
│  状态枚举: 大写，包含 AWAITING_HOST                          │
│  "CANCELLED" | "ACCEPTED" | "REJECTED" | "PENDING" |        │
│  "AWAITING_HOST"                                            │
└─────────────────────────────────────────────────────────────┘
    │
    │ 2024-08-13
    ▼
┌─────────────────────────────────────────────────────────────┐
│  API v2 2024-08-13 发布                                      │
│  状态枚举: 小写，**缺少 AWAITING_HOST**                      │
│  "cancelled" | "accepted" | "rejected" | "pending"          │
│  ⚠️ 与数据库一致，但与内部枚举不一致                           │
│  ⚠️ 即时会议状态无法通过 API 返回                              │
└─────────────────────────────────────────────────────────────┘
```

---

### 6A.10 各层映射汇总表

| 层级 | 表示方式 | 值列表 | 可核对文件 |
|-----|---------|-------|-----------|
| **数据库** | PostgreSQL 枚举（小写） | `'cancelled'`, `'accepted'`, `'rejected'`, `'pending'`, `'awaiting_host'` | `packages/prisma/migrations/20210904162403_add_booking_status_enum/migration.sql`<br>`packages/prisma/migrations/20231213153230_add_instant_meeting/migration.sql` |
| **Schema 定义** | Prisma 成员名（大写） | `CANCELLED`, `ACCEPTED`, `REJECTED`, `PENDING`, `AWAITING_HOST` | `packages/prisma/schema.prisma:843-849` |
| **@map 映射** | 数据库值（小写） | `@map("cancelled")`, `@map("accepted")`, `@map("rejected")`, `@map("pending")`, `@map("awaiting_host")` | `packages/prisma/schema.prisma:843-849` |
| **Prisma Client** | 自动转换 | 读写时自动处理大小写 | N/A（动态生成） |
| **Kysely 查询** | 手动 CASE 转换 | 数据库小写 → 枚举大写 | `packages/trpc/server/routers/viewer/bookings/get.handler.ts:483-500` |
| **业务逻辑** | Prisma 枚举（大写字符串） | `BookingStatus.CANCELLED` → `"CANCELLED"` | `packages/trpc/server/routers/viewer/bookings/reportBooking.handler.ts:70-73` |
| **API v2 2024-04-15** | 自定义常量（大写） | `"CANCELLED"`, `"ACCEPTED"`, `"REJECTED"`, `"PENDING"`, `"AWAITING_HOST"` | `apps/api/v2/src/platform/bookings/2024-04-15/outputs/get-bookings.output.ts:19-27` |
| **API v2 2024-08-13** | 硬编码数组（小写） | `"cancelled"`, `"accepted"`, `"rejected"`, `"pending"`（**缺少 awaiting_host**） | `packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts:158-161` |

---

### 6A.11 关键风险点

#### 风险 1：API v2 2024-08-13 AWAITING_HOST 缺失

**问题**:
```typescript
// packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts:158-161
@IsEnum(["cancelled", "accepted", "rejected", "pending"])  // ❌ 无 "awaiting_host"
status!: "cancelled" | "accepted" | "rejected" | "pending";
```

**影响**:
- 即时会议创建的预约状态为 `AWAITING_HOST`（数据库 `'awaiting_host'`）
- 使用 API v2 2024-08-13 时：
  - 验证可能失败
  - 或被错误映射为其他状态
  - 前端无法正确显示即时会议状态

#### 风险 2：Kysely CASE 表达式不同步

**问题**:
```typescript
// 每次新增状态都需要手动添加
.when("Booking.status", "=", "new_status")
.then(BookingStatus.NEW_STATUS)
```

**风险**: 遗漏会导致 `.else(BookingStatus.PENDING)` 静默降级

#### 风险 3：API 版本间大小写不兼容

**问题**: 2024-04-15（大写）→ 2024-08-13（小写）

**影响**: 客户端字符串比较逻辑需要修改

## 7. 逐跳证据链：EventType.length 字段

**目的**：展示 `EventType.length` 字段从 schema 到业务消费的完整可核对路径

### 第 1 跳：Schema 定义

**文件**: `packages/prisma/schema.prisma:156-159`

```prisma
model EventType {
  id                Int                     @id @default(autoincrement())
  title             String
  slug              String                  @unique(map: "EventType_slug_key")
  length            Int
  // ...
}
```

### 第 2 跳：迁移脚本

**文件**: `packages/prisma/migrations/20210605225044_init/migration.sql`

```sql
CREATE TABLE "EventType" (
    "id" SERIAL NOT NULL,
    "title" TEXT NOT NULL,
    "slug" TEXT NOT NULL,
    "description" TEXT,
    "length" INTEGER NOT NULL,
    -- ...
);
```

### 第 3 跳：Prisma Selects 定义

**文件**: `packages/prisma/selects/event-types.ts:3-20`

```typescript
export const baseEventTypeSelect = {
  id: true,
  title: true,
  description: true,
  length: true,  // ✅ 包含 length 字段 (第7行)
  schedulingType: true,
  recurringEvent: true,
  slug: true,
  // ...
} satisfies Prisma.EventTypeSelect;
```

**文件**: `packages/prisma/selects/event-types.ts:22-71`

```typescript
export const bookEventTypeSelect = {
  id: true,
  title: true,
  slug: true,
  description: true,
  length: true,  // ✅ 包含 length 字段 (第27行)
  locations: true,
  customInputs: true,
  // ...
} satisfies Prisma.EventTypeSelect;
```

**文件**: `packages/prisma/selects/event-types.ts:73-128`

```typescript
export const availiblityPageEventTypeSelect = {
  id: true,
  title: true,
  availability: true,
  description: true,
  length: true,  // ✅ 包含 length 字段 (第78行)
  offsetStart: true,
  // ...
} satisfies Prisma.EventTypeSelect;
```

### 第 4 跳：Selects 导出

**文件**: `packages/prisma/selects/index.ts:1-5`

```typescript
export { safeAppSelect } from "./app";
export * from "./booking";
export { safeCredentialSelect } from "./credential";
export * from "./event-types";  // ✅ 导出 event-types 的所有 select
export * from "./user";
```

### 第 5 跳：Prisma 包导出

**文件**: `packages/prisma/index.ts:112-113`

```typescript
export default prisma;
export * from "./selects";  // ✅ 导出所有 selects
```

### 第 6 跳：业务类型定义

**文件**: `packages/features/eventtypes/lib/types.ts:1-25`

```typescript
import type { ConnectedApps } from "@calcom/app-store/_utils/getConnectedApps";
import type { EventLocationType } from "@calcom/app-store/locations";
import type { eventTypeMetaDataSchemaWithTypedApps } from "@calcom/app-store/zod-utils";
import type { ChildrenEventType } from "@calcom/features/eventtypes/lib/childrenEventType";
import type { IntervalLimit } from "@calcom/lib/intervalLimits/intervalLimitSchema";
import type { EventTypeTranslation } from "@calcom/prisma/client";
import type {
  CancellationReasonRequirement,
  MembershipRole,
  PeriodType,
  SchedulingType,
} from "@calcom/prisma/enums";
import type {
  BookerLayoutSettings,
  CustomInputSchema,
  customInputSchema,
  EventTypeLocation,
  EventTypeMetadata,
  eventTypeBookingFields,
  eventTypeColor,
} from "@calcom/prisma/zod-utils";
import type { RecurringEvent } from "@calcom/types/Calendar";
import type { UserProfile } from "@calcom/types/UserProfile";
import type { z } from "zod";
import type { EventType } from "./getEventTypeById";
```

**说明**：业务类型通过 `import type { EventType } from "./getEventTypeById"` 间接引用，而 `getEventTypeById` 会使用 Prisma 的 `length` 字段

### 第 7 跳：平台类型桥接

**文件**: `packages/platform/libraries/index.ts`

平台桥接层不直接导出 `length` 字段，但导出了相关类型如 `PeriodType`、`SchedulingType` 等枚举

### 第 8 跳：API v2 消费（重命名解耦）

**文件**: `packages/platform/types/bookings/2024-08-13/inputs/create-booking.input.ts`

```typescript
export class CreateBookingInput_2024_08_13 {
  // ...
  @ApiPropertyOptional({ type: Number, description: "The length of the booking in minutes.", example: 30 })
  @IsNumber()
  @IsOptional()
  lengthInMinutes?: number;  // ✅ API 层重命名为 lengthInMinutes
  // ...
}
```

**文件**: `packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts`

```typescript
export class BaseBookingOutput_2024_08_13 {
  @ApiProperty({
    type: Number,
    description: "The duration of the booking in minutes.",
    example: 30,
  })
  @IsNumber()
  @Expose()
  duration!: number;  // ✅ API 层重命名为 duration
}
```

**说明**:
- 内部使用 `length` (Int)
- API 输入使用 `lengthInMinutes` (number)
- API 输出使用 `duration` (number)
- 命名不同，但语义相同，实现了内部模型与 API 契约的解耦

### EventType.length 证据链总结

```
Schema 定义 (packages/prisma/schema.prisma:156-159)
    ↓
迁移脚本 (packages/prisma/migrations/20210605225044_init/migration.sql)
    ↓
Prisma Selects (packages/prisma/selects/event-types.ts:7, 27, 78)
    ↓
Selects 导出 (packages/prisma/selects/index.ts:4)
    ↓
Prisma 包导出 (packages/prisma/index.ts:113)
    ↓
业务类型定义 (packages/features/eventtypes/lib/types.ts)
    ↓
平台桥接 (packages/platform/libraries/index.ts) [间接依赖]
    ↓
API v2 输入 (packages/platform/types/bookings/2024-08-13/inputs/create-booking.input.ts) [lengthInMinutes]
    ↓
API v2 输出 (packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts) [duration]
```

## 8. 耦合边界分析（可核对版本）

### 8.1 耦合层级

#### 层级 1：核心耦合（Prisma 包）
**位置**: `packages/prisma/`

**实际存在的文件**:
- `packages/prisma/schema.prisma` - 数据模型定义
- `packages/prisma/index.ts` - 导出入口
- `packages/prisma/selects/*` - 预定义查询字段
- `packages/prisma/zod-utils.ts` - 手动 Zod schema

**耦合特点**:
- 所有业务包都依赖 Prisma 类型
- 任何 schema 变更都会影响所有依赖方
- 这是"必要耦合"，无法避免

#### 层级 2：业务逻辑耦合（Features 包）
**位置**: `packages/features/`

**实际存在的文件**:
- `packages/features/bookings/types.ts`
- `packages/features/bookings/lib/bookingCreateBodySchema.ts`
- `packages/features/bookings/lib/handleNewBooking/createBooking.ts`
- `packages/features/eventtypes/lib/types.ts`

**耦合特点**:
- 依赖 Prisma 类型进行数据操作
- 包含业务规则和验证逻辑
- 被 tRPC 和 API v2 复用

#### 层级 3：API 层耦合
**位置**: `packages/trpc/`, `apps/api/v2/`

**实际存在的文件**:
- `packages/trpc/server/routers/viewer/bookings/get.handler.ts`
- `packages/platform/libraries/index.ts`
- `packages/platform/types/bookings/2024-08-13/*`

**耦合特点**:
- **tRPC 层**: 直接依赖 Prisma 和 features
- **API v2**: 通过 platform-libraries 间接依赖，降低耦合

### 8.2 解耦策略

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

#### 策略 3：命名解耦
**示例**:
- 内部: `EventType.length` (Int)
- API 输入: `lengthInMinutes` (number)
- API 输出: `duration` (number)

**优势**: 命名不同但语义相同，实现解耦

### 8.3 耦合边界图示

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    API 边界（实际存在文件）                                        │
│  ┌──────────────┐    ┌────────────────────────────┐     ┌────────────────────┐ │
│  │   API v2     │◄───│  platform-libraries        │◄────│  platform-types    │ │
│  │  (NestJS)    │    │  (桥接层)                   │     │  (版本化 DTO)      │ │
│  └──────────────┘    └────────────────────────────┘     └────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                 业务逻辑边界（实际存在文件）                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │              @calcom/features                                           │   │
│  │  bookings, eventtypes, calendars, webhooks...                          │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                       │                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │              @calcom/trpc                                               │   │
│  │        routers, handlers, schemas                                       │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                  数据层边界（实际存在文件）                                         │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │              @calcom/prisma                                             │   │
│  │  schema.prisma, index.ts, selects/, zod-utils.ts, enum-generator.ts    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │              PostgreSQL Database                                        │   │
│  │  migrations/*/migration.sql                                             │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  动态生成（运行时创建，磁盘上不存在）：                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  generated/prisma/client/ (PrismaClient)                                │   │
│  │  enums/ (prisma-enum-generator)                                         │   │
│  │  项目不直接使用自动生成的 zod/ 目录                                        │   │
│  │  ../kysely/types.ts (prisma-kysely)                                     │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 9. 真实字段变更案例：Attendee.phoneNumber

### 9.1 变更背景

**迁移脚本**: `packages/prisma/migrations/20240408155446_add_phone_number_in_attendee/migration.sql`

```sql
-- AlterTable
ALTER TABLE "Attendee" ADD COLUMN     "phoneNumber" TEXT;
```

### 9.2 Schema 定义

**文件**: `packages/prisma/schema.prisma:826-841`

```prisma
model Attendee {
  id           Int           @id @default(autoincrement())
  email        String
  name         String
  timeZone     String
  phoneNumber  String?
  locale       String?       @default("en")
  booking      Booking?      @relation(fields: [bookingId], references: [id], onDelete: Cascade)
  bookingId    Int?
  bookingSeat  BookingSeat?
  noShow       Boolean?      @default(false)
}
```

### 9.3 受影响模块（可核对）

| 模块 | 文件路径 | 耦合类型 |
|-----|---------|---------|
| 业务服务 | `packages/features/bookings/services/BookingAttendeesService.ts` | 类型耦合 |
| 日历构建器 | `packages/features/CalendarEventBuilder.ts` | 类型耦合 |
| API v2 服务 | `apps/api/v2/src/platform/bookings/2024-08-13/services/booking-attendees.service.ts` | 逻辑耦合 |
| 平台输出类型 | `packages/platform/types/bookings/2024-08-13/outputs/booking.output.ts` | 契约耦合 |
| 平台输入类型 | `packages/platform/types/bookings/2024-08-13/inputs/create-booking.input.ts` | 契约耦合 |
| 前端消费 | `apps/web/lib/booking.ts` | 隐式耦合 |
| 测试工具 | `packages/testing/src/lib/bookingScenario/bookingScenario.ts` | 隐式耦合 |

### 9.4 版本隔离验证

**无影响模块**: `packages/platform/types/bookings/2024-04-15/`

旧版本 API DTO 不受 `phoneNumber` 字段变更影响，版本隔离成功

## 10. 可核对引用规范

### 10.1 正确引用方式

| 类型 | 正确引用 | 错误引用 |
|-----|---------|---------|
| 枚举定义 | `packages/prisma/schema.prisma` | `packages/prisma/enums/index.ts` |
| 枚举使用 | `import { BookingStatus } from "@calcom/prisma/enums"` (文件内容) | `packages/prisma/enums/` 目录路径 |
| Prisma Client | `packages/prisma/client/index.ts` (转发入口) | `packages/prisma/generated/prisma/client/index.d.ts` |
| 生成器配置 | `packages/prisma/schema.prisma` (generator 块) | 输出目录路径 |
| 生成器实现 | `packages/prisma/enum-generator.ts` | 无 |
| Zod schema | `packages/prisma/zod-utils.ts` (手动维护) | `packages/prisma/zod/` |
| Kysely 类型 | `packages/prisma/schema.prisma` (表定义) | `packages/kysely/types.ts` |

### 10.2 验证方法

1. **验证文件存在**: 使用文件系统工具检查路径
2. **验证符号引用**: 搜索导入语句确认符号存在
3. **验证生成器**: 查看 schema.prisma 中的 generator 配置
4. **验证枚举**: 查看 schema.prisma 中的 enum 定义
5. **验证实际使用**: 查看业务代码中的具体引用

## 11. 总结

### 11.1 关键发现

1. **动态生成文件不可直接核对**:
   - `packages/prisma/generated/`、`packages/prisma/zod/`、`packages/prisma/enums/`、`packages/kysely/types.ts` 在磁盘上不存在
   - 它们是 `yarn prisma generate` 时动态生成的

2. **实际存在的关键文件**:
   - Schema 定义和迁移脚本
   - Prisma 包的转发入口和手动维护的 Zod utils
   - Prisma Selects 配置
   - 平台桥接层和版本化类型
   - 业务逻辑层代码

3. **两条完整的逐跳证据链**:
   - **Booking.status**: Schema → 迁移 → 枚举生成器 → tRPC → 业务逻辑 → 平台桥接 → API v2（字符串字面量解耦）
   - **EventType.length**: Schema → 迁移 → Prisma Selects → 业务类型 → API v2（重命名解耦）

4. **解耦策略**:
   - 平台桥接层 (`packages/platform/libraries/index.ts`)
   - 版本化平台类型 (`packages/platform/types/`)
   - 命名解耦（`length` → `lengthInMinutes`/`duration`）
   - 字符串字面量替代枚举导入

### 11.2 操作建议

1. **字段变更前**:
   - 确认 schema.prisma 和迁移脚本是唯一的"真相来源"
   - 使用 `yarn type-check:ci --force` 确认基线状态

2. **字段变更中**:
   - 小步提交，每次只变更一个字段
   - 检查实际存在的文件引用，而非动态生成的路径

3. **字段变更后**:
   - 验证所有实际存在的消费文件
   - 运行完整测试套件
