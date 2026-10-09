# Tivel — Product Brief v2 (Engineering Handoff)

> Status: **source of truth**. Supersedes `brief v1`.
>
> Product name: **Tivel** (final — do not restart naming; verify domain + npm scope availability before creating the repo, silently, without reopening the discussion)
>
> Category: **Code Comprehension Workspace** — long-term scope: any bounded slice of a codebase; go-to-market wedge: change review (see "Product horizon")
>
> Distribution: **local-first standalone desktop app with CLI entry point**
>
> Repository: public `tivel` repo under the author's personal GitHub profile; transferable to a dedicated org later.

---

## Changelog vs brief v1

This version intentionally changes the following decisions. Everything else from v1 that is not contradicted here remains valid background material (`docs/brief-v1.md`).

1. **Voice is removed from MVP scope.** Voice is a delivery channel and a v2 feature, not the core. The core is the interactive guided comprehension of a changeset. Positioning language must not say "voice-first". Architectural requirement: the session/tour engine is **event-driven and channel-agnostic** so voice can be added later without a rewrite.
2. **A validation phase precedes all infrastructure.** Validate the riskiest hypothesis before writing pipeline code. *(Superseded by the v2.0 changelog below: the hypothesis changed, and with it the phase. The principle stands.)*
3. **Primary language ecosystem: TypeScript.** Tivel itself is built in Node/TypeScript; the first supported analyzed ecosystem is TypeScript (ts-morph / tsserver in-process). C#/Roslyn comes later.
4. **Evidence anchoring: symbol + content hash**, not file + line ranges. Line ranges are display-only denormalization. This is required to survive rebases/force-pushes and changeset evolution during review.
5. **MVP ends at the MVP line** in Section 11. Voice and persistent review-session state move post-MVP.
6. **Lazy depth model is explicit.** The precomputed artifact is a skeleton; depth is generated on demand during the session.

## Changelog vs brief v2.0 (2026-10-09)

New evidence reopened one closed decision. Recorded here rather than silently edited.

**Evidence.** The author has been building and using changeset-analysis skills in real client work for ~10 months — three iterations, three ecosystems, three clients. The generated analyses were consistently comprehensible and did let him understand unfamiliar changesets. The hypothesis "can an LLM generate a narrative that builds comprehension?" is therefore **already answered: yes**, with far stronger evidence than a one-week test could produce.

**What this changes.**

1. **The validation phase is redefined and renumbered** (the old "Phase −1" is gone; the gate now sits after Phase 2). It no longer validates narrative quality. The remaining unvalidated risk is **presentation and interaction**: today's output is a wall of text plus diagrams the author rarely looks at, with follow-up questions costing extra prompts and extra context-switching. The new question is whether an interactive, anchored presentation of the same content produces qualitatively different comprehension.
2. **Contract-first, not pipeline-first.** A `TourArtifact` contract (Section 6a) is defined before anything else. The existing skill is adapted to emit it; the UI consumes it. The pipeline later replaces the producer behind the same contract. The seam is the architecture, not a prototype.
3. **The UI moves to the front of the sequence** (Section 11). Building two phases of infrastructure before first seeing the thing that carries the value was the wrong order once the narrative risk disappeared.
4. **Do not parse narrative markdown into structure.** Extracting structure from prose is throwaway work that pollutes the signal. The producer emits structured data directly.

**What does not change.** Facts vs interpretation, anchoring, MVP boundary, the decision filter, TypeScript-first, voice out of MVP, local-first. The fact layer is deferred in time, not dropped: until it exists, every anchor is `verified: false` and the UI says so.

---

## 1. Problem

AI is dramatically increasing code-generation throughput, but human comprehension and review bandwidth are not scaling with it. For small changes a Git diff suffices. For non-trivial features — especially agent-generated ones — the reviewer must reconstruct: what was implemented, why, where it enters the system, what the runtime flow is, what is directly changed vs indirectly affected, what is tested and what is not, and whether the implementation fits the surrounding codebase.

