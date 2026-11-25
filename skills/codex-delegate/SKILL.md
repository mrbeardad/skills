---
name: codex-delegate
description: "Guide a Tech Lead agent in orchestrating the local `codex` CLI as persistent subagents (`researcher`, `worker`, `reviewer` defaulting to `gpt-6.2-astra` at `high` reasoning): separate root-only work from delegation, enforce English briefs and `session_id` reuse, trust read-only findings directly, and gate delivery on `git diff --stat`."
---

# Codex Delegate (Local Codex CLI as Tech Lead Subagent)

This skill defines how any orchestrating **Tech Lead** coding agent delegates heavy work to the local `codex` CLI as an isolated subagent.
The Tech Lead retains ownership of **goal understanding, architectural boundaries, parallelism, test budgets, and human-friendly reporting to the user**; Codex handles heavy repository exploration, multi-file implementation and verification runs, and optional independent review.

---

## 1. Codex Traits to Account For

- **Read-only research is usually reliable, though verbose and defensive**: `researcher` locates files and traces call flows accurately. Trust its grounded codebase facts directly instead of re-reading the repo to verify them (spot-check key locations only when making a major architectural bet with lingering doubt), and distill its noisy caveats into concise prose for the user.
- **Implementation and review skew over-defensive and test-heavy**: Left without tight boundaries or a test budget, Codex tends to invent redundant state machines, speculative failure guards, and branch-by-branch unit tests. Always lock down `Non-goals` and `Test budget` in the brief, screen `git diff --stat` for bloat, and triage `reviewer` findings manually.

---

## 2. Delegation Gate: Direct vs. Delegated

- **Handle directly in the Tech Lead thread (do not spawn Codex)**: pure conceptual/design questions, localized questions answered by reading 1–2 files, or small, clear edits (1–2 files, minimal lines).
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
| **`researcher`** | `read-only` | Codebase discovery, call-chain tracing, and read-only fault reproduction. **Trust grounded findings directly**; synthesize into a concise plan and align boundaries with the user. |
| **`worker`** | `workspace-write` | Implements the scoped change, adds only the exact Tech-Lead-specified tests, and iterates until checks pass. Prefer `resume` on the same task's `researcher` session; pass `--worktree` when running concurrent write workers. |
| **`reviewer`** | `read-only` | **Only for high-risk or complex changes.** Must run in a **fresh `read-only` session** (independent from `worker` bias). Tech Lead and user triage findings; **never blindly feed all review comments back into `worker` in a loop**. |

---

## 4. CLI Usage, Session Reuse & Parallelism

1. **Language isolation**
   - **To the user**: communicate in the user's current language, delivering only distilled conclusions, trade-offs, and verified checks.
   - **To Codex**: **always write every brief, follow-up, `resume` prompt, and review instruction in English**, regardless of the user's language.
2. **Initial launch & async wait**
   ```sh
   codex exec -C <repo_root> -s <read-only|workspace-write> -m gpt-6.2-astra -c 'model_reasoning_effort="high"' -o <out.md> "<English brief>"
   ```
   - Always pass `-C <repo_root>` in multi-repo or submodule workspaces; run long tasks asynchronously and wait for completion rather than polling.
   - Capture **`session_id`** from the launch header on stdout/stderr (or `thread_id` from `thread.started` with `--json`); `-o <out.md>` writes only the final reply and omits `session_id`.
   - For scoped/risk-focused independent `reviewer` runs, prefer a fresh `codex exec -s read-only` reviewing `git diff` rather than `codex exec review` (`review` does not support `resume` and limits custom prompt scoping).
3. **Reuse the same session (`resume` flag ordering)**
   Continue the same task (`researcher` → `worker`, follow-up edits, diff pruning) via `resume`:
   ```sh
   codex exec -C <repo_root> -s <read-only|workspace-write> -m gpt-6.2-astra -c 'model_reasoning_effort="high"' -o <out.md> resume <session_id> "<English follow-up>"
   ```
   > `-C` and `-s` are top-level `codex exec` options and **must appear before `resume`** when switching permissions or setting the root directory. Open a new session only for an unrelated task, polluted context, or an independent `reviewer`.
4. **Fan out in parallel**: dispatch independent `researcher` tasks (or `--worktree`-isolated `worker` tasks) concurrently and wait for all of them together instead of serializing independent runs.

---

## 5. Six-Part English Delegation Brief

Include these six items in every brief sent to Codex:
1. **Objective**: the single concrete outcome required.
2. **Scope**: owned files or modules; for read-only tasks, state `Do not edit files.`
3. **Context**: minimal verified facts needed (distilled from `researcher` or prior turns).
4. **Non-goals / Constraints**: edge cases intentionally left unhandled (e.g., fail fast, log and continue, rely on idempotent refresh) and boundaries not to cross (public contracts, schemas, unrelated refactors).
5. **Test & Deliverable budget**: specify exact acceptance scenarios to test—or explicitly state `Do not add new tests; run existing checks only`—to prevent speculative branch tests; for read-only roles, specify the exact facts (`path:line`, call paths) to return.
6. **Acceptance**: minimal sufficient verification command(s) that must pass after the final edit.

---

## 6. Tiered Verification & Delivery

1. **`researcher` output**: accept its `path:line` and call-chain findings directly; translate them into a clear plan and non-goal boundaries for user approval before launching `worker`.
2. **`worker` output (lightweight gate—do not line-review every routine turn)**:
   - Inspect `git diff --stat` and the reported verification results first. If modified files stay within scope, diff size looks proportional, and required checks ran and passed **after the final edit**, accept and summarize cleanly without re-reading source files or re-running passing tests.
   - Open detailed diff hunks **only on warning signals** (unexpected files touched, disproportionate line growth in production or test files, missing post-edit verification, or high-risk state-machine/contract changes); prune bloat via `resume <session_id>`.
3. **`reviewer` output**: use selectively for high-risk changes; filter out low-value or speculative hardening suggestions before deciding what, if anything, goes back to `worker`.
