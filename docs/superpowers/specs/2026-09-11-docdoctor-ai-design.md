# docdoctor-ai — Design Spec

**Date:** 2026-09-11
**Status:** Draft v3 — survived two adversarial review rounds
**Author:** gianca2020 (with Claude)

## Pitch (TL;DR)

**The problem:** Documentation silently goes out of date the moment code changes underneath it. Every team lives with docs that quietly lie.

**The solution:** A GitHub Action that runs on every pull request, finds the docs describing the changed code, uses AI to check whether they're still *accurate*, and posts a comment flagging stale sections with a suggested fix for a human to approve. **Code is the source of truth; docs are what's checked; the tool advises, it never certifies.**

**The design in one breath:** an offline, `main`-pinned index maps code chunks ↔ doc sections; each PR runs a cheap-to-expensive **funnel** (parse the diff → drop trivial changes → retrieve suspect docs → LLM-verify staleness → LLM-suggest a fix), with every risky choice behind a **swappable interface** so it can be measured, not guessed.

**State:** Design complete and hardened across **two rounds of adversarial review** (it broke once — on a rename bug — and was repaired). This document is the **architecture handoff**: no code is written; implementation is planned separately by the team.

**Why build it:** it exercises the full applied-AI-engineering stack — parsing, embeddings, retrieval, LLM-as-judge, generation, and evals — inside a real, installable CI tool, with the tradeoff reasoning made explicit.

> **Revision note (v2 → v3):** This design was deliberately stress-tested by a hostile
> reviewer across two rounds before any code was written. Round 1 found we had
> front-loaded the *easy-to-explain* decisions (embeddings, vector store, language) and
> deferred the *hard, load-bearing* ones (index freshness, change-mapping, eval rigor);
> v2 closed those. Round 2 verified the fixes held and caught a **critical bug v2 itself
> introduced** — the index join keyed off the wrong side, so a renamed function would
> silently flag no docs (the flagship staleness case). v3 fixes that join, adds a
> tree-sitter → `ast` tripwire, orders suspects before the budget cap, and stops quoting
> point-estimate precision/recall at a sample size too small to support it. The two-round
> pressure-test is itself part of the story: this design was attacked, broke, and was repaired.

---

## 1. What we're building — and why

**What it is:** A GitHub Action that runs on every pull request. When a PR changes code, the Action finds the documentation that describes that code, uses AI to check whether the docs are still *accurate*, and posts a comment flagging the stale sections with a suggested fix for a human to approve.

**The core mental model (do not lose this):**
- **Code is the source of truth.** The tool never judges whether the code is good. It assumes the new code is correct.
- **Docs are the thing on trial.** The question is always: *now that the code changed, are the docs still telling the truth about it?*
- It runs when a PR is **opened** (before approval), so stale docs get caught *during* review, as part of the same change.
- It checks **accuracy**, not readability. A beautifully written doc that lies about the code is exactly what we catch.

**Why build it (for the author):** A portfolio project whose goal is to land AI-engineering interviews. It demonstrates the full applied-AI stack inside a real, installable CI tool. **The differentiator must be empirical and visible, not buried in this doc** — see §10.

> **Scope note:** This is **applied AI engineering** (building systems around existing models), not **ML** (training models). Deliberate and employable.

---

## 2. Architecture at a glance

```
                         ┌─────────────────────────────────────────────┐
  BUILT FROM main        │        THE INDEX  ("map" of the repo)        │
  (cached, keyed by SHA) │                                             │
                         │   code ──parse──► code chunks ─┐            │
                         │                                ├─► LINK GRAPH│
                         │   docs ──parse──► doc sections ┘   (+vectors)│
                         └───────────────────────────────┬─────────────┘
                                                          │ loaded per PR
 ═════════════════════════════════════════════════════════│════════════════════
  RUNS ON EVERY PR (online, live)                          │
                                                           ▼
   PR opened ─► [Action fires → container boots] ─► [Diff parser + re-parse touched files]
                                                           │  ← changed chunks (precise)
                                                           ▼
                                          [Meaningfulness filter — FAILS OPEN]
                                                           │  ← drop only clearly-trivial
                                                           ▼
                                     [Suspect finder] ──queries──► LINK GRAPH (main-pinned)
                                                           │  ← docs that MIGHT be stale
                                                           ▼
                                     [Staleness verifier · LLM] (budget-capped)
                                                           │  ← docs that ARE stale
                                                           ▼
                                       [Correction generator · LLM]
                                                           │  ← suggested rewrite
                                                           ▼
                                         [Validation pass · LLM]  ← quality gate (see §6)
                                                           │
                                                           ▼
                          [PR comment: flag + suggested fix + reasoning]
                          (posts ONLY when it finds something — never "all clear")
```