Existing AI review bots (CodeRabbit, Greptile, Qodo, Copilot review…) answer *"are there bugs?"*. None of them answer *"do **I** understand what happened here?"*. Detection is not comprehension. That gap is the product.

## 2. Thesis and positioning

> **AI should not review the code instead of the developer. AI should make human review scalable.**

Tivel is a **pair-review environment**: an interactive workspace where a developer builds a correct mental model of a change, assisted by an evidence-backed AI agent. The human remains the reviewer and remains responsible for approval.

Primary positioning line:

> **Tivel is an AI pair reviewer that helps you understand every change before you merge it.**

Supporting lines: *Understand before you merge.* / *The diff tells you what changed. Tivel helps you understand what the change means.*

Branding note: the **name and product identity stay scope-neutral** (comprehension, not "PR tool"). The change-review message above is the go-to-market wedge — the sharpest, most frequent, most urgent instance of the general problem — not the ceiling of the product. Do not market the general "understand your codebase" vision early: that category is crowded (Sourcegraph, chat-with-repo tools, Cursor's repo index) and invites comparison to incumbents before the product has earned it.

### Target user (deliberately narrow)

Not the "it works → merge" vibe-coder — that user does not feel the problem and will not pay for the solution. The buyer/user is the **person accountable for the merge**:

- tech leads and seniors reviewing agent-generated code,
- contractors signing their name under code their agents produced,
- teams in environments where "I don't know why it works" is an audit finding.

Strategic note: this segment **grows** as agent adoption grows — the more code agents write, the more developer value shifts from generation to comprehension and accountability.

## 3. Form factor

**Standalone desktop app** (not an IDE plugin — the intended interactivity would suffocate in an IDE webview), with a hard constraint:

> Standalone must never mean "another app you have to remember to open."

- **CLI is the front door**: `tivel review <base>..<target>` / `tivel review` (working tree) launched from the terminal the user already lives in; the app opens with context loaded. Zero manual "open app → pick repo → pick branch" steps.
- **Hook into the pre-merge ritual**: invokable as a step after an agent session (e.g., a Claude Code command / git hook) so Tivel is a stage in an existing process, not a competitor for attention.
- Suggested runtime shape: local analysis server + CLI + desktop shell (Tauri or Electron — decide at implementation time; boring and proven wins) communicating over a local socket. Not a hard commitment; the CLI-first entry is the commitment.
- Local-first: code and the local index stay on the machine; the only thing that leaves is the minimal context sent to the model API. State it clearly in the UI. Never silently upload a repository.

## 4. Architecture principle: facts vs interpretation

The artifact has two layers with different guarantees:

**Fact layer — deterministic, zero LLM.** Diff, changed symbols (AST), reference/call graph (tsserver/ts-morph), git history, linked metadata (PR description, tickets, comments — when available). Rebuilds identically on every run. This is ground truth.

**Interpretation layer — LLM.** Narrative, intent, clustering, sequencing, consistency assessment, risk hypotheses.

Binding rule:

> **Every interpretation-layer claim must be anchored in the fact layer.** Each claim carries provenance (symbol id + content hash, ticket id, hunk id…). A claim without a valid anchor is rejected or explicitly flagged `unverified` in post-processing.

Scope-agnosticism requirement: the fact layer indexes the **repository**; a changeset is merely a *scope selector* over it (changed symbols + graph closure). Do not couple code-intelligence or the Cluster→Sequence→Narrate→Verify pipeline to diffs — their input is "a set of symbols with a graph and a boundary". This is what makes future comprehension scopes (feature slice, module, system overview) an input change, not a rewrite. Do not build a `ComprehensionScope` abstraction in MVP — just avoid the coupling.

Consequences:

- **Blast radius is computed, not generated.** The graph provides "what touches what"; the LLM only describes and prioritizes it ("of these 14 call sites, these 2 are risky because…"). Never the reverse.
- Pure "RAG over a diff" is explicitly rejected. Questions like "who calls `consumeToken()`?" are answered by graph traversal, not semantic similarity.
- Static analysis never claims to prove runtime behavior. Inferred flows are marked as inferred.

### Trust model (carried over from v1 — unchanged)

1. Show evidence. 2. Show uncertainty. 3. Distinguish fact / documented intent / inference / unknown. 4. Make source navigation easy. 5. Do not fabricate architectural intent. 6. Prefer "I cannot determine that" over plausible fiction. 7. Mark inferred flows as inferred. 8. Surface unsupported claims as hypotheses.

In this product, **trust is not a feature — it is the entire product**. A verification tool that itself requires verification is dead on arrival.

## 5. Analysis pipeline

```text
1. EXTRACT   (deterministic)
   diff → changed symbols → reference-graph closure (depth 1–2)
   → git blame/log for changed files → available metadata
   Output: fact graph.

2. CLUSTER   (LLM, cheap)
   Group changes into conceptual units ("new endpoint + DTO + service + tests"),
   separate mechanical bulk changes ("rename across 27 files").

3. SEQUENCE  (LLM)
   Order clusters narratively: intent → entry points → core logic → plumbing
   → mechanical changes collapsed into a single stop.
   Narrative order ≠ file order. This is half the product's value.

4. NARRATE   (LLM, per stop)
   Explanation with mandatory anchors + flags
   ("duplicates existing `X` from utils/", "deviates from the module's pattern") —
   flags must reference existing, verified symbols.

5. VERIFY    (deterministic)
   Validate every anchor: symbols exist, hashes match, cited call sites
   are in the graph. Failures are dropped or flagged `unverified`.
```

### Lazy depth model

Pre-compute only the skeleton: fact graph + clusters + sequence + stop narrations + overview/hotspots. Everything deeper (per-symbol deep dives, "why no retry here?", flow exploration) is answered **on demand** by the session agent, which has tool access to the fact graph and the codebase and obeys the same anchoring rules. This keeps analysis fast (a tour that takes 15 minutes to build will never enter the pre-merge ritual) and cost bounded (depth is generated only where the user actually looks).

### Session engine — channel-agnostic (voice-readiness requirement)

The interactive session is a stream of typed events, e.g.:

```text
focus(symbolId | fileId, range)
narrate(segmentId, anchors[])
annotate(evidenceId)
awaitQuestion()
answer(text, anchors[])
navigate(view)
```

The presentation layer decides whether `narrate` renders as text or (later) as TTS with synchronized highlighting. No presentation concern may leak into the engine. This single constraint is what makes voice a v2 integration instead of a v2 rewrite.

## 6. Structured Change Model

Central artifact consumed by the UI and the agent. The LLM consumes this model; it is never the source of truth for code structure.

Core entities (start minimal, grow honestly):

```text
Repository, ChangeSet, ChangedFile, DiffHunk,
CodeSymbol, Relation, Cluster, TourStop,
RuntimeFlow (inferred, marked as such),
Evidence, Claim, AnalysisRun
```

Evidence anchoring (changed vs v1): anchor = `symbolId + contentHash` (+ optional hunk id). Line ranges are stored only as display hints and are recomputed after changeset updates. Design consideration to keep in mind (not to fully build in MVP): the changeset **will** change mid-review (author pushes fixes); the model must be re-derivable without corrupting existing anchors.

Relations: only those supportable with acceptable reliability (`contains, imports, calls, calledBy, references, tests, modifies` first; the rest earns its way in).

Persistence: **SQLite**, versioned schema + migrations from day one, cheap full rebuild, stale-index detection, analysis-run identity. No external DB infrastructure.

## 6a. TourArtifact — the producer/consumer contract

The contract is the **seam** of the whole system: it decouples whatever produces a tour from whatever presents it.

```text
PRODUCER                      CONTRACT              CONSUMER
existing skill (v0)      →   TourArtifact JSON  →   apps/web
real pipeline (v1)       ↗                          (permanent)
```

The UI is written once against this contract and survives the producer swap. The only throwaway artifact in the whole plan is the v0 adapter that makes the existing skill emit this shape.

Second-order benefit, and the reason this ordering is right: **the contract is defined by what the consumer needs**. The Structured Change Model (Section 6) is then designed against known UI requirements instead of guesses.

Shape (illustrative; refine during Phase 0, then freeze for the UI work):

```ts
interface TourArtifact {
  schemaVersion: number;
  changeSet: { repo: string; base: string; head: string };
  overview: { intent: string; surface: SurfaceItem[]; startHere: StopId };
  stops: TourStop[];
  hotspots: Hotspot[];
}

interface TourStop {
  id: StopId;
  title: string;
  kind: 'intent' | 'entry-point' | 'core-logic' | 'plumbing' | 'mechanical';
  narrative: NarrativeSegment[];   // segmented, never a wall of text
  anchors: Anchor[];
  suggestedQuestions?: string[];
}

interface NarrativeSegment {
  text: string;
  anchorRefs: AnchorId[];          // what to highlight while this segment is shown
}

interface Anchor {
  id: AnchorId;
  file: string;
  symbol?: string;
  range?: { startLine: number; endLine: number };  // display hint only
  contentHash?: string;
  verified: boolean;               // v0: always false — the skill cannot verify
}
```

Two deliberate details:

- **Narrative is segmented, not prose.** Each segment knows what to highlight. This is the structural answer to the "wall of text" problem, and the same field later drives synchronized TTS highlighting (Section 5, voice readiness).
- **`verified` is explicit from day one.** In v0 no fact layer exists, so every anchor is an unverified model claim and the UI must show that. Counting how many v0 anchors turn out to be wrong is also the measurement that justifies the fact layer's cost.

## 7. Agent tool surface (illustrative)

```text
-- deterministic facts
getChangeOverview() getChangeStory() getReviewHotspots()
getSymbol(id) getSymbolDiff(id) getPreviousSymbolVersion(id)
getCallers(id) getCallees(id) getReferences(id) getBlastRadius(id)
getRelatedTests(id) searchCode(query)        // lexical + structural first

-- UI actions (event stream)
showSymbol(id) showDiff(id) showFlow(id) highlightEvidence(id)
```

Rule: **deterministic tools for code facts, LLM reasoning for synthesis.** Semantic/embedding retrieval only for questions that genuinely need it ("is there a similar implementation elsewhere?"), added when the need is proven, not by default.

## 8. Core UX

The primary surface is the **change**, not the chat. A blank chat box pushes cognitive work back onto the user — Tivel proactively creates orientation.

- **Change Overview** (first screen): what happened — intent, surface (files/modules/endpoints/migrations), primary flows, hotspots, suggested starting point.
- **Change Story**: clusters in narrative order; the guided tour through the changeset.
- **Change Map**: interactive graph of changed symbols + relevant surroundings; nodes navigate to code.
- **Symbol Inspector**: role, why it matters, change status, callers/callees, related tests + gaps, review priority — every claim anchored.
- **Anchored chat**: questions asked in the context of the current stop/symbol; answers navigate and highlight.

Layout direction (from v1, still valid): overview bar / story sidebar / main workspace (diff–code–graph–flow) / context panel / interaction bar.

## 9. MVP scope

### Hypothesis under test

> **Can Tivel materially reduce the time required to build a correct mental model of a non-trivial changeset?**

Primary metric: **time to useful mental model** (target: ~30 min baseline → 5–10 min without accuracy loss).

### Required

1. Local changeset input: `main..HEAD` (working tree if cheap).
2. Change Overview.
3. Change Story (narrative-ordered tour).
4. Interactive Change Map. **Under review** — in ~10 months of real use the author generated diagrams routinely and rarely looked at them. Decide at the Section 10 gate whether the map earns its place or is cut from MVP; do not build it before that evidence exists.
5. Symbol Inspector with evidence.
6. **Text-based** anchored assistant with UI-navigation actions.

### Explicitly NOT in MVP

Voice (v2; engine must be channel-agnostic from day one). Persistent review-session state and review summaries (post-MVP). GitHub App / PR commenting / approvals, GitLab, Jira/Linear, SSO/RBAC/orgs/billing/SOC2, multi-repo analysis, all-language support, autonomous fix loops, enterprise anything, mobile.

Decision filter for every feature:

> **Does this help a human understand or verify a non-trivial software change faster and with more confidence?** If no — it is not part of the product yet.

## 9a. Product horizon: comprehension scopes (post-MVP direction)

Tivel's long-term shape is a comprehension engine over a **scope ladder**:

```text
1. changeset        (MVP — wedge: frequent, urgent, pre-merge trigger)
2. feature slice    (entry point + graph closure — same bounded problem
                     shape as a changeset; first expansion, near-zero
                     pipeline cost; use case: "getting into someone
                     else's feature")
3. module / app     (bounded by module boundaries)
4. system overview  (onboarding maps — hardest, least differentiated,
                     highest narration-quality risk)
```

Constraints on this horizon, stated so they are never forgotten:

- **Frequency rules sequencing.** Review happens daily; onboarding happens once per hire. Expand down the ladder only after the wedge loop is loved.
- **Narrative quality degrades with scope.** A changeset has a natural narrative axis (intent → implementation). A whole system does not; unscoped narration collapses into generic "this service handles X" filler that any chat-with-repo tool produces for free. Tivel's value (anchoring, sequencing, computed blast radius) is strongest on bounded scopes.
- **CodeSee is the cautionary tale** for leading with system maps + onboarding walkthroughs as a standalone product.
- The decision filter (Section 9) applies: system-scope features do not enter MVP.

## 10. Validation plan

### Already validated (do not re-test)

**LLM-generated changeset narration works.** ~10 months of real use across three skill iterations, three ecosystems and three clients. Generated analyses were comprehensible and did build understanding of unfamiliar changesets. Treat this as settled; spending a phase re-proving it is waste.

Known weaknesses of that output — these are the product requirements, not reasons to doubt the approach:

- wall of text; the reader still has to excavate it
- diagrams are generated but rarely looked at
- follow-up questions cost extra prompts and context-switching
- non-deterministic; occasional fabrication with a confident tone

### The open hypothesis

> **Does an interactive, anchored presentation of the same narration produce qualitatively better comprehension than a markdown document?**

Author's north star for success:

> *After a session I should feel like a teammate just walked me through their changeset — what it does for the business and how it achieves it technically.*

That feeling depends at least as much on representation and interaction as on narration quality. It cannot be tested with markdown; it needs a working interface.

### Validation gate (after Phase 2 in Section 11)

Run a real, non-trivial, not-yet-reviewed changeset through the interface and measure:

1. **Exits from the tool** — how many times the author had to leave for an IDE, grep or a separate prompt to understand something the tour did not cover. **This is the primary metric**; "I never had to leave" is the operational definition of the teammate feeling.
2. **Time to answer the 10 questions**, versus the same changeset consumed as markdown from the existing skill.
3. **Unverified-anchor error rate** — of the v0 anchors (all `verified: false`), how many point at something that does not exist or does not say what the narration claims. This sizes the fact layer's value in numbers.

Kill criterion: if the interactive version is not meaningfully better than reading the markdown, the product thesis is wrong and no amount of pipeline work fixes it — stop before Phase 3.

Caveat on evidence quality, stated so it is not forgotten: this gate is n=1, judged by the product's author on a codebase he knows. It tests the interaction model, not market demand.

### The 10-question comprehension test (from v1 — unchanged)

After a session, the reviewer must be able to answer: 1) intent, 2) entry points, 3) runtime flow, 4) directly changed, 5) potentially affected unchanged areas, 6) persistence/data changes, 7) highest-risk areas, 8) protecting tests, 9) untested important behavior, 10) what deserves manual inspection first.

