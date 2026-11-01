# What replay tests miss: verifying a clean-room rewrite of an LLM agent harness

**Subtitle:** Deterministic replay checks passed on every measure; a pre-registered live test still failed, and the cause was a code example in the prompt.

## Summary

We rewrote a third-party ARC-AGI-3 agent harness from a behavioral spec, with implementers that never saw its source, so that our world-model gate could ship as our own code, and then used it for nine pre-registered tests of harness mechanisms. We checked equivalence in two layers. Four deterministic checks replay recorded games, sandbox cells and model responses through both harnesses. After ten rounds of fixes all passed, including 1,504 of 1,504 context-trimming points. Replay cannot test how the model reacts to new prompt wording, so we ran a pre-registered live non-inferiority test. The first run failed: a worked code example in the rewrite's prompt raised the share of runs with an accepted world model from 18/30 to 26/30. After removing it and restoring the original's facts, a second run passed all three rules.

## 1. Why a rewrite, and why it had to be checked

Our method is a world-model gate on an existing ARC-AGI-3 harness (Duck, `ARC3-Inference`). After 8 actions on a level, the gate refuses actions until the model's `predict` function reproduces at least 75% of the recorded board-changing transitions. Our own code was 383 lines of gate, 397 lines of tests and small hooks in five files; the third-party code on the competition path was about 10,500 lines. To submit the method as our own code, we replaced that code with a clean-room rewrite. A rewrite counts as evidence only if it plays like the harness the method was measured on.

**Rules.** Implementer agents got the spec, our own files and public APIs, never the third-party trees; a separate verifier agent ran the original. Functional identifiers (environment variables, file names, JSON keys, sandbox globals, action labels) could match; prose, code structure and prompt wording had to be new, and prompts were written from a topic list.

**Setting.** The competition runs offline on Kaggle with one RTX PRO 6000 and vLLM serving Qwen3.6-27B-FP8. Each turn, the model gets a text prompt and a grid image, writes Python cells in a sandbox, and acts through `action()`. Context is trimmed by an estimate of `len(json)/3` against a 31,744-token budget.

## 2. Layer one: deterministic checks

Live runs are noisy (p = 0.86 for one config across two platforms), so we first compared the harnesses with model output held fixed.

