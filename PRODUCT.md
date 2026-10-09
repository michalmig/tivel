# Tivel — Product Brief v2 (Engineering Handoff)

> Status: **source of truth**. Supersedes `brief v1`.
>
> Product name: **Tivel** (final — do not restart naming; verify domain + npm scope availability before creating the repo, silently, without reopening the discussion)
>
> Category: **Code Comprehension Workspace** — long-term scope: any bounded slice of a codebase; go-to-market wedge: change review (see "Product horizon")
>
> Distribution: **local-first, CLI-first.** The UI is a web app served by a local server and opened in a browser window. A desktop shell is a later packaging decision, not an architectural one.
>
> Repository: public `tivel` repo under the author's personal GitHub profile; transferable to a dedicated org later.

---

## Changelog vs brief v1

This version intentionally changes the following decisions. Everything else from v1 that is not contradicted here remains valid background material (`docs/brief-v1.md`).

1. **Voice is removed from MVP scope.** Voice is a delivery channel and a v2 feature, not the core. The core is the interactive guided comprehension of a changeset. Positioning language must not say "voice-first". Architectural requirement: the session/tour engine is **event-driven and channel-agnostic** so voice can be added later without a rewrite.
2. **A validation phase precedes all infrastructure.** Validate the riskiest hypothesis before writing pipeline code. *(Superseded by the v2.0 changelog below: the hypothesis changed, and with it the phase. The principle stands.)*
3. **Primary language ecosystem: TypeScript.** Tivel itself is built in Node/TypeScript; the first supported analyzed ecosystem is TypeScript (ts-morph / tsserver in-process). C#/Roslyn comes later.
4. **Evidence anchoring moves off raw line ranges.** Line ranges become display-only denormalization. *(The specific identity chosen here — symbol + content hash — was superseded by the review changelog below: anchors are location-first, `commit + file + quote`, with symbol identity as fact-layer enrichment.)*
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

## Changelog vs v2.0 — independent review (2026-10-09)

An independent model reviewed the plan without seeing it first (`docs/validation/independent-review-prompt.md`), then compared. It converged on the Phase 0–2 ordering. Accepted findings:

1. **Phase 2 chat must be real**, not mocked — the gate's primary metric depends on it.
2. **The gate's baseline was a strawman.** The honest baseline is markdown plus follow-up prompts in an agent session.
3. **Epistemic status was missing from the contract** although the trust model demands it. Added as a mandatory field.
4. **Anchoring reopened.** `symbolId + contentHash` was justified by a post-MVP scenario and breaks on templates, migrations and config. Anchors are now location-first (`commit + file + quote`) with symbol identity as enrichment.
5. **Reference errors and claim errors split.** Counting them together would have justified a fact layer that only fixes one of them.
6. **Client-derived artifacts must never reach this public repo.**
7. **Browser-first**, desktop shell deferred; language agnosticism expressed as fact-layer tiers.
8. **Voice moved after dogfood-and-delete**; the custom pipeline is no longer a closed decision.

