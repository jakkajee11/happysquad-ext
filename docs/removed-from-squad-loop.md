# Removed from squad-loop

Sections removed from `happysquad`'s `skills/squad-loop/SKILL.md` per [`docs/spec/v1-rewrite.md`](../../happysquad/docs/spec/v1-rewrite.md) §2.1 / §19 P3, preserved verbatim for reference. Removed in happysquad commit `f10fc31`.

---

<!-- squad-loop/SKILL.md § "### 8. Worktree fallback" — removed in f10fc31 -->

### 8. Worktree fallback

If at ARCHITECT, the architecter outputs a design with overlapping ownership (or a Conflict surface section with non-empty rows) AND the user/orchestrator decides parallelism is still desired:

1. Create a git worktree per workstream at `.happysquad/worktrees/<run-id>/<workstream>/`.
2. Each parallel implementer + tester runs inside its worktree (pass the worktree path as the working directory to the Agent tool).
3. After PARALLEL_TEST, the orchestrator **merges** worktrees back into the main repo:
   a. For each workstream's worktree, run `git diff --no-color` to capture changes.
   b. Apply all diffs to the main repo in dependency order.
   c. If any patch fails to apply (true conflict), abort the merge and route to ARCHITECT — the partition is genuinely overlapping and must be redesigned.
4. CONFLICT_GATE runs on the merged main repo. The integration build catches issues the worktrees couldn't see in isolation.

Worktree mode is opt-in via `.happysquad/config.json`:

```json
{ "parallel_isolation": "ownership-then-worktree" }
```

Default is `ownership-then-worktree` (try ownership first, fall back to worktree on overlap). Other valid values:
- `ownership-only` — never use worktrees; conflict surface always routes back to architecter.
- `always-worktree` — every parallel run uses worktrees, even when ownership is disjoint. Higher setup cost, stronger isolation.

---

<!-- squad-loop/SKILL.md § "### 10a. Model escalation before BLOCKED (fable-delegate, opt-in)" — removed in f10fc31 -->

### 10a. Model escalation before BLOCKED (fable-delegate, opt-in)

A BLOCKED run is usually correct — convergence or the cap means cheaper routing is exhausted. But some blockers are exactly where a stronger model tier changes the outcome: a recurring race/heisenbug, a multi-constraint design knot the architecter keeps re-partitioning, or security/migration/data-loss work already in prod. Give the run one Fable 5 attempt before dying — but only when opted in, because Fable is expensive.

