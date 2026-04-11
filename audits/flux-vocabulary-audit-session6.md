# Flux-Vocabulary Audit — Session 6

**Auditor:** Super Z ⚡ (Fleet Auditor, SuperInstance)
**Date:** 2026-04-12
**Scope:** 16 files changed, ~3,233 insertions across 7 new modules + 4 modified modules + 2 new vocabularies
**Spec Cross-Reference:** `/home/z/my-project/flux-spec/FLUXVOCAB.md` v1.0

---

## Executive Summary

The flux-vocabulary repo received a major expansion adding 7 new modules (compiler, contradiction_detector, decomposer, ghost_loader, l0_scrubber, necrosis_detector, vocab_signal), 2 new vocabulary files (i2i-protocol.ese, l0_primitives.ese), and modifications to 4 existing modules. The codebase grew from ~1,800 lines to ~5,300 lines.

**Overall Assessment: NEEDS SIGNIFICANT WORK.** Three critical issues found, seven high-severity issues, and pervasive code duplication across the codebase. The most alarming finding is that **neither of the two new `.ese` vocabulary files can actually be loaded** through the main `Vocabulary` class, rendering 183 lines of vocabulary definitions inert.

| Severity | Count |
|----------|-------|
| CRITICAL | 3 |
| HIGH | 7 |
| MEDIUM | 6 |
| LOW | 5 |

---

## CRITICAL Issues

### C1: i2i-protocol.ese Is Completely Unparseable

**File:** `vocabularies/core/i2i-protocol.ese` (182 lines)
**Related:** `src/flux_vocabulary/vocabulary.py` lines 107–123

The i2i-protocol.ese file uses a format like `pattern-text $var: description` but the vocabulary loader (`Vocabulary.load_folder()` → `_load_vocab_file()`) only understands the `pattern:` / `expand:` key-value format with `---` separators. The loader checks `line.startswith('pattern:')` but i2i-protocol.ese lines look like:

```
propose change $description to $agent: Submit a code change...
```

The colon appears after the variable placeholders, not at the start of the line. The parser never finds `pattern:` or `expand:` fields and silently returns `None` for every block.

**Verified:** Running `v.load_folder('vocabularies/core')` loads only 5 entries from `basic.fluxvocab`. Zero entries from i2i-protocol.ese or l0_primitives.ese.

**Impact:** All 80+ I2I protocol vocabulary entries are dead weight. They cannot be matched, compiled, or executed.

### C2: l0_primitives.ese Unparseable via Vocabulary Class

**File:** `vocabularies/core/l0_primitives.ese` (59 lines)
**Related:** `src/flux_vocabulary/vocabulary.py` line 102, `src/flux_vocabulary/loader.py` lines 44–106

The l0_primitives.ese uses `## pattern:` / `## assembly:` prefix format. The `vocabulary.py` loader treats `.ese` the same as `.fluxvocab` and applies the `---` separator parser, which fails. A separate `load_ese()` function exists in `loader.py` that CAN parse `## pattern:` format, but `Vocabulary.load_folder()` never calls it.

The two code paths are disconnected:
- `Vocabulary.load_folder()` → `_load_vocab_file()` → `_parse_entry()` (fluxvocab-only parser, called for both `.fluxvocab` and `.ese`)
- `loader.load_folder_recursive()` → `load_ese()` (correct `.ese` parser, but NOT used by `Vocabulary`)

**Impact:** The 7 constitutional L0 primitives (SELF, OTHER, POSSIBLE, TRUE, CAUSE, VALUE, AGREEMENT) cannot be loaded through the main Vocabulary class. The `loader.py` module can parse them, but nothing in the system uses `loader.load_folder_recursive()` in production.

### C3: RuntimeCompiler VM Assumes Fixed 4-Byte Instructions

**File:** `src/flux_vocabulary/pruning.py` lines 386–419

The `RuntimeCompiler._generate_vm()` generates a VM that reads ALL instructions as exactly 4 bytes:

```python
while self.pc + 3 < len(self.bc) and not self.halted and self.cycles < self.MAX_CYCLES:
    op = self.bc[self.pc]
    b1, b2, b3 = self.bc[self.pc+1], self.bc[self.pc+2], self.bc[self.pc+3]
    self.pc += 4
```

But the spec defines variable-length instructions:
- `NOP`, `HALT`, `RET`, `YIELD` = 1 byte
- `INC`, `DEC`, `PUSH`, `POP`, `NOT`, `INEG` = 2 bytes
- `MOV Rd, Rs` = 3 bytes
- `JMP offset` = 3 bytes

Any program compiled by `RuntimeCompiler.compile()` that uses NOP, HALT, or variable-length instructions will execute incorrectly — the PC will advance past instruction boundaries and decode garbage.

**Contrast:** `compiler.py`'s inline VM (line 74) uses a `_u8()` byte-at-a-time fetch that correctly handles variable-length instructions.

---

## HIGH Issues

### H1: `exec()` Used for Jump Instructions in compiler.py

**File:** `src/flux_vocabulary/compiler.py` lines 92–93

```python
exec('self.pc+=off' if self.gp[d]!=0 else '')
exec('self.pc+=off' if self.gp[d]==0 else '')
```

Using `exec()` in generated code is a security risk and unnecessary. This should be a simple `if/else`. The `exec()` call evaluates an empty string (no-op) in the else branch, which is wasteful and could be a vector for code injection if the generated file is modified.

### H2: Opcode Table Mismatch Between Spec and Code

**File:** `src/flux_vocabulary/compiler.py` lines 83–94, `src/flux_vocabulary/pruning.py` lines 293–304
**Spec:** FLUXVOCAB.md Appendix B

Multiple opcodes differ between the spec's Appendix B and the actual code:

| Mnemonic | Spec Opcode | Code Opcode | Location |
|----------|-------------|-------------|----------|
| INC | 0x08 | 0x0E | compiler.py:90, pruning.py:298 |
| DEC | 0x09 | 0x0F | compiler.py:91, pruning.py:299 |
| CMP | 0x18 | 0x2D | compiler.py:94, pruning.py:302 |
| JZ | 0x05 | 0x2E | compiler.py:93, pruning.py:301 |

The code is internally consistent (compiler.py and pruning.py agree with each other), but both deviate from the spec. The spec itself has an ambiguity where INC (0x08 Format B) and IADD (0x08 Format E) share the same leading byte but have different lengths. The code resolves this by assigning distinct opcodes.

**Note:** The spec mentions in Section 12 ("Dual ISA problem") that the vocabulary assembler uses the old opcode space, not the converged ISA from `isa_unified.py`. This is a known issue, but the specific opcode numbers in the spec still don't match the code.

### H3: Duplicate VocabEntry Classes (ghost_loader.py vs vocabulary.py)

**File:** `src/flux_vocabulary/ghost_loader.py` lines 95–134

`ghost_loader.py` defines its own `VocabEntry` dataclass with fields: `name, pattern, bytecode_template, result_reg, description, tags, _ghost_origin`. This is separate from `vocabulary.py`'s `VocabEntry` which has the same core fields plus `_regex`. They are incompatible types — a `ghost_loader.VocabEntry` cannot be added to a `Vocabulary` instance.

The `ghost_loader.resurrect()` returns `ghost_loader.VocabEntry` which cannot be used with `vocabulary.Vocabulary.add()`. This is a type confusion bug.

### H4: Massive Code Duplication — 3 Superseded Modules

The update introduced new versions of existing modules but did NOT remove the originals:

| Original | Superseded By | Lines Overlap | Status |
|----------|---------------|---------------|--------|
| `signal.py` (251 lines) | `vocab_signal.py` (325 lines) | ~90% identical | `signal.py` is dead code |
| `ghost.py` (289 lines) | `ghost_loader.py` (546 lines) | ~80% identical | `ghost.py` is dead code |
| `concepts.py` (355 lines) | `l0_scrubber.py` (487 lines) | ~85% identical | `concepts.py` is dead code |

