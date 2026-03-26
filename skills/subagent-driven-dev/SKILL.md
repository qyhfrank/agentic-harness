---
name: subagent-driven-dev
description: Use when executing implementation plans with independent tasks in the current session. Dispatches fresh subagent per task with two-stage review (spec compliance then code quality).
---

Arguments: $ARGUMENTS

# Subagent-Driven Development

每个任务分配一个全新 subagent 执行，完成后经过两阶段 review（spec 合规 + 代码质量），确保质量和迭代速度。

## When to Use

有实现计划，任务基本独立，希望在当前 session 内完成。

vs. `superpowers:executing-plans`（另开 session）：
- 同 session，无上下文切换
- 每任务全新 subagent，无上下文污染
- 两阶段 review 自动化
- 无需人工介入即可连续推进

## The Process

```dot
digraph process {
    rankdir=TB;
    subgraph cluster_per_task {
        label="Per Task";
        "Dispatch implementer (./implementer-prompt.md)" [shape=box];
        "Implementer asks questions?" [shape=diamond];
        "Answer questions" [shape=box];
        "Implementer implements, tests, commits, self-reviews" [shape=box];
        "Dispatch spec reviewer (./spec-reviewer-prompt.md)" [shape=box];
        "Spec compliant?" [shape=diamond];
        "Fix spec gaps" [shape=box];
        "Dispatch code quality reviewer (./code-quality-reviewer-prompt.md)" [shape=box];
        "Quality approved?" [shape=diamond];
        "Fix quality issues" [shape=box];
        "Update ledger, mark task complete" [shape=box];
    }
    "Read plan, extract tasks, init ledger + TodoWrite" [shape=box];
    "More tasks?" [shape=diamond];
    "Final review via /critique" [shape=box];
    "superpowers:finishing-a-development-branch" [shape=box style=filled fillcolor=lightgreen];
    "Read plan, extract tasks, init ledger + TodoWrite" -> "Dispatch implementer (./implementer-prompt.md)";
    "Dispatch implementer (./implementer-prompt.md)" -> "Implementer asks questions?";
    "Implementer asks questions?" -> "Answer questions" [label="yes"];
    "Answer questions" -> "Dispatch implementer (./implementer-prompt.md)";
    "Implementer asks questions?" -> "Implementer implements, tests, commits, self-reviews" [label="no"];
    "Implementer implements, tests, commits, self-reviews" -> "Dispatch spec reviewer (./spec-reviewer-prompt.md)";
    "Dispatch spec reviewer (./spec-reviewer-prompt.md)" -> "Spec compliant?";
    "Spec compliant?" -> "Fix spec gaps" [label="no"];
    "Fix spec gaps" -> "Dispatch spec reviewer (./spec-reviewer-prompt.md)" [label="re-review"];
    "Spec compliant?" -> "Dispatch code quality reviewer (./code-quality-reviewer-prompt.md)" [label="yes"];
    "Dispatch code quality reviewer (./code-quality-reviewer-prompt.md)" -> "Quality approved?";
    "Quality approved?" -> "Fix quality issues" [label="no"];
    "Fix quality issues" -> "Dispatch code quality reviewer (./code-quality-reviewer-prompt.md)" [label="re-review"];
    "Quality approved?" -> "Update ledger, mark task complete" [label="yes"];
    "Update ledger, mark task complete" -> "More tasks?";
    "More tasks?" -> "Dispatch implementer (./implementer-prompt.md)" [label="yes"];
    "More tasks?" -> "Final review via /critique" [label="no"];
    "Final review via /critique" -> "superpowers:finishing-a-development-branch";
}
```

## Handling Implementer Status

Implementer 返回四种状态：

**DONE:** 进入 spec compliance review。

**DONE_WITH_CONCERNS:** 实现完成但有疑虑。先读 concerns：正确性/范围问题先处理再 review；观察性备注（如"文件变大了"）记录后继续 review。

**NEEDS_CONTEXT:** 缺少信息。补充上下文后重新 dispatch。

**BLOCKED:** 无法完成。评估原因：
1. 上下文不足 → 补充后重新 dispatch
2. 推理能力不够 → 用更强模型重新 dispatch
3. 任务过大 → 拆分
4. 计划本身有误 → 上报用户