**Two ideas hold the whole thing up:**

1. **The funnel — cheap filters guard expensive intelligence.** Each stage narrows a big cheap pile into a small expensive one; all free filtering happens *before* any paid LLM call. Sending the whole repo to an LLM fails three ways: cost, context limits, and *accuracy* (a needle drowns in a haystack).
2. **Offline index vs. online per-PR.** The expensive "map" is built once and cached; each PR does fast lookups. See the index lifecycle in §4 — this is the foundation, not a detail.

---

## 3. Guiding principles (the spine of every decision)

1. **Put implementations behind interfaces (seams).** Embedder, retriever, parser, and LLM are swappable. Seams let us defer decisions, swap implementations, and run comparative evals.
2. **Cheap filters guard expensive intelligence** (the funnel).
3. **Precompute offline, query cheaply online.**
4. **Recall at retrieval, precision at verification** — *but recall lost before retrieval is unrecoverable* (see D5). So earlier stages must protect recall too.
5. **Optimize for trust in what we surface.** A noisy tool gets uninstalled.
6. **Eval-driven: measure, don't guess.** Hyperparameters are tuned on a dev set and reported on a held-out test set.
7. **YAGNI, and earn autonomy before automating.** Build the seam now, fill it when a real need arrives. Ship trustworthy-but-manual first.
8. **Advise, never certify.** The tool only ever speaks when it finds a problem. It **never** posts "docs are fine ✅." A checker that gives false confidence is worse than no checker.

---

## 4. Components

### The index lifecycle (foundation — designed, not deferred)

- **The index is a `main`-pinned artifact.** It is built from the tip of `main` (the merge base), **cached in CI keyed by the commit SHA**, and **rebuilt on merge to `main`.** It is *not* rebuilt per PR.
- **Per-PR work re-parses only the diff-touched files on the fly.** Suspect-finding uses the *main-pinned* link graph — i.e. the docs linked to the code *as it was before this PR* — which is exactly what we want: those are the docs at risk of being invalidated by the change.
- **Cold start (first run, no index):** on first install (or a one-time setup step), build the index inline and cache it; subsequent runs load from cache. The first run is slow by design and says so.
- **Freshness:** because the index tracks `main` and is rebuilt on merge, it never drifts more than one merge behind. Detecting docs for *brand-new* code (not yet on `main`) is explicitly out of v1 scope — that's a "missing docs" feature, not a "stale docs" feature.

### Offline — building the index
- **Code parser** → extracts chunks. One chunk = one function / method / class (plus module-level config keys and CLI commands). Stable ID (`path::qualified_name`); carries signature, args/defaults, docstring.
- **Doc parser** → splits markdown into sections by heading; records heading path, content, referenced code symbols.
- **Link graph builder** → links doc sections ↔ code chunks (D5). Stores *why* each link exists (provenance).
- **Index store** → persists graph + embeddings; loaded per PR (see lifecycle above).

