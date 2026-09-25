# Reflection Brief — Harness Engineering Capstone

**Name:** Manas Kulkarni
**Date:** September 25, 2026

**Environment**

- Model(s): Claude through the provided Vocareum/Anthropic environment; System 4 was run using the recorded response fixture.
- OS / Python: Linux workspace / Python 3.13
- Approx. API spend: System 1 recorded an estimated total cost of $0.1282. System 2 also used the API during its evaluation run.

---

## Part 1 — Per-system

### System 1 — Agentic loop

1. **Loop control.** Quote the `stop_reason` sequence from one trace. Name the file and function that decides continue-vs-stop, and how.
   → I checked the trace for `claim_08_minor_porch_damage` in `runs/20260925_112850/traces/claim_08_minor_porch_damage.jsonl`. Its control sequence was `tool_use → tool_use → tool_use → end_turn`. The decision is handled by the `run()` function in `claims_intake/loop.py`. When the response is `tool_use`, the loop continues and executes the requested tool, while `end_turn` causes the loop to finish.

2. **Anti-pattern.** Name one anti-pattern `test_antipatterns.py` checks for. What would break in your run if the loop used it?
   → One anti-pattern checked by `test_antipatterns.py` is using a fixed integer iteration limit as the main loop control. That would be a problem because different claims need different numbers of tool calls. For example, `claim_03_water_damage` used 6 turns in my run. A fixed limit that was too small could stop the agent before it finished collecting information and routing the claim. The test therefore protects the `stop_reason`-based design.

3. **Tool design.** Pick two tools with overlapping inputs. How do the descriptions prevent misrouting? What did a structured tool error let the agent do that a generic string would not?
   → Two tools that can receive information from the same claim are `classify_claim` and `assess_severity`. Their descriptions and structured schemas make their individual purposes different, so the model has guidance about which operation belongs to which tool. The structured tool error is also useful because it gives the agent information about what went wrong instead of only returning an arbitrary text message. This means the agent can recognize an input or validation problem and decide whether another attempt is appropriate.

4. **Your numbers.** Quote the turn count and cost for one claim. How does it differ from the README sample, and why?
   → My `claim_03_water_damage` run took **6 turns** and had an estimated cost of **$0.0275**. The evidence is in `runs/20260925_112850/summary.md`. It used 22,113 input tokens and 1,086 output tokens and required one clarification. The result can differ from a README example because the actual model response determines how many tool calls and clarifications are required.

### System 2 — Context strategy

5. **The reduction.** From `budget.json`: baseline tokens, assembled tokens, reduction %. Which section dominates the assembled context, and why keep it verbatim?
   → My `budget.json` shows **38,708 baseline tokens** and **16,877 assembled tokens**, giving a **56.4% reduction**. The biggest section is `active`, with **15,789 tokens**. The other sections are much smaller: `case_facts` is 204, `resolved_refund` is 394, and `resolved_subscription` is 508 tokens. The active section is kept intact because it represents the current issue and changing its wording could remove details needed to answer the user's current question.

6. **Summarize vs preserve.** State the rule for what gets summarized vs kept byte-exact, citing your per-section token numbers.
   → The basic rule I observed is to summarize information that has already been resolved while keeping the currently active conversation exact. In my run, `resolved_refund` was 394 tokens and `resolved_subscription` was 508 tokens, while the active section remained 15,789 tokens. This keeps older information available in a compact form but gives the model the full current conversation. The approach reduces the overall context without throwing away the details that are still needed.

7. **Facts block.** Compare `eval.jsonl` to `eval_control.jsonl`. Which question regressed, and what does that prove?
   → The normal evaluation in `eval.jsonl` passed all **6 out of 6 questions**. It correctly answered details such as the `$22.14` refund, the duplicate-charge cancellation reason, `AVS_MISMATCH`, card ending `7782`, the `$28.18` combined refund, and the `in_progress` status. In `eval_control.jsonl`, Q6 failed while Q1 unexpectedly passed. This shows that the assembled context preserved the information required by the intended evaluation, while the control context did not consistently preserve all of the required information.

### System 3 — Claude Code config

8. **Path-scoped rules.** Quote the glob frontmatter from one rule file. Why is it better than a directory-level CLAUDE.md for cross-cutting conventions?
   → My System 3 evidence contains the path-scoped rule files `evidence_rule_react.md`, `evidence_rule_api.md`, and `evidence_rule_tests.md`. **[VERIFY: paste the exact `paths:` glob from `evidence_rule_react.md` here.]** The advantage of path-scoped rules is that a rule can be applied only to files where it is relevant. A directory-level `CLAUDE.md` would affect a much broader area, including files that may not need the rule. This makes the configuration more targeted and avoids unnecessary instructions in unrelated work.

9. **Forked skill.** Quote the `context: fork` and `allowed-tools` lines. What does running forked + read-only buy you? What breaks without it?
   → The skill evidence contains `context: fork` and a restricted `allowed-tools` list containing read-oriented tools such as `Read`, `Grep`, and `Glob`, along with limited Git/GitHub inspection commands. The fork means the detailed exploration can happen separately from the main conversation, so the parent context does not become filled with all of the intermediate output. The read-only restriction is useful because the skill is intended for checking and investigation rather than modifying the project. Without these controls, the skill could either add unnecessary context to the main session or have a larger write/deployment surface.

