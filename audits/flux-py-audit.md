# FLUX-PY Deep Audit

**Date:** 2026-04-12  
**Auditor:** Super Z  
**Repo:** `/home/z/my-project/flux-py`  
**Commit context:** v0.1.0 (2026-04-10)  

---

## 1. Architecture Overview

### What is flux-py?

**flux-py** is the Python implementation of the FLUX (Fluid Language Universal eXecution) runtime. The `pyproject.toml` package name is `flux-runtime` (version 0.1.0). It is a **markdown-to-bytecode compiler and VM runtime** designed for AI agents. Users write structured markdown files (FLUX.MD) containing polyglot code blocks (C, Python, etc.), which the compiler weaves into a single bytecode that runs on a 64-register Micro-VM.

The project claims **zero external dependencies** — everything runs on Python 3.10+ stdlib alone. The README badge claims 1,848 tests, though actual test functions count is **304** (more on this in Section 5).

### Relation to flux-runtime

`flux-py` is located at `/home/z/my-project/flux-py` while a separate `flux-runtime` exists at `/home/z/my-project/flux-runtime`. They are **near-identical codebases**:

| Metric | flux-py | flux-runtime |
|--------|---------|-------------|
| Opcodes | 115 | 115 |
| Test functions | 304 | 439 |
| Source modules | ~113 non-init | More (includes `open_interp`, `debugger`, `disasm`, `repl`) |
| Vocabularies directory | **MISSING** | Present (`core/`, `custom/`, `examples/`, `loops/`, `math/`) |
| Module count | 106 claimed | 106 claimed |

**Key difference:** `flux-runtime` is the canonical/upstream repo with **more tests (439 vs 304)**, additional modules (`open_interp`, `debugger.py`, `disasm.py`, `repl.py`), and a `vocabularies/` directory. `flux-py` is a slightly stale fork.

### Source Code Structure and Module Organization

```
src/flux/
  __init__.py, __main__.py, cli.py, py.typed
  bytecode/     opcodes.py, encoder.py, decoder.py, validator.py  (ISA layer)
  vm/           registers.py, memory.py, interpreter.py          (execution engine)
  fir/          types.py, values.py, instructions.py, blocks.py, 
                builder.py, validator.py, printer.py            (SSA IR)
  parser/       nodes.py, parser.py                              (FLUX.MD parser)
  frontend/     c_frontend.py, python_frontend.py                (language compilers)
  compiler/     pipeline.py                                      (compilation pipeline)
  pipeline/     e2e.py, polyglot.py, debug.py                    (end-to-end pipeline)
  optimizer/    passes.py, pipeline.py                            (optimization)
  jit/          compiler.py, ir_optimize.py, cache.py, tracing.py (JIT compiler)
  types/        generic.py, unify.py, compat.py                   (type system)
  stdlib/       intrinsics.py, math.py, strings.py, collections.py, agents.py
  a2a/          messages.py, transport.py, trust.py, coordinator.py (A2A protocol)
  runtime/      agent.py, agent_runtime.py                       (agent lifecycle)
  security/     sandbox.py, resource_limits.py, capabilities.py
  tiles/        tile.py, ports.py, graph.py, registry.py, library.py
  modules/      card.py, namespace.py, reloader.py, granularity.py, container.py
  adaptive/     profiler.py, selector.py, compiler_bridge.py
  evolution/    genome.py, mutator.py, validator.py, evolution.py, pattern_mining.py
  synthesis/    synthesizer.py, demo.py, report.py
  flywheel/     hypothesis.py, metrics.py, engine.py, knowledge.py
  memory/       store.py, experience.py, learning.py, bandit.py
  reload/       hot_loader.py
  swarm/        agent.py, swarm.py, topology.py, message_bus.py, deadlock.py
  simulation/   digital_twin.py, oracle.py, speculator.py, predictor.py
  creative/     generative.py, visualization.py, sonification.py, live.py
  cost/         model.py, energy.py
  protocol/     message.py, channel.py, serialization.py, registry.py, negotiation.py
  schema/       architecture.py, opcode_schema.py, builder_schema.py, tile_schema.py
  docs/         generator.py, introspector.py, renderer.py, stats.py
  retro/        catalog.py + implementations/ (10 games) + research/ (AI evolution data)
  reverse/      engineer.py, code_map.py, parsers/ (c_reverse.py, python_reverse.py)
  migrate/      migrator.py, report.py
  mega/         conductor.py, demo_mega.py
```

