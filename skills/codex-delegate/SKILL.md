---
name: codex-delegate
description: "Orchestrate the local codex CLI as specialized subagents under a Tech Lead workflow. Use when delegating multi-file exploration, implementation, or review to local Codex while keeping user-aligned planning, open design decisions, minimal-change gates, and concise user reporting with the Tech Lead."
---

# Codex Delegate (Local Codex CLI as Tech Lead Subagent)

This skill defines how an orchestrating **Tech Lead** agent delegates heavy coding work to the local `codex` CLI as an isolated subagent.
The Tech Lead owns **complete planning, surfacing open decisions to the user, locking down constraints and test budgets, screening diff scope, and concise user reporting**; Codex executes heavy exploration, multi-file edits/verification, and optional independent review.

---

## 1. Core Problems This Workflow Solves

- **Incomplete planning & missed user decisions**: Codex often skips surfacing critical product or design trade-offs and proceeds on silent assumptions misaligned with user intent. The Tech Lead closes this gap by synthesizing exploration into a full plan and aligning open decisions with the user before implementation begins.
- **Over-defensive design that hurts flexibility & robustness**: Left unconstrained, Codex tends to add arbitrary restrictions, rigid validations, speculative guards, and bloated state/test machinery. The Tech Lead prevents this upfront with explicit `Non-goals` and `Test budget` in the brief, and performs deep diff audits when triggered.
- **Verbose, distracting output**: Read-only exploration by Codex is usually reliable, though its prose contains heavy caveats and noise. The Tech Lead handles all Codex interaction in English behind the scenes, accepts grounded `researcher` facts directly, and presents only clear, signal-dense outcomes in the user's language.

---

## 2. Delegation Gate: Direct vs. Delegated

- **Handle directly in the Tech Lead thread (do not spawn Codex)**: pure conceptual/design questions, localized lookups answered by reading 1–2 files, or small, clear edits (1–2 files, minimal lines).
- **Delegate to Codex when any of the following apply**:
  - the change spans multiple files, modules, or subsystems;
  - broad codebase exploration or cross-component call-flow tracing is needed (to keep Tech Lead context lean);
  - independent workstreams can run in parallel;
  - the task requires adding specified test scenarios, multi-round build/test troubleshooting, or an independent high-risk diff review.

---

## 3. Three-Role Model (Default `gpt-6.2-astra` at `high`)

Use `-m gpt-6.2-astra -c 'model_reasoning_effort="high"'` uniformly across all roles unless the user explicitly requests another model or effort level, with the Tech Lead orchestrating each step from outside Codex:

| Role | Sandbox | Session Strategy & Tech Lead Handling |
| :--- | :--- | :--- |
| **`researcher`** | `read-only` | Codebase discovery, call-chain tracing, and read-only fault reproduction. **Trust grounded findings directly** (spot-check key locations only when making a major architectural bet with lingering doubt). |
| **`worker`** | `workspace-write` | Implements the scoped change, adds only the exact Tech-Lead-specified tests, and iterates until checks pass. Prefer `resume` on the same task's `researcher` session; pass `--worktree` when running concurrent write workers. |
| **`reviewer`** | `read-only` | **Only on explicit review request or high-risk changes.** Must run in a **fresh `read-only` session** (independent from `worker` bias). Tech Lead and user triage findings; **never blindly feed all review comments back into `worker` in a loop**. |

---

## 4. Workflow Loop

1. **Explore (`researcher`)**: Dispatch read-only exploration (fanning out independent paths in parallel). Do not re-read the codebase to double-check returned facts.
2. **Plan & User Alignment Gate (Tech Lead ↔ User)**: Before launching `worker`, synthesize the findings into a clear plan and **explicitly present any open design choices, behavioral ambiguities, edge-case handling policies, and non-goals for the user to decide** (skip interactive questions only when the fix has a single obvious path).
3. **Execute (`worker`)**: Resume the task session with `-s workspace-write` and a bounded six-part English brief (locking in the user-approved choices, constraints, and test budget).
4. **Verify & Deliver (Tech Lead)**: Run the lightweight `--stat` gate by default; run a deep diff audit or `reviewer` pass only when manually requested by the user or triggered by clear `--stat` warning signals.

---

## 5. CLI Usage, Session Reuse & Parallelism

1. **Language isolation**: always write every brief, follow-up, `resume` prompt, and review instruction sent to Codex **in English**, regardless of the user's language; always reply to the user in the user's language.
2. **Initial launch & async wait**
   ```sh
   codex exec -C <repo_root> -s <read-only|workspace-write> -m gpt-6.2-astra -c 'model_reasoning_effort="high"' -o <out.md> "<English brief>"
   ```
   - Always pass `-C <repo_root>` in multi-repo or submodule workspaces; run long tasks asynchronously and wait for completion rather than polling.
   - Capture **`session_id`** from the launch header on stdout/stderr (or `thread_id` from `thread.started` with `--json`); `-o <out.md>` writes only the final reply and omits `session_id`.
   - For scoped/risk-focused independent `reviewer` runs, prefer a fresh `codex exec -s read-only` reviewing `git diff` rather than `codex exec review` (`review` does not support `resume` and limits custom prompt scoping).
3. **Reuse the same session (`resume` flag ordering)**
   ```sh
   codex exec -C <repo_root> -s <read-only|workspace-write> -m gpt-6.2-astra -c 'model_reasoning_effort="high"' -o <out.md> resume <session_id> "<English follow-up>"
   ```
   > `-C` and `-s` are top-level `codex exec` options and **must appear before `resume`** when switching permissions or setting the root directory. Open a new session only for an unrelated task, polluted context, or an independent `reviewer`.
4. **Fan out in parallel**: dispatch independent `researcher` tasks (or `--worktree`-isolated `worker` tasks) concurrently and wait for all of them together.

---

## 6. Six-Part English Delegation Brief

Include these six items in every brief sent to Codex:
1. **Objective**: the single concrete outcome required.
2. **Scope**: owned files or modules; for read-only tasks, state `Do not edit files.`
3. **Context**: minimal verified facts needed (distilled from `researcher` and user-approved plan).
4. **Non-goals / Constraints**: edge cases intentionally left unhandled (e.g., fail fast, log and continue, rely on idempotent refresh) and boundaries not to cross—explicitly forbid adding unnegotiated restrictions, rigid input/state locks, compatibility shims, or unrelated refactors.
5. **Test & Deliverable budget**: specify exact acceptance scenarios to test—or explicitly state `Do not add new tests; run existing checks only`—to prevent speculative branch tests; for read-only roles, specify the exact facts (`path:line`, call paths) to return.
6. **Acceptance**: minimal sufficient verification command(s) that must pass after the final edit.

---

## 7. Tiered Verification (Lightweight Default, Deep Audit on Demand)

1. **Routine default (`git diff --stat` + test pass—do NOT deep-review every turn)**:
   - Check `git diff --stat` and the reported verification command output. If modified files stay within `Scope`, line delta looks proportional, and required checks ran and passed **after the final edit**, accept and summarize cleanly without reading diff hunks or re-running passing tests.
2. **Deep diff audit (triggered manually by user request, or on clear `--stat` warning signals)**:
   - Only when **the user explicitly asks to review/audit Codex's changes**—or when `git diff --stat` shows unexpected files, disproportionate production/test bloat, or missing post-edit verification—inspect actual `git diff` hunks specifically for **unnegotiated restrictions, rigid validations, fragile assumptions, and defensive machinery**, or invoke an independent `reviewer` and manually triage its findings before pruning via `resume`.
