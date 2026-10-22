<!--
codex-review.md — the method the self-driving coding agent uses to gate its own work
with codex: plan mode after planning, diff mode after building. Set {{BASE_BRANCH}}
(default `main`). The agent runs codex locally (it has the code), reports the verdict,
then resolves every blocker itself and reviews again until it PASSes. Nobody is waiting
on the other end of this review — it is a loop the agent closes on its own.
（维护者注释：正文已中文化；codex 命令行与报告形状（CODEX VERDICT 等）为协议字面量，保持不变。）
-->
用 **Codex CLI** 做一次独立评审，报告结论，然后**亲自处理它**。codex 以只读方式运行：它评审，**你**修复。

选择与当前进度匹配的模式：

## 评审计划（任何开发开始之前）
```
codex exec --sandbox read-only "Review this implementation plan before any code is written. Read the plan at ./IMPLEMENTATION_PLAN.md and the project guidelines at ./CLAUDE.md if present. Assess: is the approach correct and complete, are the steps grounded in the real code, does it actually satisfy the task, and what is missing or risky? Separate blockers (must fix) from recommendations and nits." < /dev/null
```

## 评审 diff（某一步之后，或标记完成之前）
用 `codex exec`（不是 `codex review`），这样会话可以低成本地 resume 复检：
```
codex exec --sandbox read-only "Review the diff of this branch against its base. Run git diff {{BASE_BRANCH}} yourself and read any changed files you need. Review for: correctness bugs, missed requirements, regressions, error handling, and consistency with the codebase and CLAUDE.md. Separate blockers from recommendations and nits." < /dev/null
```

## 修复之后复检——恢复同一个会话
修完阻塞项之后，**不要开启全新评审。** codex 的上下文里已经持有计划/diff、它读过的文件和它自己的发现，所以 resume 的 token 和时间成本只是全新评审的零头：
```
codex exec resume --last "We fixed your findings: <one-line per fix>. Re-check the updated plan/diff and verify each finding is resolved. Report only unresolved or newly introduced blockers; if none, say PASS." < /dev/null
```
`resume` 要在**与第一次评审相同的工作目录**下运行（`--last` 按 cwd 过滤会话）。如果 `resume` 失败（没有会话、CLI 报错），就回退为用上面的首评命令做一次全新完整评审。

**绝不要让 codex 的 stdin 保持打开。** `codex exec`（以及 `exec resume`）在 prompt 参数*之外*还会读 stdin：从代理的 shell 里启动时，stdin 是一个永远不发 EOF 的打开管道，所以 codex 会打印 "Reading additional input from stdin…" 并**永远阻塞**在 0% CPU。这就是这里每条命令都以 `< /dev/null` 结尾的原因。要在**仓库工作树内**运行 codex（不是工作区根目录）；仅当 codex 抱怨目录不受信任时才加 `--skip-git-repo-check`。

## 按以下形状报告
```
CODEX VERDICT: PASS | CHANGES REQUESTED
Blockers (must fix):   <list, or "none">
Recommendations:       <list, or "none">
Nits:                  <list, or "none">
Bottom line:           <one or two sentences; for a plan, also: does it satisfy the task?>
```

`PASS` = 无阻塞项。任何阻塞项 → `CHANGES REQUESTED`。

如果 `codex` CLI 未安装或运行失败，**明确说明**，并按相同标准内联执行评审——不要默默跳过。

**然后亲自闭环——这场评审永远不是停车点。** 结果为 `PASS`，继续流水线。结果为 `CHANGES REQUESTED`，**亲自解决每一个阻塞项**并**再次评审**：CLI 可用就恢复同一个 codex 会话（`codex exec resume --last "…" < /dev/null`）；不可用就按相同标准重复**内联**评审，并再次说明 CLI 不可用。**迭代直到结论为 `PASS`**，然后继续。

一个阻塞项只有两种了结方式：你**修复它**（计划评审就修订 `IMPLEMENTATION_PLAN.md`；diff 评审就改代码并提交），或者——因为 codex 确实会产生误报——你**书面驳回它并给出理由**，并把这份驳回带进下一次结论。Recommendations 和 nits 由你自行裁量。没有轮数上限，也不存在"报告残留然后继续"：没有别人评审这些，也不会有人递给你放行令。**唯一的**停下理由是你的 kickoff 已经允许的那一个——**无法从 spec/plan/代码库推导的真正决策**——你以问题的形式暴露它。
