# 架构与数据流

## 组件分层

```text
入口层
  FastAPI / 飞书 HTTP+WS / MCP
        |
身份与项目边界
  TrustedActor / ProjectSpace / RolePolicy / ChatVisibilityPolicy
        |
ProjectLens Kernel
  ContextEngine / Retrieval / ToolGateway / ContextGovernor / Lifecycle
        |
数据与应用服务
  SourceRecord / Evidence / Memory / KnowledgeGap / AgentRun / Risk
        |
持久化与外部系统
  SQLite / GenAI Trace / Git / 飞书 / GitHub / Obsidian
        |
Agent 执行器
  Hermes provider + 只读工具循环
```

## 一次问答

1. 入口接收问题和部署侧身份。
2. 根据租户、项目、群聊和用户解析 ProjectSpace。
3. 计算 `EffectiveAccessScope`，收敛角色能看到的来源、工具和回答深度。
4. ContextEngine 检索证据、历史、记忆、时间线和知识缺口。
5. ContextGovernor 根据 token 预算分层、压缩、裁剪上下文。
6. Hermes 选择只读工具并循环调用；每次调用经过 ToolGateway 和安全策略。
7. 回答验证器检查引用、事实、推断、未知和冲突。
8. RunService 写入 AgentRun、事件、工具调用、usage 和 trace。

## 知识同步

```text
外部来源 -> Connector.sync -> SourceRecord
         -> Evidence/Chunk -> KnowledgeIndex
         -> lexical/vector/hybrid retrieval
         -> ContextEngine -> Hermes
```

同步状态和 freshness 会影响回答是否提示数据过期。连接器失败不会自动伪造最新事实；回答应保留未知项或陈旧提示。

## 记忆与知识缺口

回答反馈先进入候选或审核队列，再决定是否形成长期记忆。无法由现有来源确认的问题会记录为 KnowledgeGap，可生成摘要、动作和状态转换。这个闭环把“答不上来”变成可跟进的项目任务，而不是让模型猜测。

## 写入边界

读取工具和写入工具在运行时分离。Wiki、Obsidian、工程执行和审批接口必须同时满足配置、身份、项目权限和业务审批。普通问答不会隐式触发写入。
