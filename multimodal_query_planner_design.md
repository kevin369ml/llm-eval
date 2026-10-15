# Multimodal Query Understanding and Search Planning

## 1. Purpose and scope

Build a domain-agnostic, question-conditioned planner that transforms a user image, a text question, and optional context into text queries for the Perplexity Search API.

**In scope:** input normalization; multimodal question understanding; visual evidence extraction; entity hypotheses; constraints; information gaps; structured search planning; validation; dispatch of text queries to Perplexity Search.

**Out of scope:** retrieval-result analysis, follow-up search rounds, reverse-image search, image indexing, webpage image extraction, reranking, metadata enrichment, and final answer generation.

## 2. Available components

- **Gemini 3.1 Pro:** multimodal input (image + text), question-conditioned analysis, and structured search-plan generation.
- **Perplexity Search API:** text-only web search; receives planned queries, not images.
- **Application layer:** input handling, schema validation, query deduplication, ordering, batching, observability, and error handling.

No reverse-image search or public-web visual embedding index is assumed. A visual embedding model alone cannot search the open web without an indexed corpus.

## 3. Architecture

```text
Image + User Question + Optional Context
                  |
                  v
         Input Normalization
                  |
                  v
    Gemini 3.1 Pro: Single-Pass Planner
      |-- Question and answer type
      |-- Question-conditioned visual evidence
      |-- Entity candidates and uncertainty
      |-- Constraints and information gaps
      `-- Complementary text search queries
                  |
                  v
       Structured SearchPlan JSON
                  |
                  v
     Validation / Deduplication / Ordering
                  |
                  v
       Perplexity Search API (text)
