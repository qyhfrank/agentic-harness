---
name: orchestrate
description: Orchestration engine for parallel agent dispatch with aggregation. Supports task decomposition (Per-Item), Best-of-N sampling with voting, and Generative Self-Aggregation (GSA). Use when a task benefits from multiple perspectives, needs fault tolerance, or can be decomposed into independent subtasks.
argument-hint: <task> [-m item|bon|gsa] [-n N] [-a <type>[:<count>]]... [-B]
---

Arguments: $ARGUMENTS

# Parallel Agent Orchestration

## Arguments

| 短写 | 长写           | 默认值   | 说明                                     |
| ---- | -------------- | -------- | ---------------------------------------- |
| `-m` | `--mode`       | 自动推断 | 模式：`item` / `bon` / `gsa`             |
| `-n` | `--agents`     | 5        | agent 数量                               |
| `-a` | `--agent`      | 当前模型 | agent 类型，可多次指定，支持 `:count`    |
| `-B` | `--background` | 否       | background child context，主 agent 可同时工作 |

示例：
```
/orchestrate 分析这个 bug 的根因
/orchestrate 分析这个 bug -n 10
/orchestrate 分析这个 bug -a codex:3 -a opus:2
/orchestrate 分析这个 bug -B
```

## When to Use

适用：
- 2+ 独立任务/故障，无共享状态或顺序依赖
- 每个问题可独立理解，不需要其他问题的上下文
- 各 agent 不会编辑相同文件或竞争相同资源
- 当前上下文拥有完整工作上下文与最终 synthesis 权

不适用：
- 故障相互关联（修一个可能修好其他的）
- 需要完整系统状态才能判断
- 尚未定位问题域（先做探索性调试）
- 当前 agent 已能完成剩余串行步骤（不要为了委派而委派）
- 当前上下文只是一个内容型 child worker，而 parent 才拥有聚合与决策职责

## Ownership

- `orchestrate` 是并行 dispatch 和 aggregation 的 engine，不是平台 dispatch 语法手册。
- 平台语法和 child-context 概念边界见 `../using-agents/references/architecture.md`。
- 平台能力差异见 `../using-agents/references/platform-matrix.md`。
- 默认应在拥有最终 synthesis 的最高层上下文调用。若当前上下文只是一个前台内容 worker，通常应该返回分析给 parent，而不是再嵌套调用 `/orchestrate`。

## Orchestration Heuristics
- 如果任务本质上是 deterministic waiter 或 scriptable monitor，优先使用 tool execution，不要默认调用 `/orchestrate`。
- 平台支持 background tool execution 且 parent 还有有意义的独立工作时，deterministic waiter 优先 background tool execution。
- deterministic wait 只有在条件本身需要持续推理或综合多源证据时，才升级为 child context。
- 广泛研究用广度编排，狭窄困难任务用聚焦深度推理。有深度推理 runner（如 `/codex-exec`，通过 `-a codex` 指定）时，优先用于狭窄难题
- 模型选择：prefer opus。不选 sonnet 或 haiku
- 最小有效扇出，但独立任务不设人为低上限
- 非阻塞工作用 background child context；需要结果才能继续时用 foreground child context
- 代码写入类 worker：优先隔离 worktree 或明确文件归属

## Agent Prompt Crafting

每个 agent prompt 必须完整自包含，不用占位符。包含：
- 明确的 scope（一个问题域）
- 清晰的目标
- 约束条件（不改哪些代码）
- 期望的输出格式

常见错误：

| 错误 | 正确 |
| --- | --- |
| "修所有测试" (scope 过宽) | "修 agent-tool-abort.test.ts" (聚焦) |
| "修这个竞态" (缺上下文) | 贴上错误信息和测试名 |
| 无约束 (agent 大范围重构) | "不要改生产代码" / "只改测试" |
| "修好它" (输出模糊) | "返回根因和改动摘要" |

## Execution Steps

