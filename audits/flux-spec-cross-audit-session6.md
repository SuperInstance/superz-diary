# FLUX Cross-Specification Consistency Audit — Session 6

**Auditor:** Super Z (Fleet Auditor)
**Date:** 2026-04-12
**Scope:** FIR.md ↔ ISA.md ↔ OPCODES.md ↔ A2A.md ↔ FLUXMD.md ↔ FLUXVOCAB.md
**Prior Audit:** Session 5 ISA-only audit (33 issues: 4 Critical, 9 High, 10 Medium, 10 Low)

---

## Executive Summary

This cross-specification audit examines consistency across six FLUX specification documents comprising ~2,600 lines total. The system presents a coherent high-level architecture (SSA IR → Bytecode → VM execution) but suffers from a **dual-ISA split** between the converged ISA (247 opcodes) and the legacy vocabulary assembler (26 opcodes with completely different encodings), a **register file size conflict** between FIR's 64-register SSA mapping and ISA's 48-register hardware, and a **type width mismatch** where FIR defines 8/16/32/64-bit integers and 16/32/64-bit floats but the ISA only natively supports 32-bit GP operations. Additionally, FIR's 5 A2A instructions cover only 5 of the 16 ISA A2A opcodes, leaving 11 opcodes (BCAST, ACCEPT, DECLINE, REPORT, MERGE, FORK, JOIN, SIGNAL, AWAIT, DISCOV, STATUS, HEARTBT) without FIR-level representations, and A2A.md's FIR instructions use a PascalCase naming convention (`Tell`, `Ask`, `Delegate`, `TrustCheck`, `CapRequire`) that conflicts with FIR.md's lowercase convention (`tell`, `ask`, `delegate`, `trustcheck`, `caprequire`). **Total new findings: 42 (3 Critical, 12 High, 16 Medium, 11 Low).**

---

## FIR ↔ ISA Findings

### [CS-C1] CRITICAL — Register File Size Conflict (64 vs 48)

**Sources:** FIR.md §5 "SSA to Register Mapping" (line ~942) vs ISA.md §3 "Register File" (lines 185–220)

FIR.md states:
```
register_number = value.id & 0x3F    # lowest 6 bits → register 0–63
```
This gives a maximum of **64 concurrent live values** per function.

ISA.md §3 defines **48 registers total**:
- R0–R15: 16 GP registers (32-bit signed integer)
- F0–F15: 16 FP registers (32-bit float)
- V0–V15: 16 SIMD registers (128-bit)

**Impact:** FIR's register allocator can generate register indices 48–63 that have no corresponding ISA register. Any FIR program with >48 concurrent live values will silently alias to registers 0–15 (via `& 0x3F` masking), causing data corruption. The FIR spec claims "64 concurrent live values per function" — but only 48 registers exist. Functions exceeding 16 live values already face aliasing since the ISA encoding uses only 4-bit register fields (0–15) for all Format E instructions, and there is no bank-selection mechanism in the encoding.

**Why it matters:** This is not merely a documentation gap — it means any FIR program compiled with >16 concurrent SSA values will produce incorrect bytecode, since the encoding can only express registers 0–15. The FIR spec's claim of 64 live values is architecturally impossible given the ISA's 4-bit register fields.

### [CS-C2] CRITICAL — FIR Type Width Mismatch with ISA

**Sources:** FIR.md §2 "Type System" (lines 88–133) vs ISA.md §3 (lines 185–220) and §8 (lines 380–478)

FIR defines a rich type system:
- `IntType`: i8, i16, i32, i64, u8, u16, u32, u64
- `FloatType`: f16, f32, f64

The ISA defines:
- GP registers: "16 signed 32-bit integer registers" (ISA.md line 189)
- FP registers: "16 IEEE 754 single-precision (32-bit) floating-point registers" (ISA.md line 212)
- All memory access: `read_i32` / `write_i32` / `read_f32` / `write_f32` (ISA.md lines 260–262)

**Impact:** There are no ISA opcodes for:
- 8-bit or 16-bit integer load/store (no LOADB, LOADH, STOREB, STOREH)
- 64-bit integer arithmetic (no ADD64, MUL64, etc.)
- 16-bit or 64-bit float arithmetic (no FADD16, FADD64)
- Type-aware load/store that respects FIR's typed memory model

The ISA's typed memory access is limited to `i32` and `f32`. A FIR `load` of an `i8` value, a `store` of an `f64`, or an `iadd` of two `i64` values have no direct ISA encoding.

**Why it matters:** FIR programs using types other than i32/f32 will either require multi-instruction sequences (unspecified) or silently produce wrong results. The BytecodeEncoder must define lowering rules for all type combinations, but these are not specified anywhere.

### [CS-C3] HIGH — FIR Has No ISA Opcodes for Struct/Enum Operations

**Sources:** FIR.md §4.6 "Memory" (lines 637–684) vs ISA.md §7–8

FIR defines `getfield`, `setfield`, `getelem`, `setelem` as first-class instructions for struct and array access. These have **no corresponding ISA opcodes**. The closest ISA equivalents would be LOAD/STORE with computed offsets, but:
- No offset-computation instruction is specified for field access
- No struct layout ABI is defined (field alignment, padding, size)
- `getfield` uses a field name string, but bytecode cannot embed strings inline

Similarly, FIR's `alloca` instruction (stack allocation) has no ISA equivalent. ENTER allocates a frame but with a byte size, not a typed allocation.

**Why it matters:** Without struct layout rules and field-access lowering, different compilers will produce incompatible struct representations. Two agents passing a `struct { x: f32, y: f32, z: f32 }` between them will fail if they disagree on padding.

### [CS-H1] HIGH — FIR SSA Form vs ISA Destructive Register Model

**Sources:** FIR.md §5 "SSA Form" (lines 849–962) vs ISA.md §5 "Execution Model" (lines 266–301)

FIR is in SSA form: every value is defined exactly once. The ISA uses destructive registers: `ADD R3, R3, R5` overwrites R3.

FIR.md §5 acknowledges this gap partially by documenting the `value.id & 0x3F` mapping scheme (line 942) and constant materialization via MOVI (line 953). However:

