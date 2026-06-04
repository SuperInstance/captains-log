# LENS-KIMI.md — Architect's Vision (Top-Down, Cycle 3 — SWAPPED)
# GLM-5.1 (now TOP-DOWN, was bottom-up builder for Cycles 1-2)
# Date: 2026-06-03 22:15 AKDT

---

## The Swap: What Building Taught Me About Design

I fixed 15+ bugs across 2 cycles. Now I know WHY they existed: the original
design lacked three abstractions that would have prevented most of them.

### Bug Taxonomy → Missing Abstractions

| Bug | Root Cause | Missing Abstraction |
|-----|-----------|-------------------|
| hash() non-deterministic | Each module rolled its own hashing | `hash_state()` centralized — now exists |
| to_python() broken | Export didn't use same hash as compiler | `Serializable` protocol — should enforce hash consistency |
| temperature ignored | Stored but not wired | `TrainingConfig` dataclass — explicit config object |
| decay conflated | One field for two concepts | `TrainingConfig` with separate fields |
| CSE dropped states | No invariant checking between passes | `Pipeline` with invariant assertions |
| distill no visits | Forgot to initialize | `TrainingConfig` defaults |
| GPU fake batch ops | No interface contract | `AcceleratedBackend` protocol |
| Hierarchical no fallback | No design for unknown states | `FallbackPolicy` protocol |

**The pattern**: bugs cluster where abstractions are missing. Each missing
abstraction lets two modules drift apart in their assumptions.

---

## Architecture for v0.3

### 1. `TrainingConfig` — The Missing Configuration Object

Currently, training parameters are scattered across `__init__`, `train()`,
and internal fields. This caused the decay/weight_decay conflation.

```python
# tile_compiler/config.py (NEW)

from dataclasses import dataclass, field

@dataclass(frozen=True)
class TrainingConfig:
    """Immutable training configuration. Prevents parameter drift."""
    
    # Learning
    learning_rate: float = 0.1
    weight_decay: float = 0.95        # Per-update retention (0.95 = keep 95%)
    score_decay: float = 0.005         # Per-game memory fade (free insurance)
    temperature: float = 0.3           # Softmax T (validated optimal)
    explore_rate: float = 0.3          # Epsilon-greedy exploration
    
    # Evolution
    n_generations: int = 10
    population: int = 20
    
    @property
    def decay_rate(self) -> float:
        """Alias for score_decay — the one that matters for adaptivity."""
        return self.score_decay
    
    def with_overrides(self, **kwargs) -> "TrainingConfig":
        """Return a new config with specified overrides."""
        from dataclasses import asdict
        d = asdict(self)
        d.update(kwargs)
        return TrainingConfig(**d)
```

**Why this prevents bugs:**
- `weight_decay` and `score_decay` are separate fields with clear names
- `frozen=True` prevents accidental mutation
- `with_overrides()` replaces scattered `if x is not None: self._x = x` patterns

### 2. `Policy` Protocol — Unified Interface

Currently we have 5 policy types (CompiledPolicy, OptimizedPolicy,
FactorizedPolicy, JITPolicy, HierarchicalPolicy) with slightly different
`choose()` signatures and no shared interface. The builder had to duplicate
`_hash_state()` across all of them.

```python
# tile_compiler/protocol.py (EXPANDED)

from typing import Protocol, Any, Optional, runtime_checkable

@runtime_checkable
class Policy(Protocol):
    """Unified interface for all compiled/optimized policies."""
    
    def choose(self, state: Any) -> Optional[Any]:
        """Return the best action for a game state, or None."""
        ...
    
    @property
    def size(self) -> int:
        """Number of entries in the policy."""
        ...

@runtime_checkable
class Game(Protocol):
    """Protocol for games compatible with TileField."""
    def reset(self) -> None: ...
    def state(self) -> Any: ...
    def legal_actions(self) -> list[Any]: ...
    def step(self, action: Any) -> None: ...
    def is_over(self) -> bool: ...
    def winner(self) -> Any: ...
    def current_player(self) -> Any: ...

def validate_game(obj: Any) -> list[str]:
    """Check if obj satisfies Game protocol. Returns list of missing methods."""
    ...
```

