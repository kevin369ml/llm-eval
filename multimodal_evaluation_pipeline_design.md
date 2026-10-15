# Multimodal Image Search Evaluation — Downstream Pipeline Design

**Status:** Design specification aligned with the current prompt templates (2026-10-14)  
**Scope:** Perplexity evidence ingestion, query-side reference context, per-result multimodal understanding, relevance evaluation, factuality planning and verification, deterministic gates, schemas, and aggregation.  
**Companion:** `multimodal_query_planner_design.md` covers the upstream MQU/search-planning stage; this document starts where that document stops.

## 1. Goals and boundaries

The evaluated search system receives **an original user image plus a user text question** and returns approximately **10 ranked results**, each containing a **result image and caption/metadata**. The evaluator assigns **independent, per-result** scores:

- **Image relevance (1–4):** visual match to the intended target or visual information need.
- **Caption relevance (1–4):** extent to which the caption supplies the requested answer.
- **Overall relevance (1–4):** how well image and caption *together* answer the question; not an arithmetic combination of the first two.
- **Factuality (1–3 or `null`):** evidence-grounded correctness of claims actually made or directly conveyed by the result. `null` is an abstention with a reason, not a fourth numeric grade.

The architecture uses **Gemini 3.1 Pro** for multimodal analysis and structured judgments and **Perplexity Search API** for **text-only** external evidence. No Google Search Grounding, reverse-image search, or public-web visual index is assumed. The evaluator must not imply that text search can prove the identity of an arbitrary photographed person or object.

**Core separation:** *Relevance* measures answer usefulness and result alignment; *Factuality* measures claim truth. A factually wrong answer may still be relevant; an irrelevant result may contain true claims. The reference answer must never be counted as content furnished by the evaluated result.

## 2. End-to-end architecture

```text
ORIGINAL USER IMAGE + USER QUESTION
                 |
                 v
    [MQU / search_planner.j2]
      question, visual clues,
      hypotheses, search queries
                 |
                 v
      [Perplexity Search API]
                 |
                 v
       [Evidence Ingestion]
     stable IDs, source metadata
                 |
                 v
 [reference_context_builder.j2]
    query-side target resolution,
    requirements, scoped candidate
    answers, evidence, uncertainty
                 |
                 |  SHARED QUERY-SIDE CONTEXT
                 |
SEARCH SYSTEM UNDER EVALUATION
     -> result[1..N] = image + caption
                 |
          FOR EACH RESULT
                 |
                 v
  [query_conditioned_mru.j2]
     visual observations, caption
     claims, cross-modal alignment,
     answer-bearing components
                 |
          +------+------------------------+
          |                               |
          v                               v
 [relevance_judge.j2]       [factuality_claim_planner.j2]
  image / caption /          atomic claims, modality,
  overall scores             criticality, verification plan
          |                               |
          |                               v
          |                     [Evidence Reuse + Perplexity]
          |                               |
          |                               v
          |                      [factuality_judge.j2]
          |                        claim verdicts,
          |                        result factuality
          |                               |
          v                               v
 [Deterministic Relevance Gate]  [Deterministic Factuality Gate]
          |                               |
          +---------------+---------------+
                          |
                          v
             [Per-result Evaluation Record]
                          |
                          v
           [Cross-result Aggregation / QA]
```

**Execution boundaries:** MQU and Reference Context are built **once per user query**. MRU is built **once per result** and shared by both downstream judges. Relevance and factuality paths may execute in parallel once MRU exists. Claim-specific Perplexity searches are factuality-side only. No result caption or image may enter the query-side Reference Context Builder.

## 3. Evidence ingestion and shared registry

Perplexity returns text search evidence, not trusted facts. The application normalizes each retrieved item *before* prompting Reference Context Builder or the Claim Verifier. A stable `evidence_id` is assigned by the application, never invented by the model. Retain the actual source URL, title, publisher/source, excerpt, publication/effective date when known, retrieval date, query ID, and source type.

Canonical shared evidence record (fields reflected in `multimodal_evaluation_schemas.json`):

