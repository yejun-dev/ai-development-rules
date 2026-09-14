# AI Development Rules — Claude Code 开发规范包

Vue 3 + .NET 10 全栈项目的 Claude Code 规范模板：精简版工作规范（AI 每次会话加载）+ 全量规范（人工评审参考）+ 权限规则 + 分发指南。

## 文件结构

| 路径 | 用途 |
| --- | --- |
| [CLAUDE.md](CLAUDE.md) | 精简版工作规范，放**项目根目录**，AI 每次会话自动加载 |
| [.claude/settings.json](.claude/settings.json) | 项目级权限规则（allow/ask/deny），随仓库提交；说明见 [.claude/README.md](.claude/README.md) |
| [docs/development-spec.md](docs/development-spec.md) | 全量规范（15 章 + 附录），供人工评审、新人阅读 |
| [docs/error-codes.md](docs/error-codes.md) | 业务错误码注册表，新增业务错误码必须在此登记 |
| [global/CLAUDE.md](global/CLAUDE.md) | 个人全局版工作规范，复制到 `~/.claude/CLAUDE.md` |
| [global/settings.json](global/settings.json) | 个人全局版权限规则，合并进 `~/.claude/settings.json` |
| [docs/distribution.md](docs/distribution.md) | 如何让**所有新建项目**都遵守本规范（模板 / 全局 / 同步 三层方案） |

## 快速上手

1. **单个项目**：把 `CLAUDE.md`、`.claude/`、`docs/` 复制到仓库根目录，并把 `.claude/settings.local.json` 加入项目的 `.gitignore`。
2. **所有项目**：按 [global/](global/) 里的两个文件配置一次本机。
3. **团队分发与演进**：见 [docs/distribution.md](docs/distribution.md)。
4. **使用注意**：必须从仓库根目录启动 Claude Code（子目录启动会丢失项目级设置）。