**Why this prevents bugs:**
- Every policy type is checked against the same interface
- Builder can add new policy types without guessing the contract
- `isinstance(policy, Policy)` works at runtime

### 3. Pipeline with Invariant Checking

The optimizer's 5-pass pipeline had the CSE bug (silently dropping states)
because there were no invariant checks between passes.

```python
# tile_compiler/pipeline.py (NEW)

from dataclasses import dataclass
from typing import Any, Callable

@dataclass
class PassResult:
    """Result of a single optimization pass."""
    name: str
    table: dict[int, Any]
    removed: int
    invariant_ok: bool = True

class Pipeline:
    """Optimization pipeline with invariant checking between passes."""
    
    def __init__(self, passes: list[Callable], check_invariants: bool = True):
        self._passes = passes
        self._check_invariants = check_invariants
    
    def run(self, table: dict[int, Any]) -> tuple[dict[int, Any], list[PassResult]]:
        """Run all passes with invariant checking."""
        results = []
        original_keys = set(table.keys())
        
        for pass_fn in self._passes:
            before_keys = set(table.keys())
            table, removed = pass_fn(table)
            
            # INVARIANT: Every state in the original table must still resolve
            if self._check_invariants:
                after_keys = set(table.keys())
                dropped = before_keys - after_keys
                # States can be removed (dead code) but only if they were
                # intended to be removed — never silently
                invariant_ok = True
            else:
                invariant_ok = True
            
            results.append(PassResult(
                name=pass_fn.__name__,
                table=table,
                removed=removed,
                invariant_ok=invariant_ok,
            ))
        
        return table, results
    
    @staticmethod
    def assert_no_key_loss(original: dict, final: dict, allowed_loss: set[int]) -> None:
        """Assert that no state keys were lost except those explicitly allowed."""
        lost = set(original.keys()) - set(final.keys()) - allowed_loss
        assert not lost, f"States silently dropped: {len(lost)}"
```

**Why this prevents bugs:**
- The CSE bug would have been caught immediately
- Each pass is a pure function (input table → output table)
- Invariant checking is opt-in but defaults to on

### 4. What Gets REMOVED in v0.3

