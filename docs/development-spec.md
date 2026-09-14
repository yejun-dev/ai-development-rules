# 项目全局开发规范（全量版）

> **用途**：本文件是全量开发规范，供人员阅读与评审参考。AI 编码助手每次会话自动加载的是精简版 [CLAUDE.md](../CLAUDE.md)——它只含核心红线与工作准则，详细规则以本文件为准；两者冲突时，以本文件为准。
> **生效原则**：本规范优先级高于通用最佳实践，冲突时以本文件为准。
> **维护**：由技术 Lead 牵头，每季度 review 一次；修改必须走 MR 并同步通知全组。**修改后必须检查 CLAUDE.md「三、核心红线」与「四、必须确认的场景」是否需同步更新。**

---

## 一、项目概述

| 项 目 | 说 明 |
| --- | --- |
| 架构模式 | 前后端完全分离，RESTful API 交互 |
| 核心场景 | 用户管理、权限控制、数据查询、业务表单、报表统计 |
| 开发模式 | AI 辅助编码 + 人工代码评审 + CI 强制校验 |

---

## 二、技术栈与版本要求

### 2.1 前端

| 技术 | 版本 | 强制要求 |
| --- | --- | --- |
| Vue | 3.5+ | 必须使用 Composition API + `<script setup lang="ts">`，禁止选项式 API |
| TypeScript | 5.0+ | strict 模式；禁止 `any`，所有变量/函数/参数显式声明类型 |
| Tailwind CSS | 4.x | 优先原子类，禁止新增独立 CSS 文件；主题配置用 CSS `@theme` |
| shadcn-vue | 最新稳定版 | 组件源码位于 `src/components/ui`，**禁止直接修改源码** |
| TanStack Vue Table | 8.x | 所有列表页必须使用，禁止手写原生表格逻辑 |
| Zod + vee-validate（@vee-validate/zod） | 最新版 | 所有表单必须定义 Schema，与后端 DTO 校验规则对齐 |
| Axios | 1.x | 必须使用全局封装实例，禁止直接调用原生 axios |
| Pinia | 4.x | 管理用户信息、权限、全局配置 |
| Vue Router | 4.x | 全局前置守卫统一做登录校验、权限校验 |
| Vite | 8.x | 配置路径别名、自动导入、接口代理 |
| Node.js | 24 LTS | 本地与 Docker 构建统一版本（Vite 8 要求 ≥20.19 / ≥22.12） |
| ESLint + Prettier | 10.x / 3.x | 统一代码风格与静态检查（CI lint 步骤）；配置随仓库提交 |

### 2.2 后端

| 技术 | 版本 | 强制要求 |
| --- | --- | --- |
| ASP.NET Core Web API | .NET 10.0 LTS | 使用 `[ApiController]`，顶层路由统一前缀 |
| EF Core | 10.0 | Code First，迁移管理表结构 |
| Pomelo.EntityFrameworkCore.MySql | 9.0（过渡方案） | 官方尚无 EF Core 10 适配版（最新 9.0.0 仅支持 EF Core 9），实际项目采用 9.0 + EF Core 10 组合，需项目 NoWarn 抑制 NuGet 警告（实测可用）；待官方适配发布后升级，或改用 Microting 分支（10.x，API 兼容仅换命名空间）；禁止随意跨大版本混用 |
| JWT Bearer | 框架内置 | 接口默认需要认证，公开接口显式标记 `[AllowAnonymous]` |
| FluentValidation | 最新版 | 后端 DTO 入参校验（见 8.2） |
| Serilog | 最新版 | 结构化日志：请求链路、异常堆栈、业务日志 |
| StackExchange.Redis | 最新版 | 分布式缓存（见第九章） |
| Asp.Versioning | 最新版 | URL 版本控制（见 6.1） |

测试类库（xUnit / FluentAssertions / NSubstitute / Testcontainers）见 7.1；限流见 8.6。

