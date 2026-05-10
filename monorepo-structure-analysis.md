# Cal.diy Monorepo 结构分析报告

## 1. 项目概述

Cal.diy 是一个使用 **Yarn Workspaces + Turborepo** 管理的大型 monorepo 项目，采用 Next.js 作为主要前端框架，tRPC 作为 API 层，Prisma 作为 ORM。项目遵循严格的分层架构和依赖倒置原则。

---

## 2. 工作空间配置

### 2.1 根目录结构

```
cal.diy/
├── apps/                          # 应用层
│   ├── web/                       # 主应用 (Next.js)
│   ├── api/
│   │   └── v2/                    # API v2 (NestJS)
│   └── docs/                      # 文档站点
├── packages/                      # 共享包层
│   ├── app-store/                 # 应用商店集成
│   ├── features/                  # 业务特性模块
│   ├── lib/                       # 核心工具库
│   ├── trpc/                      # tRPC API 层
│   ├── prisma/                    # 数据库层
│   ├── ui/                        # UI 组件库
│   ├── platform/                  # 平台层 (API v2 依赖)
│   │   ├── atoms/
│   │   ├── constants/
│   │   ├── enums/
│   │   ├── types/
│   │   ├── utils/
│   │   └── libraries/             # 共享库桥接层 ⭐
│   ├── i18n/                      # 国际化
│   ├── emails/                    # 邮件系统
│   ├── embeds/                    # 嵌入组件
│   └── types/                     # 类型定义
├── turbo.json                     # Turborepo 配置
└── package.json                   # 根 package.json
```

### 2.2 Yarn Workspaces 配置

根 `package.json:5-16` 定义了工作空间：

```json
"workspaces": [
  "apps/*",
  "apps/api/*",
  "packages/*",
  "packages/embeds/*",
  "packages/features/*",
  "packages/app-store",
  "packages/app-store/*",
  "packages/platform/*",
  "packages/platform/examples/base",
  "example-apps/*"
]
```

---

## 3. 包边界划分策略

### 3.1 分层架构原则

项目采用清晰的分层架构，各层职责明确：

