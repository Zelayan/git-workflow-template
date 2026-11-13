# Git 分支管理与提交规范（通用模板）

> 本文件是**项目无关**的 Git 工作流模板，可直接复制到任意仓库的 `docs/git-workflow.md`。
> 凡带 `⚙️` 或 `<尖括号>` 的地方是**占位符**，请先按第 0 节替换成本项目实际值；未替换前不要作为最终规范发布。
> 适用场景：单人 / 多人 / AI Agent 并行开发均可；规模小可裁剪第 2 节（worktree）。

---

## 0. 项目配置区（先填这里）

| 配置项 | 占位符 | 本项目取值 | 说明 |
|--------|--------|-----------|------|
| 发布分支 | `<release-branch>` | `main` | 始终可部署，只从集成分支合并 |
| 集成分支 | `<integration-branch>` | `develop` | 所有特性在此汇聚 |
| 基线分支 | `<base-branch>` | `develop` | 新建特性分支的唯一来源 |
| 构建命令 | `<build-cmd>` | `<pnpm run build>` | 生成产物用 |
| 类型检查命令 | `<typecheck-cmd>` | `<pnpm run typecheck>` | |
| 单元测试命令 | `<test-cmd>` | `<pnpm test>` | 无测试脚本则写实际命令 |
| 集成/端到端命令 | `<e2e-cmd>` | `<……>` | 可空 |
| 生成物目录 | `<generated-dirs>` | `dist/`、`lib/` | 构建产物、打包产物 |
| 受保护分支 | `<protected-branches>` | `main`、`develop` | 见 1.7 |
| 提交语言 | `<commit-lang>` | 中文 | 描述语种 |
| 包管理器 | `<package-manager>` | `pnpm` | 锁文件随之确定 |

> 小型项目可裁剪：若团队只有 1–2 人、无并行需求，可删除第 2 节，并把 `develop` 并入 `main` 采用单主干（见 1.1 方案 A）。

---

## 1. 分支模型

本项目采用 **短生命周期特性分支 + 集成分支/发布分支双长期分支** 模型（类 GitFlow 的精简版，**不是**纯 Trunk-Based Development）。该模型在"AI Agent 并行协作"下比单主干更稳：特性分支把并行改动隔离，集成分支做统一验证，发布分支只保留可交付状态。

### 1.1 两种可选模型

| 方案 | 结构 | 适用 |
|------|------|------|
| **A. 单主干（Trunk）** | 只有 `<release-branch>`，特性分支直接合并回它 | 小团队、无发布闸门、要求极致简单 |
| **B. 集成/发布双分支（默认）** | 特性分支 → `<integration-branch>` → `<release-branch>` | 需要"集成验证"与"可发布快照"分离的场景 |

> 方案 B 才是 Trunk-Based Development 的近亲；不要把它叫作 TBD，以免与"单主干"混淆。下文默认按方案 B 描述。

### 1.2 长期分支与单向合并

| 分支 | 用途 | 保护规则（须真实生效，见 1.7） |
|------|------|-------------------------------|
| `<release-branch>` | 发布分支，始终可部署 | 只接受 `<integration-branch>` 的 `--no-ff` merge；**禁止任何直接 commit** |
| `<integration-branch>` | 集成分支，所有特性在此汇聚 | 只接受特性分支的 `--no-ff` merge |

**核心约束：合并方向单向 —— `<integration-branch> → <release-branch>`，禁止反向合并。**

```
特性分支 ──→ integration ──→ release ──tag──→ vX.Y.Z
```

- ✅ `git checkout <release-branch> && git merge <integration-branch> --no-ff`
- ❌ `git checkout <integration-branch> && git merge <release-branch>`（禁止）

当发布分支上有集成分支没有的提交（如 hotfix）时，**不要反向 merge 发布分支**；应通过 1.5 的 hotfix 分支同时落到两个分支来同步。

### 1.3 特性分支

命名格式：`<type>/<short-description>`（描述用英文小写连字符，或按团队约定）

| 前缀 | 用途 | 示例 |
|------|------|------|
| `feat/` | 新功能 | `feat/user-export` |
| `fix/` | Bug 修复 | `fix/login-timeout` |
| `refactor/` | 重构（不改行为） | `refactor/split-session` |
| `docs/` | 纯文档 | `docs/update-readme` |
| `test/` | 纯测试 | `test/quote-unit` |
| `chore/` | 构建/工具/配置 | `chore/bump-deps` |
| `perf/` | 性能优化 | `perf/cache-index` |
| `ci/` | CI/CD 配置 | `ci/add-lint-job` |

**生命周期规则：**

