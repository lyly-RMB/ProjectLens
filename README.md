# ProjectLens

ProjectLens 是面向软件研发团队的证据驱动项目协作 Agent 服务。它把代码仓库、项目文档、任务、发布记录、运行信号、会话历史和项目记忆组织成受权限约束的上下文，再由 Hermes 通过只读工具调查并生成可追溯回答。

ProjectLens 的核心不是“让模型知道更多”，而是让每次回答都明确：回答的是哪个项目、使用了哪些证据、当前操作者能看到什么、哪些内容仍然未知，以及后续需要谁补充什么。

这是一个 MIT 许可的开源项目原型。公开仓库包含核心代码、示例数据、测试和技术文档，不包含生产凭据、真实项目配置、运行数据库、Obsidian 状态或内部过程材料。

## 目录

- [能力与边界](#能力与边界)
- [工作原理](#工作原理)
- [代码结构](#代码结构)
- [快速开始](#快速开始)
- [配置方法](#配置方法)
- [接入项目](#接入项目)
- [HTTP API](#http-api)
- [飞书与 MCP](#飞书与-mcp)
- [数据、持久化与安全](#数据持久化与安全)
- [测试与开发](#测试与开发)
- [故障排查](#故障排查)
- [开源说明](#开源说明)
- [文档索引](#文档索引)

## 能力与边界

### 已实现能力

- 项目问答：项目概览、状态、负责人、需求与实现、变更影响、风险、时间线和知识缺口。
- 证据检索：SourceRecord、Evidence、项目快照、历史运行、记忆和知识索引。
- 权限隔离：按租户、项目、用户角色、飞书群聊可见性和来源白名单求交。
- Hermes 工具循环：模型规划调查，ProjectLens 控制工具、权限、上下文预算和审计。
- 知识闭环：连接器同步、知识索引、知识缺口、记忆审核、风险反馈和评测回放。
- 接入方式：FastAPI、MCP server、飞书 HTTP 事件和飞书 WebSocket 长连接。

### 默认不做的事情

- 不把模型记忆当作项目事实；回答必须尽量落到可访问证据。
- 不默认改代码、开 PR、部署或回滚。
- 不允许用户通过提示词或工具参数伪造身份、项目或权限。
- 不把设计稿、测试样例和演示数据描述为生产保证。

Wiki 发布、Obsidian 导出、真实飞书消息、外部模型、后台风险审查和工程执行均由独立开关控制，默认关闭或使用安全的本地/录制实现。

## 工作原理

```text
请求入口 -> 身份/租户/会话解析 -> ProjectSpace 与群绑定
-> 角色权限 ∩ 群聊可见性 ∩ 项目资源权限
-> 证据、记忆、历史和知识索引检索
-> Hermes 只读工具循环 -> 事实/推断/未知/冲突校验
-> API、MCP 或飞书回答 -> 运行事件、引用、反馈和知识缺口持久化
```

ProjectSpace 是项目边界，通过 `config/projects/*.json` 或 `PROJECT_LENS_LOCAL_PROJECT_REGISTRY` 注册项目、数据源、成员、角色策略、公开来源和群聊绑定。公开仓库中的配置只用于示例；真实部署应使用自己的项目配置、成员和凭据。生产问答建议使用 `agent_mode=hermes`，本地默认可以保持 Stub 模式。

## 代码结构

```text
src/project_lens/  API、Agent、应用服务、上下文检索、项目空间、运行时与集成
config/projects/   ProjectSpace JSON
examples/          示例项目数据
tests/             单元、接口、集成和安全回归
scripts/           启动、冒烟和真实联调脚本
data/              SQLite、trace 和其他本地运行数据
```

## 快速开始

要求 Python 3.12+。Windows PowerShell 示例：

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
py -3.12 -m pip install -e ".[dev,mcp]"
Copy-Item .env.example .env
py -3.12 -m uvicorn project_lens.main:app --host 127.0.0.1 --port 8000 --workers 1
```

验证服务：

```powershell
Invoke-RestMethod http://127.0.0.1:8000/api/v1/health
Invoke-RestMethod http://127.0.0.1:8000/api/v1/ready
Invoke-RestMethod http://127.0.0.1:8000/api/v1/release-gate
```

浏览器入口：`http://127.0.0.1:8000/docs`。默认加载 `config/projects/`，默认数据库为 `project_lens.db`。公网 HTTP 回调部署时监听地址应改为 `0.0.0.0`，并通过反向代理提供 HTTPS；保持单 worker，避免 SQLite、后台调度器或飞书长连接重复启动。当前仓库没有 serverless、Docker 或云平台专用部署配置，推荐先部署在单台 VM/主机上；不要把当前 SQLite + 进程内调度器配置直接扩展为多副本服务。

### 部署边界

- 生产入口应使用单个 Uvicorn 进程，由 systemd、容器编排或同类进程管理器负责重启。
- 反向代理负责 HTTPS、域名、请求体限制和访问日志；ProjectLens 只监听内网端口。
- `PROJECT_LENS_DATABASE_PATH`、`PROJECT_LENS_GENAI_TRACE_DIR` 和 `.env` 应放在持久化、权限受控的目录，不要放入仓库或临时目录。
- 飞书 HTTP 回调和 WebSocket 长连接是两种互斥入口，只启用一种；部署 HTTP 回调时关闭 `PROJECT_LENS_FEISHU_LONG_CONNECTION_ENABLED`。
- 生产发布前必须通过 `/api/v1/health`、`/api/v1/ready` 和 `/api/v1/release-gate`，并配置 `PROJECT_LENS_SERVICE_TOKEN` 与 MCP 受信任身份。

## 配置方法

配置使用 `PROJECT_LENS_` 前缀，并从 `.env` 读取。建议复制 `.env.example` 后按环境修改，密钥不要提交。

### 最小本地配置

```dotenv
PROJECT_LENS_ENV=development
PROJECT_LENS_DATABASE_PATH=project_lens.db
PROJECT_LENS_CONVERSATION_STORE=sqlite
PROJECT_LENS_AGENT_MODE=hermes
PROJECT_LENS_PROJECT_SPACES_DIR=config/projects
PROJECT_LENS_MODEL_PROVIDER=stub
PROJECT_LENS_MODEL_LIVE=false
```

### 关键配置组

| 配置组 | 作用 | 默认行为 |
| --- | --- | --- |
| `DATABASE_PATH` / `CONVERSATION_STORE` | SQLite 和会话持久化 | SQLite、本地文件 |
| `MODEL_*` | 通用 ContextPrompt 模型 | stub、外部调用关闭 |
| `FEISHU_HERMES_*` | Hermes provider、模型和超时 | 生产问答使用 Hermes |
| `KNOWLEDGE_*` / `EMBEDDING_*` | 词法/向量/混合检索 | 词法、向量关闭 |
| `LOCAL_PROJECT_REGISTRY` | 内联项目注册 | 可为空，默认读取 JSON |
| `PROJECT_SPACES_DIR` | ProjectSpace JSON 目录 | `config/projects` |
| `SERVICE_TOKEN` | 服务间鉴权 | 按中间件策略校验 |
| `MCP_TRUSTED_*` | MCP 受信任调用者 | 建议生产必配 |
| `FEISHU_*` | 飞书应用、绑定和回调 | recording/off |
| `OBSIDIAN_*` / `FEISHU_WIKI_*` | 派生知识写入 | 默认关闭 |
| `FEISHU_HERMES_EXECUTION_ENABLED` | 工程执行路径 | 默认关闭 |

完整字段和示例以 [.env.example](.env.example) 为准，行为说明见 [配置与运行手册](docs/configuration-and-operations.md)。真实飞书、模型和 GitHub 凭据只能通过部署环境注入，不能写入 JSON、测试数据或文档。

## 接入项目

最小 ProjectSpace 位于 `config/projects/<project>.json`。建议至少定义：

```json
{
  "tenant_id": "demo",
  "project_id": "payment",
  "service": "order-service",
  "environment": "production",
  "repository_root": "examples/payment_service/src",
  "documents_root": "examples/payment_service/knowledge",
  "members": [
    {"actor_id": "user-1", "roles": ["developer"]}
  ],
  "public_sources": ["overview", "docs/public"]
}
```

真实项目还应配置 `feishu_chat_bindings`、角色策略、群聊可见性、连接器和来源白名单。每个飞书群只能绑定一个项目；未加入 `members` 的用户按 guest 范围处理。详细步骤见 [ProjectSpace 接入指南](docs/projectspace-onboarding-guide.md)。

## HTTP API

默认前缀为 `/api/v1`，运行中的 OpenAPI 是参数和响应的最终来源。

| 路径族 | 用途 |
| --- | --- |
| `/health`、`/ready`、`/release-gate` | 健康、依赖就绪和发布门禁 |
| `/projects/snapshot`、`timeline`、`change-impact` | 项目事实、时间线和变更影响 |
| `/projects/source-records`、`connectors/sync` | 来源记录和连接器同步 |
| `/projects/knowledge-gaps`、`fact-resolution` | 知识缺口和事实解析 |
| `/projects/knowledge-index/*` | 知识索引状态与同步 |
| `/project-agent/ask` | 受信任操作者的一次项目问答 |
| `/project-agent/tools`、`tools/call` | 工具发现和显式工具调用 |
| `/project-agent/runs/*` | 角色视图、运行详情和 Hermes 事件 |
| `/projects/*/memories`、`/memory-proposals/*` | 记忆检索、提议和审核 |
| `/runs/*` | AgentRun 创建、执行和事件 |
| `/hermes/*` | 工程审批和显式执行 |
| `/feishu/events` | 飞书事件回调 |

受保护的 Project Agent、记忆和写入接口需要可信 actor 或服务 token。不要把客户端传入的 `actor_id` 当作身份凭证。

## 飞书与 MCP

### 飞书

本地联调推荐 WebSocket 长连接：

```powershell
py -3.12 -m pip install -e ".[dev,feishu-ws]"
.\scripts\start-projectlens-feishu-ws.ps1
```

需要配置 App ID、App Secret、Verification Token、`FEISHU_PROJECT_BINDINGS`、测试群和成员 `open_id`。同一应用只能保持一条长连接；生产 HTTP 回调则使用 `POST /api/v1/feishu/events`，不要同时运行 WS ingress。

### MCP

安装 `.[mcp]` 后可使用 `project-lens-mcp`。MCP stdio 没有 HTTP middleware，因此生产环境应配置受信任的租户、操作者、群聊和类型。

## 数据、持久化与安全

SQLite 保存项目运行状态、会话、记忆、来源索引、审批和事件；GenAI trace 可按 `GENAI_TRACE_DIR` 保存为 JSON。生产环境不要使用 `:memory:`，并为数据库、trace 和 `.env` 设置访问权限与备份策略。

权限在检索前计算：

```text
用户角色权限 ∩ 会话/群聊可见范围 ∩ 项目资源权限
```

测试模式和 pytest 运行期间会阻断真实外部调用。写入和工程执行必须经过显式开关及审批链路。

## 测试与开发

```powershell
py -3.12 -m compileall -q src tests
py -3.12 -m pytest -q tests/test_project_intro.py tests/test_project_agent_ask_api.py tests/test_runtime_security.py tests/test_security_evaluation.py
```

按主题运行：

```powershell
py -3.12 -m pytest tests/test_project_intro.py tests/test_project_agent_ask_api.py -q
py -3.12 -m pytest tests/test_runtime_security.py tests/test_security_evaluation.py -q
py -3.12 -m pytest tests/test_feishu_http_and_acl.py tests/test_feishu_ws_ingress.py -q
```

公开仓库的 GitHub Actions 运行同样的编译检查和四组无外部凭据的 smoke tests。完整测试集包含需要额外环境或仍在整理中的历史用例，发布前应在目标环境单独执行并记录结果；Ruff 可用于后续增量清理，当前既有代码仍有 lint 技术债。

修改权限、检索、Agent 工具或外部集成时必须补回归测试。

## 故障排查

| 现象 | 优先检查 |
| --- | --- |
| `/ready` 不通过 | 数据库路径、ProjectSpace 加载、依赖和配置格式 |
| 回答像在答错项目 | tenant/project/chat 绑定是否唯一，ProjectSpace 是否加载 |
| 用户看不到来源 | `members`、角色、`public_sources` 和 chat visibility 求交结果 |
| 飞书无事件 | App 凭证、长连接模式、WS ingress 日志和重复连接 |
| 飞书卡片无响应 | `card.action.trigger` 回调和应用版本发布状态 |
| 只返回 stub | `MODEL_LIVE`、provider、API key 和 fallback 配置 |
| 向量检索未生效 | embedding provider、模型、向量开关和索引同步 |
| 追问丢上下文 | `CONVERSATION_STORE=sqlite` 且数据库路径一致 |
| 不能执行工程操作 | execution 开关、审批记录和服务权限 |

## 开源说明

- 许可证：[MIT License](LICENSE)
- 贡献方式：[CONTRIBUTING.md](CONTRIBUTING.md)
- 安全问题：[SECURITY.md](SECURITY.md)
- 公开仓库不承诺生产可用性、高可用性或特定行业合规结论。
- 开启真实外部服务前，必须重新配置项目空间、身份绑定、访问权限、持久化数据库和备份策略。

## 文档索引

- [项目说明](docs/project-introduction.md)
- [核心算法说明](docs/core-algorithm-description.md)
- [配置与运行手册](docs/configuration-and-operations.md)
- [架构与数据流](docs/architecture-and-data-flow.md)
- [API 与集成指南](docs/api-and-integrations.md)
- [Wiki、上下文、记忆与知识闭环](docs/features-knowledge-memory-context.md)
- [ProjectSpace 接入](docs/projectspace-onboarding-guide.md)
- [飞书快速上手](docs/feishu-test-quickstart.md)
- [飞书配置清单](docs/feishu-setup.md)
- [文档总入口](docs/README.md)

项目当前版本为 `0.1.0`，处于持续开发阶段。部署或对外演示前，先运行测试并检查 `/ready` 与 `/release-gate`。
