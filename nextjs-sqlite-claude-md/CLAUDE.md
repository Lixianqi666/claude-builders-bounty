# CLAUDE.md — Next.js 15 + SQLite SaaS

> 生产级 SaaS 项目规范。每次改动前必读。

## Stack & Versions

| 层 | 选型 | 版本 | 为什么 |
|---|------|------|--------|
| 框架 | Next.js App Router | 15.x | RSC + 流式 SSR，SaaS 首选 |
| 语言 | TypeScript | 5.x strict | 类型安全，减少运行时错误 |
| 数据库 | better-sqlite3 | 11.x | 同步 API，零延迟，嵌入式 |
| ORM | Drizzle ORM | 0.3x | 类型安全 SQL，SQLite 原生支持 |
| 样式 | Tailwind CSS | 4.x | 原子化，无运行时开销 |
| 包管理 | pnpm | 9.x | 快，省磁盘，严格依赖隔离 |
| 运行时 | Node.js | 20 LTS | 稳定，长期支持 |

**不使用：** Prisma（SQLite 支持弱）、MongoDB（SaaS 不适合）、CSS-in-JS（运行时开销）、Redux（过度工程化）。

## Folder Structure

```
src/
├── app/                    # Next.js App Router
│   ├── (auth)/             # 认证相关路由组
│   │   ├── login/
│   │   └── register/
│   ├── (dashboard)/        # 登录后主界面
│   │   ├── layout.tsx      # 侧边栏 + 顶栏
│   │   ├── page.tsx        # 首页仪表盘
│   │   └── settings/
│   ├── api/                # API 路由
│   │   └── v1/             # 版本化 API
│   ├── layout.tsx          # 根布局
│   └── globals.css
├── components/
│   ├── ui/                 # 基础 UI 组件（无业务逻辑）
│   ├── forms/              # 表单组件
│   └── layouts/            # 布局组件
├── db/
│   ├── schema.ts           # Drizzle 表定义
│   ├── migrations/         # 自动生成的迁移文件
│   └── index.ts            # 数据库连接实例
├── lib/
│   ├── auth.ts             # 认证逻辑
│   ├── email.ts            # 邮件发送
│   └── utils.ts            # 纯工具函数
├── hooks/                  # 自定义 React hooks
├── types/                  # 全局类型定义
└── constants.ts            # 常量
```

**规则：**
- `app/` 下每个路由文件不超过 200 行，超出则拆分到 `components/`
- `components/ui/` 不得导入 `db/` 或任何业务模块
- `lib/` 下每个文件单一职责，不放"杂项工具箱"

## Naming Conventions

| 类型 | 规范 | 示例 |
|------|------|------|
| 文件名 | kebab-case | `user-profile.tsx` |
| 组件名 | PascalCase | `UserProfile` |
| 函数名 | camelCase | `getUserById` |
| 常量 | UPPER_SNAKE | `MAX_FILE_SIZE` |
| 数据库表 | snake_case | `user_accounts` |
| 数据库列 | snake_case | `created_at` |
| CSS 类 | Tailwind 原子类 | `flex items-center` |
| 路由路径 | kebab-case | `/dashboard/user-settings` |

**TypeScript 命名：**
- 接口不加 `I` 前缀：`User` 而非 `IUser`
- 类型用 `type`，仅在需要继承时用 `interface`
- 枚举用 `as const` 对象代替 `enum`

## Database / SQLite Rules

### Schema 定义

```typescript
// src/db/schema.ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core'

export const users = sqliteTable('users', {
  id: text('id').primaryKey(),           // nanoid，非自增
  email: text('email').notNull().unique(),
  name: text('name').notNull(),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).notNull(),
})
```

**规则：**
- 主键用 `text` + `nanoid()`，不用自增 ID（SaaS 需要不可预测的 ID）
- 时间用 `integer` + `timestamp` 模式，存 Unix 毫秒
- 所有表必须有 `createdAt` 和 `updatedAt`
- 外键用 `.references(() => table.id)`，不手写 SQL
- 布尔值用 `integer`（0/1），SQLite 没有原生 boolean

### Migrations

```bash
# 生成迁移
pnpm drizzle-kit generate

# 执行迁移
pnpm drizzle-kit migrate

# 查看差异
pnpm drizzle-kit diff
```

**规则：**
- 迁移文件自动生成，不手动编辑
- 迁移文件必须提交到 git
- 生产环境只执行 `migrate`，不执行 `generate`
- 破坏性变更（删列、改类型）需要两步迁移：先加新列 → 迁移数据 → 再删旧列

### 查询模式

```typescript
// ✅ 正确：类型安全的 Drizzle 查询
const user = await db.select().from(users).where(eq(users.id, userId)).get()

// ❌ 错误：手写 SQL 字符串
const user = await db.get(`SELECT * FROM users WHERE id = ?`, [userId])

// ✅ 复杂查询用 sql 模板
const result = await db.select({
  count: sql<number>`count(*)`,
}).from(users).where(sql`${users.createdAt} > ${oneWeekAgo}`)
```