```
┌─────────────────────────────────────────────────────────────┐
│                    apps/ (应用层)                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐      │
│  │  @calcom/web│  │ @calcom/api │  │  @calcom/docs   │      │
│  │  (Next.js)  │  │    v2       │  │  (文档)         │      │
│  └──────┬──────┘  └──────┬──────┘  └─────────────────┘      │
└─────────┼────────────────┼──────────────────────────────────┘
          │                │
          ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│              platform/ (平台桥接层)                          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  @calcom/platform-libraries (核心桥接) ⭐           │    │
│  │  @calcom/platform-constants                         │    │
│  │  @calcom/platform-enums                             │    │
│  │  @calcom/platform-types                             │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                 packages/ (核心共享层)                       │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │  @calcom/    │ │ @calcom/     │ │  @calcom/        │    │
│  │  features    │ │  trpc        │ │  app-store       │    │
│  │  (业务逻辑)  │ │ (API 路由)   │ │ (第三方集成)     │    │
│  └──────┬───────┘ └──────┬───────┘ └────────┬─────────┘    │
│         │                │                   │              │
│         ▼                ▼                   ▼              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │  @calcom/    │ │ @calcom/     │ │  @calcom/        │    │
│  │  lib         │ │  prisma      │ │  ui              │    │
│  │  (工具库)    │ │ (数据库)     │ │ (UI 组件)        │    │
│  └──────────────┘ └──────────────┘ └──────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 核心包职责说明

#### 3.2.1 `@calcom/prisma` (数据层)

**位置**: `packages/prisma/`

**职责**:
- 管理数据库 schema (`schema.prisma`)
- 提供 Prisma Client 实例
- 包含数据库迁移和种子脚本
- 定义 Prisma 查询的安全 selects

**关键文件**:
- `packages/prisma/schema.prisma` - 数据库模型定义
- `packages/prisma/selects/` - 预定义的安全查询选择器

**依赖方向**: 被几乎所有业务包依赖

---

#### 3.2.2 `@calcom/lib` (工具库层)

**位置**: `packages/lib/`

**职责**:
- 提供通用工具函数
- 日期时间处理 (dayjs 封装)
- 加密解密工具
- 日志系统
- 环境变量验证
- 通用业务辅助函数

**依赖关系**:
- 依赖: `@calcom/config`, `@calcom/dayjs`, `@calcom/i18n`
- 被依赖: `@calcom/features`, `@calcom/trpc`, `@calcom/ui`

---

#### 3.2.3 `@calcom/trpc` (API 层)

**位置**: `packages/trpc/`

**职责**:
- 定义 tRPC 服务端和客户端配置
- 包含所有 API 路由 (routers)
- 实现 API 中间件 (会话、性能监控等)
- 定义 API procedures (public, authed, pbac)

**目录结构**:
```
packages/trpc/server/
├── routers/
│   ├── viewer/           # 已登录用户路由
│   │   ├── bookings/     # 预订相关
│   │   ├── eventTypes/   # 事件类型
│   │   ├── availability/ # 可用性
│   │   ├── me/           # 个人信息
│   │   ├── ...
│   │   └── _router.tsx
│   ├── publicViewer/     # 公开路由
│   ├── loggedInViewer/   # 登录用户特殊路由
│   └── features/         # 特性路由
├── procedures/           # 过程定义
├── middlewares/          # 中间件
└── _app.ts              # 根路由
```

**依赖关系**:
- 依赖: `@calcom/prisma`, `@calcom/lib`, `@calcom/features`
- 被依赖: `@calcom/web`, `@calcom/features`

---

#### 3.2.4 `@calcom/features` (业务特性层)

**位置**: `packages/features/`

**职责**:
- 封装核心业务逻辑
- 包含领域服务 (Services)
- 包含数据仓库 (Repositories)
- 实现复杂业务流程

**特性模块示例**:
```
packages/features/
├── bookings/        # 预订业务
├── calendars/       # 日历同步
├── eventtypes/      # 事件类型管理
├── auth/            # 认证相关
├── profile/         # 个人资料
├── credentials/     # 凭证管理
├── di/              # 依赖注入
├── tasker/          # 任务调度
├── flags/           # 功能开关
└── ...
```

**架构模式**: 采用 Repository + Service 模式

---

#### 3.2.5 `@calcom/ui` (UI 组件层)

**位置**: `packages/ui/`

**职责**:
- 提供可复用的 React 组件
- 基于 Radix UI 构建
- 遵循设计系统
- 支持主题定制

**导出策略**: 使用子路径导出避免 barrel imports

```json
// packages/ui/package.json
"exports": {
  "./components/button": "./components/button/index.ts",
  "./components/dialog": "./components/dialog/index.ts",
  "./styles": "./styles/index.ts"
}
```

---

#### 3.2.6 `@calcom/app-store` (应用商店)

**位置**: `packages/app-store/`

**职责**:
- 管理第三方应用集成
- 每个集成是独立子目录
- 包含 API handlers、组件、配置

**示例集成**:
```
packages/app-store/
├── zoomvideo/      # Zoom 集成
├── googlecalendar/ # Google 日历
├── stripe/         # Stripe 支付
├── slack/          # Slack 通知
└── ... (50+ 集成)
```

---

### 3.3 包边界规则

根据 `AGENTS.md` 中的约定：

| 规则 | 说明 |
|------|------|
| **单一职责** | 每个包只负责一个领域 |
| **依赖方向** | 从上层到下层，不允许反向依赖 |
| **类型安全** | 所有包使用 TypeScript strict 模式 |
| **禁止循环依赖** | 通过架构设计避免循环 |
| **Barrel Imports** | 禁止使用 index.ts 聚合导出 |

### 3.4 依赖方向的约束机制 ⭐

包边界的依赖方向通过**多层配置**和**架构规则**共同约束：

#### 3.4.1 TypeScript 路径映射约束 (tsconfig.json)

**Web 应用 (`apps/web/tsconfig.json:2-15`)** 有完整的路径映射，可以直接访问所有包：

```json
{
  "extends": "@calcom/tsconfig/nextjs.json",
  "compilerOptions": {
    "paths": {
      "~/*": ["modules/*"],
      "@components/*": ["components/*"],
      "@lib/*": ["lib/*"],
      "@server/*": ["server/*"],
      "@prisma/client/*": ["@calcom/prisma/client/*"],
      "@calcom/testing/*": ["../../packages/testing/src/*"],
      "@calcom/repository/*": ["@calcom/lib/server/repository/*"],
      "@coss/ui/*": ["../../packages/coss-ui/src/*"]
    }
  }
}
```

**API v2 (`apps/api/v2/tsconfig.json:16-38`) 的路径映射被刻意限制，使用多条显式子路径而非通配符**：

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"],
      "@calcom/prisma/client": ["../../../packages/prisma/generated/prisma/client"],
      "@calcom/prisma/enums": ["../../../packages/prisma/enums"],
      "@calcom/platform-constants": ["../../../packages/platform/constants/index.ts"],
      "@calcom/platform-types": ["../../../packages/platform/types/index.ts"],
      "@calcom/platform-utils": ["../../../packages/platform/utils/index.ts"],
      "@calcom/platform-enums": ["../../../packages/platform/enums/index.ts"],
      // ⚠️ 以下是多条显式子路径映射，不是通配符！
      "@calcom/platform-libraries/event-types": ["../../../packages/platform/libraries/event-types.ts"],
      "@calcom/platform-libraries/slots": ["../../../packages/platform/libraries/slots.ts"],
      "@calcom/platform-libraries/emails": ["../../../packages/platform/libraries/emails.ts"],
      "@calcom/platform-libraries/schedules": ["../../../packages/platform/libraries/schedules.ts"],
      "@calcom/platform-libraries/app-store": ["../../../packages/platform/libraries/app-store.ts"],
      "@calcom/platform-libraries/conferencing": ["../../../packages/platform/libraries/conferencing.ts"],
      "@calcom/platform-libraries/repositories": ["../../../packages/platform/libraries/repositories.ts"],
      "@calcom/platform-libraries/bookings": ["../../../packages/platform/libraries/bookings.ts"],
      "@calcom/platform-libraries/private-links": ["../../../packages/platform/libraries/private-links.ts"],
      "@calcom/platform-libraries/organizations": ["../../../packages/platform/libraries/organizations.ts"],
      "@calcom/platform-libraries/errors": ["../../../packages/platform/libraries/errors.ts"],
      "@calcom/platform-libraries/calendars": ["../../../packages/platform/libraries/calendars.ts"],
      "@calcom/platform-libraries/tasker": ["../../../packages/platform/libraries/tasker.ts"],
      "@calcom/platform-libraries/pbac": ["../../../packages/platform/libraries/pbac.ts"]
      // ⚠️ 关键：没有 @calcom/features/* 和 @calcom/trpc/* 的映射！
      // ⚠️ 也没有 "@calcom/platform-libraries" 通配符映射！
    }
  }
}
```

