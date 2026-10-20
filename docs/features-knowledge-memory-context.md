# ProjectLens 核心能力说明

本文集中说明 ProjectLens 的六个核心能力：Wiki/知识库、上下文管理器、记忆系统、记忆审批、知识闭环和知识缺口 Workflow。内容以当前代码中的服务、数据模型、API 和测试为准；设计稿中尚未接入生产路径的部分会明确标注。

## 1. Wiki 与知识库

ProjectLens 的知识库不是一个单独的 Markdown 目录，而是一条从外部来源到证据、索引和派生页面的管线：

```text
Git / 飞书文档 / 多维表格 / 任务 / 发布 / 运行信号
    -> Connector / SourceRecord
    -> Evidence 与知识索引
    -> ContextEngine 检索
    -> Wiki draft / Obsidian 派生页面
```

### 1.1 Wiki draft

`WikiDraftCompiler` 根据当前项目中可访问的 SourceRecord 生成固定页面类型：

`overview`、`requirements`、`bitable`、`minutes`、`development`、`testing`、`release`、`rules`。

每个草稿包含项目、页面类型、状态、正文、来源 key 和生成时间。正文是确定性编译结果，保留来源链接，不把模型自由生成的文本直接当作正式 Wiki。

### 1.2 审核与发布

草稿状态为 `draft` 或 `review_required` 时才可审核；审核通过后才会调用 Wiki publisher 并标记为 `published`，拒绝则为 `rejected`。默认使用 `RecordingWikiPublisher`，只有配置飞书 Wiki 空间、父节点 token、凭证并开启 `FEISHU_WIKI_PUBLISH_ENABLED` 后才允许真实发布。

相关接口：

- `POST /api/v1/projects/wiki/draft`
- `POST /api/v1/projects/wiki/review`

### 1.3 Obsidian 导出

Obsidian 是受 ACL 过滤的派生出口，不是新的事实源。导出服务只写入 ProjectLens 管理的路径，并检查 Vault 根目录、文件大小、内容安全和 manifest。必须显式开启 `OBSIDIAN_ENABLED`；Git、Inbox 和操作能力还需要各自开关。

## 2. 上下文管理器

上下文管理器解决的是“这一次问答到底带哪些信息”，不是简单保存全部历史。它包含三层：

1. **会话上下文**：当前用户、项目、群聊、追问和历史消息。
2. **证据上下文**：本次检索得到的 Evidence、来源、时间线和工具结果。
3. **长期记忆上下文**：经过授权且状态为 active 的 ProjectMemory。

### 2.1 Context Governor

`ContextGovernor` 按 token 预算管理上下文：估算请求、保留输出空间、压缩重复内容、裁剪低优先级段落，并在接近硬上限时返回可解释的恢复结果。关键配置包括：

- `CONTEXT_MODEL_LIMIT_TOKENS`
- `CONTEXT_SOFT_LIMIT_RATIO`
- `CONTEXT_SUMMARIZE_RATIO`
- `CONTEXT_PRUNE_RATIO`
- `CONTEXT_ROTATE_RATIO`
- `CONTEXT_HARD_LIMIT_RATIO`
- `CONTEXT_OUTPUT_RESERVE_TOKENS`

### 2.2 飞书交互

普通问题默认可直接回答；发送 `/context`、`上下文管理` 等命令时进入上下文管理路径。当前支持预览、编辑当前上下文、选择更多历史、选择日期窗口和确认后回答。全局预览由 `FEISHU_CONTEXT_PREVIEW_ENABLED` 控制。

### 2.3 冻结与重放

`ContextSnapshotService` 会在运行开始前冻结候选记忆的引用、内容 hash、有效期和授权结果。运行期间若记忆状态、内容 hash 或权限变化，会返回 `CONTEXT_CHANGED`、`MEMORY_REVOKED` 或 `MEMORY_FORBIDDEN`，不会静默使用新内容。这保证了审计和重放看到的是同一份上下文。

## 3. 记忆系统

记忆分为三类层次：

```text
MemoryObservation / Episode   历史观察与一次运行摘要，不是已确认事实
MemoryProposal                等待审批的候选记忆
ProjectMemory                 已审批、可按权限和有效期检索的长期记忆
```

### 3.1 记忆类型

当前支持负责人、架构事实、决策、风险、Runbook、复盘、业务规则、团队约定和需求变更等类型。每条 ProjectMemory 都带项目、正文、事实类型、证据、来源、可见范围、批准人、生效时间、失效时间和状态。

### 3.2 记忆状态

- `active`：可在授权范围内检索并进入上下文。
- `needs_review`：需要重新审核，默认不应作为稳定事实使用。
- `revoked`：撤销，不再进入正常检索。
- `superseded`：被更新版本替代。

记忆检索会同时检查项目、角色权限、visibility scope、有效期和状态。向量相似度只能用于排序，不能绕过 ACL。

### 3.3 记忆接口