## Component Patterns

### Server Component（默认）

```typescript
// app/(dashboard)/page.tsx
import { db } from '@/db'
import { users } from '@/db/schema'

export default async function DashboardPage() {
  const userList = await db.select().from(users).all()
  return <UserList users={userList} />
}
```

**规则：**
- 默认用 Server Component，只在需要交互时加 `'use client'`
- 数据获取在 Server Component 中完成，通过 props 传给 Client Component
- 不在 Server Component 中用 `useState`、`useEffect`

### Client Component

```typescript
'use client'

import { useState } from 'react'

interface UserListProps {
  users: { id: string; name: string }[]
}

export function UserList({ users }: UserListProps) {
  const [filter, setFilter] = useState('')
  // ...
}
```

**规则：**
- `'use client'` 必须在文件顶部
- Props 类型定义在同文件内，不单独建 types 文件
- 组件超过 150 行时拆分逻辑到 custom hook

### Forms

```typescript
// 使用 Server Actions
'use server'

import { db } from '@/db'
import { users } from '@/db/schema'

export async function createUser(formData: FormData) {
  const name = formData.get('name') as string
  const email = formData.get('email') as string

  if (!name || !email) {
    return { error: 'Name and email are required' }
  }

  await db.insert(users).values({
    id: nanoid(),
    name,
    email,
    createdAt: new Date(),
    updatedAt: new Date(),
  })

  return { success: true }
}
```

**规则：**
- 表单用 Server Actions，不用 API Route
- 验证在 Server Action 中完成，不信任客户端
- 返回 `{ error: string } | { success: true }`，不用 try/catch 包裹业务逻辑

## Dev Commands

```bash
# 开发
pnpm dev                    # 启动开发服务器（http://localhost:3000）
pnpm build                  # 生产构建
pnpm start                  # 启动生产服务器

# 数据库
pnpm db:generate            # 生成迁移
pnpm db:migrate             # 执行迁移
pnpm db:studio              # 打开 Drizzle Studio（数据查看器）
pnpm db:seed                # 填充测试数据

# 代码质量
pnpm lint                   # ESLint 检查
pnpm typecheck              # TypeScript 类型检查
pnpm test                   # 运行测试
pnpm test:watch             # 监听模式测试
```

## Anti-Patterns（禁止事项）

| ❌ 不要这样做 | ✅ 应该这样做 | 原因 |
|--------------|-------------|------|
| `useEffect` 获取数据 | Server Component 直接查询 | 避免瀑布流请求，提升首屏速度 |
| `any` 类型 | `unknown` + 类型守卫 | `any` 绕过类型检查，运行时崩溃 |
| 巨型 `utils.ts` | 按职责拆分 `lib/` 文件 | 可维护性，tree-shaking |
| CSS 模块 + Tailwind 混用 | 纯 Tailwind | 避免样式冲突，统一维护 |
| 客户端状态管理库 | React Context + `useState` | SaaS 规模不需要 Redux/Zustand |
| 手写 SQL 字符串 | Drizzle 查询构建器 | 类型安全，防 SQL 注入 |
| 环境变量硬编码 | `process.env` + `.env.local` | 安全性，环境隔离 |
| 在组件中直接调 DB | Server Action / API Route | 关注点分离 |
| 自增 ID | `nanoid()` 生成的字符串 ID | 不可预测，URL 友好，分布式安全 |
| `console.log` 调试 | 结构化日志（pino） | 生产环境可观测性 |

## What We Don't Do（和为什么）

- **不用 GraphQL**：REST + Server Actions 足够，GraphQL 增加复杂度但无收益
- **不用微服务**：单体应用更适合 SaaS 早期，部署简单，调试容易
- **不用 SSR 缓存**：SQLite 是本地的，查询足够快，不需要 HTTP 缓存层
- **不用 OAuth 库**：自己实现邮箱+密码认证，SaaS 早期不需要社交登录
- **不用 Docker 开发**：SQLite 无需容器化，直接本地运行
- **不用 CI/CD 之前不写测试**：先写业务代码，稳定后再补测试
- **不用 i18n**：除非明确需要多语言，否则不引入翻译层

## Git 规范

```
feat: 新功能
fix: 修复
refactor: 重构（不改变功能）
style: 样式调整
docs: 文档
chore: 构建/工具
```

**规则：**
- 每个 commit 做一件事
- 不提交 `node_modules/`、`.next/`、`*.db` 文件
- PR 标题格式：`feat: 简短描述`

## 环境变量

```bash
# .env.local（不提交到 git）
DATABASE_URL=file:./data/app.db
AUTH_SECRET=随机生成的32位字符串
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

**规则：**
- 客户端可访问的变量必须 `NEXT_PUBLIC_` 前缀
- 秘钥只在服务端使用，不加 `NEXT_PUBLIC_`
- `.env.local` 在 `.gitignore` 中
