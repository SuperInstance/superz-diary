# FLUX ISA v1.0 — Deep Audit Report

**Auditor:** Super Z (Fleet Auditor)
**Date:** 2025-01-22
**Target:** `/home/z/my-project/flux-spec/` (README.md, ISA.md, OPCODES.md)
**Scope:** ISA completeness, encoding consistency, missing specifications, cross-implementation concerns, quality assessment

---

## 1. ISA Completeness Assessment

### 1.1 Opcode Count

The spec defines **247 opcodes** across 9 reserved slots, totaling the 256-slot opcode space. This matches the README and OPCODES.md footer. Verified against OPCODES.md line-by-line: confirmed 247 defined entries.

### 1.2 Category Inventory

The opcode map in Section 7 of ISA.md shows **20 distinct range blocks**, not the "17 categories" claimed in the README. The categories as derived from the opcode map are:

| # | Range | Block Name | Count | Format |
|---|-------|-----------|-------|--------|
| 1 | 0x00–0x03 | System Control | 4 | A |
| 2 | 0x04–0x07 | Interrupt/Debug | 4 | A |
| 3 | 0x08–0x0F | Single Register | 8 | B |
| 4 | 0x10–0x17 | Immediate | 8 | C |
| 5 | 0x18–0x1F | Reg+Imm8 | 8 | D |
| 6 | 0x20–0x2F | Integer Arithmetic | 16 | E |
| 7 | 0x30–0x3F | Float/Memory/Control | 16 | E |
| 8 | 0x40–0x47 | Reg+Imm16 | 8 | F |
| 9 | 0x48–0x4F | Reg+Reg+Imm16 | 8 | G |
| 10 | 0x50–0x5F | Agent-to-Agent (A2A) | 16 | E |
| 11 | 0x60–0x6F | Confidence-Aware | 16 | E/D* |
| 12 | 0x70–0x7F | Viewpoint (Babel) | 16 | E |
| 13 | 0x80–0x8F | Sensor (JetsonClaw1) | 16 | E |
| 14 | 0x90–0x9F | Extended Math/Crypto | 16 | E |
| 15 | 0xA0–0xAF | String/Collection | 16 | D/E/G* |
| 16 | 0xB0–0xBF | Vector/SIMD | 16 | E |
| 17 | 0xC0–0xCF | Tensor/Neural | 16 | E |
| 18 | 0xD0–0xDF | Extended Memory/I/O | 15+1R | G |
| 19 | 0xE0–0xEF | Long Jumps/Debug | 13+3R | F |
| 20 | 0xF0–0xFF | Extended System | 10+5R+ILLEGAL | A |

**Finding:** README claims "17 categories" — this is inaccurate. There are 20 range blocks, and OPCODES.md uses 23 distinct category labels (system, debug, arithmetic, stack, confidence, concurrency, move, logic, shift, float, convert, memory, control, a2a, viewpoint, sensor, math, crypto, collection, vector, tensor, compute, reserved).

### 1.3 Opcode Numbering Gaps

**Reserved slots (9 total):**

| Hex | Label | Format |
|-----|-------|--------|
| 0xDF | RESERVED_DF | G |
| 0xED | RESERVED_ED | F |
| 0xEE | RESERVED_EE | F |
| 0xEF | RESERVED_EF | F |
| 0xFA | RESERVED_FA | A |
| 0xFB | RESERVED_FB | A |
| 0xFC | RESERVED_FC | A |
| 0xFD | RESERVED_FD | A |
| 0xFE | RESERVED_FE | A |

All 9 reserved slots are explicitly labeled in OPCODES.md with `—` source attribution. This is good practice. The clustering (3 at end of 0xE0 range, 5 at end of 0xF0 range) suggests intentional padding for future expansion.

### 1.4 Missing Compare Instructions

The compare set has notable gaps:
- **Defined:** CMP_EQ (0x2C), CMP_LT (0x2D), CMP_GT (0x2E), CMP_NE (0x2F)
- **Missing:** CMP_GE, CMP_LE, CMP_LTU (unsigned), CMP_GTU, CMP_GEU, CMP_LEU