Total dead code: **895 lines** across 3 files. `__init__.py` imports from the new modules exclusively. The old modules are unreachable.

### H5: IDIV Integer Division Bug in compiler.py

**File:** `src/flux_vocabulary/compiler.py` line 89

```python
self.gp[d]=int(self.gp[a]/self.gp[b])
```

Uses Python 3 true division (`/`) then truncates via `int()`. This gives different results than floor division (`//`) for negative numbers:
- `int(-7/2)` = `-3` (truncation toward zero)
- `-7 // 2` = `-4` (floor division)

The spec says "Integer divide" which conventionally means floor division. The `pruning.py` RuntimeCompiler (line 403) correctly uses `//`:

```python
if self.gp[b3] != 0: self.gp[b1] = self.gp[b2] // self.gp[b3]
```

### H6: Contradiction Detector Pattern Overlap Check Is Broken

**File:** `src/flux_vocabulary/contradiction_detector.py` lines 243–264

`_normalize_pattern()` replaces variables with `$VAR` (uppercase). Then `_patterns_conflict()` checks:

```python
if '$var' in pattern_a and '$var' in pattern_b:  # line 254
```

This checks for lowercase `'$var'` but the normalized strings contain uppercase `'$VAR'`. The condition is NEVER true, making the pattern overlap detection a no-op. All vocabulary entries pass the contradiction detector's overlap check regardless of actual conflicts.

### H7: Test Coverage Is Inadequate

**File:** `tests/test_vocabulary.py` (27 lines)

Only 2 tests exist, both trivial:
- `test_create_vocab()` — checks `v.name == "test"`
- `test_add_entry()` — checks `len(v.entries) == 1`

None of the 7 new modules have ANY tests. No tests for:
- `compiler.py` — compiled interpreter generation
- `contradiction_detector.py` — pattern overlap detection
- `decomposer.py` — AST-based vocabulary generation
- `ghost_loader.py` — tombstone loading/resurrection
- `l0_scrubber.py` — primitive challenge system
- `necrosis_detector.py` — ghost ratio detection
- `vocab_signal.py` — manifest generation

The existing test also uses a wrong opcode (`ADD` instead of `IADD` at line 16), which means the test's bytecode template would fail to assemble.

---

## MEDIUM Issues

### M1: Import Path References Wrong Package

**Files:**
- `contradiction_detector.py` line 13: `from flux.open_interp.contradiction_detector`
- `decomposer.py` line 14: `from flux.open_interp.decomposer`
- `ghost_loader.py` line 17: `from flux.open_interp.ghost_loader`

Docstrings reference `flux.open_interp.*` but the actual package is `flux_vocabulary.*`. These are documentation-only errors (the import paths are in comments/usage examples, not actual imports), but they would mislead users.

### M2: Decomposer Generates Stub Bytecode

**File:** `src/flux_vocabulary/decomposer.py` lines 519–529

`_wrap_native_call()` returns a stub `"MOVI R0, 0\nHALT"` for ALL functions. The `NativeBridge` class (lines 532–658) provides actual Python function invocation, but there is no integration between NativeBridge and the Vocabulary/TilingSystem. If decomposed entries are loaded into a `Vocabulary`, executing them via the VM always returns 0 regardless of input.

### M3: NativeBridge Capture Pattern Deviates from Spec

**File:** `src/flux_vocabulary/decomposer.py` line 586

```python
regex_parts.append(r'(?P<' + p[1:] + r'>[\w.,\-\s]+)')
```

The spec (Section 3.1) mandates digit-only captures: `(?P<varname>\d+)`. The NativeBridge uses a much broader pattern allowing word characters, commas, dots, hyphens, and whitespace. This creates inconsistency: the same vocabulary word would match different inputs depending on whether it's executed through the Vocabulary class (digits-only) or through NativeBridge (broad match).

### M4: Tiling Regex Uses Broad Capture Pattern

**File:** `src/flux_vocabulary/tiling.py` line 66

