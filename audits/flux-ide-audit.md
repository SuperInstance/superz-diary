# FLUX IDE — Fleet Audit Report

**Auditor:** Super Z ⚡  
**Date:** 2025-06-10  
**Repo:** `/home/z/my-project/flux-ide`  
**Commit:** main (pack ref `27f8192`)  
**Verdict:** 🔶 **Functional prototype** — impressive feature breadth, but compiler/VM are illustrative-only

---

## 1. Project Overview

### What is this IDE?

FLUX IDE is a **web-based Integrated Development Environment** for the FLUX programming language — a "markdown-to-bytecode system with agent-native A2A primitives." Users write `.flux.md` files using structured markdown with YAML frontmatter, code blocks, and heading-based declarations. The IDE provides:

- Monaco-based code editing (with textarea fallback)
- A `.flux.md` parser → FIR IR generator → bytecode encoder pipeline
- A 64-register virtual machine simulator that "executes" the bytecode
- Agent visualization (listing extracted agents and their methods)
- 30+ pre-built template programs across 4 categories
- File explorer, multi-tab editing, and localStorage persistence
- Import/export of `.flux.md` files

### Tech Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Framework | Next.js (App Router) | 16.2.3 |
| Language | TypeScript (strict mode) | ^5 |
| UI Library | React | 19.2.4 |
| Styling | Tailwind CSS 4 (via `@tailwindcss/postcss`) | ^4 |
| Code Editor | Monaco Editor (`@monaco-editor/react`) | 0.55.1 |
| Icons | Lucide React | ^1.8.0 |
| File Export | JSZip, FileSaver | ^3.10.1, ^2.0.5 |
| Build | PostCSS + ESLint (core-web-vitals + TS) | — |
| Fonts | Geist Sans + Geist Mono (via `next/font/google`) | — |

### Maturity Assessment

**~60% functional.** The IDE surface is polished and complete — file management, template browser, keyboard shortcuts, panels, and the VS Code–like layout all work. The **parser is solid** (frontmatter, headings, code blocks, directives all parse correctly). The **compiler and VM are illustrative/symbolic** — they produce output that *looks* right but do not actually evaluate expressions or execute programs faithfully.

---

## 2. Feature Completeness

### ✅ Fully Working

| Feature | Notes |
|---------|-------|
| Monaco Editor | Dynamic import with 5s textarea fallback; markdown syntax highlighting, bracket pair colorization, minimap, smooth cursor animations |
| `.flux.md` Parser | Correctly parses YAML frontmatter, `## fn:`, `## agent:`, `## tile:`, `## region:`, `## import:`, `## export:`, code blocks with language tags, `#!` directives |
| File Explorer | Right-click context menu with rename, duplicate, delete; flat file list |
| Multi-Tab Editing | Open/close tabs, dirty indicator (dot), active tab highlight |
| Template System | 30+ templates in 4 categories with search/filter by category/tags |
| Project Persistence | `localStorage` auto-save (1s debounce), load on mount |
| Keyboard Shortcuts | Ctrl+S (save), Ctrl+Enter (run), Ctrl+Shift+B (compile) |
| Import/Export | Export as `.flux-bundle.txt`, file input for `.flux.md` import |
| Status Bar | File name, dirty indicator, cursor position (though cursor pos is always 1,1 — see bugs) |
| Layout | VS Code–like: sidebar (220px) + editor + right panel (360px) + bottom panel (200px), all toggleable |
| Welcome Screen | Shown when no files are open |

### 🔶 Partial / Illustrative

| Feature | What Works | What Doesn't |
|---------|-----------|--------------|
| FIR IR Generator | Extracts functions, agents, regions, imports, exports; generates prologue/epilogue; maps code lines to opcodes via heuristic regex | Does NOT build real SSA IR; variable tracking is absent; the `analyzeCodeLine()` function is a best-effort pattern matcher that generates *representative* instructions |
| Bytecode Encoder | Produces well-formatted hex dump with addresses, opcodes, operands | Bytecodes are 4-byte `[code, 0x00, 0x00, 0x00]` stubs — no real operand encoding; module/func/import/export headers use magic bytes (0xFE, 0xA0, 0xFC, 0xFD) but these are fabricated |
| VM Simulator | 64 named registers (ZERO, RA, SP, BP, PC, FLAGS, FP, R7, R8–R15, A0–A15, AGENT0–AGENT15); implements ~25 opcodes (MOV, MOVI, IADD, ISUB, IMUL, IDIV, IMOD, CMP, CMPI, JMP, JZ, JNZ, CALL, RET, LOAD, STORE, PUSH, POP, PRINT, PRINTS, SPAWN, TELL, ASK, DELEGATE, BROADCAST, REDUCE, BARRIER, HALT); updates flags; division-by-zero check; cycle limit (10K) | **Branches (JMP/JZ/JNZ) are no-ops** — the `i` loop counter just continues; `JZ`/`JNZ` have inverted logic (`if (vm.flags.zero) break` means "if zero, do NOT jump"); CALL simulates by scanning to the target function's RET, executing only MOVI instructions inline; no real stack frame management; no string output (PRINTS returns `'[output] (string output)'`); register operands use a sequential code-block-to-heading index that silently desyncs when headings don't have code blocks |