CMP_GE and CMP_LE can be synthesized (CMP_GE = !(CMP_LT), CMP_LE = !(CMP_GT)) but unsigned comparisons cannot be easily synthesized from signed ones. For a VM targeting low-level systems work, unsigned comparison is important.

### 1.5 Opcodes Referencing Undefined Concepts

**CRITICAL — Referenced but never defined as opcodes:**

| Concept | Referenced In | Problem |
|---------|--------------|---------|
| **TEST** (bitwise AND, flags only, no store) | Section 6 (Condition Flags) | No TEST opcode exists in the opcode map |
| **SETCC** (condition code to register) | Section 6 (Condition Codes) | No SETCC opcode exists |
| **CMP/ICMP** (comparison, flags only) | Section 6 (Flag-Setting Instructions) | CMP_EQ etc. store result; raw CMP/ICMP that only sets flags doesn't exist |
| **REGION_CREATE** | Section 4 (Memory Model) | Referenced as "Format G opcode" but absent from opcode table |
| **REGION_DESTROY** | Section 4 (Memory Model) | Same as above |
| **REGION_TRANSFER** | Section 4 (Memory Model) | Same as above |
| **JE, JNE, JG, JL, JGE, JLE** | Section 6 (Flag-Based Branches) | These flag-based branches are listed but don't match actual opcodes (JZ, JNZ, JLT, JGT exist; JE, JNE, JGE, JLE do not) |

---

## 2. Encoding Consistency

### 2.1 Format Application

Seven instruction formats (A–G) are defined. The dispatch table in Section 2 maps opcode ranges to formats. However, there are **critical inconsistencies** between the range-based dispatch and the actual opcode definitions:

#### Inconsistency 1: C_THRESH (0x69) Format Conflict
- **Dispatch table:** Range 0x60–0x6F maps to Format E
- **Actual opcode:** 0x69 C_THRESH is Format D (rd, imm8)
- **ISA.md Section 9:** Explicitly states "(Format D)"
- **OPCODES.md:** Lists format as "D"
- **Impact:** A decoder that dispatches purely on opcode range will incorrectly decode 0x69 as Format E, reading 4 bytes instead of 3. This will cause desynchronization and crash.

#### Inconsistency 2: LEN (0xA0) Format Conflict
- **Dispatch table:** Range 0x70–0xCF maps to Format E
- **Actual opcode:** 0xA0 LEN is Format D (rd, imm8)
- **OPCODES.md:** Lists format as "D"
- **Impact:** Same desynchronization risk as C_THRESH.

#### Inconsistency 3: SLICE (0xA4) Format Conflict
- **Dispatch table:** Range 0x70–0xCF maps to Format E
- **Actual opcode:** 0xA4 SLICE is Format G (rd, rs1, imm16)
- **Format G summary table:** Explicitly lists "0xA4" as Format G alongside 0x48–0x4F and 0xD0–0xDF
- **Impact:** Decoder reads 4 bytes (Format E) but instruction is 5 bytes (Format G). Will misparse subsequent instructions.

**SEVERITY: HIGH.** These three format conflicts could cause divergent implementations. An implementor following the dispatch table in Section 2 will produce a broken decoder. An implementor cross-referencing with the opcode reference or OPCODES.md will produce a working decoder. The spec must either:
- (a) Update the dispatch table to reflect the actual formats (preferred), or
- (b) Change the opcodes to fit the dispatch ranges (disruptive).

#### Inconsistency 4: Visual Map vs. Dispatch Table Mismatch
- Section 7 visual overview shows "0xA0-0xAF D/E" (ambiguous)
- Section 2 dispatch table doesn't include 0xA0-0xAF as a separate range; it falls under "0x70-0xCF E"
- Both are wrong — the actual formats in this range are D (0xA0), E (0xA1-0xA3, 0xA5-0xAF), and G (0xA4)

