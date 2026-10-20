# 配置与运行手册

本文说明 ProjectLens 的配置层级、启动方式、运行状态检查和常见运维动作。所有配置变量都使用 `PROJECT_LENS_` 前缀，完整字段以根目录 `.env.example` 为准。

## 配置优先级

运行时配置来自：

1. 代码默认值；
2. 仓库根目录 `.env`；
3. 进程环境变量；
4. 启动参数或部署系统注入的环境变量。

不要把生产密钥写入 `config/projects/*.json`。项目路径可以写相对路径，加载时相对于仓库根目录解析。

## 三种运行模式

### 本地安全模式

```dotenv
PROJECT_LENS_MODEL_PROVIDER=stub
PROJECT_LENS_MODEL_LIVE=false
PROJECT_LENS_ALLOW_EXTERNAL_CALLS=false
PROJECT_LENS_CONVERSATION_STORE=sqlite
```

适合开发、单元测试和离线演示。飞书使用 recording adapter，外部模型和连接器不会发起真实请求。

### 外部模型模式

配置 `MODEL_LIVE=true`、provider、模型和 API key 后启用。测试期间仍会被 isolation guard 阻断真实外呼；生产环境必须确认 API key、超时、重试和 fallback 行为。

### 飞书 Hermes 模式

生产问题固定由 Hermes 处理：

```dotenv
PROJECT_LENS_AGENT_MODE=hermes
PROJECT_LENS_FEISHU_HERMES_PROVIDER=openai
PROJECT_LENS_FEISHU_HERMES_MODEL=gpt-5.4-mini
PROJECT_LENS_FEISHU_HERMES_LOOP_TIMEOUT_SECONDS=120
```

工程执行、风险后台审查和高级工具仍需单独开启，不会因为 Hermes 已启用而自动开启。

## 启动与状态

```powershell
py -3.12 -m uvicorn project_lens.main:app --host 127.0.0.1 --port 8000 --workers 1
```

本地长连接联调使用 `127.0.0.1`。如果使用飞书 HTTP 回调或反向代理部署，监听地址改为 `0.0.0.0`，前置 HTTPS 反向代理转发到 `127.0.0.1:8000`，并确保只运行一个 Uvicorn worker。当前服务使用 SQLite 和进程内调度器，不支持多 worker 横向启动。

检查顺序：

1. `GET /api/v1/health`：进程和基础依赖是否可响应。
2. `GET /api/v1/ready`：数据库、ProjectSpace、连接器或外部依赖是否满足就绪条件。
3. `GET /api/v1/release-gate`：当前配置是否满足发布门禁。
4. `GET /api/v1/integrations/feishu/status`：飞书是否配置、绑定数量和 agent mode 是否正确。

## 数据库与备份

`PROJECT_LENS_DATABASE_PATH` 默认是 `project_lens.db`。生产环境应使用固定绝对路径、定期备份并避免多个进程共享不受控的临时目录。GenAI trace 由 `PROJECT_LENS_GENAI_TRACE_DIR` 管理，按 `PROJECT_LENS_GENAI_TRACE_RETENTION_DAYS` 清理。

不要在生产使用 `:memory:`，否则重启会丢失会话、记忆、审批、索引和运行事件。

## 连接器与索引

连接器同步和知识索引是独立步骤：先同步来源，再构建/同步索引。词法检索不需要 embedding；启用向量时需要同时配置 embedding provider、模型、版本、chunker 和索引版本。发生模型或切分版本变化时，应重新同步索引并检查状态接口。

## 安全开关

- `ALLOW_EXTERNAL_CALLS`：外部调用总闸门。
- `SERVICE_TOKEN`：服务间 API 鉴权。
- `MCP_TRUSTED_*`：MCP 受信任调用者。
- `FEISHU_WIKI_PUBLISH_ENABLED`：Wiki 发布。
- `FEISHU_LONG_CONNECTION_ENABLED`：是否由当前进程启动飞书 WebSocket 长连接；HTTP 回调部署必须关闭。
- `OBSIDIAN_ENABLED` / `OBSIDIAN_OPERATIONS_ENABLED`：Obsidian 及写入操作。
- `FEISHU_HERMES_EXECUTION_ENABLED`：工程执行。
- `FEISHU_HERMES_ADVANCED_TOOLS`：高级工具。

上线前逐项确认开关，避免仅因为 `.env.example` 中存在字段就误以为功能已启用。