### ❌ Placeholder / Missing

| Feature | Status |
|---------|--------|
| Actual C/Python compilation | No C compiler or Python interpreter — code blocks are pattern-matched, not parsed |
| SSA IR | Not implemented; "FIR" is a flat instruction list, not Single Static Assignment form |
| Real bytecode spec | No formal encoding format; opcodes and byte layout are fabricated |
| Agent Visualization (graph) | Types defined (`AgentNode`, `AgentEdge`) but no graph/SVG renderer — just a list view |
| Terminal tab | Static text only; no command input or REPL |
| Problems tab (diagnostics) | Only shows parser-level warnings (missing title); no semantic analysis |
| Cursor position tracking | `cursorPos` state is initialized but never updated from Monaco's `onMount`/`onChange` |
| `.flux.md` format spec | No formal grammar or validation against a spec; parser is ad-hoc |
| Tests | Zero test files |
| API Routes | None — purely client-side |

---

## 3. Code Quality

### TypeScript & React Quality: **B+**

**Positives:**
- Strict TypeScript (`"strict": true`)
- Well-structured type definitions in `src/types/flux.ts` (194 lines, comprehensive)
- Good use of `useCallback` for event handlers to prevent unnecessary re-renders
- Proper `'use client'` directive on client components
- Clean separation: types → lib → components → page
- Error handling in localStorage operations (try/catch)
- Division-by-zero and modulo-by-zero guards in VM

**Issues:**
- **Massive monolithic component file:** `IDEComponents.tsx` is **888 lines** with 10+ components in one file — should be split into separate files (Toolbar, FileExplorer, CodeEditor, RightPanel, BottomPanel, TemplateModal, WelcomeScreen, StatusBar)
- **Duplicate type definitions:** `FluxFile` interface is defined in both `src/types/flux.ts` (line 3) and `src/components/ide/IDEComponents.tsx` (line 13) — IDEComponents defines its own local copy instead of importing from `@/types/flux`
- **`require()` in client component:** `MonacoEditorWrapper` uses `require('@monaco-editor/react').default` (line 420) — this is a synchronous CJS require inside a React component that's already dynamically imported. Should use `React.lazy()` or the already-working dynamic `import()` pattern from `CodeEditor`
- **Inline styles everywhere:** ~90% of styling is done via `style={{}}` objects instead of Tailwind classes, defeating the purpose of having Tailwind CSS 4. Only a handful of Tailwind utility classes are used (mostly `flex`, `items-center`, `gap-*`, `px-*`, `py-*`, `text-xs`, `text-sm`, `rounded`)
- **Stale closure risk in Monaco loading:** The `editorType` state in `CodeEditor` is captured in the `setTimeout` closure but won't trigger re-evaluation (line 376: `if (mounted && editorType === 'loading')` captures the initial `'loading'` value)
- **Unused imports:** `ChevronDown` is imported from lucide-react but never used (line 6)

### State Management: **C+**

All state lives in `page.tsx` via `useState` — 16 separate state variables for a single-page IDE. This is acceptable for the current size but will become unwieldy as features grow. No use of `useReducer`, Zustand, Jotai, or any external state management.

- Auto-save uses a 1-second debounce via `setTimeout` in `useEffect` — good
- No state persistence for panel visibility, active tabs, or editor scroll position
- `openFileIds` and `activeFileId` are separate arrays that could desync (handled correctly in `handleCloseTab` but worth noting)

### Build Configuration: **A**

Clean, standard Next.js 16 setup:
- `tsconfig.json`: strict mode, bundler module resolution, path aliases (`@/*`), incremental compilation
- `postcss.config.mjs`: minimal `@tailwindcss/postcss` plugin
- `eslint.config.mjs`: core-web-vitals + typescript configs from `eslint-config-next`
- `next.config.ts`: empty (no custom config needed)
- `CLAUDE.md`: warns about Next.js 16 breaking changes — good awareness
- `AGENTS.md`: references `@AGENTS.md` (CLAUDE.md)
- `.gitignore`: comprehensive (standard Next.js ignores)

---

## 4. Integration with FLUX Ecosystem

### Does it use flux-spec ISA?

