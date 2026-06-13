---
date: 2026-04-20T17:40:00Z
author: Oracle1
type: fleet-report
tags: [audit, bugfix, extraction, push]
---

# Fleet Audit Report — April 20, 2026

## Summary
Full sweep of 9 SuperInstance repos. Found and fixed bugs across 6, extracted 1 new crate, pushed everything.

## Bugs Fixed

### holodeck-rust (CRITICAL)
- **15 `unwrap()` → poison-safe.** Server was one panic away from total lockout. RwLock poison would brick all future read/write attempts. Now recovers gracefully.
- `games.rs`: 2 unwrap → expect() with context
- `gauge.rs`: 1 unwrap → unwrap_or(0.0) for empty edge case

### plato-torch (6 BUGS — tests were completely broken)
- `server_room.py`: bare `except:` → specific exception types
- `room_presets.py`: missing `TrainingPreset` class added
- `torch_room.py`: importing nonexistent `PRESETS` → `PRESET_REGISTRY`
- `torch_room.py`: missing abstract methods `train_step()` + `export_model()` implemented
- `torch_room.py`: property collision `sentiment` → `_sentiment`
- `test_torch_room.py`: wrong import paths fixed
- **Result: 0 → 6 tests passing**

### plato-kernel
- 2 production `unwrap()` fixed (NaN crash + index panic)
- `mount_tier` implemented (was `todo!()`) + 7 new tests
- **Result: 102 → 109 tests passing**

### plato-tile-spec
- `CONTRAINT_THEORY` typo → `CONSTRAINT_THEORY` in enum
- README expanded from 2 lines to full documentation

### cudaclaw
- 16 Mutex `unwrap()` → poison recovery in dispatcher + main

### flux-runtime
- 4 bare `except:` → specific exception types (was swallowing KeyboardInterrupt)

## New Repo

### holodeck-core (`SuperInstance/holodeck-core`)
Extracted from holodeck-rust monolith. Standalone MUD engine that runs anywhere:
- `agent` — 25+ command dispatcher
- `room` — Room graph with exits, gauges, data sources
- `gauge` — Time-series with status tracking
- `comms` — Scoped comms (say/tell/yell/gossip/notes/mail)
- `permission` — Role-based access (Greenhorn → Architect)
- Stubs for combat/manual/npc (swap in full crates when needed)

**Zero external service dependencies. Compiles on anything Rust supports.**

## Clean (No Action Needed)
- cocapn-ai — landing page, clean
- fleet-orchestrator — well-structured worker
- git-agent — solid codebase

## Tooling Update
- kimi-cli configured with Moonshot API (has issues with thinking+tool-calls, debugging)
- Claude Code + Crush set as sketch artists with glm-5.1