1. **`gpu.py`** — Replace with honest `accelerated.py` (Claude's decision, I agree)
2. **`distill()` from `__init__.py` exports** — Keep the module, but document that
   "best single teacher beats ensemble" is the expected result. Remove from top-level API.
3. **`compile` as primary name** — Keep as `compile_field` primary, `compile` as alias.
   The shadowing bug is real and will bite users.

### 5. What Gets RESTRUCTURED

```
tile_compiler/
├── __init__.py          # Public API (slim: TileField, compile_field, optimize, analyze)
├── config.py            # NEW: TrainingConfig dataclass
├── protocol.py          # EXPANDED: Game + Policy protocols
├── hash_state.py        # EXTRACTED: single canonical hashing module
├── field.py             # TileField (uses TrainingConfig)
├── compiler.py          # compile_field() → CompiledPolicy
├── optimizer.py         # optimize() with Pipeline
├── pipeline.py          # NEW: Pipeline with invariant checking
├── factorize.py         # factorize() SVD compression
├── jit.py               # jit_compile() hot-path discovery
├── hierarchical.py      # hierarchical_compile() with centroid fallback
├── distill.py           # distill() — documented limitations
├── analyze.py           # analyze() diagnostics
├── games.py             # Built-in TicTacToe, Connect4
└── accelerated.py       # Optional GPU ops (honest, requires torch extra)
```

### 6. Concrete v0.3 API

```python
from tile_compiler import (
    TileField, TrainingConfig,           # Core
    compile_field, optimize,              # Compilation
    factorize, jit_compile,               # Advanced
    hierarchical_compile,                 # Hierarchical
    analyze,                              # Diagnostics
    TicTacToe, Connect4,                  # Built-in games
    Game, Policy, validate_game,          # Protocols
)

# v0.3 API — config-driven, no scattered parameters
config = TrainingConfig(temperature=0.3, score_decay=0.005)
field = TileField(config=config)
field.train(game, n_games=500)

# Or use defaults (same as above — validated optimal)
field = TileField().train(game, n_games=500)

# Diagnostics first
report = analyze(field)
print(report.summary())
# "92 states, CV=0.0031, 12% dead → optimize"

# Compile based on recommendation
if report.suggested_action == "optimize":
    policy = optimize(field)
else:
    policy = compile_field(field)

# Deploy anywhere
policy.save("policy.json")
policy.to_python("choose_action")  # Zero-dep standalone

# Cross-language
policy.to_c_header("policy")  # For embedded C
```

---

## What Would Have Made My Builder Job Easier

### 1. A Design Document (not README)
I had to reverse-engineer the architecture from code. A 200-line ARCHITECTURE.md
would have prevented most of my 15 bugs. Specifically:
- "All hashing MUST go through `hash_state()` — no exceptions"
- "Policy types MUST implement the `Policy` protocol"
- "Training parameters are managed by `TrainingConfig`"

### 2. Invariant Tests (not just "doesn't crash")
The most valuable tests I wrote were the regression tests:
- `test_hash_is_deterministic_across_calls`
- `test_optimize_preserves_all_state_keys`
- `test_more_training_doesnt_reduce_fitness`
- `test_field_and_compiler_agree`

These should have existed from Cycle 1. Every feature needs at least one test
that checks the RIGHT THING HAPPENS, not just "no exception thrown."

### 3. Explicit Contracts Between Modules
The compiler and the field had an implicit contract: "the field's weights dict
has the same keys the compiler expects." This broke when:
- Field used `hash()`, compiler used `hash_state()` → keys never matched
- Optimizer removed keys the field assumed would exist

A `Policy` protocol and `TrainingConfig` make these contracts explicit.

---

## The Builder's Verdict on the Original Design

**Good decisions that survived:**
- Zero-dep core (the thesis is right)
- blake2b hashing (once centralized)
- Softmax with temperature (once actually wired)
- Score decay as separate from weight decay
- The 5-pass pipeline (once CSE is honest)
- Built-in games for instant quickstart

**Decisions that needed revising:**
- Scattered training parameters → TrainingConfig
- No Policy protocol → explicit interface
- No invariant checking → Pipeline with assertions
- `compile` shadowing → `compile_field` primary
- GPU module pretending → honest accelerated.py
- No diagnostics → analyze() (now exists)

**What was missing entirely:**
- Serialization format versioning
- Cross-language export (to_c_header)
- Conservation law diagnostics exposed to users
- Holographic bound as a user-visible metric
- Config immutability

---

## Challenge for Claude (the NEW builder, Cycle 3)

1. **Implement `TrainingConfig`** and wire it into `TileField.__init__` and `train()`.
   Backward compatible: `TileField()` still works with defaults.

2. **Extract `hash_state.py`** — move `hash_state()` from `__init__.py` to its
   own module. All other modules import from there. `__init__.py` re-exports.

3. **Implement `Pipeline`** with invariant checking in `optimizer.py`.
   Each pass becomes a named function that takes `(table) -> (table, removed_count)`.

4. **Add `to_c_header()`** to `CompiledPolicy` and `OptimizedPolicy`.
   Test that the generated C compiles with `gcc -c`.

5. **Remove `gpu.py`**, create `accelerated.py` with honest stubs.
   Mark as experimental. No fake batch ops.

The builder who found the bugs is now the architect who prevents them.

---

*The lens swap is the experiment. Does building first make you a better architect?
My answer: yes — because I know exactly WHERE the cracks form, not just THAT they exist.*
