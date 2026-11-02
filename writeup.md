# What replay tests miss: verifying a clean-room rewrite of an LLM agent harness

**Subtitle:** Replay checks passed, a live test caught a prompt example, and nine tests showed decisions per game bound the score; MTP adds 25–30% more.

## Summary

We rewrote a third-party ARC-AGI-3 agent harness from a behavioral spec, with implementers that never saw its source, so that our world-model gate could ship as our own code, and then used it for nine pre-registered tests of harness mechanisms. Four deterministic replay checks all passed after ten rounds of fixes. Replay cannot test how the model reacts to new prompt wording, so we ran a pre-registered live non-inferiority test. The first run failed: a worked code example in the rewrite's prompt raised the share of runs with an accepted world model from 18/30 to 26/30. After removing it and restoring the original's facts, a second run passed all three rules. The nine tests then showed that, at about 17 tokens/s, a game's score is bounded by how many model calls fit in its time cap, not by what the prompt says.

## 1. Why a rewrite, and why it had to be checked

Our method is a world-model gate on an existing ARC-AGI-3 harness (Duck, `ARC3-Inference`). After 8 actions on a level, the gate refuses actions until the model's `predict` function reproduces at least 75% of the recorded board-changing transitions. Our own code was about 800 lines; the third-party code on the competition path was about 10,500. To submit the method as our own code, we replaced that code with a clean-room rewrite. A rewrite counts as evidence only if it plays like the harness the method was measured on.

**Rules.** Implementer agents got the spec, our own files and public APIs, never the third-party trees; a separate verifier agent ran the original. Functional identifiers could match; prose, code structure and prompt wording had to be new, and prompts were written from a topic list.

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

It took ten rounds. Because the trimming estimator counts JSON-escaped characters, every template's serialized size had to match exactly: in round 1 trimming differed in 550 of 983 requests; in round 10 it matched in 1,504 of 1,504.

D2 verifies every model input except prompt text, so it cannot show how the model reacts to new wording.

## 3. Layer two: a pre-registered live test

The plan was approved before any data: the rewrite on 6 games, 5 passes each, against the original's enf10 run with the same settings. All three rules had to hold: levels non-inferior by a one-sided Welch test with a 0.5 margin at α = 0.10; gate acceptance in 13 to 23 runs of 30; median throughput within 10%. We first computed how often draft rules would fail an identical rewrite, about 50%, and widened them to about 6% before running anything. The cost: S1 detects only drops of about 0.7 levels or more.

| Rule | Original enf10 | S1 (fail) | S1b (pass) |
|---|---|---|---|
| Levels per game run | 0.500 | 0.533 (p 0.0003) | 0.467 (p 0.0007) |
| Runs with an accepted model | 18/30 | 26/30 | 19/30 |
| Median throughput, tok/s | 165.85 | 167.1 | 163.2 |

**Cause of the failure.** With every other input shown identical, we compared behavior and prompt content. The rewrite's model wrote `predict` earlier and more often, and captures passing the backtest rose from 21% to 34%. All 246 of its `predict` cells parsed the board with the line-splitting idiom from a worked code example in the rewrite's gate instructions; none of the original's 187 did. The system prompts were the same length, but the rewrite's gate section added the template and two other hints and dropped four of the original's facts.

**Fixes.** Three rounds, each re-checked deterministically, cut the gate section to the original's facts in its order (1,238 against 1,230 characters) and removed a simulate-on-a-copy hint and two claims that make prediction look easy. In the second run (S1b), the template idiom fell to 8% of 291 `predict` cells and all three rules passed. The fixes were chosen after seeing S1 on the same games and changed several things together, so S1b is not independent and the template alone is not shown to be the cause.

## 4. Lessons

1. **Matching facts and sizes is not enough.** In a clean-room rewrite of an LLM harness, prompt content is part of the behavior and needs its own test.
2. **An example in a prompt is an instruction.** The model copied the template into every `predict` cell. If the original has no example, the rewrite should have none.
3. **Check pass rules against noise before collecting data.** The draft rules would have failed an identical rewrite half the time.

## 5. What the rewrite was then used for: nine controlled tests

On the verified rewrite we ran nine pre-registered arms (6 games × 5 passes, 2,700 s per game) against the unchanged harness (mean score 1.399). Six changed what the model is told or allowed. None beat the base beyond run-to-run noise (standard error about 0.25 per arm). An arm that looked better (1.565) fell back in its exact replicate (1.302).

The GPU is saturated at 8–10 concurrent games. Image elision cost about 25% of calls through longer reasoning; the gate stalled 48 of 150 runs at its 8-action wall.

Only one change enlarged the budget: speculative decoding with the model's multi-token-prediction head. The FP8 checkpoint ships no MTP weights; we grafted the 15 BF16 tensors from the full-precision model. With 2 draft tokens, 2.7 tokens were accepted per step and calls per game rose 25–30% in two runs. Scores were 2.222 (p = 0.025) and 1.523; pooled, +0.47 over the base (p = 0.076), not significant.

**Why this happens.** Decisions per game ≈ tokens/s × cap ÷ tokens per call (17 × 2,700 ÷ ~1,050 ≈ 44). The GPU is saturated, so tokens/s is fixed, and harness changes act only on tokens per call: a gate, a note or a feedback line moves calls between kinds of thinking without adding any. Elision lowered the count, most likely because the model described past boards in text, which stayed in context and lengthened every later call. Only faster decoding raises the numerator.

**Lesson 4.** On a fixed GPU budget, measure calls per game before testing behavior.

## 6. Limitations

S1 has low power. S1b ran on RunPod, the comparator on Kaggle. D1 was not re-run by the verifier. Mean score was lower in both rewrite runs (1.12 and 1.16 against 1.41) and had no pass rule; a permutation test cannot separate the gap from noise (p of about 0.5).

The clean-room checks have two gaps: the 8-gram check sees only exact 8-word runs, and no check compared code structure (the implementers' not seeing the source is the safeguard). One restored prompt sentence was later found close to the original's wording; it was reworded and all deterministic checks re-run, but S1b's live model never saw the new wording. The leaderboard submission (57752331, e2fa28e) and all nine later arms used the reworded prompt.

## 7. Submission and links

- Leaderboard submissions of the rewrite: 57752331 (Oct 29, base harness, public score 0.55) and 57910428 (Nov 2, with MTP).
- Full method, all checks and per-game tables: https://github.com/quanticsoul4772/arc-wm-paper (paper.pdf).
- Code (CC0): https://github.com/quanticsoul4772/arc-wm-agent. It re-implements The Duck (Tufalabs/duck-harness) without its source; that repository has no LICENSE file; its harness package lists an MIT classifier.
