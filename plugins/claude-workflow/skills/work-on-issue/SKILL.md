---
name: work-on-issue
description: Start work using a 4-stage pipeline — Opus (high) plans, Sonnet (high) implements, then two parallel Sonnet reviews (acceptance-compliance + code review), with a feedback loop. Works in three modes — (a) bez argumentu → wybór issue z listy otwartych, (b) jeden lub wiele numerów issue (np. "12" albo "12 13 14"), (c) wolny opis zadania bez issue. Use when the user asks to "rozpocznij pracę nad issue N" / "work on issue N" / "rozpocznij pracę" / "popraw X".
argument-hint: [issue-number(s) | free-text description | empty]
disable-model-invocation: true
allowed-tools: Bash(gh issue view *) Bash(gh issue list *) Bash(gh pr *) Bash(git status *) Bash(git diff *) Bash(git log *) Bash(git checkout *) Bash(git switch *) Bash(git branch *) Bash(uv run *) Bash(docker compose *) Read Grep Glob
---

# Work on Issue / Task — multi-agent pipeline

Orchestrate work for the `llm-chat-python-tutorial-docker` project (multi-tenant FastAPI + PostgreSQL/pgvector + scheduler; Nuxt 4 admin panel w `admin-ui/`).

You are the **orchestrator**. You do not write the implementation yourself — you delegate to subagents and pass artifacts between them. Keep your own context lean: tool output from sub-phases is heavy, so use `Agent` calls (which keep raw output out of your context) rather than Reading every file the implementer touches.

## Input parsing — three modes

Raw arguments: `$ARGUMENTS`

Inspect the argument string and pick exactly one mode:

1. **Empty mode** — `$ARGUMENTS` is empty / whitespace only.
   → Run `gh issue list --state open --limit 30 --json number,title,labels,milestone` (preloaded below), present a short numbered list to the user, and ask which issue (or issues) to work on. Do **not** start the pipeline before the user answers.

2. **Issue-number mode** — every whitespace/comma-separated token is purely numeric (e.g. `12`, `12 13`, `12,13,14`, `#12 #13`). Strip leading `#`.
   → Fetch each issue with `gh issue view <N>`. If any fetch fails, stop and report — do not fabricate. With multiple issues: **one branch, one PR** referencing all numbers (e.g. `feature/issues-12-13-<slug>`), unless the issues are clearly incompatible (different stacks / conflicting acceptance criteria) — in that case ask the user whether to split into separate runs.

3. **Free-text mode** — argument is non-empty and contains non-numeric content (e.g. `popraw walidację slug w tenant API`).
   → Treat the text verbatim as the task spec. No `gh issue view`. Branch name: `feature/<slug-from-description>` (lowercase ascii, kebab-case, max ~40 chars). Surface this in your first status line so the user can correct the slug before the planner starts.

If the mode is ambiguous (e.g. `12 popraw bug`), ask the user one short clarifying question instead of guessing.

## Preloaded context