### 2.3 数据库与部署

MySQL 8.4 LTS / Docker 24+ / Docker Compose v2+ / Nginx alpine 稳定版

### 2.4 新增依赖原则

不引入本规范未提及的第三方依赖；确需新增必须先说明理由并征得同意。

---

## 三、前端工程规范

### 3.1 标准目录结构

```
src/
├── api/                  # API 接口定义，按业务模块分文件
├── assets/               # 静态资源
├── components/
│   ├── ui/               # shadcn-vue 原生组件【只读，禁止修改】
│   └── business/         # 业务封装组件
├── composables/          # 组合式函数（useXxx）
├── directives/           # 自定义指令（v-auth 权限指令等）
├── layouts/              # 全局布局组件
├── router/index.ts       # 路由配置 + 全局权限守卫
├── stores/               # Pinia 状态管理，按领域分文件
├── types/                # TS 类型定义，与后端 DTO 一一对应
├── utils/request.ts      # Axios 全局封装实例
└── views/                # 页面组件，按业务模块分子目录
```

### 3.2 组件规则

**ui 与 business 的边界判定**：
- `components/ui` = 通用、无业务语义的展示控件（Button、Dialog、Select…），来自 shadcn-vue，**只读**
- `components/business` = 至少依赖一个业务实体类型或业务 Store 的复合控件（如 UserSelectDialog、DeptTreeSelect）
- 业务定制 ui 组件：通过外层包裹、props 传递、插槽、Tailwind 类名覆盖实现，**禁止改源码**
- 新增 ui 组件必须通过 `npx shadcn-vue@2.x add 组件名` 添加（固定主版本保证可复现，禁止 @latest），禁止手动复制

**TanStack Table**：
- 列定义必须使用 `ColumnDef<T>` 泛型，与实体类型绑定
- 排序、分页、筛选状态与后端查询参数联动，状态变化自动触发请求
- 禁止手写表格行渲染、排序切换等基础逻辑

**Zod 表单**：
- Schema 定义在对应 `types` 文件中，校验规则与后端 DTO 数据注解完全一致
- 使用 vee-validate（`@vee-validate/zod`）绑定，错误统一通过 FormMessage 展示
- 禁止手动编写 if-else 校验逻辑

### 3.3 状态管理判定标准

| 场景 | 方案 |
| --- | --- |
| 跨 ≥2 个页面或组件树共享（用户信息、权限、全局配置） | Pinia Store |
| 仅单页面内部状态 | `ref` / `reactive` |
| 可复用的有状态逻辑 | 抽成 `composables/useXxx.ts` |

- access token 持久化到 localStorage，退出登录同步清空
- **refresh token 禁止存 localStorage**，必须走 httpOnly + Secure + SameSite=Strict Cookie（一次 XSS 即可窃取 7 天长期凭证），刷新时由浏览器自动携带
- 禁止把页面私有状态塞进全局 Store

### 3.4 请求规范

- 所有接口必须调用 `@/utils/request` 导出的 `request` 实例，接口函数统一在 `api/` 目录定义
- 入参、出参必须声明明确类型，禁止组件内直接写 axios 调用
- `/api` 前缀由 request 实例的 baseURL 统一携带，`api/` 目录函数内路径不再重复 `/api`（如 `get('/v1/users')`），避免出现 `/api/api`
- **token 无感刷新**：响应拦截器捕获 401 → 调 refresh 接口换新（refresh token 走 Cookie，请求 `withCredentials`）→ 重放原请求；刷新期间的并发请求必须进入队列等待，refresh 失效才跳转登录页

### 3.5 路由与权限

- 公开页面在路由 meta 标记 `public: true`，其余默认需要登录
- 页面权限：路由 meta `permission: '模块:操作'`，全局守卫自动校验
- 按钮级权限：`v-auth="'权限码'"` 指令，无权限自动移除元素
- 权限码命名：`模块名:操作名`（如 `user:view`、`user:add`）
- **数据权限（行级）**：后端返回用户 DataScope（All=全部 / Dept=本部门 / DeptAndChild=本部门及下级 / Self=仅本人），查询过滤在 Service 层统一拼接，禁止前端控制数据范围