10. **Scope.** From the validator output: project-level vs user-level scope. Give one example of each from this config.
    → The validator output in `validator_output.txt` is `OK`, so the configuration passed the validator. The project-level examples are `CLAUDE.md`, `.claude/rules/`, `.claude/commands/review.md`, and `.claude/skills/deploy-check/SKILL.md`. These are stored with the project and are intended to be shared with the team. A user-level example would be a corresponding configuration under `~/.claude/`, which would apply only to that individual user's environment rather than the repository.

### System 4 — Orchestration

11. **Push work down.** Defects the SQL query returned vs warm-tier total. Name the indexed query. Why does the model never see the full history?
    → The System 4 warm store uses SQLite to keep historical defects outside the model context. The relevant query in `shift_monitor/warm.py` is the `defects_since` query, which filters defects by timestamp and orders the results instead of sending the entire table to the model. The warm tier can therefore contain much more information than one shift needs. The model only receives the subset selected for the current shift, which keeps its context smaller and makes the filtering deterministic.

12. **Crash recovery.** The resume-vs-fresh decision and its staleness threshold (`recovery.py`). Why is a fresh start with an injected summary sometimes more reliable than resuming?
    → The recovery logic in `shift_monitor/recovery.py` uses a **30-minute** staleness threshold. A recent incomplete run can be resumed, but an older or stale run is treated as a reason to start fresh. Resuming stale intermediate state could carry forward incomplete reasoning or outdated information. Starting a new session with the persisted summary gives the model a clean context while still preserving the important state from the previous shift.

13. **Small state.** Byte size of your `hot_state.json`. Why does the budget matter for a system run once per shift, indefinitely?
    → My evidence contains the exact filesystem size in `evidence/system4_orchestration/hot_state_size.txt`. **[VERIFY: insert the exact byte count from that file here.]** The important part is that hot state should remain small instead of growing with every shift. Historical information belongs in the warm tier, while hot state contains only information needed for the next run. This matters for a system that runs indefinitely because uncontrolled state growth would eventually increase context size, processing time, and cost.

---

## Part 2 — Synthesis

14. **Three layers.** Point to a file/artifact for each layer and justify.
    → **Model:** `evidence/system1_agentic_loop/claim_01_kitchen_fire.jsonl` — this shows the model/tool interaction and the `stop_reason`-driven behavior.
    
    → **Harness:** `evidence/system3_claude_config/evidence_rule_skill.md` — this shows the Claude Code configuration, including the forked context and restricted tools.
    
    → **Orchestration:** `evidence/system4_orchestration/shift_run_output.txt` and `hot_state_size.txt` — these show the shift-level execution and persisted state. Together, these artifacts helped me separate model behavior from the controls around the model and the longer-running state management.

15. **Deterministic vs prompt.** Cite one behavior guaranteed in code (terminal tool, read-only allowlist, atomic write, byte budget) and one guided by prompt. When is each right?
    → A deterministic example is the read-only `allowed-tools` list shown in `evidence_rule_skill.md`. The configuration can restrict the tools even if the model asks to perform another type of action. A prompt-guided example is deciding which claim-related tool should be used based on the claim information. I would use deterministic rules for things such as permissions, state integrity, and safety boundaries, while prompts are more suitable for flexible reasoning and task selection.

16. **Context, two faces.** Compare context management in System 2 (intra-session) and System 4 (cross-session) with cited numbers from both. Same principle, different mechanism — how?
    → System 2 reduced the context from **38,708 tokens to 16,877 tokens**, which was a **56.4% reduction**. It achieved this by summarizing resolved parts while preserving the active part. System 4 handles the same general problem across different sessions by keeping small hot state and storing historical defects in the warm SQLite tier. So System 2 controls context within one conversation, while System 4 controls what information is carried from one shift to another.

17. **Reliability you can't see in one run.** Name one behavior a test guarantees that a single successful run would not reveal. Why does it matter before shipping?
    → One example is the System 1 test `tests/test_loop.py::test_loop_raises_on_unexpected_stop_reason`. A normal successful run may only show `tool_use` followed by `end_turn`, so I would not see what happens when the API returns an unexpected stop reason. The test verifies that this situation is handled explicitly instead of silently continuing. This matters because production systems eventually encounter unexpected API responses, and the failure behavior should be predictable before the system is shipped.

18. **Blast radius.** Pick one system. What's the blast radius if it misbehaves, and what's the kill switch? Ground it in that system's tools, enforcement points, and state.
    → I would use System 4 as the example. If its orchestration logic behaved incorrectly, it could affect the state passed between quality-monitoring shifts. The design limits this by keeping historical data in the warm tier, filtering the relevant records before the model sees them, and using recovery logic for incomplete sessions. If a previous run becomes stale, the system can choose a fresh session instead of continuing from the old state, which gives a practical recovery mechanism.

---

## Part 3 — Honest assessment

19. **What broke.** One thing that failed first try in your environment, and how you fixed it. (If nothing, what you checked to be sure.)
    → The first problem I encountered was copying the reflection template. My original command failed because the actual directory name contained a trailing space: `Project-Harness Engineering with Claude and Claude Code `. The error was `cannot stat`, so I used `find` to locate the actual file and then copied it using the exact path. After that, `/workspace/my-reflection-brief.md` was created successfully and I verified it with `ls -lh` and `cat`.

20. **What you'd change.** One architectural decision you'd make differently, grounded in what you observed.
    → If I were improving the project, I would automate the final evidence validation instead of checking every folder manually. During my run, the systems produced the required artifacts, but I still had to locate the reflection template, rename two evidence directories, and manually check individual files. A final validation script could check the four expected evidence directories, required artifact names, test logs, and whether the reflection still contains any `→` placeholders. This would make the final submission process more reliable and reduce the chance of forgetting one deliverable.