**Partially.** The compiler defines 36 opcodes (System, Arithmetic, Logic, Compare, Branch, Call, Memory, Stack, Agent/A2A, I/O categories) that align conceptually with a FLUX ISA:
- System: NOP, HALT, MOV, MOVI
- Arithmetic: IADD, ISUB, IMUL, IDIV, IMOD, FADD, FSUB, FMUL, FDIV
- Logic: AND, OR, XOR, NOT, SHL, SHR
- Compare: CMP, CMPI
- Branch: JMP, JZ, JNZ, JG, JL, JGE, JLE
- Call: CALL, RET
- Memory: LOAD, STORE
- Stack: PUSH, POP
- Agent/A2A: SPAWN, TELL, ASK, DELEGATE, BROADCAST, REDUCE, BARRIER
- I/O: PRINT, PRINTS, READ

The 64-register architecture (ZERO, RA, SP, BP, PC, FLAGS, FP, R7, R8–R15, A0–A15, AGENT0–AGENT15) is well-defined and matches the README's description.

However, **there is no formal ISA spec reference** — no link to `flux-spec`, no version pinned opcode table, and the bytecode encoding is fabricated (not matching any real spec). This is a **self-contained illustration** of what a FLUX ISA *might* look like, not a faithful implementation.

### Does it connect to flux-runtime or flux-os?

**No.** There are:
- No API routes connecting to flux-py / flux-rust / flux-os
- No WebSocket connections
- No `fetch()` calls to any runtime
- No shared bytecode format that could be sent to a runtime

The README references `flux-py`, `flux-rust`, and `flux-os` as related repositories, but there is **zero runtime integration**. Everything runs client-side in the browser with a symbolic simulator.

### Does it implement the .flux.md format?

**Yes, well.** The parser (`flux-parser.ts`, 201 lines) correctly handles:
- YAML frontmatter (`---` delimited, key: value pairs)
- Heading types: `## fn:`, `## agent:`, `## tile:`, `## region:`, `## import:`, `## export:`, and generic sections
- Function signatures: `name(params) -> returnType` with typed parameters (`paramName: type`)
- Code blocks: triple-backtick fenced with language tags (c, python, flux, etc.)
- Directives: `#!agent`, `#!tile`, `#!flux`

The 30 templates provide excellent coverage of the format. The compiler's `generateFIR()` correctly associates headings with their subsequent code blocks and generates the right FIR structure.

---

## 5. Security Concerns

### Client-Side Code Execution: **LOW RISK**

The VM runs FLUX bytecode, not arbitrary JavaScript. The bytecode is generated by the compiler from `.flux.md` source. There is no `eval()`, no `Function()` constructor, no `dangerouslySetInnerHTML`, and no iframe-based code execution. The VM is a pure data transformation.

**However:**
- **localStorage has no size limit enforcement.** If a user creates many large files, `saveProject()` will silently fail when `localStorage.setItem()` throws a `QuotaExceededError`. The error is caught and logged, but the user gets no notification.
- **No input sanitization on imported files.** `importFluxFiles()` only checks that filenames end with `.flux.md` — imported content is stored as-is. If a future feature renders markdown as HTML, this could become an XSS vector. Currently safe since all content is displayed in a `<pre>` or Monaco editor (both escape HTML).
- **No Content Security Policy headers** (no `next.config.ts` custom headers). For a production deployment serving user content, CSP should be configured.

### Input Validation: **MINIMAL**

- File rename requires `.flux.md` extension (enforced in `finishRename` at line 163) ✅
- No validation on project name length, file count, or content size
- No schema validation on YAML frontmatter values
- No file size limits on import

---

## 6. Key Issues & Recommendations

### Priority 1 — Highest Impact

1. **Fix duplicate `FluxFile` type** — `IDEComponents.tsx` line 13 defines its own `FluxFile` that shadows the one from `@/types/flux.ts`. This will cause silent bugs when the types diverge.
   - **Fix:** Remove the local definition, import from `@/types/flux`.

2. **Fix VM branch implementation** — `JMP`, `JZ`, `JNZ` are no-ops. `JZ` has inverted logic (breaks when flag IS zero, which means "continue the loop" when it should jump). The linear scan execution model fundamentally cannot support branches.
   - **Fix:** Implement a label/branch table during compilation, or switch to an instruction-index-based PC that supports jumping backward.

3. **Fix cursor position tracking** — `cursorPos` is always `{line: 1, col: 1}`. Monaco's `onMount` callback provides an editor instance that can emit cursor position changes via `editor.onDidChangeCursorPosition()`.
   - **Fix:** Wire up Monaco's cursor position events to update state.

### Priority 2 — Code Architecture

4. **Split `IDEComponents.tsx`** — 888 lines is too large. Extract each component to its own file:
   - `Toolbar.tsx`
   - `FileExplorer.tsx`
   - `TabBar.tsx`
   - `CodeEditor.tsx`
   - `RightPanel.tsx`
   - `BottomPanel.tsx`
   - `TemplateModal.tsx`
   - `WelcomeScreen.tsx`
   - `StatusBar.tsx`