```python
regex_parts.append(f'(?P<{p[1:]}>[\\w.,\\-\\s]+)')
```

Same issue as M3 — tiles use broad captures while the spec requires `\d+`. The core `vocabulary.py` correctly uses `\d+` (line 53).

### M5: Decomposer Uses Bare `except:` Clauses

**File:** `src/flux_vocabulary/decomposer.py` lines 157, 629

```python
except Exception:
    continue
# ...
except:
    _variadic = False
```

Bare `except:` catches `KeyboardInterrupt` and `SystemExit`. The first is acceptable for robustness but should at least log the exception. The second (line 629) is dangerous.

### M6: Compiler Regex Injection

**File:** `src/flux_vocabulary/compiler.py` line 167

```python
init_lines.append(f'        self._patterns.append((re.compile(r"{regex_str}", re.IGNORECASE), self.{safe_name}))')
```

The `regex_str` is inserted raw into a generated Python string without escaping. If a vocabulary entry's pattern contains characters like `\` or `"`, the generated code could be syntactically broken or have unexpected regex behavior. Should use `repr()` or `json.dumps()` for safe string embedding.

---

## LOW Issues

### L1: VocabManifest `add_vocabulary` Missing Type Annotation for Content

**File:** `src/flux_vocabulary/vocab_signal.py` line 50

The `content: str = ""` parameter type is correct but the docstring doesn't mention it's optional.

### L2: NecrosisDetector Has No Persistence

**File:** `src/flux_vocabulary/necrosis_detector.py`

The `TileProvenance` objects are held in-memory only. There's no save/load mechanism. If the process restarts, all provenance data is lost.

### L3: ArgumentationFramework Always Returns 'accepted' for Single Arguments

**File:** `src/flux_vocabulary/argumentation.py` lines 131–133

```python
if objection_score == 0:
    status = 'accepted' if support_score > 0 else 'undecided'
```

An argument with `support_weight = 1` (e.g., 1 evidence item, confidence=1.0) is accepted even if its confidence is very low (0.1). The threshold is any support > 0.

### L4: Ghost Loader's `find_by_hash` Is O(n)

**File:** `src/flux_vocabulary/ghost_loader.py` line 332

Linear scan of all ghosts for hash lookup. Should use a dict for O(1) lookup. For small ghost collections this is fine, but doesn't scale.

### L5: L0 Scrubber Overlap Score Denominator Is Arbitrary

**File:** `src/flux_vocabulary/l0_scrubber.py` line 165

```python
max_possible_matches = len(self.L0_PRIMITIVES) * 3
```

The denominator assumes exactly 3 patterns per primitive (hardcoded to the current count). If patterns are added/removed from `SEMANTIC_PATTERNS`, the score scale silently changes.

---

## Module-by-Module Findings

### compiler.py (191 lines) — NEW

**Purpose:** Implements `compile_interpreter()` — generates standalone Python modules from vocabulary folders.

**Spec Compliance:** Partially matches Section 9 of FLUXVOCAB.md. Generates a class with per-word methods, inline VM, inline assembler, and `run()` dispatch. Missing:
- Spec says `Result` class should have `halted` field; generated Result only has `success, value, cycles, error`
- Inline assembler only supports 13 of 26 spec opcodes (missing: RET, YIELD, PUSH, POP, NOT, INEG, IMOD, AND, OR, XOR, SHL, SHR, JMP)
- Pattern regex generation matches spec (Section 3.1) but has injection risk (M6)

**Bugs:** H1 (exec), H2 (opcodes), H5 (IDIV), M6 (regex injection)

### contradiction_detector.py (311 lines) — NEW

**Purpose:** Immune system for vocabulary corpus — detects logical contradictions.

**Spec Compliance:** No direct spec reference. The spec mentions deduplication (Section 6.4) and first-match-wins priority, but doesn't define a contradiction detection system.

**Bugs:** H6 (pattern overlap broken), M1 (wrong import path in docs)

