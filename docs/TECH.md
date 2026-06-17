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
    "note": "User authorized agents-md-sync to publish this project's AGENTS.md to the central rules repository branch for this project."
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
