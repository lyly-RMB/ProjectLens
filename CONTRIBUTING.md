# Contributing to ProjectLens

感谢参与 ProjectLens。提交代码前请先确认改动不会绕过项目、身份、权限、证据或审批边界。

## 开发环境

```bash
python -m venv .venv
python -m pip install -e ".[dev,mcp]"
```

运行检查：

```bash
python -m ruff check .
python -m pytest -q
```

涉及权限、检索、Agent 工具、飞书或外部服务的改动必须补回归测试。不要提交 `.env`、数据库、trace、真实凭据、真实飞书绑定或未脱敏业务资料。

## 提交要求

- 一个提交聚焦一个主题；
- PR 描述说明行为变化、测试命令和已知限制；
- 新增配置必须有安全默认值；
- 不把规划目标写成已经验证的生产能力。
