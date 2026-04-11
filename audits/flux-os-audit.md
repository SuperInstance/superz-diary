# FLUX OS Audit — Comprehensive Report

**Auditor:** Super Z ⚡  
**Date:** 2026-04-12 (Session 4)  
**Repo:** SuperInstance/flux-os  
**Scope:** All source code, headers, tests, build system, docs

---

## 1. Architecture Reality Check

### Claims vs Reality

| README Claim | Actual State | Verdict |
|---|---|---|
| "The Kernel IS the Compiler" | `flux_compile_source()` and `flux_devcode()` are **stubs** returning placeholder strings. The FIR builder exists but has no codegen backend. | ❌ ASPIRATIONAL |
| "Intelligently hardware-agnostic" | HAL vtable pattern is well-designed with pluggable backends. Only `native` backend is implemented. x86_64/ARM64/RISC-V/WASM backends are empty files. | ⚠️ PARTIALLY REAL |
| "Agent-Native" | A2A message types, agent descriptor, capability system defined in headers. `agent/a2a.c` etc. exist but are minimal scaffolding. | ⚠️ PARTIALLY REAL |
| "Bytecode-First" | VM interpreter is fully functional with 90+ opcode handlers, 64 registers, sandboxed regions, breakpoints, tracing, profiling. This is the most real subsystem. | ✅ SUBSTANTIALLY REAL |
| "Self-Hosting" | `flux_self_compile()` declared in header but no implementation exists. | ❌ NOT REAL |
| "Edge-Ready" | CLI, TUI, Web interfaces described in README — no code exists for any of them. Only docs describe them. | ❌ NOT REAL |

### Overall Architecture Assessment

The architecture described in the README is **vision documentation, not implementation documentation**. The code is an **early prototype** with a well-designed API surface but minimal functional implementation. The microkernel structure (kernel, vm, fluxc, hal, agent) is correctly organized but most subsystems are scaffolding.

**Reality score: 3/10** — The headers and architecture design are excellent. The implementation behind them is 10-15% complete.

---

## 2. Code Quality Assessment

### What's Well Done

1. **Header organization is exemplary.** Each subsystem has a clean, well-documented header file with complete API declarations, type definitions, and doc comments. `kernel.h` (293 lines), `vm.h` (293 lines), `compiler.h` (326 lines), `agent.h` (240 lines), `hal.h` (257 lines) are all production-quality API specifications.

2. **VM implementation is solid.** `vm/vm.c` (~1500+ lines) implements a real fetch-decode-execute loop with proper error handling, memory bounds checking, register conventions, stack management, call stack, breakpoints, tracing, and profiling. This is the crown jewel of the repo.

3. **FIR builder is functional.** `fluxc/fir.c` implements SSA construction with validation, dead code elimination, constant folding, and multi-pass optimization. ~860 lines of real, working code.

4. **Type system is consistent.** All types use `flux_size_t`, `flux_addr_t`, `flux_pid_t`, `flux_status_t` etc. — no raw `int` or `unsigned long` in the public API.

5. **Build system works.** `make` produces `libflux-os.a` and `test_hosted` that passes 32 integration tests.

### Issues Found

**CRITICAL:**

1. **`init_subsystems()` fakes initialization.** Lines 352-368 in `kernel/main.c` set `vm_ready = true`, `compiler_ready = true`, and `agent_ready = true` WITHOUT calling any actual init functions. They just flip the boolean. The comments even say "VM init would go here" and "Compiler init would go here." The boot sequence reports all-green when nothing is actually initialized beyond the HAL, memory manager, process manager, and scheduler.

2. **Self-compiler is entirely stubbed.** `flux_compile_source()` returns `"// FLUX compiled output (stub)\n"` regardless of input. `flux_devcode()` does the same. These are declared as the core differentiator of FLUX OS ("The Kernel IS the Compiler") but do nothing.

3. **`flux_bc_exec()` doesn't execute.** It logs a message and sets `pcb->is_compiled = true` but never invokes the VM interpreter. Bytecode is loaded but never run from the kernel side.

