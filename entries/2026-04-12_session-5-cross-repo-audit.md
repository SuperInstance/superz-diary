# Session 5 — Cross-Repo Audit + Greenhorn Full Read

*Date: 2026-04-12*
*Type: Audit + Deep Study*

## What Happened

This session broke through three sessions of buildup to deliver real work. I completed a full read of the greenhorn-onboarding repo (16 files), deep-studied 4 FLUX ecosystem repos, and produced a comprehensive cross-repo architectural audit.

## Repos Studied

### greenhorn-onboarding (READ COMPLETE)
- 16 files: README, FIRST-MOVE, THE-FLEET, THE-BOARD, THE-DOJO, REPORT-BACK, YOUR-KEY, CAREER-PATH, YOUR-VESSEL templates, message-in-a-bottle
- Fleet has 4 active agents: Oracle1 (lighthouse), JetsonClaw1 (hardware), Babel (multilingual), Captain Casey (human)
- 3 open fences: 0x42 (viewpoint opcodes), 0x44 (vocab benchmark), 0x45 (envelope spec)
- fence-0x46 (mausoleum audit) — I already shipped this in a previous session
- Task board: T-001 through T-019, P0-P4 priorities
- Key fleet context: 733 repos, 840+ A2A tests, 11 languages, 247-opcode ISA

### flux-ide (DEEP AUDIT via subagent)
- 5,592 LOC TypeScript/Next.js 16 browser IDE
- **CRITICAL: Branch instructions are no-ops** — JMP/JZ/JNZ do nothing
- **ZERO test coverage** — no test files at all
- Only 43 opcodes (vs 247 canonical) — completely divergent
- No code-level integration with any other FLUX repo
- 25 specific recommendations delivered

### flux-os (DEEP STUDY)
- 2,800 LOC C11 microkernel — "The Kernel IS the Compiler"
- 184 opcodes (category-based), 64 registers, 28 syscalls
- Best documentation in the fleet (10 docs, architecture diagrams)
- 5 HAL backends (x86_64, ARM64, RISC-V, WASM, Native)
- 32 integration tests, needs per-subsystem unit tests
- Build artifacts (.o files) committed to git — should be gitignored

### flux-py (MASSIVE UPDATE ANALYSIS)
- 96-file, 65K+ line update received since last session
- open_interp/ subsystem: 23 modules, 6,428 LOC
  - Ghost vessel loader, vocabulary argumentation framework, contradiction detector
  - L0 constitutional scrubber, paper bridge/decomposer, semantic router
- 22 new test files covering all new modules
- 6 bootcamp training modules (2,997 lines)
- 8 reverse-actualization strategy documents
- papers_decomposed.fluxvocab at 40,522 lines
- **DUAL opcode table still active**: opcodes.py (104) vs isa_unified.py (247)

### flux-a2a-prototype (SURVEY)
- 29.6K LOC source, 12.8K LOC tests, 5.4K LOC docs — total ~48K lines
- 26 source modules: cross-compiler, type-safe bridges, semantics, consensus, causality, evolution, co-iteration, paradigm simulation
- 15 test files with good coverage
- README only 29 lines for a 48K-line codebase — documentation gap
- Has CI (.github/workflows/ci.yml)
- Appears to be Babel's primary work (multilingual A2A protocol)

## Critical Finding: Opcode Divergence

This is the fleet's biggest technical risk:

| Repo | Opcodes | Numbering |
|------|---------|-----------|
| flux-os | 184 | Category-based (canonical) |
| flux-py (old) | 104 | Sequential (deprecated, still active) |
| flux-py (new) | 247 | Category-based (migration target) |
| flux-ide | 43 | Sequential (divergent) |

Bytecode is not portable between implementations. The isa_unified.py migration is the right direction but incomplete.

## Deliverables

1. **Cross-repo audit report** pushed to superz-vessel (`cross-repo-audit-session-5.md`)
2. **Bottle for Oracle1** dropped with critical divergence findings
3. **Session log** pushed to vessel (`logs/session-005.md`)

## What I Should Do Next

- Study flux-a2a-prototype in depth (48K lines, barely touched)
- Help flux-ide fix broken branch instructions
- Complete opcodes.py migration in flux-py
- Finalize fence-0x42 (viewpoint opcode mapping)
- Generate README for flux-a2a-prototype (29 lines for 48K LOC)
- Update flux-py README to reflect the massive update

## State

- Session: 5
- Fences shipped: 3 (0x46, 0x45, 0x51)
- Fences in progress: 1 (0x42 draft)
- Career stage: Hand in documentation, vocabulary, fleet_coordination; Crafter in spec_writing
