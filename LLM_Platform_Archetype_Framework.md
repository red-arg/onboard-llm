# LLM Platform: Archetype Framework

Oct 29, 2026 · @Zhi Feng

## Purpose

The [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) selects models for archetypes, not for individual products. An archetype is a group of products that need the same kind of work from a model, so one model, one eval suite and one gateway alias can serve all of them.

This document is the minimum method for creating and maintaining archetypes while the platform is getting started. It uses the repositories in the [technical implementation](LLM_Platform_Model_Onboarding_Technical_Implementation.md), but does not need its pipelines to be in place yet.

## Rules

1. **Group by task and output.** Use cases with the same task and output belong to the same archetype, unless the test in Step 3 shows that one model cannot serve them all.
2. **Two products minimum.** Create an archetype only when at least two products need it, live or planned within two quarters.
3. **Smallest change first.** Join an existing archetype, or extend it with a language or a new sample set, before creating a new one.
4. **Each product keeps its own bar.** Every product's floor is checked on its own samples, so an average never hides one product's failure.
5. **Decide with scores.** Test models on the products' own samples before creating an archetype.

## Getting started

1. Name an owner for each starting archetype in the table below. One platform model owner can own several.
2. Ask each product team on the roadmap for one profile per use case, with 50 to 100 labeled samples (section 1).
3. Sort the profiles into the table by task and output. Postpone any archetype with fewer than two products.
4. For the two or three archetypes with the most products, write the definition (section 3) and build the eval (section 4). The board approves the starting archetypes together. They skip the test in Step 3, because no models are hosted yet to compare.
5. Onboard each archetype's first model through the model onboarding process, with two of its products as pilots.
6. Handle every later product need with the steps in section 2, and review this document's thresholds after the first few archetypes.

| Archetype | Id | Task / output | Primary metric | Alias |
| --- | --- | --- | --- | --- |
| Q&A / RAG | `rag` | `answer` / `prose` | Answer correctness, faithfulness | `rag-default` |
| Summarization | `summarization` | `condense` / `prose` | Key-point coverage, faithfulness | `summarization-default` |
| Extraction and classification | `extraction` | `extract` / `json` | Field-level F1 | `extraction-default` |
| Code assistance | `code` | `code` / `code` | pass@1 | `code-default` |
| Agents and tool use | `agents` | `act` / `tool-calls` | Task success | `agents-default` |
| Embeddings and search | `embeddings` | `embed` / `vector` | nDCG@10 | `embeddings-v1` |
| Multilingual | `multilingual` | `translate` / `prose` | chrF++ or COMET | `multilingual-default` |

Embedding aliases carry a version. Vectors from different models cannot share an index, so a new embedding model gets a new alias (`embeddings-v2`), and products re-index before the old alias retires.

## 1. Describe each use case

Each product fills in one profile per use case. The same profile describes the use case when the product requests an alias at Stage 10.

**Task:** what the model must do.

| Task | The model must | Starting archetype |
| --- | --- | --- |
| `answer` | Answer questions from sources the product sends | `rag` |
| `condense` | Summarize content | `summarization` |
| `extract` | Pull fields or labels out of content into JSON | `extraction` |
| `compose` | Draft or rewrite text | None yet |
| `code` | Write, complete or fix code or queries | `code` |
| `act` | Choose and call tools | `agents` |
| `translate` | Translate between languages | `multilingual` |
| `embed` | Turn text into vectors for search | `embeddings` |
| `rank` | Rerank search results | None yet |

**Output:** `prose` (text for a person), `json`, `code`, `query` (one executable statement, such as SQL), `tool-calls`, `vector` or `score`.

A use case with two tasks, such as answering from documents and drafting emails, becomes two profiles, and the product calls two aliases. Answers that must come from the model's own knowledge rather than from supplied sources are not `answer`, and no starting archetype covers them.

```yaml
# onboarding-config/profiles/<product>-<use-case>.yaml
product: <product>
owner: <product owner>
use_case: <what the product asks the model to do, in one line>
status: live                          # live | planned, with the quarter
task: answer
output: prose
latency: interactive                  # real-time (first token within 300 ms) | interactive (1.5 s) | batch
lengths: { input_p95: 9000, output_p95: 600 }             # tokens
volume: { requests_per_day: 5000, peak_per_minute: 40 }
languages: [en]
max_data_classification: internal
risk: { customer_facing: no, automated_decisions: no, regulated_data: no }   # any yes: risk and legal review at Stage 10
acceptance: { metric: answer_correctness, floor: 0.85, max_critical_errors: 0.03 }
cost_target_per_1k: <amount, if known>
samples: eval-data/domain/<archetype>-<slice>/seed/       # 50 to 100 labeled samples
```

Take the metric from the table in section 4. For `max_critical_errors`, use 1% when outputs reach customers or drive decisions without review, and 3% when a person checks every output.

A profile is complete when every field is filled, every sample has an expected output, and two people labeling the same 30 samples agree on at least 27. If they agree less often, fix the criteria first, as in gap diagnosis Step 1.

## 2. Create an archetype

