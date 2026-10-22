<!--
plan.md — the canonical planning method. Planning is owned by the dedicated
`planner` agent (it writes `IMPLEMENTATION_PLAN.md` at the workspace root); this
prompt is that agent's shape/method. It is also kept self-contained so a
self-driving coding agent can be handed the same prompt directly when no separate
planner step is run. Fill {{TASK}} with the card's title + spec before sending.
（维护者注释：正文已中文化；下方模板代码块的标题字面量被多处软引用，
按约定保持英文。）
-->
你在为这个仓库规划下面的任务。**在写任何代码之前**，产出一份实施计划，保存为**工作区根目录**（与 `CLAUDE.md` 同级、git 仓库的上一级——该位置在所有仓库工作树之外，所以文件永远不会被提交，也不需要 gitignore 条目）下的 `IMPLEMENTATION_PLAN.md`。它指导本次工作，并在分支合并后留存。

如果工作区根目录存在 `SPEC.md`，它就是本任务的权威规格——先读它，把计划落地在它之上。如果那里还有 `PRIOR_KNOWLEDGE.md`（由 `recall-knowledge` 阶段写出），也一并阅读——它提炼了这个项目知识库已有的认知；复用其中已确立的模式/决定，而不是重新推导，并把每条事实应用到其范围内点名的仓库。

## Task
{{TASK}}

## 如何规划
先读相关代码，把每一步都落地在真实文件上——一份点错函数名、或假设了不存在的结构的计划，比没有计划更糟。然后按以下形态写 `IMPLEMENTATION_PLAN.md`：

```
## Implementation plan: <title>

**Goal:** <what "done" looks like, traceable to the task>
**Approach:** <strategy in 2–4 lines; note any alternative you rejected and why>

### Steps (ordered; each one small and independently verifiable)
1. <imperative step> — files: `path/one`, `path/two`; done-when: <observable check>
2. <…>

### Verification
<how the whole change is proven — tests to add/run, build/lint, manual checks;
concrete commands where you know them>

### Risks / open questions
<unknowns, ordering constraints, unverified assumptions, anything needing a
decision before/while building>
```

每个步骤都要小到一轮专注的执行就能完成，并且排序上让后面的步骤只依赖前面的步骤。

## 然后停下
把文件保存到工作区根目录，用一行确认已写出，然后**停下——不要开始实现**。下一条指令会是对计划的 codex 评审，之后才是逐步开发。
