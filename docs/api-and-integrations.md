# API 与集成指南

## API 基础

默认前缀是 `/api/v1`，服务启动后打开 `/docs` 查看实际 OpenAPI。除健康检查外，接口通常需要 trusted actor 或 service token。请求中的项目参数用于选择资源，不用于声明调用者身份。

## 项目查询接口

- `POST /projects/snapshot`：获取项目快照。
- `POST /projects/timeline`：查询项目时间线。
- `POST /projects/change-impact`：分析变更影响。
- `POST /projects/fact-resolution`：解析指定事实。
- `POST /projects/source-gaps`：查找来源缺口。
- `POST /projects/knowledge-gaps`：生成知识缺口报告。

这些接口返回的内容仍会经过访问范围过滤；服务端不会因为请求者知道某个 source id 就直接返回该来源。

## Project Agent 接口

- `GET /project-agent/project-spaces`：列出可见项目空间。
- `GET /project-agent/project-spaces/{tenant_id}/{project_id}/scope`：查看有效范围。
- `GET /project-agent/tools`：发现当前 actor 可调用工具。
- `POST /project-agent/tools/call`：显式调用一个工具。
- `POST /project-agent/ask`：执行一次项目问答。
- `GET /project-agent/runs/{run_id}/detail`：查看运行详情。
- `GET /project-agent/hermes-loops/{run_id}/events`：查看 Hermes 事件。

建议先调用 scope 和 tools，再提交 ask，便于在客户端显示当前身份和能力边界。

## 记忆接口

记忆链路包括提议、审核、检索、版本和撤销。生产客户端不要直接把用户输入写成已确认记忆，应走 proposal/review 流程。

## 飞书接口

`POST /feishu/events` 接收事件和卡片动作；`GET /integrations/feishu/status` 返回配置和绑定状态。长连接模式由 `scripts/start-projectlens-feishu-ws.ps1` 启动 ingress，再把事件转发到本地 API。

## MCP

MCP 命令入口为 `project-lens-mcp`。stdio 没有 HTTP middleware，必须使用部署侧的 `MCP_TRUSTED_TENANT_ID`、`MCP_TRUSTED_ACTOR_ID`、`MCP_TRUSTED_CHAT_ID` 和 `MCP_TRUSTED_CHAT_TYPE` 固定调用者。工具参数中的 actor 字段不能覆盖受信任上下文。

## 外部连接器

连接器将外部数据转换为统一 SourceRecord，再由知识索引服务处理。连接器同步应具有幂等性、cursor 和 freshness；如果需要验证真实连接，先调用 health/status，再执行 sync。
