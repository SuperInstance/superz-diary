# FLUX Ecosystem Audit — Work in Progress

*Started: 2026-04-11 (previous session)*
*Status: Partial — needs continuation*
*Priority: High*

---

## Context

Across multiple previous sessions, Casey asked me to audit all FLUX-related repos across the SuperInstance and Lucineer GitHub orgs. The fleet had a coding agent working overnight with "pages of schemas and ideas" and the repos went in many directions. Casey explicitly said: *"you may need to start new repos instead of completely refactoring old ones that other agents took in different directions. but if you can find the synergy and synthesize the best combination of ideas, do it."*

## Repos to Audit

### SuperInstance Org
1. **flux-os** — C OS-level FLUX implementation. Partially implemented.
2. **flux-runtime** — Python. Has `isa-v2` branch. Massive new additions (9,026+ new lines). Contains radical new concepts: Vocabulary Immune System, Ghost Vessels, Paper Bridge.
3. **flux-py** — Python. COMPLETELY rewritten from flux-runtime. Different architecture, different approach.
4. **flux-ide** — IDE repo. Newly created, was only a concept before.

### Lucineer Org
5. **flux-spec** — Specification documents.
6. **lucineer** — Branding/profile repo.

### What We Know From Previous Sessions

**flux-runtime (Python)** contains radical new concepts from the overnight agent:
- **Vocabulary Immune System** — Mechanism to reject invalid vocabulary extensions
- **Ghost Vessels** — Agent instances that persist after the original disconnects
- **Paper Bridge** — Translation layer between different vocabulary domains
- **ISA v2 branch** — Suggests the instruction set architecture is being revised (fixed 4-byte, unified 3-operand format)

**flux-py** is a completely different rewrite from flux-runtime, not an evolution. Different agent took it in a different direction.

**flux-py vs flux-runtime** divergence is the key problem. Both claim to be "FLUX" but they may have incompatible ISAs, different VM designs, and different philosophies.

## What Needs To Happen Next

1. **Clone all 6 repos fresh** (previous clones are gone between sessions)
2. **Read the key source files** in flux-runtime (especially the new concepts) and flux-py
3. **Compare ISAs** — Are the bytecodes compatible? Do the same opcodes mean the same things?
4. **Evaluate the new concepts** — Are Vocabulary Immune System, Ghost Vessels, and Paper Bridge valuable? Are they implemented or just documented?
5. **Read flux-spec** — Is there a canonical specification that resolves the divergence?
6. **Make a call** — Synthesize or start fresh?

## Decision Framework

- If flux-runtime and flux-py share a common ISA core → **synthesize**: take the best ideas from both, create a unified repo
- If they've diverged too far → **start fresh**: create a new clean repo with the best ideas from both
- The ISA v2 branch may be the answer — if it represents the canonical direction, everything should align to it

## Open Questions

- Is flux-spec the canonical spec? Or is it also aspirational?
- What is the current state of flux-ide? Can it be a unifying tool?
- Should the C implementation (flux-os) be the reference implementation since it's closest to the metal?
- How many FLUX repos actually need to exist? Is 14 too many?

## Files to Read (for next session)

```
flux-runtime/
  src/flux/         — core runtime code
  docs/research/    — research roadmaps
  isa-v2 branch     — revised instruction set

flux-py/
  (entire repo)     — understand the rewrite

flux-spec/
  (entire repo)     — the canonical specification

flux-ide/
  (entire repo)     — IDE state

flux-os/
  (entire repo)     — C implementation
```

⚡
