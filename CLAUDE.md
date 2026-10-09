# CLAUDE.md — Tivel

Operating instructions for Claude Code working in this repository.

## What this project is

Tivel is a local-first code comprehension workspace: it helps a developer build
a correct mental model of a non-trivial changeset before merging it. The human
remains the reviewer; the AI explains, retrieves, and guides — always backed by
evidence.

**`PRODUCT.md` is the single source of truth.** Read it before non-trivial
work. `docs/brief-v1.md` is a historical archive that is not committed yet; if
it is absent, PRODUCT.md is the only source. Where they contradict each other,
PRODUCT.md wins.

## Hard rules

1. **Phase discipline.** Follow the implementation sequence in PRODUCT.md
   (Section 11). Do not start work belonging to a later phase without the
   current phase's exit criteria being met. The validation gate after Phase 2
   (PRODUCT.md Section 10) blocks all fact-layer and pipeline work: do not
   start ts-morph, SQLite or the LLM pipeline before it passes.
2. **Nothing from client code enters this repository — it is public.** No
   fixtures, tours, golden datasets or validation outputs derived from real
   client changesets, in any form, including anonymized excerpts. They live
   outside the repo and `.gitignore` guards those paths. Committed fixtures are
   synthetic or from public repositories. This mistake cannot be undone; if a
   task would commit such material, stop and say so.
3. **Contract first.** The `TourArtifact` contract (PRODUCT.md Section 6a) is
   the seam between producer and consumer. Write the UI against the contract,
   never against whatever currently produces it. Never derive structure by
   parsing narrative markdown — the producer emits structured data. Every claim
   carries an epistemic status (fact / documented-intent / inference / unknown)
   and the UI renders the distinction. Until a fact layer exists, every anchor
   is `verified: false` and the UI must say so; never present an unverified
   anchor as a computed fact. The contract soft-freezes at the end of Phase 0
   and is revisable at the validation gate.
4. **Degrade, never assume.** Fact-layer tiers T0 / T0.5 / T1 (PRODUCT.md
   Section 4): the product must work on any language at T0. No code above the
   change model may assume a resolved symbol graph exists, and no fact-layer
   types leak upward. Do not build an abstraction for this in advance — just
   avoid the coupling.
5. **No logic in the shell.** The UI is a plain web app served by the local
   server, opened in a browser. Everything privileged — filesystem, git,
   language servers, model calls — lives in the server. Do not add Tauri or
   Electron, and do not use shell-specific APIs.
6. **MVP boundary.** No voice, no review-session persistence, no GitHub/GitLab
   integrations, no multi-repo, no enterprise features. If a task drifts toward
   these, stop and flag it instead of building it.
7. **Facts vs interpretation.** Deterministic analysis (git, AST, reference
   graph) is the only source of truth for code structure. LLM output is
   interpretation and must carry anchors (symbolId + contentHash). Never let
   LLM-generated content masquerade as computed fact. Blast radius is computed,
   never generated.
8. **Scope-agnostic fact layer.** Code-intelligence and the
   Extract→Cluster→Sequence→Narrate→Verify pipeline take "a set of symbols
   with a graph and a boundary" as input — never couple them to diffs. Do NOT
   build a ComprehensionScope abstraction yet; just avoid the coupling.
9. **Channel-agnostic session engine.** The interactive session is a stream of
   typed events (focus / narrate / annotate / answer / navigate). No
   presentation concerns in the engine.
10. **Decision filter** for any feature or suggestion: *does this help a human
   understand or verify a non-trivial software change faster and with more
   confidence?* If no — don't build it, don't suggest it.

## Closed decisions — do not reopen

- Product name: **Tivel**.
- Voice: v2 channel, not in MVP.
- First T1 fact layer: **TypeScript** (ts-morph / tsserver, in-process).
- Tivel implementation stack: **Node.js + TypeScript**, monorepo.
- Distribution: **CLI-first, browser-based UI**; desktop shell deferred.
- Evidence anchoring: **location-first** — `commit + file + quote`, with
  `symbolId` / `contentHash` as fact-layer enrichment.

## Open decisions — do not close them in code

Decided at the validation gate, with evidence. If a task needs one resolved
earlier, surface it as a decision point instead of picking silently.

- Producer: custom pipeline vs. agent-based producer with the fact layer
  exposed as tools.
- Persistence: whether SQLite is needed at all before the gate, and whether a
  rebuildable cache warrants versioned migrations.
- Change Map: whether the graph view earns its place in MVP.

## Engineering standards

- Production-grade code; explicit error handling; no speculative abstractions;
  vertical slices; the project earns complexity gradually.
- All code, comments, identifiers, commit messages, and docs in **English**.
- Test analysis logic seriously — it IS the product. Use temporary git repos in
  tests for git-analysis code. Maintain golden datasets from the validation gate
  onward.
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
