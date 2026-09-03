---
name: workflow-routing
description: Plan workflow depth and cross-runtime model routing for a Bounded or Architectural coding task, then write branch_doc/<branch-name>/model-routing.md. Do not use for Trivial tasks or to execute later workflow steps.
---

# Workflow Routing

Use this skill once during Step 0, after the task has been classified as Bounded or Architectural. Select both the executor and model; Codex and Claude Code are separate runtimes, so routing may require a user handoff.

## Preconditions

- Read the current Git branch without changing it.
- Stop if HEAD is detached or the branch name contains `/`.
- Write only to `branch_doc/<branch-name>/model-routing.md`.
- Do not switch models, start another runtime, or execute later Steps.

## Workflow depth

| Stage | Bounded | Architectural |
|---|---|---|
| Control | Step 0 chooses; default Step-gated | Step-gated |
| 1. Spec | Lightweight acceptance criteria | Full spec, design, and plan |
| 2. Implementation | Scoped | Staged |
| 3. Correctness | Focused tests + risk-based regression | Comprehensive verification |
| 4. Architecture | Short gate | Full gate |
| 5. Simplification | When justified findings exist | Resolve all findings; no change is a valid conclusion |
| 6. Regression | If Step 3 is followed by changes | Final regression |

## Executor and model routing

> **Snapshot — 2026-09-02 (Asia/Taipei):** Re-evaluate concrete models at least monthly or after a major release. Preserve implementer/reviewer separation when models change.

Use the following quality-first routing when both Codex and Claude Code are available:

`Sol` runs in Codex; `Opus5` runs in Claude Code.

| Step | Ordinary Bounded | High-risk Bounded | Architectural |
|---|---|---|---|
| **0. Classification** | Sol | Sol | Sol |
| **1. Spec** | Sol | Sol | Sol |
| **2. Implementation** | Opus5 | Opus5 | Opus5 |
| **3. Correctness** | Sol (fresh) | Sol (fresh) | Sol (fresh) |
| **4. Architecture** | Sol | Sol | Sol |
| **5. Simplification** | Opus5* | Opus5* | Opus5* |
| **4. Recheck** | Sol* | Sol* | Sol* |
| **6. Regression** | Sol* | Sol* | Sol |

- `*`: Run only when the workflow condition triggers it.
- `fresh`: Start a reviewer context that reads only `spec.md`, the repository, actual diff, and tests before independently judging correctness and architecture.
- High-risk Bounded applies to persistence, state transitions, public contracts, security, or behavior that is difficult to regress. It changes routing, not workflow depth.

## Output

Create `branch_doc/<branch-name>/model-routing.md` with:

```markdown
# Model Routing

Branch: <branch-name>
Task class: <Bounded or Architectural>
Routing tier: <Ordinary, High-risk, or Architectural>
Control mode: <Continuous or Step-gated>
Workflow path: <ordered Steps and conditional loops>

| Step | Executor | Model | Context | Condition |
|---|---|---|---|---|
| ... | ... | ... | ... | ... |
```

Include only the selected task profile. Use `Current`, `Fresh`, or `Handoff` for Context, and state each conditional Step explicitly. After writing the file, report its path and the next executor/model to the user. The file is a handoff plan, not authorization to switch runtimes or perform later Steps.