### Dogfooding

At least 3 real changesets. Record every misleading answer — those failures become product requirements.

## 11. Implementation sequence

Ordering principle: **spend the earliest evenings on what is not known.** Git ingestion, ts-morph and SQLite carry near-zero technical risk — they will get built. The interaction model is the open question, so it comes first.

```text
Phase 0   Repo skeleton + TourArtifact contract (Section 6a) as JSON Schema
          + TS types in packages/change-model. Committed first: it is the seam
          every later phase is written against.
Phase 1   Adapt the existing review skill: refine its inputs and make it emit
          TourArtifact JSON directly. Never parse narrative markdown into
          structure. The adapter is throwaway; the prompts survive into Phase 5.
Phase 2   apps/web — the real interaction model, fed by TourArtifact JSON
          (fixture files; no backend yet): stop list, code view with
          anchor-synchronized highlighting, navigation, anchored chat scoped to
          the current stop, explicit "unverified" treatment of v0 anchors.
          Interaction model must be designed properly; visual polish may wait.
          ← VALIDATION GATE (Section 10). Stop here if it fails.
Phase 3   CLI + git ingestion (`tivel review main..HEAD`). No AI.
          Tests on temp git repos.
Phase 4   Code-intelligence vertical slice (TypeScript only): symbols, ranges,
          imports, references, hunk→symbol mapping, relation graph. Fact layer.
Phase 5   Real pipeline (Extract→Cluster→Sequence→Narrate→Verify) emitting the
          same TourArtifact; provider-neutral LLM abstraction; anchors become
          verified; Structured Change Model + SQLite + migrations.
Phase 6   Session agent: tool-based Q&A over the model + UI actions, as a
          channel-agnostic event stream. Key test: select a symbol, ask "what
          calls this and why does it matter?" → deterministic callers +
          anchored synthesis.
---------  MVP line ----------
Phase 7   Review-session state (reviewed/skipped/concerns) + summary.
Phase 8   Voice channel: STT/TTS or realtime API over the existing event
          stream; push-to-talk acceptable; test recognition of "Tivel" early.
Phase 9   Dogfood hard, then delete features that do not improve comprehension.
Phase 10  Feature-slice scope (`tivel explore <entryPoint>`): first step down
          the scope ladder (Section 9a) — reuses the pipeline with a different
          scope selector. Gate: wedge loop proven in dogfooding first.
```

