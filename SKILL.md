---
name: codexcc
description: Use when the user says "codexcc" or describes a complex multi-module task that should be decomposed by Codex before parallel execution — refactors, multi-file features, cross-cutting bug investigations, or any task the user wants Codex to plan and then CC agents to execute and then Codex to verify.
---

# CodexCC

**Codex 当脑，CC 当手。** 先判断任务复杂度，再决定是否进入三步闭环：

```
用户任务 → 复杂度自检 → 简单任务直接做 / 复杂任务进入三步闭环
```

## Complexity Gate

在启动 `/codex:rescue`、Workflow 多 agent、`/codex:review` 之前，先做一次任务复杂度自检：

1. **Single file?** 任务是否只需要修改或检查一个文件？
2. **Trivial change?** 任务是否是局部、机械、低风险的小改动，例如修 typo、改一行配置、加一条日志、调整一处文案？
3. **No architectural impact?** 任务是否不改变模块边界、公共 API、数据流、权限模型、构建流程、测试策略或跨文件行为？

判定规则：

- 如果三个问题答案都是 **Yes**，跳过 Codex analyze → multi-agent execute → Codex verify，直接由当前模型完成。
- 如果任一问题答案是 **No**，进入完整三步闭环。
- 如果答案不确定，按 **No** 处理，进入完整三步闭环。

## The Loop

仅当 Complexity Gate 判定任务不是简单任务时，执行以下流程：

### Step 1 — Analyze

将用户任务转发给 `/codex:rescue`（默认 `--background`，任务很小才 `--wait`）。prompt 中要求 Codex 输出：

- 子任务清单，每个有明确边界和交付物
- 子任务间的依赖关系
- 每个子任务的验收标准

拿到 Codex 输出后，向用户展示拆解结果，确认后进入执行。

### Step 2 — Execute

用 Workflow 的 `pipeline()` 将子任务分派给 agent 并行执行。默认 pipeline（无 barrier），仅当后续阶段需要全部前置结果时才用 `parallel()`。

- 无依赖的子任务 → 同一 stage 内并发
- 有依赖的 → 串行 stage
- 每个 agent 完成后检查其输出是否符合 Step 1 的验收标准

### Step 3 — Verify

所有 agent 完成后，运行 `/codex:review --background` 验收全部改动。关键路径用 `/codex:adversarial-review`。

Codex 发现问题 → 将问题路由回对应 agent 修复 → 修复后重新 `/codex:review`。循环直到通过或无 CRITICAL 问题。

## 触发方式

用户只需说：

```
codexcc: 重构 user-auth 模块，支持 OAuth2
```

或直接描述复杂任务时提 "用 codexcc 做"。

## 守则

- **先过 Complexity Gate**：只有当任务不是单文件、不是 trivial change、或存在架构影响时，才启动三步闭环
- **复杂任务 Step 1 不跳过**：一旦进入三步闭环，必须先让 Codex 拆。Codex（gpt-5.4）在深度推理上可能比当前模型强，拆出来的结构往往比自己想的更完整
- **拆解结果必须用户确认**：Codex 理解可能偏差，执行前让用户看一眼
- **验收不省略**：多 agent 并行改代码，合并后出冲突或逻辑不一致的概率远高于单人改动。Codex review 是最后一道防线
- **简单任务直接做**：单文件小改、修 typo、加一行日志 — 直接用当前模型完成，别走三轮。codexcc 的 overhead 只对复杂任务值回票价