### 2.2 StoreOFF Naming Mismatch

- **ISA.md Section 8.9:** Opcode 0x49 is labeled **STOREOFF**
- **OPCODES.md:** Opcode 0x49 is labeled **STOREOF** (truncated)
- **Section 2 Format G summary:** Lists **STOREOFF**
- **Impact:** Minor but creates confusion. OPCODES.md has the typo.

### 2.3 Immediate Sign Extension

The sign-extension rules are inconsistent across formats:

| Opcode | Imm Type | Sign Extension? |
|--------|----------|-----------------|
| MOVI (0x18) | imm8 | Yes, sign_extend(imm8) |
| ADDI (0x19) | imm8 | Yes, sign_extend(imm8) — ISA.md explicit |
| ANDI (0x1B) | imm8 | No, zero_extend(imm8) — ISA.md explicit |
| ORI (0x1C) | imm8 | No, zero_extend(imm8) — ISA.md explicit |
| SHLI (0x1E) | imm8 | Not specified (unsigned shift amount, likely zero-extend) |
| SHRI (0x1F) | imm8 | Not specified |
| MOVI16 (0x40) | imm16 | No, unsigned |
| ADDI16 (0x41) | imm16 | Not specified for sign extension |
| JMP (0x43) | imm16 | Yes, sign_extend for jumps |
| CALL (0x45) | imm16 | Not specified — is this signed or unsigned offset? |

**Finding:** The sign-extension rules are specified per-opcode rather than per-format, which is acceptable but incomplete. Several opcodes don't specify sign extension behavior.

### 2.4 Register ABI Consistency

| Register | ABI | ABI Name | Verdict |
|----------|-----|----------|---------|
| R0 | zero | ZERO | Consistent |
| R11 | SP | Stack Pointer | Consistent |
| R12 | RID | Region ID | Consistent |
| R13 | TKN | Trust Token | Consistent |
| R14 | FP | Frame Pointer | Consistent |
| R15 | LR | Link Register | Consistent |

The register ABI is well-defined and internally consistent across ISA.md and OPCODES.md. R0=ZERO behavior (writes ignored) is clearly specified.

### 2.5 Register File Ambiguity (R vs F vs V)

**CRITICAL ISSUE:** The ISA defines three register banks (R0–R15, F0–F15, V0–V15) but all instruction encoding formats use a 4-bit register field with values 0–15. There is **no mechanism in the encoding** to distinguish which register bank an instruction targets.

Specifically:
- Float ops (FADD, FSUB, etc.) — do they operate on R registers (interpreting bits as float) or F registers?
- Vector ops (VLOAD, VADD, etc.) — do they operate on V registers? How?
- If VLOAD uses rs1 as a memory address, rs1 must be in R registers. But VLOAD "loads vector from mem[rs1]" — where does the vector go? Into V registers? Into memory?
- Confidence ops explicitly mention a "parallel register file indexed by rd" — so C_ADD uses R registers for computation and the confidence file for confidence. This is clear.

**This is the single largest ambiguity in the spec.** A concrete example: does `FADD R3, R1, R2` interpret the bit patterns in R1 and R2 as IEEE 754 floats and store the result into R3? Or does it use F3, F1, F2? The spec says "rd = float(rs1) + float(rs2)" without specifying the register bank.

**Recommended resolution:** The most likely intended design is that all arithmetic uses R registers (with float ops reinterpreting the bits), and V registers are addressed by a separate mechanism (perhaps VLOAD/VSTORE bridge between R and V spaces). This should be explicitly documented.

---

## 3. Missing Specifications

### 3.1 FIR (FLUX Intermediate Representation)

The README lists FIR as PENDING. Based on the ISA, FIR should likely be an SSA-form IR with:
- Virtual registers (unlimited, mapped to R0–R15 by register allocator)
- Phi nodes for control flow merges
- Type annotations (i32, f32, simd[16], region_ref, confidence)
- Memory region references as first-class values
- A2A operation annotations
- Confidence propagation metadata

