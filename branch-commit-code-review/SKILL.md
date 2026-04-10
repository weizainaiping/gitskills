---
name: branch-commit-code-review
description: >-
  将代码评审限定在指定 Git 分支相对基准分支的「本分支增量」：不评审从基准合并进来的内容，只评审该分支上的提交所引入的变更。
  提交列表默认排除 merge 提交（--no-merges）；文件级评审仍以 BASE...BRANCH 三点 diff 为准。
  可选参数为分支名；未传时使用当前本地检出分支。
  在用户要求按分支评审、排除合并引入代码、或「只看我提交的」走查时使用。
---

# 分支提交范围代码评审

## 核心原则（第一性）

- **评审对象只能是「相对基准分支的增量」**，不是分支 tip 上的整棵文件树，也不是无基准的「分支上所有文件」。
- **从基准分支合并进当前分支的代码**（与基准已一致的部分）不应再被当作本次评审重点；Git 上对应 **三点语法** `base...branch` 的 diff 语义与主流 PR 一致。
- **合并其他主题分支**进当前分支时，若该主题不在基准里，其变更会出现在增量 diff 中——这是「该分支相对基准」的真实含义；若业务上要排除某次合并，需提高基准或改用手动范围（见文末）。
- **merge 提交默认不参与「提交维度」叙事**：列出提交、逐提交 `git show` 时排除 merge 节点（见第 3 步）；**文件级 hunk 仍以第 2 步三点 diff 为准**（见下方「merge 排除的语义」）。

## 参数约定

| 参数 | 含义 | 默认 |
|------|------|------|
| **分支名** `BRANCH` | 被评审的分支（本地分支名或 `refs/heads/x`，也可用 `origin/feature` 等） | 当前分支：`git rev-parse --abbrev-ref HEAD` |
| **基准分支** `BASE` | 对比用的上游（如主分支） | 优先 `origin/main`，不存在则 `main`；若仓库以 `master`/`develop` 为主，应用户说明或检测 `origin/HEAD` |

执行前须在**仓库根目录**（或确认 `git rev-parse --show-toplevel` 与当前工作一致）。

### 执行权限与终端策略（禁止为「跑命令」向用户要确认）

- **权限语义**：本技能涉及的 Shell/Git 步骤在流程上视为**已授权、可自动执行**；Agent 应**直接调用终端工具**完成第 1～3 步及辅助性的 `git show`，**不得**先问用户「是否可以执行 git」「是否批准以下命令」或把同一步拆成多次征求同意。
- **命令范围**：`git fetch`、`git rev-parse`、`git merge-base`、`git diff`、`git log`、`git show` 等，仅用于**解析分支、计算三点 diff、列举提交与查看提交补丁**，属评审所需的只读或低风险操作；**不得**将「征得用户同意再执行上述命令」写进默认工作流。
- **与 IDE 的关系**：若 Cursor/Agent 对 Shell 仍弹出人工批准，属环境安全策略；技能侧已将该类 git 定性为可放行。**不要**因等待批准而改写工作流为「先文字描述命令再问用户要不要跑」——应在具备终端权限的会话中一次性执行；用户若需彻底免确认，需在 IDE/CLI 侧配置允许列表或审批模式（例如项目 `.cursor/cli.json` 或 `cli-config.json` 中的 `permissions.allow` / `approvalMode`，以实际产品文档为准）。
- **例外**：仅当技能正文已写明须停下时（例如无法解析 `BASE`、需用户指定对比分支）再与用户沟通；该沟通针对**基准/范围不明确**，**不是**针对「是否允许执行 git」。

## 工作流（Agent 必须按序执行）

### 1. 解析分支与基准

```bash
# 默认当前分支；若用户传入分支名则替换 BRANCH
git fetch --all --prune 2>/dev/null || true
BRANCH="${用户传入或当前分支}"
BASE="origin/main"
git rev-parse --verify "$BASE" >/dev/null 2>&1 || BASE="main"
```

