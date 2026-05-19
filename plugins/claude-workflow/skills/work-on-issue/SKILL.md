---
name: work-on-issue
description: Start work using a 4-stage pipeline — Opus (high) plans, Sonnet (high) implements, then two parallel Sonnet reviews (acceptance-compliance + code review), with a feedback loop. Works in three input modes — (a) bez argumentu → wybór issue z listy otwartych, (b) jeden lub wiele numerów issue (np. "12" albo "12 13 14"), (c) wolny opis zadania bez issue. Adapts to environment — runs full git/PR workflow when in a git repo, falls back to a prompt-only pipeline (plan → implement → review, no branch, no PR) when not. Use when the user asks to "rozpocznij pracę nad issue N" / "work on issue N" / "rozpocznij pracę" / "popraw X".
argument-hint: [issue-number(s) | free-text description | empty]
disable-model-invocation: true
allowed-tools: Bash(gh issue view *) Bash(gh issue list *) Bash(gh pr *) Bash(git status *) Bash(git diff *) Bash(git log *) Bash(git checkout *) Bash(git switch *) Bash(git branch *) Bash(git rev-parse *) Bash(git remote *) Bash(ls *) Bash(test *) Bash(uv run *) Bash(docker compose *) Bash(npm *) Read Grep Glob
---

# Work on Issue / Task — multi-agent pipeline

Generic orchestrator for issue- or task-driven work. The pipeline is the same regardless of environment:

> **Opus plans → Sonnet implements → two parallel Sonnet reviews (compliance + code review) → loop back if needed.**

What changes is how much git/PR plumbing wraps it:

- **Repo mode** (current directory is a git repo) — create a feature branch, end with a commit + PR offer.
- **No-repo mode** (not a git repo) — skip branching and PR. The implementer still produces concrete file changes/output; the user takes it from there.

If the cwd looks like the `llm-chat-python-tutorial-docker` project (multi-tenant FastAPI + PostgreSQL/pgvector + scheduler; Nuxt 4 admin panel w `admin-ui/`), apply the project-specific conventions from `CLAUDE.md` automatically. Otherwise stay generic and let the planner derive conventions from whatever signals are in the cwd (or from the prompt alone in no-repo mode).

You are the **orchestrator**. You do not write the implementation yourself — you delegate to subagents and pass artifacts between them. Keep your own context lean: tool output from sub-phases is heavy, so use `Agent` calls (which keep raw output out of your context) rather than Reading every file the implementer touches.

## Input parsing — three modes

Raw arguments: `$ARGUMENTS`

Inspect the argument string and pick exactly one mode:

1. **Empty mode** — `$ARGUMENTS` is empty / whitespace only.
   → Requires `gh issue list` to work. If it did (see preloaded block), present a short numbered list to the user and ask which issue(s) to work on. Do **not** start the pipeline before the user answers. If `gh issue list` failed, tell the user empty mode is unavailable here and ask them to pass an issue number or a free-text description instead.

2. **Issue-number mode** — every whitespace/comma-separated token is purely numeric (e.g. `12`, `12 13`, `12,13,14`, `#12 #13`). Strip leading `#`.
   → Fetch each issue with `gh issue view <N>`. If `gh` is not authenticated, has no default repo, or any fetch fails, stop and report — do not fabricate. With multiple issues in **repo mode**: one branch, one PR referencing all numbers (e.g. `feature/issues-12-13-<slug>`), unless the issues are clearly incompatible — then ask the user whether to split. In **no-repo mode** there is no branch/PR; treat the issues as a bundled task spec.

3. **Free-text mode** — argument is non-empty and contains non-numeric content (e.g. `dodaj package.json` / `popraw walidację slug w tenant API`).
   → Treat the text verbatim as the task spec. No `gh issue view`. In repo mode, derive a branch name `feature/<slug-from-description>` (lowercase ascii, kebab-case, max ~40 chars) and surface it in the first status line so the user can correct it. In no-repo mode, no branch — just announce the task and proceed.

