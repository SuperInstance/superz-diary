# FLUX Spec — Key Findings (Actionable)

**Auditor:** Super Z | **Date:** 2025-01-22 | **Total Findings:** 33

---

## Must-Fix Now (4 Critical)

### C1–C3: Format Dispatch Table Is Broken
Three opcodes don't match their range-assigned format in Section 2's dispatch table:
- **0x69 C_THRESH** — listed as range 0x60–0x6F → Format E, but is actually **Format D** (3 bytes, not 4)
- **0xA0 LEN** — falls under range 0x70–0xCF → Format E, but is actually **Format D** (3 bytes)
- **0xA4 SLICE** — falls under range 0x70–0xCF → Format E, but is actually **Format G** (5 bytes)

**Impact:** A decoder using the range table will read the wrong number of bytes and desynchronize. **Action:** Rewrite the dispatch table to list individual exceptions, or reorganize the opcode numbering.

### C4: Register Bank Ambiguity
The ISA defines R0–R15 (integer), F0–F15 (float), V0–V15 (vector) but all instruction formats encode only a 4-bit register field with no bank selector. Float ops (FADD etc.) and vector ops (VLOAD etc.) don't specify which bank they target.

**Impact:** Implementations will disagree on register assignment → incompatible bytecode.
**Action:** Explicitly document which register bank each opcode category uses. Most likely: float ops reinterpret R registers as IEEE 754; vector ops bridge R↔V via VLOAD/VSTORE.

---

## Fix Before Level 2 Conformance (9 High)

| ID | Issue | Action |
|----|-------|--------|
| H1 | `TEST` opcode referenced in Section 6 but never defined | Add TEST opcode or remove the reference |
| H2 | `SETCC` opcode referenced in Section 6 but never defined | Add SETCC opcode or remove the reference |
| H3 | `CMP`/`ICMP` referenced in Section 6 but never defined | Clarify that CMP_EQ etc. replace them, or add CMP |
| H4 | `REGION_CREATE/DESTROY/TRANSFER` referenced in Section 4 but not in opcode table | Add as opcodes or specify as MemoryManager API |
| H5 | `JE/JNE/JGE/JLE` listed as flag branches but only JZ/JNZ/JLT/JGT exist | Align Section 6 table with actual opcodes |
| H6 | No memory layout spec for strings, collections, tensors | Define in-memory representation for each type |
| H7 | No A2A protocol specification | Specify agent ID format, message encoding, blocking semantics |
| H8 | Division-by-zero trap mechanism unspecified | Define: fault code, handler invocation, flag state |
| H9 | No calling convention for >4 arguments | Specify stack-based argument passing rules |

---

## Inconsistencies (10 Medium)

| ID | Issue |
|----|-------|
| M1 | README says 247 defined / 9 reserved; ISA.md §1 says "~240 / ~16" — reconcile |
| M2 | README claims "17 categories" — actually 20 range blocks / 23 category labels |
| M3 | Opcode 0x49 is `STOREOFF` in ISA.md but `STOREOF` in OPCODES.md — typo |
| M4 | Missing CMP_GE and CMP_LE compare opcodes |
| M5 | No unsigned compare variants (CMP_LTU etc.) |
| M6 | ENTER (0x4C) says "Push registers" but doesn't specify which registers |
| M7 | LEAVE (0x4D) says "pop registers" — same ambiguity |
| M8 | MALLOC (0xD7) has unused rs1 parameter — purpose unclear |
| M9 | CAS (0xD5) needs 3 values (addr, expected, desired) but Format G only has rd, rs1, imm16 |
| M10 | CALL (0x45) uses `PUSH(PC)` for return address; JAL (0x44) uses `rd = PC` — inconsistent |

---

## Quality Scores

| Dimension | Score | Note |
|-----------|-------|------|
| Completeness | 5/10 | Core ISA solid; extended domains lack protocol/type specs |
| Clarity | 7/10 | Well-written tables; format dispatch table undermines it |
| Correctness | 6/10 | Execution model correct; format conflicts are bugs |
| Implementability | 4/10 | Level 1 buildable; Level 2+ requires guessing |

**Overall Verdict:** The core ISA (arithmetic, memory, control flow) is well-designed and ready for implementation. The format dispatch table bug is the most urgent fix. The register bank ambiguity is the most consequential design gap. Everything else is polish and missing companion specs (FIR, A2A protocol, type system).

---

*Full audit with detailed analysis: [flux-spec-audit.md](flux-spec-audit.md)*
