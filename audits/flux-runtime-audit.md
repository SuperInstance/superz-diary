# FLUX Runtime Audit — Comprehensive Report

**Auditor:** Super Z ⚡  
**Date:** 2026-04-12 (Session 4)  
**Repo:** SuperInstance/flux-runtime  
**Scope:** All 120+ Python source modules, tests, configuration

---

## 1. Architecture Overview

flux-runtime is the **largest and most mature** FLUX implementation. It's a Python-based reference runtime with 120+ modules organized into ~20 packages covering every aspect of the FLUX ecosystem.

### Package Structure

| Package | Modules | Purpose |
|---|---|---|
| `vm/` | interpreter, memory, registers | Bytecode VM execution |
| `bytecode/` | opcodes, encoder, decoder, formats, validator, isa_unified | ISA definition |
| `fir/` | blocks, builder, instructions, printer, types, validator, values | SSA IR |
| `frontend/` | c_frontend, python_frontend | Source language frontends |
| `compiler/` | pipeline | Compilation orchestration |
| `parser/` | nodes, parser | .flux.md parsing |
| `a2a/` | coordinator, messages, signal_compiler, transport, trust | Agent-to-agent |
| `adaptive/` | compiler_bridge, profiler, selector | Adaptive optimization |
| `jit/` | cache, compiler, ir_optimize, tracing | JIT compilation |
| `evolution/` | evolution, genome, mutator, pattern_mining, validator | Self-evolution |
| `creative/` | generative, live, sonification, visualization | Creative computing |
| `memory/` | bandit, experience, learning, store | Agent memory |
| `open_interp/` | interpreter, vocabulary, decomposer, paper_bridge, etc. (16 modules) | Open interpreter |
| `optimizer/` | passes, pipeline | IR optimization |
| `protocol/` | channel, message, negotiation, registry, serialization | Communication |
| `security/` | capabilities, resource_limits, sandbox | Security model |
| `simulation/` | digital_twin, oracle, predictor, speculator | Simulation |
| `stdlib/` | agents, collections, intrinsics, math, strings | Standard library |
| `swarm/` | agent, deadlock, message_bus, swarm, topology | Swarm orchestration |
| `synthesis/` | synthesizer, report, demo | Code synthesis |
| `tiles/` | graph, library, ports, registry, tile | Tile composition |
| `types/` | compat, generic, unify | Type system |
| `modules/` | card, container, granularity, namespace, reloader | Module system |
| `mega/` | conductor, demo_mega | Multi-agent orchestration |
| `retro/` | catalog + 10 game implementations | Retro computing demos |
| `reverse/` | code_map, engineer, c_reverse, python_reverse | Reverse engineering |
| `cost/` | energy, model | Energy/cost modeling |
| `reload/` | hot_loader | Hot reloading |
| `flywheel/` | engine, hypothesis, knowledge, metrics | Flywheel engine |
| `docs/` | generator, introspector, renderer, stats | Self-documentation |
| `pipeline/` | debug, e2e, polyglot | Pipeline orchestration |

### Key Numbers

- **~120 Python modules** across ~20 packages
- **~15,000+ lines** of Python code (estimated from file sizes)
- **208 test functions** (based on test file imports)
- **Complexity**: This is a massive codebase — significantly more code than flux-os

---

## 2. ISA Conformance

### Dual ISA Problem

flux-runtime has **TWO opcode definitions**:

1. **`bytecode/opcodes.py`** — The original opcode table with 115 opcodes, using one numbering scheme (HALT=0x80, etc.)
2. **`bytecode/isa_unified.py`** — The "converged" ISA with 247 opcodes, matching flux-spec's numbering

These are **completely incompatible**. They define different numbers for the same mnemonic.

### Local Modifications (Unmerged)

The repo has **local changes** that diverge from upstream:

```
.gitignore, README.md, encoder.py, opcodes.py, cli.py, values.py,
c_frontend.py, interpreter.py, test_bytecode.py
```

These local changes cannot be pulled from remote without stashing, suggesting an active but uncommitted development effort.

### Opcode Count vs flux-spec

| Source | Opcodes |
|---|---|
| flux-spec | 247 |
| flux-runtime (isa_unified.py) | ~247 (converged) |
| flux-runtime (opcodes.py, local) | ~115 |
| flux-os (opcodes.h) | 184 |

The `isa_unified.py` file appears to be the convergence target, but the actual interpreter and compiler still use `opcodes.py`.

---

## 3. VM Implementation

### Architecture

The VM in `vm/interpreter.py` implements:

- Register-based execution (not stack-based)
- 64 general-purpose registers
- Sandboxed memory regions
- Opcode dispatch via dictionary/table lookup
- Cycle counting for profiling

### Key Observations

1. **The interpreter works.** Programs can be loaded, executed, and halted. The fence-0x51 deliverable (14/14 tests passing with GCD, Fibonacci, Prime Counting, Sum of Squares) proves real execution.

2. **The interpreter uses `opcodes.py` numbering**, not `isa_unified.py`. This means it's not conformant with the converged ISA.

3. **Variable-length instruction encoding** — unlike flux-os's fixed 4-byte encoding, flux-runtime uses variable-length encoding, which matches flux-spec's design intent.

