# Reflection Brief — Harness Engineering Capstone

**Name:** Ayaan Rajak
**Date:** 2nd Nov 2026

**Environment**

- Model(s): `claude-haiku-4-5-20251001` for the live claims run; the context run uses the Anthropic token-count endpoint.
- OS / Python: Linux, Python 3.13.0 (all four `pytest` headers)
- Approx. API spend: $0.1217 recorded for the eight-claim run; the other evidence records token counts but not a dollar total (`Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20261102_082603/summary.md`).

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → The trace in `evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl` has `tool_use → tool_use → end_turn` across turns 1–3. The continue/stop decision is in `exercises/03-dynamic-decomposition/solution/claims_intake/loop.py`, function `run`: it continues only for `response.stop_reason == "tool_use"`, returns for `"end_turn"`, and raises `UnexpectedStopReason` otherwise. A `Budget` check is the safety net, not the primary loop condition.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → `tests/test_antipatterns.py` checks that the loop does not use string-membership tests against model text to control flow. If it did, a wording change in an assistant response could bypass the intended tool protocol or terminate incorrectly; the test suite would flag the AST pattern. The same test also rejects a literal iteration cap, preserving stop-reason-driven behavior.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → `lookup_policy(policy_id)` and `record_claim_fact(field, value)` both accept identifiers/fields plus strings, but their descriptions distinguish lookup (“confirm the policy exists, what is covered, and the deductible”) from accumulation (“one normalized fact” used by routing) in `claims_intake/tools.py`. The schemas also give concrete examples and required fields, reducing model misrouting. Tool failures are structured JSON with `is_error`, `error_category`, and `is_retryable`, so the agent can retry transient failures or choose another action; a generic string would hide that decision signal (`exercises/02-tools-and-audit/README.md`).

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → For `claim_01_kitchen_fire`, the run summary records 3 turns, 9,373 input tokens, 483 output tokens, and `$0.0118` (`Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/solution/runs/20261102_082603/summary.md`). The exercise README’s sample says approximately `$0.05` for the whole run, whereas the completed eight-claim run totals `$0.1217`; that is a different scope and real API usage rather than the sample estimate (`Build a Claims Intake Agent with a stop_reason-Driven Loop/exercises/03-dynamic-decomposition/README.md`).

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → `budget.json` reports 38,708 baseline tokens, 16,803 assembled tokens, and a 56.59% reduction (`evidence/system2_context_strategy/budget.json`). The `active` section dominates at 15,789 tokens because it is the live payment-method issue and contains the most recent conversational detail. It remains verbatim so details such as the exact failure code and user wording are not altered by summarization.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → Summarize resolved segments and preserve the active segment byte-exact. The measured sections are `case_facts: 204`, `resolved_refund: 382`, `resolved_subscription: 446`, and `active: 15,789` tokens in `budget.json`; the active segment is also protected by the byte-exact assertion in `tests/test_assemble.py`. This keeps stable history compact while retaining current-work precision.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → Q6 regressed: the assembled run passed it with the exact token `in_progress`, while `eval_control.jsonl` failed and answered that no structured status token was available (`evidence/system2_context_strategy/eval.jsonl` and `eval_control.jsonl`). The control comparison proves the facts block is not cosmetic: placing structured facts in context makes a precise answer recoverable, while removing it causes a targeted factual failure even though Q1 still passed.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → The API rule frontmatter quotes `paths:` followed by `- "src/api/**/*"` in `.claude/rules/api.md`. That is better than a directory-level `CLAUDE.md` for cross-cutting conventions because the rule activates by file pattern wherever matching files live, including tests or future layout changes, without creating nested instruction hierarchies. The validator evidence confirms API, React, and test matching behavior (`evidence/system3_claude_config/pytest_S3.log`).

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → `.claude/skills/deploy-check/SKILL.md` declares `context: fork` and an allowlist beginning with `allowed-tools: Read, Grep, Glob` plus read-only Git/GitHub commands. Forking keeps verbose discovery output out of the parent session, while the allowlist prevents mutation, pushes, or deployment; without either, a deploy check could pollute the main context and acquire unsafe write capability. The file explicitly states that it returns only a structured pass/fail summary.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator output is `OK`, and the 35-test log shows the scope checks passed (`evidence/system3_claude_config/validator_output.txt`, `pytest_S3.log`). Project-level examples are `./CLAUDE.md` and `.claude/skills/deploy-check/`, committed and shared with the team. User-level examples are `~/.claude/CLAUDE.md` and `~/.claude/skills/deploy-check-strict/`, described in `CLAUDE.md` as personal and not versioned.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The indexed query returns 17 April defects from the fixture, while the warm-tier test also verifies `limit=5` is respected (`tests/test_us01_tiered_state.py`). The query is `SELECT * FROM defects WHERE ts > ? ORDER BY ts DESC LIMIT ?` in `shift_monitor/warm.py`, and `EXPLAIN QUERY PLAN` must show `idx_defects_shift_ts`. Filtering, ordering, and limiting happen in SQLite, so the model receives only the selected defect rows rather than the full warm history.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → `recovery.py` resumes only when the manifest is incomplete, non-empty, and the last step is no more than 30 minutes old; otherwise it starts fresh. The truth-table tests verify 30 minutes resumes and 31 minutes starts fresh (`tests/test_us03_crash_recovery.py`). Fresh is sometimes safer because a stale partial may reflect an obsolete working set, while a new invocation can receive the manifest’s captured findings as a concise injected summary instead of replaying corrupted or outdated conversational state.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → The captured `hot_state.json` is 643 bytes (`evidence/system4_orchstration/hot_state_size.txt`), under the 5,120-byte budget enforced by `HotState.write_atomic` in `shift_monitor/state.py`. A shift-run system operates indefinitely, so an unbounded hot state would eventually crowd out current defects and increase prompt cost or exceed model context. The budget makes the long-lived state predictable and the atomic replacement prevents partial writes.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → Model: `budget.json` and `claim_01_kitchen_fire.jsonl` show the model using compressed context and stop-reason/tool calls rather than receiving an undifferentiated transcript.
    → Harness: `claims_intake/loop.py` and `.claude/skills/deploy-check/SKILL.md` enforce protocol control, fork isolation, and read-only tools.
    → Orchestration: `warm.py`, `recovery.py`, and `state.py` provide indexed retrieval, crash decisions, bounded state, and atomic persistence. These are separate artifacts because the model chooses actions, the harness constrains actions, and orchestration controls durable history.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A behavior guaranteed in code is the read-only deploy allowlist (`SKILL.md`) and the atomic, budget-checked hot-state write (`state.py`); tests can verify these without trusting model wording. A prompt-guided behavior is asking one clarification only when claim type is genuinely ambiguous, as specified by `tools.py` descriptions. Deterministic enforcement is right for safety and invariants; prompts are right for semantic judgment where rigid branching would make the harness own domain decisions.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 reduces 38,708 baseline tokens to 16,803 assembled tokens, preserving the active segment; System 4 reduces cross-shift state to a 643-byte hot file and retrieves only bounded SQL results (`budget.json`, `hot_state_size.txt`). Both apply the same principle—keep the smallest reliable working set—but System 2 transforms one session’s transcript, while System 4 persists and queries state across sessions and shifts.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → The crash-recovery tests guarantee behavior a successful run would not reveal: a 30-minute incomplete manifest resumes, a 31-minute one starts fresh, and a complete manifest always starts fresh (`tests/test_us03_crash_recovery.py`). A single healthy run exercises neither the crash boundary nor stale-state path. Shipping without this guarantee could silently resume obsolete work after an outage and produce misleading shift conclusions.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → System 4 has the largest blast radius: a bad retrieval or stale state could cause an entire shift’s quality-monitoring decision to miss defects or recommend the wrong quarantine. Its controls are the indexed `defects_since` query with a row limit, the 5,120-byte hot-state budget, atomic writes, the 30-minute recovery threshold, and the single shift invocation tested in `pytest_S4.log`. The practical kill switch is to stop the scheduled shift invocation and inspect the durable warm/cold records before resuming.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → Nothing in the completed artifacts shows a failed first attempt; I checked the evidence logs and all recorded final suites are green: 30 tests for System 1, 35 for System 3, and 33 for System 4 (`evidence/system1_agentic_loop/pytest_S1.log`, `pytest_S3.log`, `pytest_S4.log`).

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → I would make the cost/telemetry contract a first-class shared artifact instead of recording dollar cost only in the claims summary. The context pipeline has authoritative token counts and the orchestration pipeline has testable state sizes, but neither joins usage, latency, and cost into one run manifest (`budget.json`, `summary.md`, and `shift_run_output.txt`). A shared run ledger would make cross-system cost regressions and operational trade-offs visible before deployment without changing the existing correctness boundaries.