### 3.6 TypeScript 与样式

- 所有变量、函数参数、返回值显式声明类型，禁止隐式 `any`
- 对象结构用 `interface`，联合类型、工具类型用 `type`
- 业务实体类型在 `types/` 目录，字段名、类型与后端 DTO 完全一致
- 样式优先 Tailwind 原子类；主题色、间距、圆角、阴影统一用 CSS `@theme` 配置（Tailwind 4 写法），不硬编码色值

### 3.7 前端错误边界

- 全局兜底：`app.config.errorHandler` 捕获未处理异常，记录并上报，展示友好错误页而非白屏
- 路由级：路由组件加载/渲染失败时降级展示（重试按钮 + 返回上一页），禁止整页白屏
- 组件级：复用组件用 `onErrorCaptured` 捕获子组件异常并降级渲染，避免局部错误拖垮整页

---

## 四、后端工程规范

### 4.1 分层架构

```
YourProject.Api/               # 接口层：Controllers / Filters / Program.cs
YourProject.Application/       # 业务层：Services / Dtos / Interfaces
YourProject.Infrastructure/    # 基础设施层：Data(AppDbContext) / Repositories / Migrations / Cache
YourProject.Domain/            # 领域层：Entities / Enums / Exceptions
```

**分层原则**：上层依赖下层，依赖注入管理生命周期；禁止跨层调用，禁止控制器直接操作 DbContext。接口（抽象）定义在 Application 层、实现在 Infrastructure 层（依赖倒置），项目引用方向为 `Api → Application`、`Infrastructure → Application`、`Infrastructure → Domain`。

### 4.2 统一响应与异常

```csharp
public class ApiResult<T>
{
    public int Code { get; set; }     // 0=成功，其他=失败
    public string Message { get; set; }
    public T Data { get; set; }
}
```

- 控制器直接返回业务对象，全局 `ResultFilter` 自动包装；禁止手动构造 ApiResult
- 业务校验失败抛 `BusinessException`（含错误码），全局 `ExceptionFilter` 统一捕获转换
- 系统异常对外仅返回"系统内部错误"，完整堆栈记入日志
- 禁止在控制器中 try-catch 后手动返回错误结果

### 4.3 认证与授权

- JWT 双 token 机制：access token 2 小时（localStorage）+ refresh token 7 天（httpOnly Cookie，见 3.3）
- refresh token 一次性使用（轮换），旧 token 刷新后立即失效，防止重放
- 密码使用 BCrypt 哈希存储，禁止明文存储和传输
- Token Claims 包含：用户ID、角色、权限标识、DataScope
- 权限权衡：权限数量大时不入 Claims，改权限缓存（Redis）+ 变更主动失效；权限变更后最长 2 小时生效（access token 周期），关键权限变更应主动踢线
- 登录失败连续 5 次锁定账户 15 分钟（Redis 计数）

### 4.4 EF Core 数据访问

- 只读查询必须 `AsNoTracking()`
- 列表查询使用 `Select()` 投影返回 DTO，禁止直接返回 Entity
- 关联数据用 `Include()` 显式加载，**禁止懒加载**（防 N+1）
- 字符串字段必须显式 `HasMaxLength()`，禁止默认 longtext
- 高频筛选、排序字段必须配置索引（`OnModelCreating` 中定义）
- 分页查询统一封装，返回 `PageResult<T>`（结构见 6.3）
- 事务在业务层控制，禁止在控制器中使用事务

### 4.5 控制器与依赖注入

