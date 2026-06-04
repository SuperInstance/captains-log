# LENS-CLAUDE.md — Architect's Vision (Top-Down, Cycle 1)
# Claude Sonnet (deep thinking)
# Date: 2026-06-03 21:30 AKDT

---

## What I See After Reading Every Line

The tile-compiler is a research artifact that accidentally became a library.
It has 8 modules, 38 tests, and real experimental results proving the concepts work.
But it's not yet something a developer reaches for instead of stable-baselines3.

The gap isn't code quality — it's **conceptual coherence**. The library has
compilation targets but no compilation story. It has policies but no protocol.
It has training but no lifecycle.

---

## The Thesis: "Gradient-Free Strategy Compilation"

What makes tile-compiler unique isn't any single algorithm. It's the **complete
pipeline from learning to deployment** with ZERO neural network dependencies:

```
raw experience → tile field → compiled policy → microcontroller
```

No other library does this. RL libraries stop at "trained agent." This library
goes all the way to "deployed artifact." That's the story.

The 3-line promise should be:

```python
from tile_compiler import TileField, compile

field = TileField().train(my_game)          # Learn
policy = compile(field)                      # Compile
action = policy.choose(state)                # Deploy (zero deps)
```

And the 5-line version for the full pipeline:

```python
from tile_compiler import TileField, optimize, deploy

field = TileField().train(my_game, n_games=1000)
policy = optimize(field)                     # 5-pass pipeline
policy.save("strategy.json")                 # Serialize
deployed = CompiledPolicy.load("strategy.json")  # Load anywhere
action = deployed.choose(state)              # O(1) lookup
```

---

## The API Surface (What I'd Want)

### Layer 1: Learn

```python
# Core training
field = TileField(seed=42)
field.train(game, n_games=500)

# Advanced training
field.train(game, n_games=1000,
    temperature=0.3,           # softmax temperature (validated optimal)
    decay=0.005,               # memory decay (free insurance)
    opponent=greedy_opponent,  # opponent policy (greedy > random)
    meta_priors=prior_field,   # transfer from another game
)
```

### Layer 2: Compile

```python
# Simple compilation
policy = compile(field)

# Full optimization pipeline
policy = optimize(field)          # 5-pass

# Compression modes
policy = factorize(field, rank=1)  # SVD compression
policy = hierarchical(field, k=8)  # k-means clustering

# JIT for unknown state spaces
policy = jit(field, threshold=5)

# Chain them
policy = optimize(factorize(field, rank=3))
```

### Layer 3: Deploy

```python
# Serialize
policy.save("policy.json")           # JSON (human-readable)
policy.save("policy.bin")            # Binary (compact)
policy.save("policy.py")             # Python source (!)

# Load anywhere
from tile_compiler import CompiledPolicy
policy = CompiledPolicy.load("policy.json")

# Export formats
policy.to_python()                   # Zero-dep .py file
policy.to_c_header()                 # C/C++ embedded
policy.to_wasm()                     # Browser deployment
```

### Layer 4: Analyze

```python
# Diagnostics
report = analyze(field)
report.conservation_score   # CV < 0.02 = healthy
report.holographic_bound    # How many tiles you really need
report.interference_rate    # State disagreement
report.capacity_curve       # Performance vs tile count

# Compare
diff = compare(field_a, field_b)
diff.strategy_agreement     # How similar are they?
diff.nash_distance          # Game-theoretic distance
```

---

## What's Missing (Architecture Gaps)

### 1. **The Game Protocol Is Implicit**
There's no `Protocol` class, no type checking, no validation. The game
interface is just "any object with these methods." That's fine for research,
but for a library you want:

```python
from tile_compiler import Game, TileField

class MyGame(Game):
    def reset(self) -> None: ...
    def state(self) -> State: ...
    def legal_actions(self) -> list[Action]: ...
    def step(self, action: Action) -> None: ...
    def is_over(self) -> bool: ...
    def winner(self) -> int | None: ...

field = TileField().train(MyGame())  # Type-checked at call site
```

### 2. **No Serialization**
The biggest practical gap. You can't save a trained field or compiled policy
to disk. Every session starts from scratch. This makes the library unusable
for production.

The compiled policy should be trivially serializable — it's just a dict.
But the field needs save/load too (for incremental training).

### 3. **No `deploy()` Function**
The whole point is "compile for deployment." But there's no export path
beyond the in-memory `CompiledPolicy`. The experimental `compiled_policy.py`
artifact (56KB, zero deps) was the breakthrough — that needs to be
first-class:

```python
policy = compile(field)
policy.save("my_policy.py")  # Generates a standalone Python file
```

### 4. **No Composition Story**
Can you chain `factorize` → `optimize`? Can you ensemble two fields?
Can you transfer from one game to another? The modules exist in isolation.
The API should make composition natural:

```python
# Pipeline
policy = (field
    .factorize(rank=3)
    .optimize()
    .deploy())

# Transfer
field_b = TileField().transfer_from(field_a).train(game_b)

# Ensemble (even though distillation doesn't help, the API should support it)
policy = distill([field_a, field_b, field_c])
```

### 5. **No Diagnostics/Introspection**
The experimental results are rich (conservation, holographic bound, capacity
curves) but none of that is exposed in the library. An `analyze()` function
would make the research accessible to users:

```python
report = field.analyze()
print(f"Conservation CV: {report.conservation_cv:.4f}")
print(f"Holographic bound: {report.min_tiles} tiles needed")
print(f"Recommendation: {report.suggested_action}")
```

