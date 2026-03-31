---
name: critique
description: >
  Use when code changes need structured multi-agent review before merging,
  when you want multiple review perspectives on a diff or codebase area,
  or when a quality gate requires blocking-only findings with source anchors.
  Triggers: /critique, "review this", "code review", "quality check",
  "find blocking issues", "multi-angle review".
argument-hint: <what to review> [-a codex|opus] [--refactor]
---

Arguments: $ARGUMENTS

# Critique

目标：在不把用户淹没在长输出里的前提下，用并行审阅提高覆盖率与稳定性。

## When to Use

必须 review：
- 完成主要功能后
- 合并到 main 前

推荐 review：
- 卡住时（换个视角）
- 重构前（建立 baseline）
- 修复复杂 bug 后

## Invocation Boundary

- `critique` 默认由拥有最终 review judgment 和源码复核责任的最高层上下文调用。
- 若当前上下文只是 implementer、reviewer、researcher 之类的内容型 child worker，通常不应再次递归调用 `/critique`。
- Per-task review（例如 `plan-runner` 的 spec/quality loop）继续使用各自的 leaf reviewer prompt；不要在这些 leaf worker 里升级成全局 critique。
- 平台 dispatch 语法和 child-context 概念边界见 `../using-agents/references/architecture.md`。

Per-task review（在 plan-runner 内）不走 `/critique`，直接用 `../plan-runner/code-quality-reviewer-prompt.md`（封装层）dispatch 单个 reviewer。

本 skill 是对 `/orchestrate` 的一层薄封装：

- `/orchestrate` 负责并行调度与 GSA 聚合
- `critique` 负责定义审阅角色、输出契约与聚合后的人工复核规则
- review 专属策略仅在 `critique` 维护，`orchestrate` 保持通用引擎语义

核心思想：

- 角色分工用于覆盖面（不同视角）
- 同质 BoN 用于稳定性（降低随机性）
- GSA 用于聚合（主 agent 批判性合并，过滤无证据建议）

## Output Contract

默认模式只输出 `blocking` 与 `near-blocking`。`non-blocking` 仅在用户明确要求时输出。

reviewer 输出：

```markdown
### F-001 · `blocking`

**Finding:**
这里写较长正文。
**Impact:** short impact
**Trigger:** short trigger sentence

**Anchor:**
path:line
path:line
**Minimal Fix:** shortest safe change
```

主 agent 复核输出：

```markdown
### Main Verification · F-001

**Status:** verified
**Evidence Anchor:** path:line
**Note:** concise verification note
```

展示规则：

- 默认使用 `md` / `markdown` code block 展示
- 字段名用粗体
- 字段顺序默认按 `Finding -> Impact -> Trigger -> Anchor -> Minimal Fix`
- 单个短字段可同一行
- 多个 anchor、较长 finding，或视觉上更好读时可换行
- 短字段紧排；长字段块和列表型字段块之间再留空行
- 同一条输出里允许混用同一行、换行和空行
- 用户明确要求 quote 版本时，可改用 blockquote 形状

## Strategy Selection

默认由 main agent 先判断并选择审阅策略：

- Strategy R (`Role-Diverse`): 变更面广、涉及架构边界或多类风险
- Strategy H (`Homogeneous Cross-Validation`): 变更局部、目标单一、希望降低漏检随机性
- 用户明确指定策略时，按用户指定

是否单列 over-engineering 检查也由 main agent 判断：

- Strategy R 下，可单列专门角色
- Strategy H 下，可并入统一 prompt

row 与 prompt 生成规则：

- row 的数量、命名、关注角度由 AI 按当前任务动态生成
- 不再手动固定为少数预定义 row
- reviewer 总数由 AI 按任务复杂度决定，用户可覆盖

## Reviewer Policy

reviewer 发现必须遵守：

- 先确认实现真正使用的对齐单位、消费边界和语义 owner，再判断问题发生在哪里
- 若问题只来自错误的分析粒度、通用经验类猜测、或未被当前实现触发的路径，不应输出为 blocking 或 near-blocking
- skill 与 reviewer 提示词保持项目无关；项目专有术语只允许作为被审代码中的证据与锚点，不应写成全局审阅准则

当下列信号出现时，默认按 `near-blocking` 输出；只有能证明会造成真实错误或线上风险时才升级为 `blocking`：

- 引入 legacy config key / 宽松解析（coerce bool, 容错类型转换）但没有真实历史用户
- no-op 阶段引入调试开关或观测结构（snapshot/stats/复杂日志骨架）并暴露到外部配置
- 引入 per-forward wrapper/hook 或额外抽象层，且不在计划范围内
- 引入 “expected/enabled” 影子状态变量贯穿运行时（应在 init 断言）
- 计划外扩展范围（例如接入 async 路径、dp_size>1 优化、diffusion 分桶等）
- 配置桥接注入导致归属混乱（把 rollout 策略写回 engine cfg）

这类 finding 还必须满足：

- `finding` 陈述具体事实，不给泛泛的“过度设计”评价
- `finding` 说明当前实现里的触发路径或消费关系
- `minimal_fix` 是最小安全改动

## Main Agent Workflow

主 agent 在并行前负责：

- 界定审阅范围（diff / 目标风险 / 是否包含重构诉求）
- 选择 Strategy R 或 Strategy H
- 决定是否单列 over-engineering 检查

`/orchestrate` 只负责 subagents 并行调度、失败处理与 GSA 聚合。

主 agent 在聚合后必须：

- 只保留带 `anchor` 且可复核的项
- 只保留由当前源码中的真实生产者、消费者、状态迁移或配置消费链支撑的项
- 将 subagent 发现视为假设，回到源码逐条复核后再定性
- 产出 `Main Verification` 输出块

当用户明确要求重构提案时（例如在 $ARGUMENTS 中包含 `--refactor` 或 "重构"），允许附加：

```markdown
### Optional Refactor · R-001

**Proposal:** ...
**Benefit:** ...
**Cost:** ...
**Risk:** ...
**Rollback:** ...
```

`optional_refactors` 不能挤占 blocking 的篇幅，且不得默认输出。

执行顺序：

1. 解析参数：当 `$ARGUMENTS` 中出现 `/codex-exec`、`/codex`、`codex`、或 `-a codex` 时，识别为 agent 类型指令，传递给 orchestrate 作为 `-a codex`。先通过 Skill tool 加载 `/codex-exec`，确保命令模板可用
2. 加载 `/orchestrate`
3. 选择 Strategy R 或 Strategy H，并使用 `-m gsa` 及解析出的 `-a` 参数启动审阅 subagents
4. 收集输出后做 GSA 聚合
5. 主 agent 对所有 blocking 项做源码复核，并输出 `Main Verification` 输出块