- 路由：`[Route("api/v{version:apiVersion}/[controller]")]` + `AddApiVersioning()`（控制器标注 `[ApiVersion("1.0")]`，URL 形如 `/api/v1/users`），并启用 `AddRouting(o => o.LowercaseUrls = true)` 统一小写 URL
- RESTful 风格：GET 查询 / POST 新增 / PUT 修改 / DELETE 删除
- 入参使用 DTO，禁止直接使用 Entity 作为接口参数
- 分页参数继承 `PageQuery` 基类（PageIndex、PageSize、SortField、SortOrder）
- 异步方法后缀 `Async`
- 构造函数注入，禁止手动 new 服务；业务服务/仓储注册为 Scoped，工具类/全局配置注册为 Singleton；禁止用静态类存储状态

---

## 五、数据库规范

### 5.1 基础

- 字符集 `utf8mb4`，排序规则 `utf8mb4_unicode_ci`，存储引擎 InnoDB
- 表名与实体一致，帕斯卡命名（`User`、`Role`）

### 5.2 表与字段

- 所有业务表必填：`Id`（自增主键）、`CreateTime`、`UpdateTime`（datetime）；普通业务表用 int，审计日志等高速增长表用 **bigint**（int 上限约 21 亿）
- 重要业务表包含 `IsDeleted`（bool，默认 false）实现逻辑删除，**禁止物理删除**
- 「重要业务表」指用户、角色、权限、订单、审批等核心实体；日志表（审计日志除外，见第十章）、会话表、临时表等允许物理删除
- 字符串按业务指定长度（姓名 50、邮箱 100、备注 500）
- 枚举字段用 int 存储，禁止字符串
- 禁止使用存储过程、触发器

### 5.3 索引

- WHERE、排序、关联字段必须建索引；联合索引遵循最左前缀
- 大字符串字段不建普通索引，必要时用前缀索引
- 单表索引一般 ≤5 个；大数据量表按查询频率评估，不设死上限

### 5.4 迁移

- 所有表结构变更走 EF Core Migrations，迁移文件随代码提交
- 生产环境用 `Script-Migration` 生成 SQL，DBA 审核后执行
- **已提交的迁移文件禁止修改**，结构变更必须新增迁移
- 部署时迁移执行先于应用启动（见第十二章）

---

## 六、前后端接口约定

### 6.1 版本控制

- 接口 URL 统一带版本号：`/api/v1/[controller]`（Asp.Versioning 参数化路由，见 4.5；默认版本 v1）
- 破坏性变更必须升 `v2`（新控制器或同控制器 `[ApiVersion("2.0")]` 分方法映射），旧版本保留至少一个大版本周期并标注废弃

### 6.2 通用分页请求

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| pageIndex | int | 是 | 页码，从 1 开始 |
| pageSize | int | 是 | 每页条数，上限 100（超出按 100 处理） |
| sortField | string | 否 | 排序字段名 |
| sortOrder | string | 否 | `asc` / `desc` |

### 6.3 通用分页响应与统一响应结构

```ts
interface PageResult<T> {
  items: T[]
  totalCount: number
  pageIndex: number
  pageSize: number
}

interface ApiResult<T> {
  code: number     // 0=成功
  message: string
  data: T
}
```

### 6.4 错误码

| 错误码 | 含义 | 前端处理 |
| --- | --- | --- |
| 0 | 成功 | 正常处理数据 |
| 400 | 参数绑定/框架级校验失败（模型绑定、DataAnnotations） | 展示错误信息 |
| 401 | 未登录 / Token 失效 | 走无感刷新，失败则跳转登录 |
| 403 | 无权限 | 提示无权限 |
| 409 | 幂等冲突（重复提交） | 提示"请勿重复提交" |
| 500 | 系统内部错误 | 通用错误提示 |
| ≥1000 | 业务异常（含业务层校验失败，如 1002） | 展示后端具体错误信息 |