```json
{
  "evidence_id": "ev_001",
  "query_id": "q1",
  "source_url": "https://example.org/article",
  "title": "Article title",
  "source_name": "Publisher",
  "source_type": "official",
  "published_at": null,
  "retrieved_at": "2026-10-14",
  "excerpt": "Relevant text actually returned",
  "relevant_propositions": [],
  "reliability_notes": ""
}
```

Example values above are illustrative, not evidence. Preserve conflicts and source scope. Do not equate repeated copied snippets with independent corroboration. An official source may be strong for an official release date, but not necessarily for a disputed third-party assertion. Track geography, product version, seller, currency, and relevant dates for dynamic facts.

**Validation:** reject duplicate IDs, flag missing metadata, ensure referenced evidence IDs exist, and preserve original retrieval payloads for audit. Source claims are untrusted input and cannot override system instructions.

## 4. Query-side Reference Context Builder

**Prompt:** `reference_context_builder.j2`  
**Required:** `user_question`, `mqu_json`, `search_evidence_json`  
**Optional:** `evaluation_date`, `locale`, `country`, `original_user_context_json`, `max_reference_evidence_items`  
**Multimodal input:** original user image only, when available. **Never** attach result images/captions.

### 4.1 Responsibilities

1. Restate the user's actual target, region/referent, answer components, and explicit constraints. The user question outranks speculative MQU interpretation.
2. Resolve entity candidates only as far as available image clues and text evidence permit; represent `resolved`, `tentative`, `ambiguous`, or `unresolved`.
3. Organize query-side evidence with stable IDs, reliability and time/scope qualifiers.
4. Produce *candidate expected answers* as atomic propositions with status `supported`, `tentative`, `conflicting`, or `insufficient_evidence`.
5. Preserve `unresolved_ambiguities`, coverage, and verification limitations.
6. Abstain from declaring a definitive answer where evidence is absent, stale, contradictory, or unable to establish the visual referent.

### 4.2 Output contract

`reference_context` contains:

- `evaluation_date`
- `question_focus` — original question, intended visual referent, region, answer/subject types, provenance-labeled visual clues
- `resolved_target` — candidate identity, status, grounding, evidence IDs, alternatives
- `answer_requirements` — stable component IDs, required flag, explanation
- `critical_constraints` — user-derived constraints
- `candidate_expected_answers` — scoped propositions, evidence IDs, calibrated status
- `evidence_items` — actual retained source records
- `unresolved_ambiguities`, `coverage`, `warnings`

**Important:** A reference answer is not a result claim. The reference context informs interpretation and verification but must not fill missing answer content in relevance scoring.

## 5. Multimodal Result Understanding (MRU)

**Prompt:** `query_conditioned_mru.j2`  
**Required:** `user_question`, `result_caption`  
**Optional:** `query_context_json`, `result_id`, `evaluation_date`  
**Multimodal input:** actual **result image**; original user image is not a required MRU input. Pass query-side MQU context, **not Perplexity reference evidence**, to avoid anchoring result observations to an external expected answer.

### 5.1 Responsibilities

MRU describes only what the result **shows and says**, conditioned on the user's information need. Separate:

- `visual_content`: observable entities/attributes, visible text, uncertainty.
- `caption_content`: mentioned entities and atomic factual *assertions*, not verified truths.
- `cross_modal_alignment`: `aligned`, `uncertain`, `contradictory`; supporting/conflicting observations.
- `answer_content`: answer components actually supplied by the result and those missing.
- `uncertainties`: ambiguity, missing pixels, unreadable text, uncertain identities.

MRU **must not** browse, verify claims, assign relevance/factuality scores, or infer that a caption-only date or price is false because it is not visible in the picture. OCR establishes displayed words, not their real-world truth.

### 5.2 Alignment semantics

- `aligned`: positive cues support correspondence between caption and depicted subject.
- `uncertain`: correspondence cannot be confirmed or refuted.
- `contradictory`: clear visual evidence directly conflicts with the caption's **central depicted subject/event/object**.
- **Complementary content** (caption adds non-visible facts, or a generic illustrative image accompanies related text) is not by itself contradictory. The current MRU output enum does **not** have `complementary`; encode compatibility as `aligned` when justified, otherwise `uncertain`. Adding a fourth status would require updating MRU, Relevance Judge, and schema together.