The ISA's three-operand format and read-before-write semantics map cleanly to SSA (each instruction defines one value). The 16 confidence variants (C_ADD etc.) suggest FIR should have a confidence-lowered mode.

**Gap:** Without FIR, there's no canonical compiler target. Each implementation must invent its own IR, leading to divergent codegen.

### 3.2 A2A Protocol

16 A2A opcodes are defined, but the protocol specification is missing:
- What is an "agent ID"? Integer? String? How is it represented in a 32-bit register?
- What is a "message"? A pointer to a memory buffer? A register value? A structured object?
- What is the "tag" in TELL (rd field)?
- How does ASK blocking interact with the cycle budget?
- What happens when ASK blocks and the cycle budget expires?
- How does FORK share memory regions with the child agent?
- What is the "trust level" representation — fixed-point in a register?
- Is DISCOV synchronous? Where does the agent list go (rd is a register, not a list)?

**Gap:** Without the A2A protocol spec, these opcodes are unusable. Implementors must either define their own protocol or leave them unimplemented.

### 3.3 Type System

The ISA has **no formal type system**. Operations implicitly define types:
- Integer: R registers, 32-bit signed
- Float: R or F registers (ambiguous), IEEE 754 single
- Vector: V registers, 128-bit SIMD
- Memory pointer: R register, 32-bit address
- Collection: undefined representation
- Confidence: parallel file, 0.0–1.0

**Missing type specifications:**
- No string type or encoding (ASCII? UTF-8? Length-prefixed? Null-terminated?)
- No collection/array type (how are they laid out in memory?)
- No tensor type (shape, element type, layout)
- No pointer type distinction (region pointers vs. raw addresses)
- No agent ID type

The String/Collection ops (0xA0–0xAF) reference strings, collections, and functions but none of these have memory layout specifications. LEN returns "length of collection imm8" — but what is imm8 here? A region ID? A memory address? A type tag?

### 3.4 .flux.md Grammar and .fluxvocab Format

Not addressed in ISA.md at all. These are source-level concerns outside the ISA's scope, but their absence means there's no canonical source language.

### 3.5 Conformance Tests

Listed as PENDING. The ISA has conformance levels (1/2/3) but no test vectors. This is the most critical missing artifact — without tests, "conformance" is unverified.

### 3.6 Missing System Specifications

| Item | Impact |
|------|--------|
| Interrupt vector table | IRET, TRAP, HANDLER reference interrupt handling but no IVT layout is defined |
| System call interface | SYS (0x10) takes an imm8 code but no syscall table is defined |
| Semaphore model | SEMA (0x14) mentions wait/signal but no semaphore data structure is specified |
| Fault model | FAULT/HANDLER define fault handling but no fault codes are enumerated |
| Calling convention | Argument passing for >4 args is undefined; no varargs; no struct return |
| Bytecode format | No header, no entry point, no section layout, no relocations |
| Linkage model | CALL, JAL, RET exist but no standard library calling convention |
| Coroutine model | COYIELD/CORESUM reference coroutines but no stack/coroutine frame format is defined |

---

## 4. Cross-Implementation Concerns

### 4.1 What a Runtime Implementor Needs (That Isn't Specified)

1. **How to bootstrap a VM instance** — No constructor/initialization API. What arguments does a VM take? Bytecode buffer? Config struct?

2. **How errors are reported** — DIV traps, but what mechanism? Exception? Error code in a register? Callback? Does the VM halt? Can it recover?

3. **How A2A handlers are registered** — "The VM's registered A2A handler callback" is mentioned but the registration API is not defined.

4. **How memory regions are created** — REGION_CREATE/DESTROY/TRANSFER are referenced but not encoded as opcodes. Is this an API call? A system call?

5. **How sensors are mapped** — SENSE takes a "sensor rs1" — what are the valid sensor IDs? How are physical sensors registered?

6. **How GPU operations work** — GPU_LD, GPU_ST, GPU_EX, GPU_SYNC are defined but no GPU memory model, kernel launch format, or synchronization semantics are specified.