Rejected: a monetization plan now (the stated acceptable outcomes do not require one) and a full timebox regime (only Phase 2 is timeboxed).

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
- Runtime shape: **local analysis server + CLI + a web UI the CLI opens in a browser window** (`--app`-style chromeless window where available). The server does every privileged thing — filesystem, git, language servers, model calls; the UI is a plain web app talking to it over HTTP/WS.
- **Binding rule: no logic in the shell.** As long as that holds, wrapping the same UI in Tauri or Electron later is a packaging exercise, not a rewrite — so the shell decision stays deferred instead of being made now. A desktop shell buys tray, global shortcuts and native menus; it buys nothing this product needs today.
- **The local server is a privileged process on a listening port.** It can read the repository and run git, so any page open in the browser could reach it (trivial CORS, DNS rebinding). Non-negotiable: a per-session token minted by the CLI and carried in the URL it opens, `Origin` checking, bind to loopback only, and a short-lived session. A desktop shell would not need this; browser-first does, and it is cheap compared to what it buys.
- **Latency is not the reason to pick a shell.** Electron and Tauri also cross a process boundary (renderer↔main, webview↔core) — the choice is which transport, not whether there is one. Loopback adds fractions of a millisecond on small payloads, against file reads in milliseconds, language-server queries in tens to hundreds, and model calls in seconds. What actually makes such a UI feel slow is independent of the shell: oversized payloads per interaction, chatty request patterns, parsing large artifacts on the main thread, and language-server cold start. Discipline instead: WebSocket for the session event stream, ranged/incremental file fetches, prefetch the next stop, render the editor and syntax highlighting client-side so no keystroke ever crosses the boundary.
- Operational cost browser-first does carry, and must be handled rather than discovered: port selection, "is the server already running", and orphaned processes. The CLI owns the lifecycle; the server exits when the last client disconnects.
- Things sometimes assumed to require a desktop shell, and where they actually live: file editing (server writes; Monaco/CodeMirror render in the browser), dependency navigation and go-to-definition (server + fact layer), microphone for the later voice channel (works on localhost), "open in IDE" (`vscode://` / `idea://` handlers). None of them is a reason to ship a shell.
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

### Fact layer capability tiers (language agnosticism)

Only one layer in the system knows the language. The contract, the UI, the git layer and the agent-based producer are all language-agnostic — quote-based anchoring (Section 6a) works identically in TypeScript, C#, Go, YAML and a SQL migration.

Consequence, stated so it is never designed away: **Tivel works on any language from day one, at reduced fidelity.** The fact layer is a per-ecosystem upgrade, never a precondition.

```text
T0    git only — diff, history, paths. Works everywhere. No verification,
      no computed blast radius; every anchor stays unverified.
T0.5  tree-sitter — symbols and approximate call sites without resolution.
      Nearly every language, fast, cheap.
T1    resolved references — real call/reference graph, verified anchors,
      computed blast radius. Per ecosystem: ts-morph for TypeScript;
      a generic LSP client for the rest (references / definition /
      documentSymbol are standard server capabilities).
```

Requirements this imposes:

- The UI and the contract must degrade gracefully: a tier is a property of the run, surfaced to the user, never an assumption in the code.
- No fact-layer types leak above the change model. No abstraction built in advance for it — just no coupling.
- Dogfooding note: the author's current work is .NET + Angular. The Angular half exercises T1, the .NET half exercises T0/T0.5. That is an advantage — the degraded path is the path every new ecosystem starts on, so it gets tested from the beginning instead of discovered at the first non-TypeScript user.

### Trust model (carried over from v1 — unchanged)

1. Show evidence. 2. Show uncertainty. 3. Distinguish fact / documented intent / inference / unknown. 4. Make source navigation easy. 5. Do not fabricate architectural intent. 6. Prefer "I cannot determine that" over plausible fiction. 7. Mark inferred flows as inferred. 8. Surface unsupported claims as hypotheses.

In this product, **trust is not a feature — it is the entire product**. But state the claim honestly, because the strong version is false: Verify can only check that symbols exist, quotes match and cited call edges are real. Intent, "risky because" and runtime behavior can never be verified deterministically, so a promise of verified truth would be a promise the product cannot keep.

What Tivel actually offers is **checkability plus declared epistemic status**: every claim is one click from the evidence it rests on, and every claim says whether it is fact, documented intent, inference or unknown (Section 6a). A tool whose inferences are labelled as inferences is trustworthy; one that presents them as findings is not.

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

Evidence anchoring: as defined in Section 6a — location-first (`commit + file + quote`), with `symbolId` and `contentHash` added by a T1 fact layer. Line ranges are display hints, recomputed after changeset updates. Design consideration to keep in mind (not to build in MVP): the changeset **will** change mid-review as the author pushes fixes, so the model must be re-derivable without corrupting existing anchors — a quote survives a rebase that moves it, which is why it is the primary identity.

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

Freeze policy: **soft-freeze at the end of Phase 0** — stable enough to build the UI against, explicitly revisable at the validation gate. Hard-freezing before the gate would lock in guesses about what the consumer needs.