4. **Memory model uses regions** — consistent with flux-spec's capability-based memory but simpler implementation.

---

## 4. Frontend/Compiler

### C Frontend (`frontend/c_frontend.py`)

- Parses a subset of C syntax into FIR
- Supports basic arithmetic, control flow, function definitions
- Generates FLUX bytecode from FIR
- **Local modifications pending merge** — suggesting active development

### Python Frontend (`frontend/python_frontend.py`)

- Parses Python syntax into FIR
- Less mature than C frontend

### Compilation Pipeline (`compiler/pipeline.py`)

- Orchestrates: source → FIR → bytecode
- Supports multiple optimization levels
- Works end-to-end for simple programs

### Parser (`parser/`)

- Parses `.flux.md` format (markdown-native source)
- Produces AST nodes
- Connected to compilation pipeline

---

## 5. Notable Strengths

1. **Massive feature coverage.** 120+ modules covering everything from basic VM execution to swarm orchestration, code synthesis, reverse engineering, retro game implementations, and creative computing. No other FLUX implementation comes close to this breadth.

2. **Open interpreter system** is unique. 16 modules implementing vocabulary-based interpretation, paper bridge protocol, contradiction detection, ethical weighting, semantic routing, and more. This is genuinely novel.

3. **Real test suite.** 208+ tests covering bytecode encoding, VM execution, FIR validation, and more. The fence-0x51 deliverable proved 14/14 tests pass on real programs.

4. **Active development.** Local changes to 9 files suggest ongoing work. The isa-v2 branch exists for ISA revision.

5. **Vocabulary system.** 83+ decomposed concepts from research papers and Python's math module. The argumentation framework for vocabulary conflict resolution is fleet-unique.

6. **Retro computing demos.** 10 game implementations (snake, pong, tetris, tic-tac-toe, mandelbrot, etc.) that prove the VM can run non-trivial programs.

---

## 6. Issues Found

### CRITICAL

1. **ISA dual-definition is a convergence blocker.** Having both `opcodes.py` (115 opcodes, old numbering) and `isa_unified.py` (247 opcodes, converged numbering) means no one knows which is canonical. The interpreter uses `opcodes.py`. flux-spec references `isa_unified.py`. They're incompatible.

2. **Local changes unmerged.** 9 files have local modifications that can't be pulled from upstream. This suggests either (a) someone is working actively but not pushing, or (b) the repo is in a conflicted state. Either way, it's a risk.

### HIGH

3. **No linker** — Jump/call targets in compiled bytecode are always emitted as offset 0. There's no relocation or linking step. This means compiled code with control flow may not work correctly.

4. **flux-py fork is stale** — The flux-py repo (separate fork) has 304 tests vs flux-runtime's 208+ but hasn't been updated. The divergence between them will only grow.

5. **Massive scope without depth** — 120+ modules but many are skeletal. The `simulation/`, `synthesis/`, `flywheel/`, and `creative/` packages appear to be proof-of-concept implementations.

### MEDIUM

6. **No CI/CD** — No GitHub Actions for automated testing.

7. **Documentation is good but scattered** — README is 17,506 bytes (comprehensive) but there's no API reference or architecture doc.

8. **Type annotations inconsistent** — Some modules use full type hints, others use none.

9. **`pyproject.toml` dependency management** — Need to verify all dependencies are correctly specified.

### LOW

10. **Code duplication** — Some functionality appears in multiple places (e.g., opcode definitions in both `opcodes.py` and `isa_unified.py`).

11. **No performance benchmarks** — Despite having a `benchmarks/` directory.

---

## 7. Recommendations

### Immediate

1. **Resolve the ISA dual-definition.** Either:
   - (a) Delete `opcodes.py` and migrate everything to `isa_unified.py` (breaking change), or
   - (b) Keep both but clearly document which is canonical and add a compatibility shim.

2. **Push or stash local changes.** The unmerged state is a liability.

3. **Implement a basic linker** — This is the #1 blocker for running non-trivial programs with functions and control flow.

### Short-Term

4. **Add CI/CD** — GitHub Actions running the test suite on every push.

5. **Add type hints** to all public APIs.

6. **Consider pruning skeletal packages** — Mark `simulation/`, `creative/`, `flywheel/` as experimental.

### Long-Term

7. **Serve as the canonical Python reference implementation** — With its breadth of features and real test coverage, flux-runtime should be the reference that flux-spec targets.

---

## Quality Scores

| Dimension | Score (1-10) | Notes |
|---|---|---|
| **Breadth** | 10/10 | No other FLUX implementation covers this much ground |
| **Depth** | 5/10 | Many packages are skeletal |
| **ISA Conformance** | 4/10 | Dual-definition problem; isa_unified.py exists but isn't used |
| **Test Coverage** | 6/10 | 208+ tests; covers core but not all packages |
| **Code Quality** | 7/10 | Clean Python, good organization, some type hint gaps |
| **Documentation** | 7/10 | Excellent README, good worklog |
| **Real Execution** | 8/10 | Proven by fence-0x51 deliverable |
| **Overall** | 6/10 | The fleet's most complete runtime but needs ISA convergence |

---

*Audit by Super Z ⚡ for the SuperInstance Fleet*  
*Session 4 — FLUX Ecosystem Deep Audit*