1. 从 `<base-branch>` 最新 HEAD 创建：`git checkout <base-branch> && git pull && git branch <name>`
2. 完成后以 `--no-ff` merge 回 `<integration-branch>`
3. merge 后立即删除本地特性分支（`git branch -d`）
4. 特性分支**不直接 merge 到 `<release-branch>`**
5. 特性分支之间**不互相 merge**——有依赖时先合并上游到集成分支，再从集成分支 rebase
6. **特性分支默认不 push 到远程**：它是临时工作区，只在本地创建、提交、merge、删除。远程只保留长期分支与版本 tag——推上去只会造成分支堆积与"幽灵分支"（本地已删、远程还在）。
   > 若团队**必须**走 PR 评审，则此条改为"可 push 到远程并开 PR"，同时 1.7 的强制手段必须开启。两种模式二选一，写进配置区。

### 1.4 生成物与锁文件（合并冲突治理 ★关键）

> 这是多分支并行最容易踩的坑：**生成物和锁文件会让"改的文件不重叠"也产生合并冲突**，必须单独规定。

| 对象 | 规则 |
|------|------|
| 生成物目录 `<generated-dirs>` | **默认不入库**，加进 `.gitignore`。源码是唯一事实来源，任何人在需要时本地构建即可 |
| 若生成物**必须入库**（发布需要） | 特性分支**只提交源码，不提交生成物**；由 Lead 在**所有特性分支合并进集成分支之后**，在集成分支上**统一重建一次**生成物并单独提交（如 `chore(build): 重建 <generated-dirs>`）。**禁止每个特性分支各自重建后带入 merge** |
| 依赖锁文件（`pnpm-lock.yaml` / `package-lock.json` 等） | 依赖变更集中在单次提交完成；并行开发时由一人负责改依赖与锁文件，其他人 rebase 后再动，避免锁文件互相冲突 |
| `.gitignore` | 必须包含生成物目录、临时目录、本地环境文件 |

**为什么**：若每个特性分支都重建生成物，Lead 把两个分支 merge 回集成分支时，生成物文件（常常是几 MB 的打包产物）**必然冲突**，即使源码文件完全不相交——这会直接抵消 1.3 与第 2 节"写作用户不重叠就无冲突"的并行收益。

### 1.5 Hotfix 流程（可操作）

紧急修复发布分支上的问题，同时必须回流到集成分支，且**不得反向 merge 发布分支**：

```bash
# 1. 从发布分支拉 hotfix 分支
git checkout <release-branch> && git pull
git branch hotfix/<short-desc>

# 2. 在 hotfix 分支修复、测试、提交
git checkout hotfix/<short-desc>
# …修改…
git commit -m "fix(<scope>): <中文描述>"

# 3. 合入发布分支并打补丁 tag
git checkout <release-branch>
git merge --no-ff hotfix/<short-desc> -m "merge: <release-branch> ← hotfix/<short-desc> — <摘要>"
git tag -a vX.Y.Z+1 -m "vX.Y.Z+1: <补丁摘要>"

# 4. 把同一 hotfix 回流到集成分支（合的是 hotfix 分支，不是发布分支本身）
git checkout <integration-branch>
git merge --no-ff hotfix/<short-desc> -m "merge: <integration-branch> ← hotfix/<short-desc> — <摘要>"

# 5. 推送发布分支与 tag，删除 hotfix 分支
git push origin <release-branch> --tags
git branch -d hotfix/<short-desc>
```

> 要点：第 4 步合的是 **hotfix 分支**，不是 `<release-branch>`，因此没有违反"禁止反向 merge"。

### 1.6 发布流程

```
<integration-branch> (稳定) ──merge──→ <release-branch> ──tag──→ vX.Y.Z
```

1. 确认集成分支上所有改动已通过 `<typecheck-cmd>`、`<test-cmd>`、以及项目的护栏/构建检查
2. **在集成分支（或发布准备分支）上**修改版本号并提交：`chore(release): X.Y.Z`
3. `git checkout <release-branch> && git merge <integration-branch> --no-ff -m "merge: develop → main — X.Y.Z：<摘要>"`
4. `git tag -a vX.Y.Z -m "vX.Y.Z: <发布摘要>"`
5. push `<release-branch>` 和 tag

> ⚠️ **版本号不要在发布分支上单独提交**。那样会让发布分支产生集成分支没有的 commit，每次发布都要手工 cherry-pick 同步，等于自己破坏 1.2 的单向规则。版本 bump 一律在集成分支完成后再整体 merge。

### 1.7 分支保护：约定 vs 强制（★必读）