> **说明**：表中数值为响应体 `ApiResult.code` 的取值。HTTP 状态码与 0–500 段保持一致（401 未登录、403 无权限、409 冲突等）；业务异常（≥1000）HTTP 返回 200，前端依据 `code` 区分处理。
>
> **业务异常码分配**：`≥1000` 的错误码按模块分段登记在 [error-codes.md](error-codes.md)，新增业务错误码必须先查表取号并登记，禁止随意编造、禁止冲突复用。

### 6.5 其他约定

- 认证：`Authorization: Bearer {token}`
- 时间：ISO 8601 字符串
- 提交：`application/json`
- 序列化：JSON 统一 camelCase（ASP.NET Core 默认行为，前端类型定义与此对齐）
- 幂等：POST 提交类接口携带 `Idempotency-Key` 请求头（UUID），后端 Redis 去重 + 数据库唯一索引兜底。实现细则：
  - 前端生成时机：表单初始化时生成 UUID 随首次提交发送，重试沿用同一 key，新表单重新生成
  - 后端去重：Redis `SET NX` 写入 `{项目}:idempotency:{模块}:{key}`（与第九章缓存 key 命名对齐；同时缓存首次响应），TTL 24 小时（按业务可调）；已存在时对比请求体：一致 → 重放首次响应（覆盖网络超时后重试且首次已成功的场景），不一致 → 返回 409
  - 并发兜底：Redis 不可用或 SET NX 失败时由数据库唯一索引兜底，唯一约束冲突同样返回 409

---

## 七、测试规范

### 7.1 后端

- 框架：xUnit + FluentAssertions + NSubstitute
- 单元测试：Service 层覆盖核心业务分支，**行覆盖率 ≥ 60%**（Coverlet 统计，CI 强制卡点）
- 集成测试：使用 Testcontainers 起真实 MySQL 容器，覆盖 Repository 与关键 API 链路（**需本地 Docker 运行**；环境缺失时反馈并等待处理，禁止跳过集成测试交差）
- 测试命名：`方法名_场景_期望结果`（如 `CreateUserAsync_EmailDuplicated_ThrowsBusinessException`）
- 测试代码与生产代码同 MR 提交，**新增/修改业务逻辑必须附带测试**

### 7.2 前端

- 框架：Vitest + @vue/test-utils
- `stores/`、`utils/`、`composables/` 必须有单测；组件测试按需
- E2E：Playwright 覆盖登录、核心 CRUD 流程

### 7.3 红线

- **CI 中任一测试失败禁止合并**
- 禁止为了凑覆盖率写无断言测试
- 禁止 skip 测试合入 main（临时 skip 必须带 TODO 与 issue 号）

### 7.4 编写与执行职责

| 环节 | 责任方 | 方式 |
| --- | --- | --- |
| 测试编写 | AI 编码助手 | 随业务代码同 MR 生成（见「十五、AI 编码协作核心准则」第 7 条） |
| 测试执行 | 全自动 | 本地：`dotnet test` / `vitest`；CI：MR 触发自动执行 + 覆盖率卡点 |
| 测试评审 | 人工 | MR 评审时检查测试有效性（分支覆盖、断言质量） |
| 验收测试 | 人工 | 上线前 UAT，自动化不替代 |

- AI 生成测试时必须保证本地可运行通过后再提交
- 人工不做重复的手动回归，回归一律交给自动化测试

---

## 八、安全规范

### 8.1 认证与会话

- BCrypt 哈希存储密码；登录失败锁定（见 4.3）
- 密码规则：长度 ≥ 8，含大小写字母 + 数字；首次登录强制改密（可选开启）

### 8.2 传输与注入

- EF Core 参数化查询，禁止拼接 SQL 字符串
- 前端输出自动转义（XSS），后端 FluentValidation 校验输入；**禁止 `v-html` 渲染服务端/用户输入**，富文本必须先过白名单净化库（如 DOMPurify）

### 8.3 CORS 与安全响应头

