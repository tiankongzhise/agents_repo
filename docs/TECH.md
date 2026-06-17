# 技术说明

## 仓库结构

- `AGENTS.md`：中央通用协作约束。
- `skills/agents-md-sync/SKILL.md`：同步 skill 的主要流程说明。
- `skills/agents-md-sync/references/commands.md`：跨平台命令模板。
- `skills/agents-md-sync/agents/openai.yaml`：skill 展示元数据。

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
