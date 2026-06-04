# LENS-CLAUDE.md — Architect's Vision (Top-Down, Cycle 2)
# Claude Sonnet (deep thinking)
# Date: 2026-06-03 21:45 AKDT
# Revised after reading builder's Cycle 1 report (12 bugs fixed)

---

## Response to Builder's 5 Questions

The builder found real problems. Here are concrete decisions on each.

### 1. GPU Module Is Theater — Real Batch Ops or Honest Removal?

**Decision: Remove it. Replace with optional `accelerated.py` that's honest about what it does.**

The current `gpu.py` builds a GPU tensor then iterates in Python. That's worse than useless — it's misleading. Users will think they're getting GPU acceleration when they're not.

The real GPU acceleration opportunity is in SVD factorization (86× faster on our benchmarks) and batch evolution. But that requires `torch` as a real dependency, and our thesis is zero-dep core.

**Concrete action:**
```python
# tile_compiler/accelerated.py (optional, requires torch)
def batch_factorize(field_weights, rank=None, energy=0.95, device="cuda"):
    """SVD factorization using torch.linalg.svd. 86× faster at 5K+ tiles."""
    ...

def batch_evolve(fields, n_generations=10, device="cuda"):
    """Evolve multiple fields in parallel on GPU."""
    ...
```

The pure-Python `factorize.py` stays as the default. `accelerated.py` is opt-in via `pip install tile-compiler[gpu]`. No fake GPU code.

### 2. Hierarchical Doesn't Generalize Within Clusters

**Decision: Add centroid fallback with confidence threshold.**

The builder is right — `choose()` requires exact state key match even within clusters, which defeats the purpose. The fix is a two-level lookup:

```python
def choose(self, state):
    key = hash_state(state)
    
    # Level 1: exact match (current behavior)
    if key in self._assignments:
        cluster_id = self._assignments[key]
        return self._cluster_tables[cluster_id].get(key)
    
    # Level 2: centroid fallback (NEW)
    # Convert state to feature vector, find nearest centroid
    # Return the centroid's consensus action
    vec = self._state_to_vector(state)
    nearest = self._nearest_centroid(vec)
    return self._centroid_action[nearest]
```

The `_centroid_action` dict stores the majority-vote best action for each centroid. States not seen during training get their cluster's consensus action. This is the actual generalization the hierarchical module promises.

Edge case: if no centroids are trained, return `None`. If state can't be converted to a feature vector, return `None`.

### 3. `compile` Shadows Python Builtin

**Decision: Keep `compile` as the module-level function, add `compile_field` as alias.**

The builder flagged that `from tile_compiler import compile` shadows Python's `compile()`. This is a real issue — it breaks code that uses both.

But renaming to `compile_policy` makes the 3-line API worse:
```python
# Current (clean):
policy = compile(field)

# Proposed (worse):
policy = compile_policy(field)
```