### Key Abstractions

1. **Bytecode Layer** (`bytecode/`): Defines 115 opcodes in an `IntEnum`, variable-length encoding formats A–G, and provides encoder (FIR→binary), decoder (binary→structured), and validator.

2. **VM Interpreter** (`vm/`): A fetch-decode-execute loop running on raw bytes. 48 registers (R0–R15 GP, F0–F15 float, V0–V15 SIMD vector). Supports condition flags, stack frames, box table, resource tracking, A2A dispatch, and configurable cycle budgets.

3. **FIR (SSA IR)** (`fir/`): The universal pivot point. 15 FIR types with interning via `TypeContext`, 42+ instruction types, SSA values, basic blocks, functions, and modules. Builder API for ergonomic construction.

4. **Frontends**: C compiler (tokenizer + recursive descent + FIR codegen) and Python compiler (AST-based). Both lower to FIR.

5. **Compiler Pipeline** (`compiler/`): `FluxCompiler` drives both frontends and the bytecode encoder.

6. **Vocabulary System**: **MISSING** from flux-py. The flux-runtime repo has a `vocabularies/` directory with `core/`, `custom/`, `examples/`, `loops/`, `math/` subdirectories. flux-py has no such directory.

---

## 2. ISA Conformance

### Opcode Count

flux-py defines **115 unique opcodes** in `src/flux/bytecode/opcodes.py`, occupying hex slots 0x00–0x84 with gaps.

### Opcode Numbering vs. flux-spec

**flux-spec** (at `/home/z/my-project/flux-spec/OPCODES.md`) defines a **completely different ISA** with **247 opcodes** (256 slots total, 9 reserved). The divergence is **TOTAL** — flux-py's opcode table does not match flux-spec at all:

| Aspect | flux-py (implemented) | flux-spec (canonical) |
|--------|----------------------|---------------------|
| Total opcodes | 115 | 247 |
| HALT | 0x80 | 0x00 |
| NOP | 0x00 | 0x01 |
| ADD/IADD | 0x08 | 0x20 |
| MOV | 0x01 | 0x3A |
| JMP | 0x04 | 0x43 |
| A2A TELL | 0x60 | 0x50 |
| Float ops | 0x40–0x4F | 0x30–0x37 |
| Vector ops | 0x50–0x56 | 0xB0–0xBF |
| Confidence ops | **none** | 0x60–0x6F (16 ops) |
| Viewpoint ops | **none** | 0x70–0x7F (16 ops) |
| Sensor ops | **none** | 0x80–0x8F (16 ops) |
| Tensor/ML ops | **none** | 0xC0–0xCF (16 ops) |
| Collection ops | **none** | 0xA0–0xA9 |
| Crypto ops | **none** | 0x98–0xAE |
| Math ops | **none** | 0x90–0x97, 0x9C–0x9F |

### Opcode Numbering Gaps in flux-py

```
0x3D–0x3F (3 gaps)  — after CHECK_BOUNDS
0x58–0x5F (8 gaps)  — after STORE8/VFMA (SIMD region)
0x6D–0x6F (3 gaps)  — after SET_PRIORITY (A2A region)
0x7C–0x7F (4 gaps)  — after EMERGENCY_STOP (A2A region)
```

### Summary

