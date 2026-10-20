# ProjectLens

ProjectLens 是一个面向软件研发团队的证据驱动项目上下文与问答服务。

它把代码仓库、项目文档、任务、发布记录、运行信号、会话历史和项目记忆组织成一个受权限约束的 ProjectSpace，再由 Hermes Agent 使用只读工具调查并生成可追溯回答。

ProjectLens 关注的不是“让模型知道更多”，而是让每次回答都能说明：

- 当前回答针对哪个项目；
- 当前操作者可以看到哪些来源；
- 回答中的事实来自哪些证据；
- 哪些内容是推断、未知或存在冲突；
- 后续需要谁补充什么信息。

这是一个 MIT 许可的开源项目原型，当前版本为 `0.1.0`。公开仓库包含核心代码、示例数据、测试和技术文档，不包含生产凭据、真实项目配置、运行数据库、Obsidian 状态或内部过程材料。

## 为什么需要 ProjectLens

研发项目的信息通常分散在不同系统中：

```text
代码仓库 + 项目文档 + 任务记录 + 发布记录 + 运行信号 + 群聊 + 历史会话
```

普通聊天机器人通常只能根据输入上下文生成答案，无法可靠判断：

- 这些资料是否属于当前项目；
- 当前用户是否有权查看这些内容；
- 信息是否已经过期；
- 一个结论到底来自哪条证据；
- 没有证据时应该回答“不知道”，还是继续猜测。

ProjectLens 把这些问题放进运行时流程中处理：先解析身份和项目绑定，再计算有效访问范围，随后检索证据、驱动 Agent 工具循环、校验事实和未知项，最后保存运行轨迹与审计信息。

## 适合什么场景

### 项目状态问答

例如：

- 这个项目当前是做什么的？
- 最近一次发布改了哪些内容？
- 当前有哪些开放风险？
- 哪些需求已经实现，哪些还没有证据？

### 代码与文档调查

例如：

- `create_order` 为什么要检查 `coupon`？
- 订单服务的主要依赖是什么？
- 某个变更会影响哪些模块？
- 这个问题在代码、文档和任务中分别有什么记录？

### 项目知识治理

例如：

- 哪些知识缺口反复出现？
- 哪些记忆需要人工审核？
- 哪些事实之间存在冲突？
- 哪些来源还没有同步或缺少时间信息？

### 团队入口整合

ProjectLens 当前提供以下入口：

- FastAPI HTTP API；
- 飞书 HTTP 事件回调；
- 飞书 WebSocket 长连接；
- MCP stdio server。

## 一个回答长什么样

用户提出问题后，ProjectLens 不只返回一段模型文本，而是生成一个回答信封（answer envelope）。典型结构如下：

```json
{
  "ok": true,
  "project": {
    "tenant_id": "demo",
    "project_id": "payment"
  },
  "answer_summary": "订单服务通过 coupon 参数决定折扣策略。当前证据显示该检查用于避免无效优惠券进入结算流程。",
  "facts": [
    {
      "text": "create_order 会先校验 coupon，再计算订单金额。",
      "citations": [
        {
          "source_id": "repo:payment_service:src/order_service.py"
        }
      ]
    }
  ],
  "inferences": [
    "该校验可能用于保持结算阶段的输入稳定。"
  ],
  "unknowns": [
    "当前示例资料没有说明优惠券规则由哪个业务角色维护。"
  ],
  "citations": [],
  "audit_ref": {
    "allow_apply": false,
    "agent_mode": "hermes"
  }
}
```

字段含义：

| 字段 | 含义 |
| --- | --- |
| `facts` | 有来源支持的事实，事实项应带引用 |
| `inferences` | 基于证据的推断，不伪装成原始事实 |
| `unknowns` | 当前资料无法确认的内容 |
| `conflicts` | 不同来源之间的矛盾 |
| `citations` | 本次回答使用的来源摘要 |
| `audit_ref` | 运行模式、权限和执行边界等审计信息 |

## ProjectLens 与普通 RAG 的区别

普通 RAG 通常是“向量或关键词检索 -> 拼接文本 -> 生成回答”。ProjectLens 在这条链路上增加了项目和权限边界：