```

The four planner activities are logical responsibilities, not necessarily four separate model calls.

## 4. Design principles

1. **Question-first:** inspect the image for evidence relevant to the user's actual information need, not merely to produce a generic caption.
2. **Joint multimodal understanding:** combine text constraints with visual observations; do not discard either modality.
3. **Domain-agnostic:** support people, products, places, events, news, music, games, sports, politics, and unknown subjects.
4. **Evidence-aware:** separate user assertions, direct visual observations, OCR, and model hypotheses.
5. **Uncertainty-aware:** represent ambiguous or unidentified entities explicitly; never turn a tentative guess into a fact.
6. **Search-oriented:** produce complementary text queries that can be sent directly to a text-only search API.
7. **No fabricated evidence:** the planner must not invent prices, identities, dates, image URLs, or search results.
8. **Bounded complexity:** generate 2–5 queries by default, with fewer allowed when sufficient; use a single model pass in the MVP.

## 5. Semantic representation

Avoid routing solely on interrogative words (what, who, where, when, how). Instead, classify along independent, multi-label dimensions:

| Dimension | Examples |
|---|---|
| `answer_types` | `entity_identity`, `numeric_fact`, `location`, `date_time`, `explanation`, `comparison`, `event_summary`, `listing` |
| `subject_types` | `person`, `product`, `place`, `event`, `music`, `game`, `organization`, `document`, `object`, `unknown` |
| `evidence_requirements` | `entity_verification`, `visual_match`, `current_fact`, `historical_source`, `official_source` |

These labels describe information needs and are not mutually exclusive. The planner should not rely on a closed list to represent all possible domains.

## 6. Planner responsibilities

### 6.1 Question understanding

Normalize the question; identify the requested answer type, referenced object(s), geographic or temporal context, and explicit constraints (e.g., currency, new versus used, preferred source, language).

### 6.2 Question-conditioned visual understanding

Extract only relevant evidence: object attributes, logos, visible text/OCR, landmarks, spatial relations, UI elements, scoreboards, dates, and event clues. When a question refers to a region (e.g., “person on the left”), preserve that grounding.

### 6.3 Entity and evidence modeling

Record possible entity names as candidates, not established identities, unless the evidence supports a high-confidence identification. Preserve provenance, such as `user_text`, `image_observation`, `ocr`, or `model_hypothesis`. A user-provided name can be a search clue without necessarily being a verified fact.

### 6.4 Search planning

Select a strategy:

- **Entity-first:** reliable name/title/model is available.
- **Attribute-first:** identity is unclear; use discriminative visual features and textual constraints.
- **Event-first:** identify event context using teams, scores, locations, dates, headlines, or other clues.
- **Hybrid:** combine a candidate-entity query with an attribute-based fallback and a verification query.

Possible query purposes: `verify_entity`, `retrieve_fact`, `visual_description`, `find_images`, `disambiguate`. `find_images` is still a **text query** for image-related pages; it is not reverse-image search.

Generate complementary queries rather than near-duplicate paraphrases. Do not contaminate every query with the same uncertain model guess.

## 7. SearchPlan output contract

The canonical JSON object should contain:

- `question`: original/normalized question, `answer_types`, `subject_types`.
- `target`: referenced subject/region, `visual_evidence`, `visible_text`, `entity_candidates` (name, status, evidence/provenance).
- `constraints`: geography, time, currency, language, and other user-specified filters; allow null or empty values.
- `information_gaps`: facts still requiring search.
- `evidence_requirements`: evidence needed to support an answer.
- `retrieval_strategy`: one of `entity_first`, `attribute_first`, `event_first`, `hybrid`.
- `search_queries`: array of `{id, query, purpose, priority, dependencies}`.

Illustrative output (not a verified identification):

```json
{
  "question": {
    "original": "What is the current price of these headphones?",
    "normalized": "Find the current selling price of the pictured headphones",
    "answer_types": ["numeric_fact"],
    "subject_types": ["product"]
  },
  "target": {
    "reference": "headphones in the image",
    "region": null,
    "visual_evidence": ["black over-ear headphones", "Sony logo"],
    "visible_text": [],
    "entity_candidates": [
      {"name": "Sony WH-1000XM5", "status": "hypothesis", "provenance": "model_hypothesis"}
    ]
  },
  "constraints": {"location": null, "time": "current", "currency": null, "other": []},
  "information_gaps": ["exact model", "current selling price"],
  "evidence_requirements": ["entity_verification", "current_fact"],
  "retrieval_strategy": "hybrid",
  "search_queries": [
    {"id": "q1", "query": "Sony WH-1000XM5 current retail price", "purpose": "retrieve_fact", "priority": 1, "dependencies": []},
    {"id": "q2", "query": "Sony WH-1000XM5 headphones model identification", "purpose": "verify_entity", "priority": 2, "dependencies": []},
    {"id": "q3", "query": "Sony black over ear wireless headphones model", "purpose": "disambiguate", "priority": 3, "dependencies": []}
  ]
}
```

For production, define and enforce a machine-readable JSON Schema or typed model rather than relying on prompt text alone. Treat example values as illustrative, not ground truth.

## 8. Execution flow

1. Accept image, question, and optional `locale`, `country`, `current_date`, and user constraints.
2. Render `search_planner.j2` with text/context fields; attach the **actual image** as a separate multimodal input part (do not interpolate binary image bytes into Jinja).
3. Call Gemini 3.1 Pro once with structured JSON output enabled where supported.
4. Validate the response against the SearchPlan schema. Reject or retry malformed output with a bounded retry policy.
5. Check query relevance and constraints; remove exact/near duplicates and order by priority. Avoid rewriting uncertain hypotheses into hard filters.
6. Send the resulting text queries to Perplexity Search, batching according to current API limits.
7. Stop at API dispatch. Processing search results is explicitly out of scope.

## 9. Edge cases and safeguards

- **No recognizable entity:** use attributes, OCR, location clues, and uncertainty-preserving queries.
- **Multiple subjects:** preserve the referent indicated by the question (e.g., left/right/foreground).
- **Text-image disagreement:** represent the discrepancy; do not silently assume either is correct.
- **Temporal queries:** pass an explicit reference date when available; do not infer “today” from the image.
- **Identity-sensitive queries:** do not assert a person's identity from appearance alone; use text clues and externally verifiable context where appropriate.
- **Search insufficiency:** a text-only search API cannot guarantee the exact pictured object will be found.
- **Privacy:** avoid unnecessarily including sensitive personal information from images in generated search queries.

## 10. Evaluation

Use a cross-domain test set spanning what/who/where/when/how, multiple subjects, ambiguous images, screenshots, text-image conflicts, and sparse visual clues. Track:

| Metric | Goal |
|---|---|
| Intent accuracy | Correct user information need |
| Target grounding accuracy | Correct referenced image subject |
| Constraint preservation | Explicit constraints retained |
| Entity hypothesis calibration | Tentative names not presented as facts |
| Query relevance/diversity | Complementary, useful text queries |
| Schema validity | Parseable, contract-compliant JSON |
| Cost and latency | Bounded planning overhead |

Downstream retrieval quality can be evaluated later, but is outside this design's current execution boundary.

## 11. Prompt architecture decision

**MVP: one prompt template is sufficient.** Use `search_planner.j2` for all four logical tasks (question understanding, visual grounding, entity/evidence modeling, and query planning). Add a strict JSON Schema at the API layer, plus deterministic code validation; neither requires another LLM prompt.

**Optional future prompts, not needed now:**

- `search_plan_repair.j2` only if bounded automatic repair of invalid output becomes necessary (prefer native structured output and validation first).
- `search_plan_refinement.j2` only if a later phase adds result-aware iterative searching.
- Domain-specific prompt variants only after evaluations demonstrate a consistent domain-specific failure mode.

Avoid separate captioning, entity-extraction, and query-generation prompts in the MVP: they increase latency and can lose question-conditioned visual information between stages.