If the mode is ambiguous (e.g. `12 popraw bug`), ask the user one short clarifying question instead of guessing.

## Preloaded context

Raw arguments string (orchestrator parses this itself — see „Input parsing"):

```
$ARGUMENTS
```

Working-tree state (preloaded; tolerates failure so the skill loads regardless of environment):

```!
git rev-parse --abbrev-ref HEAD 2>&1 || true
```

```!
git status --short 2>&1 || true
```

```!
git remote -v 2>&1 || true
```

Open issues (works only when `gh` is authenticated and a default repo is resolvable):

```!
gh issue list --state open --limit 30 --json number,title,labels,milestone 2>&1 || true
```

Project signals (used to decide whether to apply `llm-chat-python-tutorial-docker` conventions):

```!
ls -1 CLAUDE.md compose.yml compose.dev.yml pyproject.toml admin-ui 2>&1 || true
```

## Environment detection

Before the pipeline starts, classify the environment from the preloaded blocks above. Surface the result in your first status line.

- **`git_repo`** — `true` if `git rev-parse --abbrev-ref HEAD` returned a branch name without a `fatal:` error. Otherwise `false` → **no-repo mode**.
- **`gh_ok`** — `true` if `gh issue list` returned JSON (`[` at the start). Otherwise `false`. When `false`, empty mode and issue-number mode are blocked unless you can recover (ask user to authenticate or provide the issue manually).
- **`project_detected`** — `true` if `CLAUDE.md` exists AND (`compose.yml` or `pyproject.toml` or `admin-ui` also exists). When `true`, apply project conventions (RLS, `with_tenant_context`, `uv`, `npm`, branch off `multi-tenant`, etc.). When `false`, stay generic and infer from cwd signals.

State the resulting mode tag in your first status line, e.g.:
- „Tryb: free-text + repo + project — branch `feature/<slug>` off `multi-tenant`."
- „Tryb: issue #12 + no-repo + generic — produkuję plan i artefakty bez brancha."
- „Tryb: free-text + no-repo — pipeline na podstawie samego promptu."

## Fetching issue bodies

If you detected **issue-number mode** and `gh_ok = true`, fetch each issue body with one `gh issue view <N> --json number,title,body,labels,state,assignees,milestone` call per number (parallel — independent). If any call fails, stop and report — do not fabricate.

If `gh_ok = false` in issue-number mode: tell the user `gh` is not available here and either (a) ask them to paste the issue body inline (treat as free-text mode after), or (b) abort.

In **empty mode** and **free-text mode** skip this step entirely.

## Stack detection

Pick the implementer subagent from cwd signals (skip in no-repo mode unless cwd has any project files):

- **Python / FastAPI backend** → `src/chat_ai/**`, `tests/**`, `migrations/**`, `pyproject.toml` → `voltagent-lang:fastapi-developer` (or `voltagent-lang:python-pro` for non-API code, `voltagent-lang:sql-pro` for migrations).
- **Docker / infra** → `compose.yml`, `compose.dev.yml`, `docker/**`, `.env*`, `alembic.ini`, scheduler container → `devops-engineer`.
- **Nuxt admin panel** → `admin-ui/**` (Nuxt 4 + @nuxt/ui v4 + Composition API) → `voltagent-lang:vue-expert`. Use **npm**, not pnpm/yarn.
- **Generic / unknown** (no signals match, no-repo mode, or task is language-agnostic like "create a config file") → `general-purpose` with `model: "sonnet"`.
- **Multi-area** → pick the dominant area; mention secondary concerns in the brief.

If scope is ambiguous, ask one clarifying question before phase 1.

## Pipeline

Run phases sequentially. Phases 1 and 2 are each one `Agent` tool call. Phase 3 spawns **two** review agents (3a compliance + 3b code review) **in parallel — both `Agent` calls in a single message** — then wait for both completions before evaluating the joint verdict. Do **not** Read the agent's transcript file mid-flight; wait for the completion notification.

### Phase 1 — PLAN (Opus, high effort)

Spawn `subagent_type: "Plan"` (inherits Opus, read-only).

Brief the planner with:
- **Task source** — depending on mode:
  - Issue-number (single): full issue body (title + description + labels + acceptance criteria if present).
  - Issue-number (multi): all issue bodies + note that they're bundled. Planner flags if too unrelated.
  - Free-text: the user's text verbatim + note there's no GitHub issue — planner proposes acceptance criteria for user confirmation.
- **Environment tags** from detection: `git_repo`, `gh_ok`, `project_detected`. In no-repo mode tell the planner explicitly: „nie jesteś w repo git — zaplanuj file changes/artefakty względem cwd, bez brancha, bez PR".
- **Branching instruction** (repo mode only):
  - `project_detected = true` → branch off `multi-tenant`, names `feature/issue-<N>-<slug>` / `feature/issues-<N1>-<N2>-<slug>` / `feature/<slug>` (single / multi / free-text). If already on a relevant feature branch, stay on it.
  - `project_detected = false` → branch off the current default branch (`git remote show origin` or just current `HEAD`), names same scheme.
- **Project context** (only when `project_detected = true`): conventions from `CLAUDE.md` (multi-tenant, RLS, slug regex, no DELETE tenant, `with_tenant_context`, ruff + pytest, alembic, `uv`, `npm` for admin-ui), plus `docs/ADMIN.md`, `docs/TENANT.md`, `docs/SCHEDULER.md`, `docs/ADMIN_UI.md`, `tutorial-python-high-level.md`.
- **Deliverable**: file-by-file changes, migration needs (if relevant), test strategy, rollout impact, risks, and explicit **acceptance checks** for the reviewer. In free-text mode planner proposes them; in issue mode they come from the issue.
- Tell it to think carefully (high effort) and **not** write any code.

In **free-text mode**, after the plan returns, surface the proposed acceptance criteria to the user and get a quick confirmation before moving to Phase 2.

Capture the returned plan verbatim — it is the contract for phases 2 and 3.

### Phase 2 — IMPLEMENT (Sonnet, high effort)

Spawn the implementer. Pick `subagent_type` from stack detection. Pass `model: "sonnet"`.

Brief the implementer with:
- The full plan from Phase 1 (verbatim, as the contract).
- Task ID / source: `#<N>` (issue mode), or `no issue — ad-hoc task: <short description>` (free-text mode).
- **Environment tags** (`git_repo`, `gh_ok`, `project_detected`) + branching instruction (repo mode only — no branching in no-repo mode).
- **Hard rules** — apply only when relevant:
  - Always: do **not** introduce new dependencies without flagging them; report changes concisely.
  - `project_detected = true`: follow `CLAUDE.md`; respect RLS via `with_tenant_context`; new migrations via Alembic; tests must pass `uv run pytest`; lint must pass `uv run ruff check`; admin-ui uses npm + `npm run lint` + `npm run test`.
  - `project_detected = false`: follow conventions visible in cwd (existing test runner, linter, package manager). If cwd is empty/unrelated (no-repo mode), produce the deliverable as the plan describes — files in cwd or output in the response.
- Report a concise summary: files changed, commands run, test/lint results, anything skipped vs the plan and why.
- Do **not** commit unless the user has previously authorized auto-commit — final commit + PR is a user-confirmed step.

After it returns:
- In repo mode: spot-check with `git diff --stat` and `git status`.
- In no-repo mode: spot-check by listing affected files (`ls -la` on the implementer's reported paths). Skip if implementation was conceptual/output-only.

### Phase 3 — REVIEW (two Sonnet reviews, parallel)

Spawn **two `Agent` calls in a single message** (parallel). Wait for both before deciding.

#### Phase 3a — Compliance review (Sonnet, high)

- `subagent_type: "general-purpose"`, `model: "sonnet"`.
- Brief:
  - Plan from Phase 1 (contract).
  - Acceptance criteria — from issue (issue-number mode) or user-confirmed (free-text mode); in multi-issue mode include each issue's criteria.
  - Implementer summary from Phase 2.
  - Evidence: in repo mode, `git diff <base>...HEAD`. In no-repo mode, the implementer's reported file paths + their contents (or output if no files).
- Task: verify the implementation realizes the plan and acceptance criteria **1:1** (item-by-item). **Do not** assess code quality — only conformance. Each acceptance item must be marked satisfied or missing (with `file:line` showing where it is or where it should have been).
- Demand a structured verdict: **COMPLIANT** or **GAPS** with a list (each item: which requirement + what's wrong + suggested fix location).

#### Phase 3b — Code review (Sonnet, high)

- `subagent_type: "code-reviewer"`, `model: "sonnet"`.
- Brief:
  - Plan from Phase 1 — as **context** (compliance is 3a's job; here, assess code quality in light of the plan).
  - Implementer summary from Phase 2.
  - Evidence: diff (repo mode) or file contents/output (no-repo mode).
  - **Review checklist** — apply only relevant items:
    - Always: error handling at boundaries, secrets/env handling, code clarity.
    - `project_detected = true`: tenant isolation correctness (RLS context, slug validation, no cross-tenant data leak), migration safety (reversible? RLS policies updated?), ruff + pytest expectations, Docker entrypoint impact, observability (per-tenant metrics labeled correctly), admin-ui lint/test if touched.
    - Otherwise: follow cwd conventions, language-specific best practices for the implementer's stack.
- Demand a structured verdict: **APPROVE** or **CHANGES_REQUESTED** with an itemized list of blockers (file:line + concrete fix), plus optional non-blocking suggestions.

### Decision

Joint verdict from 3a + 3b:

- **APPROVE** (green light) — requires **3a = COMPLIANT and 3b = APPROVE**. Report to user: files changed, test/lint status (if run), non-blocking suggestions from 3b. Then:
  - Repo mode → ask whether to commit and open a PR (base = `multi-tenant` if `project_detected`, else the default branch). Do **not** push or open a PR without explicit confirmation.
  - No-repo mode → just deliver the result. Optionally suggest the user `git init` + commit if they want versioning, but don't do it for them.
- **CHANGES_REQUESTED** — when **3a = GAPS or 3b = CHANGES_REQUESTED**. Loop back to Phase 2 with the merged blocker list: 3a compliance gaps first, then 3b code-review blockers. Skip non-blocking suggestions in the loop. Cap at **2 review iterations**. After the third cycle still not green, stop and escalate to the user with the unresolved items from both reviews — do not keep looping.

## Reporting back to the user

In your first status line state the chosen mode tag (see „Environment detection"). Between phases send one short status line („Plan gotowy, startuję implementację", „Implementacja gotowa, startuję dwa równoległe review"). At the end, report:
- **Compliance (3a):** COMPLIANT / GAPS.
- **Code review (3b):** APPROVE / CHANGES_REQUESTED.
- Overall verdict (approved / escalated).
- Repo mode only: branch + diff stat.
- Test/lint results (if applicable).
- Issue references (or „no issue — ad-hoc" in free-text mode).
- Open follow-ups (non-blocking reviewer notes from 3b, deferred work).
- Suggested next action:
  - Repo mode: commit + PR against the chosen base, or address X then re-run.
  - No-repo mode: copy the produced files / apply the diff in the target repo, or address X then re-run.

## Guardrails

- Repo mode only: never run destructive git ops (`reset --hard`, `push --force`, `branch -D`) without user confirmation; never auto-merge or auto-push; if working tree was dirty at start, ask the user how to proceed (stash, commit, abort) before creating a new branch.
- Issue-number mode: if `gh issue view <N>` fails, stop and tell the user — do not fabricate.
- Empty mode: never start the pipeline before the user picks issue(s) or supplies a description.
- No-repo mode: do **not** invoke any `git` command that mutates state. Read-only `git rev-parse` etc. is fine but generally unnecessary here. Skip the branch/PR steps entirely.
- Project conventions (RLS, `multi-tenant` base branch, etc.) apply **only** when `project_detected = true`. Don't force them onto unrelated tasks.