| 普通 RAG | ProjectLens |
| --- | --- |
| 主要按文本相似度召回 | 先确定租户、项目、用户和会话范围 |
| 召回内容未必属于当前项目 | 来源必须落在 ProjectSpace 和允许来源内 |
| 模型直接总结检索文本 | Hermes 通过受控只读工具调查 |
| 事实、推断和猜测容易混在一起 | 分离事实、推断、未知和冲突 |
| 通常只返回最终文本 | 保存工具调用、证据引用、运行事件和审计信息 |
| 权限常由上层应用另行处理 | 权限在检索前计算，并贯穿工具调用和回答回放 |

ProjectLens 不是为了让模型代替项目系统，而是让 Agent 在项目系统已有事实的范围内工作。

## 核心概念

### ProjectSpace

ProjectSpace 是一个项目的边界定义，描述：

- 租户和项目标识；
- 代码仓库和文档目录；
- 任务、发布、运行信号等来源；
- 飞书群绑定；
- 成员和角色；
- 允许读取的来源与路径；
- 角色可以使用的工具；
- 检索、记忆和知识索引命名空间。

公开示例位于 `config/projects/payment.json`，对应的示例代码和资料位于 `examples/payment_service/`。

### Evidence 与 SourceRecord

外部来源不会直接作为一段无来源文本注入模型，而是被整理为带有来源、标识、时间和访问范围的记录。回答使用这些记录生成引用，便于追溯“这句话来自哪里”。

### Hermes Agent

Hermes 是生产问答路径中的 Agent runtime。ProjectLens 提供：

- 当前项目和身份上下文；
- 有效访问范围；
- 受策略约束的只读工具；
- 上下文预算；
- 事实验证和审计记录。

Hermes 负责模型调用和工具循环，但不能通过提示词或请求参数绕过 ProjectLens 的项目与权限边界。

### Knowledge Gap 与 Memory

回答中的未知项可以进入知识缺口流程，供项目成员补充和跟踪。长期记忆先进入候选或审核队列，不会因为一次模型回答就自动成为永久事实。

## 工作流程

```text
请求入口
  -> 身份、租户、会话和项目解析
  -> ProjectSpace 与群聊绑定校验
  -> 角色权限 ∩ 群聊可见性 ∩ 来源白名单
  -> 证据、记忆、历史和知识索引检索
  -> Hermes 只读工具循环
  -> 事实 / 推断 / 未知 / 冲突校验
  -> API、MCP 或飞书回答
  -> 运行事件、引用、反馈和知识缺口持久化
```

## 快速开始

### 环境要求

- Python 3.12+
- Git
- Windows PowerShell、Linux shell 或 macOS shell

本地默认使用 SQLite、示例 ProjectSpace 和 Stub provider，不需要真实模型、飞书或 GitHub 凭据即可启动服务。

### 安装和启动

Windows PowerShell：

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
py -3.12 -m pip install -e ".[dev,mcp]"
Copy-Item .env.example .env
py -3.12 -m uvicorn project_lens.main:app --host 127.0.0.1 --port 8000 --workers 1
```

Linux/macOS：

```bash
python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -e ".[dev,mcp]"
cp .env.example .env
python -m uvicorn project_lens.main:app --host 127.0.0.1 --port 8000 --workers 1
```

启动后打开：

- Swagger UI：`http://127.0.0.1:8000/docs`
- 健康检查：`GET /api/v1/health`
- 就绪检查：`GET /api/v1/ready`
- 发布门禁：`GET /api/v1/release-gate`

PowerShell 验证：

```powershell
Invoke-RestMethod http://127.0.0.1:8000/api/v1/health
Invoke-RestMethod http://127.0.0.1:8000/api/v1/ready
Invoke-RestMethod http://127.0.0.1:8000/api/v1/release-gate
```

### 发起第一次项目问答

开发环境下，Project Agent 路由可以使用可信身份请求头进行本地测试。下面的请求使用仓库中的 `demo/payment` 示例项目：