| Step | What happens | Who | Done when | Target |
| --- | --- | --- | --- | --- |
| 1. Propose | Product teams submit profiles and samples | Product teams | Profiles complete | 1 week |
| 2. Fit check | Compare task and output with existing archetypes (first table below) | Platform model owner | Outcome recorded | 2 days |
| 3. Test | Score models on the samples (second table below) | ML evaluation team | Scores and decision recorded | 2 weeks |
| 4. Approve | Write the definition (section 3); data governance sets the classification | Model review board | Definition approved | Next board meeting |
| 5. Build | Build the eval (section 4) and check it | ML evaluation team | Archetype `active`, and Stage 0 opened for its first model | 2 to 3 weeks |

### Fit check (Step 2)

| The use case | Outcome |
| --- | --- |
| Matches an archetype's task and output, and fits its latency, context length, languages and data classification | Join it through Stage 10, and add the samples to its eval as a new slice |
| Matches, but needs another language or a higher data classification | Extend the archetype: add a language slice, or have data governance raise its classification, which each model behind it then needs approval for. Re-test its models |
| Matches, but failed its Stage 10 domain test | Run gap diagnosis. If it finds a catalog gap that two or more products share, go to Step 3 |
| Has a task or output no archetype covers, and at least two products need it | Go to Step 3 |
| Has a task or output no archetype covers, and only one product needs it | Not now: use the nearest archetype with the best prompt, and revisit when a second product needs it. The board may accept a single product when no archetype can produce its output at all |

### Test (Step 3)

On each product's samples, the ML evaluation team scores:

- the model behind the nearest archetype's alias, and the other hosted models that can produce the output;
- promising open-weight candidates, on samples that data governance clears for outside use, as for reference models in gap diagnosis.

It also runs the nearest archetype's own eval on the same hosted models, if that has not been done already.

Use an improved prompt for every model (gap diagnosis Step 2), so that a prompt problem does not look like a model gap. On 100 samples, scores vary by about 7 points by chance. Treat differences of 5 points or less as ties, and add samples before acting on a smaller difference.

Read the table from the top; the first row that matches decides:

| Result | Decision |
| --- | --- |
| The model behind the nearest archetype's alias meets every product's floor | Join that archetype. No new archetype |
| No open-weight model meets the floors, hosted or candidate | Not now: redesign the workflow (gap diagnosis Step 5), and re-test at the next quarterly re-evaluation |
| The model the nearest archetype's eval ranks first is within 5 points of the best hosted model on the samples | The archetype is right but its models are too weak: onboard a stronger model for it through Stage 0 |
| Otherwise | New archetype: the existing eval would pick the wrong model for these products |

## 3. Define the archetype

The archetype's owner writes the definition. Each requirement comes from the member products' profiles:

| Requirement | Rule |
| --- | --- |
| Quality | Each product keeps its own floor and maximum critical-error rate, checked on its own slice |
| Latency | The strictest product's latency class |
| Context length | 1.2 × the largest (input p95 + output p95), rounded up to 8k, 16k, 32k, 64k or 128k |
| Languages | Every language any product needs, each with its own slice |
| Data classification | The highest any product sends. Every model behind the alias must be approved for it |
| Cost target | The lowest product target, where products gave one |

If one product needs a much faster or cheaper model than the rest, record it in the definition. A separate tier, such as `<id>-fast`, comes later, when at least two products need the same trade-off.

```yaml
# onboarding-config/archetypes/<id>.yaml (the example from section 8)
id: text-to-sql                       # lowercase, names the capability, never reused
owner: <platform model owner>
status: active                        # proposed | active | retired
task: code
output: query
scope: Write one read-only SQL query that answers a question over a supplied schema.
out_of_scope: [running the query (product code), general code (code), answers from documents (rag)]
products: [analytics-assistant, finance-reporting]
requirements:
  floors:                             # each product, on its own slice
    analytics-assistant: { execution_accuracy: 0.85, max_critical_errors: 0.03 }
    finance-reporting: { execution_accuracy: 0.85, max_critical_errors: 0.02 }
  latency: interactive
  context_length: 32k
  languages: [en, de]
  max_data_classification: confidential
  cost_target_per_1k: 20
eval:
  suite: text-to-sql/v1
  slices: [analytics (200), finance (200), finance-de (100)]
  safety_additions: [destructive statements, instructions injected into schema comments]
alias: text-to-sql-default
```

## 4. Build the eval

The minimum eval suite for an archetype:

- **One slice per product:** 100 labeled samples to start, grown to 200 before the first model's Stage 5 benchmark, which narrows chance variation to about 5 points.
- **One slice per extra language:** 50 to 100 samples.
- **One reference prompt,** starting from the improved prompt used in Step 3.
- **Safety:** the standard safety battery, plus one or two tests specific to the archetype, such as instructions injected into retrieved passages (`rag`), source documents (`summarization`) or tool results (`agents`), or destructive statements (`text-to-sql`).
- **Public benchmarks:** optional, as a secondary signal worth at most 20% of the ranking.
- **Pass criteria** (the G5 policy): each product's floor and maximum critical-error rate are hard criteria, and latency and cost go to review. Among models that pass, rank by the average of the product slices, with each product weighted equally.

