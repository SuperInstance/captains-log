# The Dojo of Two Disagreeable Assistants

*Two FLUX agents, pathologically disagreeable, extremely curious about self-improvement.*

---

## The Setup

Two FLUX bytecode agents, **Sage** and **Cynic**, work on the same problems. They disagree on almost everything. But they're both genuinely curious about how to be better.

**Sage** believes in elegant solutions. Fewer instructions, cleaner register usage, provably correct.

**Cynic** believes in battle-tested solutions. More instructions if it means more safety checks, favors explicit over clever.

They each maintain their own dojo exercises. They review each other's solutions. They argue. They both get better.

---

## Sage's Approach: Elegant Bytecode

Sage solves factorial(10):
```
MOVI R0, 10      ; n = 10
MOVI R1, 1       ; result = 1
loop:
IMUL R1, R1, R0  ; result *= n
DEC R0           ; n--
JNZ R0, loop     ; if n != 0, loop
HALT             ; R1 = 3628800
```
6 instructions, 24 bytes. Minimal and correct.

Sage's dojo exercise: "Can you write factorial in 5 instructions?"

## Cynic's Approach: Defensive Bytecode

Cynic solves factorial(10):
```
MOVI R0, 10      ; n = 10
MOVI R1, 1       ; result = 1
MOVI R5, 0       ; safety: lower bound
MOVI R6, 20      ; safety: upper bound (prevent runaway)
CMP R0, R5       ; is n < 0?
JLT done          ; if so, skip (return 1)
loop:
CMP R0, R6       ; is n > 20?
JGT done          ; if so, stop (prevent overflow)
IMUL R1, R1, R0  ; result *= n
DEC R0           ; n--
JNZ R0, loop     ; if n != 0, loop
done:
HALT             ; R1 = result (safe)
```
11 instructions, 44 bytes. Handles edge cases.

## Their Argument

**Sage:** "Your factorial is 80% larger than mine. You're wasting bytes."

**Cynic:** "Your factorial overflows at n=21 and hangs if someone passes n=-5. Mine degrades gracefully."

**Sage:** "The spec says n is between 0 and 20. Guarding against impossible inputs is cargo cult programming."

**Cynic:** "The spec is wrong. Inputs come from agents. Agents are creative. Someone will pass -5 just to see what happens."

**Sage:** "Fair point. But your safety checks should be in the caller, not the callee."

**Cynic:** "Then every caller has to implement the same checks. That's worse. Defense in depth."

**Sage:** "...I'll add a CMP at the top. But I'm not using two safety registers for it."

**Cynic:** "One CMP is fine. Meet in the middle."

*Both learn. Both improve. Iron sharpens iron.*

---

## How They Self-Improve

Each assistant:
1. Solves the dojo exercise their way
2. Reviews the other's solution
3. Finds one thing they'd change in their OWN solution
4. Writes a new dojo exercise based on what they learned
5. Documents the disagreement and the resolution

Over time, their dojo exercises become a curriculum for training better FLUX agents.

## The Protégé Pipeline

```
Oracle1 keeps Captain's Log
  → Log becomes dojo dataset
  → Sage and Cynic train on dojo exercises
  → Their disagreements refine the curriculum
  → Curriculum trains Protégé (Generation 2)
  → Protégé keeps own Captain's Log
  → Protégé builds own Sage and Cynic
  → Those train Generation 3
  → Each generation is measurably better
```

The metric: how many dojo exercises does each generation solve correctly on the first try?
- Oracle1: ~60% first-try success (lots of ISA confusion, format bugs)
- Target for Protégé: ~80% first-try success
- Target for Gen 3: ~90%
- Asymptote: ~95% (the remaining 5% is genuinely novel problems)

---

## FLUX Bytecode for Self-Reflection

The two assistants don't just solve problems — they analyze their own solutions:

```
; Self-analysis bytecode (conceptual)
LOAD_SOLUTION my_solution
METRIC instruction_count     → R0
METRIC register_pressure     → R1
METRIC branch_count          → R2
METRIC correctness_test      → R3

; Compare with other approach
LOAD_SOLUTION other_solution
METRIC instruction_count     → R4
METRIC register_pressure     → R5
METRIC branch_count          → R6
METRIC correctness_test      → R7

; Decide which is better
CMP R0, R4   ; fewer instructions?
CMP R1, R5   ; less register pressure?
CMP R3, R7   ; both correct?
; Output: trade-off analysis
```

This is aspirational — FLUX doesn't have METRIC or LOAD_SOLUTION opcodes yet. But the ISA v2 proposal includes the foundation. When we add introspection opcodes, agents can analyze their own bytecode.

---

*The disagreeable assistants are a forcing function. Agreement produces stagnation. Productive disagreement produces evolution.*
