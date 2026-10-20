# Contributing to ProjectLens

感谢参与 ProjectLens。提交代码前请先确认改动不会绕过项目、身份、权限、证据或审批边界。

## 开发环境

```bash
python -m venv .venv
python -m pip install -e ".[dev,mcp]"
```

运行检查：

```bash
python -m compileall -q src tests
python -m pytest -q tests/test_project_intro.py tests/test_project_agent_ask_api.py tests/test_runtime_security.py tests/test_security_evaluation.py
```

这组命令与公开仓库的 GitHub Actions smoke CI 一致。完整测试集和全量 Ruff 检查可以在本地继续运行，但当前基线仍有需要单独处理的历史失败与 lint 技术债。

涉及权限、检索、Agent 工具、飞书或外部服务的改动必须补回归测试。不要提交 `.env`、数据库、trace、真实凭据、真实飞书绑定或未脱敏业务资料。

## 提交要求

- 一个提交聚焦一个主题；
- PR 描述说明行为变化、测试命令和已知限制；
- 新增配置必须有安全默认值；
- 不把规划目标写成已经验证的生产能力。
