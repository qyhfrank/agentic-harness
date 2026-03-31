---
name: critique
description: Use when code changes need structured review with anchored findings, including spec compliance gates, code quality gates, plan/config review, or full multi-angle reviews. Triggers on /critique, "review this", "code review", "quality check", "spec compliance check", "review the plan", "review the config", "find blocking issues".
argument-hint: <what to review> [--spec|--quality|--plan] [-a thinker|doer|codex|opus] [--refactor]
---

Arguments: $ARGUMENTS

# Critique

Review stage abstraction: select a profile and engine, get structured findings with source anchors.

## Profiles and Engines

| Profile | What | Default Engine |
|---|---|---|
| `spec` | Spec compliance: does implementation match requirements? | `single` |
| `quality` | Code quality: clean, tested, maintainable? | `single` |
| `plan` | Plan/config review: complete, coherent, ready for execution? | `single` |
| `full` | Multi-angle review with role diversity / cross-validation | `fanout` |

| Engine | How |
|---|---|
| `single` | One reviewer child context, no `/fanout`. For focused per-task gates. |
| `fanout` | Parallel reviewers via `/fanout`. For full reviews and merge gates. |

Arguments:

- `--spec`: profile=spec, engine=single
- `--quality`: profile=quality, engine=single
- `--plan`: profile=plan, engine=single
- No flag: profile=full, engine=fanout
- `-a thinker|doer|codex|opus`: explicit worker override for the full-profile fanout engine. No flag -> let `/fanout` infer the worker type.
- `--refactor`: include optional refactor proposals (full profile only)

## Verdict Envelope

All profiles return a structured verdict:

```markdown
**Verdict:** pass | fail | needs_escalation
**Findings:** F-001, F-002, ... (or "none")
**Assessment:** <one-line summary>
**Coverage:** requirements checked, files reviewed, risk dimensions assessed
**Unchecked:** areas not reviewed or not reviewable via static analysis
```

- `pass`: no blocking findings within checked scope. Caller may proceed, but `pass` is bounded by coverage — it does not assert correctness beyond what was checked.
- `fail`: blocking findings exist, caller must address
- `needs_escalation`: cannot determine, needs broader context or human judgment

Coverage fields are mandatory when verdict is `pass`. "No findings" is only meaningful when accompanied by coverage evidence (see `verification-gate.md` stage 5).

Single-reviewer verdicts are gate inputs, not completion oracles. When used inside `/harness`, a single-reviewer pass does not grant close authority.

## Profile: spec

Read `references/spec-review-profile.md` for the reviewer prompt.

Input: task requirements text + implementer's claimed changes.

Checks: missing requirements, extra unneeded work, misunderstandings. Verify by reading code, not by trusting the implementer report.

## Profile: quality

Read `references/quality-review-profile.md` and `references/code-reviewer.md`.

Input: BASE_SHA, HEAD_SHA, implementation description, plan/requirements reference.

Prerequisite: spec compliance must pass first when both profiles are used in sequence.

## Profile: plan

Read `references/plan-review-profile.md`.

Input: planning artifact (plan document, harness config.yaml, or task strategy) + goal/spec reference.

Use after: plan mode produces a plan, harness scaffold/plan finishes config.yaml, or any planning artifact is ready for execution. Checks completeness, goal alignment, boundary coverage, verification readiness.

## Profile: full

Multi-angle review. Existing behavior unchanged.

### When to Use Full

必须 review：
- 完成主要功能后
- 合并到 main 前

推荐 review：
- 卡住时（换个视角）
- 重构前（建立 baseline）
- 修复复杂 bug 后

### Strategy Selection

默认由 main agent 先判断并选择审阅策略：

- Strategy R (`Role-Diverse`): 变更面广、涉及架构边界或多类风险
- Strategy H (`Homogeneous Cross-Validation`): 变更局部、目标单一、希望降低漏检随机性
- 用户明确指定策略时，按用户指定

是否单列 over-engineering 检查也由 main agent 判断：

- Strategy R 下，可单列专门角色
- Strategy H 下，可并入统一 prompt

