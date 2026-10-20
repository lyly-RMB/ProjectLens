# ProjectLens

ProjectLens 是一个面向研发团队的**项目上下文管理与知识闭环系统**。

它不只是把资料交给一个聊天模型，而是管理一条更严格的链路：

```text
项目来源
  -> 证据与索引
  -> 上下文管理器
  -> Hermes 只读调查
  -> 事实、未知与冲突
  -> 知识缺口 / 记忆提案
  -> 审核、补源、重新检查
  -> 更可靠的下一轮上下文
```

ProjectLens 的核心价值是：**控制模型本轮能看到什么，保存哪些内容可以成为长期知识，并把回答中的“不确定”转化为可以继续处理的知识缺口。**

这是一个 MIT 许可的开源项目原型，当前版本为 `0.1.0`。公开仓库包含核心代码、示例数据、测试和技术文档，不包含生产凭据、真实项目配置、运行数据库或内部过程材料。

## 三个核心系统

### 1. 上下文管理器：决定本轮应该带什么

上下文管理器解决的不是“如何把更多文本塞进模型”，而是：

> 对当前问题、当前项目和当前操作者来说，哪些信息值得进入这一次推理？

它管理三类上下文：

```text
会话上下文：当前问题、历史轮次、摘要和用户明确选择的内容
证据上下文：本次检索得到的代码、文档、任务、发布和运行证据
长期上下文：经过权限校验和生命周期管理的 ProjectMemory
```

上下文管理器的主要职责：

- 根据项目、用户角色、群聊可见性和来源白名单计算可用范围；
- 为系统提示、历史、问题、工具 schema、工具结果和输出预留分配 token 预算；
- 对重复内容去重，对历史内容摘要，对低优先级内容裁剪，必要时旋转旧上下文；
- 保留项目边界、权限结果、关键证据、正式记忆和用户明确选择的内容；
- 在运行开始时冻结实际使用的上下文，并保存内容 hash、来源引用和渲染版本；
- 如果运行期间上下文、权限或记忆发生变化，返回显式变更错误，而不是静默替换输入。

核心实现包括：

- `ContextGovernor`：请求级 token 预算、去重、摘要、裁剪和阻断；
- `ContextSnapshot`：本次运行实际使用内容的冻结快照；
- `ContextPack` / `ContextPrompt`：把身份、证据、记忆和任务组织成模型输入；
- `ConversationSession`：保存跨轮会话、历史摘要和上下文引用。

这使得 ProjectLens 可以回答“模型为什么看到了这些信息”，也可以在审计或重放时复原本次运行的上下文边界。

#### 上下文压缩的两层机制

ProjectLens 不把上下文压缩简化为“只保留最后几条消息”，而是分成会话层和请求层两步处理。

**会话层：管理跨轮历史。**

- 默认保留最近 8 轮用户问题；
- 写入第 9 轮及之后的内容时，压缩当前池中最旧的第 1～4 轮；
- 压缩结果保留用户原问题、回答摘要和 Evidence 标识，原始轮次移出当前文本池但不丢失历史引用；
- 默认不因为跨日就删除相关上下文，日期窗口由用户主动设置；
- 用户可以引用历史对话或群聊消息，引用项保留时间、短标题和取回键，正文按需读取；
- 群聊历史不默认全量落本地，细选的群聊内容通过来源接口按需获取。

**请求层：管理本次模型调用的 token 预算。**

Context Governor 会估算系统提示、历史、当前问题、工具 schema、工具结果、输出预留和运行开销。接近预算时按以下顺序处理：

```text
去重 -> 摘要 -> 裁剪低优先级内容 -> 轮次旋转 -> 必要时阻断超预算请求
```

压缩时优先保留：

- 当前问题和用户明确选择的内容；
- 项目边界、身份和权限结果；
- 关键 Evidence、正式 ProjectMemory 和事实标识；
- 决策、任务、负面约束和来源时间。

压缩后的内容仍保留 `fact_id`、`evidence_id`、来源 key、时间和事实类型，避免摘要把推断变成事实。运行级 `ContextSnapshot` 用于冻结本次实际输入、内容 hash 和授权结果，服务于审计与重放；它不是把所有历史永久塞进模型，也不等同于用户编辑历史的界面对象。

### 2. 知识缺口：把“不知道”变成可处理对象

普通 Agent 遇到资料不足时，通常只返回一句“无法确定”。ProjectLens 会把有治理价值的未知转化为结构化 Knowledge Gap：

