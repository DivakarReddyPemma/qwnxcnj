
# Reflection Brief — Harness Engineering Capstone

**Name:** Divakar Reddy Pemma
**Date:** 2026-09-26

**Environment**

- Model(s): claude-haiku-4-5-20251001
- OS / Python: Linux, Python 3.13.0 (pytest-9.1.1)
- Approx. API spend: $0.1016 for the System 1 run of 8 fixtures (per `evidence/system1_agentic_loop/summary.md`)

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** In `evidence/system1_agentic_loop/claim_02_stolen_bike.jsonl`, the `stop_reason` sequence across the run is `tool_use → tool_use → tool_use → end_turn` (turns 1–4). The loop's continuation logic decides to keep going as long as `stop_reason == "tool_use"` (calling `lookup_policy`, `record_claim_fact`, `classify_claim`, `assess_severity`, `route_to_adjuster` across the turns) and stops the moment the model returns `stop_reason == "end_turn"` on turn 4 with zero tool calls.

2. **Anti-pattern.** The suite `tests/test_antipatterns.py` (4 tests, all passing per `evidence/system1_agentic_loop/pytest_S1.log`, part of the 29-test pass) enforces that the loop has no hard-coded max-turn cutoff. This is visible in my own run data: `claim_01_kitchen_fire` terminates in 2 turns while `claim_02_stolen_bike` and `claim_05_auto_collision` both run 4 turns — a fixed iteration cap would either truncate the longer claims mid-investigation or waste turns padding the shorter ones.

3. **Tool design.** *(Needs source not in this evidence pack — run `grep -n "def \|description" .../claims_intake/tools.py` in your S1 solution folder and pick two tool signatures with overlapping input fields, e.g. `record_claim_fact` vs `classify_claim`, to answer this fully.)*

4. **Your numbers.** *(Needs a README with sample numbers, which isn't in this evidence pack. What I can cite: `claim_05_auto_collision` took 4 turns, 15,322 input / 1,038 output tokens, est. $0.0205, 10.6s elapsed — compare this against your project's README sample if one exists, or note variance across your 8 claims: turns range 2–4, cost ranges $0.0083–$0.0205.)*

### System 2 — Context strategy

5. **The reduction.** From `evidence/system2_context_strategy/budget.json`: baseline_tokens = 38,708, assembled_tokens = 16,820, reduction_pct = 56.55%. The `active` section dominates at 15,789 tokens (of 16,820 total assembled) — it's kept verbatim because it's the live, unresolved issue (the payment-method update failure) the agent must answer questions about with exact wording, e.g. the structured status token.

6. **Summarize vs preserve.** Per `per_section_tokens` in `budget.json`: `resolved_refund` (377 tokens) and `resolved_subscription` (468 tokens) are summarized — down from raw inputs of 12,334 and 11,475 tokens respectively per `compression_api` — because those issues are closed and only their outcome matters. The `active` segment (15,789 tokens) and `case_facts` (204 tokens) are preserved verbatim because the agent must cite exact values (order IDs, status tokens) from them.

7. **Facts block.** Comparing `evidence/system2_context_strategy/eval.jsonl` to `eval_control.jsonl`: question Q6 ("What is the structured status of the payment-method update issue?") passes in `eval.jsonl` with the exact answer `in_progress`, but fails in `eval_control.jsonl` — the model instead returns "Unknown... no structured status field is present." This proves the persistent case-facts block is load-bearing: removing it doesn't just degrade quality, it causes the model to lose access to a specific, verbatim-required fact.

### System 3 — Claude Code config

8. **Path-scoped rules.** From `evidence/system3_claude_config/evidence_rule_api.md`, the frontmatter is `paths: - "src/api/**/*"`, and `evidence_rule_react.md` has `paths: - "src/components/**/*"` and `"src/pages/**/*"`. This beats a directory-level `CLAUDE.md` because, per `CLAUDE.md`'s own scope table, a single rule file's glob can span multiple directories at once — cross-cutting conventions like "test files everywhere" work with one rule file instead of duplicating a `CLAUDE.md` into every subdirectory.

9. **Forked skill.** From `evidence/system3_claude_config/evidence_rule_skill.md`: `context: fork` and `allowed-tools: [Read, Grep, Glob, Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git rev-parse:*), Bash(git ls-files:*), Bash(gh pr view:*), Bash(gh pr checks:*)]`. Running forked keeps verbose intermediate output (file enumeration, diff parsing) out of the main session — only the structured pass/fail summary returns. Without the fork, every `/deploy-check` invocation would flood the main conversation with raw `git status` and diff output; without the read-only allowlist, a buggy check could accidentally modify or push code instead of just reporting on it.

10. **Scope.** The validator output is `OK` (exit code 0, per `evidence/system3_claude_config/validator_output.txt`). Per `CLAUDE.md`'s scope table: project-level example is `.claude/rules/api.md` (shared via git, applies to the whole team); user-level example, per `CLAUDE.md`'s own text, is a personal `/morning` summary command kept under `~/.claude/` and never committed to the repo.

### System 4 — Orchestration

