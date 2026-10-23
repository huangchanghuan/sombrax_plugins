---
name: codex-review
description: >
  Run iterative code review with Codex CLI until no significant findings.
  Use this skill when the user wants a code review, asks to review changes, or says
  "review my code", "codex review", "run a review", "check my changes".
  Triggers on code review requests for any Rust or general codebase.
  （用 Codex CLI 迭代评审代码直到无重大发现；触发词：代码评审 / codex review / review my code 等。）
---

用 Codex CLI 对当前分支做迭代式代码评审：对照基础分支（默认 `main`）评审变更、修复问题、再次评审，直到不再有重大发现。

参数：`[base-branch]` —— 用于 diff 的基础分支（默认 `main`）

## 评审标准

### 代码质量
- 逻辑正确、错误处理完备
- 无死代码、无未使用的导入
- 资源清理到位（文件句柄、连接）
- 没有本应可配置的硬编码值
- 日志与错误信息得当

### 效率
- 没有不必要的分配或拷贝
- 数据结构适合使用场景
- 避免 N+1 查询或冗余操作
- 正确使用 async/await 模式

### 一致性
- 遵循代码库已有的代码模式
- 命名约定一致（函数 snake_case、类型 PascalCase）
- 错误处理方式一致
- 模块组织一致

### CLAUDE.md 合规
- 通读项目的 CLAUDE.md，检查所有改动符合其约定
- 检查架构模式（如三层架构、repository 模式）
- 检查数据库使用方式符合项目惯例
- 检查共享状态模式（`Arc<RwLock<T>>` 等）
- 所有 I/O 均为 tokio 运行时上的异步

## 评审流程

1. **确定要评审的变更：**
   - 确定基础分支（有参数用参数，否则 `main`）
   - 运行 `git log <base>..HEAD --oneline` 查看提交
   - 运行 `git diff <base> --stat` 查看变更文件

2. **带标准跑第一轮 Codex 评审：**
   - 用 `codex exec`（不是 `codex review`），这样会话可以低成本 resume 复检：
   ```bash
   codex exec --sandbox read-only "Review the diff of the current branch against the base branch <base-branch>. Run git diff <base-branch> yourself and read any changed files you need.

   Review for: code quality, efficiency, consistency with codebase patterns, and CLAUDE.md compliance. Check error handling, async patterns, data layer architecture, and database usage. Separate blockers (Critical/Important) from Minor findings and false positives." < /dev/null
   ```
   - 重定向 `< /dev/null` 是强制的：`codex exec` 在 prompt 之外还会读 stdin，不关就会永远阻塞。

3. **分析发现并分级：**
   - Critical：安全漏洞、数据丢失风险、破坏性变更、违反 CLAUDE.md
   - Important：bug、逻辑错误、性能问题、与代码库模式不一致
   - Minor/误报：风格偏好、主观改进、非问题

4. **对 Critical 和 Important 发现：**
   - 读相关文件，理解问题
   - 按 CLAUDE.md 约定实施修复
   - 说明修了什么

5. **修复之后：**
   - 暂存变更：`git add <fixed-files>`
   - 提交，消息为 "fix: address code review findings"
   - **通过恢复同一个 Codex 会话来复检**——不要开全新评审。Codex 上下文里已有全部内容，一条简短的跟进 prompt 的 token 和时间成本只是全新评审的零头：
   ```bash
   codex exec resume --last "We fixed the findings: <one-line summary of each fix>. Re-run git diff <base-branch> on the updated code and verify each of your previous findings is resolved. Report only unresolved or newly introduced issues, by severity. If everything is resolved, say so explicitly." < /dev/null
   ```
   - `resume` 要在**与首轮评审相同的工作目录**下运行（`--last` 按 cwd 过滤会话）。
   - 如果 `resume` 失败（找不到会话、CLI 报错），回退为用第 2 步的 prompt 做一次全新完整评审。

6. **重复第 3-5 步，直到：**
   - 只剩 Minor 发现，或
   - 剩余发现都是误报

7. **最终报告：**
   - 已应用修复的摘要
   - 剩余 minor/误报发现清单（如有）
   - 确认评审循环已完成