不要忽略 escalation，不要让同一模型在没有变化的情况下重试。

## Task Ledger

任务状态持久化到项目级 `.agents/tasks.json`，确保跨会话、跨 CLI 可恢复。

ledger 是权威源。先写 ledger，再从 ledger 同步 TodoWrite。
启动时：若 `.agents/tasks.json` 存在且有未完成任务，从 ledger 恢复进度并重建 TodoWrite 状态，而非重新提取。
每次状态变更（dispatch、review pass/fail、complete）时先更新 ledger，再同步 TodoWrite。

Schema：

- `plan_source`: 计划文件路径
- `tasks[].id`: 任务序号
- `tasks[].subject`: 任务标题
- `tasks[].status`: `pending | in_progress | review | completed | blocked`
- `tasks[].owner`: 当前负责的 subagent 类型
- `tasks[].review_log`: spec/quality review 结果摘要
- `tasks[].updated_at`: ISO 时间戳

`.agents/` 目录应加入 `.gitignore`（运行时状态不入 repo）。

## Prompt Templates

- `./implementer-prompt.md` — dispatch implementer subagent
- `./spec-reviewer-prompt.md` — dispatch spec compliance reviewer
- `./code-quality-reviewer-prompt.md` — dispatch code quality reviewer

## Example Workflow

```
[读取计划，提取全部 5 个任务，初始化 .agents/tasks.json + TodoWrite]

Task 1: Hook installation script
[Dispatch implementer，提供完整任务文本 + 上下文]
Implementer: "hook 装在 user 还是 system level？"
→ 回答后重新 dispatch
Implementer: 实现完成，5/5 测试通过，已 commit
[Dispatch spec reviewer] → ✅ 合规
[Dispatch code quality reviewer] → ✅ 通过
[标记 Task 1 完成]

Task 2: Recovery modes
[Dispatch implementer]
[Dispatch spec reviewer] → ❌ 缺少 progress reporting
[Implementer 修复] → [Re-review] → ✅
[Dispatch code quality reviewer] → ✅
[标记 Task 2 完成]

... 剩余任务同理 ...

[全部完成后 dispatch /critique 做最终 review]
```

## 优势

- 每任务全新 subagent，无上下文污染
- 两阶段 review（spec → quality）自动化质量门禁
- Spec compliance 防止过度/不足构建
- Controller 预提取任务文本，subagent 无需读文件
- 成本更高（每任务 3 次 subagent），但问题发现更早

## Red Flags

Never:
- 未经用户同意在 main/master 上开始实现
- 跳过任何一阶段 review（spec compliance 或 code quality）
- 带着未修复的问题继续
- 并行 dispatch 多个 implementer（文件冲突）
- 让 subagent 自己读计划文件（应提供完整文本）
- 省略场景上下文（subagent 需要理解任务在整体中的位置）
- 忽略 subagent 提问（回答后再让其继续）
- spec compliance 有问题时接受"差不多"
- 跳过 review loop（reviewer 发现问题 → implementer 修复 → 再次 review）
- 用 implementer self-review 替代正式 review（两者都需要）
- 在 spec compliance ✅ 之前开始 code quality review（顺序不可颠倒）
- 任一 review 有未关闭问题时进入下一任务

Subagent 提问时：完整回答，必要时补充上下文，不要催促实现。
Reviewer 发现问题时：implementer 修复 → reviewer 再次 review → 循环直到通过。
Subagent 失败时：dispatch 新 subagent 修复，不要手动修（上下文污染）。

## Integration

必需的工作流 skill：
- `superpowers:using-git-worktrees` — 开始前建立隔离工作区
- `superpowers:writing-plans` — 创建本 skill 执行的计划
- `superpowers:finishing-a-development-branch` — 全部任务完成后收尾

Per-task review 使用本 skill 内置的 `./code-quality-reviewer-prompt.md` 模板。
全量 review（最终 review 或独立 review）使用 `/critique`。

Subagent 推荐使用：
- `superpowers:test-driven-development` — 每个任务遵循 TDD
