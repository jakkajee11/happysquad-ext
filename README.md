# happysquad-ext

Pieces cut from `happysquad` core during the v1.0 rewrite (spec: `docs/spec/v1-rewrite.md` §2.1 in the `happysquad` repo). Nothing here is wired to the v1 `hs` engine; it is parked, not maintained.

| what | from | status |
|---|---|---|
| External executors (glm/opencode): tmux pane rule, mechanical offload, cross-review, quota fallback | `skills/squad-loop` §11 (text in `docs/removed-from-squad-loop.md`) | not ported |
| Model escalation before BLOCKED (Fable-specific) | `skills/squad-loop` §10a | superseded by `escalation.model` in core 1.1.0 |
| Worktree-per-workstream fallback | `skills/squad-loop` §8 | dropped by design (ownership overlap = design error) |
| `/squad-assemble` + `team-assembly` skill | commands, skills | not ported |
| `/ask-kilo` + `ask-kilo` skill | commands, skills | not ported |
| `/squad-implement`, `/squad-test` (manual mid-loop steps) | commands | dropped: they bypass the evidence gates |
| `/squad-drain` | commands | superseded by `hs fleet start --drain` in core 1.1.0 |
| `requirement-reviewer`, `standard-reviewer` | agents | dropped: never dispatched in the default review mode |
| `security-reviewer`, `performance-reviewer` | agents | superseded by `hs-specialist` + `references/axis-*.md` in core |
| `architecter`, `implementer`, `tester`, `reviewer` (0.16 bodies) | agents | superseded by `agents/hs-*.md` in core; `reviewer.md` is still used by core's `evals/bench/recall.sh` as the 0.16 recall baseline |

Install only if you want the 0.16 commands back for reference; they expect the 0.16 `/happysquad-loop` protocol, which core no longer ships.