### Online — per-PR pipeline
- **Diff parser + change-mapping (keys off the `main` side — CRITICAL)** → the join between the diff and the index must use **`main`-side chunk IDs**, not the re-parsed new IDs. Intersect the diff's **old-side** hunk line-ranges against the **main-pinned chunk spans** to identify *which chunks as they existed on `main`* were touched. The post-change file is re-parsed too, but **only to supply the *after* code to the verifier** — never as the suspect-finding join key. Edge cases (all eval cases, §7): decorator lines above a function, shared-import changes, whitespace-only reflows that shift line numbers, added lines forming a new chunk, deletions.
- **Renames and deletions are first-class staleness signals** → if a touched `main` chunk's ID vanishes in the new parse (e.g. `login()` renamed to `authenticate()`, or removed), **every doc linked to that old ID is a prime suspect.** A rename is the *highest-value* staleness case (docs still say `login()`), so it must never silently return nothing — which is exactly what a new-ID join would do.
- **Meaningfulness filter — fails open** → drops only *clearly* trivial changes (whitespace, comment-only, test-only). **When in doubt, keep the change.** Dropping a real change here is an unrecoverable silent miss (D5); a false keep just costs one downstream check.
- **Suspect finder** → queries the main-pinned link graph for doc sections linked to the touched **`main`-side** chunk IDs (candidates).
- **Suspect ordering → budget cap** → **order suspects by change type first** (signature / config / behavior changes before comment-adjacent ones) so the budget is spent on the highest-staleness-probability suspects, *then* apply the cap. Without ordering, the cap might burn the whole budget on a reformatted file and never reach the one real signature change.
- **Staleness verifier (LLM) — budget-capped** → given old code + new code + doc section, decides whether the doc is now inaccurate. Capped at **max N verifications per PR**; on a huge PR it degrades gracefully ("N more suspects not checked — re-run or split the PR").
- **Correction generator (LLM)** → rewrites only the stale parts of confirmed-stale sections, preserving style. Output is a *suggestion*, never applied.
- **Validation pass (LLM)** → checks the suggested fix is accurate and preserves still-correct parts (v1 priority in §6).
- **PR comment** → posts findings: which docs are stale, *why* (provenance/reasoning), the suggested fix. **Posts only when it finds something** (principle #8).

---

## 5. Decisions log (rationale · alternatives · tradeoffs)

### D1 — Language: **Python**
Matches the test-target repos (FastAPI, Pydantic). Rich AI ecosystem.

### D2 — Code parsing: **tree-sitter engine, one language adapter at a time (Python adapter for v1)**
- **Rationale:** Author has non-Python repos they intend to run this on, and multi-language is a genuine differentiator (see §10). Parsing is a solved, deterministic problem; the right tool is a real parser, behind a uniform chunk interface.
- **Alternatives rejected:** regex (can't parse a language reliably); LLM chunking (wasteful, non-deterministic — violates principle #2).
- **Honest caveat / shipping risk:** every v1 eval target is Python, so v1's *committed deliverable is the Python adapter only.* The multi-language differentiation becomes real **only once a second-language adapter is actually shipped and demoed** — an un-exercised capability is not a selling point. **Reversible fallback:** Python's `ast` (simpler, richer for Python, faster to ship) drops in behind the same parser interface if shipping speed becomes the binding constraint. This decision costs nothing to reverse — that's the point of the seam.
- **Tripwire (do not skip):** set a *written* time budget (default: ~2 focused days — author to confirm) for getting the tree-sitter Python adapter extracting chunks correctly. If it isn't working by then, **invoke the fallback and switch to `ast`.** A fallback with no trigger is never actually taken — sunk cost keeps you fighting the grammar. The tripwire is what makes the "reversible" claim real.
- **Interview sentence:** *"I built the parser on tree-sitter behind a uniform chunk interface so it's language-agnostic at the foundation, shipped the Python adapter first, and kept `ast` as a zero-cost fallback — the seam means the multi-language choice is reversible."*

### D3 — Embeddings: **Voyage code-embeddings (default), swappable interface, bring-your-own-key**
- **Rationale:** retrieval target is *code*, so a code-specialized embedder yields sharper code↔doc links; author has Voyage credits (free dev/demo); top-tier quality.
- **Honest limits:** still a hosted API → privacy concern for closed-source repos unchanged (a local model is the answer, via the same interface). Author credits power dev/demo only; installed users bring their own key from config.
- **Payoff:** behind a seam, "Voyage vs OpenAI vs local" is a one-line swap *and an eval* (D7).
- Separate from the vector store (D4): embeddings *make* the numbers; the store *holds and searches* them.

### D4 — Vector store: **brute-force cosine search first, retriever interface; ChromaDB as documented scale-up**
- **Rationale:** a vector DB solves a *scale* problem (fast approximate search over millions of vectors). At our scale (hundreds–thousands), exact brute-force search is sub-millisecond and needs zero infrastructure.
- **Interview sentence:** *"Exact search is sub-millisecond at my scale, so a vector DB solves a problem I don't have — I left a seam to add ChromaDB if scale ever demands it."*

### D5 — Link graph: **hybrid (name-match ∪ embedding-similarity), biased for recall; PR comments surface provenance**
- **Error asymmetry:** a *missed* link means a stale doc never becomes a suspect → **silent failure** (worst). A *wrong* link is a cheap extra suspect the verifier prunes. So optimize **recall** here; precision is restored downstream.
- **Correction from review (v2):** the "precision-at-verification restores everything" story only holds for recall lost *at the embedding-threshold stage*. Recall lost **earlier** — the meaningfulness filter dropping a real change, or the parser never extracting a chunk/section — is **unrecoverable**, because no verifier can prune a candidate that was never created. Mitigations: the meaningfulness filter **fails open** (§4), and **evals measure end-to-end recall** (parser + filter + link-graph together), not just link-graph recall (§7).
- **Mechanisms:** heuristic name-matching (precise, cheap, explainable) ∪ embedding similarity above a threshold (catches semantic links).
- **Threshold tuned on a dev set, reported on held-out test** (D7). Alternatives rejected: heuristic-only (silent misses), embedding-only (loses exact matches + explainability), LLM-built graph (N×M calls, too costly as base).
- **Interview sentence:** *"The link graph optimizes for recall because a missed link fails silently — and I measure recall end-to-end, since a change dropped by the filter or missed by the parser can't be recovered downstream."*

### D6 — Output mode: **flag + suggested fix (human approves); no auto-PR machinery in v1**
- **Earn autonomy before automating.** Auto-applying fixes opens PRs that modify a real repo — high stakes. Flag-with-suggestion is low-stakes and still showcases generation. Auto-fix is a v2 feature unlocked *after* evals prove a low error rate. Cuts v1 scope (no branch/PR-creation machinery).

### D7 — Evals: **dev/test split; honest small-sample reporting for v1; git-mined set as the scale path**
- **Ground truth:** examples where you already know the answer.
- **v1 set:** ~30–40 hand-crafted cases, roughly half positive (should flag) / half negative (should not).
- **Data-leakage fix (v2):** **split into a dev set (tune the D5 threshold) and a held-out test set (report metrics, never tuned against).** Tuning and reporting on the same set is training on your test data.
- **Honest reporting (v3):** at v1's small n, **do not quote point-estimate precision/recall.** On ~15 test cases, "recall = 0.87" carries a 95% confidence interval of roughly [0.62, 0.98] — the number misleads even when caveated, and reads worse than not reporting it. Instead report the **raw confusion-matrix counts (TP/FP/FN/TN)** plus a **qualitative failure taxonomy**, and state plainly that defensible point estimates await the git-mined set. Demonstrating that you understand sampling error is itself the signal. **Report false negatives prominently** (principle #8: a confident miss is the dangerous failure).
- **Gray-zone cases required:** intent-vs-mechanism ("returns a sorted list" when sorting became conditional), partial behavior changes — the hard cases where the LLM verifier is most likely to be wrong.
- **Include negatives** (whitespace, comment-only, behavior-preserving refactor, test-file changes) — the only thing that exposes false positives.
- **Metrics:** v1 → confusion-matrix **counts** + failure taxonomy, measured **end-to-end** (D5). Quoted precision/recall → only on the git-mined set, once CIs are tight enough to mean something.
- **Two things evaluated separately:** detection quality (numeric) vs. correction quality (human rating + narrow LLM-as-judge, spot-checked).
- **Scale path:** mine real git history (commits that changed code *and* docs together) for 100+ cases → defensible numbers. v1's hand-crafted set is honest-but-small and doubles as the regression suite.
- **Eval-driven development:** re-runnable script (ideally CI) re-scores on every change.
- **Interview sentence:** *"On a ~15-case test set I report confusion-matrix counts and a failure taxonomy rather than a fake-precise 0.87 recall whose CI runs 0.6–0.98 — quoted precision/recall waits for the git-mined set. And the threshold is tuned on dev, never on the reported set."*

### D8 — LLM access: **behind an interface, with model tiering** *(pin exact models at build time)*
Wrap all model calls (swappable, eval-comparable). Cheap model for cheap steps (meaningfulness, verification), stronger model for generation.

---

## 6. v1 scope vs. deferred

| Capability | v1 | Deferred |
|---|---|---|
| Output mode | Flag + suggested fix (comment) | Auto-fix + PR-creation machinery |
| Languages | Python adapter only | JS / Go / … adapters (same interface) |
| Vector search | Brute-force cosine | ChromaDB (scale) |
| Index lifecycle | **main-pinned, SHA-cached, rebuilt on merge; cold-start builds inline** | Incremental partial re-index optimization |
| Huge-PR handling | **Suspect ordering by change-type → budget cap + graceful degradation** | *Smarter* (e.g. learned) suspect prioritization |
| Change-mapping | **Precise span-intersection + edge cases** | — |
| Eval set | ~30–40 hand-crafted, dev/test split | Git-mined 100+ (defensible metrics) |
| Validation pass (LLM) | **Optional / lightweight** — a human reviews every suggestion, so the human *is* the validator | **Essential** once auto-fix lands (no human in loop) |

---

## 7. Evals design (detail)

- **Unit:** `{ before-code, after-code (diff), doc section, label: stale|not-stale, [correct fixed doc] }`.
- **Split:** a **dev set** for tuning the threshold and a **held-out test set** for reporting — never tuned against.
- **Positives:** renamed parameter, changed default, removed capability, new required argument, changed behavior, **plus gray-zone** (intent-vs-mechanism, partial behavior change).
- **Negatives:** whitespace/formatting, comment-only, behavior-preserving refactor, test-file changes, **plus change-mapping edge cases** (decorator changes, shared-import changes, reflows).
- **Report (v1):** confusion-matrix **counts** (TP/FP/FN/TN, FN prominent) + **failure taxonomy**, measured **end-to-end** (parser + filter + link graph + verifier); correction quality via spot-checked LLM-judge + human rating. **No quoted point-estimate precision/recall until the git-mined set** — n is too small to mean anything.
- **Automation:** one script re-runs the whole set; run in CI to catch regressions.

---

## 8. Open items — pinned for build time

- **Exact Voyage code-embedding model name** (confirm from Voyage docs at build).
- **Exact tree-sitter packages** (core + Python grammar; confirm current names).
- **Budget-cap value N** (max verifications/PR) — tune empirically against cost/latency.
- **Confidence measurement (for v2 auto-fix):** change-type heuristics and/or agreement between generate & validate passes (LLM self-reported confidence is unreliable). Design when v2 is scoped.

*(Resolved in v2: index lifecycle, cold-start, huge-PR handling, change-mapping, and eval methodology were open items in v1 and are now specified above.)*

---

## 9. Explicitly rejected / out of scope (and why)

- **Regex code parsing** — cannot reliably parse a language.
- **LLM-based chunking / LLM-built link graph** — wasteful / non-deterministic / too costly for solved or expensive-at-scale problems.
- **Dumping whole repo + docs into an LLM per PR** — fails on cost, context limits, accuracy.
- **Auto-fix in v1** — trust earned via evals first.
- **Hosted vector DB in v1** — YAGNI at current scale.
- **Posting "docs are fine ✅"** — the tool advises, never certifies (principle #8).
- **Detecting missing docs for brand-new code** — different feature; out of v1.

---

## 10. Presentation (how the differentiation becomes visible)

This is a common portfolio shape; the reasoning in this doc is invisible to someone skimming the repo. So the differentiator ships as **evidence, not architecture**:

- The **README leads with real evidence** on a real repo (e.g. FastAPI): the **confusion-matrix counts** and concrete before/after catches — *not* a fake-precise precision/recall point estimate (see D7), and *not* a stack diagram. Quoted precision/recall appears only once the git-mined set makes it defensible.
- It **shows honest failure examples** — cases the tool gets wrong — because deliberately surfacing failures reads as confidence and calibration, not weakness.
- The **demo shows the tool catching a real stale doc** on a real PR, end to end.
- The design doc (this file) and the "adversarial review → v2 hardening" story are the *depth* an interviewer finds when they dig — but the numbers are what they see first.
