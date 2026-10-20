# ProjectLens 核心算法说明

> 本文是商业计划书的技术附录，说明 ProjectLens 如何把项目边界、权限、证据、上下文和知识治理组合成受控 Agent 流程；不把设计目标写成已验证的准确率、延迟或商业收益。

## 1. 算法目标

ProjectLens 不重新训练通用大模型，而是在 Hermes 或其他 Agent Runtime 外增加一层项目治理逻辑，使一次回答满足以下条件：

1. 只使用当前操作者在当前项目和当前会话中可访问的资料。
2. 关键结论可以回溯到来源、版本、时间和 Evidence。
3. 事实、推断、未知、冲突和过期内容被明确区分。
4. 上下文有预算，运行过程可审计、冻结和重放。
5. 资料不足时生成可跟进的 Knowledge Gap，而不是补造结论。
6. 稳定结论只有经过证据、权限、冲突和人工审批，才能进入长期记忆。

算法的商业目标是降低跨系统查找和重复解释成本，同时降低 Agent 越权、幻觉和错误知识沉淀的风险。

## 2. 输入数据

### 2.1 可信运行上下文

- `tenant_id`：租户或组织边界。
- `project_id`：当前项目空间。
- `actor_id`：由部署侧或可信会话解析得到的操作者。
- `chat_id` / `chat_type`：群聊或私聊边界。
- 角色与策略：产品、研发、测试、项目经理、管理者、guest 等。
- 用户问题、时间范围、明确选中的历史或 Evidence。

请求参数不能自行声明可信身份，也不能通过 Prompt 要求模型扩大读取范围。

### 2.2 项目资料

资料由连接器转换为统一的 `SourceRecord`、`Evidence` 或记忆对象，保留：

- 来源系统、原始 URI、对象 ID 和 revision；
- 来源类型、事实类型、authority、confirmed 状态和有效时间；
- 更新时间、freshness、superseded/revoked/expired 状态；
- 来源 ACL、项目归属、可见范围和内容 hash；
- 与需求、代码、任务、会议、测试、发布和运行信号的关联。

### 2.3 运行约束

- 允许调用的只读工具和工具风险等级；
- 检索模式：lexical、vector 或 hybrid；
- Context Governor 的 token 预算和输出预留；
- 记忆、Wiki、工程执行和审批开关；
- 审计、trace、fallback 和超时策略。

## 3. 总体处理流程

```text
问题 + 可信身份
  -> 解析 ProjectSpace 和会话边界
  -> 计算 EffectiveAccessScope
  -> 在授权范围内检索和规范化 Evidence
  -> 按角色投影统一事实
  -> Context Governor 排序、去重、压缩和裁剪
  -> 生成 ContextSnapshot
  -> Hermes 通过只读工具调查
  -> 判断证据是否充分、是否存在冲突或未知
  -> 校验引用、事实、推断、未知和过期提示
  -> 输出回答、审计事件和下一步
  -> 需要时生成 MemoryProposal 或 Knowledge Gap
```

## 4. 核心算法机制

### 4.1 项目边界与有效访问范围

ProjectSpace 把租户、项目、来源、成员、角色策略、飞书群绑定和连接器放在同一个配置边界内。系统在检索之前求交：

```text
EffectiveAccessScope
  = 角色权限
  ∩ 群聊可见范围
  ∩ ProjectSpace 项目范围
  ∩ 来源 ACL
  ∩ 记忆可见范围
```

只有通过求交的来源、Evidence、Memory 和工具结果，才可以进入候选集和模型上下文。该机制避免“先把敏感内容召回，再在回答阶段过滤”的风险。

### 4.2 Evidence 检索与事实状态

检索流程先执行授权过滤，再使用精确代码检索、BM25/词法检索和可选向量通道进行排序。混合检索可以使用 Reciprocal Rank Fusion 合并通道结果；向量服务不可用时保留 lexical 结果并记录降级原因。

候选事实按以下维度判断权威性和有效性：

```text
事实可信度排序
  = authority
  + confirmed 状态
  + effective_from / effective_to
  + freshness
```

系统不静默丢弃冲突。旧规则与新规则、会议结论与正式文档、需求与代码不一致时，保留冲突来源并将状态标记为 `conflict`、`stale`、`missing` 或 `unconfirmed`。

### 4.3 角色化事实投影

系统先维护一份统一事实和 `fact_id` / `evidence_id`，再按角色执行字段过滤、关注点排序和解释深度调整：

```text
统一事实
  -> 识别角色
  -> 过滤不可见字段
  -> 选择角色关注点
  -> 调整解释深度
  -> 生成角色化回答
```

产品角色优先看到业务影响和需求状态，研发角色看到模块、依赖和代码证据，测试角色看到验收和回归风险，项目经理看到负责人和阻塞，管理者看到风险和决策。角色变化只改变可见范围和表达方式，不复制多套相互矛盾的事实。

### 4.4 上下文预算、压缩与快照

Context Governor 先估算系统提示、历史、问题、工具 schema、工具结果、输出预留和运行开销：

```text
总预算
  = system + history + question + tool_schema
  + tool_result + output_reserve + overhead
```

上下文排序综合考虑：

```text
ContextScore
  = 问题相关性 + 角色相关性 + Evidence 权威性
  + 时间新鲜度 + 用户明确选择
  - token 成本 - 重复度 - 过期风险
```

接近预算时按以下顺序处理：去重 -> 摘要 -> 裁剪低优先级内容 -> 轮次旋转。项目边界、权限结果、关键 Evidence、错误信息、正式记忆和用户明确选择的内容具有更高保留优先级。