```powershell
$headers = @{
  "X-ProjectLens-Actor-Id" = "u1"
  "X-ProjectLens-Chat-Id" = "chat-1"
  "X-ProjectLens-Tenant-Key" = "demo"
  "X-ProjectLens-Chat-Type" = "group"
}

$body = @{
  question = "order_service.py 里 create_order 为什么要检查 coupon？"
  project = @{
    tenant_id = "demo"
    project_id = "payment"
    service = "order-service"
    environment = "production"
  }
  user_id = "u1"
  channel_id = "chat-1"
  audience = "team"
  mode = "read_only"
  format = "concise"
} | ConvertTo-Json -Depth 5

Invoke-RestMethod `
  -Method Post `
  -Uri http://127.0.0.1:8000/api/v1/project-agent/ask `
  -Headers $headers `
  -ContentType "application/json" `
  -Body $body
```

这些身份请求头只适用于本地开发/测试。Pilot 或 production 环境必须配置 `PROJECT_LENS_SERVICE_TOKEN`，并通过部署侧身份系统提供可信 actor 上下文；不能让客户端自由填写 `actor_id` 来获得权限。

## 配置方法

配置使用 `PROJECT_LENS_` 前缀，并从 `.env` 读取。推荐复制 `.env.example` 后按环境修改，密钥不要提交。

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

### 常用配置组

| 配置组 | 作用 | 本地默认行为 |
| --- | --- | --- |
| `DATABASE_PATH` / `CONVERSATION_STORE` | SQLite 和会话持久化 | SQLite、本地文件 |
| `MODEL_*` | 通用模型 provider、模型和外呼开关 | Stub、外部调用关闭 |
| `FEISHU_HERMES_*` | Hermes provider、模型和超时 | 生产问答使用 Hermes |
| `KNOWLEDGE_*` / `EMBEDDING_*` | 词法、向量和混合检索 | 词法开启、向量关闭 |
| `LOCAL_PROJECT_REGISTRY` | 以内联 JSON 注册项目 | 可为空，默认读取项目目录 |
| `PROJECT_SPACES_DIR` | ProjectSpace JSON 目录 | `config/projects` |
| `SERVICE_TOKEN` | 内部服务 API 鉴权 | pilot/production 建议必配 |
| `MCP_TRUSTED_*` | MCP 受信任调用者身份 | 留空时对 MCP 请求失败关闭 |
| `FEISHU_*` | 飞书应用、绑定和回调 | recording/off |
| `OBSIDIAN_*` / `FEISHU_WIKI_*` | 派生知识写入 | 默认关闭 |
| `FEISHU_HERMES_EXECUTION_ENABLED` | 工程执行路径 | 默认关闭 |

完整字段和示例以 [.env.example](.env.example) 为准，行为说明见[配置与运行手册](docs/configuration-and-operations.md)。真实飞书、模型和 GitHub 凭据只能通过部署环境注入，不能写入 JSON、测试数据或文档。

## 接入自己的项目

### 最小 ProjectSpace

新建 `config/projects/<project>.json`，最小结构可以是：

```json
{
  "tenant_id": "demo",
  "project_id": "payment",
  "display_name": "Payment Service",
  "repositories": [
    {
      "name": "payment_service",
      "path": "examples/payment_service",
      "default_branch": "main"
    }
  ],
  "documents_root": "examples/payment_service/knowledge",
  "members": [
    {"actor_id": "user-1", "roles": ["developer"]}
  ],
  "public_sources": ["knowledge/"],
  "file_allowlist": ["src/", "tests/", "knowledge/"]
}
```

真实项目通常还需要配置：

- `services` 和 `environments`；
- `source_connectors`；
- `feishu_chat_bindings`；
- 角色策略和允许工具；
- 任务、发布、运行信号等来源；
- `access_scope`、`public_sources` 和文件白名单；
- 检索、记忆和知识索引命名空间。

每个飞书群只能绑定一个项目；没有加入成员列表的用户按 guest 范围处理。完整接入步骤见 [ProjectSpace 接入指南](docs/projectspace-onboarding-guide.md)。

## HTTP API

默认前缀为 `/api/v1`，运行中的 OpenAPI 文档是参数和响应的最终来源。

### 常用接口

| 路径 | 用途 |
| --- | --- |
| `/health` | 进程和基础依赖是否可响应 |
| `/ready` | 数据库、ProjectSpace 和依赖是否就绪 |
| `/release-gate` | 当前配置是否满足发布门禁 |
| `/projects/snapshot` | 项目当前事实快照 |
| `/projects/timeline` | 项目时间线 |
| `/projects/change-impact` | 变更影响分析 |
| `/projects/source-records` | 来源记录查询 |
| `/projects/knowledge-gaps` | 知识缺口查询和状态处理 |
| `/project-agent/ask` | 创建并执行一次项目问答运行 |
| `/project-agent/tools` | 查看当前可用的受控工具 |
| `/project-agent/tools/call` | 显式调用单个受控工具 |
| `/project-agent/runs/*` | 查看运行详情和角色视图 |
| `/runs/*` | 查看 AgentRun 和运行事件 |
| `/feishu/events` | 飞书 HTTP 事件回调 |

### `/project-agent/ask` 请求结构

```json
{
  "question": "这个项目最近的主要风险是什么？",
  "project": {
    "tenant_id": "demo",
    "project_id": "payment",
    "service": "order-service",
    "environment": "production"
  },
  "user_id": "u1",
  "channel_id": "chat-1",
  "audience": "team",
  "mode": "read_only",
  "format": "concise"
}
```

接口会返回 `run_id` 和 `trace_id`，并在成功时提供 `answer_summary`、`facts`、`inferences`、`unknowns`、`conflicts`、`citations` 和 `audit_ref`。一次问答不是无状态的字符串生成，而是可以通过运行接口继续查看的 AgentRun。

受保护的 Project Agent、记忆、Hermes 和写入接口需要可信 actor 或服务 token。不要把客户端传入的 `user_id`、`actor_id` 或 `channel_id` 当作身份凭证。

## 飞书与 MCP

### 飞书

飞书支持两种互斥入口：

1. HTTP 事件回调：部署在有公网 HTTPS 地址的服务后面；
2. WebSocket 长连接：适合本地联调或不方便暴露回调地址的环境。

本地长连接联调：

```powershell
py -3.12 -m pip install -e ".[dev,feishu-ws]"
.\scripts\start-projectlens-feishu-ws.ps1
```

需要自行配置 App ID、App Secret、Verification Token、`FEISHU_PROJECT_BINDINGS`、测试群和成员 `open_id`。同一应用只能保持一条长连接；生产 HTTP 回调使用 `POST /api/v1/feishu/events`，不要同时运行 WS ingress。

### MCP

安装 `.[mcp]` 后可使用 `project-lens-mcp`。MCP stdio 没有 HTTP middleware，因此生产环境应配置 `MCP_TRUSTED_TENANT_ID`、`MCP_TRUSTED_ACTOR_ID`、`MCP_TRUSTED_CHAT_ID` 和 `MCP_TRUSTED_CHAT_TYPE`。身份未配置时应保持 fail closed。

## 部署边界

当前公开版本适合单机或单台 VM 部署，推荐架构如下：

```text
客户端 / 飞书
        ↓ HTTPS
反向代理（TLS、域名、访问日志、请求限制）
        ↓ 内网
单个 Uvicorn worker
        ↓
SQLite + 持久化 trace + ProjectSpace 配置
```

必须注意：

- 当前仓库没有 serverless、Docker 或云平台专用部署配置；
- SQLite、进程内调度器和飞书长连接不适合直接横向扩展为多副本；
- 生产入口只运行一个 Uvicorn worker，并由 systemd、容器编排或同类进程管理器负责重启；
- `PROJECT_LENS_DATABASE_PATH`、`PROJECT_LENS_GENAI_TRACE_DIR` 和 `.env` 应放在持久化、权限受控的目录；
- 飞书 HTTP 回调和 WebSocket 长连接只能启用一种；
- HTTP 回调部署时监听 `0.0.0.0`，但服务本身不应直接承担公网 TLS；
- 上线前通过 `/api/v1/health`、`/api/v1/ready` 和 `/api/v1/release-gate`；
- pilot/production 配置 `PROJECT_LENS_SERVICE_TOKEN`，并配置 MCP 受信任身份；
- 生产不要使用 `:memory:` 数据库，也不要把 `.env`、数据库或 trace 提交到 Git。

## 数据、安全和权限

权限在检索和工具调用之前计算：

```text
用户角色权限 ∩ 会话/群聊可见范围 ∩ 项目资源权限 ∩ 来源白名单
```

系统默认遵循以下边界：

- 不把模型记忆当作项目事实；
- 不允许用户通过提示词或工具参数伪造身份、项目或权限；
- 不默认改代码、开 PR、部署、回滚或重启服务；
- 写入和工程执行需要独立开关与审批链路；
- 测试模式和 pytest 运行期间阻断真实外部调用；
- `.env`、数据库、trace、真实 token 和真实业务资料不能进入公开仓库。

发现凭据泄露、越权读取、跨项目数据泄露或未审批写入问题，请参阅 [SECURITY.md](SECURITY.md)，不要在公开 Issue 中提交敏感信息。

## 代码结构

```text
src/project_lens/  API、Agent、应用服务、上下文检索、项目空间、运行时与集成
config/projects/   ProjectSpace JSON 示例
examples/          示例项目代码与知识数据
tests/             单元、接口、集成和安全回归
scripts/           启动、冒烟和真实联调脚本
docs/              架构、配置、接入和开发文档
evaluation/        脱敏评测配置和数据集
```

主要模块：

| 模块 | 责任 |
| --- | --- |
| `src/project_lens/api/` | FastAPI 路由、schema 和鉴权中间件 |
| `src/project_lens/application/` | 项目问答、知识缺口、记忆、运行和集成编排 |
| `src/project_lens/context/` | 证据、来源、检索、快照和上下文构建 |
| `src/project_lens/project_space/` | 项目注册、角色策略、可见性和访问范围 |
| `src/project_lens/integrations/` | 飞书、Hermes、MCP、GitHub 等适配器 |
| `src/project_lens/persistence/` | SQLite 和运行数据持久化 |

## 测试与开发

公开仓库的 GitHub Actions 运行编译检查和不需要外部凭据的 smoke tests：

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

完整测试集包含需要额外环境或仍在整理中的历史用例，发布前应在目标环境单独执行并记录结果。Ruff 可用于后续增量清理，当前既有代码仍有 lint 技术债。

涉及权限、检索、Agent 工具、飞书或外部服务的改动必须补回归测试。贡献流程见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 已知限制

- 当前版本是开源项目原型，不承诺生产级高可用或特定行业合规结论；
- 示例 ProjectSpace、代码和知识文件只是演示数据，不代表真实项目配置；
- SQLite 和进程内调度器限制了多副本部署方式；
- 向量检索、真实模型、飞书消息、Wiki 发布、Obsidian 导出和工程执行都由独立配置控制；
- 设计文档中标记为“规划中”“待实施”或“目标架构”的内容不等于当前已实现能力；
- 当文档与运行中的 OpenAPI 或配置模型冲突时，以代码和接口行为为准。

## 文档导航

按目标查找：

| 目标 | 文档 |
| --- | --- |
| 了解产品和当前实现 | [项目说明](docs/project-introduction.md) |
| 理解系统结构 | [架构与数据流](docs/architecture-and-data-flow.md) |
| 配置和运行服务 | [配置与运行手册](docs/configuration-and-operations.md) |
| 接入一个新项目 | [ProjectSpace 接入指南](docs/projectspace-onboarding-guide.md) |
| 接入飞书 | [飞书快速上手](docs/feishu-test-quickstart.md) 和 [飞书配置清单](docs/feishu-setup.md) |
| 理解检索、记忆和知识闭环 | [功能说明](docs/features-knowledge-memory-context.md) |
| 理解核心算法 | [核心算法说明](docs/core-algorithm-description.md) |
| 参与开发 | [CONTRIBUTING.md](CONTRIBUTING.md) |
| 报告安全问题 | [SECURITY.md](SECURITY.md) |

## 开源说明

- 许可证：[MIT License](LICENSE)
- 公开仓库不承诺生产可用性、高可用性或特定行业合规结论；
- 开启真实外部服务前，必须重新配置项目空间、身份绑定、访问权限、持久化数据库和备份策略；
- 公开仓库不包含生产凭据和内部比赛材料。

## License

ProjectLens is released under the [MIT License](LICENSE).