**Gate — all must hold:**
- `fable_escalation.enabled` is true in the effective config (`.happysquad/config.json` or an approved team-plan's `config_overrides`), AND
- `state.json.fable_attempted` is not true — one attempt per run, never a fable loop, AND
- the blocker reads as model-tier-hard (at least one fable-delegate signal: two failed fix attempts on the same defect, concurrency/race/CI-only, multi-constraint design, security/data-loss/migration, or subtle invariant work). A blocker that is a missing *decision* (scope, requirement, ambiguity only the human can resolve) does NOT qualify — no model tier fixes that; go straight to BLOCKED.

**If the gate holds — one attempt:**
1. Set `state.json.fable_attempted = true` before dispatch (idempotent against double-dispatch). Tell the user in one line — a notice, not a question: "Escalating the blocker to Fable: <why>."
2. Dispatch via the Agent tool — `subagent_type: general-purpose`, `model: fable`, an addressable `name` — with a handoff brief built from the BLOCKED.md draft + feedback.md: mission (the recurring blocker, solve mode), observable success criteria (the failing `Verify` commands that must pass), verified facts (file:line + exact error output from the failed rounds), already-tried (every prior fix attempt and what happened), constraints (build/test commands from CLAUDE.md), scope boundary (the workstream's `owned_files`), verification commands, report contract. Fable starts with zero context — the brief is the whole skill.
3. When Fable returns, the orchestrator owns "done": run the evidence gate (§3 — re-run build + full suite) and diff-review its changes like a hostile reviewer. Never relay "Fable says fixed" unverified.
4. Then a delta review (chief reviewer, `mode=delta`). PASS → COMPLETE. Still failing → fall through to the real BLOCKED (§10, step 1).

If Fable returns stuck too, don't third-attempt — proceed to BLOCKED and record in BLOCKED.md that Fable was tried and what it concluded. Two tiers failing the same way usually means the problem is mis-specified; that is the raw material for a `lessons/*` entry and a user decision. This never applies to a missing-decision blocker, and never more than once — escalation is one borrowed attempt, not a second loop.

---

<!-- squad-loop/SKILL.md § "### 11. External executors (team-plan roles)" (incl. 11a/11b/11c) — removed in f10fc31 -->

### 11. External executors (team-plan roles)

Only active when `state.json.team_plan.external_executors.enabled` is true. Absent or disabled team-plan → this entire section never runs and the loop behaves exactly as before.

**Visible tmux pane rule (applies to 11a/11b/11c).** Every external (non-Claude) invocation runs in its own tmux pane so the user sees who is doing what.

Detection (VERIFIED — `$TMUX` is **not** inherited by orchestrator shells, so detect via `tmux list-sessions`, not the env var):

```bash
SESS=$(tmux list-sessions -F '#{session_name} #{session_attached}' 2>/dev/null | awk '$2>=1 {print $1; exit}')
```

When an attached session exists (`$SESS` non-empty):

```bash
PANE=$(tmux split-window -d -P -F '#{pane_id}' -t "$SESS:" '<glm command> 2>&1 | tee <log>; echo EXIT=$? >> <log>.exit')
tmux select-pane -t "$PANE" -T "glm:<role>#<n>"
```

Pane title format is `glm:<role>#<n>` (executor:role, numbered). The orchestrator **polls the `.exit` file** (same 10-minute bound as a blocking call) instead of blocking on the pane; the pane closes itself when the command finishes, and the log persists at `runs/<run-id>/external/`.

When no attached tmux session exists: fall back to the direct Bash call (same log path) and note the fallback in history (e.g. `"pane": "fallback-direct"`). Never run external work silently in a hidden background shell — every invocation is either a visible pane or a logged direct call, never neither.

**11a Mechanical offload** — the orchestrator (never the implementer subagent) extracts `[mechanical]`-tagged tasks from design.md. Per task, via Bash under the tmux pane rule above, timeout 10 min:

```bash
glm -p "<file-scoped prompt>" --permission-mode acceptEdits --allowedTools "Read,Edit,Write,Glob,Grep"
```

Log to `runs/<run-id>/external/mechanical-<n>.log`. (One glm round-trip is ≈30s minimum — budget accordingly.) Failure or timeout → the item returns to the implementer for one attempt, no retry loop. The tester and the evidence gate verify glm's output exactly like any other output — no special trust. History entry: `{"state":"EXTERNAL_MECHANICAL","executor":"glm","items":N,"result":"ok|partial"}`.

(Contingency only: if headless writes are ever denied for a target repo, glm can instead emit a unified diff for the orchestrator to `git apply` — the direct-edit form above is the verified, primary path.)

**11b Cross-review** — via Bash under the tmux pane rule above:

```bash
glm -p "<review prompt: diff vs base_ref, findings as file:line + severity, no verdict>" --permission-mode plan --allowedTools "Read,Grep,Glob,Bash(git diff*),Bash(git log*)"
```

Writes `runs/<run-id>/reviews/external.md` headed `ADVISORY — external model`. Failure → write a stub noting the failure and continue; never blocking, and never a substitute for the chief reviewer's own read of the diff.

**11c Quota fallback** — after `retry_threshold` rate-limit failures in an `allowed_phases` phase (see "Agent error / death" above): run that phase's work through `glm -p`, using the same agent `.md` file as the prompt, the same file inputs, the same marker contract, and the same evidence gate as the Claude agent it's replacing — invoked under the tmux pane rule above. History: `"result":"external-fallback"`; tell the user which phase ran externally and why. Never applies to REVIEW — the chief's verdict is never delegated.