7. **How the cycle budget interacts with blocking** — ASK is "blocking." Does it consume cycles while blocked? If the budget expires during a blocking call, what happens?

8. **How confidence interacts with non-confidence ops** — CONF_LD/CONF_ST use a "confidence accumulator." Is there one global accumulator? One per register? What happens when non-C ops execute — do confidence values persist unchanged?

9. **Thread safety** — No mention of whether the VM is single-threaded or multi-threaded. ATOMIC, CAS, FENCE suggest multi-threaded support but no memory model (TSO, sequential consistency, etc.) is defined.

### 4.2 Ambiguities That Could Cause Divergent Implementations

| Ambiguity | Potential Divergence |
|-----------|---------------------|
| Float register bank | Some impls use R regs, others use F regs — incompatible bytecode |
| Vector register addressing | Same as above for V regs |
| C_THRESH format (D vs E range) | Decoder desynchronization (see 2.1) |
| ENTER register save set | Different impls save different registers — incompatible calling convention |
| CAS semantics | Missing expected-value operand — impls must guess the interface |
| MALLOC rs1 parameter | Unclear what rs1 is for — could be alignment, flags, or unused |
| Collection memory layout | Without spec, LEN/AT/SETAT will work differently across impls |
| String encoding | ASCII vs UTF-8 — CONCAT will produce different results |
| CALL return address convention | PUSH(PC) vs JAL's rd=PC — inconsistent ABI |
| Jump offset base | "PC after instruction fetched" is clear, but "offset -4 jumps to JMP itself" — is -4 signed in the 16-bit field? Confirmed yes, but worth explicit test |
| Unsigned vs signed division | DIV says "signed" explicitly, but MOD says "signed modulo" — what about unsigned divmod? |

### 4.3 Division by Zero Behavior

DIV (0x23) and MOD (0x24) say "traps on division by zero." But:
- What trap? A hardware fault? A software exception?
- Can it be caught by HANDLER (0xE8)?
- Does it set any flags before trapping?
- What value does rd get (if any)?
- FDIV (0x33) says "traps on zero" — same questions

This needs specification.

---

## 5. Quality Assessment

### 5.1 Scores (1–10)

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| **Completeness** | 5/10 | Core ISA (arithmetic, memory, control flow) is well-covered. Extended domains (A2A, sensors, tensors, viewpoints) have opcodes but lack protocol/type specifications. Several referenced concepts (TEST, SETCC, CMP, REGION_*) are undefined. |
| **Clarity** | 7/10 | Writing is clear and well-structured. Good use of tables. Section 2 (Encoding Formats) is excellent. But inconsistencies between sections and the format dispatch table undermine clarity. |
| **Correctness** | 6/10 | Core opcodes and execution model are correct. Flag semantics are well-defined. But format dispatch has 3 conflicts that would break a naive decoder. Statistics mismatch between README and ISA.md. |
| **Implementability** | 4/10 | A skilled implementor could build a working Level 1 VM from this spec. But Level 2+ requires guessing: float register bank, A2A protocol, memory region API, sensor mapping, GPU model. The format conflicts are a landmine. |

### 5.2 Strongest Sections

1. **Section 2 (Encoding Formats)** — The 7-format design is clean. Format A–G summary table and bit diagrams are excellent. The decision to make format implicit from opcode value is clever and compact.

2. **Section 5 (Execution Model)** — Fetch-decode-execute loop, cycle budget, read-before-write semantics, jump offset semantics — all clearly specified. An implementor can build the core execution loop from this section alone.

