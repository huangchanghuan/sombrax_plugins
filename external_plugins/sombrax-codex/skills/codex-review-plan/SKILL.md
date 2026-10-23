---
name: codex-review-plan
description: >
  Launch Codex CLI to review a Claude Code implementation plan for completeness, correctness, and potential issues.
  Use when a plan is ready for approval before implementation. Trigger when the user says "review the plan",
  "approve the plan", "codex review plan", "check the plan", or after plan mode produces a plan.
  （在实现开始前用 Codex CLI 对实施计划做独立评审；触发词：review the plan / approve the plan / 评审计划等。）
---

在实现开始之前，用 Codex CLI 对实施计划做一次独立第二意见评审。

参数：`[plan-file-name]` —— 要评审的计划文件（可选，默认取最近一份）

## 流程

1. **定位计划文件：**
   - 如果有参数，用它作为计划文件名：`~/.claude/plans/<argument>`
   - 否则找最近一份：`ls -t ~/.claude/plans/ | head -1`
   - 读取计划文件内容

2. **收集项目上下文：**
   - 读 `CLAUDE.md` 获取架构约定与项目惯例
   - 运行 `git diff main --stat` 了解当前分支的改动范围（如有变更）

3. **发起第一轮 Codex 评审：**
   - 把计划内容写到临时文件，让 Codex 能读到
   - 以 exec 模式运行 Codex：

   ```bash
   PLAN_PATH="$HOME/.claude/plans/<plan-file>"
   codex exec --sandbox read-only "Review this implementation plan before I start coding. Read the plan at $PLAN_PATH and the project guidelines at ./CLAUDE.md.

   Review for:
   1. COMPLETENESS: Does the plan cover all necessary changes? Missing migrations, tests, error handling?
   2. ARCHITECTURE: Does it follow the project's patterns from CLAUDE.md?
   3. RISKS: Race conditions, breaking changes, edge cases, performance concerns?
   4. ORDER OF OPERATIONS: Are the steps in the right sequence? Dependencies respected?
   5. SCOPE CREEP: Is anything unnecessary or over-engineered for what's being solved?

   Give a verdict: APPROVED / NEEDS CHANGES / REJECTED
   APPROVED means no Critical or Important findings remain (Minor findings are acceptable).
   List specific issues by severity (Critical / Important / Minor).
   For each issue, reference the specific section of the plan." < /dev/null
   ```
   - 重定向 `< /dev/null` 是强制的：`codex exec` 在 prompt 之外还会读 stdin，不关就会永远阻塞。

4. **处理结果并迭代：**
   - 解析 Codex 的结论与发现
   - 如果 **APPROVED**（无 Critical/Important 发现）：报告通过，确认计划可以进入实现
   - 如果 **NEEDS CHANGES** 或 **REJECTED**（存在 Critical 或 Important 发现）：
     a. 向用户呈现全部发现
     b. 根据发现修订计划文件
     c. **通过恢复同一个 Codex 会话来复检**——不要开全新评审。Codex 上下文里已有完整计划和约定，一条简短跟进的成本只是全新评审的零头：
     ```bash
     codex exec resume --last "We updated the plan at $PLAN_PATH to address your findings: <one-line summary of each change>. Re-read the plan and verify each finding is resolved. Give a new verdict (APPROVED / NEEDS CHANGES) and report only unresolved or newly introduced issues." < /dev/null
     ```
     d. `resume` 要在**与首轮评审相同的工作目录**下运行（`--last` 按 cwd 过滤会话）。如果 `resume` 失败，回退为全新完整评审（第 3 步）。
     e. 重复直到结论为 APPROVED（只剩 Minor 发现）

## 注意

- Codex 是只读的（`--sandbox read-only`）——它不能修改文件
- 评审迭代之间，由 Claude（不是 Codex）根据发现修订计划
- 修复后的复检恢复同一个 Codex 会话（`codex exec resume --last`），Codex 上下文里保有计划和 CLAUDE.md，只重读变化的部分——这是相比每轮重启完整评审的主要 token/时间节省
- 如果 Codex 未安装，回退为按相同标准内联执行评审