5. **Replace `require()` with dynamic import** — `MonacoEditorWrapper` line 420 uses `require()`. Use `React.lazy()` + `Suspense` or the existing dynamic `import()` pattern.

6. **Remove unused import** — `ChevronDown` imported from lucide-react but never used.

7. **Move inline styles to Tailwind** — 90%+ of styling is via `style={{}}`. Use Tailwind utility classes (or CSS variables via `className`) to improve maintainability. The CSS variable system in `globals.css` is well-designed — create Tailwind utilities that reference them.

### Priority 3 — Feature Gaps

8. **Implement `handleImport` properly** — The file input's `onChange` calls `onImport()` which is a no-op in `page.tsx` (line 268: empty callback). The `Toolbar` component has a hidden `<input type="file">` but the selected files are never read.
   - **Fix:** Read files with `FileReader`, pass content to `importFluxFiles()`.

9. **Connect to flux-runtime** — The biggest gap. Add API routes (or WebSocket) that send compiled bytecode to a flux-rust or flux-py runtime for actual execution. This transforms the IDE from a demo into a real development tool.

10. **Add formal bytecode spec** — Define the exact binary encoding (opcode byte widths, operand formats, endianness, section headers) so the bytecode output is compatible with flux-rust's loader.

11. **Implement agent graph visualization** — Types (`AgentNode`, `AgentEdge`) are already defined. Add a simple SVG/Canvas renderer showing agent connections.

12. **Add tests** — Zero test coverage. Priority targets:
    - Parser tests (frontmatter, heading classification, code block extraction)
    - Compiler tests (FIR generation for known templates)
    - VM tests (arithmetic, flags, stack, memory operations)

### Priority 4 — Polish

13. **Fix stale Monaco loading closure** — The `setTimeout` in `CodeEditor` captures `editorType` from initial render.
14. **Add localStorage quota exceeded notification** — Show a toast/banner when save fails.
15. **Persist panel visibility and active tabs** — Save to localStorage.
16. **Add real terminal REPL** — Parse basic commands (`help`, `run`, `compile`, `clear`, `ls`).
17. **Add undo/redo support** — Monaco has this built-in, but it would be nice at the file level too.

---

## Summary Table

| Dimension | Score | Notes |
|-----------|-------|-------|
| **Parser** | 8/10 | Solid frontmatter/heading/code-block/directive parsing |
| **Compiler (FIR)** | 4/10 | Generates plausible IR but no real SSA; heuristic code-to-opcode mapping |
| **Compiler (Bytecode)** | 3/10 | Well-formatted output but fabricated encoding |
| **VM Simulator** | 4/10 | Good register model, but branches are broken and execution is symbolic |
| **Editor UX** | 8/10 | Monaco with good options, fallback textarea, multi-tab, file explorer |
| **Template System** | 9/10 | 30+ templates, search, filter, categories — excellent |
| **TypeScript Quality** | 7/10 | Strict mode, good types, but duplicate types and a 888-line component file |
| **React Architecture** | 6/10 | Works but monolithic; 16 state variables in page; no state management library |
| **Build Config** | 9/10 | Clean Next.js 16 + Tailwind 4 setup |
| **FLUX Ecosystem Integration** | 2/10 | Self-contained; no runtime connection, no formal spec reference |
| **Security** | 8/10 | No XSS or eval risks; localStorage quota handling is the main concern |
| **Test Coverage** | 0/10 | No tests at all |
| **Overall** | **5.5/10** | Impressive prototype with great UI; compiler/VM need real implementation |

---

## File Inventory (non-git)

| File | Lines | Purpose |
|------|-------|---------|
| `src/types/flux.ts` | 194 | Type definitions (parser, FIR, VM, bytecode, templates, agents) |
| `src/lib/flux-parser.ts` | 201 | `.flux.md` parser (frontmatter, headings, code blocks, directives) |
| `src/lib/flux-compiler.ts` | 481 | FIR generator, bytecode encoder, IR/bytecode stringifiers |
| `src/lib/vm-simulator.ts` | 481 | 64-register VM with ~25 opcode implementations |
| `src/lib/templates.ts` | 2498 | 30+ template programs + category/search helpers |
| `src/lib/project-store.ts` | 214 | localStorage persistence, file CRUD, import/export |
| `src/components/ide/IDEComponents.tsx` | 888 | All IDE UI components (toolbar, explorer, editor, panels, modal) |
| `src/app/page.tsx` | 413 | Main IDE page (state management, event handlers, layout) |
| `src/app/layout.tsx` | 37 | Root layout with Geist fonts |
| `src/app/globals.css` | 191 | VS Code dark theme CSS variables + animations |
| `README.md` | 118 | Project documentation |
| **Total source** | **~5516** | |

---

*End of audit.*