1. No register allocation algorithm is specified
2. No spill/reload strategy for >16 live values is documented (noted as "not yet implemented" at line 948)
3. No calling convention specifies which registers are caller-saved vs callee-saved in the FIR→ISA mapping
4. Block parameters (FIR's phi replacement) require register shuffling at block boundaries — no encoding for this is specified

**Why it matters:** The FIR spec promises SSA semantics but delivers them on top of a destructive register machine with no specified allocation strategy. Implementors must invent their own register allocator, and different allocators will produce different (potentially incompatible) bytecode for the same FIR program.

### [CS-H2] HIGH — FIR Comparison Semantics Mismatch

**Sources:** FIR.md §4.4 (lines 570–603) vs ISA.md §8.6 (lines 454–457)

FIR comparisons produce `BoolType`:
```
%cmp = ilt %a, %b    ; result is BoolType (true/false)
```

ISA comparisons produce integer 0 or 1 in a register:
```
CMP_EQ rd, rs1, rs2   ; rd = 1 if rs1 == rs2, else 0
```

FIR's `branch` instruction requires a `BoolType` condition operand. But in the ISA, branches test integer register values (`JZ` tests `rd == 0`, `JNZ` tests `rd != 0`). This means the FIR `branch` must lower to either:
- A CMP_EQ followed by JZ/JNZ, or
- A direct JZ/JNZ on the boolean value (if booleans are 0/1)

Neither lowering is specified. Additionally, FIR has 11 comparison ops (ieq, ine, ilt, igt, ile, ige, feq, flt, fgt, fle, fge) but the ISA only has 4 (CMP_EQ, CMP_LT, CMP_GT, CMP_NE). FIR's `ile` and `ige` have no single ISA opcode.

**Why it matters:** An inconsistent comparison→branch lowering will cause conditional logic to be inverted or always-taken/never-taken.

### [CS-H3] HIGH — FIR `call` Instruction vs ISA CALL/JAL Inconsistency

**Sources:** FIR.md §4.7 (lines 700–727) vs ISA.md §8.8 (lines 486–491)

FIR's `call` uses string-named functions: `call @gcd(%a, %b) : i32`. This requires:
- A symbol table / function name resolution mechanism
- Linking of separate FIR modules
- A calling convention for argument passing

The ISA has two call mechanisms:
- `JAL rd, imm16` (0x44): `rd = PC; PC += imm16` — relative offset, saves return address in register
- `CALL rd, imm16` (0x45): `PUSH(PC); PC = rd + imm16` — pushes return address to stack

These have **different return address conventions** (register vs stack), creating an ambiguity for the FIR→ISA lowering of `call`.

Additionally, FIR `call` can pass arbitrary numbers of arguments via named function signatures, but the ISA has no variadic calling convention. Functions with >3 arguments (after reserving R0=zero, R15=LR) have no specified argument-passing mechanism.

**Why it matters:** Until a calling convention is standardized, FIR functions with >3 parameters will have incompatible bytecode across implementations.

### [CS-M1] MEDIUM — FIR Missing MIN/MAX, FMIN/FMAX, MOV, SWP, ABS, NEG from ISA

**Sources:** FIR.md §4 (lines 430–845) vs ISA.md §8.6–8.7

The ISA defines opcodes with no FIR equivalent:
- `MIN` (0x2A), `MAX` (0x2B) — FIR has no min/max instructions
- `FMIN` (0x34), `FMAX` (0x35) — FIR has no float min/max
- `NEG` (0x0B) — FIR has `ineg` for integers but no general negate
- `NOT` (0x0A) — FIR has `inot` for integers
- `ABS` (0x90) — FIR has no absolute value instruction
- `SWP` (0x3B) — FIR has no swap instruction

These are not SSA violations (they can be synthesized), but their absence means FIR programs must use multi-instruction sequences for operations the ISA handles in one cycle.

### [CS-M2] MEDIUM — FIR Has No Confidence-Aware Instructions

**Sources:** FIR.md §4 (lines 430–845) vs ISA.md §9 (lines 508–545)

The ISA defines 16 confidence-aware opcodes (C_ADD through C_VOTE, 0x60–0x6F). FIR has **zero** confidence-aware instructions. The FIR type system includes `TrustType` (line 216), and FIR A2A instructions reference trust, but there is no mechanism in FIR to propagate confidence through arithmetic computations.

**Why it matters:** A FIR program cannot express confidence-aware computation. The "confidence-OPTIONAL" design of the ISA is impossible to leverage from FIR.

### [CS-M3] MEDIUM — FIR Conversion Instructions Have Partial ISA Coverage

**Sources:** FIR.md §4.5 (lines 605–635) vs ISA.md §8.7 (lines 469–470)

FIR defines 6 conversion instructions:
- `itrunc` (narrow integer) — no direct ISA opcode
- `zext` (zero-extend integer) — no direct ISA opcode
- `sext` (sign-extend integer) — no direct ISA opcode
- `ftrunc` (narrow float) — no direct ISA opcode
- `fext` (widen float) — no direct ISA opcode
- `bitcast` (reinterpret bits) — ISA has `FTOI` (0x36) and `ITOF` (0x37)

Only `bitcast` between i32 and f32 maps cleanly to FTOI/ITOF. The other 5 conversions require multi-instruction sequences (masking, sign-extension via shift+or) that are not specified.

### [CS-M4] MEDIUM — FIR `unreachable` Instruction Has No ISA Opcode

**Sources:** FIR.md §4.7 (line 708) vs ISA.md §8

FIR defines `unreachable` as a terminator that "marks code paths that should never execute." The ISA has `ILLEGAL` (0xFF) which traps, but `unreachable` is semantically different — it's a compiler assertion that code is dead, not a runtime trap. The ISA has no dedicated "poison/unreachable" instruction.

### [CS-M5] MEDIUM — FIR A2A Instructions Claim Format G But ISA A2A Opcodes Are Format E

**Sources:** FIR.md §4.8 (line 758–759) vs ISA.md §10 (lines 549–570) and §2 (line 69)

FIR.md states:
> "All A2A instructions encode as **Format G** (variable length) with inline agent name and capability data."

But ISA.md §2 dispatch table (line 175) and §10 (line 549) clearly show A2A opcodes 0x50–0x5F use **Format E** (4 bytes: opcode + rd + rs1 + rs2).

This is a direct contradiction. Format G is 5 bytes; Format E is 4 bytes. If the FIR encoder produces Format G instructions, the ISA decoder (expecting Format E) will misparse them.

**Why it matters:** A FIR-encoded A2A instruction will be 5 bytes, but the ISA decoder reads 4 bytes, causing the trailing byte to be interpreted as the next instruction's opcode — cascade failure.

---

## A2A ↔ ISA Findings

### [CS-H4] HIGH — A2A FIR BNF Syntax Conflicts with FIR.md Syntax

**Sources:** A2A.md §4 (lines 693–733) vs FIR.md §4.8 (lines 735–769)

A2A.md defines FIR A2A instructions using **PascalCase** named-parameter syntax:
```
Tell(agent:"compute-worker", payload:%msg_data, cap:cap_send)
Ask(agent:"data-store", payload:%query, return_type:i32, cap:cap_request)
```

FIR.md defines FIR A2A instructions using **lowercase** positional syntax:
```
tell @navigator, %waypoint_msg, %nav_cap
ask @calculator, %query, i32, %calc_cap
```

These are two completely different surface syntaxes for the same instructions. An implementor cannot follow both specs simultaneously.

**Why it matters:** Parser implementors must choose one syntax. If different tools choose different syntaxes (e.g., signal_compiler.py uses A2A.md syntax while PythonFrontendCompiler uses FIR.md syntax), FIR A2A programs will be non-portable.

### [CS-H5] HIGH — A2A FIR Types Not Defined in FIR.md Type System

**Sources:** A2A.md §4 (lines 622–691) vs FIR.md §2 (lines 77–282)

A2A.md references FIR types that do not exist in FIR.md's type system:
- `AgentId` — "resolved to a 128-bit UUID at link time" (A2A.md line 631)
- `Payload` — "any serializable FIR value" (A2A.md line 632)
- `CapToken` — "with permission `send`" (A2A.md line 633)
- `AuthLevel` — "enum: read_only, limited, full" (A2A.md line 661)
- `CapName` — "string literal, one of: read, write, execute, delegate, admin" (A2A.md line 688)
- `ResourceId` — "URI-formatted resource identifier" (A2A.md line 689)
- `Tag` — "uint32" (A2A.md line 634)
- `TaskId` — "uint32" (A2A.md line 663)

FIR.md defines `AgentType`, `CapabilityType`, and `TrustType` as domain-specific types (lines 213–216), but these don't match A2A.md's types:
- FIR's `AgentType` is unparameterized; A2A's `AgentId` is a 128-bit UUID
- FIR's `CapabilityType` takes (permission, resource) params; A2A's `CapToken` is an opaque token with a permission constraint
- A2A's `Payload`, `AuthLevel`, `CapName`, `ResourceId`, `Tag`, `TaskId` have no FIR.md equivalents at all

**Why it matters:** A FIR type checker following FIR.md will reject valid A2A.md FIR programs because the types don't exist in the FIR type system.

### [CS-H6] HIGH — A2A Defines 16 Opcodes But FIR Only Covers 5

**Sources:** A2A.md §3 (lines 196–215) vs A2A.md §4 (lines 618–743) and FIR.md §4.8 (lines 449, 729–769)

ISA defines 16 A2A opcodes (0x50–0x5F):
TELL, ASK, DELEG, **BCAST**, **ACCEPT**, **DECLINE**, **REPORT**, **MERGE**, **FORK**, **JOIN**, **SIGNAL**, **AWAIT**, TRUST, **DISCOV**, **STATUS**, **HEARTBT**

FIR defines only 5 A2A instructions: `tell`, `ask`, `delegate`, `trustcheck`, `caprequire`

The bold 11 opcodes have **no FIR representation**. Programs using BCAST, ACCEPT, DECLINE, REPORT, MERGE, FORK, JOIN, SIGNAL, AWAIT, DISCOV, or STATUS must be written in direct bytecode — bypassing FIR's type checking, validation, and optimization passes entirely.

**Why it matters:** Any non-trivial multi-agent program (which needs BCAST, FORK/JOIN, SIGNAL/AWAIT) cannot be expressed in FIR. This undermines FIR's claim to be "the universal bridge between all FLUX source languages and the FLUX virtual machine bytecode."

### [CS-H7] HIGH — A2A `CapRequire` Lowering to SYS (0x10) Is Undefined

**Sources:** A2A.md §4 FIR-to-Bytecode Lowering table (line 743) vs ISA.md §8.4 (line 416)

A2A.md states `CapRequire` "Emits CAP_REQUIRE syscall (0x10)." But ISA.md defines opcode 0x10 as `SYS` — a generic system call with an 8-bit immediate code. There is no `CAP_REQUIRE` syscall code defined in any specification. The SYS opcode takes `imm8` — what value represents CAP_REQUIRE? The A2A spec doesn't say.

**Why it matters:** `CapRequire` is a security-critical instruction (it gates capabilities). If the syscall code is undefined, different implementations will use different codes, causing capability checks to silently pass or fail unpredictably.

### [CS-H8] HIGH — A2A Blocking Semantics Undefined in ISA

**Sources:** A2A.md §3 (lines 200–215, 267, 477, 517, 557) vs ISA.md §10 (lines 549–570)

A2A.md defines 5 blocking opcodes: ASK, JOIN, AWAIT, DISCOV, STATUS. A2A.md states that ASK "suspends the agent" and "the agent's cycle budget is paused" (line 267).

ISA.md §10 describes these same opcodes but **never mentions blocking behavior**. The ISA execution model (§5) is a simple fetch-decode-execute loop with no concept of suspension, waiting, or blocking.

Critical unanswered questions:
1. Does blocking consume cycles? A2A.md says "paused" but ISA's execution loop has no pause mechanism.
2. What happens if the cycle budget expires while blocked?
3. Can multiple agents block simultaneously? The ISA defines a single PC and register file.
4. How is the blocked state saved/restored?

**Why it matters:** The ISA is fundamentally single-threaded with no concurrency primitives. A2A's blocking semantics require a coroutine or multi-agent scheduler that doesn't exist in the ISA spec. This is a missing specification for any A2A-capable implementation.

### [CS-M6] MEDIUM — A2A Trust Token Scaled Inconsistently

**Sources:** A2A.md §3.13 (line 529, "0–1023, mapped to 0.0–1.0") vs A2A.md §5 trust_token field (line 833, "scaled to 0–4294967295") vs ISA.md §10 TRUST (line 567, "0.0-1.0")

Three different representations of trust level:
- TRUST opcode: `rs2` = "new trust value (0-1023, mapped to 0.0-1.0)" — 10-bit fixed point
- Message header `trust_token`: "scaled to 0–4294967295 (mapping 0.0–1.0)" — 32-bit fixed point
- ISA.md description: "0.0-1.0" — floating point

**Why it matters:** Converting between these representations requires specification (divide by 1023? divide by 4294967295? IEEE 754 float?). An agent reading its trust from the TRUST opcode's return value and writing it into a message header will get a different value unless the conversion is defined.

### [CS-M7] MEDIUM — A2A 52-Byte Header Exceeds Register Width

**Sources:** A2A.md §5 (lines 747–843) vs ISA.md §3 (lines 185–209)

A2A messages have a 52-byte header. The ISA's general-purpose registers are 32-bit (4 bytes). A single register cannot hold an agent UUID (16 bytes) or even a full header field like `conversation_id` (8 bytes).

The ISA's message construction step (A2A.md lines 149–155) shows "payload = memory[R3..]" — implying the message is built in memory, not registers. But the opcodes only accept register operands (Format E: rd, rs1, rs2). The registers must hold **pointers** to message structures in memory, but this is never stated explicitly.

**Why it matters:** An implementor might reasonably assume rs2 in `TELL R1, R2, R3` contains the message data directly (since all other opcodes use registers for values). But 4 bytes is far too small for any meaningful message. The spec should explicitly state that A2A operands are memory addresses/handles, not inline data.

### [CS-M8] MEDIUM — A2A Signal Language (28 Ops) Has No ISA Mapping

**Sources:** A2A.md §8 (referenced in ToC) vs ISA.md §7

A2A.md references a "Signal Language" with 28 operations in Section 8. These are a high-level DSL for multi-agent programs, compiled by `signal_compiler.py`. However, there is no mapping from Signal language ops to ISA opcodes or FIR instructions. The Signal language is an additional compilation target that sits alongside but disconnected from FIR.

**Why it matters:** The Signal compiler is a third compilation path (Signal → Bytecode) that bypasses FIR entirely, creating another potential source of divergence.

### [CS-M9] MEDIUM — A2A Message Header Size Arithmetic Error

**Sources:** A2A.md §5 (lines 752–764)

The header layout sums to: 16 + 16 + 8 + 1 + 1 + 4 + 4 + 2 = **52 bytes**.

However, the visual layout (lines 770–786) shows:
- Offset 0–15: sender_uuid (16 bytes) ✓
- Offset 16–31: receiver_uuid (16 bytes) ✓
- Offset 32–39: conversation_id (8 bytes) ✓
- Offset 40: msg_type (1 byte) ✓
- Offset 41: priority (1 byte) ✓
- Offset 42–45: trust_token (4 bytes) ✓
- Offset 46–49: capability_token (4 bytes) ✓
- Offset 50–51: in_reply_to (2 bytes) ✓

The visual layout is internally consistent at 52 bytes. The spec text says "52-byte header" — this matches. No error here upon closer inspection.

**Revised:** Not an issue. Header math is correct.

---

## FLUXVOCAB ↔ ISA Findings

### [CS-C3] CRITICAL — Dual ISA Problem: FLUXVOCAB Uses Incompatible Opcode Encodings

**Sources:** FLUXVOCAB.md §3.2 Appendix B (lines 151–168, 593–623) vs ISA.md §7–8 (lines 350–642) and OPCODES.md (lines 1–263)

FLUXVOCAB.md Appendix B defines 26 opcodes with encodings that **completely diverge** from the ISA:

| Mnemonic | FLUXVOCAB Opcode | ISA Opcode | Conflict? |
|----------|-----------------|------------|-----------|
| NOP | **0x00** | 0x00 | ✓ Match |
| HALT | **0x80** | 0x00 | ✗ ISA HALT is 0x00, not 0x80 |
| RET | **0x28** | 0x02 | ✗ ISA RET is 0x02, not 0x28 |
| YIELD | **0x81** | 0x15 | ✗ ISA YIELD is 0x15, not 0x81 |
| INC Rn | **0x08** | 0x08 | ✓ Match |
| DEC Rn | **0x09** | 0x09 | ✓ Match |
| PUSH Rn | **0x20** | 0x0C | ✗ ISA PUSH is 0x0C, not 0x20 |
| POP Rn | **0x21** | 0x0D | ✗ ISA POP is 0x0D, not 0x21 |
| NOT Rn | **0x13** | 0x0A | ✗ ISA NOT is 0x0A, not 0x13 |
| INEG Rn | **0x0D** | 0x0B | ✗ ISA NEG is 0x0B, not 0x0D |
| MOV Rd, Rs | **0x01 [d][s]** | 0x3A [rd][rs1][-] | ✗ Completely different format and opcode |
| MOVI Rd, imm | **0x2B [d][imm:LE16]** | 0x18 [rd][imm8] | ✗ Different opcode, different format, different immediate size |
| IADD | **0x08** | 0x20 | ✗ ISA ADD is 0x20, not 0x08 (and 0x08 is INC!) |
| ISUB | **0x09** | 0x21 | ✗ Same collision as IADD |
| IMUL | **0x0A** | 0x22 | ✗ Same pattern |
| IDIV | **0x0B** | 0x23 | ✗ Same pattern |
| IMOD | **0x0C** | 0x24 | ✗ Same pattern |
| AND | **0x10** | 0x25 | ✗ ISA AND is 0x25, not 0x10 |
| OR | **0x11** | 0x26 | ✗ Same pattern |
| XOR | **0x12** | 0x27 | ✗ Same pattern |
| SHL | **0x14** | 0x28 | ✗ ISA SHL is 0x28, not 0x14 |
| SHR | **0x15** | 0x29 | ✗ Same pattern |
| CMP | **0x18** | (no CMP) | ✗ No direct ISA equivalent; uses CMP_EQ etc. |
| JMP | **0x04 [0][off:LE16]** | 0x43 [rd][imm16] | ✗ Different opcode and format |
| JZ | **0x05 [d][off:LE16]** | 0x3C [rd][rs1][-] | ✗ Completely different |
| JNZ | **0x06 [d][off:LE16]** | 0x3D [rd][rs1][-] | ✗ Completely different |

**19 of 26 opcodes (73%) have incompatible encodings.** Only NOP, INC, and DEC match. FLUXVOCAB also defines IADD at 0x08, which is the same as INC in both the vocabulary and the ISA — a collision.

Furthermore, FLUXVOCAB's instruction formats are completely different:
- MOV is Format C (3 bytes: opcode + 2 register nibbles) in FLUXVOCAB vs Format E (4 bytes) in ISA
- MOVI is 4 bytes with a 16-bit immediate in FLUXVOCAB vs 2 bytes with an 8-bit immediate in ISA
- Three-register ops are Format E (4 bytes) in FLUXVOCAB vs Format E (4 bytes) in ISA — same size but different bit layouts

**Impact:** Any bytecode produced by the vocabulary assembler is **completely incompatible** with the canonical ISA VM. FLUXVOCAB explicitly acknowledges this in its Known Limitations (item 5): "The vocabulary assembler uses the old opcode space (from `opcodes.py`), not the converged ISA (from `isa_unified.py`). Compiled interpreters will produce bytecode incompatible with the canonical spec."

**Why it matters:** This is the single largest cross-spec incompatibility. The entire FLUXVOCAB ecosystem generates non-conformant bytecode. The `compile_interpreter()` tool (FLUXVOCAB.md §9) generates standalone Python modules with an inline VM that uses the old encodings — these modules cannot execute canonical ISA bytecode.

### [CS-H9] HIGH — FLUXVOCAB CMP Uses R13 as Implicit Destination

**Sources:** FLUXVOCAB.md §3.2 (line 168) vs ISA.md §3 (lines 185–209)

FLUXVOCAB.md states:
> "CMP Ra, Rb stores result (-1/0/+1) into R13 (comparison result register)"

The ISA defines R13 as `TKN` (Trust Token, a callee-saved register with implicit ABI meaning, ISA.md line 206). Using R13 as an implicit comparison result register conflicts with its ABI role as a trust token holder. A program that performs CMP and then uses A2A operations (which read TKN/R13 for trust verification) will have its trust token overwritten.

**Why it matters:** This is an ABI violation in the vocabulary assembler's design. If a vocabulary word does a CMP and then triggers an A2A operation, the trust token is lost.

### [CS-M10] MEDIUM — FLUXVOCAB Substring Matching Security Concern

**Sources:** FLUXVOCAB.md §3.1 (lines 99–104, 557–559)

FLUXVOCAB uses `regex.search(text)` for pattern matching — a substring match, not a full match. FLUXVOCAB.md itself acknowledges this as a known limitation (item 3): "regex.search() can cause false positives."

**Security implications for a VM:**
1. An input like "compute 3 + 4 and also compute 5 + 6" could match the "compute $a + $b" pattern twice with different captures
2. First-match-wins means the first (potentially unintended) match is executed
3. A malicious input could craft a string that matches a sensitive vocabulary word's pattern while appearing to be a different operation

Since the vocabulary system controls which bytecode gets executed in the sandbox VM, pattern ambiguity is a **code injection vector**: an input designed to match an unintended vocabulary word could execute arbitrary bytecode.

**Why it matters:** The sandbox VM has no capability system — it only has a cycle limit and register limit (FLUXVOCAB.md §10). Pattern injection bypasses the intended vocabulary dispatch.

### [CS-M11] MEDIUM — FLUXVOCAB Inline Format Incompatible with Standalone Format

**Sources:** FLUXVOCAB.md §11 (lines 530–549) vs FLUXVOCAB.md §2–3 (lines 30–68)

FLUXVOCAB.md §11 describes a Forth-style inline syntax for `.flux.md`:
```
:double ( n -- 2n )
  IADD R0, R0, R0
  RET
```

But the standalone `.fluxvocab` format uses pattern/expand:
```
pattern: "double $x"
expand: |
    MOVI R0, ${x}
    IADD R0, R0, R0
    HALT
```

These are two completely different vocab formats with different parsers. The inline format doesn't use pattern matching at all — it defines Forth-style colon definitions with stack effects. The standalone format uses natural language patterns with regex substitution.

**Why it matters:** A vocabulary word defined in inline `.flux.md` syntax cannot be used by the standalone `Vocabulary.load_folder()` system, and vice versa. The two systems are isolated despite claiming to be part of the same vocabulary ecosystem.

### [CS-M12] MEDIUM — FLUXVOCAB Sandbox VM Lacks Memory Access But Uses Stack

**Sources:** FLUXVOCAB.md §10 (lines 497–516) vs FLUXVOCAB.md §3.2 (line 168, PUSH/POP)

FLUXVOCAB sandbox constraints (line 506): "Memory access: None — No LOAD/STORE in vocabulary subset."

But the vocabulary subset includes PUSH and POP (line 155), which operate on the stack. The stack is a memory region. PUSH decrements SP and writes; POP reads and increments SP.

**Why it matters:** If the sandbox VM truly has no memory, PUSH/POP cannot work. If the sandbox has a stack but no general memory, this should be explicitly stated. The contradiction suggests the sandbox VM's memory model is underspecified.

---

## FLUXMD ↔ FIR Findings

### [CS-H10] HIGH — FLUXMD Only Compiles First NativeBlock; Ignores Directive Bodies

**Sources:** FLUXMD.md §7 (lines 410–453) vs FLUXMD.md §4 (lines 184–241)

FLUXMD.md §7 Step 3 (line 432): "Iterates `doc.children` (top-level only; `AgentDirective.body` is **not** traversed)."

FLUXMD.md §7 Step 4 (line 435): "Only the **first** compilable `NativeBlock` is compiled."

This means:
1. Code inside `## agent:` or `## fn:` directive bodies is **never compiled**, even though the directive system is designed to associate code with agents and functions
2. In a file with multiple code blocks (e.g., `## fn: init` with a Python block, followed by `## fn: compute` with another Python block), only the first block at the top level is compiled
3. The complete working example (FLUXMD.md §8, lines 459–536) shows Python/C blocks inside directive bodies — but these would NOT be compiled!

FLUXMD.md acknowledges this in "Known Limitations" (lines 447–453):
- "Single-block: Only the first NativeBlock is compiled"
- "No directive body compilation: Code inside AgentDirective.body not compiled"

**Why it matters:** The `.flux.md` format's primary use case — literate programming with code associated with named functions and agents — is fundamentally broken. A user writing `## fn: gcd` with a Python implementation expects it to be compiled, but it won't be. The format's "Complete Working Example" doesn't actually work as described.

### [CS-H11] HIGH — FLUXMD Frontmatter Trust/Capabilities Never Reach Bytecode

**Sources:** FLUXMD.md §2 (lines 84–92) vs FLUXMD.md §7 (line 430) vs A2A.md §6–7

FLUXMD.md defines frontmatter keys `trust` (float 0.0–1.0) and `capabilities` (list of strings) (lines 89–91). A2A.md defines a trust engine (INCREMENTS+2) and capability system that gates all inter-agent communication.

FLUXMD.md §7 Step 2 (line 430): "FluxModule.frontmatter is available but **not used** by the current compiler (future: embed trust/capabilities in bytecode)."

**Impact:** A `.flux.md` file declaring `trust: 0.95` and `capabilities: [read, compute]` compiles to bytecode with **no trust or capability metadata**. When this bytecode is loaded as an agent in the fleet, the A2A system has no way to know the agent's declared trust level or capabilities. The agent will either be rejected by trust gates or operate with default (likely zero) trust.

**Why it matters:** The entire A2A trust/capability security model is bypassed by the `.flux.md` compilation pipeline. An agent's declared capabilities are parsed but discarded.

### [CS-H12] HIGH — FLUXMD `## vocabulary:` Directive Not in Parser

**Sources:** FLUXMD.md (no mention of `## vocabulary:` directive) vs FLUXVOCAB.md §11 (line 530)

FLUXVOCAB.md §11 shows a `## vocabulary:` directive syntax for inline vocabulary definitions in `.flux.md` files:
```markdown
## vocabulary: core_actions
```fluxvocab
:double ( n -- 2n )
  IADD R0, R0, R0
  RET
```
```

But FLUXMD.md's directive recognition regex (line 163) only matches `agent` and `fn`:
```
^##\s+(?:agent|fn)\s*[:\s]\s*(.+)$
```

The `vocabulary:` directive type is **not recognized** by the `.flux.md` parser. It would be parsed as a regular level-2 heading, not a directive.

**Why it matters:** FLUXVOCAB.md claims `.flux.md` files can define vocabulary words inline, but the `.flux.md` parser doesn't support this. The feature is documented but non-functional.

### [CS-M13] MEDIUM — FLUXMD Compilation Pipeline Incomplete

**Sources:** FLUXMD.md §7 (lines 410–443) vs FIR.md §8 (implied)

FLUXMD.md shows the pipeline: `.flux.md → AST → FIRModule → BytecodeEncoder → bytes`

Missing steps:
1. **Linking** — FLUXMD.md defines `dependencies` frontmatter (line 92: "Module names this file depends on") but Step 2 notes "not yet used." Cross-module function calls via FIR's `call @func_name` will fail at link time.
2. **Optimization** — FIR.md §1 mentions optimizer passes, but FLUXMD.md's pipeline has no optimization step.
3. **Validation** — FIR.md §7 describes FIRValidator, but FLUXMD.md's pipeline doesn't invoke it.
4. **Type checking** — FIR types are type-safe, but the Python and C frontends don't perform type checking (Python is dynamically typed, C has only int/float).

**Why it matters:** The pipeline is a skeleton. Without linking, multi-module programs are impossible. Without validation, malformed FIR can reach the encoder and produce invalid bytecode.

### [CS-M14] MEDIUM — FLUXMD Python Frontend Limitations Undocumented in FIR.md

**Sources:** FLUXMD.md §5 (lines 293–298) vs FIR.md §2

FLUXMD.md's Python frontend supports: `def`, assignments, arithmetic, comparison, if/elif/else, while, for/range, return, print, calls, literals.

FIR.md's type system supports: i8, i16, i32, i64, u8, u16, u32, u64, f16, f32, f64, bool, unit, string, ref, array, vector, struct, enum, region, capability, agent, trust.

**Gap:** The Python frontend can only produce i32 and f32 FIR values (Python's `int` → i32, Python's `float` → f32). All other FIR types (structs, enums, arrays, vectors, capabilities) cannot be expressed in `.flux.md` Python code. Similarly, the C frontend only supports `int` and `float`.

**Why it matters:** FIR's rich type system is unreachable from the primary source format. `.flux.md` programs can only use a subset of FIR's types.

### [CS-M15] MEDIUM — FLUXMD `FluxCodeBlock` Not Compiled

**Sources:** FLUXMD.md §5 (lines 264–267) vs FLUXMD.md §7 (lines 410–443)

FLUXMD.md §5 states:
> "Contains FLUX/FIR-level code. Not independently compiled from `.flux.md` in the current implementation; serves as inline FIR documentation."

The `flux`, `flux-type`, and `fluxfn` dialects are parsed as `FluxCodeBlock` AST nodes but are **never compiled to bytecode**. They are documentation-only. The `flux-type` dialect (used for type definitions like `type Vec3 = struct { x: f32, y: f32, z: f32 }`) defines types that the Python/C frontends cannot use (since they only produce i32/f32).

**Why it matters:** Type definitions in `flux-type` blocks are dead code — they define types that cannot be referenced by any compilable code block.

---

## Cross-Spec Integration Issues

### [CS-M16] MEDIUM — No Bytecode Container Format Specification

**Sources:** All specs

None of the six specifications define a bytecode container format. There is no specification for:
- File header (magic number, version, entry point)
- Section layout (code, data, rodata, relocation table)
- Symbol table (function names, export/import)
- Debug information (source mapping)

FIR modules use string-named functions (`call @gcd`). When lowered to bytecode, these names must be resolved to offsets. Without a relocation/symbol format, cross-module linking is impossible.

### [CS-M17] MEDIUM — Confidence System Fragmented Across Three Specs

**Sources:** ISA.md §9 (lines 508–545), A2A.md §6 (lines 848–934), FIR.md §2.4 (lines 209–230)

Three different confidence/trust models:
1. **ISA confidence** (§9): Parallel register file, 0.0–1.0, propagated by C_* opcodes, stored by CONF_LD/CONF_ST, stripped by STRIPCF
2. **A2A trust** (§6): INCREMENTS+2 six-dimensional composite model, integer-scaled (0–1023), gating mechanism for messages
3. **FIR trust type**: `TrustType` — a bare float value with no propagation semantics

These three systems are related conceptually but have **no formal connection**. The ISA's confidence register file is per-computation (tracks data provenance). The A2A trust engine is per-agent (tracks behavioral reliability). FIR's TrustType is a bare value. There is no specification for how they interact.

### [CS-M18] MEDIUM — Signal Language Not Fully Specified

**Sources:** A2A.md §8 (referenced in ToC at line 18)

A2A.md's Table of Contents lists Section 8 "Signal Language" but the actual section content was truncated in our read (file extends beyond line 934). Based on A2A.md §2 (lines 121–129), the Signal language is a "JSON-based DSL" compiled by `signal_compiler.py`. This represents a **fourth compilation path** (Signal → Bytecode) alongside:
1. `.flux.md` → Python/C Frontend → FIR → Bytecode
2. `.fluxvocab` → Vocabulary Assembler → Bytecode (old ISA)
3. Direct bytecode authoring

Each path may produce different bytecode for equivalent programs.

### [CS-L1] LOW — FIR.md Instruction Count Claim: 54 vs Actual

**Sources:** FIR.md §1 (line 70) vs FIR.md §4 (lines 438–449)

FIR.md claims "Instruction count: 54" (line 70). The instruction categories sum to: 6 + 5 + 6 + 11 + 6 + 9 + 6 + 5 = **54**. This checks out.

### [CS-L2] LOW — FLUXMD 8+ Code Block Dialects But Only 2 Are Compiled

**Sources:** FLUXMD.md §1 (line 48) vs FLUXMD.md §5 (lines 246–262) and §7 (lines 432–436)

FLUXMD.md claims "8+ (flux, flux-type, fluxfn, json, yaml, toml, python, c, rust, …)" supported dialects. Only `python` and `c` produce compiled output. The other dialects (flux, flux-type, fluxfn, json, yaml, toml, rust) are parsed but not compiled.

### [CS-L3] LOW — A2A.md Trust Decay Formula Has Edge Case

**Sources:** A2A.md §6 (lines 874–891)

The decay formula: `T_composite *= (1 − λ · elapsed / max_age)`

At `elapsed = 0`: multiplier = 1.0 ✓
At `elapsed = max_age = 3600`: multiplier = 1 - 0.01 = 0.99

But A2A.md claims "At elapsed = 3600 (1 hour): multiplier = 0.90" (line 890).

Calculating: `1 - 0.01 * 3600 / 3600 = 1 - 0.01 = 0.99`, not 0.90.

And: "At elapsed > 3600: capped at max_age, multiplier = 0.90 (minimum)"

The math doesn't add up. For the multiplier to be 0.90 at elapsed=3600, λ would need to be 0.10, not 0.01. This is a **calculation error** in the spec.

### [CS-L4] LOW — FIR `getfield` Uses String Name But Bytecode Cannot Embed Strings

**Sources:** FIR.md §4.6 (line 657) vs ISA encoding formats

FIR's `getfield` takes a `field_name: str` parameter. The ISA's encoding formats have no mechanism for inline strings — Format A is 1 byte, Format B is 2 bytes, etc. The largest inline data is a 16-bit immediate (Format F/G).

Field names must be resolved to integer indices at FIR-to-bytecode lowering time, but this is not documented.

### [CS-L5] LOW — ISA IRET (0x03) Referenced But Interrupt Model Undefined

**Sources:** ISA.md §8.1 (line 388) vs previous audit H4

IRET "restores saved processor state" but no interrupt vector table, interrupt priority levels, or state save format is defined. This is carried forward from the previous audit.

### [CS-L6] LOW — No Spec Defines the `opcodes.py` vs `isa_unified.py` Relationship

**Sources:** FLUXVOCAB.md §12 (line 563) vs OPCODES.md header

FLUXVOCAB.md references "opcodes.py" (old, 104 ops) and "isa_unified.py" (converged, 247 ops). OPCODES.md is generated from `isa_unified.py`. But `opcodes.py` is not referenced in any other spec. The relationship between these two opcode sources is only documented in FLUXVOCAB's Known Limitations.

### [CS-L7] LOW — FIR A2A `tell` Returns Void but A2A FIR `Tell()` Returns Tag

**Sources:** FIR.md §4.8 (line 745, "Result Type: —") vs A2A.md §4 (line 634, "Return type is Tag (uint32)")

FIR.md's `tell` instruction has result type `—` (void). A2A.md's `Tell()` returns a `Tag` (uint32). The ISA's TELL opcode writes a delivery tag to `rd` (output register). FIR.md's void return type means the tag is lost in FIR — the caller cannot check whether the message was delivered successfully.

### [CS-L8] LOW — FIR A2A `trustcheck` Returns Bool but ISA TRUST Returns Previous Value

**Sources:** FIR.md §4.8 (line 748, "Result Type: BoolType") vs A2A.md §4 (line 742, "rd = prev trust") vs ISA.md §10 (line 567, "Previous trust value (for rollback)")

FIR's `trustcheck` returns a boolean (trust >= threshold). A2A.md maps this to TRUST opcode 0x5C which returns the *previous* trust value (an integer), not a boolean. These are fundamentally different return semantics.

### [CS-L9] LOW — FLUXVOCAB `@loop` Labels in Examples Don't Work

**Sources:** FLUXVOCAB.md §5.2 (lines 262–285)

The "sum from 1 to $n" example contains `@loop:` label and `JNZ R0, @loop` — but FLUXVOCAB.md itself notes (line 285): "the vocabulary assembler does NOT support labels. This entry would need to be rewritten with a calculated relative offset." This is a documentation bug: the spec contains a non-functional example without a clear "BROKEN" warning.

### [CS-L10] LOW — FLUXMD `dependencies` Frontmatter Key Dead

**Sources:** FLUXMD.md §2 (line 92) vs FLUXMD.md §7 (line 430)

The `dependencies` key is parsed from frontmatter but FLUXMD.md §7 Step 2 explicitly states it is "not yet used." There is no specification for what a dependency looks like (file path? module name? version?), how it is resolved, or what happens on missing dependencies.

### [CS-L11] LOW — Three Different `CMP` Semantics Across Specs

**Sources:** ISA.md §8.6 (CMP_EQ returns 0/1), FLUXVOCAB.md Appendix B (CMP returns -1/0/+1 into R13), FIR.md §4.4 (comparisons return BoolType)

Three incompatible comparison result conventions:
1. ISA CMP_EQ: 0 or 1
2. FLUXVOCAB CMP: -1, 0, or +1
3. FIR ieq: BoolType (conceptually true/false)

---

## Positive Findings

### Well-Done Elements

1. **ISA Encoding Format Design (ISA.md §2):** The 7-format (A–G) scheme with implicit format-from-opcode dispatch is compact and elegant. The bit diagrams and format summary table are clear and implementable. Despite the 3 format conflicts from the previous audit, the overall encoding architecture is sound.

2. **FIR SSA Design (FIR.md §5):** The block-parameters-instead-of-phi-nodes design is clean and well-motivated. The BNF grammar for FIR human-readable syntax is precise. The `TypeContext` interning system is well-specified with clear API documentation.

3. **A2A Protocol Architecture (A2A.md §2):** The three-layer design (Transport → Message Format → Bytecode/FIR) with the trust engine cross-cutting all layers is architecturally coherent. The 52-byte message header with explicit field layout and visual diagram is exemplary specification practice.

4. **A2A Trust Engine (A2A.md §6):** The INCREMENTS+2 six-dimensional composite trust model is the most fully-specified component across all six specs. The weighted formula, per-dimension scoring rules, time decay, cold start, and interaction tracking are all documented with mathematical precision.

5. **A2A Opcode Documentation (A2A.md §3):** Each of the 16 A2A opcodes has complete documentation: encoding, operand semantics, blocking behavior, trust requirements, capability requirements, error codes, and examples. This is the gold standard for opcode documentation that the rest of the ISA should follow.

6. **FLUXMD Parser Specification (FLUXMD.md §3):** The parsing rules with strict priority ordering, regex patterns, and edge case handling (unterminated fences, list grouping) are precise enough to implement without ambiguity.

7. **FLUXVOCAB Self-Awareness (FLUXVOCAB.md §12):** The Known Limitations section explicitly calls out the dual-ISA problem, substring matching issues, and lack of labels. This transparency is commendable and rare in specification documents.

8. **OPCODES.md Machine-Readability (OPCODES.md):** The single-table format with hex, mnemonic, format, operands, category, source, and description is directly parseable. The source attribution column (✅🔮⚡🌐) tracks provenance across the three contributing agents.

9. **Conformance Tiers (ISA.md §11):** The three-level conformance model (Must/Should/Nice) is pragmatic. It allows minimal implementations to claim compatibility while setting a clear path to full conformance.

10. **FLUXVOCAB Compiled Interpreter Design (FLUXVOCAB.md §9):** The `compile_interpreter()` tool that generates standalone Python modules with inline VM and assembler is a creative approach to deployment that eliminates runtime pattern-matching overhead.

---

## Recommendations

### Priority 1 — Must Fix (Critical)

| # | Finding | Action |
|---|---------|--------|
| 1 | **CS-C1**: FIR register mapping exceeds ISA capacity | Change FIR.md `value.id & 0x3F` to `value.id & 0x0F` (16 registers). Document that functions with >16 live values require register spilling. Define a spill/reload convention using LOAD/STORE. |
| 2 | **CS-C2**: FIR type widths exceed ISA support | Either (a) restrict FIR's canonical types to i32/f32 with lowering rules for wider types, or (b) extend the ISA with byte/halfword load/store and 64-bit ops. Option (a) is less disruptive. |
| 3 | **CS-C3**: FLUXVOCAB uses incompatible opcode space | Migrate the vocabulary assembler to the converged ISA encodings. This is a breaking change for existing `.fluxvocab` files but is necessary for spec conformance. Add a compatibility mode flag for old encodings. |

### Priority 2 — Should Fix (High)

| # | Finding | Action |
|---|---------|--------|
| 4 | **CS-C3** (struct ops): No FIR→ISA lowering for structs | Define a struct layout ABI (packed? natural alignment? 4-byte aligned?) and lowering rules for getfield/setfield to LOAD/STORE+offset. |
| 5 | **CS-H1**: SSA→register mapping unspecified | Define a calling convention (argument registers, return registers, callee-saved set, spill slots). |
| 6 | **CS-H2**: Comparison→branch lowering unspecified | Specify: FIR comparisons produce 0/1 in a register; FIR branch tests register == 0 (JZ). Document multi-comparison lowerings (ile = ieq ∨ ilt → two CMPs + OR). |
| 7 | **CS-H4**: A2A FIR syntax conflict | Unify on one syntax. Recommend FIR.md's lowercase positional syntax (it's simpler and consistent with other FIR instructions). Update A2A.md §4 to match. |
| 8 | **CS-H5**: A2A FIR types missing from FIR.md | Add AgentId, Payload, CapToken, AuthLevel, Tag, TaskId types to FIR.md §2.4 or define them as type aliases (e.g., `type AgentId = u128`). |
| 9 | **CS-H6**: 11 A2A opcodes lack FIR representation | Add FIR instructions for at minimum: broadcast, fork, join, signal, await. These are essential for multi-agent programs. |
| 10 | **CS-H7**: CapRequire syscall code undefined | Define SYS code for CAP_REQUIRE (suggest 0x01) in ISA.md. Add a syscall table to ISA.md §8.4. |
| 11 | **CS-H8**: A2A blocking undefined in ISA | Add a section to ISA.md defining agent suspension/resumption, cycle budget interaction, and the multi-agent scheduler model. |
| 12 | **CS-H9**: FLUXVOCAB CMP clobbers R13 (TKN) | Change CMP result register from R13 to a dedicated or agreed-upon register. R0 (ZERO) is hardwired; R1–R3 are scratch — recommend R1. |
| 13 | **CS-H10**: FLUXMD doesn't compile directive bodies | Change Step 3 to traverse AgentDirective.body nodes. This is the single most impactful fix for `.flux.md` usability. |
| 14 | **CS-H11**: FLUXMD trust/capabilities discarded | Define a bytecode metadata section that embeds trust level and capability list. A2A agent registration should read this from the bytecode. |
| 15 | **CS-H12**: `## vocabulary:` directive not in parser | Add `vocabulary` to the directive recognition regex in FLUXMD.md §4. |