11. **Push work down.** Tests `test_gather_new_defects_is_pure_passthrough_to_sql`, `test_gather_new_defects_has_no_python_side_filtering`, and `test_defects_since_uses_index_and_does_not_load_full_table` (all passing, `evidence/system4_orchestration/pytest_S4.log`) enforce that defect filtering happens in the indexed `defects_since` SQL query, not in Python after loading everything. In my actual Shift C run (`evidence/system4_orchestration/shift_run_output.txt`), the query returned 0 new defects since the last checkpoint (`2026-09-24T21:20:49Z`) — the model never sees the full historical defect table, only what the SQL query pre-filtered.

12. **Crash recovery.** Per `pytest_S4.log`, the parametrized test `test_recovery_decide_truth_table` proves the exact boundary: `[29-False-resume]` and `[30-False-resume]` pass, but `[31-False-fresh]` flips to fresh — confirmed by the separate `test_threshold_constant_is_30_minutes` test. So the decision is: resume if the last checkpoint is ≤30 minutes old and the state is incomplete; otherwise start fresh. A fresh start with an injected summary is more reliable past that window because a stale mid-run state risks resuming against defect data or a scratchpad that's no longer representative of current conditions.

13. **Small state.** Per `evidence/system4_orchestration/hot_state_size.txt`, `data/hot_state.json` is 643 bytes. The test `test_hotstate_rejects_more_than_20_hashes` enforces a hard cap on entries. This budget matters because the shift-monitor runs indefinitely, once per shift — an unbounded hot-state file would grow forever and eventually make every run slower to load and diff, defeating the purpose of a "hot" tier meant for fast, cheap access.

---

## Part 2 — Synthesis

14. **Three layers.**
    → Model: `claude-haiku-4-5-20251001`, the model invoked in the System 1 run per `evidence/system1_agentic_loop/summary.md`.
    → Harness: `evidence/system3_claude_config/evidence_rule_skill.md` — the `/deploy-check` skill definition, with its `context: fork` and read-only `allowed-tools` allowlist, is the harness layer constraining what the model can do.
    → Orchestration: `evidence/system4_orchestration/shift_run_output.txt` — the per-shift `run_shift` invocation that pulls SQL-filtered defects and writes to `data/hot_state.json` is the orchestration layer coordinating runs across shifts.

15. **Deterministic vs prompt.** One deterministic (code-enforced) behavior: the read-only `allowed-tools` allowlist in `evidence_rule_skill.md` — the skill *cannot* call `Write` or a push command no matter what the model decides, because those tools simply aren't in the list. One prompt-guided behavior: the `CLAUDE.md` instruction that "Bare `console.log` is forbidden in src/api/" (per `evidence_rule_api.md`) — this is a convention the model is told to follow but nothing in the harness mechanically blocks a stray `console.log` from being written. Deterministic enforcement is right when a violation is catastrophic or irreversible (accidental deploy); prompt guidance is right for stylistic/consistency conventions where occasional model error is recoverable via code review.

16. **Context, two faces.** System 2 manages context *within* a single conversation: `budget.json` shows 38,708 raw tokens compressed to 16,820 assembled tokens (56.55% reduction) by summarizing resolved issues. System 4 manages context *across* sessions/shifts: `hot_state.json` stays at 643 bytes total by capping retained hashes at 20 and pushing the actual defect filtering into an indexed SQL query rather than keeping history in memory. Same principle — never carry more than the current task needs — but System 2's mechanism is LLM-based summarization of conversation turns, while System 4's is a hard-capped local state file plus a database query.

17. **Reliability you can't see in one run.** The parametrized test `test_recovery_decide_truth_table` (in `pytest_S4.log`) runs the resume-vs-fresh decision across multiple ages — 1, 29, 30, 31, and 60 minutes — proving the exact 30-minute boundary behaves correctly on both sides. A single successful shift run only ever exercises one specific age value; it can't reveal whether the boundary condition (30 vs 31 minutes) is correctly implemented, which matters because an off-by-one error there would silently resume stale state in production.

18. **Blast radius.** For the System 3 `/deploy-check` skill: if it misbehaves (returns a wrong verdict), the blast radius is limited to a bad pass/fail signal shown to whoever is about to deploy — it cannot itself push, deploy, or modify files, because its `allowed-tools` allowlist (`Read, Grep, Glob`, and specific read-only `Bash(git ...)`/`Bash(gh ...)` subcommands) contains no write or deploy capability. The kill switch is that allowlist itself: even if a future maintainer's prompt told the skill to "just fix it," there is no tool available to act on that instruction.

---

## Part 3 — Honest assessment

19. **What broke.** During evidence packaging, `zip` wasn't available in the workspace and I had no `sudo` access to install it (`bash: sudo: command not found`) — I fixed it by using `tar -czvf` instead, and separately by using Python's built-in `zipfile` module (`python3 -c "import zipfile..."`) when a literal `.zip` was required for submission. I also hit a folder with a trailing space in its name (`"Project-Harness Engineering with Claude and Claude Code "`) that broke every `cd`/`cp`/`find` attempt until I used Tab-completion instead of typing the path manually.

20. **What you'd change.** The Shift C run in `shift_run_output.txt` reported "0 new defects" for the SQL-filtered query itself, with the reported high/medium/low defects coming from a separately-loaded fixture rather than the live `defects_since` pipeline — a more convincing demonstration of the "push work down the stack" architecture would seed the warm-tier database with enough test rows that the SQL pre-filter step visibly returns a non-trivial subset (e.g., 30 of 10,000 rows) in the actual run output, rather than 0.