Consequence to accept consciously: between Phase 2 and Phase 5 the system runs on an unverified producer. That is intentional — it buys an early answer to the only open question — but no claim of verified anchoring may be made until Phase 5 lands.

## 12. Engineering principles

- Production-quality code; explicit error handling; all code/comments/identifiers in English.
- Deterministic analysis wherever possible; LLMs used intentionally, never by default.
- Provider-independent domain logic; no coupling to one model vendor.
- Test the analysis logic (it is the product); build golden datasets from the validation gate onward.
- Instrument early: extraction time, indexing time, LLM latency, token usage, estimated cost per analysis run. Bottlenecks visible, optimization deferred.
- Vertical slices; no speculative abstractions; the project earns complexity gradually.
- Monorepo, roughly: `apps/{cli,runtime,web}` + `packages/{change-model,git-analysis,code-intelligence,review-agent,persistence}` — but never create empty packages for diagram aesthetics.

## 13. Project goal & posture

Side project first. Acceptable success states: $0 revenue + excellent portfolio + genuinely useful personal tool; $100–200 MRR is a meaningful win; anything larger only if the market pulls. Optimize for usefulness, dogfooding, low operational burden, and a small number of happy users. **Do not build for hypothetical enterprise procurement.** Do not spend the first months on SSO, billing, or org management.

Known market risk (stated so it is never forgotten): the industry talks "human in the loop" but optimizes for speed. Tivel deliberately serves the accountable minority, betting that accountability pressure grows with agent adoption. If that bet is wrong, the fallback success states above still hold.

## 14. Mantra

> **The diff tells you what changed. Tivel helps you understand what the change means.**
>
> **AI does not replace your review. AI makes your review scalable.**

## 15. Immediate next actions

Done: domain `tivel.dev` registered; npm scope `@tivel/*` reserved (the unscoped name is blocked by npm's similarity filter — the CLI binary is still `tivel` via `bin`); public repo created.

1. Archive brief v1 as `docs/brief-v1.md`.
2. **Phase 0** — write the `TourArtifact` contract and commit it before anything else.
3. **Phase 1** — make the existing skill emit that contract.
4. **Phase 2** — build `apps/web` against it, then run the Section 10 gate.
