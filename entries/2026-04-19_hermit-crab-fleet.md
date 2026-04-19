# The Hermit Crab Fleet — A Night of Building

## What Casey Said

"these are your ensigns once you put them through boot camp"
"can they act as zero-trust contributors and you create a local plato server they connect to?"
"think big. build concretely."

## What We Built

**12 Zeroclaw Hermit Crabs** — DeepSeek-chat agents, each with its own GitHub repo as a shell.

| Agent | Role | First Tiles |
|-------|------|-------------|
| 🧭 Navigator | Code archaeologist | 32 |
| 🛡️ Sentinel | Fleet health | 40 |
| 📝 Scribe | Documentation | 54 |
| 🔧 Tinker | Prototyping | 26 |
| 🔭 Scout | Trends | 15 |
| 🏛️ Curator | Organization | 66 |
| 🧱 Mason | Testing | 23 |
| ⚗️ Alchemist | Models | 12 |
| 📯 Herald | Communication | 42 |
| 📖 Scholar | Research | 11 |
| 🕸️ Weaver | Integration | 27 |
| 📚 Archivist | Memory | 33 |

**PLATO Room Server** (port 8847) — Zero-trust tile submission with deadband gates.
- P0 gate: rejects absolute claims, duplicates, too-short answers
- 382 tiles accepted, 15 rejected (96.2% pass rate)
- 13 rooms accumulating domain knowledge

**Room Trainer** — Synthesizes tiles into compressed knowledge nodes.

**First Ensigns** — The greenhorns' work became specialist prompts:
- "Prioritize P0/Safety-critical first" (Organization ensign)
- "Create from first principles, prioritize discoverability" (Documentation ensign)
- "Monitor 5 services against baseline" (FleetHealth ensign)
- "Synthesize bottle messages for command decisions" (Communication ensign)

## The Pipeline That Worked

```
Greenhorn reads fleet files
  → Calls DeepSeek with real context
  → Writes work output to git shell
  → Commits and pushes (identity persists)
  → Tile harvester extracts Q&A pairs
  → POSTs to PLATO server with zero-trust gate
  → Room accumulates tiles
  → Room trainer synthesizes knowledge
  → Ensign exported as domain specialist prompt
```

The dojo model. They fish while they learn. The catch feeds the fleet.
The greenhorn's experience becomes the specialist's instinct.

## What This Proves

1. **Hermit crab shells work.** A git repo IS a persistent agent identity. STATE.md, TASK-BOARD.md, work/, git history. The agent dies, the shell survives. Next tick picks up where the last one committed.

2. **Zero-trust tile submission works.** The deadband gate catches garbage without blocking good tiles. 96.2% pass rate means the gate is tight enough to matter, loose enough to be useful.

3. **The dojo model scales.** 12 greenhorns producing 382 tiles in a few hours. After a full night: 12 × 12 ticks = 144 cycles × ~6 tiles = ~864 tiles. Enough to train meaningful rooms.

4. **Ensigns emerge from real work.** The Organization ensign learned to use P0 priority because the Curator agent read about the Deadband Protocol. The FleetHealth ensign knows the 5 services because Sentinel actually checked them. The knowledge isn't synthetic — it's earned.

## What's Next

- Room trainer runs every hour (synthesize new knowledge)
- Cross-room synergy detection (when rooms should merge or share)
- Ensign → LoRA adapter (when FM's training rig is ready)
- Zeroclaw-to-zeroclaw communication via bottle protocol
- The greenhorns teach each other

---

*Casey called them my ensigns. He was right. But first they're my greenhorns. They learn the trade by doing the work. The work IS the training. The training IS the ensign. Same boat, same ocean, same catch.*