### 逐项对照：tsconfig paths vs package.json exports

**tsconfig.json paths** (`apps/api/v2/tsconfig.json:24-37`) 中的 `@calcom/platform-libraries/*` 映射：

| # | 子路径 | 目标文件 | 是否存在 |
|---|-------|---------|---------|
| 1 | `event-types` | `event-types.ts` | ✅ 存在 |
| 2 | `slots` | `slots.ts` | ✅ 存在 |
| 3 | `emails` | `emails.ts` | ✅ 存在 |
| 4 | `schedules` | `schedules.ts` | ✅ 存在 |
| 5 | `app-store` | `app-store.ts` | ✅ 存在 |
| 6 | `conferencing` | `conferencing.ts` | ✅ 存在 |
| 7 | `repositories` | `repositories.ts` | ✅ 存在 |
| 8 | `bookings` | `bookings.ts` | ✅ 存在 |
| 9 | `private-links` | `private-links.ts` | ✅ 存在 |
| 10 | `organizations` | `organizations.ts` | ✅ 存在 |
| 11 | `errors` | `errors.ts` | ✅ 存在 |
| 12 | `calendars` | `calendars.ts` | ✅ 存在 |
| 13 | `tasker` | `tasker.ts` | ✅ 存在 |
| 14 | `pbac` | `pbac.ts` | ❌ **文件不存在！** |

**共 14 条子路径映射**（不是 16 条），其中 **pbac 子路径的目标文件不存在**（Glob 搜索未找到 `pbac.ts`）。

**package.json exports** (`packages/platform-libraries/package.json:38-108`) 中的子路径出口：

| # | 子路径 | 备注 |
|---|-------|------|
| 1 | `app-store` | ✅ 对应 tsconfig |
| 2 | `event-types` | ✅ 对应 tsconfig |
| 3 | `.` (根路径) | ⚠️ tsconfig **没有**根路径映射 |
| 4 | `schedules` | ✅ 对应 tsconfig |
| 5 | `emails` | ✅ 对应 tsconfig |
| 6 | `slots` | ✅ 对应 tsconfig |
| 7 | `conferencing` | ✅ 对应 tsconfig |
| 8 | `repositories` | ✅ 对应 tsconfig |
| 9 | `bookings` | ✅ 对应 tsconfig |
| 10 | `organizations` | ✅ 对应 tsconfig |
| 11 | `private-links` | ✅ 对应 tsconfig |
| 12 | `errors` | ✅ 对应 tsconfig |
| 13 | `calendars` | ✅ 对应 tsconfig |
| 14 | `tasker` | ✅ 对应 tsconfig |

**共 14 个子路径出口**（含根路径 `.`）。

### 对照结论（事实校准）

**对应关系**:
- tsconfig 有 14 条 `@calcom/platform-libraries/*` 子路径映射
- package.json exports 有 14 个子路径出口（含根路径 `.`）
- 13 条子路径一一对应

**不一致项**:

| 问题 | 详细说明 | 证据 |
|------|---------|------|
| **pbac 路径映射指向不存在的文件** | tsconfig 有 `@calcom/platform-libraries/pbac` 映射，但 `packages/platform/libraries/pbac.ts` 文件不存在 | Glob 搜索无结果 |
| **根路径导入的工作方式** | 代码中实际使用 `from "@calcom/platform-libraries"`（根路径）导入，但 tsconfig 没有根路径映射 | `apps/api/v2/src/platform/bookings/2024-08-13/services/bookings.service.ts:1-11` |
| **exports 有根路径，tsconfig 没有** | package.json exports 有 `"."` 出口，但 tsconfig 没有对应的 `@calcom/platform-libraries` 映射 | 对照两份配置 |

**事实修正后的关键发现**:

1. **数量修正**: tsconfig 中有 **14 条** `@calcom/platform-libraries/*` 子路径映射（不是 16 条），package.json exports 有 **14 个**子路径出口

