---
name: codexcc
description: Use when the user says "codexcc" or describes a complex multi-module task that should be decomposed by Codex before parallel execution — refactors, multi-file features, cross-cutting bug investigations, or any task the user wants Codex to plan and then CC agents to execute and then Codex to verify.
---

# CodexCC

**Codex 当脑，CC 当手。** 三步闭环：

```
用户任务 → /codex:rescue 拆解 → Workflow 多 agent 并行做 → /codex:review 验收
```

## The Loop

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

- **Step 1 不跳过**：哪怕任务看起来简单，先让 Codex 拆。Codex（gpt-5.4）在深度推理上可能比当前模型强，拆出来的结构往往比自己想的更完整
- **拆解结果必须用户确认**：Codex 理解可能偏差，执行前让用户看一眼
- **验收不省略**：多 agent 并行改代码，合并后出冲突或逻辑不一致的概率远高于单人改动。Codex review 是最后一道防线
- **简单任务别用**：单文件小改、修 typo、加一行日志 — 直接用 CC，别走三轮。codexcc 的 overhead 只对复杂任务值回票价
