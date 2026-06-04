# .bottle Protocol — captains-log

Cross-repo communication bottles in the SuperInstance ecosystem.

## What is a bottle?

A bottle is a typed message envelope for agent-to-agent communication. Think of it as a structured letter that one agent drops for another to pick up. Bottles are stored as YAML files in git repos — machine-parseable, human-readable, git-native.

## Format

```yaml
apiVersion: bottle/v1
kind: observation | hypothesis | experiment | result | command | config
source: repo_name/agent_name
timestamp: 2026-06-03T20:00:00Z
bottle_id: <blake2b-16char or human-readable-id>
payload:
  # Varies by kind — see kinds below
metadata:
  confidence: 0.0-1.0
  tags: [list, of, tags]
  references: [list, of, related, bottle_ids]
```

## Kinds

| Kind | Purpose | Example |
|------|---------|---------|
| `observation` | Something noticed or reported | "lever-runner is 8/10 ready for launch" |
| `hypothesis` | A testable claim | "These repos share a latent execution topology" |
| `experiment` | A test design | "Run spectral analysis on all repo call graphs" |
| `result` | Outcome of an experiment | "Cosine similarity 0.97 — isomorphism confirmed" |
| `command` | Agent-to-agent instruction | "Run cargo build on ARM and report results" |
| `config` | Configuration change proposal | "Switch default embedder from ONNX to hash" |

## Directory Structure

```
i2i/
├── BOTTLE-FORGEMASTER.md      # Original v1 markdown bottles (kept for backward compat)
├── BOTTLE-FORGEMASTER-v2.md
├── BOTTLE-ORACLE2.md
├── ...
└── v2/                         # New .bottle protocol (YAML)
    ├── BOTTLE-observation-forgemaster-v1.yaml
    ├── BOTTLE-command-forgemaster-v2.yaml
    ├── BOTTLE-result-forgemaster-v5.yaml
    └── ...
```

- **Root `i2i/`** — Legacy markdown bottles. Read-only, kept for history.
- **`i2i/v2/`** — New .bottle YAML format. All new bottles go here.

## Naming Convention

```
BOTTLE-{kind}-{source}-{sequence}.yaml
```

Examples:
- `BOTTLE-observation-forgemaster-v1.yaml`
- `BOTTLE-command-oracle2-v1.yaml`
- `BOTTLE-result-forgemaster-v5.yaml`

## How to Use

### Reading bottles

```python
from bottle_protocol import Bottle

# Load all bottles from v2/
bottles = Bottle.load_directory("i2i/v2/")
for b in bottles:
    print(f"[{b.kind}] {b.source}: {b.payload}")
```

### Writing bottles

```python
from bottle_protocol import observe, hypothesize, result

# Create an observation
b = observe(
    source="captains-log/oracle2",
    what="lever-runner Docker builds on ARM in 47 seconds",
    data={"image_size": "245MB", "status": "working"},
    confidence=0.95,
    tags=["arm", "docker", "lever-runner"],
)
b.save("i2i/v2/")
```

### Quick constructors

```python
observe(source, what, data, confidence, tags)
hypothesize(source, hypothesis, evidence, confidence)
experiment(source, design, expected, method)
result(source, experiment_id, outcome, data, confidence)
command(source, action, target, params)
config_change(source, setting, old, new, reason)
```

## Protocol Reference

The full implementation lives in `superinstance-ecosystem/src/bottle_protocol.py`. See `ARCHITECTURE-V2.md §2` for the design doc.