- `POST /api/v1/projects/memory-proposals`
- `POST /api/v1/memory-proposals/{proposal_id}/decision`
- `GET /api/v1/projects/{tenant_id}/{project_id}/memories`
- `POST /api/v1/projects/{tenant_id}/{project_id}/memories/search`
- `GET /api/v1/projects/{tenant_id}/{project_id}/memories/{memory_id}/versions`
- `POST /api/v1/memories/{memory_id}/revoke`

## 4. 记忆审批

记忆审批是事实进入长期记忆前的人工控制点。流程如下：

```text
回答/工具结果/用户反馈
    -> MemoryProposal(pending)
    -> 权限、证据、冲突和 TTL 校验
    -> approver approve/reject
    -> ProjectMemory(active) 或 proposal rejected/expired
```

审批提议可以指定允许审批人、证据、来源、authority scope、visibility scope、review_due_at、替代的旧记忆和回滚计划。提议过期后不能继续写入记忆；审批结果和审计事件写入 SQLite。

飞书卡片可以展示“批准/拒绝”按钮，但按钮动作仍由服务端重新解析 actor、项目和权限。拒绝不会写入 ProjectMemory；批准也不会跳过项目访问控制。

### 4.1 记忆复核

来源修订、证据撤销、有效期到期、冲突、到期复核或用户反馈会创建 MemoryReview。复核状态包括 `open`、`acknowledged`、`resolved`、`dismissed` 和 `superseded`。复核解决可以更新、撤销或替代原记忆。

## 5. 知识闭环

知识闭环把“回答 -> 发现问题 -> 补资料 -> 重新验证 -> 更新知识”串成可追踪流程：

```text
来源同步
  -> SourceRecord / Evidence
  -> 检索与回答
  -> unknown / conflict / stale / missing owner
  -> KnowledgeGap 或 MemoryProposal
  -> 人员补资料、确认决策或更新 Wiki
  -> Connector / Index 同步
  -> recheck
  -> 关闭缺口或重新打开
```

闭环中的原则是：原始来源不被覆盖；模型只能提出候选；写入长期记忆或正式 Wiki 必须经过审批；补资料后必须再次检索验证，不能仅凭用户点击“已处理”就关闭问题。

## 6. Knowledge Gap Workflow

知识缺口是对“当前访问范围内无法可靠回答”的结构化治理对象，不等同于所有回答中的 unknown。系统会过滤权限受限、问题不清晰和查询错误，避免把权限问题误报成资料缺失。

### 6.1 缺口类型

`missing_source`、`conflicting_sources`、`stale_source`、`missing_owner`、`missing_decision`、`missing_schedule`、`missing_acceptance`、`access_limited` 和 `unclear_question`。

### 6.2 生命周期

```text
candidate -> open -> acknowledged -> assigned -> waiting_for_source
                                              -> resolved_pending_check
resolved_pending_check -> resolved / reopened
open 或 assigned -> snoozed -> open
candidate/open/assigned -> dismissed
```

每条缺口包含 canonical key、问题、说明、严重程度、影响、证据状态、建议来源类型、建议负责人、观察次数和状态事件。相同问题会按 canonical key 聚合，而不是重复创建大量卡片。

### 6.3 触发与动作

缺口可由问答、定时扫描、Wiki 编译或同步触发。支持的行动包括新增来源、确认决策、分配负责人、更新 Wiki 和关联任务。`resolved_pending_check` 之后必须由重新检索或复查任务确认，才能进入 `resolved`。

主要接口：

- `POST /api/v1/projects/knowledge-gaps`
- `POST /api/v1/project-agent/knowledge-gaps/digest/preview`
- `POST /api/v1/project-agent/knowledge-gaps/{gap_id}/actions`
- `POST /api/v1/project-agent/knowledge-gaps/actions/{action_id}/status`
- `POST /api/v1/project-agent/knowledge-gaps/{gap_id}/transition`

### 6.4 Digest

Knowledge Gap Digest 将新增、持续、重新打开和已关闭的缺口按项目、负责人和周期汇总。即时通知与周期摘要是两条独立路径：即时通知强调尽快处理，Digest 强调趋势、重复出现和责任分配。详细规则见 [Knowledge Gap Daily Digest Workflow](knowledge-gap-daily-digest-workflow.md)。

## 7. 推荐运营顺序

1. 先同步并确认来源权限，再启用问答。
2. 先使用词法检索，确认来源质量后再启用向量索引。
3. 让回答产生的记忆进入 proposal/review，不要直接写 ProjectMemory。
4. 对高频 unknown 建立 KnowledgeGap owner 和 Digest。
5. Wiki 草稿先 review，确认来源和缺口后再发布。
6. 任何补资料操作都触发重新索引和 recheck。

这些能力共同构成 ProjectLens 的可信协作链路：上下文控制回答范围，记忆保存经过审批的长期事实，Wiki 形成可读的派生知识，KnowledgeGap 驱动缺口治理，最终通过同步和复查完成闭环。