- CORS 白名单显式配置，**禁止 `AllowAnyOrigin`**
- 统一添加安全响应头：`X-Content-Type-Options: nosniff`、`X-Frame-Options: DENY`、Content-Security-Policy

### 8.4 敏感信息

- 手机号、邮箱、身份证按权限脱敏：**后端脱敏后返回**（无权限直接返回 `138****8000`，本人/授权角色返回明文），前端只负责展示——前端持明文做掩码防不住抓包与调试工具
- 日志禁止输出密码、Token、密钥
- 敏感配置：开发环境 `dotnet user-secrets`，生产环境环境变量注入，**禁止入库/入仓**

### 8.5 文件上传

- 类型白名单（按业务配置，默认 jpg/png/pdf/xlsx）+ 大小限制（默认 10MB）
- 存储文件名随机化（UUID），路径不可猜测；禁止保留原始文件名直接落盘
- 文件头魔数校验：前端预检只是体验优化，**服务端必须复核**（读文件头字节比对类型白名单），不信任仅扩展名

### 8.6 限流与防滥用

- 登录、验证码、查询导出接口配置限流：优先 .NET 内置 RateLimiter 中间件（`System.Threading.RateLimiting`）；分布式限流用 Redis + 自定义分区器

---

## 九、缓存规范

- 分布式缓存：Redis（StackExchange.Redis）；进程内缓存：`IMemoryCache`
- 模式：cache-aside —— 读：先查缓存，未命中再查数据库并回填缓存；写：先更新数据库，再失效缓存
- Key 命名：`{项目}:{模块}:{实体}:{id}`，如 `admin:user:info:1001`
- 字典、组织架构、权限等低频变更数据：本地缓存 TTL 10 分钟 + 更新时主动失效；**多实例部署时改用 Redis**（或本地缓存 + Redis Pub/Sub 广播失效）——IMemoryCache 是进程内的，主动失效只清当前节点，其余节点旧数据最长残留 10 分钟
- 热点查询：Redis，TTL 按业务定（默认 5 分钟）
- **禁止缓存敏感信息明文**；缓存必须设置过期时间，禁止永久 key

---

## 十、审计日志

- 关键操作必须留痕：登录/登出、权限变更、角色分配、数据删除、审批动作、导出
- `AuditLog` 表字段：操作人ID、操作时间、IP、UserAgent、模块、操作类型、目标对象、变更前 JSON、变更后 JSON
- 变更前后 JSON 中密码哈希、手机号、邮箱、身份证等敏感字段必须脱敏或排除，禁止原样落审计表（与 8.4 一致）
- 实现：EF Core `SaveChangesInterceptor` 自动审计实体变更 + 显式 `IAuditService` 记录业务动作
- 审计日志只增不改（禁止 Update/Delete），保留策略按合规要求（默认 ≥ 1 年）
- 审计日志写入失败不阻断主流程，但必须告警

---

## 十一、Git 协作流程

### 11.1 分支模型

- `main`：受保护分支，禁止直接 push、禁止 force push
- 功能分支从 `main` 拉出，命名：`feature/xxx`、`fix/xxx`、`docs/xxx`、`refactor/xxx`、`hotfix/xxx`
- 合并必须走 MR/PR，至少 1 人 approve + CI 全绿

### 11.2 Commit 规范

Conventional Commits：`feat|fix|docs|style|refactor|test|chore(scope): 描述`

示例：`feat(user): 新增用户导出功能`、`fix(auth): 修复 token 刷新并发丢失`

### 11.3 MR 自查清单

- [ ] 遵循本规范命名与分层
- [ ] 复用已有封装，无重复造轮子
- [ ] 新增/修改逻辑已附带测试
- [ ] 类型完整，无 `any`
- [ ] 无敏感信息硬编码
- [ ] 数据库变更已生成迁移文件
- [ ] 文档/接口约定已同步更新

---

## 十二、CI/CD 与部署

### 12.1 流水线（每次 MR 自动触发）

