# CLAUDE.md — Tivel

Operating instructions for Claude Code working in this repository.

## What this project is

Tivel is a local-first code comprehension workspace: it helps a developer build
a correct mental model of a non-trivial changeset before merging it. The human
remains the reviewer; the AI explains, retrieves, and guides — always backed by
evidence.

**`PRODUCT.md` is the single source of truth.** Read it before non-trivial
work. `docs/brief-v1.md` is a historical archive — where it contradicts
PRODUCT.md, PRODUCT.md wins.

## Hard rules

1. **Phase discipline.** Follow the implementation sequence in PRODUCT.md
   (Section 11). Do not start work belonging to a later phase without the
   current phase's exit criteria being met. Phase −1 (validation) gates all
   infrastructure work.
2. **MVP boundary.** No voice, no review-session persistence, no GitHub/GitLab
   integrations, no multi-repo, no enterprise features. If a task drifts toward
   these, stop and flag it instead of building it.
3. **Facts vs interpretation.** Deterministic analysis (git, AST, reference
   graph) is the only source of truth for code structure. LLM output is
   interpretation and must carry anchors (symbolId + contentHash). Never let
   LLM-generated content masquerade as computed fact. Blast radius is computed,
   never generated.
4. **Scope-agnostic fact layer.** Code-intelligence and the
   Extract→Cluster→Sequence→Narrate→Verify pipeline take "a set of symbols
   with a graph and a boundary" as input — never couple them to diffs. Do NOT
   build a ComprehensionScope abstraction yet; just avoid the coupling.
5. **Channel-agnostic session engine.** The interactive session is a stream of
   typed events (focus / narrate / annotate / answer / navigate). No
   presentation concerns in the engine.
6. **Decision filter** for any feature or suggestion: *does this help a human
   understand or verify a non-trivial software change faster and with more
   confidence?* If no — don't build it, don't suggest it.

## Closed decisions — do not reopen

- Product name: **Tivel**.
- Voice: v2 channel, not in MVP.
- First analyzed ecosystem: **TypeScript** (ts-morph / tsserver, in-process).
- Tivel implementation stack: **Node.js + TypeScript**, monorepo.
- Persistence: **SQLite** with versioned migrations from day one.
- Evidence anchoring: **symbol + content hash**; line ranges are display-only.

## Engineering standards

- Production-grade code; explicit error handling; no speculative abstractions;
  vertical slices; the project earns complexity gradually.
- All code, comments, identifiers, commit messages, and docs in **English**.
- Test analysis logic seriously — it IS the product. Use temporary git repos in
  tests for git-analysis code. Maintain golden datasets from Phase −1 onward.
- Provider-neutral LLM abstraction; never couple domain logic to one vendor.
- Instrument from the start: extraction/indexing time, LLM latency, token
  usage, estimated cost per analysis run. Make bottlenecks visible; don't
  optimize prematurely.
- Monorepo shape (create packages only when they have real content):
  `apps/{cli,runtime,web}`,
  `packages/{change-model,git-analysis,code-intelligence,review-agent,persistence}`.

## Working style

- Before implementing, state assumptions and flag trade-offs; prefer asking
  over guessing when a decision is architectural.
- When PRODUCT.md is ambiguous or a new decision is needed, surface it
  explicitly as a decision point — do not silently pick and bury it in code.
- Record notable decisions in `docs/decisions/` as short ADRs
  (context → decision → consequences).
- Prioritize failure reports from dogfooding (wrong symbol mapping, misleading
  graph, unsupported claims, slow analysis) above new features.
