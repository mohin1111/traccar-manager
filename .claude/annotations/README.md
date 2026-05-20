# Annotation Layer

This directory is a **read-only knowledge layer** for the source tree. It mirrors the structure of the source so each `.dart` file has a sibling `<filename>.md` describing it in LLM-friendly terms.

## Why a parallel tree?

- **Upstream-sync clean.** Upstream Traccar releases regularly. If we commented inside the source files, every upstream pull would conflict in every commented file. The annotation layer lives outside the upstream paths, so `git pull upstream main` merges without friction.
- **Higher signal than line comments.** Annotations focus on the *why*, the *where it fits*, the *cross-references*, and the *gotchas* — what an LLM (or new contributor) needs and what good code can't self-document.
- **Cheap to regenerate.** When a file is heavily refactored upstream, re-read it and rewrite the annotation. No painful merge.

## Structure

```
.claude/annotations/
├── README.md          ← this file — convention + how to use
└── lib/
    ├── PACKAGE.md     ← module overview, file index, dependency graph, flows
    └── <file>.dart.md ← one per source file in lib/
```

## Per-file annotation format (L1)

```
# <filename>

**Role:** 1-2 sentence one-liner.
**Fits in:** where in the system / who calls this.
**Read next:** [[other-file]] — cross-references.

## Public API
- `ClassName` (lines X-Y) — what it exports

## Key flows
Stepwise narrative of the non-obvious flows.

## Gotchas / non-obvious
Invariants, race conditions, license-restricted bits, surprising patterns.

## Line index
Jump-list of the lines worth knowing.
```

## Per-package annotation format (L2)

Each `PACKAGE.md` covers directory purpose, file index, dependency graph, multi-file flows, and "how to add X" recipes.

## Conventions

- Cross-references use `[[name]]` matching the annotation filename without `.md`/path.
- Line refs use `<file>:<line>` notation.
- Keep it terse — readable in under 90 seconds.
- Descriptive, not prescriptive — design decisions belong in the parent `CLAUDE.md`.

## Maintenance

When upstream changes a file significantly, regenerate the annotation rather than patching. Stale annotations are worse than none.

## Top-of-module pointers

- Parent module CLAUDE.md: [`../../CLAUDE.md`](../../CLAUDE.md)
- Top-level traccar index: [`../../../CLAUDE.md`](../../../CLAUDE.md)