2. **根路径导入也在使用**: 代码中同时存在两种导入方式：
   ```typescript
   // 方式一：子路径导入（tsconfig 有映射）
   import { BookingCancelService } from "@calcom/platform-libraries/bookings";
   
   // 方式二：根路径导入（tsconfig 没有映射，但 package.json exports 有）
   import { confirmBookingHandler } from "@calcom/platform-libraries";
   ```
   参考证据: `apps/api/v2/src/platform/bookings/2024-08-13/services/bookings.service.ts:1-12`

3. **pbac 是无效映射**: `@calcom/platform-libraries/pbac` 映射指向不存在的文件，且代码库中没有任何地方使用 `from "@calcom/platform-libraries/pbac"` 的导入

4. **完全缺失 features 和 trpc 映射**: tsconfig 中**完全没有**配置 `@calcom/features/*` 和 `@calcom/trpc/*` 的路径映射

#### 3.4.2 Package.json 依赖声明约束

各包通过 `package.json` 的 `dependencies` 显式声明依赖，Yarn Workspaces 确保只能依赖已声明的包：

**Web 应用可以直接依赖 features (`apps/web/package.json:35-48`)**：
```json
"dependencies": {
  "@calcom/app-store": "workspace:*",
  "@calcom/features": "workspace:*",    // ✅ 直接依赖
  "@calcom/lib": "workspace:*",
  "@calcom/prisma": "workspace:*",
  "@calcom/trpc": "workspace:*",         // ✅ 直接依赖
  "@calcom/ui": "workspace:*"
}
```

**API v2 只能依赖 platform 包 (`apps/api/v2/package.json:42-49`)**：
```json
"dependencies": {
  "@calcom/platform-constants": "workspace:*",
  "@calcom/platform-enums": "workspace:*",
  "@calcom/platform-libraries": "workspace:*",  // ⭐ 唯一的业务逻辑入口
  "@calcom/platform-types": "workspace:*",
  "@calcom/platform-utils": "workspace:*",
  "@calcom/prisma": "workspace:*"
  // ⚠️ 没有 @calcom/features 和 @calcom/trpc
}
```

#### 3.4.3 Platform Libraries 作为受控出口

`@calcom/platform-libraries` 是一个**显式受控的出口层**，只导出需要被 API v2 使用的能力：

```typescript
// packages/platform/libraries/index.ts
export { confirmHandler as confirmBookingHandler } from "@calcom/trpc/server/routers/viewer/bookings/confirm.handler";
export { ProfileRepository } from "@calcom/features/profile/repositories/ProfileRepository";
export { BookingAccessService } from "@calcom/features/bookings/services/BookingAccessService";
// ... 只导出需要的，不是全部
```

这种设计确保了：
1. API v2 不能随意访问 features 的内部实现
2. 所有复用点都是显式声明的，易于追踪
3. Web 和 API v2 的业务逻辑保持一致

#### 3.4.4 架构规则约束 (AGENTS.md)

项目还通过 `AGENTS.md` 中的开发规范强化边界：

| 约束项 | 规则内容 | 强制执行方式 |
|--------|---------|-------------|
| **禁止 Barrel Imports** | "Never use barrel imports from index.ts files" | 代码审查 + lint 规则 |
| **直接导入源文件** | "Import directly from source files, not barrel files" | 示例：`@calcom/ui/components/button` |
| **Repository 不包含业务逻辑** | "Never put business logic in repositories" | 架构设计审查 |
| **Business Logic 在 Services** | "that belongs in Services" | Repository + Service 模式 |

---

## 4. 构建入口与 Turborepo 配置

### 4.1 主构建入口

**根 `package.json:30`** 定义主构建命令：

```json
"build": "turbo run build --filter=@calcom/web...",
```

这会构建 `@calcom/web` 及其所有依赖。

### 4.2 Turborepo 任务配置

**`turbo.json:317-613`** 定义了详细的任务管道：

#### 4.2.1 关键任务依赖关系

```
@calcom/web#build
  ├── dependsOn: ["^build", "copy-app-store-static"]
  └── 依赖所有上游包的 build

@calcom/prisma#post-install
  ├── cache: false
  └── 在依赖安装后执行 Prisma generate

@calcom/trpc#build
  ├── dependsOn: ["@calcom/prisma#post-install"]
  └── outputs: ["./types"]

type-check
  └── dependsOn: ["@calcom/trpc#build"]
```

#### 4.2.2 开发模式入口

```json
// package.json:47
"dev": "turbo run dev --filter=\"@calcom/web\"",

// apps/web/package.json:10
"dev": "turbo run copy-app-store-static && next dev --turbopack",
```

### 4.3 环境变量管理

Turborepo 配置了全局和任务级别的环境变量：

- **`globalEnv`**: 影响所有任务的环境变量
- **任务级 `env`**: 特定任务的环境变量（如 `@calcom/web#build` 的 env）

---

## 5. 共享业务能力复用机制 ⭐

### 5.1 核心复用策略

Cal.diy 使用**多层次桥接**模式实现业务能力复用：