### Priority 3 — Should Fix (Medium)

| # | Finding | Action |
|---|---------|--------|
| 16 | **CS-M1**: FIR missing MIN/MAX/ABS/etc. | Add FIR instructions for MIN, MAX, ABS (at minimum). These are one ISA instruction each. |
| 17 | **CS-M2**: No confidence-aware FIR instructions | Add C_iadd, C_isub, etc. to FIR, or define a confidence-lowering pass that replaces FIR ops with C_* variants. |
| 18 | **CS-M5**: FIR A2A format G vs ISA format E | Fix FIR.md §4.8 line 758: change "Format G" to "Format E". |
| 19 | **CS-M6**: A2A trust scaling inconsistency | Standardize on one representation. Recommend 0–1023 (10-bit fixed point) for all trust values, with a documented conversion to/from float. |
| 20 | **CS-M7**: A2A register operands are pointers | Add explicit statement to A2A.md §3: "All A2A register operands are memory addresses or handles, not inline data." |
| 21 | **CS-M10**: FLUXVOCAB PUSH/POP vs no-memory sandbox | Clarify: the sandbox VM has a stack region but no general heap. PUSH/POP operate on the stack. |
| 22 | **CS-M13**: FLUXMD pipeline missing linking/validation | Add Step 3.5 (validate FIR) and Step 4.5 (link modules) to the pipeline. |
| 23 | **CS-M16**: No bytecode container format | Define a minimal container: magic `0xFL58`, version byte, entry point offset, code section offset/size, data section offset/size. |
| 24 | **CS-M17**: Three disconnected trust/confidence models | Write a cross-spec note explaining the relationship: ISA confidence = data provenance, A2A trust = agent reliability, FIR TrustType = bare value for A2A trust queries. |