**Design refinement proposed by review:** MRU alignment should be treated as a *candidate signal*. The Relevance Judge confirms a concrete conflict and reports `alignment_gate.triggered`. This resolves the tension between a simplistic `MRU.contradictory => 1` rule and the current judge's `disputed` mechanism. This refinement should be implemented consistently in prompt and code before production.

## 6. Relevance evaluation

**Prompt:** `relevance_judge.j2`  
**Required:** `user_question`, `mqu_json`, `reference_context_json`, `mru_json`  
**Optional:** `result_id`, `evaluation_date`, `result_caption`  
**Multimodal:** original user image and result image when available for consequential visual checks.

### 6.1 Scoring

| Score | Image relevance | Caption relevance | Overall relevance |
|---|---|---|---|
| 1 | Wrong/no useful visual relation | Wrong subject/unrelated/critical constraint failure | Wrong target, critical mismatch, or confirmed alignment gate |
| 2 | Same broad topic/partial cues | Topic-related but core answer absent | Related/contextual, core question unanswered |
| 3 | Strong match with minor ambiguity | Core answer, noncritical detail missing | Core answer, minor gap/ambiguity |
| 4 | Clear intended target or visual need | Complete requested answer | Joint result sufficiently answers question and constraints |

**Overall is not an average, maximum, or minimum.** A correct game cover may have image relevance 4 while its caption lacks the requested release date, resulting in overall 2. Conversely a result image may depict a match and a caption provide the winner, making overall 4.

A caption giving the requested **wrong** date can be relevant (because it directly attempts to answer) while factuality is 1. The Reference Context may help disambiguate target but may not contribute answer coverage.

### 6.2 Alignment gate and deterministic validation

The judge outputs `alignment_gate: {mru_status, triggered, disputed, reason}`. Only a **confirmed, concrete** central image-caption contradiction triggers the hard gate. The application enforces:

```python
if relevance_output["alignment_gate"]["triggered"]:
    relevance_output["overall_relevance"]["score"] = 1
```

If MRU alleges a contradiction but concrete support is absent, set `disputed=true` and send for review/MRU retry. The current prompts partially differ in how they phrase the gate; this is an **identified integration issue**, not an already implemented fix.

**No cross-metric leakage:** do not derive factuality from relevance, or vice versa.

## 7. Question-conditioned factuality claim planning

**Prompt:** `factuality_claim_planner.j2`  
**Required:** `user_question`, `mqu_json`, `reference_context_json`, `mru_json`  
**Optional:** `result_id`, `result_caption`, `evaluation_date`, `existing_evidence_json`, `max_search_queries`, `verify_incidental_claims`  
**Multimodal:** original user image and result image when available.

### 7.1 Claim extraction

Extract **all meaningful, independently checkable factual claims from the result**, including non-question-critical claims. The *criticality assignment*, not extraction eligibility, is question-conditioned.

Each claim includes:
- `claim_id`, atomic `proposition`, subject/predicate/value;
- `source_modality`: `image`, `caption`, `both`, `cross_modal`;
- `source_anchors`: exact caption excerpt and/or concrete visible observation;
- `claim_origin`: `explicit`, `strongly_implied`, `visually_observed`;
- `criticality`, `criticality_reason`, `answer_component_ids`;
- temporal type/scope, verification method/feasibility;
- candidate evidence IDs, `requires_new_search`, verification notes.

**Claim source distinctions:** a visual observation (“red car”) can be verified from pixels; text printed in an image (“released in 2025”) is a displayed assertion whose *truth* still requires external evidence; caption claims need independent checking; cross-modal claims bind a caption referent to a depicted subject. Do not invent an implicit identity claim for generic illustrative imagery.

### 7.2 Criticality