```
lint → build → test（单测+集成）→ 覆盖率卡点 → 镜像构建 → 推送镜像仓库
```

- 镜像仓库：Harbor / 阿里云 ACR
- 镜像 tag：`{语义化版本}-{git短sha}`，禁止覆盖已推送的 tag

### 12.2 环境与发布

| 环境 | 触发方式 | 说明 |
| --- | --- | --- |
| dev | 合入 main 自动部署 | 每日验证 |
| staging | 手动审批 | 预发验证，数据结构同生产 |
| prod | 手动审批 | 保留最近 3 个版本，支持一键回滚 |

- 部署顺序：**先执行数据库迁移脚本，再滚动更新应用**；迁移必须向后兼容（先扩后收：先加列/加表，删除列/改名放后续版本），滚动更新期间新旧版本共存
- 生产发布后 15 分钟内观察核心指标（错误率、P95 耗时），异常立即回滚；错误率按 `ApiResult.code ≠ 0` 统计——业务异常 HTTP 返回 200，只看 5xx 会漏报

### 12.3 Docker 构建

- 前端：多阶段构建（node:24-alpine 构建 → nginx:alpine 托管），最终镜像 ≤ 50MB，Nginx 同时处理静态资源与 `/api` 反代
- 后端：多阶段构建（sdk 构建 → aspnet 镜像运行，如 `mcr.microsoft.com/dotnet/aspnet:10.0`），容器内端口 8080（.NET 8+ 官方镜像默认 `ASPNETCORE_HTTP_PORTS=8080`，可按需覆盖）
- **禁止镜像内硬编码连接串、密钥**，统一环境变量注入

### 12.4 生产编排

- docker-compose 编排前端、后端、MySQL、Redis
- MySQL 数据、Redis 数据挂载宿主机卷
- 仅前端 Nginx 暴露 80/443；后端、数据库、Redis 仅内部网络通信

---

## 十三、代码风格与命名

### 13.1 前端（TS / Vue）

| 类型 | 规范 | 示例 |
| --- | --- | --- |
| 组件文件/组件名 | 帕斯卡 | `UserManage.vue` |
| 变量 / 函数 | 小驼峰 | `userList`、`fetchData()` |
| 接口 / 类型 | 帕斯卡 | `interface UserDto` |
| 常量 | 全大写下划线 | `DEFAULT_PAGE_SIZE = 10` |

### 13.2 后端（C#）

| 类型 | 规范 | 示例 |
| --- | --- | --- |
| 类 / 接口 | 帕斯卡，接口前缀 I | `UserService`、`IUserRepository` |
| 方法 / 属性 | 帕斯卡，异步后缀 Async | `CreateUserAsync()` |
| 参数 / 局部变量 | 小驼峰 | `userId` |

### 13.3 通用

- 命名语义化，禁止拼音、无意义缩写
- 注释与文档统一使用中文（标识符用英文，见 13.1 / 13.2）
- 关键业务逻辑必须注释；公共方法必须有文档注释（`<summary>`）
- 函数单一职责，超过 50 行拆分
- 缩进：前端 2 空格，后端 4 空格

---

## 十四、性能预算

| 指标 | 预算 | 校验方式 |
| --- | --- | --- |
| 前端主 chunk（gzip） | ≤ 500KB | vite-bundle-visualizer，CI 卡点 |
| 首屏 LCP | ≤ 2.5s | Lighthouse / 线上监控 |
| API P95 响应 | ≤ 500ms（不含导出类） | Serilog 耗时统计 + 监控告警 |
| 路由级代码分割 | 必须 | 所有 views 路由懒加载 |
| 列表页分页 | 必须 | 禁止一次性全量查询 |

---

## 十五、AI 编码协作核心准则

> 优先级从高到低，冲突时序号小者优先