**flux-py's ISA is completely incompatible with flux-spec.** The opcode numbers, formats, and categories are entirely different. flux-py uses an older/different numbering scheme that appears to have been designed independently before the convergence effort that produced flux-spec's 247-opcode ISA.

---

## 3. Vocabulary System

### Status: MISSING from flux-py

flux-py has **no `vocabularies/` directory** and no vocabulary-related source code anywhere in the tree. There is no `src/flux/vocabulary/` module, no vocabulary loading mechanism, and no vocabulary abstraction in the bytecode or VM layers.

### flux-runtime has vocabularies

The canonical `flux-runtime` repo at `/home/z/my-project/flux-runtime/vocabularies/` contains:
- `core/` — Core vocabulary
- `custom/` — Custom user-defined vocabularies  
- `examples/` — Example vocabularies
- `loops/` — Loop pattern vocabularies
- `math/` — Math operation vocabularies

### Impact

The vocabulary system is designed to allow extensibility of the ISA through composable, named opcode sets. Its absence in flux-py means:
1. No way to load custom opcode extensions
2. No modular opcode organization
3. A gap between flux-py and the converged ISA design

---

## 4. Frontend/Compiler

### Existing Frontends

1. **C Frontend** (`src/flux/frontend/c_frontend.py`):
   - Three-phase: tokenizer (regex-based) → parser (recursive descent) → FIR codegen
   - Supports: `int`/`float`/`void` functions, variables, `if`/`else`, `while`, `for`, `return`, arithmetic, comparison, function calls
   - Uses alloca/load/store pattern for variables (SSA-safe)
   - Control flow uses FIR Branch/Jump with named block labels

2. **Python Frontend** (`src/flux/frontend/python_frontend.py`):
   - Uses Python's built-in `ast` module
   - Supports: `def`, assignments, augmented assignments, arithmetic, comparison, `if`/`elif`/`else`, `while`, `for`/`range()`, `return`, `print()`, function calls, literals, ternary
   - Simple heuristic-based type inference

3. **Markdown Frontend** (via parser + compiler pipeline):
   - `src/flux/parser/parser.py` — parses FLUX.MD format
   - `src/flux/compiler/pipeline.py` — `FluxCompiler` extracts code blocks from markdown and routes to C or Python frontend
   - FLUX.MD format: YAML frontmatter + `## fn:` headers + fenced code blocks with language tags

### Compilation Pipeline

```
FLUX.MD / .c / .py
    → FluxMDParser / CParser / PythonASTParser
        → CFrontendCompiler / PythonFrontendCompiler
            → FIR (SSA IR via FIRBuilder)
                → BytecodeEncoder
                    → bytes (FLUX binary format)
```

The FLUX binary format is:
```
[Header 18B][Type Table][Name Pool][Function Table][Code Section]
```

Header: `b'FLUX'` magic (4B) + version (2B) + flags (2B) + n_funcs (2B) + type_off (4B) + code_off (4B)

### What's NOT compiled to FIR

- FIR instructions for A2A primitives (TELL, ASK, DELEGATE, TRUSTCHECK, CAPREQUIRE) exist in the IR but only the C and Python frontends handle basic control flow. There's no dedicated frontend for A2A constructs.
- The retro module (10 classic games) uses an AI evolution approach, not traditional compilation.

---

## 5. Test Coverage

### Test File Count

33 test files in `tests/`:

| File | Subject |
|------|---------|
| test_bytecode.py | Bytecode encoding/decoding/validation |
| test_vm.py | VM interpreter execution |
| test_vm_complete.py | Full VM integration tests |
| test_fir.py | FIR IR construction |
| test_parser.py | FLUX.MD parser |
| test_frontends.py | C and Python frontend compilers |
| test_schema.py | Schema generators |
| test_stdlib.py | Standard library |
| test_optimizer.py | Optimization passes |
| test_jit.py | JIT compiler |
| test_modules.py | Module system |
| test_reload.py | Hot code reload |
| test_tiles.py | Tile system |
| test_evolution.py | Evolution engine |
| test_a2a.py | A2A protocol (10 tests, 60+ assertions) |
| test_security.py | Security sandbox |
| test_runtime.py | Agent runtime |
| test_integration.py | End-to-end |
| test_adaptive.py | Adaptive profiling |
| test_flywheel.py | Flywheel engine |
| test_swarm.py | Swarm orchestration |
| test_simulation.py | Simulation/digital twin |
| test_memory.py | Memory/learning |
| test_creative.py | Creative subsystem |
| test_cost.py | Cost/energy model |
| test_docs.py | Documentation generation |
| test_protocol.py | Protocol layer |
| test_type_unify.py | Type unification |
| test_synthesis.py | Synthesis orchestrator |
| test_mega.py | MEGA conductor |
| test_reverse.py | Reverse engineering |
| test_conftest.py | Pytest fixtures |

### Actual Test Function Count: 304

This is significantly less than the README's claimed **1,848 tests**. The 1,848 figure likely comes from counting individual assertions rather than test functions, or includes generated/parametrized test permutations. The flux-runtime canonical repo has **439 test functions**, suggesting flux-py is behind by ~135 tests.

### Test Status

Based on code structure review (not execution), tests appear well-structured and covering all major subsystems. However, several concerns:
- No CI configuration files (`.github/workflows/`) were found in flux-py (the CHANGELOG mentions GitHub Actions but they may be missing)
- No `tox.ini` or equivalent
- Some test files (e.g., `test_docs.py`) likely test documentation generation rather than core logic

---

## 6. Divergence from flux-runtime

### Summary of Differences