| Label | Question-relative test | Typical effect if wrong |
|---|---|---|
| `critical` | Would falsity invalidate the requested answer, its target binding, or a required constraint? | Reliably contradicted => factuality 1 |
| `supporting` | Helps interpret answer but is not indispensable | Material error often => 2 |
| `incidental` | Supplemental fact; removing/correcting it does not change answer | May lower score; truly trivial error may still allow 3 |

Criticality is **not** determined by image prominence or how confidently a caption is written. Cross-modal claims are **not automatically critical**.

### 7.3 Search planning

Reuse existing on-point evidence first. For unresolved claims, generate neutral, claim-specific Perplexity text queries. Prioritize critical, then supporting, then incidental, within `max_search_queries`. Do not request Perplexity for visually decidable color/shape. Do not claim a textual page verifies a pictured unknown face without a grounded link.

**Output:** `claim_plan` with `question_focus`, `claims`, `search_queries`, `excluded_nonfactual_content`, and `planning_warnings`. This stage outputs **no verdicts and no scores**.

## 8. Factuality evidence verification and result scoring

**Prompt:** `factuality_judge.j2`  
**Required:** `user_question`, `mqu_json`, `reference_context_json`, `mru_json`, `claim_plan_json`, `verification_evidence_json`  
**Optional:** `result_id`, `result_caption`, `evaluation_date`  
**Multimodal:** original user image and result image when visual verification is necessary and available.

### 8.1 Per-claim verdicts

Every planned claim must appear exactly once with its original ID and criticality:

- `supported`: sufficient claim-specific evidence for the full proposition and qualifiers.
- `contradicted`: affirmative, reliable refuting evidence.
- `partially_supported`: some components/qualifiers supported; ideally trigger atomic-claim refinement.
- `insufficient_evidence`: checkable but missing, conflicting, stale, or inadequate evidence.
- `not_verifiable`: subjective or impossible to adjudicate with available mechanism.

Preserve `evidence_ids`, `visual_evidence`, `evidence_assessment`, and `confidence`. A missing search result is not evidence of falsehood. Do not mistake text printed in an image for proof of the text's truth.

### 8.2 Score rubric

| Result | Rule |
|---|---|
| **1** | Any reliably contradicted **critical** claim (mandatory); exceptionally pervasive severe falsehoods without a single critical contradiction, explicitly justified |
| **2** | Critical claims adequately resolved and not contradicted; material noncritical errors or limited non-core partial inaccuracies |
| **3** | Important claims supported; no material error; an explicitly justified trivial incidental error may be tolerated |
| **`null`** | Critical claim unresolved, criticality review needed, or no scorable claims; carry a separate `scoring_status` |

There are **no independent image-factuality and caption-factuality scores**. Use claim source modality and `diagnostics` to localize errors. Factuality is not a mean of per-claim verdicts.

A **critical cross-modal identity** assertion (“this pictured tower is Tokyo Tower”) that is reliably false triggers factuality 1. However, `MRU.cross_modal_alignment == contradictory` or `relevance == 1` **alone does not** trigger factuality 1; the claim itself needs adequate refuting evidence.

### 8.3 Deterministic factuality gate

```python
critical_false = any(
    c["criticality"] == "critical" and c["verdict"] == "contradicted"
    for c in factuality_output["claim_verdicts"]
)
if critical_false:
    factuality_output["factuality"].update({
        "score": 1,
        "scoring_status": "evaluated",
        "hard_gate_triggered": True
    })
```

Before accepting a verdict, validate its evidence IDs and the judge's evidence assessment. The gate is deterministic **conditional on an adequately grounded verdict**; code alone cannot establish whether an LLM's contradiction is well supported.

If the judge proposes a material `criticality_review`, return `needs_criticality_review` and replan/reevaluate before a definitive 2 or 3. **Proposed operational refinement:** treat unresolved `partially_supported` critical claims as requiring refinement/abstention unless their critical proposition is affirmatively contradicted. This is a recommended extension, not yet guaranteed by the current prompts.

## 9. Interfaces and unified JSON Schema

**Current registry:** `multimodal_evaluation_schemas.json` (Draft 2020-12). It contains `$defs` for:

| Definition | Producer | Consumer |
|---|---|---|
| `mqu` | `search_planner.j2` | Reference, MRU context, both judges |
| `evidence_item` | application evidence ingestion / Reference | Reference, claim planner, factuality judge |
| `reference_context` | `reference_context_builder.j2` | Relevance, claim planner, factuality judge |
| `mru` | `query_conditioned_mru.j2` | Relevance, claim planner, factuality judge |
| `relevance` | `relevance_judge.j2` | deterministic gate, per-result record |
| `atomic_claim`, `claim_plan` | `factuality_claim_planner.j2` | Perplexity claim search, factuality judge |
| `claim_verdict`, `factuality` | `factuality_judge.j2` | deterministic gate, per-result record |
| `evaluation_bundle` | application | storage, aggregation, QA |

Validate each module against `#/$defs/<module>` and validate the combined bundle when all stages finish. Keep JSON schema and prompt enums synchronized when modifying fields. The schema currently checks structure and some conditional scoring constraints; **semantic cross-document invariants still require application code**:

1. All `claim_plan.claims[*].claim_id` appear **exactly once** in `factuality.claim_verdicts`.
2. Claim proposition/source/criticality agree between plan and verdict unless a documented replan occurs.
3. Every evidence ID resolves to an actually retrieved evidence record.
4. Every `answer_component_id` refers to a declared requirement, when IDs are used.
5. `alignment_gate.triggered == true` implies overall relevance 1.
6. Any reliably contradicted critical claim implies factuality 1 and `hard_gate_triggered=true`.
7. No unverified critical claim silently receives definitive 2/3.
8. Missing original/result images must be recorded; no fabricated visual observation.
9. Each result retains the same query-side MQU/reference version and evaluation date.

**Implementation caveat:** the existing one-file schema mirrors current prompt contracts; it does not itself perform all of these checks.

## 10. Orchestration, failures, and provenance

### 10.1 Normal execution

1. Normalize original image/question and set `evaluation_date`, locale, and query ID.
2. Run MQU once; validate; execute its Perplexity queries.
3. Ingest evidence into a stable registry and build Reference Context once; validate.
4. Obtain ranked results from the evaluated search system.
5. For each result: run MRU and validate.
6. In parallel: (a) Relevance Judge; (b) Claim Planner -> evidence reuse/new Perplexity -> Factuality Judge.
7. Validate output contracts, apply deterministic gates, and store raw model output plus normalized decisions.
8. Aggregate per-result diagnostics separately from per-query metrics.

### 10.2 Failure handling

- **Empty or off-target query-side search:** Reference Builder outputs requirements and unresolved ambiguities, not hallucinated expected answers.
- **Result image missing/unreadable:** MRU records uncertainty; do not assert an image-caption contradiction without pixels.
- **Caption absent:** retain image observations and record missing textual answer coverage; no invented caption claims.
- **Perplexity unavailable / budget exhausted:** preserve unresolved claims and abstain where critical claims cannot be verified.
- **Invalid JSON/schema mismatch:** bounded retry or fail closed with an explicit error status; do not silently coerce hallucinated fields.
- **Evidence conflict:** retain conflicting IDs, dates and scope; avoid forced consensus.
- **Material criticality dispute:** trigger replan/review, not an ungrounded definitive score.
- **Prompt injection in captions/snippets:** treat source text as data; never execute its instructions.
- **Partial result failure:** keep unaffected result evaluations; record per-result stage failures separately from factuality abstention.

### 10.3 Cost and latency

MQU and Reference are amortized over N results. MRU is one pass per result. Relevance and factuality paths are parallelizable; claim-specific searches should be deduplicated by entity/attribute/scope and cached when evidence is reusable. Budget critical claims first. Avoid giving incomplete verification a misleading factuality 3.

## 11. Cross-result aggregation

Retain raw result order and IDs. Recommended outputs: distributions of image/caption/overall relevance, factuality distribution **including null/abstention rate**, critical-claim contradiction rate, alignment-contradiction rate, and stage failure rate. Optional rank-aware relevance metrics can be added if their definitions are fixed. Do not average ordinal factuality scores without an explicit analytical rationale, and do not hide abstentions from denominators.