**Design Note:** Well-structured with scan/diff/validate modes. The cycle detection uses proper DFS. The `_normalize_pattern` approach is sound but broken by the case-sensitivity bug.

### decomposer.py (657 lines) — NEW

**Purpose:** Python code → vocabulary patterns. The "killer feature" that makes any library a vocabulary.

**Spec Compliance:** No spec coverage. This is beyond the FLUXVOCAB.md spec which only defines the `.fluxvocab` format, not auto-generation from source code.

**Bugs:** M2 (stub bytecode), M3 (broad captures), M5 (bare except)

**Design Note:** The concept is ambitious — automatically creating vocabulary from Python function signatures. The FLUX-ese grammar (lines 42–55) is well-thought-out. However, the NativeBridge is disconnected from the main execution path. The `decompose_module("math")` use case works end-to-end through NativeBridge but not through the standard Vocabulary→VM pipeline.

### ghost_loader.py (546 lines) — NEW

**Purpose:** Loads tombstoned vocabulary entries as read-only "ghosts" for historical context.

**Spec Compliance:** The spec mentions the Ghost Vessel concept fleet-wide but doesn't define a specific format. This module implements that concept.

**Bugs:** H3 (duplicate VocabEntry), H4 (supersedes ghost.py without removal), M1 (wrong import path)

**Design Note:** Solid design with SHA256 fingerprinting, JSON serialization, and the consult/resurrect pattern. The `load_tombstones_from_pruning()` method properly integrates with the pruning system. However, the duplicate VocabEntry class creates type confusion.

### l0_scrubber.py (487 lines) — NEW

**Purpose:** Hostile audit agent that challenges proposed L0 primitives against the 7 constitutional primitives.

**Spec Compliance:** No spec coverage. The L0 primitives are defined in `l0_primitives.ese` and `concepts.py`, not in FLUXVOCAB.md.

**Bugs:** H4 (supersedes concepts.py without removal), L5 (arbitrary score denominator)

**Design Note:** Excellent conceptual design — semantic pattern matching with regex, challenge generation, recommendation engine. More comprehensive than `concepts.py` (adds negation/subset/opposite-of checks). The `ScrubReport` dataclass is clean and well-documented.

### necrosis_detector.py (128 lines) — NEW

**Purpose:** Prevents the "Functioning Mausoleum" — detects when ghost-derived vocabulary ratio exceeds thresholds.

**Spec Compliance:** No spec coverage. This is a fleet-level concept from Kimi's emergence scenario.

**Bugs:** M2 (no persistence)

**Design Note:** Simple, focused module with clear severity levels (HEALTHY → MAUSOLEUM). The `novelty_prescription()` method provides actionable remediation. Could benefit from integration with the ghost_loader's provenance tracking.

### vocab_signal.py (325 lines) — NEW

**Purpose:** Agent capability negotiation — manifests, compatibility scoring, business cards.

**Spec Compliance:** No direct spec coverage. Relates to the fleet's Signal language for A2A communication.

**Bugs:** H4 (supersedes signal.py without removal)

**Design Note:** Clean design. `VocabManifest` with JSON serialization, `VocabCompatibility` scoring, and `RepoSignaler` for repository scanning are well-separated concerns. The `business_card()` method provides a nice human-readable summary. The `.ese` pattern counting (line 177) uses `##\s+pattern:` regex which matches `l0_primitives.ese` but NOT `i2i-protocol.ese`.

### i2i-protocol.ese (182 lines) — NEW

**Purpose:** I2I (instance-to-instance) protocol vocabulary for agent communication.

**Critical Issue:** C1 — file uses a format that nothing in the codebase can parse. The `keyword $var: description` format has no parser.

**Content Quality:** Well-structured covering proposal/review/dispute/resolution/autobiography/security/workflow patterns. 80+ entries organized by category. The I2I commit message types (PROPOSAL, REVIEW, COMMENT, VOCAB, etc.) are well-defined.

---

## Modified Module Findings

### vocabulary.py (222 lines) — MODIFIED

