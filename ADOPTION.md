# Adopt Project-Base in an Existing Project

Use this guide when applying Project-Base to a repository that already has code, tooling, documentation, or agent instructions. Project-Base provides defaults; it does not automatically override the target project's existing rules.

## 1. Inspect the Existing Project

Before copying anything, inspect the repository for:

- existing `AGENTS.md` files or other agent instruction entrypoints
- build, test, lint, formatting, schema, migration, and deployment instructions
- architecture, contribution, compatibility, and security documentation
- modified, staged, and untracked working-tree files

Treat existing instructions and working-tree changes as project-owned content. Do not overwrite them blindly.

## 2. Merge `AGENTS.md`

Merge the Project-Base entrypoint into the existing root `AGENTS.md`. If the project does not have one, create it from the Project-Base version.

Preserve project-specific commands, boundaries, and restrictions. Resolve conflicts explicitly instead of keeping two contradictory rules or assuming that the Project-Base default always wins. Keep the final root file concise and use links to detailed documentation.

## 3. Copy the Workflow Documents

Copy these files into the target repository:

```text
docs/agentic-ai-coding-workflow.md
docs/agent-context-guideline.md
```

Keep them under version control so changes to the workflow can be reviewed with the project. Confirm that the merged `AGENTS.md` points to the copied paths and requires the coding workflow to be followed before modifying existing code.

## 4. Adapt the Workflow to the Project

Review the copied defaults against the target project's actual practices. Pay particular attention to:

- existing test-first, test isolation, mocking, coverage, and regression requirements
- the availability and cost of focused, integration, acceptance, and full regression suites
- build, lint, formatting, schema, generated-file, and migration rules
- public API, data format, backward-compatibility, deployment, and security constraints
- directories or subsystems with different ownership or more specific instructions

When a generic default conflicts with a valid project rule, update the target project's local copy or add a focused project-specific rule. Do not change the shared Project-Base merely to accommodate one repository's special case.

## 5. Restart and Validate the Adoption

Start a new agent session from the target repository so its instruction entrypoints are loaded. Then exercise the workflow with representative tasks:

- **Trivial:** a documentation or clearly behavior-neutral change
- **Bounded:** a local bug fix or small feature
- **Architectural:** a cross-module or responsibility-changing scenario

Confirm that the agent reads the workflow, preserves existing changes, chooses proportionate planning and tests, performs the appropriate Architecture Gate, and reports verifiable completion evidence. Fix unclear or conflicting instructions discovered during this validation before relying on the workflow for larger changes.
