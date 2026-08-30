# Agent Context Guideline

This document defines how persistent instructions for coding agents should be organized.

The goal is not to give the agent every possible piece of project knowledge up front. The goal is to provide the **smallest amount of context that is broadly useful**, then let the agent load deeper information when the task actually requires it.

---

## Core Principle

Keep the root agent instruction file small.

For Codex, this usually means `AGENTS.md`.

The root file should contain only information that is:

- broadly applicable across many tasks
- important enough that the agent should almost always know it
- difficult or costly to rediscover from the repository
- related to how the agent should navigate the project

Detailed knowledge should live elsewhere.

```text
AGENTS.md
    ↓
points to relevant documentation
    ↓
agent reads deeper context when needed
```

This is **progressive disclosure**.

---

## What Belongs in AGENTS.md

Good candidates include:

- project-level development principles
- important architectural boundaries
- required workflow before declaring a task complete
- where important documentation lives
- non-obvious build or test commands
- rules that apply to nearly every modification

For example:

```text
Before modifying existing code, follow:
docs/agentic-ai-coding-workflow.md

Architecture documentation:
docs/architecture/

Testing instructions:
docs/testing.md
```

The root file acts primarily as a **map and a small set of global rules**.

---

## What Does Not Belong in AGENTS.md

Avoid filling the root file with information that is:

- relevant only to one subsystem
- easy to discover directly from the repository
- already enforced by tooling
- rarely needed
- overly detailed implementation knowledge
- generated automatically without review

Examples include:

```text
- every coding convention
- descriptions of every directory
- every class and subsystem
- formatter rules already enforced automatically
- linter rules
- long API documentation
- large architecture explanations
```

These consume context without helping most tasks.

Move them into dedicated documentation instead.

---

## Prefer References Over Duplication

Do not copy large amounts of documentation into `AGENTS.md`.

Prefer:

```text
For architecture decisions, read:
docs/architecture.md
```

instead of duplicating the architecture document inside the agent instructions.

This keeps one source of truth and prevents the instructions from becoming stale.

---

## Let Tools Handle Deterministic Rules

Do not spend agent context teaching rules that deterministic tools can enforce reliably.

Prefer:

```text
formatter → formatting
linter    → mechanical style checks
compiler  → type / syntax errors
tests     → behavioral regression
```

Use agent instructions for things that require judgment:

```text
architecture
ownership
design intent
workflow
tradeoffs
project-specific conventions
```

If a rule can be enforced automatically, automation is usually more reliable than repeatedly asking the model to remember it.

---

## Avoid Instruction Accumulation

Do not respond to every AI mistake by permanently adding another instruction.

Otherwise:

```text
mistake
  ↓
new rule
  ↓
another mistake
  ↓
another rule
  ↓
AGENTS.md becomes an instruction dump
```

Before adding a persistent rule, ask:

1. Does this problem happen repeatedly?
2. Will this rule apply to many future tasks?
3. Could tooling enforce it instead?
4. Could better documentation solve the problem?
5. Does the rule belong in a more specific document?

Persistent instructions should remain intentional.

---

## Generated Instructions Are Drafts

Automatic repository analysis can be useful for creating an initial `AGENTS.md`.

However, generated instructions should be treated as a **draft**, not as authoritative project documentation.

Review them manually and remove:

- obvious information
- redundant descriptions
- unnecessary directory summaries
- speculative architecture explanations
- instructions that rarely matter

The final file should contain only information worth permanently occupying agent context.

---

## Project-Base Rule

When creating a new project from `Project-Base`:

1. Keep the inherited global rules.
2. Add project-specific documentation separately.
3. Extend `AGENTS.md` only when the new information is broadly applicable.
4. Prefer linking to project documentation rather than expanding the root file.
5. Periodically remove rules that are obsolete or no longer useful.

The goal is for `AGENTS.md` to remain readable at a glance.

---

## Relationship to the Coding Workflow

This guideline controls **what context the agent receives**.

`agentic-ai-coding-workflow.md` controls **how the agent modifies the system**.

Together:

```text
Agent Context Guideline
        ↓
Give the agent the right information

Agentic AI Coding Workflow
        ↓
Use that information to modify the codebase cleanly
```

The two documents solve different problems and should remain separate.

---

## Rule of Thumb

When considering adding something to `AGENTS.md`, ask:

> Does the agent need to know this for most tasks?

If yes, it may belong there.

If no, put it in a more specific document and let the agent read it when needed.