Completeness test for the contract: **a `TourArtifact` must be able to answer all ten questions of the comprehension test (Section 10).** Questions 5 (affected but unchanged areas), 8 (protecting tests) and 9 (untested important behavior) need a structured home, not a paragraph of prose.

Shape (illustrative, not a specification):

```ts
interface TourArtifact {
  schemaVersion: number;
  changeSet: { repo: string; base: string; head: string };
  factTier: 'T0' | 'T0.5' | 'T1';   // see Section 4; drives UI degradation
  overview: { intent: Claim; surface: SurfaceItem[]; startHere: StopId };
  stops: TourStop[];
  hotspots: Hotspot[];
  coverage: {                        // answers questions 5, 8, 9
    affectedUnchanged: Claim[];
    protectingTests: Claim[];
    testGaps: Claim[];
  };
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
  status: EpistemicStatus;         // mandatory — see below
  anchorRefs: AnchorId[];          // what to highlight while this is shown
}

/** Trust model point 3, made structural. */
type EpistemicStatus =
  | 'fact'               // derived deterministically; verifiable
  | 'documented-intent'  // stated by a ticket, PR, spec or agent plan
  | 'inference'          // the model's reasoning; not verifiable
  | 'unknown';           // explicitly undetermined

interface Claim {
  text: string;
  status: EpistemicStatus;
  anchorRefs: AnchorId[];
}

/** Location-first so it works in any language and in non-symbol files. */
interface Anchor {
  id: AnchorId;
  side: 'base' | 'head';           // deleted code can only be anchored on base
  commit: string;
  file: string;
  quote: string;                   // exact snippet — the portable identity
  range?: { startLine: number; endLine: number };  // display hint, recomputable
  symbolId?: string;               // enrichment when a fact layer resolved one
  contentHash?: string;
  verified: boolean;               // T0/T0.5: always false
}

interface SurfaceItem {
  kind: 'file' | 'module' | 'endpoint' | 'migration' | 'config' | 'dependency';
  label: string;
  anchorRefs: AnchorId[];
}

interface Hotspot {
  title: string;
  why: Claim;                      // carries its own epistemic status
  priority: 'first' | 'high' | 'normal';
  anchorRefs: AnchorId[];
}
```

Four deliberate decisions:

- **Narrative is segmented, not prose.** Each segment knows what to highlight. This is the structural answer to the "wall of text" problem, and the same field later drives synchronized TTS highlighting (Section 5, voice readiness).
- **Epistemic status is mandatory on every claim.** Trust model point 3 requires distinguishing fact / documented intent / inference / unknown; a contract without that field cannot express the product's central promise. The UI renders the distinction; it is not decoration.
- **Anchors are location-first, symbols optional.** The portable identity is `commit + file + quote`, which works in a C# file, an Angular template, a SQL migration and a YAML config alike, and is checkable with a string search at tier T0. `symbolId` and `contentHash` are enrichments a fact layer adds at T1, not the primary key. This also keeps the Verify stage useful before any language support exists.
- **`verified` and `factTier` are explicit from day one.** Before the fact layer exists every anchor is an unverified model claim and the UI must say so. Counting how many of them are wrong is also the measurement that decides whether the fact layer is worth its cost (Section 10).

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

Secondary metric: time to a useful mental model (rough target: ~30 min unaided → 5–10 min). Useful as a direction, not as the gate — the single primary metric is defined in Section 10, and only one metric may be primary.

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

**Primary metric — exits from the tool.** How many times the author had to leave for an IDE, a grep or a separate prompt to understand something the tour did not cover. "I never had to leave" is the operational definition of the teammate feeling.

Supporting measurements:

- **Time to answer the 10 questions**, against an honest baseline (below).
- **Reference errors** — anchors pointing at something that does not exist or is not where the anchor says. Cheap to check by string search at any tier; **this is what a fact layer fixes**.
- **Claim errors** — the code exists but does not do what the narration says. **A fact layer does not fix these.** Counting them together with reference errors would justify the wrong investment, so they are counted separately.

