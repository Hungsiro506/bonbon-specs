# Guides — lifecycle docs for the bonbon testing system

How a spec AC here becomes a test scenario in [bonbon](https://github.com/Hungsiro506/bonbon). Written for engineers and PMs. No TypeScript required to read.

## Index

| Guide | For whom | What it covers |
|-------|----------|----------------|
| **[`test-first-with-ai.md`](./test-first-with-ai.md)** | Everyone | The cross-repo workflow. Role split (PM writes AC → engineer writes test → AI bridges), the 6-step golden path, the 1:N rule, the concrete-value requirement, commands cheat sheet, troubleshooting. |
| **[`ac-to-scenario-derivation.md`](./ac-to-scenario-derivation.md)** | Engineers + reviewers | The thinking guide. How to look at an abstract AC, see the hidden variables, and pick the right 2–5 concrete scenarios. Principles, worked example, stop criteria, checklist. |
| **[`prompts/scenario-derivation.md`](./prompts/scenario-derivation.md)** | Engineers | A portable AI prompt. Paste into Claude / ChatGPT / Gemini with three inputs filled in to get ready-to-paste `scenario({...})` blocks. |

## Where do engineering-side docs live?

In the [bonbon repo](https://github.com/Hungsiro506/bonbon/tree/main/docs/testing), not here. Specifically:

- [`.claude/rules/e2e-scenarios.md`](https://github.com/Hungsiro506/bonbon/blob/main/.claude/rules/e2e-scenarios.md) — the enforceable rule; what CI checks.
- [`docs/testing/spec-synced-tests-plan.md`](https://github.com/Hungsiro506/bonbon/blob/main/docs/testing/spec-synced-tests-plan.md) — rollout plan with phase-by-phase deliverables.
- [`docs/testing-strategy.md`](https://github.com/Hungsiro506/bonbon/blob/main/docs/testing-strategy.md) — overall test-pyramid strategy (unit / integration / E2E).

Rule of thumb: **process and product stuff lives here** (`bonbon-specs`), **code-level enforcement lives in `bonbon`**.

## Who edits what

- **PM / stakeholder**: AC prose in `<nnn>-<slug>/spec.md`. Propose changes to the guides above via PR.
- **Engineer**: test implementation in bonbon. Can edit guides here too — they're the same PR workflow.
- **QA**: read-only today; can open PRs for AC additions or guide improvements. A future coverage-matrix PR (Phase 4) will give QA a per-AC coverage view.

## See also

- [Top-level README](../README.md) — how this repo is structured.
- [CONTRIBUTING](../CONTRIBUTING.md) — PR workflow and the append-only rule.