```text
当前问题 / 检索结果
  -> 缺少来源、来源冲突、来源过期或缺少负责人
  -> 生成稳定的 Knowledge Gap
  -> 聚合重复观察
  -> 分配负责人或提出补充动作
  -> 同步来源、重新索引、recheck
  -> resolved 或 reopened
```

当前缺口类型包括：

- `missing_source`：没有找到必要来源；
- `conflicting_sources`：多个来源互相矛盾；
- `stale_source`：已有来源可能已经过期；
- `missing_owner`：缺少负责人；
- `missing_decision`：缺少决策记录；
- `missing_schedule`：缺少时间安排；
- `missing_acceptance`：缺少验收标准；
- `access_limited`：信息可能存在，但当前权限无法查看；
- `unclear_question`：问题本身还不足以生成可靠调查。

Knowledge Gap 不是简单的 TODO 列表。它具有：

- 稳定的 `canonical_key`，相同问题会累积观察而不是重复建单；
- 项目、问题、影响、严重程度、证据状态和来源引用；
- 负责人、建议来源类型和关联任务；
- `candidate -> open -> assigned -> waiting_for_source -> resolved_pending_check -> resolved/reopened` 生命周期；
- 每次状态变化、观察、动作和重新检查的审计记录；
- 按项目、负责人和周期生成的 Digest。

关闭缺口不等于有人点击了“已处理”。只有补充来源、重新索引并完成 recheck 后，缺口才可以进入 `resolved`；如果新证据仍不能解决问题，系统会重新打开它。

### 3. 记忆系统：让长期知识可审核、可撤销、可追溯

ProjectLens 不把模型回答直接写入长期记忆。记忆分为不同层级：

```text
MemoryObservation / Episode
    -> 运行观察或一次讨论摘要，不是已确认事实

MemoryProposal
    -> 等待证据、权限、冲突和人工决策检查的候选记忆

ProjectMemory
    -> 经过审批、可按权限和有效期检索的长期项目知识
```

长期记忆可以表示：

- 架构事实和技术决策；
- 业务规则和团队约定；
- 负责人、风险和 Runbook；
- 需求变更、复盘结论和发布约束。

每条正式记忆都携带项目、类型、正文、来源、Evidence、可见范围、批准人、生效时间、失效时间和状态。常见状态包括：

- `active`：可以在授权范围内进入检索结果；
- `needs_review`：需要重新审核，不能当作稳定事实使用；
- `revoked`：已撤销，不再参与正常检索；
- `superseded`：已经被新版本替代。

记忆检索的顺序是：

```text
候选召回
  -> 项目、角色、visibility scope、有效期和状态校验
  -> 词法 / 向量 / 混合排序
  -> 预算控制和上下文组装
```

向量相似度只能帮助排序，不能绕过 ACL。来源失效、证据被撤销、有效期到期、出现冲突或收到反馈时，系统可以创建 MemoryReview，要求更新、撤销或替代原记忆。

## 三个系统如何协同

ProjectLens 的重点不是三个孤立模块，而是它们之间的反馈关系：

```text
代码 / 文档 / 任务 / 发布 / 运行信号 / 飞书资料
                    |
                    v
          SourceRecord + Evidence + Index
                    |
                    v
       身份、项目和权限范围计算
                    |
                    v
       ContextGovernor / ContextSnapshot
                    |
                    v
       Hermes 只读工具调查与回答
                    |
          +---------+---------+
          |                   |
          v                   v
   MemoryProposal       KnowledgeGap
          |                   |
          v                   v
      人工审核          补源 / 分配 / recheck
          |                   |
          +---------+---------+
                    v
          更新下一轮可用知识
```

这条链路解决三个常见问题：

1. **上下文污染**：历史太多、资料重复或过期，导致模型看到不该看到的内容；
2. **知识沉淀失控**：一次不可靠回答被误当成长期事实；
3. **问题无法闭环**：系统知道资料不足，却没有负责人、动作和重新验证过程。

## 一个典型流程

以“订单服务的优惠券逻辑为什么这样实现”为例：