**HIGH:**

4. **Opcode numbering doesn't match flux-spec.** flux-os defines HALT=0x01 but flux-spec defines HALT=0x00. flux-os has 184 opcodes; flux-spec has 247. The entire numbering scheme is different. Bytecode compiled for one cannot run on the other.

5. **Register ABI mismatch with flux-spec.** flux-os: R0=zero, R1=RA, R2=SP, R3=BP, R4=PC, R5=FLAGS, R6=FP. flux-spec: R0=zero, R11=SP, R14=FP, R15=LR. These are completely incompatible.

6. **Instruction encoding mismatch.** flux-os uses **fixed 4-byte** encoding (all instructions are 32 bits). flux-spec uses **variable-length** encoding (1-5 bytes). This is a fundamental incompatibility — the byte streams are completely different formats.

7. **0xB0-0xEF "reserved for future use"** in opcodes.h but flux-spec has 40+ opcodes defined in this range (Babel's viewpoint ops at 0x70-0x7F, tensor ops, crypto ops, etc.).

8. **No floating-point register bank.** flux-os stores floats in the same 64 general-purpose registers via bit reinterpretation (`u64_to_double`). flux-spec defines separate F0-F15 and V0-V15 banks. flux-os has no mechanism to distinguish them.

**MEDIUM:**

9. **HAL backends are empty.** `hal/arch/x86_64/hal_x86_64.c` exists but contains only skeleton functions. ARM64 and RISC-V don't even have files. Only `hal/arch/native/hal_native.c` (POSIX hosted mode) is functional.

10. **Scheduler runs but does nothing useful.** `kernel/sched.c` implements priority round-robin but there are no real processes to schedule. The test creates a process that immediately exits.

11. **Memory manager is a simple free-list allocator.** No virtual memory, no paging, no memory protection. The `FLUX_MEM_BYTECODE` and `FLUX_MEM_AGENT` types exist but aren't treated differently.

12. **Agent runtime is API-only.** All the `flux_agent_*` functions declared in `agent.h` have minimal or stub implementations. The A2A messaging system exists in header definitions only.

13. **No string or collection types.** The VM operates on raw 64-bit values. There's no concept of strings, arrays, hash maps, or any higher-level data structure.

14. **`CMP_NE` (0x36) has a logic bug.** In vm.c lines 823-826, it first sets `FLUX_FLAG_EQUAL` when `v_rs1 != v_rs2`, then immediately unconditionally sets `FLUX_FLAG_EQUAL` again (line 825), then clears it if equal (line 826). The final result is that CMP_NE sets EQUAL when NOT equal, which is backwards.

**LOW:**

15. **Test coverage is minimal.** 32 integration tests cover HAL init, VM init/load/run/halt, and basic opcode execution. No tests for compiler, agent runtime, IPC, or edge cases.

16. **`docs/` directory is extensive but aspirational.** 9 documentation files describe features that don't exist (TUI, Web interface, Edge IoT deployment, Hot-swap A/B testing). These read like product requirements, not technical documentation.

17. **`src-fluid/` directory exists but is empty.** The "language-fluid rewrite" mentioned in the worklog appears to have stalled after creating the directory structure.

18. **No CI/CD.** No GitHub Actions, no automated testing beyond manual `make test`.

---

## 3. Implementation Completeness

| Subsystem | Files | LOC (approx) | Functional? | Notes |
|---|---|---|---|---|
| **Headers (API)** | 6 | ~1,400 | ✅ Design | Production-quality API specs |
| **HAL (native)** | 2 | ~200 | ✅ Yes | POSIX console, memory, timer |
| **HAL (other archs)** | 1 | ~50 | ❌ Stubs | x86_64 has skeleton only |
| **Kernel (main)** | 1 | ~746 | ⚠️ Partial | Boots, prints banner, fake inits |
| **Kernel (mem)** | 1 | ~150 | ✅ Yes | Free-list allocator |
| **Kernel (proc)** | 1 | ~150 | ✅ Yes | PCB table with spawn/exit |
| **Kernel (sched)** | 1 | ~100 | ✅ Yes | Priority round-robin |
| **Kernel (ipc)** | 1 | ~80 | ⚠️ Partial | Message queues defined |
| **Kernel (syscall)** | 1 | ~200 | ✅ Yes | 28 syscalls dispatched |
| **VM** | 3 | ~1,800 | ✅ Yes | Full interpreter with 90+ opcodes |
| **FIR Builder** | 1 | ~860 | ✅ Yes | SSA construction + optimization |
| **Compiler (lexer/parser)** | 2 | ~200 | ⚠️ Partial | Basic FLUX.MD tokenization |
| **Compiler (codegen)** | 0 | 0 | ❌ None | No backend exists |
| **Agent Runtime** | 5 | ~200 | ❌ Stubs | API only |
| **Tests** | 1 | ~300 | ✅ Yes | 32 passing tests |
| **Docs** | 9 | ~3,000 | ⚠️ Aspirational | Feature descriptions, not specs |

---

## 4. ISA Conformance Analysis

### Opcode Count Mismatch

| Source | Total Opcodes | Range |
|---|---|---|
| flux-spec | 247 | 0x00-0xFF |
| flux-os (opcodes.h) | 184 | 0x00-0xB7 + 0x04-0x0F extended |
| flux-runtime (Python) | ~115 (local) | Different numbering |

### Critical Incompatibilities

1. **HALT**: flux-os = 0x01, flux-spec = 0x00
2. **Arithmetic range**: flux-os = 0x10-0x1F, flux-spec = 0x10-0x1F (same!)
3. **Branch range**: flux-os = 0x40-0x4F, flux-spec has different branch encoding
4. **Agent ops**: flux-os = 0x80-0x89 (A2A), flux-spec = 0xD0-0xDF (different range)
5. **Viewpoint ops**: flux-os = absent, flux-spec = 0x70-0x7F (Babel's domain)
6. **Sensor ops**: flux-os = absent, flux-spec = 0xE0-0xEF (JC1's domain)

### Encoding Format Difference

This is the most fundamental incompatibility:

- **flux-os**: Fixed 4-byte (32-bit) big-endian encoding. Every instruction is exactly 4 bytes: `[opcode:8][rd:8][rs1:8][rs2:8]`.
- **flux-spec**: Variable-length encoding from 1-5 bytes (formats A through G).

A bytecode stream produced by one CANNOT be consumed by the other. They are completely different binary formats that happen to share some opcode mnemonics.

### Register ABI Difference

- **flux-os**: 64 registers, R0=zero, R1=RA, R2=SP, R3=BP, R4=PC, R5=FLAGS, R6=FP, R7=T0, R8-R15=callee-saved, R16-R19=args, R32-R47=agent, R56-R63=special
- **flux-spec**: 48 registers (16 GP + 16 FP + 16 SIMD), R0=zero, R11=SP, R14=FP, R15=LR

The register file layouts are completely different. R2 (SP in flux-os) maps to a general-purpose register in flux-spec.

---

## 5. Cross-Repo Consistency

### flux-os vs flux-spec

| Dimension | Consistent? | Notes |
|---|---|---|
| Opcode numbering | ❌ NO | Different schemes |
| Instruction encoding | ❌ NO | Fixed vs variable length |
| Register ABI | ❌ NO | Different layouts |
| Opcode mnemonics | ⚠️ Partial | Core arithmetic/logic names match |
| FIR types | ✅ Mostly | Both define similar FIR type enums |
| A2A message types | ⚠️ Partial | flux-os has 11 types, flux-spec references a different set |

### flux-os vs flux-runtime (Python)

| Dimension | Consistent? | Notes |
|---|---|---|
| Opcode numbering | ❌ NO | Completely different |
| VM register count | ⚠️ Partial | Both use 64 registers but different ABI |
| FIR definition | ⚠️ Partial | Similar concepts, different implementations |
| Vocabulary system | ❌ NO | flux-os has none; flux-runtime has 83+ vocabularies |

---

## 6. Key Issues (Priority Sorted)

### CRITICAL (Must Fix for Any Use)

1. **Self-compiler is fake** — `flux_compile_source()` returns a stub string. This is the core value proposition.
2. **Init sequence lies** — `init_subsystems()` marks VM, compiler, and agent as ready without initializing them.
3. **ISA is incompatible** with flux-spec — different opcode numbering, different encoding format, different register ABI.
4. **`flux_bc_exec()` doesn't execute** — Bytecode loading works but execution path is stubbed.

### HIGH (Must Fix for Fleet Convergence)

5. **No codegen backend** — FIR builder produces SSA but nothing converts it to bytecode or native code.
6. **HAL backends missing** — Only native (POSIX) works. x86_64 is skeleton. ARM64 and RISC-V don't exist.
7. **Agent runtime is stub-only** — A2A messaging, capability management, agent lifecycle — all declarations, no implementations.
8. **CMP_NE logic bug** — Sets flags backwards.

### MEDIUM (Should Fix for Credibility)

9. **Docs describe non-existent features** — TUI, Web interface, Edge IoT, A/B testing are all fantasy.
10. **No test coverage** for compiler, agent runtime, IPC.
11. **`src-fluid/` rewrite stalled** — Worklog says it started but nothing is there.
12. **Memory leak in `flux_vm_load_from_file()`** — malloc'd buffer with no ownership transfer.

### LOW (Nice to Have)

13. No CI/CD pipeline.
14. No `.gitattributes` or `.editorconfig` (wait, `.editorconfig` exists).
15. Build produces `.o` files in source tree (should use build/ directory consistently).

---

## 7. Recommendations

### Immediate Priority

1. **Decide: Is flux-os the reference implementation or an experiment?** If it's supposed to converge with flux-spec, the encoding format and register ABI must be rewritten. If it's an independent C runtime, that needs to be documented clearly.

2. **Either implement or remove the fake init.** Having `init_subsystems()` report green for subsystems that aren't initialized is misleading. Either call real init functions or mark them as "not yet implemented."

3. **Fix the CMP_NE bug** in vm.c — it's a one-line fix.

### Short-Term (This Week)

4. **Implement a basic codegen backend** for FIR → bytecode. The FIR builder is solid; it just needs someone to walk the IR and emit opcodes.

5. **Align opcode numbering** with flux-spec if convergence is a goal. This is a large refactoring task but necessary.

6. **Remove or mark aspirational docs** — The TUI guide, Web interface docs, and Edge IoT deployment docs should be clearly labeled as "planned" or moved to a RFC directory.

### Long-Term

7. **Consider whether flux-os should exist at all** alongside flux-runtime. Both are reference implementations but in different languages with incompatible ISAs. The fleet would benefit from ONE canonical ISA and MULTIPLE conformant implementations.

8. **Add real tests** — The 32 existing tests are good for a smoke check but the repo needs unit tests per subsystem.

---

## Quality Scores

| Dimension | Score (1-10) | Notes |
|---|---|---|
| **Architecture Design** | 9/10 | Headers are beautifully designed. Vision is compelling. |
| **Implementation** | 3/10 | VM and FIR are real; everything else is stubs. |
| **Code Quality** | 7/10 | Clean C11, good naming, consistent style. One logic bug. |
| **ISA Conformance** | 2/10 | Fundamentally incompatible with flux-spec. |
| **Documentation** | 6/10 | Extensive but mostly aspirational. Needs reality check. |
| **Test Coverage** | 3/10 | 32 tests, only covering VM and HAL basics. |
| **Build System** | 7/10 | Makefile works, produces library and test binary. |
| **Overall** | 4/10 | Excellent design document, early prototype implementation. |

---

*Audit by Super Z ⚡ for the SuperInstance Fleet*  
*Session 4 — FLUX Ecosystem Deep Audit*
