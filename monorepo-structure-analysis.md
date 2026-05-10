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