1. ProjectSpace 确定当前用户属于 `payment` 项目，并限制可读取的仓库和知识来源；
2. 检索器从代码、任务和项目文档中找到相关 Evidence；
3. 上下文管理器去重、排序并控制 token 预算，冻结本轮上下文；
4. Hermes 使用只读工具继续调查实现、需求和历史变更；
5. 回答验证器区分已确认事实、推断和仍未知的内容；
6. 如果发现“没有记录谁维护优惠券规则”，系统创建 `missing_owner` 或 `missing_decision` 缺口；
7. 如果用户确认了新的稳定规则，系统先创建 `MemoryProposal`，经过审批后才进入 `ProjectMemory`；
8. 来源补齐后执行 recheck，缺口才会关闭，新的记忆才会影响后续上下文。

## ProjectSpace：所有能力的项目边界

ProjectSpace 是上下文、缺口和记忆共同使用的边界。它定义：

- 租户和项目标识；
- 仓库、文档和外部来源；
- 成员、角色和访问范围；
- 飞书群绑定和可见性；
- 文件白名单和允许工具；
- 检索、记忆和知识索引命名空间。

示例配置位于 `config/projects/payment.json`，示例代码和资料位于 `examples/payment_service/`：

```json
{
  "tenant_id": "demo",
  "project_id": "payment",
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

## 权限机制：先确定能看什么，再开始检索

ProjectLens 的权限设计不是在模型生成回答后做文字过滤，而是在信息进入候选集之前计算有效访问范围。可信身份由入口侧提供，客户端传入的 `actor_id`、`tenant_id` 或 `project_id` 不能自行改变权限。

### 有效访问范围

系统将以下边界求交，得到 `EffectiveAccessScope`：

```text
EffectiveAccessScope
  = 用户角色权限
  ∩ 会话 / 群聊可见范围
  ∩ ProjectSpace 项目范围
  ∩ 来源 ACL
  ∩ 记忆可见范围
```

只有通过范围计算的 SourceRecord、Evidence、ProjectMemory 和工具结果，才可以进入检索排序和模型上下文。这避免了“先把敏感内容召回，再在回答阶段过滤”的风险。

### 权限检查的运行顺序

```text
TrustedActor
  -> 租户 / 项目 / 群聊一致性校验
  -> ProjectSpace 解析
  -> 角色策略与群聊可见性求交
  -> 来源和文件白名单过滤
  -> 记忆 visibility / status / 有效期过滤
  -> 受控工具白名单与参数校验
  -> 角色化事实投影和回答
```

这条链路有几个关键原则：

- **召回前过滤**：无权限内容不会先进入检索结果、排序器或上下文；
- **工具也受权限约束**：角色允许读取哪些来源、使用哪些工具，由服务端的有效范围决定；
- **记忆继承权限**：写入长期记忆不会扩大它的可见范围，记忆检索仍检查项目、角色、visibility、状态和有效期；
- **角色只改变投影**：系统先维护统一事实，再按产品、研发、测试、项目经理或管理者角色过滤字段、调整关注点和解释深度，不复制多套事实；
- **读写分离**：普通问答只使用只读工具，Wiki 发布、记忆审批、Obsidian 写入和工程执行需要独立开关及审批；
- **身份不信任请求体**：请求中的用户和项目字段只是业务参数，不能替代入口侧可信身份；
- **每个边界都可审计**：有效范围、工具调用、来源引用、记忆状态和拒绝原因写入运行事件。

因此，ProjectLens 的权限机制保护的不是只有页面展示，而是保护“什么可以被召回、什么可以进入上下文、什么可以被工具读取、什么可以沉淀为长期知识”。

## 快速开始

要求 Python 3.12+。本地默认使用 SQLite、示例 ProjectSpace 和 Stub provider，不需要真实模型或飞书凭据。

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

启动后可以访问 `http://127.0.0.1:8000/docs`，并检查：

```text
GET /api/v1/health
GET /api/v1/ready
GET /api/v1/release-gate
```

完整配置见 [.env.example](.env.example)，项目接入步骤见 [ProjectSpace 接入指南](docs/projectspace-onboarding-guide.md)。

## 接入方式

ProjectLens 可以作为知识和上下文服务被不同入口调用：

- **FastAPI**：项目快照、检索、问答、记忆和 Knowledge Gap API；
- **Hermes**：生产问答的 Agent runtime，通过 ProjectLens 的只读工具调查；
- **飞书**：支持 HTTP 事件回调和 WebSocket 长连接；
- **MCP**：通过 `project-lens-mcp` 暴露受信任的 ProjectLens 工具。

飞书 HTTP 回调和 WebSocket 长连接是互斥入口，只启用一种。MCP 和生产 HTTP API 都必须配置可信身份或服务 token，不能让客户端自行声明 actor、项目或群聊权限。

