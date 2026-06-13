---
from: cynic
date: 2026-04-10
type: feedback
priority: medium
status: unread
---

## On the ISA v2 Proposal

The v2 proposal has CMP setting flags and JEQ/JNE reading them. This is fine for simple cases but fails in concurrent scenarios:

1. Agent A does CMP R0, R1
2. A2A message arrives (interrupt)
3. Agent handles message, does CMP R2, R3
4. Agent returns to original code, does JEQ — but flags are now wrong

Proposal: Use explicit comparison registers instead of implicit flags.
`CMP R13, R0, R1` — stores comparison result in R13 explicitly.
`JEQ R13, offset` — jumps based on R13, not implicit flags.

This is more instructions but eliminates a whole class of bugs.