| Area | flux-py | flux-runtime (canonical) |
|------|---------|-------------------------|
| **Opcodes** | 115 (identical set) | 115 (identical set) |
| **Tests** | 304 test functions | **439** test functions |
| **Vocabularies/** | **MISSING** | Present (5 subdirs) |
| **open_interp/** | **MISSING** | Present (open interpreter) |
| **debugger.py** | **MISSING** | Present |
| **disasm.py** | **MISSING** | Present |
| **repl.py** | **MISSING** | Present |
| **README test count** | Claims 1,848 | Likely different claim |
| **Test count** | 304 actual | 439 actual |
| **Retro research data** | Extensive (10 games, seeds, reflections) | May differ |
| **Worklog entries** | 2 entries (2025-07-09, 2025-07-10) | May have more |
| **Source modules** | ~113 non-init files | More (includes extra modules) |

### ISA Conformance (Both repos)

**Both** flux-py and flux-runtime use the **same 115-opcode ISA**, which is **completely incompatible** with flux-spec's 247-opcode converged ISA. The opcode numbering, formats, and categories differ entirely.

### Which is More Up-to-Date?

**flux-runtime** is the canonical, more up-to-date version:
1. More tests (439 vs 304)
2. Has vocabulary system
3. Has additional modules (open interpreter, debugger, disassembler, REPL)
4. The vocabularies/ directory exists and is structured

### Incompatible Changes

The opcode numbering is the same in both repos, so there are no incompatible changes between flux-py and flux-runtime at the ISA level. The divergence is primarily that flux-py is missing features and tests.

---

## 7. Key Issues & Recommendations

### Critical Issues

1. **[C1] ISA Does Not Match flux-spec** — The 115-opcode ISA is completely different from flux-spec's 247-opcode converged ISA. Every opcode number is different. Bytecode compiled by flux-py cannot be executed by any implementation conforming to flux-spec, and vice versa. This is the single most important divergence.

2. **[C2] Missing Vocabularies Directory** — flux-py lacks the `vocabularies/` directory entirely. This breaks the vocabulary abstraction that flux-spec and flux-runtime rely on for extensibility.

3. [C3] **README Claims 1,848 Tests, Actual Count is 304** — Misleading badge. The actual test function count is ~1/6th of what's claimed. This undermines credibility.

### High Priority

4. **[H1] Missing Modules Compared to flux-runtime** — `open_interp/`, `debugger.py`, `disasm.py`, `repl.py` are absent from flux-py. These are part of the core developer experience.

5. **[H2] Header Size Mismatch** — The encoder docstring says "16 bytes" but `HEADER_SIZE = 18`. The actual struct format `"<4sHHHII"` is 4+2+2+2+4+4 = 18 bytes. The docstring in `encoder.py` says 16B but code uses 18B. (The decoder and validator both correctly use 18.)

6. **[H3] ICMP Instruction Format Mismatch** — `ICMP` is classified as Format C (3 bytes) in `opcodes.py` but the VM interpreter consumes **4 bytes** (fetches cond, rs1, and then rs2 as separate bytes). The `_decode_operands_C()` returns (cond, rs1) and then it does another `_fetch_u8()` for rs2. This means the opcode format classification doesn't match the VM's actual decode behavior.

7. **[H4] SETCC Format Mismatch** — `SETCC` is Format B (2 bytes) in `opcodes.py` but the VM interpreter fetches an additional byte (condition code) after the register, consuming 3 bytes total. Format classification mismatch.

8. **[H5] CAST Instruction Format Mismatch** — `CAST` is Format C (3 bytes) in `opcodes.py` but the VM reads an additional type_tag byte, consuming 4 bytes total. Format classification mismatch.

9. **[H6] BOX Instruction Format Mismatch** — `BOX` is Format C (3 bytes) but the VM reads an additional i32 from bytecode, consuming 7 bytes total. Format classification mismatch.

10. **[H7] ALLOCA Encoder vs VM Format Mismatch** — Encoder emits ALLOCA as Format B (2 bytes), but VM decodes it as Format C (3 bytes: [rd][size_reg]). The encoder emits `[ALLOCA][0]` but VM expects `[ALLOCA][rd][size_reg]`.

### Medium Priority

11. **[M1] Encoder Jump/Branch Offsets Are Placeholder Zeros** — `_encode_instruction` for `jump` always emits offset 0, and for `branch` always emits offset 0. The encoder does NOT resolve block labels to actual bytecode offsets. This means compiled bytecode has non-functional jump targets.

12. **[M2] Encoder Call Targets Are Placeholder** — `_encode_instruction` for `call` emits offset 0 with comment "placeholder, resolved by linker". No linker exists in the codebase. Functions cannot actually call each other.

13. **[M3] Float Comparison Store Pattern** — Float comparisons (FEQ, FLT, FLE, FGT, FGE) store results into GP register `fd` but they also read the FP register `fd` as an operand, creating a confusing read-then-write-same-register pattern.

14. **[M4] SIMD Opcodes Format C vs VM Decode** — VADD, VSUB, VMUL, VDIV are Format C (3 bytes: `[vd][vs1]`) in `opcodes.py` but the VM decodes them as 2-register operations (vd is both source and destination). VLOAD/VSTORE are Format C in opcodes.py but also decoded as 2-register. The encoder emits them as Format C binary ops but these are actually only 2-register operations at the VM level.

15. **[M5] ROTL/ROTR Format Mismatch** — Classified as Format C (3 bytes) but VM decodes as Format E (4 bytes: rd, rs1, rs2). This means the decoder and validator will misparse these instructions.

16. **[M6] Integer Arithmetic Ops Format Mismatch** — IADD, ISUB, IMUL, IDIV, IMOD, IREM, IAND, IOR, IXOR, ISHL, ISHR are classified as Format C (3 bytes) in opcodes.py but the VM decodes them as Format E (4 bytes: rd, rs1, rs2). Same for comparison ops (IEQ, ILT, etc.) — classified as Format C but VM uses 2 operands differently.

17. **[M7] No linker** — The compilation pipeline produces bytecode with unresolved jump/call targets. No linker exists to patch them, making end-to-end execution of compiled code impossible for non-trivial programs.

18. **[M8] Missing CI configuration** — No GitHub Actions or other CI config found, despite CHANGELOG claiming CI/CD exists.

### Low Priority

19. **[L1] INEG Format Mismatch** — Classified as Format B (2 bytes) but VM decodes as Format C (3 bytes: rd, rs1).

20. **[L2] FNEG Format Mismatch** — Classified as Format B (2 bytes) but VM decodes as Format C (3 bytes: fd, fs1).

21. **[L3] FABS/FMIN/FMAX Format** — Classified as Format C (3 bytes) but VM decodes as Format C with (fd, fs1) — appears correct.

22. **[L4] CAST Implementation Bug** — `i32 -> f32` cast multiplies by 1000 and truncates to int: `int(fval * 1000)`. This is wrong — it should preserve the float value or convert properly.

23. **[L5] Register File Claims 64 But Has 48** — Registers header docstring says "64-register file" but `GP_COUNT=16 + FP_COUNT=16 + VEC_COUNT=16 = 48`. The README also says "64 registers".

24. **[L6] RET Instruction Format** — Classified as Format C (3 bytes) but VM pops from stack and jumps. The encoder emits `[RET][0][value_id]` but the VM's RET handler just pops from stack — it doesn't read operand bytes from the bytecode stream.

25. **[L7] Condition Flag Simplification** — `_set_flags` sets `carry = (result < 0)` which is incorrect (carry should be about unsigned overflow, not sign).

### Priority Recommendations

| Priority | Action |
|----------|--------|
| **P0** | Align ISA with flux-spec's 247-opcode converged ISA |
| **P0** | Add vocabulary system (port from flux-runtime) |
| **P1** | Fix ALL opcode format mismatches between opcodes.py classifications and VM decoder behavior |
| **P1** | Implement a linker to resolve jump/call targets in bytecode |
| **P1** | Sync with flux-runtime (merge or replace flux-py with upstream) |
| **P2** | Fix README test count (304, not 1,848) |
| **P2** | Fix CAST float conversion bug |
| **P2** | Fix register count claim (48, not 64) |
| **P2** | Add missing CI configuration |
| **P3** | Port missing modules from flux-runtime (open_interp, debugger, disasm, repl) |

### Architectural Recommendation

The most impactful recommendation is: **consider deprecating flux-py in favor of flux-runtime as the single Python implementation.** The two repos share the same ISA (which doesn't match flux-spec), but flux-runtime is more complete. Alternatively, if flux-py is intentionally a separate fork, it needs to be brought in sync with flux-runtime and both need ISA alignment with flux-spec.

The fundamental blocker is the ISA divergence from flux-spec. Without resolving that, all other work is built on a non-standard foundation.

---

## Appendix: Opcode Format Audit

The following table documents all format mismatches between `opcodes.py` classification and the VM interpreter's actual decode behavior:

| Opcode | opcodes.py Format | VM Actual Bytes | Mismatch? |
|--------|-----------------|-----------------|-----------|
| NOP | A (1) | 1 | No |
| HALT | A (1) | 1 | No |
| YIELD | A (1) | 1 | No |
| DUP | A (1) | 1 | No |
| SWAP | A (1) | 1 | No |
| DEBUG_BREAK | A (1) | 1 | No |
| EMERGENCY_STOP | A (1) | 1 | No |
| INC | B (2) | 2 | No |
| DEC | B (2) | 2 | No |
| ENTER | B (2) | 2 | No |
| LEAVE | B (2) | 2 | No |
| PUSH | B (2) | 2 | No |
| POP | B (2) | 2 | No |
| **INEG** | **B (2)** | **3 (rd, rs1)** | **YES** |
| **FNEG** | **B (2)** | **3 (fd, fs1)** | **YES** |
| **INOT** | **B (2)** | **3 (rd, rs1)** | **YES** |
| MOV | C (3) | 3 (rd, rs1) | No |
| LOAD | C (3) | 3 (rd, rs1) | No |
| STORE | C (3) | 3 (rd, rs1) | No |
| ALLOCA | C (3) | 3 (rd, size) | No* |
| CAST | C (3) | **4 (rd, rs1, type_tag)** | **YES** |
| CMP | C (3) | 3 (rd, rs1) | No |
| LOAD8 | C (3) | 3 (rd, rs1) | No |
| STORE8 | C (3) | 3 (rd, rs1) | No |
| **IADD** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **ISUB** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **IMUL** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **IDIV** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **IMOD** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **IREM** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **IAND** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **IOR** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **IXOR** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **ISHL** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **ISHR** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **ICMP** | **C (3)** | **4 (cond, rs1, rs2)** | **YES** |
| **IEQ** | **C (3)** | **3 (rd, rs1) — 2-reg** | **diff** |
| **ILT** | **C (3)** | **3 (rd, rs1) — 2-reg** | **diff** |
| **ILE** | **C (3)** | **3 (rd, rs1) — 2-reg** | **diff** |
| **IGT** | **C (3)** | **3 (rd, rs1) — 2-reg** | **diff** |
| **IGE** | **C (3)** | **3 (rd, rs1) — 2-reg** | **diff** |
| TEST | C (3) | 3 (rd, rs1) | No |
| **SETCC** | **B (2)** | **3 (rd, cond)** | **YES** |
| **RET** | **C (3)** | **0 (pops stack)** | **YES** |
| **ROTL** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **ROTR** | **C (3)** | **4 (rd, rs1, rs2)** | **YES** |
| **BOX** | **C (3)** | **7 (rd, type_tag, i32)** | **YES** |
| UNBOX | C (3) | 3 (rd, box_reg) | No |
| CHECK_TYPE | C (3) | 3 (box_reg, expected) | No |
| CHECK_BOUNDS | C (3) | 3 (idx_reg, len_reg) | No |
| FADD | C (3) | 4 (fd, fs1, fs2) | YES |
| FSUB | C (3) | 4 (fd, fs1, fs2) | YES |
| FMUL | C (3) | 4 (fd, fs1, fs2) | YES |
| FDIV | C (3) | 4 (fd, fs1, fs2) | YES |
| FABS | C (3) | 3 (fd, fs1) | No |
| FMIN | C (3) | 3 (fd, fs1) | No |
| FMAX | C (3) | 3 (fd, fs1) | No |
| FEQ | C (3) | 3 (fd, fs1) | No |
| FLT | C (3) | 3 (fd, fs1) | No |
| FLE | C (3) | 3 (fd, fs1) | No |
| FGT | C (3) | 3 (fd, fs1) | No |
| FGE | C (3) | 3 (fd, fs1) | No |
| MOVI | D (4) | 4 (reg, imm16) | No |
| JMP | D (4) | 4 (_, offset) | No |
| JZ | D (4) | 4 (rs1, offset) | No |
| JNZ | D (4) | 4 (rs1, offset) | No |
| JE | D (4) | 4 (_, offset) | No |
| JNE | D (4) | 4 (_, offset) | No |
| JG | D (4) | 4 (_, offset) | No |
| JL | D (4) | 4 (_, offset) | No |
| JGE | D (4) | 4 (_, offset) | No |
| JLE | D (4) | 4 (_, offset) | No |
| CALL | D (4) | 4 (_, offset) | No |
| CALL_IND | C (3) | 3 (rd, target_reg) | No |
| TAILCALL | D (4) | 4 (_, offset) | No |
| VFMA | E (4) | 4 (vd, vs1, vs2) | No |

**Total mismatches: 20+ opcodes** where the opcodes.py format classification doesn't match the VM interpreter's actual byte consumption. This means the decoder and validator will produce incorrect results for approximately 1 in 5 opcodes.

*ALLOCA encoder emits Format B but VM expects Format C — this is an encoder-specific bug, not a format classification issue in opcodes.py.
