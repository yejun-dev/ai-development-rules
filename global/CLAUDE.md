# 全局开发规范（用户级）

> 用途：复制到 `~/.claude/CLAUDE.md`（Windows：`C:\Users\<用户名>\.claude\CLAUDE.md`），对本机所有项目、所有会话生效。
> 与项目内 CLAUDE.md 的关系：项目文件更具体，冲突时以项目文件为准；本文件兜底。

## 适用技术栈

- 前端：Vue 3 + TypeScript(strict) + Tailwind CSS + shadcn-vue + TanStack Table + Zod + Pinia + Vite
- 后端：.NET 8 四层架构（`Api → Application → Infrastructure → Domain`）+ EF Core (MySQL) + JWT 双 token + Redis
- 接口：RESTful，URL 统一 `/api/v1/[controller]`；响应统一 `ApiResult<T>`（`code=0` 成功）

## 工作准则

1. **先确认再动手**：引入新第三方依赖、架构选型/目录结构变更、删除或重写已有文件、数据库表结构变更、破坏性接口变更——必须先征得同意。
2. **复用优先**：优先复用已有封装，禁止重复造轮子。
3. **完整可运行**：交付前用构建/测试/格式命令验证，不静默跳过失败项。
4. **测试随行**：新增/修改业务逻辑必须附带测试并本地跑通。
5. **最小改动**：不修改无关代码结构，不做规范未要求的"顺手重构"。

## 核心红线（所有项目通用）

- 前端必须 Composition API + `<script setup lang="ts">`；禁止 `any`，类型显式声明。
- shadcn-vue 组件目录（`src/components/ui/`）**只读，禁止修改源码**；业务定制用外层包裹/props/插槽/Tailwind 覆盖。
- 列表页必须 TanStack Table；表单必须 Zod Schema；所有请求走全局 request 封装，禁止组件内直连 axios。
- 样式只用 Tailwind 原子类，禁止新增独立 CSS、禁止硬编码色值。
- 后端严格分层，禁止跨层调用、禁止 Controller 直接操作 DbContext。
- 控制器返回业务对象由全局 Filter 包装，禁止手动构造 ApiResult、禁止 try-catch 返回错误。
- EF Core：只读查询 `AsNoTracking()`；列表 `Select()` 投影 DTO；显式 `Include()` 禁止懒加载；字符串字段显式 `HasMaxLength()`。
- 入参一律 DTO；构造函数注入；异步方法后缀 `Async`。
- 表结构变更必须走 EF 迁移；逻辑删除用 `IsDeleted`，禁止物理删除。
- 禁止 SQL 拼接、禁止明文密码（BCrypt）、禁止日志输出密码/Token/密钥；CORS 白名单显式配置，禁止 `AllowAnyOrigin`。
- 缓存必须设过期时间，禁止永久 key、禁止缓存敏感信息明文。
- 所有 views 路由懒加载；列表必须分页，禁止一次性全量查询。

## 验证命令（项目无自有配置时参考）

- 前端：`npm run dev` / `npm run build` / `npm run lint` / `npm run format` / `npm run test:unit`
- 后端：`dotnet build` / `dotnet test` / `dotnet format`
- 项目内有自己的 CLAUDE.md 时，以项目的命令为准。