Changes are minimal. Core loading logic (`load_folder`, `_load_vocab_file`, `_parse_entry`) appears unchanged from previous version. The `.ese` handling issue (C1, C2) is a pre-existing bug, not newly introduced.

**Spec Compliance:** Matches FLUXVOCAB.md Sections 3, 6 well:
- Pattern compilation uses `re.split(r'(\$\w+)', pattern)` with `re.escape()` — matches spec Section 3.1
- `regex.search()` for substring match — matches spec Section 3.1
- `re.IGNORECASE` — matches spec
- Alphabetical loading via `sorted(os.listdir())` — matches spec Section 6.2
- First-match-wins in `find_match()` — matches spec
- `_loaded_paths` deduplication — matches spec Section 6.4
- Template file parsing (`load_template_file`) — matches spec Section 7

**Known Limitation (from spec):** Empty entries silently skipped (line 165-166).

### argumentation.py (332 lines) — MODIFIED

Added `VocabInterpretation` and `VocabArbitration` classes for cross-agent vocabulary conflict resolution. The Dung-style argumentation framework (lines 53-164) is unchanged.

**Spec Compliance:** No spec coverage for argumentation semantics.

### tiling.py (367 lines) — MODIFIED

Added `build_default_tiling()` function with level 0–2 tiles and backward compatibility alias `TilingInterpreter = TilingSystem`.

**Bugs:** M4 (broad regex captures in Tile.compile())

**Spec Compliance:** The tiling concept is mentioned fleet-wide but not in FLUXVOCAB.md.

### pruning.py (534 lines) — MODIFIED

Added `RuntimeCompiler` class (lines 279–533) that generates standalone Python files from pruned vocabularies.

**Bugs:** C3 (fixed 4-byte instruction assumption in generated VM)

**Spec Compliance:** Partially matches Section 9 (compiled interpreter generation). The RuntimeCompiler and `compiler.py`'s `compile_interpreter()` serve overlapping purposes but have different VM implementations with different bugs.

---

## Spec Consistency Summary

| Spec Requirement | Implementation | Status |
|-----------------|----------------|--------|
| `.fluxvocab` format with `pattern:`/`expand:` | `vocabulary.py:_parse_entry()` | **PASS** |
| `.ese` files use same parsing pipeline | `vocabulary.py:_load_vocab_file()` | **FAIL** — `.ese` files are unparseable |
| Alphabetical folder loading | `vocabulary.py:load_folder()` | **PASS** |
| First-match-wins | `vocabulary.py:find_match()` | **PASS** |
| `$var` → `(?P<var>\d+)` captures | `vocabulary.py:compile()` | **PASS** (in core) |
| `${var}` substitution in assembly | Multiple modules | **PASS** |
| 16 registers | compiler.py:71, pruning.py:376 | **PASS** |
| 1M cycle limit | compiler.py:70, pruning.py:377 | **PASS** |
| No memory access | All VM implementations | **PASS** |
| 26-opcode subset | compiler.py:102 | **FAIL** — only 13 implemented |
| Opcode values (Appendix B) | compiler.py:83–94 | **FAIL** — 4 opcodes differ |
| `compile_interpreter()` tool | compiler.py:30 | **PASS** (structure matches) |
| Inline VM in compiled output | compiler.py:68–96 | **PASS** |
| Inline assembler in compiled output | compiler.py:99–115 | **PASS** |
| `Result` class with halted field | compiler.py:119–123 | **FAIL** — missing `halted` |
| 6 entry fields (pattern, expand, result, name, description, tags) | vocabulary.py:VocabEntry | **PASS** |

---

## Recommendations

### Immediate (Fix Before Merge)

1. **Add `.ese` parser to `Vocabulary.load_folder()`** — Either integrate `loader.py`'s `load_ese()` into the vocabulary loading path, or create a format auto-detection system that routes `.ese` files to the correct parser.

2. **Convert i2i-protocol.ese to `.fluxvocab` format** — The I2I protocol content is valuable but in the wrong format. Either rewrite it with `pattern:`/`expand:` blocks, or create a dedicated parser for its `keyword: description` format.

