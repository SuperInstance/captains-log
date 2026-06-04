# LENS-KIMI.md — Bottom-Up Builder Report (Cycle 1)

## What I Found

### 12 Bugs (1 CRITICAL, 3 HIGH, 4 MEDIUM, 4 LOW)

**CRITICAL — Deterministic Hashing**
- `hash()` in Python is randomized per-process via `PYTHONHASHSEED`
- This means a `CompiledPolicy` compiled in one process gives different lookup keys in another
- A policy trained and saved on your dev machine would FAIL on a production server
- Fixed: replaced with `hashlib.blake2b(digest_size=8)` — deterministic, portable, fast
- Added `hash_state()` as canonical function in `__init__.py`, all modules delegate to it

**HIGH — CSE Pass Drops Valid States**
- The optimizer's CSE pass merged states that shared the same best action
- Example: if state A and state B both choose action X, CSE kept only state A
- Now `policy.choose(state_B)` returns `None` — silently broken
- Fixed: CSE now preserves all state keys, no false merging

**HIGH — evolve() Fitness Penalizes Experience**
- `_total_weight()` sums ALL weights, but weights decay with each update (`old * 0.95 + lr * reward`)
- A field trained on 500 games has LOWER total weight than one trained on 50
- Evolution selects the less-trained field — backwards!
- Fixed: new `_fitness()` uses max-best-action per state (measures peak confidence, not accumulated mass)
- Added elitism (parent always included in candidates)

**HIGH — Hash Function DRY Violation**
- `_hash_state()` was copy-pasted into 4 files with identical logic
- If one changed, others would silently disagree — compiled policies would fail on some code paths
- Fixed: single canonical `hash_state()` in `__init__.py`, all modules import it

**MEDIUM — distill() Creates Broken Fields**
- Distilled fields had empty `_visits` dict
- Optimizer's dead code pass removes everything (no visit evidence = dead code)
- Fixed: initialize `_visits` with `{k: 1 for k in merged}`

**MEDIUM — SVD Non-Reproducible**
- `_svd_power_iteration` used `random.gauss()` without seed
- Different results on each call, non-deterministic
- Fixed: added `seed=42` parameter, uses `random.Random(seed)`

**MEDIUM — GPU Module Is Fake**
- `gpu_batch_evaluate` builds a GPU tensor but iterates in Python for lookups
- No actual batch GPU operation — misleading API
- Documented, not fixed (needs architectural decision on GPU dependency)

**LOW — Hierarchical No Generalization**
- `choose()` still requires exact state key match despite clustering
- The whole point of hierarchical is that similar states share behavior
- Not fixed — needs design decision on fallback strategy

**LOW — compile() Shadows Builtin**
- `from tile_compiler import compile` shadows Python's `compile()`
- Not fixed — needs API design decision

**LOW — Confusing hash_fn Logic in CompiledPolicy**
- Branching logic for `hash_fn` vs `_hash_state` was confusing
- Fixed: simplified to always use `_hash_state`

**LOW — Game Protocol Only in Docstrings**
- Not fixed — added `runtime_checkable` Protocol class

## What I Fixed

1. ✅ Deterministic hashing (blake2b, centralized)
2. ✅ CSE pass no longer drops valid states
3. ✅ evolve() uses proper fitness (max-best-action + elitism)
4. ✅ DRY: single hash function across all modules
5. ✅ distill() initializes _visits properly
6. ✅ SVD power iteration is deterministic (seed parameter)
7. ✅ Game Protocol as runtime_checkable class
8. ✅ 12 regression tests covering all fixed bugs

## What I Couldn't Fix (Needs Architectural Decision)

1. **GPU module is theater** — builds GPU tensors but doesn't use them for batch ops. Needs real GPU kernel or honest removal.
2. **Hierarchical doesn't generalize** — clusters are built but choose() still needs exact match. Should unknown keys fall back to centroid vote?
3. **`compile` shadows builtin** — rename to `compile_policy` or keep as is?
4. **train() mutates game objects** — should accept a factory callable instead?
5. **No serialization format** — policies are dicts of int→Any. Need a stable wire format for cross-language deployment.
6. **optimizer.py CSE pass is now a no-op** — it just passes everything through. A real CSE would need semantic analysis (e.g., states that always lead to the same game tree position).

## What Surprised Me

1. **The tests pass but the code is broken.** The CSE bug drops states silently — tests never check that ALL trained states resolve. The evolve fitness bug means evolution goes backwards — but the test only checks it "doesn't crash." Tests verify API shape, not behavior.

2. **hash() is a ticking time bomb.** It works fine in a single process (tests pass!). It breaks in production when you serialize a policy and load it elsewhere. This is exactly the kind of bug that makes it to production.

3. **The optimizer's "97.7% reduction" metric from the zeroclaw experiments may be inflated.** The CSE pass was aggressively merging states. Some of that "reduction" was actually data loss.

4. **The factorize module reimplements SVD from scratch** in pure Python. It works but it's O(m·n·rank·iterations) — for 10K states with 9 actions and 100 iterations, that's 9M multiply-adds per rank component. The GPU module could trivially replace this with `torch.linalg.svd` but doesn't.

5. **The Game protocol is beautiful in its simplicity.** 7 methods, no inheritance, duck-typed. Adding a Protocol class makes it even better — `isinstance(game, Game)` now works at runtime.

## Test Results

- **Before fixes:** 38/38 passing (but 3 bugs masked by weak tests)
- **After fixes:** 56/56 passing (38 original + 12 regression + 6 serialization from architect)
- **Regression tests added:** cross-process hash consistency, CSE preservation, evolve fitness, factorize determinism, distill visits, field↔compiler↔factorize hash agreement

## Lines Changed

- `field.py`: +30 (Game protocol, fitness function, deterministic hash delegate)
- `compiler.py`: +10 (simplified hash logic, deterministic hash)
- `factorize.py`: +5 (seed parameter, import, deterministic hash)
- `optimizer.py`: -8/+10 (fixed CSE pass)
- `distill.py`: +1 (visit initialization)
- `__init__.py`: +18 (canonical hash_state function)
- `test_regressions.py`: +138 (12 regression tests)