每次运行生成 `ContextSnapshot`，保存实际使用的内容 hash、Evidence/Memory 引用、授权结果和 renderer 版本。运行期间若内容变化、记忆被撤销或权限失效，返回显式错误，例如 `CONTEXT_CHANGED` 或 `MEMORY_REVOKED`，而不是静默替换上下文。

### 4.5 Hermes 只读调查与停止决策

Hermes 负责理解问题、规划调查、选择工具和组织自然语言；ProjectLens 负责工具发现、参数校验、权限检查、预算、超时、事件和结果验证。

一次调查可以表示为：

```text
问题分类
  -> 选择只读工具
  -> 获取证据
  -> 判断关键事实是否覆盖
  -> 判断是否存在冲突、未知或权限限制
  -> 继续调查，或在收益低于成本时停止
  -> 生成候选回答
```

停止条件包括：关键证据已覆盖、当前角色所需信息已满足、冲突已显式呈现、权限范围内无更多有效资料，或继续调用工具的预期收益低于延迟和 token 成本。所有工具调用、结果、引用和 usage 都写入运行审计，便于复查和重放。

### 4.6 回答验证与不确定性表达

回答输出前，验证器检查：

- 结论是否能绑定到可访问 Evidence；
- 引用的对象、revision、时间和权限是否仍有效；
- 事实、推断、未知、冲突和过期状态是否被正确区分；
- 是否出现未经证据支持的负责人、时间、完成状态或风险结论；
- 是否需要生成下一步动作或 Knowledge Gap。

没有充分证据时，系统优先返回未知、冲突或资料缺口，不用模型的语言流畅性替代事实。

### 4.7 记忆提议与 Knowledge Gap

回答和用户反馈不会直接写入正式项目记忆。稳定结论先形成 `MemoryProposal`，再经过证据、权限、冲突、有效期、允许审批人和人工决策检查：

```text
运行观察
  -> 候选记忆
  -> provenance / ACL / 冲突 / TTL 检查
  -> 人工 approve 或 reject
  -> ProjectMemory(active) 或拒绝/过期/撤销
```

资料不足、来源冲突、来源过期、缺少负责人或缺少验收标准等问题会规范化为 Knowledge Gap。系统使用项目、缺口类型、事实类型、目标对象和归一化问题形成稳定 `canonical_key`，重复发现会累积观察而不是重复建单。缺口经分配、补源、重新索引和 recheck 后，才能关闭或重新打开。

## 5. 参考伪代码

```python
scope = intersect(
    actor_role_acl,
    chat_visibility,
    project_space,
    source_acl,
    memory_visibility,
)

candidates = retrieve(query, scope, mode="lexical|vector|hybrid")
facts = project_by_role(candidates, actor_role)
context = compose_and_compress(facts, selected_history, budget)
snapshot = freeze_context(context)

while investigation_needed(snapshot):
    tool = choose_read_only_tool(snapshot)
    result = call_tool(tool, scope)
    snapshot = update_and_revalidate(snapshot, result)

answer = generate_answer(snapshot)
verified = verify_claims_citations_uncertainty(answer, snapshot)

return verified, maybe_create_memory_proposal(verified), maybe_create_gap(verified)
```

伪代码表达的是运行时边界，不代表当前系统对所有外部数据源都已完成生产接入。

## 6. 输出与审计对象

一次运行可以产生：

- 按角色组织的回答文本；
- Evidence、fact、source revision、时间和权限边界；
- 事实、推断、未知、冲突、过期提示和下一步动作；
- ContextSnapshot、工具调用、模型用量、重试和 fallback 事件；
- MemoryProposal、审批结果、复核和撤销记录；
- Knowledge Gap、负责人、动作、Digest、recheck 和最终状态。

这些输出让客户可以同时查看“答案是什么”“依据是什么”“哪些还不确定”和“下一步由谁处理”。

## 7. 复杂度与降级策略

复杂度取决于来源数量、索引类型、候选规模、模型轮次和上下文预算，当前版本不对生产 QPS 或 P95 延迟作固定承诺。工程上采用以下控制：

- ACL 在召回前过滤，避免无权限内容进入排序和模型上下文。
- 候选数量、返回数量、工具循环墙钟时间和重试次数均有上限。
- 向量或外部模型失败时保留词法检索或明确降级状态。
- 上下文超预算时优先压缩和裁剪；无法安全恢复时拒绝超预算运行。
- 数据库、连接器或同步失败不会伪造“最新事实”。
- 写入、发布、改代码、部署和回滚与普通问答分离，默认关闭或要求审批。

## 8. 验证口径与当前边界

算法效果应使用固定版本和固定数据集验证，至少覆盖：

- Evidence 引用正确性和完整性；
- 时间有效性、冲突识别和未知/拒答准确性；
- 跨项目、跨角色、撤销和过期来源的权限安全；
- 上下文压缩后的决策、引用、任务和负面约束保留；
- 记忆审批、撤销、Knowledge Gap 去重和 recheck；
- 失败回退、重启恢复、重复事件幂等和审计完整性。

当前仓库已经具备相应的代码路径和自动化/模拟测试，但真实飞书端到端效果、生产准确率、并发、延迟、token 成本和商业 ROI 仍需在固定环境的试点与压测中建立基线。算法创新是多项治理机制的工程化组合，不宣称某一组固定权重在所有企业项目上通用最优。