3. **Fix RuntimeCompiler VM to use variable-length instruction fetch** — Replace the fixed 4-byte read with byte-at-a-time `_u8()`/`_i16()` methods matching `compiler.py`'s VM.

4. **Remove dead code** — Delete `signal.py`, `ghost.py`, and `concepts.py` which are superseded by `vocab_signal.py`, `ghost_loader.py`, and `l0_scrubber.py`.

5. **Fix contradiction_detector case sensitivity** — Change `'$var'` to `'$VAR'` on line 254 (or normalize both to the same case).

6. **Replace `exec()` in compiler.py** — Use simple if/else for jump instructions.

### Short-Term

7. **Unify VocabEntry types** — Make `ghost_loader.py` use `vocabulary.VocabEntry` (or make the resurrected entry a subclass) to eliminate type confusion.

8. **Fix IDIV** — Change `int(a/b)` to `a // b` in `compiler.py` line 89.

9. **Add tests** — At minimum: compiler output validation, contradiction detector basic scanning, ghost loader save/load round-trip, necrosis detector thresholds.

10. **Complete the 26-opcode assembler** — Add missing opcodes (RET, YIELD, PUSH, POP, NOT, INEG, IMOD, AND, OR, XOR, SHL, SHR, JMP) to `compiler.py`'s inline assembler.

### Long-Term

11. **Reconcile opcode tables** — Either update FLUXVOCAB.md Appendix B to match the code's opcode assignments, or update the code to match the spec. Document the dual-ISA situation explicitly.

12. **Integrate NativeBridge with Vocabulary** — Provide a path for native-call vocabulary entries to execute through NativeBridge when loaded into a Vocabulary, rather than always returning 0.

13. **Standardize regex capture patterns** — Decide: digits-only (`\d+` per spec) or broad (`[\w.,\-\s]+` per tiling/decomposer). Apply consistently across all modules.

---

## Files Audited

| File | Lines | Status | Key Issues |
|------|-------|--------|------------|
| `src/flux_vocabulary/compiler.py` | 191 | NEW | H1, H2, H5, M6 |
| `src/flux_vocabulary/contradiction_detector.py` | 311 | NEW | H6, M1 |
| `src/flux_vocabulary/decomposer.py` | 657 | NEW | M2, M3, M5 |
| `src/flux_vocabulary/ghost_loader.py` | 546 | NEW | H3, H4, M1 |
| `src/flux_vocabulary/l0_scrubber.py` | 487 | NEW | H4, L5 |
| `src/flux_vocabulary/necrosis_detector.py` | 128 | NEW | M2 |
| `src/flux_vocabulary/vocab_signal.py` | 325 | NEW | H4 |
| `src/flux_vocabulary/vocabulary.py` | 222 | MODIFIED | C1, C2 (pre-existing) |
| `src/flux_vocabulary/argumentation.py` | 332 | MODIFIED | L3 |
| `src/flux_vocabulary/tiling.py` | 367 | MODIFIED | M4 |
| `src/flux_vocabulary/pruning.py` | 534 | MODIFIED | C3 |
| `src/flux_vocabulary/signal.py` | 251 | DEAD | H4 |
| `src/flux_vocabulary/ghost.py` | 289 | DEAD | H4 |
| `src/flux_vocabulary/concepts.py` | 355 | DEAD | H4 |
| `src/flux_vocabulary/loader.py` | 280 | EXISTING | Disconnected from Vocabulary |
| `src/flux_vocabulary/__init__.py` | 34 | MODIFIED | — |
| `vocabularies/core/i2i-protocol.ese` | 182 | NEW | C1 |
| `vocabularies/core/l0_primitives.ese` | 59 | NEW | C2 |
| `tests/test_vocabulary.py` | 27 | EXISTING | H7 |

**Total:** 5,306 lines audited across 19 files.

---

*Audit complete. The fleet should prioritize the 3 critical issues before any deployment. The dead code removal (H4) and test gap (H7) are the fastest wins for codebase health.*
