# Test Report — CLAUDE.md for Next.js + SQLite

## 测试方法

1. 使用 `create-next-app` 创建全新项目
2. 将 CLAUDE.md 粘贴到项目根目录
3. 向 Claude Code 提出典型开发任务
4. 验证 Claude Code 无需澄清问题即可正确执行

## 测试场景

### 场景 1：创建用户表

**Prompt:** "创建一个用户表，包含邮箱、姓名、创建时间"

**预期行为：**
- 使用 `sqliteTable` 定义表
- 主键用 `text` + `nanoid()`
- 时间用 `integer` + `timestamp` 模式
- 自动生成迁移文件

**结果：** ✅ Claude Code 直接按规范执行，未询问任何澄清问题

### 场景 2：创建注册表单

**Prompt:** "创建一个注册表单，包含姓名和邮箱字段"

**预期行为：**
- 使用 Server Action 处理表单提交
- 服务端验证输入
- 返回 `{ error } | { success }` 结构
- Client Component 使用 `'use client'`

**结果：** ✅ Claude Code 按规范执行，自动使用 Server Actions

### 场景 3：查询用户列表

**Prompt:** "查询所有用户并显示在仪表盘页面"

**预期行为：**
- 在 Server Component 中直接查询数据库
- 使用 Drizzle 查询构建器（非手写 SQL）
- 通过 props 传给 Client Component

**结果：** ✅ Claude Code 按规范执行，使用 Server Component + Drizzle

### 场景 4：数据库迁移

**Prompt:** "给用户表添加一个 avatar_url 字段"

**预期行为：**
- 修改 schema.ts 添加字段
- 运行 `pnpm drizzle-kit generate` 生成迁移
- 运行 `pnpm drizzle-kit migrate` 执行迁移
- 不手动编辑迁移文件

**结果：** ✅ Claude Code 按规范执行，自动生成迁移

## 覆盖的验收标准

- [x] 项目结构、命名规范、数据库迁移规则
- [x] 开发命令、遵循的模式、避免的反模式
- [x] 有主见——每条规则都有理由
- [x] 可在全新 Next.js + SQLite 项目中直接使用
- [x] 已在真实项目上测试
