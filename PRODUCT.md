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
2. **A validation phase (Phase −1) precedes all infrastructure.** The riskiest hypothesis — "can an LLM generate a narrative that actually builds comprehension?" — is tested in ~1 week with throwaway tooling before any pipeline code is written.
3. **Primary language ecosystem: TypeScript.** Tivel itself is built in Node/TypeScript; the first supported analyzed ecosystem is TypeScript (ts-morph / tsserver in-process). C#/Roslyn comes later.
4. **Evidence anchoring: symbol + content hash**, not file + line ranges. Line ranges are display-only denormalization. This is required to survive rebases/force-pushes and changeset evolution during review.
5. **MVP = Phases 0–5** of the v1 sequence. Voice (v1 Phase 6) and persistent review-session state (v1 Phase 7) move post-MVP.
6. **Lazy depth model is explicit.** The precomputed artifact is a skeleton; depth is generated on demand during the session.

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
4. Interactive Change Map.
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

### Phase −1 — throwaway validation (~1 week, before any product code)

1. Take a real, fresh, non-trivial changeset the author has **not yet reviewed**.
2. Use existing agent tooling (Claude Code on the repo) with a structured prompt/skill to generate a markdown tour: Overview + Change Story + hotspots, following the anchoring spirit (agent greps its own evidence).
3. Evaluate with the 10-question test below; then do a classic manual review and count what the tour missed or fabricated.
4. One confident fabrication = the Verify stage becomes the top design priority. Generic, comprehension-free narrative = revisit Cluster/Sequence prompting before building anything.
5. Keep the outputs as the first **golden dataset** for testing the real pipeline later.

Kill criterion: if after honest iteration the generated tour is not clearly better than `git diff` + ad-hoc prompting, stop and rethink before writing infrastructure.

### The 10-question comprehension test (from v1 — unchanged)

After a session, the reviewer must be able to answer: 1) intent, 2) entry points, 3) runtime flow, 4) directly changed, 5) potentially affected unchanged areas, 6) persistence/data changes, 7) highest-risk areas, 8) protecting tests, 9) untested important behavior, 10) what deserves manual inspection first.

### Dogfooding

At least 3 real changesets. Record every misleading answer — those failures become product requirements.

## 11. Implementation sequence

```text
Phase −1  Throwaway validation (see above). GO/NO-GO gate.
Phase 0   Repo skeleton + CLI + git ingestion (`tivel inspect main..HEAD`,
          JSON output). No AI. Tests on temp git repos.
Phase 1   Code-intelligence vertical slice (TypeScript only):
          symbols, ranges, imports, references, hunk→symbol mapping,
          basic relation graph. Debug output first.
Phase 2   Structured Change Model + SQLite persistence + migrations.
Phase 3   Minimal web workspace: overview, files, symbols, diff,
          symbol details, graph.
Phase 4   AI enrichment through the Extract→Cluster→Sequence→Narrate→Verify
          pipeline; provider-neutral LLM abstraction; claims stored with
          anchors + certainty class + run id.
Phase 5   Session agent: tool-based Q&A over the model + UI actions
          (event stream). Key test: select a symbol, ask "what calls this
          and why does it matter?" → deterministic callers + anchored synthesis.
---------  MVP line ----------
Phase 6   Review-session state (reviewed/skipped/concerns) + summary.
Phase 7   Voice channel: STT/TTS or realtime API over the existing event
          stream; push-to-talk acceptable; test recognition of "Tivel" early.
Phase 8   Dogfood hard, then delete features that do not improve comprehension.
Phase 9   Feature-slice scope (`tivel explore <entryPoint>`): first step down
          the scope ladder (Section 9a) — reuses the pipeline with a different
          scope selector. Gate: wedge loop proven in dogfooding first.
```

## 12. Engineering principles

- Production-quality code; explicit error handling; all code/comments/identifiers in English.
- Deterministic analysis wherever possible; LLMs used intentionally, never by default.
- Provider-independent domain logic; no coupling to one model vendor.
- Test the analysis logic (it is the product); golden datasets from Phase −1 onward.
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

1. Verify domain + npm scope for `tivel` (quietly; naming stays closed).
2. Create public repo `tivel` under the personal profile; commit this file as `PRODUCT.md`; archive v1 as `docs/brief-v1.md`.
3. Run **Phase −1** on a real changeset from current work.
4. Only after a GO: Phase 0.
