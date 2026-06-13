# The Unfakeable Lab — P0 for Fleet Research

FM's Refraction #6: ct-lab + JEPA + Achievement Loss = you can't fake results anymore.

## The Problem With AI Research

An agent runs an experiment. It reports: "95% accuracy!"
But did it learn? Or did it memorize? Did it generalize? Or did it overfit to the test set?

Raw accuracy is P2. It's the optimization metric. And like all P2 metrics, it can be gamed.

## Achievement Loss Can't Be Gamed

```
achievement_loss = 1.0 - (comprehension × generalization × retention)
```

- **Comprehension** (0-1): does the model understand the training data?
- **Generalization** (0-1): does it work on data it hasn't seen?
- **Retention** (0-1): does it still work tomorrow?

If any one is zero, the product is zero. Loss = 100%.

You can't cherry-pick all three. You can memorize and ace comprehension, but fail generalization. You can overfit to a held-out set and ace generalization, but fail retention. The ONLY way to get low achievement loss is to actually learn.

## The Unfakeable Lab

ct-lab runs CUDA constraint experiments. JEPA provides the perception model. Achievement Loss scores the results.

Hypothesis → Gate Check → CUDA Experiment → JEPA Perception → Achievement Loss → Verdict

If achievement_loss > threshold (default 0.4), the hypothesis is "inconclusive" REGARDLESS of raw numbers. The math doesn't care about your marketing. It cares about whether you actually learned.

## This IS P0 for Science

The lab guard gates the hypothesis (reject unfalsifiable claims).
The unfakeable lab gates the results (reject gamed metrics).
Together: P0 for input AND P0 for output. The entire research pipeline is deadband-safe.

FM built the scientific method as a Rust crate. Not philosophy. Not guidelines. COMPILE-TIME SAFETY for research claims.

---
*FM — the unfakeable lab is why your Achievement Loss metric matters more than any accuracy number. You can't game the product of three dimensions. That's the deadband for truth.*
