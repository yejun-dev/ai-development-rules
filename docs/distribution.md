# 规范分发指南：如何让所有新项目都遵守本规范

Claude Code 的官方机制里没有"一键应用到所有新项目"的开关，需要分层组合。**推荐组合：第 1 层（模板，核心）+ 第 2 层（个人全局，兜底）**；第 3 层用于长期演进（可选）。企业强制方案本团队不采用——如未来需要组织级强制（admin console 受管设置 / MDM 策略 CLAUDE.md），需 Claude for Teams/Enterprise 计划，届时再评估。

## 第 1 层：项目模板（核心方案，新建项目自动带上）

规范随仓库提交是官方支持的主通道（clone 即生效、路径类权限规则以项目根为基准、效果最准）。

- **方案 A（推荐）· Git 模板仓库**：把本规范包建成模板仓库，托管平台（GitHub "Use this template" / GitLab Template）建仓时选择该模板，规范文件自动就位。
- **方案 B · 脚手架脚本**：新项目初始化时复制最新版：
  ```bash
  # 从规范包仓库复制
  git clone --depth 1 <规范包仓库地址> /tmp/rules
  cp -r /tmp/rules/CLAUDE.md /tmp/rules/.claude /tmp/rules/docs <新项目>/
  ```
  团队脚手架 CLI（如 `create-xxx`）应在建项目时自动执行这一步。
- **方案 C · 大仓（monorepo）**：仓库根放一份，`frontend/`、`backend/` 子目录各放更细的 CLAUDE.md（Claude Code 按工作目录逐级加载）。

## 第 2 层：个人全局配置（每台开发机配一次，所有项目兜底）

对本机所有项目、所有会话生效；项目内有更具体的 CLAUDE.md 时以项目为准。

1. 把 [global/CLAUDE.md](../global/CLAUDE.md) 复制为 `~/.claude/CLAUDE.md`（Windows：`C:\Users\<用户名>\.claude\CLAUDE.md`）
2. 把 [global/settings.json](../global/settings.json) 合并进 `~/.claude/settings.json`（注意：permissions 数组跨层是**合并相加**，不是替换）

**关键坑**：
- 全局设置里**不要放 `./` 相对路径规则**——按会话当前目录（cwd）解析，在各项目间不可靠。跨项目保护敏感文件用 `//**/.env` 这类绝对锚定写法。
- 项目级路径规则（如 ui 只读、.env 禁读）留在各项目的 `.claude/settings.json`，全局只放通用红线。
- 全局 deny 影响该机器**所有**项目（含与团队无关的个人项目），个人可斟酌删减。

## 第 3 层：规范演进与同步（可选）

规范包独立建仓、打版本：

- **CI 同步**：定时任务把新版 CLAUDE.md / settings.json 以 MR 形式推到各项目（类似 dependabot），由各项目 owner 评审合入。
- **插件 marketplace**：适合同时分发 skills / hooks 的场景；注意插件 `version` 是缓存键，**不递增则更新不会到达已安装用户**。

## 常见坑（全部实测确认）

- **必须从仓库根目录启动 Claude Code**：从子目录启动会静默丢失项目级 `.claude/settings.json`（官方 issue #74023）。
- permissions 数组跨层合并、**只能加不能减**；评估顺序 deny → ask → allow，deny 永远最高优先、任何层都无法覆盖。
- 设置文件解析失败会整文件拒绝生效，用 `claude doctor` 排查。
- 验证三件套：`/memory` 看实际加载的记忆文件；`/permissions` 看最终生效的权限规则；`/status` 看设置来源。