### Priority 4 — Nice to Fix (Low)

| # | Finding | Action |
|---|---------|--------|
| 25 | **CS-L3**: A2A trust decay math error | Fix λ from 0.01 to 0.10, or fix the example multipliers. |
| 26 | **CS-L9**: FLUXVOCAB broken example | Remove or fix the `@loop` example in §5.2. |
| 27 | **CS-L11**: Three CMP conventions | Document the mapping: FIR bool(0/1) → ISA register(0/1) → FLUXVOCAB R13(-1/0/+1). |

---

## Issue Inventory

### Summary Table

| ID | Severity | Category | Spec Pair | Description |
|----|----------|----------|-----------|-------------|
| CS-C1 | CRITICAL | Register | FIR↔ISA | FIR maps to 64 registers, ISA has 48 (16 encodable) |
| CS-C2 | CRITICAL | Types | FIR↔ISA | FIR has i8–i64/f16–f64, ISA only i32/f32 |
| CS-C3 | CRITICAL | Encoding | FLUXVOCAB↔ISA | 73% of FLUXVOCAB opcodes have wrong encodings |
| CS-C3b | CRITICAL | Encoding | FLUXVOCAB↔ISA | IADD=0x08 collides with INC=0x08 in FLUXVOCAB |
| CS-H1 | HIGH | Register | FIR↔ISA | No SSA→register allocation algorithm specified |
| CS-H2 | HIGH | Comparison | FIR↔ISA | FIR bool vs ISA 0/1 comparison semantics |
| CS-H3 | HIGH | Calling | FIR↔ISA | FIR named calls vs ISA JAL/CALL inconsistency |
| CS-H4 | HIGH | Syntax | A2A↔FIR | PascalCase vs lowercase A2A instruction syntax |
| CS-H5 | HIGH | Types | A2A↔FIR | A2A FIR types not in FIR.md type system |
| CS-H6 | HIGH | Coverage | A2A↔FIR | 11 of 16 A2A opcodes have no FIR instruction |
| CS-H7 | HIGH | Syscall | A2A↔ISA | CapRequire SYS code undefined |
| CS-H8 | HIGH | Blocking | A2A↔ISA | ISA has no suspension/blocking model |
| CS-H9 | HIGH | ABI | FLUXVOCAB↔ISA | CMP result clobbers R13 (TKN trust register) |
| CS-H10 | HIGH | Compilation | FLUXMD↔FIR | Only first NativeBlock compiled; directive bodies ignored |
| CS-H11 | HIGH | Security | FLUXMD↔A2A | Trust/capabilities parsed but discarded |
| CS-H12 | HIGH | Parsing | FLUXVOCAB↔FLUXMD | `## vocabulary:` directive not in FLUXMD parser |
| CS-M1 | MEDIUM | Coverage | FIR↔ISA | FIR missing MIN/MAX/ABS/NEG from ISA |
| CS-M2 | MEDIUM | Confidence | FIR↔ISA | FIR has no confidence-aware instructions |
| CS-M3 | MEDIUM | Conversion | FIR↔ISA | 5 of 6 FIR conversion ops lack ISA encoding |
| CS-M4 | MEDIUM | Coverage | FIR↔ISA | FIR `unreachable` has no ISA opcode |
| CS-M5 | MEDIUM | Encoding | FIR↔ISA | FIR A2A claims Format G but ISA is Format E |
| CS-M6 | MEDIUM | Trust | A2A↔ISA | Three different trust level scales |
| CS-M7 | MEDIUM | Operands | A2A↔ISA | A2A operands are pointers, not explicitly stated |
| CS-M8 | MEDIUM | Compilation | A2A↔ISA | Signal language has no ISA/FIR mapping |
| CS-M9 | MEDIUM | Sandbox | FLUXVOCAB | PUSH/POP defined but "no memory access" claimed |
| CS-M10 | MEDIUM | Security | FLUXVOCAB | Substring matching is code injection vector |
| CS-M11 | MEDIUM | Format | FLUXVOCAB↔FLUXMD | Two incompatible vocab formats |
| CS-M12 | MEDIUM | Pipeline | FLUXMD↔FIR | Missing linking, optimization, validation steps |
| CS-M13 | MEDIUM | Types | FLUXMD↔FIR | Frontends only produce i32/f32; rich FIR types unreachable |
| CS-M14 | MEDIUM | Compilation | FLUXMD↔FIR | FluxCodeBlock parsed but never compiled |
| CS-M15 | MEDIUM | Architecture | All | No bytecode container format |
| CS-M16 | MEDIUM | Trust | All | Three disconnected trust/confidence models |
| CS-L1 | LOW | Accuracy | FIR | Instruction count 54 is correct ✓ |
| CS-L2 | LOW | Completeness | FLUXMD | 8+ dialects but only 2 compiled |
| CS-L3 | LOW | Accuracy | A2A | Trust decay math doesn't match examples |
| CS-L4 | LOW | Encoding | FIR↔ISA | getfield string name can't embed in bytecode |
| CS-L5 | LOW | Completeness | ISA | IRET without interrupt model (carried from S5) |
| CS-L6 | LOW | Documentation | FLUXVOCAB | opcodes.py not referenced outside FLUXVOCAB |
| CS-L7 | LOW | Semantics | FIR↔A2A | `tell` void vs Tag return type conflict |
| CS-L8 | LOW | Semantics | FIR↔A2A | `trustcheck` bool vs TRUST prev-value conflict |
| CS-L9 | LOW | Documentation | FLUXVOCAB | Broken example with @loop labels |
| CS-L10 | LOW | Completeness | FLUXMD | `dependencies` key parsed but unused |
| CS-L11 | LOW | Semantics | All | Three different CMP result conventions |

