# 开发计划

## 核心文档颗粒度标准

- [x] 阅读中央 `AGENTS.md`、项目文档、近期 Git 历史和 `agents-md-sync` skill。
- [x] 参考 `skill-creator`、`plugin-creator` 和 `openai-docs` 的文档组织惯例，提炼“最少充分、可验证、可交接”的颗粒度标准。
- [x] 在中央 `AGENTS.md` 的“文档权威与同步”章节补充 PRD、DESIGN、TECH、SPEC、DEVELOPMENT_PLAN 和 AGENTS 的职责边界与最低覆盖项。
- [x] 更新 `docs/PRD.md`，补充本次需求的目标、范围、约束和验收标准。
- [x] 新增 `docs/DESIGN.md`，说明本次无前端影响，并记录文档体验目标。
- [x] 更新 `docs/SPEC.md`，将文档颗粒度要求拆成可检查规格。
- [x] 更新 `docs/TECH.md`，说明技术取舍、仓库结构和运行时无影响。
- [x] 执行文档一致性检查、Git diff review 和 Markdown 空白检查。
- [x] 创建清晰 Git commit。

## AGENTS 同步授权优化

- [x] 梳理现有 `agents-md-sync` 流程和命令模板。
- [x] 在 skill 默认配置中加入 `publish_authorization`。
- [x] 增加安装或首次使用时显式索权的流程说明。
- [x] 增加发布前授权校验规则。
- [x] 补充 PowerShell 和 Bash 命令模板。
- [x] 更新中央 `AGENTS.md` 通用约束。
- [x] 补充需求、技术和规格文档。
- [x] 完成文档一致性检查和 Git diff review。

## 发布分支命名防冲突

- [x] 阅读中央 `AGENTS.md`、本地项目文档、近期 Git 历史和 `agents-md-sync` skill。
- [x] 将 skill 默认发布分支规则从裸仓库名改为 GitHub 完整名 `owner/repo`。
- [x] 更新 PowerShell 和 Bash 命令模板，优先使用 `nameWithOwner`，并在 Git remote fallback 中解析 owner。
- [x] 增加无法解析 owner 时阻塞发布的规则，避免回退到裸仓库名。
- [x] 同步更新中央 `AGENTS.md`、PRD、SPEC、TECH 和计划文档。
- [x] 执行文档一致性检查、Git diff review 和 Markdown 空白检查。

## 发布内容收敛与 gh 沙箱误判修正

- [x] 阅读中央 `AGENTS.md`、本地项目文档、近期 Git 历史、`agents-md-sync` skill 和命令模板。
- [x] 明确中央项目分支只允许发布目标项目根目录给 AI 读取的 `AGENTS.md`。
- [x] 更新 skill 描述、默认值和发布流程，禁止发布 `docs/`、`skills/`、源码、配置、报告或其他路径。
- [x] 更新 PowerShell 和 Bash 发布模板，使用单文件 orphan 发布分支、显式暂存 `AGENTS.md`，并在发现其他路径时中止。
- [x] 补充 Codex 沙箱中 `gh` 权限类失败的处理规则：先按沙箱限制判断，必要时提权重试同一命令，再决定是否 fallback 或报告真实授权问题。
- [x] 同步更新中央 `AGENTS.md`、PRD、SPEC、TECH 和计划文档。
- [x] 执行文档一致性检查、Git diff review、Markdown 空白检查和发布模板单文件模拟。
- [x] 创建清晰 Git commit。