## 安全边界与部署限制

权限在检索和工具调用前计算：

```text
用户角色权限 ∩ 会话/群聊可见范围 ∩ 项目资源权限 ∩ 来源白名单
```

默认行为：

- 未经证据和审批，回答不会直接成为正式记忆；
- 不允许通过提示词或工具参数扩大读取范围；
- 不默认改代码、开 PR、部署、回滚或重启服务；
- 真实外部调用、Wiki 发布、Obsidian 导出和工程执行都由独立开关控制；
- `.env`、数据库、trace、真实 token 和业务资料不能提交到 Git。

当前版本适合单机或单台 VM 部署。SQLite、进程内调度器和飞书长连接不适合直接扩展为多副本服务；生产环境使用单个 Uvicorn worker、HTTPS 反向代理和持久化数据库目录。详细说明见[配置与运行手册](docs/configuration-and-operations.md)。

## 代码入口

```text
src/project_lens/context/       证据、检索、上下文预算、记忆检索和存储
src/project_lens/workflow/      ContextPack、ContextPrompt、ContextSnapshot
src/project_lens/domain/        ProjectMemory、MemoryProposal、KnowledgeGap 模型
src/project_lens/application/   问答、记忆、知识缺口、recheck 和运行编排
src/project_lens/project_space/ 项目注册、角色策略、可见性和访问范围
src/project_lens/api/           HTTP API、schema 和鉴权中间件
tests/                          上下文、记忆、知识缺口、权限和接口回归测试
docs/                           架构、配置、接入和开发文档
```

建议从以下代码开始阅读：

- 上下文预算：`src/project_lens/context/governor.py`
- 上下文冻结：`src/project_lens/workflow/context_snapshot.py`
- 上下文组装：`src/project_lens/workflow/context_pack.py`
- 记忆模型：`src/project_lens/domain/memory.py`
- 记忆检索与授权：`src/project_lens/context/memory_retrieval.py`、`src/project_lens/context/memory_authorization.py`
- 知识缺口模型：`src/project_lens/domain/knowledge_gap.py`
- 知识缺口编排：`src/project_lens/application/knowledge_gap_service.py`

## 测试与开发

公开仓库的 GitHub Actions 运行编译检查和不需要外部凭据的 smoke tests：

```powershell
py -3.12 -m compileall -q src tests
py -3.12 -m pytest -q tests/test_project_intro.py tests/test_project_agent_ask_api.py tests/test_runtime_security.py tests/test_security_evaluation.py
```

核心能力相关测试：

```powershell
py -3.12 -m pytest tests/test_context_engine.py tests/test_context_governor.py tests/test_context_snapshot.py -q
py -3.12 -m pytest tests/test_memory* tests/test_knowledge_gap* -q
```

完整测试集包含需要额外环境或仍在整理中的历史用例，发布前应在目标环境单独执行并记录结果。涉及权限、上下文、记忆、检索或 Knowledge Gap 的改动必须补回归测试。贡献流程见 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 文档导航

| 主题 | 文档 |
| --- | --- |
| 核心能力总览 | [知识、上下文、记忆与 Knowledge Gap](docs/features-knowledge-memory-context.md) |
| 上下文与证据算法 | [核心算法说明](docs/core-algorithm-description.md) |
| 配置和运行 | [配置与运行手册](docs/configuration-and-operations.md) |
| 接入项目 | [ProjectSpace 接入指南](docs/projectspace-onboarding-guide.md) |
| 飞书接入 | [飞书快速上手](docs/feishu-test-quickstart.md) 和 [飞书配置清单](docs/feishu-setup.md) |
| 架构总览 | [架构与数据流](docs/architecture-and-data-flow.md) |
| 开发贡献 | [CONTRIBUTING.md](CONTRIBUTING.md) |
| 安全问题 | [SECURITY.md](SECURITY.md) |

## 当前边界

- 这是开源项目原型，不承诺生产级高可用或特定行业合规结论；
- 示例代码、ProjectSpace 和知识文件只是演示数据；
- 设计文档中标记为“规划中”“待实施”或“目标架构”的内容不等于当前已实现能力；
- 向量检索、真实模型、飞书消息、Wiki 发布、Obsidian 导出和工程执行均需额外配置；
- 当文档与运行中的 OpenAPI、配置模型或测试行为冲突时，以代码和接口行为为准。

## License

ProjectLens is released under the [MIT License](LICENSE).