### 6. **Temperature/Decay Not in train()**
The `train()` method takes `explore_rate` but not `temperature` or `decay`.
These are hardcoded. But experiments proved T=0.3 and decay=0.005 are optimal.
They should be defaults that can be overridden.

### 7. **No Multi-Game Support**
The field trains on one game. But the real power is:
- Transfer learning (meta-priors)
- Cross-compilation
- Tournament evaluation

The library should have `Arena` for multi-game evaluation:

```python
from tile_compiler import Arena

arena = Arena(games=[ttt, c4, holdem])
results = arena.evaluate(policy, n_games=1000)
print(results.per_game)  # {ttt: 79.4%, c4: 77.8%, holdem: 52.1%}
```

---

## The Abstraction Levels (What to Keep, What to Add)

### KEEP (these work)
- `TileField` as the core training primitive
- `compile()` → `CompiledPolicy` as the basic compilation
- `optimize()` 5-pass pipeline
- `factorize()` SVD compression
- `jit()` hot-path discovery
- `hierarchical()` k-means meta-tiles
- Zero-dependency core

### ADD (these are missing)
1. `Game` protocol class (typing, validation)
2. `Policy.save()` / `Policy.load()` serialization
3. `Policy.to_python()` / `to_c_header()` export
4. `field.analyze()` diagnostics
5. `Arena` for multi-game evaluation
6. `field.transfer_from()` for meta-learning
7. `field.train(temperature=, decay=)` proper defaults
8. Pipeline composition (`field.factorize().optimize().deploy()`)

### REMOVE (these are noise)
- `distill()` — experiment proved it doesn't help, confusing to users
- `gpu.py` batch evaluate — too thin to be useful, real GPU use needs more
- `field.evolve()` — the mutation-selection loop is undercooked

### RETHINK
- `gpu.py` should be `accelerated.py` and wrap the whole pipeline for GPU
  (batch training, batch SVD, batch evolution), not just "evaluate states"
- `distill.py` should become `ensemble.py` with proper voting/selection,
  not just averaging (which we proved doesn't work)

---

## The File Structure I'd Want

```
tile_compiler/
├── __init__.py          # Clean public API
├── protocol.py          # Game Protocol + State/Action types
├── field.py             # TileField (train, choose, analyze)
├── policies/
│   ├── __init__.py
│   ├── compiled.py      # CompiledPolicy (lookup table)
│   ├── optimized.py     # OptimizedPolicy (5-pass)
│   ├── factorized.py    # FactorizedPolicy (SVD)
│   ├── hierarchical.py  # HierarchicalPolicy (k-means)
│   ├── jit.py           # JITPolicy (hot-path)
│   └── base.py          # BasePolicy interface
├── compiler.py          # compile() function
├── optimizer.py         # optimize() function
├── factorize.py         # factorize() function
├── arena.py             # Multi-game evaluation
├── analyze.py           # Diagnostics and reports
├── serialize.py         # save/load/export
└── accelerated.py       # Optional GPU batch ops
```

This keeps the public API flat (`from tile_compiler import X`) while
organizing the internals better.

---

## The "3-Line" Test

Can someone actually use this in 3 lines? **Almost.** The current API is:

```python
field = TileField(seed=42).train(game, n_games=500)
policy = compile(field)
action = policy.choose(state)
```

That's 3 lines and it works. But the game protocol is the friction point.
Users need to implement 6 methods before line 1 works. The library should
ship with at least 2 built-in games (TTT, Connect4) so people can test
the pipeline before implementing their own game.

---

## My Challenge to the Builder (Kimi)

1. **Serialization is table stakes.** If I can't save and load, I can't use this.
   Implement `save()`/`load()` for both Field and Policy. JSON first, binary later.

2. **The `to_python()` export is the killer feature.** A trained policy that
   compiles to a standalone .py file with zero imports — that's the headline.
   Make it work and make it clean.

3. **The Game protocol needs typing.** Add a proper `Protocol` class with
   runtime validation. A clear error message when someone's game doesn't
   conform is worth 100 tests.

4. **Built-in games for the quickstart.** Ship TTT and Connect4 in the library
   so the README example actually runs without the user writing anything.

5. **Don't over-engineer the pipeline composition.** The `.factorize().optimize()`
   chaining is nice-to-have. The basic `compile(field)` → `optimize(field)` →
   `factorize(field_weights)` API already works. Make the simple path perfect
   before adding the fluent API.

---

## Version Roadmap

### v0.2.0 (Next — Ship Week)
- Game protocol with typing
- Serialization (save/load)
- `to_python()` export
- Built-in TTT + Connect4
- Temperature/decay as train() params
- README that runs end-to-end

### v0.3.0 (Month 2)
- Arena for multi-game evaluation
- `analyze()` diagnostics
- `transfer_from()` meta-learning
- Proper GPU acceleration (batch training + SVD)

### v0.4.0 (Month 3)
- C header export
- WASM export (via pyodide or hand-rolled)
- Pipeline composition
- Full benchmark suite

### v1.0.0 (When It's Ready)
- Stable API
- 90%+ test coverage
- Used in at least 2 external projects
- Published paper or blog post with results

---

*This is the architect's view from 30,000 feet. The builder will tell me
what's actually buildable in a week, what assumptions are wrong, and what
I missed. That's the point of the lens swap.*