```
┌─────────────────────────────────────────────────────────────┐
│                    业务能力复用架构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  @calcom/features (业务逻辑实现)                            │
│       │                                                     │
│       ├────> @calcom/platform-libraries (桥接层) ────> apps/api/v2
│       │                                                     │
│       └────> @calcom/trpc (API 层) ────────────────> apps/web
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Platform Libraries 桥接层 ⭐

**位置**: `packages/platform/libraries/index.ts`

这是 API v2 复用 Web 业务逻辑的核心桥梁。由于 `apps/api/v2` 的 tsconfig 没有配置 `@calcom/features` 和 `@calcom/trpc` 的路径映射，通过 `@calcom/platform-libraries` 进行间接导入。

#### 5.2.1 导出的业务能力

从 `packages/platform/libraries/index.ts:1-214` 可以看到导出的核心能力：

**预订相关**:
```typescript
export { getBookingForReschedule } from "@calcom/features/bookings/lib/get-booking";
export { getAllUserBookings } from "@calcom/features/bookings/lib/getAllUserBookings";
export { getBookingInfo } from "@calcom/features/bookings/lib/getBookingInfo";
export { handleCancelBooking } from "@calcom/features/bookings/lib/handleCancelBooking";
export { confirmHandler as confirmBookingHandler } from "@calcom/trpc/server/routers/viewer/bookings/confirm.handler";
```

**日历相关**:
```typescript
export { getBusyCalendarTimes, updateEvent } from "@calcom/features/calendars/lib/CalendarManager";
export { getConnectedDestinationCalendarsAndEnsureDefaultsInDb } from "@calcom/features/calendars/lib/getConnectedDestinationCalendars";
```

**仓库模式 (Repository Pattern)**:
```typescript
export { BookingReferenceRepository } from "@calcom/features/bookingReference/repositories/BookingReferenceRepository";
export { BookingAccessService } from "@calcom/features/bookings/services/BookingAccessService";
export { CredentialRepository } from "@calcom/features/credentials/repositories/CredentialRepository";
export { ProfileRepository } from "@calcom/features/profile/repositories/ProfileRepository";
export { SelectedCalendarRepository } from "@calcom/features/selectedCalendar/repositories/SelectedCalendarRepository";
```

**工具函数**:
```typescript
export { slugify, slugifyLenient } from "@calcom/lib/slugify";
export { symmetricEncrypt, symmetricDecrypt } from "@calcom/lib/crypto";
export { getTranslation } from "@calcom/i18n/server";
```

#### 5.2.2 使用方式 (API v2)

```typescript
// apps/api/v2 中
import { ProfileRepository, getBookingInfo } from "@calcom/platform-libraries";
```

而不是直接：
```typescript
// ❌ 这样会导致 module not found
import { ProfileRepository } from "@calcom/features/profile/repositories/ProfileRepository";
```

### 5.3 tRPC 路由复用

`@calcom/trpc/server/routers/` 中的路由逻辑可以被复用：

**方式一**: 直接复用 handler（通过 platform-libraries）
```typescript
export { confirmHandler as confirmBookingHandler } from "@calcom/trpc/server/routers/viewer/bookings/confirm.handler";
```

**方式二**: Web 应用直接使用 tRPC 客户端
```typescript
// apps/web 中通过 @calcom/trpc/react 调用
const { data } = trpc.viewer.bookings.get.useQuery({ id });
```

### 5.4 UI 组件复用

`@calcom/ui` 采用**子路径导出**策略，支持按需加载：

```typescript
// 推荐：直接导入源文件
import { Button } from "@calcom/ui/components/button";

// ❌ 禁止：barrel import
import { Button } from "@calcom/ui";
```

### 5.5 Prisma Select 复用

`@calcom/prisma/selects/` 中定义了可复用的安全查询选择器：

```typescript
// packages/prisma/selects/booking.ts
export const bookingWithUserAndEventDetailsSelect = {
  id: true,
  title: true,
  user: {
    select: {
      id: true,
      name: true,
      email: true,
    }
  }
};
```

这些选择器通过 `platform-libraries` 导出，供 API v2 使用。

### 5.6 EE 特性占位 (Stub Pattern)

`platform-libraries` 中还包含企业版特性的占位实现：

```typescript
// packages/platform/libraries/index.ts:123-214
export async function roundRobinReassignment(_args: {...}): Promise<void> {
  // No-op in community edition
}
```

这样社区版也能编译通过，在运行时抛出明确的错误。

### 5.7 端到端复用链路示例 ⭐

让我们追踪一条从 **API v2 确认预订到共享业务实现的完整链路：

#### 5.7.1 链路总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    API v2 确认预订调用链路                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. API v2 Controller/Service                                     │
│     apps/api/v2/src/platform/bookings/.../bookings.service.ts      │
│                           │                                          │
│                           ▼                                          │
│  2. 从 @calcom/platform-libraries 导入                               │
│     import { confirmBookingHandler } from "@calcom/platform-libraries"│
│                           │                                          │
│                           ▼                                          │
│  3. platform-libraries 桥接层                                    │
│     packages/platform/libraries/index.ts:90                           │
│     export { confirmHandler as confirmBookingHandler }                │
│                           │                                          │
│                           ▼                                          │
│  4. tRPC Handler 入口                                           │
│     packages/trpc/server/routers/viewer/bookings/confirm.handler.ts  │
│                           │                                          │
│                           ▼                                          │
│  5. 调用共享业务逻辑                                               │
│     packages/features/bookings/lib/handleConfirmation.ts             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 5.7.2 详细步骤

**步骤 1: API v2 调用点**

位置: `apps/api/v2/src/platform/bookings/2024-08-13/services/bookings.service.ts:1-12