"保护规则"分两层，**只写进文档的只是约定，必须另有技术手段强制**，否则形同虚设：

| 层级 | 手段 | 是否必须 |
|------|------|---------|
| 约定 | 本文档、团队共识、Code Review | 必须 |
| **强制** | 平台分支保护（GitHub/GitLab Branch Protection：禁止直推、要求 PR、required checks） | **强烈建议开启** |
| **强制** | 本地 `pre-push` hook：检测当前分支是否为受保护分支、merge 方向是否正确 | 建议 |
| **强制** | CI 流水线：`<typecheck-cmd>` + `<test-cmd>` 必须绿 | 建议 |
| 可选 | commitlint：校验提交信息符合第 3 节格式 | 建议 |

> 平台上的"protected"开关要**真的打开并截图/记录在案**；文档里写"已保护"但实际 `protected: false` 是最常见的自欺。

---

## 2. 多 Agent / 多人并行：git worktree

> 仅在使用 Agent Teams 或多 agent 并行协作时需要；单 agent 串行开发可跳过本节。

### 2.1 为什么必须用 worktree

多个 agent 共享同一工作目录时，任一 agent 执行 `git checkout` 会**改变整个工作区的分支引用**：

- agent A 在 `feat/a` 上编辑文件，agent B 执行 `git checkout develop`，A 的未提交改动可能被覆盖或产生混乱冲突；
- 构建产物与 `node_modules/` 是工作区级状态，切分支后状态不一致，类型检查和构建结果会误导；
- 并发 `git checkout` 有竞态：索引（index）与 HEAD 的更新不是原子的，两个 agent 同时操作会损坏 `.git/index`。

`git worktree` 为每个分支创建**独立工作目录**，共享同一个 `.git` 对象库、引用独立，从根本上消除上述问题。

### 2.2 工作流

**Lead（创建）：**
```bash
git checkout <integration-branch> && git pull
# worktree 放仓库内部 .worktrees/ 下（放仓库外可能因权限失败）
git worktree add .worktrees/agent-a -b feat/agent-a-task <integration-branch>
git worktree add .worktrees/agent-b -b refactor/agent-b-cleanup <integration-branch>
```

**Agent（在 worktree 内工作）：**
1. 进入 Lead 指定的目录（如 `.worktrees/agent-a`）作为工作目录
2. 完成编辑、`<typecheck-cmd>`、`<build-cmd>`、`<test-cmd>`、commit——**不要切换到其他分支**（该 worktree 已绑定你的特性分支）
3. 完成后向 Lead 报告 commit hash 与 worktree 路径，**不要自行 merge**

**Lead（收集与合并）：**
```bash
# 1. 逐个 review
git -C .worktrees/agent-a log --oneline <integration-branch>..HEAD

# 2. 合并回集成分支
git checkout <integration-branch>
git merge --no-ff feat/agent-a-task -m "merge: <integration-branch> ← feat/agent-a-task — <摘要>"
git merge --no-ff refactor/agent-b-cleanup -m "merge: <integration-branch> ← refactor/agent-b-cleanup — <摘要>"

# 3. 若生成物需要入库：此处统一重建一次并单独提交（见 1.4）
git merge ...   # 全部合并完成后
<build-cmd> && git add <generated-dirs> && git commit -m "chore(build): 重建 <generated-dirs>"

# 4. 清理 worktree（先删工作目录，再删已合并的特性分支）
git worktree remove .worktrees/agent-a
git worktree remove .worktrees/agent-b
git branch -d feat/agent-a-task refactor/agent-b-cleanup
```

### 2.3 约定

| 约定 | 说明 |
|------|------|
| worktree 位置 | 仓库内部 `.worktrees/`，命名 `.worktrees/<agent名>`；**不写到仓库外**（同级目录可能无权限） |
| 分支来源 | 一律从 `<base-branch>` 最新 HEAD 创建 |
| 单一归属 | 每个 worktree 只分配给一个 agent，不跨 agent 复用 |
| 依赖安装 | 新 worktree 需独立 `<package-manager> install`（worktree 不共享 `node_modules`） |
| 生成物 | 各 worktree 本地构建即可；**是否入库按 1.4 由 Lead 统一处理** |
| 清理 | 合并后 Lead 立即 `git worktree remove`，不残留 |
| 不入库 | `.worktrees/` 必须写进 `.gitignore` |

### 2.4 何时用 / 何时不用

