# Git 工作流规范模板 / Git Workflow Template

一份**与具体项目无关**的 Git 分支管理与提交规范模板，适用于单人、多人以及 AI Agent 并行开发场景。

## 内容

- [`docs/git-workflow.md`](docs/git-workflow.md) — 完整规范模板（分支模型、提交规范、worktree 并行协作、检查清单、禁止事项、自动化建议）

## 使用方式

1. 把 `docs/git-workflow.md` 复制到你的仓库 `docs/` 下；
2. 按文首 **第 0 节「项目配置区」** 替换占位符（分支名、构建/测试命令、生成物目录、模块 scope 表等）；
3. 小团队可裁剪第 2 节（worktree）并采用单主干方案；
4. 按第 6 节逐项接入分支保护 / CI / commitlint 等自动化。

## 本模板相对常见实践修正的坑

- **生成物合并冲突**：特性分支不提交构建产物，由集成分支合并后统一重建，避免 worktree 并行时产物冲突。
- **分支保护流于形式**：区分「约定」与「强制」，要求平台 branch protection 真实开启。
- **发布流程与单向合并矛盾**：版本号在集成分支 bump，发布分支只 merge + tag。
- **hotfix 可操作流程**：用 hotfix 分支同时流入发布分支与集成分支，避免反向 merge。
- **提交类型/范围补全**：类型含 `merge`，范围含 `release`、`build`。
- **merge 消息箭头语义**：箭头一律指向目标分支。

## License

MIT