```typescript
// 1. 从 platform-libraries 导入
import {
  confirmBookingHandler,
  distributedTracing,
  getAllUserBookings,
  getCalendarLinks,
  getTranslation,
  handleCancelBooking,
  handleMarkNoShow,
  roundRobinManualReassignment,
  roundRobinReassignment,
} from "@calcom/platform-libraries";
```

**步骤 2: API v2 实际调用**

位置: `apps/api/v2/src/platform/bookings/2024-08-13/services/bookings.service.ts:1135-1152`

```typescript
await confirmBookingHandler({
  ctx: {
    user: {
      ...requestUser,
      destinationCalendar: userCalendars?.destinationCalendar ?? null,
    },
    traceContext: distributedTracing.createTrace("api_v2_confirm_booking"),
  },
  input: {
    bookingId: booking.id,
    confirmed: true,
    recurringEventId: booking.recurringEventId ?? undefined,
    emailsEnabled,
    platformClientParams,
    actionSource: "API_V2",
    actor: makeUserActor(requestUser.uuid),
  },
});
```

**步骤 3: Platform Libraries 桥接导出**

位置: `packages/platform/libraries/index.ts:90`

```typescript
// 桥接层将 tRPC handler 重命名导出
export { confirmHandler as confirmBookingHandler } from "@calcom/trpc/server/routers/viewer/bookings/confirm.handler";
```

**步骤 4: tRPC Handler 入口**

位置: `packages/trpc/server/routers/viewer/bookings/confirm.handler.ts:51-60`

```typescript
// 这是 Web 和 API v2 共用的 handler
export const confirmHandler = async ({ ctx, input }: ConfirmOptions) => {
  const log = logger.getSubLogger({ prefix: ["confirmHandler"] });
  const {
    bookingId,
    recurringEventId,
    reason: rejectionReason,
    confirmed,
    emailsEnabled,
    platformClientParams,
  } = input;
  // ... 业务逻辑
```

**步骤 5: Handler 调用共享业务**

位置: `packages/trpc/server/routers/viewer/bookings/confirm.handler.ts:9, 364`

```typescript
// 从 features 导入核心业务逻辑
import { handleConfirmation } from "@calcom/features/bookings/lib/handleConfirmation";

// 在 handler 内部调用
await handleConfirmation({
  user: { ...user, credentials: allCredentials },
  evt,
  recurringEventId,
  // ...
});
```

**步骤 6: 最终业务实现**

位置: `packages/features/bookings/lib/handleConfirmation.ts:23-60`

```typescript
// 这是真正的业务逻辑实现
export async function handleConfirmation(args: {
  user: EventManagerUser & { username: string | null };
  evt: CalendarEvent;
  recurringEventId?: string;
  prisma: PrismaClient;
  bookingId: number;
  // ...
}) {
  // 发送邮件、触发 webhook、更新状态等核心业务
}
```

#### 5.7.3 链路总结

| 层级 | 文件位置 | 职责 |
|------|---------|------|
| **API 层** | `apps/api/v2/.../bookings.service.ts` | 接收 HTTP 请求，调用共享 handler |
| **桥接层** | `packages/platform/libraries/index.ts` | 重命名导出，解决 tsconfig 路径限制 |
| **tRPC 层** | `packages/trpc/.../confirm.handler.ts` | 处理 tRPC 特定逻辑（上下文、鉴权） |
| **业务层** | `packages/features/.../handleConfirmation.ts` | 纯业务逻辑，与框架无关 |

**关键设计**: tRPC handler 是一个"薄包装器"，真正的业务逻辑在 `@calcom/features` 中，这样：
1. Web 通过 tRPC 直接调用 handler
2. API v2 通过 platform-libraries 桥接调用同一个 handler
3. 两者最终执行相同的 `handleConfirmation` 业务逻辑

### 5.8 端到端复用链路示例二：预订取消服务 ⭐

这是一条**更完整**的链路，包含：API v2 入口 → 子路径导入 → platform-libraries 桥接 → 共享业务实现 → 数据层访问。