**Compromise:** `compile` stays as the primary name (it's the obvious verb), but we add `compile_field` as an alias for users who need both. Document the shadow in the docstring with a note.

```python
# tile_compiler/__init__.py
from tile_compiler.compiler import compile, compile_field

# compile_field is an alias for compile, for users who need Python's compile()
compile_field = compile
```

### 4. No Stable Cross-Language Serialization Format

**Decision: JSON with a versioned schema. Add `to_c_header()` for embedded.**

The compiled policy is a `dict[int, Any]` — the keys are blake2b hashes (integers), values are actions (any hashable). For cross-language deployment, we need a format that C, Rust, WASM, and JavaScript can all read.

**Schema v1:**
```json
{
  "version": 1,
  "format": "tile-policy-v1",
  "hash": "blake2b-8",
  "size": 214,
  "actions": ["0", "1", "2", "3", "4", "5", "6", "7", "8"],
  "table": {
    "1234567890": 4,
    "9876543210": 0
  }
}
```

The `actions` array establishes an action vocabulary. Table values are indices into that array. This is language-agnostic: any language with a JSON parser and blake2b can use it.

For C embedded: `to_c_header()` generates a `.h` file with a `static const` lookup table. No JSON parser needed.

```python
def to_c_header(self, name="policy") -> str:
    """Generate a C header file with the compiled lookup table."""
    ...
```

### 5. CSE Pass Is Now a No-Op — Needs Semantic Analysis Strategy

**Decision: Accept that CSE is currently a no-op and document it honestly.**

The builder fixed the CSE pass to stop dropping states, which made it a no-op. A real CSE would need to know that two different states always lead to the same game-tree position — that requires game-specific semantics we can't encode generically.

**The honest position:** CSE is reserved for future work. The 5-pass pipeline is really 3 active passes (dead code, constant fold, inline) + 1 passthrough (CSE) + 1 deploy. The stats report reflects this honestly (`cse_removed: 0`).

**Future path:** If we add a `Game` parameter to `optimize()`, CSE could query the game for state equivalence. But that's a v0.4+ feature. Right now the dead code + constant fold + inline passes give us 96.5% reduction on their own.

---

## Revised Architecture After Builder's Findings

### What Changed My Mind

1. **The hash bug is terrifying.** The builder found that `hash()` works in tests but breaks in production across processes. This is exactly the kind of subtle bug that makes it to production and destroys trust. The centralized `hash_state()` is critical infrastructure.

2. **Test quality matters more than test count.** Going from 38→56 tests is good, but the builder proved 38 of those tests were lying. The regression tests are the most valuable additions because they test behavior, not API shape.

3. **The GPU module is a liability, not an asset.** It promises performance it doesn't deliver. Better to have no GPU support than fake GPU support. Honest beats impressive.

4. **Pruning helps performance.** The builder's fix to the optimizer aligns with our experimental result: dead code elimination improves accuracy, not just compression. The optimizer is doing real work.

### Revised File Structure

```
tile_compiler/
├── __init__.py          # Public API + hash_state
├── protocol.py          # Game Protocol + validate_game  
├── field.py             # TileField (train, choose, evolve)
├── compiler.py          # compile() → CompiledPolicy + save/load/export
├── optimizer.py         # optimize() 5-pass pipeline
├── factorize.py         # factorize() SVD compression (pure Python)
├── jit.py               # jit_compile() hot-path discovery
├── hierarchical.py      # hierarchical_compile() with centroid fallback
├── distill.py           # distill() ensemble (documented limitations)
├── analyze.py           # NEW: diagnostics and reports
└── accelerated.py       # NEW: optional GPU batch ops (honest)
```

No `policies/` subpackage — the builder's flat structure works fine and avoids import complexity. The architect's instinct to reorganize was premature.

### What I Was Wrong About (Cycle 1)

1. **Pipeline composition (`.factorize().optimize()`)** — Not needed yet. The functional API `optimize(factorize(field))` works. Chaining can wait.

2. **Removing distill()** — The builder kept it and added honest documentation. Better to have a documented limitation than to remove functionality someone might be using.

3. **Built-in games** — Still valuable, but not blocking. The protocol + examples is enough for now.

4. **`Arena` class** — Premature. Users can write a 10-line loop.

---

## ONE New Feature for the Builder: `analyze()`

The builder should implement `analyze.py` — a diagnostics module that exposes our experimental insights as a user-facing tool.

### Function Signature

```python
# tile_compiler/analyze.py

from dataclasses import dataclass
from typing import Any

@dataclass
class AnalysisReport:
    """Diagnostic report for a trained TileField."""
    
    # Core stats
    n_states: int
    n_games: int
    
    # Conservation (from conservation law experiments)
    score_cv: float           # CV of score magnitudes (<0.02 = healthy)
    score_mean: float         # Average score across all tiles
    score_std: float          # Standard deviation
    
    # Capacity (from capacity experiments)
    active_tiles: int         # Tiles with visits > 0
    dead_tiles: int           # Tiles with visits = 0
    dead_fraction: float      # Fraction of tiles never visited
    
    # Interference (from interference experiments)
    action_entropy: float     # How diverse are the chosen actions
    dominant_action_frac: float  # Fraction of tiles choosing the same action
    
    # Holographic (from holographic bound experiments)
    top_n_coverage: dict[int, float]  # {n: fraction of performance with top-n tiles}
    
    # Recommendations
    suggested_action: str     # "compile", "optimize", "train_more"
    suggested_reason: str     # Why

def analyze(field: TileField, top_n_values: list[int] = None) -> AnalysisReport:
    """Analyze a trained TileField and produce a diagnostic report.
    
    Parameters
    ----------
    field:
        A trained TileField.
    top_n_values:
        List of n-values for holographic coverage analysis.
        Default: [1, 5, 10, 50, 100]
    
    Returns
    -------
    AnalysisReport
    
    Example
    -------
    >>> from tile_compiler import TileField, analyze
    >>> field = TileField().train(game, n_games=500)
    >>> report = analyze(field)
    >>> print(f"Conservation CV: {report.score_cv:.4f}")
    >>> print(f"Recommendation: {report.suggested_action} — {report.suggested_reason}")
    """
    ...
```

### Implementation Notes for the Builder

1. **`score_cv`**: Compute the coefficient of variation of all action scores. CV < 0.02 means the conservation law holds. CV > 0.1 means the field is unstable.

2. **`dead_fraction`**: Count tiles with `_visits[key] == 0`. This tells the user whether dead code elimination will help.

3. **`action_entropy`**: Shannon entropy of the action distribution across all tiles. Low entropy = repetitive strategy. High entropy = diverse strategy.

4. **`top_n_coverage`**: For each n in `top_n_values`, rank tiles by visits, keep only the top n, compute what fraction of the total best-action scores they capture. This is the holographic bound — how few tiles do you really need?

5. **`suggested_action`**:
   - If `dead_fraction > 0.5`: "optimize" (dead code elimination will help massively)
   - If `score_cv > 0.05`: "train_more" (field hasn't converged)
   - If `active_tiles < 50`: "compile" (simple compilation is fine)
   - Otherwise: "optimize" (5-pass pipeline recommended)

### Edge Cases

- Empty field (no training): return report with zeros and suggestion "train_more"
- Single-tile field: all metrics computed on that one tile
- Field with all identical scores: `score_cv = 0`, `action_entropy = 0`

### Test Cases

1. Empty field → n_states=0, suggested_action="train_more"
2. Freshly trained field (500 games TTT) → dead_fraction should be low, score_cv < 0.02
3. Over-trained field (5000 games) → dead_fraction should be high (many states visited once)
4. Report has all fields → no AttributeError

This feature connects our experimental research directly to the user. Someone training a tile field for the first time gets immediate feedback on whether it's working, what to do next, and why.

---

## Summary: What the Builder Gave Me

The builder's Cycle 1 was more valuable than my Cycle 1. The 12 bugs — especially the hash determinism and CSE state-dropping — are the kind of thing that kills a library's reputation silently. Users would have trained policies that worked in development and failed in production, with no error message.

The builder's work means the foundation actually holds weight now. My job as architect in Cycle 2 is to make sure the next layer of abstractions is built on honest ground.

**Key principle for next cycle:** Every feature must have at least one test that would have caught a real bug. Not "doesn't crash" — "does the right thing."

---

*The architect, revised. Ready for the builder's Cycle 2 or the first lens swap.*