row 与 prompt 生成规则：

- row 的数量、命名、关注角度由 AI 按当前任务动态生成
- reviewer 总数由 AI 按任务复杂度决定，用户可覆盖

### Core Thinking (Full Profile)

- 角色分工用于覆盖面（不同视角）
- 同质采样用于稳定性（降低随机性）
- sample 聚合（主 agent 批判性合并，过滤无证据建议）

## Invocation Boundary

- `critique` 默认由拥有最终 review judgment 的最高层上下文调用。
- 若当前上下文只是 implementer、reviewer、researcher 之类的内容型 child worker，通常不应再次递归调用 `/critique`。
- `/planning` 的 per-task review 通过 `/critique --spec` 和 `/critique --quality`（single engine）调用。最终全量 review 通过默认 `/critique`（full profile）。

## Output Contract

默认模式只输出 `blocking` 与 `near-blocking`。`non-blocking` 仅在用户明确要求时输出。

Finding 格式（所有 profile 共用）：

```markdown
### F-001 · `blocking`

**Finding:**
具体事实描述。
**Impact:** short impact
**Trigger:** short trigger sentence

**Anchor:**
path:line
**Minimal Fix:** shortest safe change
```

Full profile 的主 agent 复核输出：

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
- 短字段紧排；长字段块之间留空行
- 用户明确要求 quote 版本时，可改用 blockquote 形状

当用户明确要求重构提案时（`--refactor`），允许附加：

```markdown
### Optional Refactor · R-001

**Proposal:** ...
**Benefit:** ...
**Cost:** ...
**Risk:** ...
**Rollback:** ...
```

## Reviewer Policy

所有 profile 的 reviewer 发现必须遵守：

- 先确认实现真正使用的对齐单位、消费边界和语义 owner，再判断问题发生在哪里
- 若问题只来自错误的分析粒度、通用经验类猜测、或未被当前实现触发的路径，不应输出为 blocking 或 near-blocking
- skill 与 reviewer 提示词保持项目无关；项目专有术语只允许作为被审代码中的证据与锚点

当下列信号出现时，默认按 `near-blocking` 输出；只有能证明会造成真实错误或线上风险时才升级为 `blocking`：

- 引入 legacy config key / 宽松解析但没有真实历史用户
- no-op 阶段引入调试开关或观测结构并暴露到外部配置
- 引入 per-forward wrapper/hook 或额外抽象层，且不在计划范围内
- 引入影子状态变量贯穿运行时（应在 init 断言）
- 计划外扩展范围
- 配置桥接注入导致归属混乱

这类 finding 还必须满足：

- `finding` 陈述具体事实，不给泛泛的"过度设计"评价
- `finding` 说明当前实现里的触发路径或消费关系
- `minimal_fix` 是最小安全改动

## Execution

### Single Engine (--spec, --quality)

1. 解析参数，确定 profile
2. 启动单个 reviewer child context，使用对应 profile 的 prompt template
3. 收集 verdict envelope
4. 主 agent 验证 anchor 是否指向真实源码位置（轻量复核）
5. 返回 verdict

### Fanout Engine (full, default)

1. 解析参数：当 `$ARGUMENTS` 中出现 `-a` 参数时，传递给 `/fanout`。若请求了 `codex` worker，先通过 Skill tool 加载 `/codex-exec`，确保命令模板可用
2. 加载 `/fanout`
3. 选择 Strategy R 或 Strategy H，并使用 `-m sample` 及解析出的 `-a` 参数启动审阅 subagents
4. 收集输出后做 sample 聚合
5. 主 agent 对所有 blocking 项做源码复核，并输出 `Main Verification` 输出块
6. 返回 verdict envelope

## Reference Index

| File | Purpose | Used by |
|---|---|---|
| `references/spec-review-profile.md` | Spec compliance reviewer prompt | spec profile |
| `references/quality-review-profile.md` | Code quality reviewer prompt wrapper | quality profile |
| `references/plan-review-profile.md` | Plan/config reviewer prompt | plan profile |
| `references/code-reviewer.md` | Base code review agent prompt + policy | quality profile, full profile |