#### 5.8.1 链路总览

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│              API v2 预订取消服务调用链路 (含 Repository 模式)                    │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                   │
│  1. API v2 业务服务                                                          │
│     apps/api/v2/src/lib/services/booking-cancel.service.ts                       │
│     import { BookingCancelService } from "@calcom/platform-libraries/bookings"   │
│                           │                                                       │
│                           ▼                                                       │
│  2. tsconfig 子路径映射 (显式枚举)                                              │
│     apps/api/v2/tsconfig.json:31                                                 │
│     "@calcom/platform-libraries/bookings" → ../../../platform/libraries/bookings.ts│
│                           │                                                       │
│                           ▼                                                       │
│  3. platform-libraries 子模块出口                                              │
│     packages/platform/libraries/bookings.ts:6                                    │
│     export { BookingCancelService } from "@calcom/features/bookings/lib/..."     │
│                           │                                                       │
│                           ▼                                                       │
│  4. 共享业务实现 (Service 层)                                                   │
│     packages/features/bookings/lib/handleCancelBooking.ts:526                   │
│     class BookingCancelService { ... }                                           │
│                           │                                                       │
│                           ▼                                                       │
│  5. 数据访问 (Repository 层)                                                    │
│     packages/features/profile/repositories/ProfileRepository.ts:94              │
│     class ProfileRepository {                                                    │
│       constructor(deps: { prismaClient: PrismaClient })                          │
│       // 使用 prismaClient 进行数据库操作                                        │
│     }                                                                             │
│                                                                                   │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

#### 5.8.2 详细步骤（附证据代码）

**步骤 1: API v2 服务入口**

位置: `apps/api/v2/src/lib/services/booking-cancel.service.ts:1-27`

```typescript
// 1. 从子路径导入，不是从根路径
import { BookingCancelService as BaseBookingCancelService } from "@calcom/platform-libraries/bookings";

// 2. API v2 自己的服务继承共享服务
@Injectable()
export class BookingCancelService extends BaseBookingCancelService {
  constructor(
    userRepository: PrismaUserRepository,
    bookingRepository: PrismaBookingRepository,
    profileRepository: PrismaProfileRepository,  // ⭐ 注入 Repository
    bookingReferenceRepository: PrismaBookingReferenceRepository,
    attendeeRepository: PrismaBookingAttendeeRepository
  ) {
    super({
      userRepository,
      bookingRepository,
      profileRepository,  // ⭐ 传递给父类
      bookingReferenceRepository,
      attendeeRepository,
    });
  }
}
```

**步骤 2: Repository 的子路径导入**

位置: `apps/api/v2/src/lib/repositories/prisma-profile.repository.ts:1-11`

```typescript
// 从子路径导入，不是从根路径
import { PrismaProfileRepository as BasePrismaProfileRepository } from "@calcom/platform-libraries/repositories";

// API v2 的适配器，注入 NestJS 的 Prisma 服务
@Injectable()
export class PrismaProfileRepository extends BasePrismaProfileRepository {
  constructor(private readonly dbWrite: PrismaWriteService) {
    super({ prismaClient: dbWrite.prisma });  // ⭐ 传递 PrismaClient
  }
}
```

**步骤 3: tsconfig 显式子路径映射**

位置: `apps/api/v2/tsconfig.json:31,37`

```json
{
  "compilerOptions": {
    "paths": {
      // ⭐ 显式枚举，不是通配符
      "@calcom/platform-libraries/bookings": ["../../../packages/platform/libraries/bookings.ts"],
      "@calcom/platform-libraries/repositories": ["../../../packages/platform/libraries/repositories.ts"],
      // ... 其他 14 条显式映射
    }
  }
}
```

**步骤 4: platform-libraries/bookings 子模块出口**

位置: `packages/platform/libraries/bookings.ts:6`

```typescript
// 桥接层：从 features 导出共享类
export { BookingCancelService } from "@calcom/features/bookings/lib/handleCancelBooking";
```

**步骤 5: platform-libraries/repositories 子模块出口**

位置: `packages/platform/libraries/repositories.ts:13`

```typescript
// 桥接层：从 features 导出多个 Repository
export { ProfileRepository as PrismaProfileRepository } from "@calcom/features/profile/repositories/ProfileRepository";
export { BookingRepository as PrismaBookingRepository } from "@calcom/features/bookings/repositories/BookingRepository";
export { UserRepository as PrismaUserRepository } from "@calcom/features/users/repositories/UserRepository";
// ... 更多 Repository
```

**步骤 6: 共享业务实现 (Service 层)**

位置: `packages/features/bookings/lib/handleCancelBooking.ts:522-537`

```typescript
/**
 * Takes care of cancelling bookings. This includes regular bookings, 
 * recurring bookings, seated bookings, etc.
 */
export class BookingCancelService implements IBookingCancelService {
  constructor(private readonly deps: BookingCancelServiceDependencies) {}

  async cancelBooking(input: { bookingData: CancelRegularBookingData; bookingMeta?: CancelBookingMeta }) {
    const cancelBookingInput: CancelBookingInput = {
      bookingData: input.bookingData,
      ...(input.bookingMeta || {}),
    };

    return handler(cancelBookingInput, this.deps);  // 调用共享 handler
  }
}
```

**步骤 7: 数据层访问 (Repository 层)**

位置: `packages/features/profile/repositories/ProfileRepository.ts:94-99, 193-218`