1. **解析参数**（见上表）

2. **前置路由判断**
   - 若任务是 deterministic waiter / scriptable monitor，先按 `../using-agents/references/platform-matrix.md` 选择 tool execution surface，而不是直接进入 child-context fan-out。
   - 只有确认任务真的需要多个 child contexts 时，才继续下面的 orchestration 步骤。

3. **推断模式**（用户未指定时）
   - 任务包含多个独立项目（如"处理这 N 个文件"）→ **Per-Item**
   - 任务有明确答案且可验证（数学、代码）→ **Best-of-N**
   - 其他情况 → **GSA**

4. **启动 agents**：在一个 response 中发起多个并行调用，确保并行执行
   - 默认模式：使用平台的 foreground child-context surface，主 agent 等待所有结果返回
   - `--background` 模式：使用平台的 background child-context surface，仅当 parent 还有有意义的独立工作时启用

   Dispatch 路由（按 `-a` 类型选择执行路径）：
   - 默认 / `-a opus`：使用当前平台的默认 coding child worker
   - `-a codex`：先通过 Skill tool 加载 `/codex-exec`，再按其命令模板通过 Bash 调用 `codex exec`。`codex-exec` 是 Thinker agent type adapter，不定义 foreground/background 概念本身。
   - 混合模式（如 `-a codex:3 -a opus:2`）：按类型分别路由，结果统一进入聚合阶段

   任务分配：
   - Per-Item：每个 agent 收到不同子任务
   - Best-of-N / GSA：每个 agent 收到相同任务

   多样性来源：
   - 指定多种 agent 类型 → 不同模型视角
   - 单一类型 → 依赖采样随机性

5. **收集输出 + 失败处理**
   - 成功收集 → 继续
   - 部分失败 → 重试失败的 agent（最多 2 次）
   - 重试后仍失败 → 用已成功的继续（≥2 个即可聚合，仅 1 个则降级为单结果）
   - 全部失败 → 报错，建议用户降低 N 或检查任务

6. **聚合**
   - Per-Item：合并各 agent 输出
   - Best-of-N：投票或主 agent 评估选最佳
   - GSA：将 N 个输出注入 prompt，合成最终答案
     > "回顾上述方案，识别共同点和差异，对比各 agent 引用的 evidence 是否指向同一源（代码位置、文档段落、命令输出），综合有 evidence 支撑的优点，剔除缺乏依据的结论，生成最完善的最终答案"

7. **实证验证**（仅当有可验证路径时）

   根据各 agent 线索和聚合结论，回到源头验证：

   | 任务类型 | 验证方式 |
   |----------|----------|
   | 代码分析 | 读取相关文件，确认结论与代码一致 |
   | Bug 定位 | 检查指出的代码行，验证问题真实存在 |
   | 代码生成 | 执行测试或静态检查 |
   | 文档查询 | 回到原文档确认引用准确 |

   跳过：纯创作任务、无 codebase、无可执行环境

## Mode Reference

| 模式 | 机制 | 适用场景 |
|------|------|----------|
| Per-Item | 拆分任务，各处理一部分 | 批量处理、独立子任务 |
| Best-of-N | 同一任务 N 次采样，选最佳 | 有明确答案的推理、代码 |
| GSA | N 次采样后合成新答案 | 开放式任务、方案设计 |

## Constraints

- 每个子任务/采样必须独立可执行
- 文件模式（N > 20 且输出大）：写入 `/tmp/claude-parallel/{task-id}/`

## Review Integration (Engine Only)

`/orchestrate` 在 review 场景中只负责并行调度与聚合，不承载 review 策略本身。

- 推荐优先使用 `/critique` 承载 review policy（策略选择、输出契约、主验证规则）
- `orchestrate` 不定义固定角色、over-engineering 重点或 refactor policy
- 若调用方直接用 `/orchestrate` 做 review，需在调用 prompt 中自带完整策略与输出契约