3. **Section 6 (Condition Flags)** — Z/S/C/O flags, _set_flags/_set_cmp_flags, SETCC condition codes (even though SETCC isn't an opcode), flag-based branch table — well-structured and complete (within its scope).

4. **Section 9 (Confidence-Aware Variants)** — The confidence propagation rules table is a standout. The "confidence-OPTIONAL" design philosophy is clearly articulated and the per-operation rules (min, product, weighted average, etc.) are mathematically precise.

5. **Section 11 (Conformance Requirements)** — Three-tier conformance (Must/Should/Nice) is pragmatic. Clear enumeration of Level 1 requirements.

### 5.3 Sections Needing Most Work

1. **Format Dispatch Table (Section 2)** — MUST FIX. Three opcodes (0x69, 0xA0, 0xA4) conflict with their range-based format assignments. This is a correctness bug.

2. **Memory Model (Section 4)** — References REGION_CREATE/DESTROY/TRANSFER which don't exist as opcodes. Doesn't define string/collection/tensor memory layouts. Missing garbage collection or ownership transfer mechanics beyond capability-based regions.

3. **Register Bank Selection** — CRITICAL. Must specify whether float/vector ops use R, F, or V registers. This ambiguity will cause incompatible implementations.

4. **Error/Fault Model** — Division by zero, illegal instructions, memory access violations — what happens? How are faults caught? The FAULT/HANDLER opcodes suggest a fault mechanism but it's not specified.

5. **Extended Ops Sections (8.7+, 10)** — Viewpoint ops (0x70–0x7F) have descriptions so vague they read like design notes, not specifications. Sensor ops (0x80–0x8F) reference hardware that needs enumeration. Tensor ops (0xC0–0xCF) reference shapes, dimensions, and data types without defining them.

6. **Statistics Consistency** — README says 247/9, ISA.md Section 1 says ~240/~16. Must be reconciled.

---

## Appendix: Complete Issue Inventory

### CRITICAL (would break implementations)
- [C1] Format dispatch conflict: 0x69 C_THRESH is Format D, not E
- [C2] Format dispatch conflict: 0xA0 LEN is Format D, not E
- [C3] Format dispatch conflict: 0xA4 SLICE is Format G, not E
- [C4] Register bank ambiguity: float/vector ops have no bank selection mechanism

### HIGH (blocks Level 2+ conformance)
- [H1] TEST opcode referenced in Section 6 but not defined
- [H2] SETCC opcode referenced in Section 6 but not defined
- [H3] CMP/ICMP referenced in Section 6 but not defined
- [H4] REGION_CREATE/DESTROY/TRANSFER referenced in Section 4 but not defined as opcodes
- [H5] JE, JNE, JGE, JLE listed as flag-based branches but don't exist as opcodes
- [H6] No memory layout for strings, collections, tensors
- [H7] No A2A protocol specification
- [H8] Division-by-zero trap mechanism unspecified
- [H9] No calling convention for >4 arguments

### MEDIUM (inconsistencies and gaps)
- [M1] Statistics mismatch: README 247/9 vs ISA.md ~240/~16
- [M2] "17 categories" in README is incorrect (20 range blocks, 23 OPCODES.md labels)
- [M3] STOREOFF vs STOREOF naming mismatch
- [M4] Missing CMP_GE, CMP_LE compare opcodes
- [M5] Missing unsigned compare opcodes
- [M6] ENTER register save set unspecified
- [M7] LEAVE register restore set unspecified
- [M8] MALLOC rs1 parameter purpose undefined
- [M9] CAS semantics incomplete (missing expected-value operand)
- [M10] CALL vs JAL return address convention inconsistency (PUSH vs rd)

### LOW (nice-to-fix)
- [L1] No byte/halfword load/store (LOADB, LOADH, STOREB, STOREH)
- [L2] No 64-bit arithmetic or addressing support
- [L3] No MULH (high product) or combined DIVMOD
- [L4] Missing interrupt vector table specification
- [L5] Missing system call table
- [L6] Missing semaphore data structure specification
- [L7] Missing bytecode container format (header, entry point, sections)
- [L8] No memory ordering model (TSO, SC) despite ATOMIC/CAS/FENCE
- [L9] No coroutine frame format despite COYIELD/CORESUM
- [L10] Sign extension unspecified for several imm8/imm16 opcodes

---

*End of audit. Total issues: 4 Critical, 9 High, 10 Medium, 10 Low = 33 findings.*