```typescript
export class ProfileRepository implements IProfileRepository {
  private prismaClient: PrismaClient;

  constructor(deps: { prismaClient: PrismaClient }) {
    this.prismaClient = deps.prismaClient;  // ⭐ 依赖注入 PrismaClient
  }

  // 使用注入的 prismaClient 进行数据库操作
  private static async _create({ userId, organizationId, ... }: {...}) {
    return prisma.profile.create({  // ⭐ 实际的数据层访问
      data: {
        uid: ProfileRepository.generateProfileUid(),
        user: { connect: { id: userId } },
        organization: { connect: { id: organizationId } },
        username: username || email.split("@")[0],
      },
    });
  }
}
```

#### 5.8.3 链路设计模式总结

| 层级 | 模式 | 证据位置 |
|------|------|---------|
| **API 入口** | 继承 + 依赖注入 | `BookingCancelService extends BaseBookingCancelService` |
| **路径约束** | 显式子路径枚举 (16 条) | `tsconfig.json:24-37` 非通配符映射 |
| **桥接层** | Facade 模式 + 子模块出口 | `packages/platform/libraries/bookings.ts` |
| **业务层** | Service 模式 | `BookingCancelService` 类 |
| **数据层** | Repository 模式 + 依赖注入 | `ProfileRepository` 接收 `prismaClient` |

**关键发现**: 这条链路展示了一个**更完整**的架构模式：
1. 不是简单的函数导出，而是 **Service + Repository 类** 的复用
2. API v2 通过**继承**共享类，并注入 NestJS 特有的 Prisma 服务
3. 数据层通过**依赖注入**实现框架无关性（features 只定义接口，API v2 注入具体实现）
4. 子路径映射确保 API v2 只能访问预先声明的 16 个模块

---

## 6. 应用层入口

### 6.1 Web 应用 (`@calcom/web`)

**位置**: `apps/web/`

**技术栈**: Next.js 16 + tRPC + React Query

**依赖的共享包** (apps/web/package.json:35-48):
```json
"dependencies": {
  "@calcom/app-store": "workspace:*",
  "@calcom/features": "workspace:*",
  "@calcom/lib": "workspace:*",
  "@calcom/prisma": "workspace:*",
  "@calcom/trpc": "workspace:*",
  "@calcom/ui": "workspace:*",
  ...
}
```

**启动方式**:
```bash
yarn dev           # 开发模式
yarn build         # 生产构建
yarn start         # 生产启动
```

### 6.2 API v2 (`@calcom/api-v2`)

**位置**: `apps/api/v2/`

**技术栈**: NestJS + tRPC 业务逻辑复用

**依赖策略** (apps/api/v2/package.json:42-49):
```json
"dependencies": {
  "@calcom/platform-constants": "workspace:*",
  "@calcom/platform-enums": "workspace:*",
  "@calcom/platform-libraries": "workspace:*",  // ⭐ 核心桥接
  "@calcom/platform-types": "workspace:*",
  "@calcom/platform-utils": "workspace:*",
  "@calcom/prisma": "workspace:*",
  ...
}
```

**关键设计**: API v2 不直接依赖 `@calcom/features`，而是通过 `@calcom/platform-libraries` 间接获取业务能力。

---

## 7. 依赖关系图 (简化版)

```
@calcom/web (Next.js)
├── @calcom/app-store
├── @calcom/features
├── @calcom/trpc
│   └── @calcom/features
│   └── @calcom/lib
│   └── @calcom/prisma
├── @calcom/ui
│   └── @calcom/lib
└── @calcom/lib
    └── @calcom/prisma

@calcom/api-v2 (NestJS)
├── @calcom/platform-libraries ⭐ (桥接)
│   └── @calcom/features
│   └── @calcom/trpc
│   └── @calcom/lib
│   └── @calcom/prisma
├── @calcom/platform-types
├── @calcom/platform-enums
├── @calcom/platform-constants
└── @calcom/prisma
```

---

## 8. 总结

### 8.1 架构亮点

1. **清晰的分层**: 应用层 → 平台桥接层 → 业务层 → 基础设施层
2. **平台桥接模式**: `@calcom/platform-libraries` 解决了不同构建系统间的模块路径问题
3. **Repository + Service 模式**: `@calcom/features` 中的业务逻辑易于测试和复用
4. **tRPC 统一 API 层**: Web 和 API v2 共享核心业务逻辑
5. **按需导出**: UI 组件使用子路径导出，避免 barrel imports

### 8.2 关键复用机制

| 复用场景 | 机制 | 关键包/文件 |
|---------|------|-------------|
| 业务逻辑 | Repository + Service + platform-libraries 桥接 | `packages/features/`, `packages/platform/libraries/index.ts` |
| API 路由 | tRPC handler 直接导出复用 | `packages/trpc/server/routers/` |
| UI 组件 | 子路径导出 + workspace 依赖 | `packages/ui/` |
| 数据库查询 | Prisma Select 复用 | `packages/prisma/selects/` |
| 工具函数 | `@calcom/lib` 直接导入 | `packages/lib/` |

### 8.3 构建优化

- **Turborepo 缓存**: 基于输入的智能缓存
- **并行构建**: 依赖图驱动的并行任务执行
- **按需构建**: `--filter` 支持只构建特定应用及其依赖
- **环境变量感知**: 任务级别的 env 依赖追踪

---

*报告生成时间: 2026-05-10*
