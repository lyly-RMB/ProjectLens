# ProjectLens 文档入口

文档按用途分为四类。项目当前行为以代码、配置模型、运行中的 OpenAPI 和测试为准；标注“设计中/待实施/目标架构”的文件只用于规划。

## 先读这些

| 文档 | 用途 |
| --- | --- |
| [`../README.md`](../README.md) | 项目总览、快速启动、接口和安全边界 |
| [`project-introduction.md`](project-introduction.md) | 面向产品与研发的当前实现说明 |
| [`core-algorithm-description.md`](core-algorithm-description.md) | ProjectSpace、权限、Evidence、上下文、记忆和知识缺口算法说明 |
| [`configuration-and-operations.md`](configuration-and-operations.md) | 配置、启动、状态和安全开关 |
| [`architecture-and-data-flow.md`](architecture-and-data-flow.md) | 组件分层、问答和知识数据流 |
| [`api-and-integrations.md`](api-and-integrations.md) | HTTP API、MCP、飞书和连接器 |
| [`features-knowledge-memory-context.md`](features-knowledge-memory-context.md) | Wiki、上下文、记忆、审批和知识闭环 |
| [`projectspace-onboarding-guide.md`](projectspace-onboarding-guide.md) | 新项目接入和 ProjectSpace 配置 |
| [`feishu-test-quickstart.md`](feishu-test-quickstart.md) | 飞书真实联调步骤 |
| [`feishu-setup.md`](feishu-setup.md) | 飞书权限、事件和长连接配置 |
| [`pilot-launch-config.md`](pilot-launch-config.md) | `.env` 可执行配置参考 |

## 按主题查找

- **架构与边界**：`enterprise-architecture-v2.md`、`projectlens-tool-envelope.md`
- **检索与知识**：`knowledge-loop-remediation-and-test-plan.md`、`projectlens-knowledge-retrieval-vector-and-evaluation-plan.md`
- **记忆与上下文**：`projectlens-long-term-memory-architecture.md`、`projectlens-context-composer-product-design.md`
- **飞书与试点**：`projectlens-feishu-real-test-plan.md`、`projectlens-pilot-development-spec.md`
- **评测与发布**：`evaluation/`、`docs/projectlens-change-log.md`

## 阅读约定

介绍文档只陈述当前已落地或由配置明确支持的能力。规划文档中的路线图、目标架构和待实施方案不能当作现有接口承诺。发现冲突时，优先检查实现和测试，再更新说明。