Raw arguments string (orchestrator parses this itself — see „Input parsing"):

```
$ARGUMENTS
```

Always-useful working-tree state (preloaded commands tolerate failure so the skill still loads outside a git repo / without `gh`; treat „not a git repository" or `gh` errors as a signal, not a crash):

```!
git status --short 2>&1 || true
```

```!
git rev-parse --abbrev-ref HEAD 2>&1 || true
```

Open issues (cheap, helpful regardless of mode — w empty mode to twoja lista do zaprezentowania użytkownikowi; w pozostałych trybach to kontekst pokrewnych ticketów):

```!
gh issue list --state open --limit 30 --json number,title,labels,milestone 2>&1 || true
```

### Preflight — abort early if the environment is wrong

Before doing anything else, inspect the three blocks above:

- If `git rev-parse` output contains `not a git repository` (or similar), **stop**. Tell the user: „Skill `work-on-issue` musi być uruchomiony w repozytorium projektu `llm-chat-python-tutorial-docker`. Aktualny katalog to nie jest repo git — przełącz się do właściwego katalogu projektu i odpal skilla ponownie." Do **not** continue to phases 1–3.
- If `gh issue list` failed (auth error, no GitHub remote, network), report what failed and ask the user whether to: (a) fix the issue and retry, or (b) continue in **free-text mode** (skips `gh`). In empty mode and issue-number mode you **cannot** continue without `gh`.

## Fetching issue bodies

If you detected **issue-number mode**, fetch each issue body yourself with one `gh issue view <N> --json number,title,body,labels,state,assignees,milestone` call per number (run them in parallel — they are independent). If any call fails, stop and report — do not fabricate.

In **empty mode** and **free-text mode** skip this step entirely.

## Stack detection

Decide which environment(s) the task touches before picking subagents:

- **Python / FastAPI backend** → `src/chat_ai/**`, `tests/**`, `migrations/**`, `pyproject.toml` → use `voltagent-lang:fastapi-developer` (or `voltagent-lang:python-pro` for non-API code, `voltagent-lang:sql-pro` for migrations).
- **Docker / infra** → `compose.yml`, `compose.dev.yml`, `docker/**`, `.env*`, `alembic.ini`, scheduler container → use `devops-engineer`.
- **Nuxt admin panel** → `admin-ui/**` (Nuxt 4 + @nuxt/ui v4 + Composition API) → use `voltagent-lang:vue-expert`. Use **npm**, not pnpm/yarn.
- **Multi-area** → pick the dominant area for the implementer; mention secondary concerns in the brief.

If the task is ambiguous about scope (e.g. mówi „dodaj endpoint" ale nie wskazuje warstwy), **ask the user one clarifying question** before phase 1 instead of guessing.

## Pipeline

Run phases sequentially. Phases 1 and 2 are each one `Agent` tool call. Phase 3 consists of **two** review agents (3a compliance + 3b code review) that you spawn **in parallel — both `Agent` calls in a single message**, then wait for both completions before evaluating the joint verdict. Do **not** Read the agent's transcript file mid-flight; wait for the completion notification.

### Phase 1 — PLAN (Opus, high effort)

Spawn a planner. Use `subagent_type: "Plan"` — it inherits Opus and is read-only, perfect for design without touching code.

Brief the planner with:
- **Task source** — depending on mode:
  - Issue-number mode (single): full issue body (title + description + labels + acceptance criteria if present).
  - Issue-number mode (multi): all issue bodies, plus an explicit note that they are being delivered together. Ask the planner to flag if it considers them too unrelated to bundle.
  - Free-text mode: the user's description verbatim, plus a note that there is no GitHub issue — the planner should propose acceptance criteria itself and surface them for user confirmation in the plan.
- Current branch and branching instruction. Default: branch off `multi-tenant` (developer branch — see `CLAUDE.md`), name `feature/issue-<N>-<slug>` for single issue, `feature/issues-<N1>-<N2>-<slug>` for multi, `feature/<slug>` for free-text. If already on a relevant feature branch, stay on it.
- Project conventions from `CLAUDE.md` (multi-tenant, RLS, slug regex, no DELETE tenant, `with_tenant_context`, ruff + pytest, alembic, `uv` for tooling, npm dla admin-ui).
- Relevant docs: `docs/ADMIN.md`, `docs/TENANT.md`, `docs/SCHEDULER.md`, `docs/ADMIN_UI.md`, `tutorial-python-high-level.md`.
- Demand a deliverable plan with: file-by-file changes, migration needs, test strategy (unit + tenant-isolation integration if applicable), rollout/compose impact, risks. Plan must include explicit **acceptance checks** the reviewer will verify (in free-text mode the planner proposes them; in issue mode they come from the issue).
- Tell it to think carefully (high effort) and to NOT write any code.

In **free-text mode**, after the plan returns, surface the proposed acceptance criteria to the user and get a quick confirmation before moving to Phase 2 — this prevents the implementer from delivering the wrong thing when there is no issue body to anchor on.

Capture the returned plan verbatim — it is the contract for phases 2 and 3.

### Phase 2 — IMPLEMENT (Sonnet, high effort)

Spawn the implementer. Pick `subagent_type` from stack detection above. Pass `model: "sonnet"`.

Brief the implementer with:
- The full plan from Phase 1 (verbatim, as the contract).
- The issue number(s) `#<N>` (or "no issue — ad-hoc task: <short description>" in free-text mode) and branch instruction.
- Hard rules: follow `CLAUDE.md`; respect RLS via `with_tenant_context`; new migrations via Alembic; tests must pass `uv run pytest`; lint must pass `uv run ruff check`; admin-ui used npm + `npm run lint` + `npm run test`; do **not** introduce new dependencies without flagging them.
- Tell it to think carefully (Sonnet high) and to report a concise summary of: files changed, commands run, test/lint results, anything skipped vs the plan and why.
- Tell it **not to commit** unless the user has previously authorized auto-commit — final commit + PR is a user-confirmed step at the end.

After it returns, spot-check the diff with `git diff --stat` and `git status` (these are cheap and worth doing) before moving on.

### Phase 3 — REVIEW (dwa Sonnet review, równolegle)

Spawn **dwa `Agent` calls w jednym message** (parallel). Czekaj na obie odpowiedzi przed podjęciem decyzji.

#### Phase 3a — Compliance review (Sonnet, high)

- `subagent_type: "general-purpose"`, `model: "sonnet"`.
- Brief:
  - Plan z fazy 1 (kontrakt).
  - Acceptance criteria — z issue (issue-number mode) lub potwierdzone przez użytkownika (free-text mode); w trybie wieloma issue dołącz acceptance criteria z każdego z nich.
  - Podsumowanie implementera z fazy 2 (files changed, commands run, test/lint results, anything skipped vs plan).
  - Diff (`git diff multi-tenant...HEAD` lub względem parent brancha).
- Zadanie: zweryfikować, czy implementacja realizuje plan i acceptance criteria **1:1** (pole-po-polu). **Nie** oceniać jakości kodu — wyłącznie zgodność z założeniami. Każda pozycja acceptance criteria musi być oznaczona jako spełniona lub brakująca (z `file:line` wskazującym, gdzie powinno być, lub gdzie obecna implementacja rozjeżdża się z założeniem).
- Demand a structured verdict: **COMPLIANT** albo **GAPS** z listą braków (każdy item: który wymóg + co jest nie tak + sugerowane miejsce poprawki).

#### Phase 3b — Code review (Sonnet, high)

- `subagent_type: "code-reviewer"`, `model: "sonnet"`.
- Brief:
  - Plan z fazy 1 — jako **kontekst** (od zgodności z planem jest 3a; tu nie oceniaj zgodności, oceniaj jakość kodu w świetle planu).
  - Podsumowanie implementera z fazy 2.
  - Diff (`git diff multi-tenant...HEAD` lub względem parent brancha).
  - Project-specific review checklist: tenant isolation correctness (RLS context, slug validation, no cross-tenant data leak), migration safety (reversible? RLS policies updated?), error handling at boundaries only, ruff + pytest expectations, secrets/env handling, Docker entrypoint impact, observability (per-tenant metrics still labeled correctly), admin-ui lint/test if touched.
- Demand a structured verdict: **APPROVE** lub **CHANGES_REQUESTED** z itemized listą blokerów (file:line + konkretna poprawka), plus opcjonalne non-blocking suggestions.

### Decision

Joint verdict z fazy 3a + 3b:

- **APPROVE** (zielone światło) → wymaga **3a = COMPLIANT i 3b = APPROVE**. Zgłoś użytkownikowi: branch name, files changed, test/lint status, non-blocking suggestions z 3b. Zapytaj, czy commitować i otwierać PR (against `multi-tenant`, per `CLAUDE.md`). Nie pushuj i nie twórz PR bez wyraźnego potwierdzenia.
- **CHANGES_REQUESTED** → gdy **3a = GAPS lub 3b = CHANGES_REQUESTED**. Loop back to Phase 2 z połączoną listą blokerów: najpierw braki zgodności z 3a, potem code-review blokery z 3b. Pomiń non-blocking suggestions w briefingu pętli. Cap at **2 review iterations**. Jeśli trzeci cykl nadal nie jest APPROVE+COMPLIANT, zatrzymaj się i eskaluj do użytkownika z podsumowaniem nierozwiązanych pozycji z obu review — nie kontynuuj pętli.

## Reporting back to the user

State the chosen mode in your first status line (e.g. „Tryb: opis zadania — branch `feature/<slug>`", „Tryb: 3 issue (#12, #13, #14) — bundle do jednego PR"). Between phases, send one short status line („Plan gotowy, startuję implementację", „Implementacja gotowa, startuję dwa równoległe review"). At the end, report:
- **Compliance (3a):** COMPLIANT / GAPS.
- **Code review (3b):** APPROVE / CHANGES_REQUESTED.
- Overall verdict (approved / escalated).
- Branch + diff stat.
- Test/lint results.
- Issue references (or "no issue — ad-hoc" in free-text mode).
- Open follow-ups (non-blocking reviewer notes z 3b, deferred work).
- Suggested next action (commit + PR against `multi-tenant`, or address X then re-run).

## Guardrails

- Never run destructive git ops (`reset --hard`, `push --force`, `branch -D`) without user confirmation.
- Never auto-merge or auto-push.
- In issue-number mode, if any `gh issue view <N>` failed, stop and tell the user — do not fabricate issue content.
- In empty mode, never start the pipeline before the user picks issue(s) or supplies a description.
- If the working tree was dirty at start, ask the user how to proceed (stash, commit, or abort) before creating a new branch.
- PR base branch is `multi-tenant`, not `main` (developer branch — see `CLAUDE.md`).
