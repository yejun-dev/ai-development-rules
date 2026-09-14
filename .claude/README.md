# .claude/settings.json — 权限配置说明

把 [CLAUDE.md](../CLAUDE.md)「四、必须确认的场景」固化为 Claude Code 的权限规则，与规范形成强制闭环。

**接入方式**：本目录随仓库提交，团队全员共享。个人差异（如本机路径）写在 `.claude/settings.local.json` 并加入 `.gitignore`，不提交。

## 规则与规范的对应关系

| 分组 | 规则 | 对应规范 |
| --- | --- | --- |
| **允许**（免确认） | 日常验证命令：build / lint / format / test（前端与后端）、`dotnet run` 本地启动、只读 git 命令（status/diff/log/show） | CLAUDE.md「一、常用命令」与工作准则第 4 条（交付前必须验证） |
| **确认**（弹出询问） | 新增/卸载依赖（npm、`dotnet add`）、EF 迁移与数据库操作、新增 shadcn 组件、git 写操作（commit/push/merge/rebase 等）、Docker、删除文件（rm）、外网访问（WebFetch/WebSearch/curl/wget） | 「四、必须确认的场景」：新依赖、数据库变更、破坏性操作；curl/wget 额外覆盖安全规范 8.4（防外传敏感信息） |
| **拒绝**（不可执行） | 修改 `src/components/ui/` 源码、读取 `.env` 与 `.env.*.local` 密钥文件、force push、`rm -rf`、`npm publish` | 3.2（shadcn 只读红线）、8.4（敏感信息）、11.1（main 禁止 force push） |

未列入上表的命令按 Claude Code 默认模式处理（首次执行时询问）。

## 按实际仓库调整

1. **前端目录**：模板假设前端在仓库根。若在 `frontend/` 子目录，把 deny 中的 `./src/components/ui/**` 改为 `./frontend/src/components/ui/**`。
2. **包管理器**：团队使用 npm；若改用 pnpm/yarn，需在 ask 中补充 `Bash(pnpm add *)` 等规则。
3. **后端路径**：`dotnet run --project *`、迁移命令中的项目名以实际 `.sln` 为准（CLAUDE.md 已同步占位）。

## 已知局限（依赖人工兜底）

- **必须从仓库根目录启动 Claude Code**：从子目录启动会静默丢失项目级 `.claude/settings.json`（官方 issue #74023），团队请统一从仓库根启动。
- **前缀匹配**：权限规则按命令前缀匹配。`git push -f` 写在命令末尾（如 `git push origin main -f`）时 deny 规则拦不住——但所有 push 都已列入 ask，人工确认时能看到完整命令，这是最终防线。
- **deny 只拦 AI 的工具调用**：`npx shadcn-vue add`（ask 放行后）由 CLI 进程写入 `ui/` 目录，不会触发 Edit/Write 拦截；靠规范 3.2 约束。

## 全局开关说明

- `defaultMode: "default"`：默认权限模式，allow/ask/deny 之外的操作首次执行时询问。
- `disableBypassPermissionsMode: true`：禁止绕过权限模式（AI 和人都无法跳过确认）。若团队觉得过严可删除此键——但不建议，它与规范「先确认再动手」原则一致。