Design rules for the gate, because the obvious version of this test is rigged to pass:

1. **Honest baseline.** The comparison is not "the same changeset as markdown". It is the author's actual current workflow: the generated markdown **plus follow-up questions in the same agent session**. Beating markdown alone proves nothing.
2. **Different changesets, not the same one twice.** Running one changeset through both modes measures the learning effect, not the tool. Use changesets of comparable size and alternate which mode goes first.
3. **Thresholds pre-registered in this repo before the first run.** "Meaningfully better" decided after seeing the result is not a criterion. Write the numbers down first, in `docs/validation/`.
4. **The judge is the author, who wants it to pass.** This cannot be eliminated at n=1; it can only be bounded by the three rules above. Record it as a known weakness of the evidence rather than pretending otherwise.

Kill criterion: if the interactive version does not clear the pre-registered thresholds against the honest baseline, the product thesis is wrong and no amount of pipeline work fixes it — stop before Phase 3.

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
Phase 2   apps/web — the real interaction model, fed by TourArtifact JSON:
          stop list, code view with anchor-synchronized highlighting,
          navigation, explicit "unverified" treatment of anchors, and
          anchored chat that calls the existing agent harness headlessly with
          the tour and the current stop as context. The chat must be real: the
          primary gate metric is exits from the tool, and unanswered questions
          are what cause them, so a mocked chat makes the gate meaningless.
          Interaction model must be designed properly; visual polish may wait.
          Timeboxed — if it is not gate-ready in a few weeks of evenings, that
          is itself information.
          ← VALIDATION GATE (Section 10). Stop here if it fails.
Phase 3   CLI + git ingestion (`tivel review main..HEAD`). No AI.
          Tests on temp git repos.
Phase 4   Code-intelligence vertical slice (TypeScript only): symbols, ranges,
          imports, references, hunk→symbol mapping, relation graph. Fact layer.
Phase 5   Producer hardening — OPEN DECISION, taken at the gate, not now:
          (a) a custom Extract→Cluster→Sequence→Narrate→Verify pipeline with
              its own LLM abstraction, or
          (b) keep the agent-based producer permanently and expose the fact
              layer to it as tools (e.g. MCP), with Verify as a deterministic
              post-step over the emitted TourArtifact.
          (b) preserves the thing actually validated over 10 months — skill +
          agent + free repo exploration — and costs far fewer evenings; (a)
          buys deterministic sequencing, finer instrumentation and a tighter
          privacy boundary, since an agentic producer reads what it likes,
          which is in tension with "only minimal context leaves the machine".
          Verify, anchoring and the contract are identical either way.
          Persistence (SQLite, and whether versioned migrations are warranted
          for a rebuildable cache) is decided here too, not earlier.
Phase 6   Session agent: tool-based Q&A over the model + UI actions, as a
          channel-agnostic event stream. Key test: select a symbol, ask "what
          calls this and why does it matter?" → deterministic callers +
          anchored synthesis.
---------  MVP line ----------
Phase 7   Review-session state (reviewed/skipped/concerns) + summary.
Phase 8   Dogfood hard, then delete features that do not improve comprehension.
          Deleting precedes adding a channel — doing it the other way round
          means polishing voice for features that should not exist.
Phase 9   Voice channel: STT/TTS or realtime API over the existing event
          stream; push-to-talk acceptable; test recognition of "Tivel" early.
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
- **Nothing derived from client code enters this repository.** It is public. Fixtures, tours, golden datasets and validation outputs built from real client changesets live outside it, and `.gitignore` guards the paths where they would land. Committed fixtures are synthetic or taken from public repositories. This is the one mistake in the whole plan that cannot be undone: git history, forks and indexers make it permanent.
- Confidentiality is a per-engagement question, not a product setting: which providers a client permits, and what may leave the machine. An agentic producer reads whatever it decides to read, so "only minimal context leaves the machine" is a design goal to verify per setup, not a guarantee to print in the UI.

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