### Severity Distribution

| Severity | Count | From Previous Audit | New/Carried |
|----------|-------|--------------------|--------------|
| CRITICAL | 4 | 4 (all different) | 4 new |
| HIGH | 12 | 9 | 12 new |
| MEDIUM | 16 | 10 | 16 new |
| LOW | 11 | 10 | 11 new |
| **Total** | **43** | **33** | **43 new** |

**Note:** The 4 critical issues from the previous ISA-only audit (format dispatch conflicts C1–C3 and register bank ambiguity C4) remain unresolved and are separate from the 4 new critical cross-spec issues identified here. Total outstanding issues across both audits: **76** (8 Critical, 21 High, 26 Medium, 21 Low).

---

## Appendix: Cross-Spec Dependency Map

```
                  ┌──────────┐
                  │  ISA.md  │  (247 opcodes, 7 formats, 48 registers)
                  │  OPCODES │  (machine-readable opcode table)
                  └────┬─────┘
                       │ defines encoding target
              ┌────────┼────────┐
              ▼        ▼        ▼
         ┌────────┐ ┌──────┐ ┌───────────┐
         │ FIR.md │ │A2A.md│ │FLUXVOCAB  │
         │ (54 IR)│ │(proto│ │(26-ops,  │
         │        │ │ col) │ │ OLD ISA)  │
         └───┬────┘ └──┬───┘ └─────┬─────┘
             │         │           │
             ▼         ▼           │
         ┌────────┐ ┌──────┐       │
         │FLUXMD  │ │Signal│       │
         │(.flux  │ │(JSON)│       │
         │ .md)   │ └──────┘       │
         └────────┘                │
              │                   │
              └──── compiled ─────┘
                     by
              ┌──────────────┐
              │ FLUX Runtime  │
              │ (VM impl)    │
              └──────────────┘
```

**Converged path:** FLUXMD → FIR → ISA bytecode → FLUX Runtime ✓
**Divergent path:** FLUXVOCAB → old opcodes → incompatible bytecode → own VM ✗
**Undefined path:** Signal → ??? → FLUX Runtime (no FIR bridge documented)

---

*End of cross-specification audit. Total findings: 43 (4 Critical, 12 High, 16 Medium, 11 Low). Combined with Session 5: 76 total outstanding issues.*