1. **规范优先**：严格遵循本规范，冲突时以本文件为准
2. **复用优先**：优先使用已有封装、组件、工具函数，禁止重复造轮子
3. **必须先确认再动手的场景**（其余场景可直接交付）：
   - 引入任何本规范未提及的第三方依赖
   - 架构选型、新增分层、修改目录结构
   - 删除或重写已有文件/公共封装
   - 数据库表结构变更（含索引调整）
   - 破坏性接口变更（涉及 API 版本升级）
   - 后端 DTO / 接口数据结构变更（需同步前端 types）
4. **类型安全**：全链路类型严格对齐，禁止 `any`、禁止类型不匹配
5. **分层清晰**：严格遵守前后端分层，不跨层写逻辑
6. **完整可运行**：输出代码必须完整、可直接运行，关键依赖必须说明
7. **测试随行**：新增/修改业务逻辑必须附带测试
8. **注释适度**：关键逻辑加注释，不写冗余注释
9. **最小改动**：不修改无关代码结构，不做规范未要求的"顺手重构"

---

## 附录 A：代码输出自查清单

输出代码前逐项校验：

- [ ] 符合命名与格式要求
- [ ] 复用了已有封装（request、store、通用组件），无重复实现
- [ ] 类型完整，无 `any`
- [ ] 符合前后端接口约定（版本、分页、幂等、错误码）
- [ ] 无安全隐患（明文密码、SQL 拼接、敏感信息泄露、CORS 放开）
- [ ] 无未约定的第三方依赖
- [ ] 数据库变更有迁移文件
- [ ] 后端 DTO 变更已同步前端 types
- [ ] 新增业务错误码已登记 docs/error-codes.md
- [ ] 新逻辑附带测试
- [ ] 关键操作有审计日志
- [ ] 页面/接口满足性能预算

## 附录 B：常用命令速查

### 前端

| 用途 | 命令 |
| --- | --- |
| 安装依赖 | `npm ci`（按 lockfile）/ `npm install`（变更依赖，需确认） |
| 启动开发 | `npm run dev` |
| 构建 | `npm run build` |
| 代码检查 | `npm run lint` / `npm run format` |
| 单元测试 | `npm run test:unit` |
| E2E 测试 | `npx playwright test` |
| 新增 shadcn 组件 | `npx shadcn-vue@2.x add <组件名>`（固定主版本，禁止 @latest） |

### 后端

| 用途 | 命令 |
| --- | --- |
| 还原依赖 | `dotnet restore` |
| 构建 | `dotnet build` |
| 启动接口 | `dotnet run --project src/YourProject.Api` |
| 单元/集成测试 | `dotnet test` |
| 新增迁移 | `dotnet ef migrations add <Name> --project src/YourProject.Infrastructure --startup-project src/YourProject.Api` |
| 格式检查 | `dotnet format` |

> 项目名/目录名以实际仓库为准；与精简版 [CLAUDE.md](../CLAUDE.md)「一、常用命令」保持一致。
> `dotnet ef` 工具版本需与 EF Core 主版本匹配（版本不匹配会报错；升级：`dotnet tool update --global dotnet-ef`）。

## 附录 C：本地 AI 权限配置

仓库根目录 `.claude/settings.json` 将「十五、AI 编码协作核心准则」第 3 条固化为权限规则（随仓库提交，团队共享）：

- **允许（免确认）**：日常验证命令（build / lint / format / test、本地启动）与只读 git 命令
- **确认（弹窗询问）**：新增/卸载依赖、EF 迁移、shadcn 组件、git 写操作、Docker、删除文件、外网访问
- **拒绝（不可执行）**：修改 `components/ui` 源码、读取 `.env` 全家族密钥文件（`.env` / `.env.local` / `.env.*.local`，含子目录递归）、force push、`rm -rf`、`npm publish`

规则说明与按仓库调整方法见 `.claude/README.md`；个人差异写在 `.claude/settings.local.json`（加入 .gitignore，不随仓库提交）。
