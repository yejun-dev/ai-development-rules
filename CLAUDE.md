# CLAUDE.md — AI 编码助手工作规范（精简版）

> 本文件是 AI 编码助手的永久工作规范，每次会话自动加载。
> 完整技术规范见 [docs/development-spec.md](docs/development-spec.md)：本文件只保留每次工作都必须遵守的核心约束。两者冲突时，以全量规范为准。

## 0. 工作准则（优先级从高到低）

1. **规范优先**：严格遵守本文件与全量规范，冲突时序号小者优先。
2. **复用优先**：优先复用已有封装（`utils/request`、`stores/`、`components/business/`、后端公共服务与仓储），禁止重复造轮子。
3. **先确认再动手**：命中「四、必须确认的场景」必须先征得同意，其余场景直接交付。
4. **完整可运行**：输出代码必须完整、可直接运行；交付前必须用「一、常用命令」中的命令验证（构建/测试/格式检查），不静默跳过失败项。
5. **测试随行**：新增/修改业务逻辑必须附带测试，且本地跑通后才算完成。
6. **最小改动**：不修改无关代码结构，不做规范未要求的"顺手重构"。
7. **注释适度**：关键业务逻辑必须注释，不写冗余注释。

## 一、常用命令

### 前端（目录名以实际仓库为准，如 `frontend/`）

| 用途 | 命令 |
| --- | --- |
| 安装依赖 | `npm ci`（按 lockfile，免确认）/ `npm install`（变更依赖，需确认） |
| 启动开发 | `npm run dev` |
| 构建 | `npm run build` |
| 代码检查 | `npm run lint` / `npm run format`（ESLint + Prettier） |
| 单元测试 | `npm run test:unit`（Vitest） |
| E2E 测试 | `npx playwright test` |
| 新增 shadcn 组件 | `npx shadcn-vue@2.x add <组件名>`（固定主版本，禁止 @latest；禁止手动复制源码） |

### 后端（解决方案名以实际仓库为准）

| 用途 | 命令 |
| --- | --- |
| 还原依赖 | `dotnet restore` |
| 构建 | `dotnet build` |
| 启动接口 | `dotnet run --project src/YourProject.Api` |
| 单元/集成测试 | `dotnet test` |
| 新增迁移 | `dotnet ef migrations add <Name> --project src/YourProject.Infrastructure --startup-project src/YourProject.Api` |
| 格式检查 | `dotnet format`（**进到各项目目录**执行） |

> 命令跑不通时先排查环境原因并反馈，禁止静默跳过验证。集成测试（Testcontainers 需本地 Docker；或 `WebApplicationFactory<Program>` 直连开发态容器）详见全量规范 7.1。
> ⚠️ `dotnet format` **不支持 `.slnx` 工作区**：在解决方案根目录执行会打印 help 并**以退出码 0 结束**——静默通过，CI 会绿着放过未格式化的代码。进到各项目目录执行 `dotnet format whitespace --verify-no-changes`（该子命令不接受 `--nologo`，会被当成文件路径）。
> `dotnet ef` 工具版本需与 EF Core 主版本匹配（版本不匹配会报错；升级：`dotnet tool update --global dotnet-ef`）。
> Git 提交信息遵循 Conventional Commits：`feat|fix|docs|style|refactor|test|chore(scope): 描述`（细则见全量规范 11.2）。

## 二、架构速览

- 前端：Vue 3 + TypeScript(strict) + Tailwind CSS + shadcn-vue + TanStack Table + Zod + Pinia + Vite，目录结构见全量规范 3.1。
- 后端：.NET 10 四层架构 `Api → Application → Infrastructure → Domain`，EF Core (MySQL) + JWT 双 token + Redis，分层原则见全量规范 4.1。
- 接口：RESTful，URL 统一 `/api/v1/[controller]` 小写；响应统一 `ApiResult<T>`（`code=0` 成功）。
- 关键封装（优先复用）：前端 `utils/request` 请求实例、`api/` 接口定义、`stores/` 状态、`components/business/` 业务组件；后端 `ApiResult` 与全局 Filter、`BusinessException`、`PageQuery` 分页基类、公共审计/缓存服务。

## 三、核心红线（每次输出必须满足）

### 前端

- 必须 Composition API + `<script setup lang="ts">`；禁止选项式 API；禁止 `any`，所有变量/函数/参数显式声明类型。
- 业务实体类型放 `types/`，字段与后端 DTO 完全一致；表单必须 Zod Schema + vee-validate（`@vee-validate/zod`），禁止手写 if-else 校验。
- 列表页必须用 TanStack Table（`ColumnDef<T>` 泛型），禁止手写表格渲染/排序/分页逻辑。
- 所有请求必须走 `utils/request` 封装实例，接口函数统一定义在 `api/`；禁止组件内直接调 axios。
- `components/ui/` 为 shadcn-vue 源码，**只读，禁止修改**；业务定制用外层包裹/props/插槽/Tailwind 类名覆盖。
- 样式只用 Tailwind 原子类；主题色/间距/圆角/阴影用 CSS `@theme` 配置（Tailwind 4），禁止新增独立 CSS、禁止硬编码色值。
- 权限：页面级用路由 meta `permission: '模块:操作'`，按钮级用 `v-auth` 指令（**动态渲染的按钮例外**：`v-auth` 靠替换 DOM 节点实现，TanStack Table 行内按钮每次翻页/排序都会重建，用它必抛 `NotFoundError`，改走条件渲染）；数据权限（DataScope）只能由后端过滤，禁止前端控制。
- 所有 views 路由懒加载；列表必须分页，禁止一次性全量查询。
- 必须配置前端错误边界（`app.config.errorHandler` + `onErrorCaptured` 降级 UI），禁止未捕获异常导致白屏。