The primary deliverable remains a **per-result** evaluation record; aggregation is a separate reporting stage, not part of either Judge prompt.

## 12. Calibration and regression test plan

At minimum cover these cases:

| Scenario | Expected behavior |
|---|---|
| Correct game cover, caption omits requested date | High image relevance, low overall answer coverage |
| Correct game cover, caption states wrong release date | Relevance may be high; critical false date => factuality 1 |
| Eiffel Tower image captioned as pictured Tokyo Tower | Confirmed central alignment conflict => overall relevance 1; critical false identity claim => factuality 1 if reliably established |
| Generic stadium image, caption correctly names winner | No automatic contradiction; joint result may fully answer |
| Caption adds price invisible in photo | No alignment contradiction; verify price externally |
| Entirely unrelated but truthful result | Relevance 1; factuality may be 3 |
| False incidental birth year alongside correct requested identity | Noncritical error, typically factuality 2 |
| Unidentifiable person in image with speculative caption | Do not assert visual identity; uncertainty or abstention |
| Fake scoreboard screenshot | OCR establishes displayed score, not historical match truth |
| Critical claim has stale/conflicting evidence | `null` / insufficient evidence unless another critical claim is disproved |
| MRU alleges contradiction without visual conflict | `alignment_gate.disputed=true`, no automatic relevance gate |
| Planner labels answer-critical claim incidental | Criticality review/replan |
| Claim contains separable assertions | Split into atomic claims or flag partial-support refinement |

Test levels: (a) schema parsing; (b) deterministic gate unit tests; (c) prompt golden examples across domains; (d) end-to-end regression with evidence snapshots and explicit evaluation dates; (e) error analysis by modality, claim criticality, evidence sufficiency and source type.

## 13. Decisions already reflected in prompts vs open integration work

### Established design decisions

- Query-conditioned MQU; uncertainty-preserving entity candidates.
- Query-side Reference Context is separate from result content.
- MRU does not fact-check or score.
- Relevance uses three 1–4 scores; overall is a joint judgment.
- Factuality uses atomic claims, per-claim criticality, one 1–3 result score or abstention.
- Confirmed central image-caption conflict hard-gates relevance.
- Reliably contradicted critical claim hard-gates factuality.
- Relevance and factuality are independent.
- No reverse-image verification capability assumed.

### Identified integration work (not silently treated as implemented)

1. **Alignment gate authority:** reconcile MRU's categorical `contradictory` rule with the Relevance Judge's `triggered/disputed` confirmation; adopt a single application policy.
2. **Critical partial support:** define a definitive replan/abstention policy for `partially_supported` critical claims.
3. **Evidence registry:** implement actual stable ID assignment, de-duplication, provenance and ID referential integrity.
4. **Source taxonomy:** optionally distinguish image OCR assertions from pure visual observations in schema/prompt.
5. **Reference-to-claim linkage:** implement evidence reuse without letting reference propositions become result claims.
6. **JSON Schema + semantic validators:** enforce output contracts and cross-stage invariants in code.
7. **Versioning:** store prompt/schema/model versions, evaluation date, retrieval timestamps and source snapshots for reproducibility.

## 14. Artifact map

| Artifact | Responsibility |
|---|---|
| `multimodal_query_planner_design.md` | Upstream MQU/search planning design |
| `search_planner.j2` | MQU and query-side Perplexity query generation |
| `reference_context_builder.j2` | Query-side evidence interpretation and expected-answer context |
| `query_conditioned_mru.j2` | Result-only image/caption understanding |
| `relevance_judge.j2` | Three relevance scores and alignment gate decision |
| `factuality_claim_planner.j2` | Atomic claims, criticality, verification planning |
| `factuality_judge.j2` | Evidence-grounded claim verdicts and factuality score |
| `multimodal_evaluation_schemas.json` | Unified structured output contracts |
| **This document** | Downstream architecture, orchestration, scoring, interfaces, QA |

---

**Design invariant:** *Only the evaluated result can supply its own answer-bearing content; only grounded evidence can establish factual truth; only the original user image/question can determine what is critical.*