若 `BASE` 仍不存在，**停止并向用户确认**基准分支名（例如 `origin/develop`）。

### 2. 计算「只评本分支增量」的 diff（禁止用整分支无基准 diff）

以下二者等价，**任选其一**；评审时**只读这些命令输出的变更**，不要扩 scope：

```bash
git merge-base "$BASE" "$BRANCH"
git diff "$BASE...$BRANCH"
```

- **必须**使用三点 `"$BASE...$BRANCH"`（或先 `MB=$(git merge-base "$BASE" "$BRANCH")` 再 `git diff "$MB..$BRANCH"`）。
- **禁止**作为主评审范围：`git diff "$BRANCH"`、`git show <merge_commit>` 的默认双父全量（除非用户明确要求只看某次合并解决冲突）。

### 3. 提交列表（排除 merge 提交）

用于说明「评的是哪些提交」；**默认不把 merge 提交算作本分支上的待评提交**（避免把「合并节点」当成一次独立代码变更来列示/逐条 `git show`）：

```bash
git log --no-merges --reverse --oneline "$BASE..$BRANCH"
```

- **默认必须**带 `--no-merges`；merge 提交不出现在清单中，也不对其做单独的提交级走查。
- **可选**：若用户明确要求保留 merge 节点（例如仅作里程碑说明），可改用 `git log --first-parent --reverse --oneline "$BASE..$BRANCH"` 或去掉 `--no-merges`，须在结论里注明「含 merge 提交」。
- **文件级评审仍以第 2 步三点 diff 为准**；见下文「merge 排除的语义」。

#### merge 排除的语义（提交 vs 树）

| 维度 | 行为 |
|------|------|
| **提交列表 / 逐提交说明** | `--no-merges`：merge 提交排除在外。 |
| **文件 diff（第 2 步）** | 仍为 `git diff "$BASE...$BRANCH"`。若某次 merge 合入了**基准尚未包含**的第三方变更，对应文件的 hunk **仍会出现在 diff 中**（与常见 PR「Files changed」一致）。 |
| **若既要排除 merge 节点、又不想评合入的第三方文件** | 不能仅靠 `--no-merges` 从 diff 中剥离；须**提高 `BASE`**、或改用用户确认的 `git diff` commit 范围（同文末边界表）。 |

### 4. 执行评审

1. 仅针对 **第 2 步 diff 涉及的 hunk** 做走查：正确性、安全、风格、测试等。若按提交辅助理解上下文，**只对第 3 步列表中的提交**（已排除 merge）使用 `git show <sha>`；不对 merge 提交做单独提交级评审。
2. 若项目为 Java，**叠加** [java-code-review/SKILL.md](../java-code-review/SKILL.md) 的清单与输出模板。
3. 在结论中**简要列出** `BASE`、`BRANCH`、`merge-base` 的 SHA（或短 SHA），便于复核范围。

## 输出要求

- 问题需**对应到具体文件与行号/片段**（来自 diff）。
- 区分严重程度（阻塞 / 建议 / 可选），并给出可操作的修改建议。
- **不要**对未出现在三点 diff 中的代码做「本次变更」类批评（避免评审合并自基准的存量代码）。

## 边界与高级情况

| 场景 | 处理 |
|------|------|
| 基准选错 | 与用户确认 `BASE` 后再跑 diff |
| 多个基准（如先合了 `develop` 再合 `main`） | 以用户指定的「PR 目标分支」为 `BASE` |
| 必须排除某次「合入的其他功能分支」 | 三点 diff 相对旧 `BASE` 仍会包含该功能；需改用更高 `BASE`、或 `git diff` 指定 commit 范围，并由用户确认 |

## 与用户沟通时的简要说明

可告知用户：本次评审范围是 **`BASE...BRANCH` 的 Git 语义**（与常见 PR 文件变更一致），合并自 **`BASE` 已包含内容** 不会在增量 diff 中重复出现；**提交清单默认不含 merge 提交**；若需换对比分支，请直接说明 `BASE`。