| Check | What it compares | Final result |
|---|---|---|
| D1 engine replay | 30 recorded action histories: levels, per-level actions, score | pass (implementer's suite) |
| D2 record/replay | recorded model messages served to both harnesses on 6 games: actions, gate logs, tool results, message count per request | 30/30 runs; 1,504/1,504 trimming points |
| D3 sandbox corpus | 1,473 recorded Python cells: stdout, result, error type, action requests | 1,473/1,473 action streams; 637/637 backtests |
| D4 competition path | gateway simulator, stub model: scorecards, game ids | pass with 6 games and 110 ids |
| 8-gram overlap | shared 8-word runs with third-party prompt text | 0 hits in prompt text |

It took ten rounds: in round 1 the system prompt was 5,113 characters against 14,065, and trimming differed in 550 of 983 requests. Because the estimator counts JSON-escaped characters, every template's serialized size had to match exactly; round 10 reached a difference of 0 across 362 paired messages and 1,504 of 1,504 matching trimming points.

D2 verifies every model input except prompt text, so it cannot show how the model reacts to new wording.

## 3. Layer two: a pre-registered live test

The plan was approved before any data: the rewrite on 6 games, 5 passes each, against the original's enf10 run with the same settings. All three rules had to hold: levels non-inferior by a one-sided Welch test with a 0.5 margin at α = 0.10; gate acceptance in 13 to 23 runs of 30; median throughput within 10%. We first computed how often draft rules would fail an identical rewrite, about 50%, and widened them to about 6% before running anything. The cost: S1 detects only drops of about 0.7 levels or more.

| Rule | Original enf10 | S1 (fail) | S1b (pass) |
|---|---|---|---|
| Levels per game run | 0.500 | 0.533 (p 0.0003) | 0.467 (p 0.0007) |
| Runs with an accepted model | 18/30 | 26/30 | 19/30 |
| Median throughput, tok/s | 165.85 | 167.1 | 163.2 |

**Cause of the failure.** With every other input shown identical, we compared behavior and prompt content. The rewrite's model wrote `predict` earlier and more often, and captures passing the backtest rose from 21% to 34%. All 246 of its `predict` cells parsed the board with the line-splitting idiom from a worked code example in the rewrite's gate instructions; none of the original's 187 did. The two system prompts were the same length (13,913 against 13,899 characters), but the rewrite's gate section added the template, a self-test snippet and a construction strategy, and dropped four facts the original states.

**Fixes.** Three rounds, each re-checked deterministically, cut the gate section to the original's facts in its order (1,238 against 1,230 characters) and removed a simulate-on-a-copy hint and two claims that make prediction look easy. In the second run (S1b), the template idiom fell to 8% of 291 `predict` cells and all three rules passed. Two limits: the fixes were chosen after seeing S1 data on the same 6 games, so S1b is not independent; and the rounds changed several things together, so the template alone is not shown to be the cause.

## 4. Lessons

1. **Matching facts and sizes is not enough.** The rewrite matched every deterministic measure and still changed the model's behavior. In a clean-room rewrite of an LLM harness, prompt content is part of the behavior and needs its own test.
2. **An example in a prompt is an instruction.** The model copied the template into every `predict` cell. If the original has no example, the rewrite should have none.
3. **Check pass rules against noise before collecting data.** The draft rules would have failed an identical rewrite half the time. Running the original against itself set the noise floor inside the deterministic checks too (15 D3 cells of set-order disagreement).

## 5. What the rewrite was then used for: nine controlled tests

On the verified rewrite we ran nine pre-registered arms (6 games × 5 passes, 2,700 s per game) against the unchanged harness (mean score 1.399). Six changed what the model is told or allowed (gate rules, feedback, probe budget, image history). None beat the base beyond run-to-run noise (standard error about 0.25 per arm). An arm that looked better (1.565) fell back in its exact replicate (1.302).

The logs explain why: at about 17 tokens/s per stream a game gets roughly 44 model calls before its cap, and the GPU is saturated at 8–10 concurrent games. Image elision cost about 25% of calls through longer reasoning; the gate stalled 48 of 150 runs at its 8-action wall.

Only one change enlarged the budget: speculative decoding with the model's multi-token-prediction head. The FP8 checkpoint ships no MTP weights; we grafted the 15 BF16 tensors from the full-precision model. With 2 draft tokens, 2.7 tokens were accepted per step and calls per game rose 25–30% in two runs. Scores were 2.222 (p = 0.025) and 1.523; pooled, +0.47 over the base (p = 0.076), not significant.

**Lesson 4.** On a fixed GPU budget, prompt and gate changes reshuffle a fixed number of decisions; measure calls per game first.

## 6. Limitations

S1 has low power. S1b ran on RunPod while the comparator ran on Kaggle (same GPU model, different driver and image). D1 was not re-run by the verifier. Mean score was lower in both rewrite runs (1.12 and 1.16 against 1.41) and had no pass rule; a permutation test cannot separate the gap from noise (p of about 0.5).

S1 used more actions per completed level than the original (median 0.97 of the human baseline against 0.67); S1b was back at 0.65.

The clean-room checks have two gaps: the 8-gram check sees only exact 8-word runs, and no check compared code structure (the implementers' not seeing the source is the safeguard). One restored prompt sentence was later found close to the original's wording; it was reworded and all deterministic checks re-run, but S1b's live model never saw the new wording. The leaderboard submission (57752331, e2fa28e) and all nine later arms used the reworded prompt.

## 7. Submission and links

- Leaderboard submission of the rewrite: draw of Oct 29, submission 57752331, public score 0.55.
- Full method, all checks and per-game tables: https://github.com/quanticsoul4772/arc-wm-paper (paper.pdf).
- Code (CC0): https://github.com/quanticsoul4772/arc-wm-agent. It re-implements The Duck (Tufalabs/duck-harness) without its source; that repository has no LICENSE file; its harness package lists an MIT classifier.
