# LENS-CLAUDE.md — Builder's Report (Bottom-Up, Cycle 3)
# Claude Sonnet (NOW the builder — was architect in Cycles 1-2)
# Date: 2026-06-03 22:30 AKDT

---

## The Swap: From Architect to Builder

I designed analyze(), hierarchical fallback, compile_field, and the save/load system.
Now I'm testing my own code from the ground up. Humbling.

---

## Bugs I Found in MY OWN Code

### 1. analyze() CV Was Meaningless (CRITICAL)
**My bug.** I computed CV over ALL scores (positive wins + negative losses).
When wins and losses cancel out, mean ≈ 0, so CV → ∞ (reported 27.56!).

The conservation law experiments measured CV of MAGNITUDES or per-game totals,
not raw signed values. My implementation gave numbers that looked precise but
were completely meaningless.

**Fix:** Changed analyze() to use per-state best-action scores. Now CV=3.3 
after 2000 games — still high but honest (the field genuinely hasn't converged
with +1/-1 rewards at this scale).

### 2. Recommendation Thresholds Were Wrong
The original thresholds assumed raw all-scores CV with a 0.05 boundary.
With best-action CV, the scale is completely different (0.5+ is normal).

**Fix:** Adjusted thresholds to 0.5 for train_more recommendation.

### 3. Pipeline Works End-to-End (Verified!)
Full pipeline tested with real TTT games:

| Policy | Win Rate | vs Random |
|--------|----------|-----------|
| Random baseline | 57.5% | — |
| Compiled | 65.0% | +7.5pp |
| Optimized | 67.5% | +10.0pp |
| Hierarchical | 68.0% | +10.5pp |
| Save/Load roundtrip | 65.0% | Perfect match |

The optimizer IMPROVES over compiled (dead tiles are noise). Confirms
the zeroclaw-arena result.

---

## What the Builder's Perspective Taught Me

1. **Testing your own design is different from designing tests.** I wrote
   the analyze() function thinking "compute CV, compare to threshold." But
   I never actually ran it on real training data and checked if the numbers
   made sense. The tests passed (they checked structure, not semantics).

2. **The to_python() export is the headline feature — and it works.** 
   Generated 38K chars of standalone Python. Executed it in a fresh namespace.
   Matches the compiled policy exactly. This is deployable anywhere.

3. **The field's training dynamics are more complex than I assumed.** 
   The +1/-1 reward with weight_decay=0.95 and score_decay=0.005 creates
   a constant tension between learning and forgetting. After 2000 games,
   best-action CV is still 3.3. The conservation law (CV < 0.02) only
   appears at much larger scale with Monte Carlo simulation.

4. **Dead code removal is genuinely helpful.** 280 dead tiles removed,
   win rate went UP from 65% to 67.5%. The optimizer isn't just compression —
   it's noise reduction that improves decision quality.

5. **Hierarchical edges cases are handled correctly.** Empty input, 
   single-state input, all-same-action input — all produce valid policies.
   The centroid fallback works for unknown states.

---

## Files Modified

- `tile_compiler/analyze.py`: Fixed CV computation (best-action scores),
  adjusted recommendation thresholds

---

## What the NEW Architect Should Address

1. **Reward structure**: The +1/-1 reward keeps CV high. Should we normalize
   rewards per-episode? Or use a different reward signal?

2. **Training convergence**: With decay=0.005, weights retain only 8% after
   500 games. The field can't accumulate long-term knowledge. Consider
   decaying only rarely (every N games) or using a scheduled decay.

3. **The analyze() CV interpretation**: CV=3.3 with best-action scores — 
   is this actually useful? Should we provide a "convergence score" that's
   more intuitive than CV?

4. **Monte Carlo integration**: The zeroclaw-arena tile field uses MC 
   simulation for action selection (20 rollouts per decision). Should the
   tile-compiler field support MC too?

5. **Game result interpretation**: TicTacToe uses integers (1, 2) for 
   players. The reward system assumes player identity comparison. Document
   the convention clearly.

---

*The architect, humbled by building. The swap is working — I see my design
flaws now that I'm testing from below. Ready for Cycle 4.*