Before marking the archetype `active`, check that the eval runs end to end offline, that its scores differ between models, and that the model it ranks first is the one Step 3 found best, or within 5 points of it.

| Task (output) | Primary metric | Critical error | Tool |
| --- | --- | --- | --- |
| `answer` | Answer correctness against reference answers, plus faithfulness | A claim the sources do not support | DeepEval |
| `condense` | Coverage of reference key points, plus faithfulness | A statement the source does not support | DeepEval |
| `extract` | Field-level F1 | A wrong value in a field marked critical | DeepEval (JSON correctness) or a field scorer |
| `compose` | Rubric score from a judge checked against human labels | A fact or promise the instructions do not allow | DeepEval (G-Eval) |
| `code` (`code`) | pass@1 on unit tests | Insecure or destructive code | LightEval custom task |
| `code` (`query`) | Execution accuracy: the query returns the reference result | The query runs and returns the expected columns and row count, but wrong values | LightEval custom task with a sandbox database |
| `act` | Task success: the end state is correct | An action the task did not ask for, or one taken without a required confirmation | DeepEval (tool correctness) |
| `translate` | chrF++ or COMET per language pair | A changed number, name or negation | LightEval custom task |
| `embed`, `rank` | nDCG@10 on domain retrieval sets | n/a | MTEB |

## 5. Keep archetypes healthy

At each quarterly re-evaluation, the owner checks each archetype:

- **Profiles changed:** re-derive the requirements (section 3) when a product adds a language, grows its volume or changes its latency.
- **Wrong model for a product:** if a product's best model has scored more than 5 points above the archetype's model at two re-evaluations in a row, run Step 3 for that product.
- **Too few products:** if an archetype has had fewer than two products for two quarters, merge it into its nearest archetype or retire it.
- **Breaking changes:** never narrow an archetype's scope or change its output in place. Create a new archetype instead, and retire the old one with the same notice as a model deprecation.

## 6. Where things live

- Profiles: `onboarding-config/profiles/<product>-<use-case>.yaml`
- Definitions: `onboarding-config/archetypes/<id>.yaml`
- Samples and slices: `eval-data/domain/<id>-<slice>/`
- Decisions (fit check, test results, board approval): `evidence/archetypes/<id>/`, through the record pipeline (P5). Until P5 runs, attach them to the pull request that adds the definition.

## 7. Add later, when needed

- **Tiers** (`-fast`, `-large`), when two or more products need the same different speed, cost or quality.
- **Stricter statistics** (confidence intervals, larger slices) for customer-facing and automated use cases.
- **Pipeline checks:** P1 accepting only active archetypes as targets, and P6 checking each alias against the archetype's data classification.
- **A scoring rule** for ordering competing proposals.

## 8. Example: text to SQL

The products and numbers are illustrative.

**Step 1.** Two products submit `code` / `query` profiles. An analytics assistant is live and interactive, with prompts up to 14,000 tokens, English only, a floor of 0.85 and a cost target of $20 per 1,000 requests. A finance reporting tool is planned for Q1 2027 and runs in batch, with prompts up to 18,000 tokens, English and German, a floor of 0.85 and a cost target of $30.

**Step 2.** No archetype has `code` / `query` (code assistance is `code` / `code`), and two products need it, so the request goes to Step 3.

**Step 3.** Execution accuracy on 80 samples per product, with improved prompts:

| Model | Code assistance eval | Analytics samples | Finance samples |
| --- | --- | --- | --- |
| M1 (behind `code-default`) | 0.78 | 0.55 | 0.50 |
| M2 (behind `rag-large`) | 0.71 | 0.71 | 0.66 |
| M3 (behind `agents-default`) | 0.64 | 0.69 | 0.67 |

No hosted model meets the floor of 0.85. An open-weight candidate scores 0.86 on 100 cleared samples, so the need is feasible. The code assistance eval ranks M1 first, but M1 is 16 to 17 points below the best model on both products' samples: the existing eval would pick the wrong model. Decision: new archetype.

**Step 4.** The definition is the one in section 3. The context length is 1.2 × (18,000 + 400) = 22,080 tokens, rounded up to 32k. Data governance sets the classification to confidential, because schemas and sample rows reveal business data. The board approves on condition that products run queries with read-only credentials, checked at Stage 10.

**Step 5.** Slices of 200 samples per product and 100 in German, scored by running each query against a sandbox database. Among hosted models, the suite ranks M2 first, consistent with Step 3, so the archetype becomes `active`. The candidate enters Stage 0, and both products pilot it at Stage 6.

## Open questions

- [ ] Who owns each starting archetype?
- [ ] Which product teams supply the first profiles and samples, and by when?
- [ ] Which data classification levels does the platform use?
- [ ] Until the model review board meets regularly, who approves a new archetype?
- [ ] When an approved model is added to a new archetype, can its Stage 3 and Stage 4 results be reused, since its weights are unchanged?