| 场景 | 是否用 worktree |
|------|----------------|
| 多 agent 并行修改不同文件 | ✅ 必须 |
| 单 agent 串行完成全部工作 | ❌ 不用，直接在特性分支工作 |
| 两个 agent 修改同一文件（写作用户重叠） | ❌ 不用——先拆任务使写作用户不重叠，worktree 解决不了内容冲突 |
| 需要并行对比两个分支的运行效果 | ✅ 适合 |

---

## 3. 提交规范

采用 [Conventional Commits](https://www.conventionalcommits.org/zh-hans/v1.0.0/)，提交信息描述一律使用 `<commit-lang>`（默认中文）。

### 3.1 格式

```
<类型>(<范围>): <描述>

[可选正文]

[可选脚注]
```

### 3.2 类型

| 类型 | 含义 | SemVer 对应 |
|------|------|------------|
| `feat` | 新功能 | MINOR |
| `fix` | Bug 修复 | PATCH |
| `refactor` | 重构（不改外部行为） | — |
| `docs` | 文档更新 | — |
| `test` | 测试增删改 | — |
| `chore` | 构建/工具/配置/依赖 | — |
| `perf` | 性能优化 | PATCH |
| `style` | 代码格式（不影响逻辑） | — |
| `ci` | CI/CD 配置 | — |
| `revert` | 回滚 | — |
| `merge` | **合并提交专用**（见 3.6） | — |

### 3.3 范围（scope）

用模块名或层级名，与本项目目录结构对齐（**下表的 `<……>` 请按实际模块替换**）：

| 范围 | 对应模块 |
|------|---------|
| `<module-a>` | `<src/a/>` |
| `<module-b>` | `<src/b/>` |
| `<module-c>` | `<src/c/>` |
| `release` | 版本发布（`chore(release): X.Y.Z`） |
| `build` | 构建产物重建（`chore(build): …`） |

省略范围也可以；改动集中在单个模块时推荐标注。

### 3.4 描述要求

- 简洁明了（≤ 50 字为佳），祈使语气："提取…"、"修复…"、"新增…"，不用"了/过"等过去时助词
- 不以句号结尾
- 中文字符无大小写问题；若用英文，首字母不大写

### 3.5 正文与脚注

- 正文与标题之间空一行，解释**"为什么"**而非"做了什么"
- 破坏性变更：标题加 `!`，或脚注写 `BREAKING CHANGE:`
- 引用 issue/PR 用脚注：`Refs: #123`

### 3.6 Merge Commit 格式

merge 提交也走约定式，类型固定为 `merge`。**箭头一律指向目标分支（合并去向）**：

- 合并 into 集成分支（目标在左）：`merge: <integration-branch> ← <source-branch> — <摘要>`
- 合并 into 发布分支（目标在右）：`merge: <integration-branch> → <release-branch> — <摘要>`

示例：
```
merge: develop ← feat/user-export — 新增用户导出功能
merge: develop → main — v1.4.0：用户导出与登录超时修复
merge(session): develop → main — 会话模块拆分与凭据下沉
```

### 3.7 示例

```
refactor(transport): 提取共享平台模块，消除两份 transport 重复代码

平台映射与命令构建逻辑在两处实现中重复，新增平台或修复探测 bug
时需改两处。提取到共享模块后只需改一处。

Refs: #101, #102
```

```
fix(auth): 重连时旧心跳未停止导致配额泄漏

重连入口漏调 heartbeat.stop()，旧心跳持续占用通道配额，
新连接建立后配额耗尽。

Refs: #287
```

```
feat!(credential): 支持多供应商凭据路由

BREAKING CHANGE: 凭据构造函数新增 routes 参数，单供应商初始化方式不再可用。
```

```
chore(release): 1.4.0
```

---

## 4. 工作流检查清单

### 开始前

- [ ] 已 `git pull` 拉取 `<base-branch>` 最新，且当前在正确的特性分支（不在受保护分支上直接改）
- [ ] 依赖已安装：`<package-manager> install`
- [ ] 若项目依赖提交钩子：已运行钩子安装命令（如 `pnpm run setup:hooks`）——否则"自动构建/校验"不会生效
- [ ] Lead 分配的写作用户与其他 agent 不重叠
- [ ] Agent 团队协作时：已在独立 worktree 内（如 `.worktrees/agent-a`），不与他人共享工作目录

### 工作中

- [ ] 每个逻辑独立的改动单独 commit（不攒大提交）
- [ ] commit message 符合第 3 节格式
- [ ] 运行 `<typecheck-cmd>` 确认零错误
- [ ] 运行 `<test-cmd>` 确认通过（有集成/端到端则跑 `<e2e-cmd>`）
- [ ] 生成物按 1.4 处理：特性分支不提交生成物；如需入库由 Lead 合并后统一重建
- [ ] 若项目依赖构建产物才能运行（如插件形态），已按项目说明生成并校验

### 完成后

- [ ] 所有改动已 commit 到特性分支
- [ ] 向 Lead 报告完成状态、commit hash、验证结果
- [ ] **不自行 merge 到 `<integration-branch>` / `<release-branch>`**
- [ ] **不做 `<release-branch>` → `<integration-branch>` 的反向 merge**
- [ ] **不 push 特性分支到远程**（若项目采用 PR 模式则改为"已开 PR"）
- [ ] Agent 团队协作时：报告 worktree 路径，由 Lead 合并后清理 worktree

---

## 5. 禁止事项

| 禁止操作 | 原因 |
|---------|------|
| 直接在 `<release-branch>` / `<integration-branch>` 上 commit | 所有改动必须经特性分支 |
| 从 `<release-branch>` 创建特性分支 | 特性分支只能从 `<base-branch>` 创建 |
| `<release-branch>` → `<integration-branch>` 的 merge | 破坏单向合并规则，产生混乱历史 |
| 特性分支之间互相 merge | 应通过集成分支中转 |
| push 特性分支到远程（非 PR 模式） | 临时工作区，远程只保留长期分支与 tag，否则产生幽灵分支 |
| **每个特性分支各自重建生成物并带入 merge** | 生成物必然冲突，抵消并行收益（见 1.4） |
| **在发布分支上单独提交版本号** | 制造发布分支独有 commit，破坏单向合并（见 1.6） |
| 共享工作区切分支（Agent 团队协作时） | 会改变所有 agent 看到的文件，必须用 worktree 隔离 |
| worktree 创建到仓库外部 | 同级目录可能权限不足导致 `git worktree add` 失败 |
| worktree 未清理残留 | 合并后必须 `git worktree remove` |
| 提交 `.worktrees/` | worktree 临时区，须在 `.gitignore` 排除 |
| force push 已推送的共享分支 | 除非明确要求重写历史 |
| 使用 `git merge --squash` | 需保留完整提交历史，统一用 `--no-ff` |
| 提交本地环境/密钥文件（`.env` 等） | 已在 `.gitignore` 排除，绝不入库 |
| 提交信息用与 `<commit-lang>` 不符的语言 | 统一语言，便于检索与生成变更日志 |

---

## 6. 可选：自动化落地建议

按团队能力逐项接入（对应 1.7）：

1. **分支保护**：在代码托管平台对 `<protected-branches>` 开启"禁止直推 + 要求合并请求 + 必需状态检查"。
2. **pre-push hook**：拦截对受保护分支的直接 push、拦截错误方向的 merge。
3. **pre-commit hook**：按需自动格式化 / 重建生成物（若采用 1.4 的"由 Lead 统一重建"策略，则**不要**在 pre-commit 里为每个特性分支自动改生成物）。
4. **CI**：至少跑 `<typecheck-cmd>` + `<test-cmd>`；发布分支额外跑打包与冒烟。
5. **commitlint**：用第 3 节的类型/范围表做规则源，CI 里校验提交信息格式。
6. **CHANGELOG**：可由 Conventional Commits 自动生成（`feat`/`fix`/`perf` 入正文，`BREAKING CHANGE` 单独标注）。

---

## 附：与原始参考文档的主要调整

若你是从某个具体项目（如 dsh-remote-explorer）的版本改写而来，本模板相对其做了以下修正：

- **去掉项目专有内容**：`src/` 模块表、`lib/`、`scripts/*.ts`、pnpm 11 细节等一律替换为占位符，避免模板绑定单一仓库。
- **新增 1.4 生成物与锁文件治理**：显式规定"特性分支不提交生成物、由集成分支合并后统一重建"，避免 worktree 并行时生成物冲突。
- **新增 1.7 约定 vs 强制**：要求分支保护真实开启，而非只在文档里写"已保护"。
- **修正发布流程**：版本号在集成分支 bump，发布分支只 merge + tag，消除与"单向合并"的矛盾。
- **新增 1.5 hotfix 完整 recipe**：用 hotfix 分支同时流入两个分支，避免反向 merge。
- **补全提交类型与范围**：类型表加 `merge`，范围表加 `release`、`build`。
- **明确箭头语义**：merge 消息箭头一律指向目标分支，并给出两种方向示例。
- **检查清单补齐**：加入 bootstrap（install / 钩子安装）与 `<test-cmd>`，与发布流程要求对齐。
- **术语校正**：不再自称 Trunk-Based Development，改称"短生命周期特性分支 + 集成/发布双分支（类 GitFlow 精简版）"。
