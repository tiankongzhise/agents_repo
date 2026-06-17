# 技术说明

## 仓库结构

- `AGENTS.md`：中央通用协作约束。
- `skills/agents-md-sync/SKILL.md`：同步 skill 的主要流程说明。
- `skills/agents-md-sync/references/commands.md`：跨平台命令模板。
- `skills/agents-md-sync/agents/openai.yaml`：skill 展示元数据。
- `docs/PRD.md`：中央规则库需求与验收标准。
- `docs/DESIGN.md`：文档协作体验与前端影响说明。
- `docs/TECH.md`：中央规则库技术说明与关键取舍。
- `docs/SPEC.md`：可实现、可验证的规则规格。
- `docs/DEVELOPMENT_PLAN.md`：任务拆分、状态和验证记录。

## 文档颗粒度技术取舍

文档标准写入中央 `AGENTS.md`，而不是新增脚本或校验器。原因是该仓库服务于跨项目协作约束，当前需求需要给 AI 助手和开发者提供判断边界，不需要引入运行时依赖或固定格式校验。

规则采用职责边界加最低覆盖项的方式：

- 职责边界防止 PRD、SPEC、TECH、计划和 AGENTS 互相重复。
- 最低覆盖项保证新增需求能从用户目标追溯到实现规格、验证命令和后续状态。
- “最少充分、可验证、可交接”保留项目按复杂度调整篇幅的自由度。

参考惯例来自已安装 skill 的编写原则：保持核心说明精简，按任务脆弱度设置自由度，把可重复或细节较多的内容拆成专门文档或参考资料，避免把所有上下文堆入单个文件。

## 运行时影响

本次变更只修改 Markdown 文档，不涉及：

- 同步脚本或命令模板执行流程。
- 外部 API、SDK 或云服务。
- 数据模型、数据库迁移或部署环境。
- 前端构建、路由、组件或样式。
- 安全凭据、授权配置或发布权限。

## 授权配置

`agents-md-sync` 使用目标项目根目录的 `.agents-sync.json` 保存同步配置。发布授权使用 `publish_authorization` 字段：

```json
{
  "publish_authorization": {
    "granted": true,
    "scope": "push_local_agents_to_central_repo",
    "central_repo": "git@github.com:tiankongzhise/agents_repo.git",
    "local_agents_path": "AGENTS.md",
    "granted_at": "YYYY-MM-DDTHH:MM:SSZ",
    "granted_by": "user",
    "note": "User authorized agents-md-sync to publish only this project's root AI-facing AGENTS.md to the central rules repository branch for this project."
  }
}
```

## 发布校验

发布前必须同时满足：

- `.agents-sync.json` 未设置 `enabled: false`。
- `.agents-sync.json` 未设置 `auto_publish_on_agents_change: false`。
- `publish_authorization.granted` 为 `true`。
- `publish_authorization.scope` 为 `push_local_agents_to_central_repo`。
- 授权记录中的 `central_repo` 和 `local_agents_path` 与本次发布目标一致。

缺少或不匹配时，只允许执行读取、比较和本地更新，不能推送中央仓库。

## 发布内容边界

中央项目分支不是目标项目镜像，只承载目标项目根目录给 AI 读取的 `AGENTS.md`。发布流程使用中央仓库临时 clone，但项目分支应作为单文件 artifact 处理：

- 源文件限定为目标项目根目录 `AGENTS.md`。
- 目标文件限定为中央临时仓库根目录 `AGENTS.md`。
- 临时中央仓库只显式暂存 `AGENTS.md`，不得使用 `git add .`。
- 提交前读取 `git status --porcelain`，如出现任何非 `AGENTS.md` 路径则中止。
- 远端项目分支已有相同 `AGENTS.md` 时跳过 commit 和 push。

命令模板采用 orphan 发布分支，避免从中央 `main` 继承本仓库自己的 `docs/`、`skills/` 等内容。这样中央项目分支只表达“该项目当前 AI 协作规则文件是什么”，维护者再从中人工提炼公共 `AGENTS.md`。

## 发布分支命名

中央仓库的项目回传分支使用 GitHub 完整仓库名 `owner/repo`，而不是只使用 repo 名。这样可以避免不同 owner 拥有同名仓库时写入同一个中央分支。

解析优先级：

- GitHub CLI 可用时，使用 `gh repo view --json nameWithOwner --jq .nameWithOwner`。
- GitHub CLI 不可用时，从 `git remote get-url origin` 解析 `github.com:owner/repo.git`、`https://github.com/owner/repo.git` 或等价 GitHub URL。
- 如果无法解析 owner 或 repo，发布流程必须失败并要求用户确认完整 `owner/repo`，不能静默降级为裸仓库名。

清洗规则：owner 和 repo 分段转小写，仅保留字母、数字、`.`、`_`、`-`，其他字符替换为 `-`；清洗后使用 `/` 拼接为发布分支，例如 `tiankongzhise/auto_backup_bdnetdesk`。

## Codex 沙箱中的 GitHub CLI

Codex 沙箱可能限制 `gh` 读取本机凭据、keychain、配置目录或访问网络。因此 `gh auth status`、`gh repo view`、`gh api` 在沙箱内出现权限类、认证类或网络类失败时，不能直接判定为 GitHub CLI 未登录或 token 无效。

处理顺序：

- 先把错误归类为“可能的沙箱限制”。
- 若该 `gh` 命令对当前同步或发布必要，按 Codex escalation 规则提权重试同一命令。
- 提权后仍失败，再根据错误内容判断是否为真实 GitHub 授权、仓库权限或 token 问题。
- 不因沙箱内 `gh` 失败改写或撤销项目 `.agents-sync.json` 中的 `publish_authorization`。
- `gh` 不可用时，按 skill 规则 fallback 到 plain Git。