### 后端

- 严格分层，禁止跨层调用；禁止控制器直接操作 DbContext。
- 控制器直接返回业务对象，由全局 ResultFilter 包装为 `ApiResult<T>`；禁止手动构造 ApiResult、禁止在控制器 try-catch 返回错误结果。
- 业务校验失败抛 `BusinessException`（带错误码），由全局 ExceptionFilter 统一转换；系统异常对外只返回"系统内部错误"。
- 入参一律用 DTO（分页参数继承 `PageQuery`）；禁止 Entity 作为接口参数。
- EF Core：只读查询 `AsNoTracking()`；列表查询 `Select()` 投影 DTO，禁止返回 Entity；关联显式 `Include()`，禁止懒加载；字符串字段显式 `HasMaxLength()`；高频筛选/排序字段在 `OnModelCreating` 配置索引；事务在业务层，禁止控制器中使用。
- 逻辑删除的三个连带坑：唯一性校验必须 `IgnoreQueryFilters()`（唯一索引建在物理列上，软删除不释放唯一性，否则校验放行、数据库报重复键）；禁止无脑 `Update(entity)`（会把全部字段标脏，审计退化成"所有字段都变了"）；`ExecuteUpdateAsync`/`ExecuteDeleteAsync` **绕过变更跟踪与审计拦截器**，必须显式补写审计日志。
- 构造函数注入，禁止手动 new 服务；注册生命周期按规范（业务服务/仓储 Scoped，工具类/全局配置 Singleton）；禁止静态类存储状态。
- 异步方法后缀 `Async`；公共方法必须有 `<summary>` 文档注释。

### 接口与数据

- 接口 URL 带版本号；破坏性变更必须升 `v2`，旧版本保留至少一个大版本周期并标注废弃。
- 认证 `Authorization: Bearer {token}`；时间用 ISO 8601 字符串；POST 提交类接口带 `Idempotency-Key` 请求头（UUID）。
- 错误码按全量规范 6.4 处理：`code=0` 成功，`≥1000` 业务异常；新增业务错误码必须先查阅并登记 [docs/error-codes.md](docs/error-codes.md)。
- 表结构变更必须走 EF Core 迁移并随代码提交；禁止直接改库；已提交迁移文件禁止修改，只能新增迁移。
- 所有业务表必含 `Id`（int 自增，审计日志等高速增长表用 bigint）、`CreateTime`、`UpdateTime`；重要业务表逻辑删除用 `IsDeleted`，禁止物理删除。
- utf8mb4 / InnoDB；枚举存 int；禁止存储过程、触发器。

### 安全

- 禁止 SQL 字符串拼接（EF Core 参数化）；密码 BCrypt 哈希，禁止明文存储和传输。
- 禁止 `v-html` 渲染服务端/用户输入；富文本必须先过白名单净化库（DOMPurify）。
- 禁止日志输出密码、Token、密钥；敏感配置走 `dotnet user-secrets` / 环境变量，禁止入库入仓。
- CORS 白名单显式配置，禁止 `AllowAnyOrigin`；统一安全响应头。
- 文件上传：类型白名单 + 大小限制 + 文件头魔数校验 + UUID 随机文件名，禁止保留原始文件名落盘。
- 敏感字段（手机号/邮箱/身份证）后端按权限脱敏后返回，前端只负责展示。
- 缓存 key 按 `{项目}:{模块}:{实体}:{id}` 命名；必须设过期时间，禁止永久 key、禁止缓存敏感信息明文。

## 四、必须确认的场景（未征得同意前禁止动手）

- 引入任何全量规范未提及的第三方依赖
- 架构选型、新增分层、修改目录结构
- 删除或重写已有文件、公共封装
- 数据库表结构变更（含索引调整）
- 破坏性接口变更（涉及版本升级）
- 后端 DTO / 接口数据结构变更（需同步前端 types）

## 五、交付前自查清单

逐项校验，不满足则补齐后再交付：

- [ ] 命名与格式符合规范（前端 2 空格缩进、后端 4 空格；命名语义化，禁止拼音/无意义缩写）
- [ ] 复用了已有封装（request、store、通用组件、公共服务），无重复实现
- [ ] 类型完整，无 `any`
- [ ] 接口约定合规（版本、分页、幂等、错误码、ISO 8601 时间）
- [ ] 无安全隐患（明文密码、SQL 拼接、敏感信息泄露、CORS 放开）
- [ ] 无未约定的第三方依赖
- [ ] 数据库变更有迁移文件
- [ ] 后端 DTO 变更已同步前端 types（字段/类型一致）
- [ ] 新增业务错误码已登记 docs/error-codes.md
- [ ] 新逻辑附带测试，且已本地跑通
- [ ] 关键操作有审计日志
- [ ] 满足性能预算（路由懒加载、列表分页、无全量查询）

## 六、全量规范

详细规则（完整目录结构、组件/表格/表单细则、状态管理判定、token 无感刷新细节、错误码表与注册表（docs/error-codes.md）、缓存/审计细则、Git 协作与 CI/CD、性能预算、命名对照表）见 [docs/development-spec.md](docs/development-spec.md)。

> 文件组织：本文件放在仓库根目录。前后端分目录（如 `frontend/`、`backend/`）时，可在各目录放置更细的 CLAUDE.md，Claude Code 会按工作目录自动加载。
>
> 权限配置：仓库根目录 `.claude/settings.json` 已将「四、必须确认的场景」固化为权限规则（allow/ask/deny 三组，说明见 `.claude/README.md`），日常验证命令免确认、红线操作不可执行。
