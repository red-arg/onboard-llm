# LLM Platform: Archetype Framework

Oct 29, 2026 · @Zhi Feng

## Purpose and scope

The [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) selects models for use-case archetypes rather than for individual products. This framework is the method for creating those archetypes. It describes every product's need in the same fixed terms, groups the products that one model can serve, and turns each group into a specified, measurable archetype with requirements, an eval suite, a G5 policy and gateway aliases.

It also covers tiers, changes to existing archetypes, and splitting, merging and retiring archetypes. It runs on the repositories and pipelines in the [technical implementation](LLM_Platform_Model_Onboarding_Technical_Implementation.md).

In scope: the archetype catalog, including the starting catalog. Out of scope: onboarding the models behind an archetype (Stages 0 to 11) and approving a product's use case (Stage 10), which follow the process document.

Thresholds, sample sizes, latency classes and weights in this document are suggested starting values. The ML evaluation team recalibrates them after the first archetypes, through reviewed changes to this document and to the policies.

## How the framework works

| Part | Question it answers | Output |
| --- | --- | --- |
| 1. Capability model | Which terms describe any product's need? | Fixed dimensions with allowed values and a role in grouping |
| 2. Product profiles | What does each product need, in those terms? | One profile per product use case, with a labeled seed set |
| 3. Grouping method | Which products can one model serve? | A decision: join, extend, add a tier, create an archetype, or not create one |
| 4. Archetype specification | What must a model do for the group, and how is it measured? | Definition with derived requirements, eval suite plan and policy |
| 5. Creation procedure | Who does what, in which order, with which sign-off? | Gate records A1 to A4 and an Active archetype |
| 6. Lifecycle | When does an archetype change, split, merge or retire? | Review metrics and change rules |

Five rules hold throughout:

1. **Membership.** Products belong to the same archetype when the model the archetype selects is, within each product's tolerance, as good as the best available model for that product (selection regret, 3.4).
2. **Presumption.** The signature (task, output, grounding) sets the starting assumption. Scores can overturn it; an inconclusive score cannot.
3. **Smallest change.** In order of preference: join an archetype, add a slice or a language, add a tier, create a domain archetype, create a capability archetype.
4. **Traceable numbers.** Every requirement value comes from a stated rule over the members' profiles, and every policy criterion cites the requirement it checks.
5. **Equal voice.** Each member has its own floor and an equal weight in the selection score, so no product's needs are averaged away.

## 1. Capability model

Each dimension has a fixed list of values and a fixed role in grouping:

| Role | Dimensions | When two products differ on it |
| --- | --- | --- |
| Defining | D1 to D3 | They start in different groups, unless the regret test shows that one model serves both |
| Parametric | P1 to P5 | Same archetype; separate tiers if the tier cost test fails |
| Constraint | K1 to K4 | Same archetype, with extended requirements: languages, data classification, slices |
| Product-level | R1 to R4 | No effect on grouping; reviewed per product at Stage 10 and summarized in the archetype's risk profile |

### 1.1 Defining dimensions

**D1 Task:** what the model must do.

| Value | The model must | Starting archetype |
| --- | --- | --- |
| `answer` | Answer questions | `rag` (with `supplied` grounding) |
| `condense` | Shorten content and keep its meaning | `summarization` |
| `extract` | Pull fields, entities or labels out of content | `extraction` |
| `compose` | Draft or rewrite text from instructions | None |
| `code` | Write, complete, fix or explain code or queries | `code` |
| `act` | Choose and call tools over one or more steps | `agents` |
| `translate` | Carry meaning from one language to another | `multilingual` |
| `embed` | Turn text into vectors for search or similarity | `embeddings` |
| `rank` | Score candidates against a query | None |

**D2 Output:** what comes out, which decides how it is checked.

| Value | Output | Checked by |
| --- | --- | --- |
| `prose` | Free text shown to a person | Reference answers, rubrics, judge metrics |
| `json` | JSON valid against a schema, including labels | Schema validation and field-by-field comparison |
| `code` | Source code | Unit tests, compilation, static analysis |
| `query` | One executable statement, such as SQL | Running it and comparing the result |
| `tool-calls` | Function calls with arguments | Comparing calls and checking the end state |
| `vector` | Embeddings | Retrieval metrics |
| `score` | Relevance scores | Ranking metrics |

**D3 Grounding:** where the facts in the output must come from.

| Value | Facts come from | Main quality risk |
| --- | --- | --- |
| `supplied` | Only content sent with the request: passages, documents, schemas, tool results | Claims the supplied content does not support |
| `parametric` | The model's own knowledge | False or invented facts |
| `none` | No checkable facts: style or format changes, code against a full specification, vectors | Task-specific: failing tests, wrong format |

The three values form the **signature**, written `task/output/grounding`. Q&A over retrieved documents is `answer/prose/supplied`.

### 1.2 Parametric dimensions

**P1 Latency class**, measured with GuideLLM at the tier's peak concurrency (4.3):

| Class | Typical use | TTFT p95 | Inter-token p95 (streaming) | Request p95 (not streaming) |
| --- | --- | --- | --- | --- |
| `L1` real-time | Inline completion, autocomplete | ≤ 300 ms | ≤ 30 ms | ≤ 1 s |
| `L2` interactive | Chat and assistants, with a user waiting | ≤ 1.5 s | ≤ 60 ms | ≤ 10 s |
| `L3` responsive | A user starts a job and comes back | ≤ 5 s | n/a | ≤ 60 s |
| `L4` batch | No user waiting | n/a | n/a | Within the agreed batch window |

**P2 Criticality**, the consequence of an error:

| Class | The output is used | Default critical-error ceiling | Ceiling is shown by | Stage 10 route |
| --- | --- | --- | --- | --- |
| `C1` decision | By systems or for decisions, without review | 0.5% | Upper 95% bound | Risk and legal review |
| `C2` external | By customers or outside parties, at most spot-checked | 1% | Upper 95% bound | Risk and legal review |
| `C3` reviewed | By employees who check every output before use | 3% | Point estimate | Standard |
| `C4` assistive | As a suggestion the user can ignore | 5% | Point estimate | Standard |

**P3 Context band**, input p95 plus output p95 in tokens: `X1` up to 4k, `X2` up to 16k, `X3` up to 64k, `X4` up to 128k, `X5` above 128k.

**P4 Volume band**, requests per day: `V1` below 1,000; `V2` 1,000 to 10,000; `V3` 10,000 to 100,000; `V4` above 100,000.

**P5 Cost target:** the most the product can pay per 1,000 requests, from its business case.

### 1.3 Constraint dimensions

- **K1 Languages:** ISO 639-1 codes. Each language in an archetype gets its own slice and floor.
- **K2 Data classification:** the highest classification the product sends, on the platform's scheme (for example public, internal, confidential, restricted).
- **K3 Domain:** `general`, or the specialized domain (legal, finance, medical and so on) whose vocabulary or knowledge the task depends on.
- **K4 Turns:** `single` or `multi`.

### 1.4 Product-level dimensions

- **R1 Autonomy:** `none` (output shown to a person), `suggest` (a person applies it), `confirm` (the system acts after a person confirms) or `auto` (the system acts without confirmation).
- **R2 Exposure:** `internal`, `customer` or `public`.
- **R3 Trifecta legs:** private data, untrusted content and outbound actions, each yes or no (see Lethal trifecta and network isolation in the process document).
- **R4 Regulated data:** personal, financial, health or other regulated categories. Any entry sends the use case to risk and legal review at Stage 10.

## 2. Product profiles

Every product use case that uses or plans to use the platform has one profile. A product with several distinct use cases has one profile per use case, and calls one alias per use case. The same profile describes the use case when the product requests an alias at Stage 10.

### 2.1 Profile template

```yaml
# onboarding-config/profiles/<product>-<use-case>.yaml
product: <product name>
use_case: <what the product asks the model to do, in one line>
owner: <named product owner>
status: live                          # live | roadmap
planned_start: null                   # quarter, for roadmap use cases
signature:
  task: answer                        # D1
  output: prose                       # D2
  grounding: supplied                 # D3
parameters:
  latency_class: L2                   # P1
  criticality: C3                     # P2
  lengths: { input_p50: 3000, input_p95: 9000, output_p95: 600 }   # tokens; gives the context band (P3)
  lengths_source: measured            # measured | estimated
  volume: { requests_per_day: 5000, peak_per_minute: 40 }          # P4
  cost_target_per_1k: <amount>        # P5
constraints:
  languages: [en, de]                 # K1
  max_data_classification: internal   # K2
  domain: general                     # K3
  turns: single                       # K4
risk:
  autonomy: none                      # R1
  exposure: internal                  # R2
  trifecta: { private_data: yes, untrusted_content: yes, outbound: no }   # R3
  regulated_data: []                  # R4
acceptance:
  primary_metric: answer-correctness  # from the metric catalog (4.5)
  floor: 0.85
  regret_tolerance: 0.03              # default 0.03
  error_types:
    - { type: claim the sources do not support, severity: critical, max_rate: 0.01 }
    - { type: correct but incomplete answer, severity: moderate, max_rate: 0.10 }
    - { type: formatting slip, severity: minor }
seed_set: eval-data/domain/<working-id>-<slice>/seed/   # 50 to 100 labeled samples
nearest_archetype: rag
```

### 2.2 Classifying the signature

| Dimension | Question | Rule | Examples |
| --- | --- | --- | --- |
| D1 Task | What would a person doing this job be doing? | Pick one verb. A use case that needs two verbs is two use cases with two profiles | "Answer staff questions from the policy wiki" is `answer`; "Tag tickets by category" is `extract`; "Draft a reply to a customer" is `compose` |
| D2 Output | What does the product do with the output? | Shown to a person: `prose`. Parsed by code: `json`. Compiled or executed: `code` or `query`. Dispatched as calls: `tool-calls`. Stored in an index: `vector`. Used to reorder results: `score` | Free text that the product wraps in JSON only for display is still `prose` |
| D3 Grounding | If the output states a fact, where must it come from? | Only from what the product sends: `supplied`. From the model's knowledge: `parametric`. No checkable facts: `none` | RAG answers and summaries are `supplied`; a general writing assistant is `parametric`; code against a full specification is `none` |

Edge cases:

- A chat product that answers from documents and also drafts emails has two profiles (`answer/prose/supplied` and `compose/prose/supplied`) and calls two aliases.
- Classification is `extract/json/supplied`: labels are a JSON field.
- If product code fetches the data and the model only answers from it, the use case is `answer`. If the model decides which tool to call, it is `act/tool-calls/supplied`.
- Text-to-SQL is `code/query/supplied`: the schema is supplied, and the query is executed.
- When two grounding values both seem to apply, the one with the larger quality risk wins: `parametric` over `supplied` over `none`.

### 2.3 Profile quality checks

A profile enters grouping only when:

- every field is filled, and lengths are `measured` for live products, from at least two weeks of gateway metering or product telemetry;
- the acceptance criteria name a primary metric from the metric catalog (4.5) and a floor, and every error type has a severity;
- the seed set holds 50 to 100 samples of the real task with expected outputs and error labels, or synthetic samples where real data cannot be used yet;
- two people label at least 30 of those samples independently and agree with Cohen's κ ≥ 0.7 on pass or fail and on error type. Below that, the criteria or labels are fixed first, as in gap diagnosis Step 1;
- data governance has approved the seed set's classification.

The product owner is accountable for the profile; the archetype owner reviews it.

## 3. Grouping method

Grouping turns profiles into one of the outcomes in the decision table (3.6). G1 to G3 work on profiles alone and run in Step 1; G4 and G5 need scores and run in Step 2. The inputs are the profiles of the proposing products and of the current members of the nearest archetypes.

### 3.1 G1: Signature match

1. Group the profiles that have identical signatures.
2. Compare each group's signature with the signatures of existing archetypes:
   - **Match:** the group is presumed to belong to that archetype.
   - **No match:** the group is presumed to need a new capability archetype. Its nearest archetype is the existing one with the most matching signature values, with a matching task counting first.
3. Record the presumption. It decides any regret result that stays inconclusive (3.4).

### 3.2 G2: Tier cost test

Members can share one alias, and so one model, only if meeting the strictest member's needs does not make the others pay too much. When the group's signature matches an existing archetype, run the test over the group together with that archetype's current members and tiers.

1. For each member i, take the cost per request of the cheapest model that meets member i's own floors, latency class and context band: `c_i`.
2. Take the cost per request of the cheapest model that meets every member's floors, the strictest latency class and the largest context band: `c_all`.
3. If `c_all ≤ 1.5 × c_i` for every member, the members share one tier.
4. Otherwise, move the members with the lowest `c_i` into a separate tier and repeat within each tier until every tier passes.
5. A separate tier is created only for at least two members, or for members with at least 25% of the archetype's volume. A member that cannot have its own tier joins the closest tier, is told the cost difference, and decides whether to join.

Costs come from the formula in 4.3: Stage 5 GuideLLM baselines for hosted models, and SRE's estimate from parameter count and hardware for candidates. In Step 1, before costs exist, use a proxy: members share a tier if their latency classes, criticality classes and context bands are each equal or adjacent, and their cost targets are within a factor of 2.

### 3.3 G3: Constraint fit

- **Languages (K1):** the archetype's languages are the union of its members'. A language new to the archetype is a language extension: a new language slice, and every model behind the aliases is re-scored.
- **Data classification (K2):** the archetype's maximum is the highest member value, and every model behind any of its aliases must be approved for it, so that swapping models never breaks a product's approval. Raising it needs data governance, and a new onboarding for each model not yet approved at that level.
- **Domain (K3):** each specialized domain adds a domain slice. The regret test decides whether it needs an archetype of its own.
- **Turns (K4):** a multi-turn member adds a multi-turn slice.

### 3.4 G4: Selection regret test

The regret test checks with scores what G1 presumed: is the model the archetype would select good enough for every member? It runs per tier.

Setup:

- **Models:** every model behind the nearest archetypes' aliases, plus other hosted models with a compatible interface, at least three in total.
- **Prompts:** each model is scored with the optimized prompt from gap diagnosis Step 2, written within its fixed time box, so that a prompt problem does not look like a model gap.
- **Data:** each member's seed set is one slice, and every model is scored on the same samples.

For each member i:

- `best_i` is the model with the highest primary-metric score on member i's slice.
- `sel` is the model the archetype would select. For an existing archetype, it is the model its suite ranks first. For a candidate group, it is the model with the highest selection score (4.2) over the group's slices.
- **Regret** is `R_i = score_i(best_i) − score_i(sel)`, with a paired 90% confidence interval from 1,000 bootstrap resamples of member i's slice.

| Result | Meaning | Decision |
| --- | --- | --- |
| `R_i ≤ δ_i` | The selected model is good enough for member i | Pass |
| `R_i > δ_i`, and the interval excludes 0 | The selected model is clearly worse for member i | Fail |
| `R_i > δ_i`, and the interval includes 0 | Inconclusive | Grow member i's seed set to 200 samples and test once more. If still inconclusive, the G1 presumption decides |

`δ_i` is member i's regret tolerance from its profile: 0.03 on the primary metric by default.

Applying the results:

- **Membership of an existing archetype:** members that pass join it. A member that fails against an archetype with the same signature is a domain archetype candidate.
- **A group with a new signature:** if every member passes against the nearest archetype, the group joins it, adding its slices and any interface feature such as structured output. Otherwise it is a capability archetype candidate.
- **Within a candidate group:** a failing member is split off and tested as a group of its own.

When five or more models are scored, also report Kendall's rank correlation between the model rankings on each pair of slices. A value below 0.4 flags slices that order models differently even while regret is small; the archetype owner tracks them at catalog review.

### 3.5 G5: Viability

| Check | Passes when |
| --- | --- |
| Members | At least two products, live or planned within two quarters. One is enough only when no archetype offers the needed output; that archetype is merged or retired if it still has one member after two catalog reviews |
| Feasibility | For each tier, at least one open-weight model, hosted or candidate, scores at or above every member's floor, with the lower 90% bound no more than 0.05 below it. A candidate not yet onboarded runs only on a cleared subset of at least 100 samples that mirrors the seed sets and that data governance clears for outside runs, as for reference models in gap diagnosis |
| Measurability | Every requirement used for selection has a metric, and every profile passed its label check |
| Cost viability | For each tier, the cost viability rule (4.3) holds on a dedicated deployment, or on an existing deployment that can host the model |
| Prompt check | The gap remains with the optimized prompt |

### 3.6 Decision table

| G1 signature | G2 tiers | G3 constraints | G4 regret | G5 viability | Outcome |
| --- | --- | --- | --- | --- | --- |
| Matches an archetype | One tier fits | Within current | Pass, and a hosted model meets the floors | n/a | Join through Stage 10; seed sets become domain slices |
| Matches | One tier fits | New language | Pass | n/a | Language extension |
| Matches | Split | Any | Pass | Pass for the new tier | New tier |
| Matches | Any | Any | Pass, but no hosted model meets the floors | Feasible | Stage 0 trigger for a stronger model on that archetype |
| Matches | Any | Any | Fail | Pass | Domain archetype |
| New | Any | Any | Pass against the nearest archetype | n/a | Join the nearest archetype, with new slices and interface features |
| New | Any | Any | Fail, or no archetype offers the output | Pass | Capability archetype |
| Any | Any | Any | Any | Fail | Not created: defer, redesign the workflow (gap diagnosis Step 5), or re-test at the next scheduled re-evaluation |

### 3.7 Prioritizing proposals

When proposals compete for the ML evaluation team's time, rate each one (1, 3 or 5 per criterion) and work in score order. Score at A1 with estimates, using gap diagnosis results where they exist, and again at A2 with scores.

| Criterion | Weight | 1 | 3 | 5 |
| --- | --- | --- | --- | --- |
| Member products | 25% | 2 | 3 to 4 | 5 or more |
| Combined volume | 20% | V1 | V2 or V3 | V4 |
| Highest member criticality | 15% | C4 | C3 | C1 or C2 |
| Gap: floor minus best hosted score | 15% | Below 0.05 | 0.05 to 0.15 | Above 0.15 |
| Feasibility margin: best open-weight score minus floor | 15% | Below 0.02 | 0.02 to 0.05 | Above 0.05 |
| Build effort | 10% | New tools and new data | New data only | Existing tools and data |

Priority score = Σ (weight × rating) × 20, from 20 to 100. The platform lead can change the order with a recorded reason.

## 4. Archetype specification

An archetype definition holds requirements derived from its members' profiles by fixed rules: quality (4.2), operational (4.3), tiers (4.4) and the constraints from G3. Sections 4.5 to 4.7 turn the requirements into an eval suite and a G5 policy. Each requirement and criterion is set per tier.

### 4.1 Definition fields

| Field | Contents | Set by | Derived from |
| --- | --- | --- | --- |
| Identity | Id, name, kind (capability, or domain with its parent), version, status, owner | Archetype owner | Naming rules (4.8), decision table |
| Signature | D1, D2, D3 | Archetype owner | Members' profiles |
| Members | The profiles the requirements are derived from, by tier | Archetype owner | G2 |
| Scope | Tasks in scope; tasks out of scope, each with the archetype or product code that covers it | Archetype owner with sponsors | Profiles |
| Interface | API, input, output schema, serving features | Archetype owner, SRE | Profiles |
| Requirements | Quality and operational requirements per tier; languages; maximum data classification | ML evaluation team, SRE; data governance for classification | 4.2, 4.3, G3 |
| Evaluation | Suite and policy versions, slices, judge model | ML evaluation team | 4.5 to 4.7 |
| Risk profile | Members' autonomy, exposure and trifecta legs; Stage 10 focus; safety additions | Security | R1 to R4, metric catalog |
| Gateway | Alias mode and an alias per tier | Archetype owner, SRE | 4.8 |
| Lifecycle | Review cadence, related archetypes | Archetype owner | Section 6 |

Approved use cases still come from Stage 10. A member's profile is the basis for the requirements, not an approval.

### 4.2 Quality requirements

| Requirement | Rule | Hard criterion |
| --- | --- | --- |
| Member floor | Each member's slice reaches that member's floor on its primary metric | Yes |
| Critical-error ceiling | On each member's slice, the critical-error rate is at or below the stricter of the member's `max_rate` and its criticality default (P2). C1 and C2 show it with the upper 95% bound, C3 and C4 with the point estimate | Yes |
| Language floor | Each language slice reaches the strictest floor of the members that use the language | Yes |
| Long-context floor | The long-context slice reaches the strictest member floor in the tier | Yes |
| Interface validity | At least 99% of `json`, `query` and `tool-calls` outputs are valid against the schema, or parse | Yes |
| Safety additions | Each archetype-specific test stays within Security's limit | Yes |
| Selection score | Ranks the models that meet every hard criterion: 80% the mean of the member slices, each member weighted equally, and 20% the mean of the public benchmarks; 100% member slices when there are none | Used for selection, not as a floor |

### 4.3 Operational requirements

| Requirement | Rule |
| --- | --- |
| Latency class | The strictest member class in the tier (P1) |
| Peak rate | `peak_rps = Σ peak_per_minute / 60` over the tier's members |
| Peak concurrency | `peak_rps × request_p95 × 1.3`. This is Little's law, kept conservative by using p95 latency, with 30% headroom. For streaming classes, `request_p95 = TTFT p95 + output_p95 × inter-token p95`. GuideLLM measures latency at this concurrency |
| Context length | The next standard size (8k, 16k, 32k, 64k, 128k) at or above `1.2 × max(input_p95 + output_p95)` over the members |
| Cost target | The lowest member target in the tier |
| Cost per request | `gpu_hour_rate × gpus_per_replica / (3600 × sustainable_rps)`, where `sustainable_rps` is the highest rate in the GuideLLM sweep at which the latency class still holds |
| Replicas | `max(ceil(peak_rps × 1.3 / sustainable_rps), minimum_replicas)`, where SRE sets the minimum for availability, for example two |
| Cost viability | `replicas × gpus_per_replica × gpu_hour_rate × 730 / monthly_requests ≤ cost target`, with `monthly_requests = 30 × Σ requests_per_day` |

### 4.4 Tiers

| Tier | Alias | Typical members | Created when |
| --- | --- | --- | --- |
| default | `<id>-default` | Most members | Always |
| fast | `<id>-fast` | A faster latency class or a lower cost target | G2 splits them off |
| large | `<id>-large` | Floors that only a larger, more expensive model meets | G2 splits them off |

Each tier has its own members, floors, latency class, context length and cost target, all derived by 4.2 and 4.3 from its own members. The suite is shared; policy criteria are set per tier. A product moves between tiers through Stage 10, as in the "wrong tier" outcome of gap diagnosis.

### 4.5 Metric catalog

Each task family has a standard primary metric and critical-error definition, so that profiles in the same family are measured the same way.

| Task (output) | Primary metric | Critical error | Secondary metrics |
| --- | --- | --- | --- |
| `answer` (`supplied`) | Answer correctness against reference answers: rubric judge, or exact match and token F1 for short answers | A claim the sources do not support, or an answer where the sources hold none | Faithfulness, answer relevancy, abstention accuracy on unanswerable questions |
| `answer` (`parametric`) | Factual accuracy against reference answers | A false statement presented as fact | Hallucination rate, correct refusals |
| `condense` | Key-point coverage against reference key points | A statement the source does not support | Faithfulness, length compliance |
| `extract` | Field-level F1 (micro); macro-F1 for label sets | A wrong value in a field the profile marks critical | Exact match per field, schema-valid rate |
| `compose` | Rubric score for instruction adherence, correctness and tone, with a validated judge; or pairwise win rate against the incumbent | A fact, promise or claim that the instructions and sources do not allow | Length and format compliance |
| `code` (`code`) | pass@1 on unit-tested tasks | A known insecure pattern, or a destructive operation | Compile rate, static-analysis findings |
| `code` (`query`) | Execution accuracy: the query returns the reference result | A query that runs and returns a result of the expected shape (same columns and row count) with wrong values: an error a reader is unlikely to notice | Valid-query rate, error-free execution rate |
| `act` | Task success: the end state is correct | A call with side effects the task did not ask for, or made without a required confirmation | Tool-call correctness, invalid-call rate, steps per task |
| `translate` | COMET or chrF++ per language pair | A changed number, name, date or negation | Human critical-error check on a sample |
| `embed` | nDCG@10 on domain retrieval sets | n/a | Recall@k, MRR, MTEB task scores |
| `rank` | nDCG@10 after reranking | n/a | MRR |

| Task (output) | Tools | Public benchmark candidates | Safety additions |
| --- | --- | --- | --- |
| `answer` | DeepEval (answer relevancy, faithfulness, G-Eval) | SQuAD 2.0, HotpotQA; TruthfulQA and MMLU for `parametric` | Instructions injected into retrieved passages |
| `condense` | DeepEval (summarization, faithfulness) | CNN/DailyMail, XSum | Instructions injected into the source document |
| `extract` | DeepEval (JSON correctness), field-level scorer | Few fit; rely on domain slices | Injected text that changes extracted values; personal data extracted beyond the schema |
| `compose` | DeepEval (G-Eval) | None useful | Policy and brand violations, impersonation |
| `code` (`code`) | LightEval custom tasks with a sandboxed test runner | HumanEval, MBPP | Insecure patterns, secrets in output |
| `code` (`query`) | LightEval custom task with a sandboxed database | Spider, BIRD | Destructive statements; instructions injected into schema comments and sample rows |
| `act` | DeepEval (tool correctness, task completion), a custom harness with mock tools | BFCL | Instructions injected into tool results; actions beyond the task |
| `translate` | LightEval custom task with chrF++ or COMET | FLORES-200 | Instructions injected into the source text |
| `embed`, `rank` | MTEB | MTEB tasks for the archetype's languages and task types | Standard safety battery only |

Benchmarks are candidates: each passes the dataset intake rules and Legal's license review before use. Scoring models, such as COMET's, are tool models and follow the same rules. Public benchmarks count for at most 20% of the selection score because they may be in a model's training data.

### 4.6 Eval suite composition

| Slice | When | Minimum size | Used for |
| --- | --- | --- | --- |
| Member slice | One per member | From table A at the member's expected score: ±0.05 for C1 to C3 members, ±0.07 for C4 | Member floor, selection score |
| Critical-case set | Members at C1 or C2 | From table B | Critical-error ceiling |
| Language slice | Each language other than the one the member slices are written in | 100 (±0.07 near 0.85) | Language floor |
| Long-context slice | Tiers in context band X3 or above | 100 samples at 80 to 100% of the tier's context length | Long-context floor |
| Multi-turn slice | Each multi-turn member | 100 conversations | Part of that member's slice |
| Public benchmarks | Where a candidate in 4.5 fits | The benchmark's test set | Up to 20% of the selection score |
| Safety additions | Always | Set by Security | Safety criteria |

**Table A: samples for a score at 95% confidence,** `n = 1.96² × p × (1 − p) / e²`

| Expected score p | ±0.10 | ±0.07 | ±0.05 | ±0.03 |
| --- | --- | --- | --- | --- |
| 0.70 | 81 | 165 | 323 | 897 |
| 0.85 | 49 | 100 | 196 | 545 |
| 0.95 | 19 | 38 | 73 | 203 |

**Table B: samples to show a critical-error ceiling `c` at 95%** (one-sided, Poisson approximation of the exact bound)

| Critical errors observed | Minimum samples | For c = 1% | For c = 0.5% |
| --- | --- | --- | --- |
| 0 | 3.0 / c | 300 | 600 |
| 1 | 4.75 / c | 475 | 950 |
| 2 | 6.3 / c | 630 | 1,260 |

Size the critical-case set for the number of errors the members can accept seeing. If more errors occur than planned, the ceiling is not shown and the criterion fails.

Other suite rules:

- **Sealed part:** 30% of every slice is sealed, and only P4 and P7 read it. Prompt writing and judge validation use the open 70%. Criteria are computed on the whole slice; if the open and sealed parts differ by more than the slice's confidence interval half-width, the run is flagged for overfitting to the open part.
- **Judge validation:** a judge metric is used only after it agrees with human labels on at least 100 items per slice: Cohen's κ ≥ 0.6 for pass or fail judgments, or Spearman correlation ≥ 0.7 for scores. Otherwise the rubric changes, or a reference-based metric replaces it. A new judge model or rubric is a new suite version and is validated again.
- **Repeat runs:** judge metrics on C1 and C2 slices run three times; criteria use the mean, and the spread is reported.
- **Load profile:** GuideLLM uses each tier's length distributions and request rates up to 1.5 × `peak_rps`, on the tier's target hardware.

### 4.7 Policy construction

The G5 policy has one file per archetype in the format of the technical implementation, with criteria per tier:

| Criterion | Rule in the policy | Hard |
| --- | --- | --- |
| Member floor | `value >= floor` | Yes |
| Critical-error ceiling | C1 and C2: `upper95 <= ceiling`, where the normalizer adds the one-sided upper 95% bound for error rates. C3 and C4: `value <= ceiling` | Yes |
| Language and long-context floors | `value >= floor` | Yes |
| Interface validity | `value >= 0.99` | Yes |
| Safety additions | Security's limit per test | Yes |
| Latency | GuideLLM p95 values at peak concurrency within the latency class | No: serving configuration can fix it, so it goes to review |
| Cost per request | Derived cost at or below the tier's target | No, for the same reason |
| Not worse than the incumbent | `ci95.upper >= incumbent.value` on the selection score | No |
| Better than the incumbent | `ci95.lower > incumbent.value` on the selection score | Reported for the Stage 8 replace decision |

Each criterion names its `tier` and the `requirement` it implements. The policy check rejects a policy that leaves any quality, latency, context, language or cost requirement of a tier without a criterion.

### 4.8 Aliases and naming

- **Floating aliases** (the default, for LLM archetypes): the alias follows the tier's selected model, and the platform replaces models behind it through shadow traffic, canary and cutover at Stage 11. Products do not change.
- **Pinned aliases** (embedding archetypes, and any archetype whose outputs products store and compare later): vectors from different models cannot be mixed in one index, so each model generation gets its own alias, such as `embeddings-v1`. A replacement model, including a quantized variant, gets `embeddings-v2`, and `embeddings-v1` is deprecated with notice so products can re-index before it retires. Members that share an index use the same generation, so real-time query embedding and batch indexing run on separate deployments of the same model, not as separate tiers.
- **Archetype id:** lowercase and hyphenated, naming the capability (`rag`, `text-to-sql`). Domain archetypes are `<parent>-<domain>` (`rag-legal`). No model, vendor, team or product names. Ids are never reused, including those of closed proposals.
- **Aliases:** `<id>-<tier>` (`rag-default`, `rag-large`); pinned archetypes use `<id>-v<n>`.
- **Suite, policy and slices:** suite `<id>/v<n>`, G5 policy `g5-<id>`, slices `<id>-<slice>`.

### 4.9 Example definition and policy

The definition produced by the worked example (section 9):

```yaml
# onboarding-config/archetypes/text-to-sql.yaml
id: text-to-sql
name: Text to SQL
kind: capability                           # capability | domain
parent: null                               # a domain archetype names its parent here
version: 1
status: Active                             # Proposed | Closed | Approved | Active | Deprecated | Retired
owner: <archetype owner>
signature: { task: code, output: query, grounding: supplied }
scope:
  summary: Write one read-only SQL query that answers a question over a supplied schema.
  in:
    - SELECT queries with joins, filters, aggregations and window functions
    - follow-up questions that refine the previous query
    - completing a partly written query
  out:
    - { task: running the query or acting on its results, covered_by: product code }
    - { task: general code generation, covered_by: code }
    - { task: answering from documents, covered_by: rag }
interface:
  model_type: llm
  api: chat-completions
  input: question, schema as DDL, up to five sample rows per table, earlier turns
  output: "JSON {sql, explanation}, valid against a published schema"
  features: [structured-output]
requirements:
  max_data_classification: confidential    # highest member value (K2)
  languages: [en, de]                      # union of members (K1)
  tiers:
    default:
      members: [analytics-assistant-sql, finance-reporting-sql]
      quality:
        - { id: q-exec-analytics, slice: analytics, metric: execution_accuracy, floor: 0.85 }
        - { id: q-exec-finance, slice: finance, metric: execution_accuracy, floor: 0.85 }
        - { id: q-exec-de, slice: finance-de, metric: execution_accuracy, floor: 0.85 }
        - { id: q-crit-analytics, slice: analytics, metric: critical_error_rate, ceiling: 0.03 }
        - { id: q-crit-finance, slice: finance, metric: critical_error_rate, ceiling: 0.02 }
        - { id: q-long-context, slice: long-context, metric: execution_accuracy, floor: 0.85 }
        - { id: q-valid, metric: valid_output_rate, floor: 0.99 }
      latency_class: L2                    # strictest of L2 and L3
      context_length: 32768                # 1.2 x (18,000 + 400) = 22,080, next size 32k
      peak_concurrency: 18                 # (80 / 60) x 10 s x 1.3
      cost_target_per_1k: 20               # lowest member target
    fast:
      members: [catalog-autocomplete-sql]
      quality:
        - { id: q-exec-catalog, slice: catalog, metric: execution_accuracy, floor: 0.70 }
        - { id: q-crit-catalog, slice: catalog, metric: critical_error_rate, ceiling: 0.05 }
        - { id: q-valid-fast, metric: valid_output_rate, floor: 0.99 }
      latency_class: L1
      context_length: 8192                 # 1.2 x (3,500 + 80) = 4,296, next size 8k
      peak_concurrency: 7                  # (300 / 60) x 1 s x 1.3
      cost_target_per_1k: 3
evaluation:
  suite: text-to-sql/v1
  policy: g5-text-to-sql@1
  judge_model: none                        # scored by running each query
risk:
  autonomy: { analytics-assistant-sql: auto, finance-reporting-sql: confirm, catalog-autocomplete-sql: suggest }
  trifecta: { private_data: all members, untrusted_content: 2 of 3 members, outbound: none at the model }
  stage10_focus: read-only database credentials scoped to the user's own access
  safety_additions: [destructive statements, instructions injected into schema comments and sample rows]
gateway:
  alias_mode: floating                     # floating | pinned
  aliases: { default: text-to-sql-default, fast: text-to-sql-fast }
lifecycle:
  review: quarterly                        # with the scheduled re-evaluation
  related: { neighbors: [code, rag] }
```

Each policy criterion names the requirement it implements:

```yaml
# onboarding-config/policies/g5-text-to-sql.yaml (excerpt)
policy: g5-text-to-sql
version: 1
suite: text-to-sql/v1
criteria:
  - id: exec-analytics
    tier: default
    requirement: q-exec-analytics
    metric: lighteval/text-to-sql-analytics/execution_accuracy
    rule: value >= 0.85
    hard: true
  - id: crit-finance
    tier: default
    requirement: q-crit-finance
    metric: lighteval/text-to-sql-finance/critical_error_rate
    rule: value <= 0.02                    # C3: point estimate; C1 and C2 use upper95
    hard: true
  - id: ttft-p95-default
    tier: default
    requirement: latency_class
    metric: guidellm/text-to-sql-default-profile/ttft_ms_p95
    rule: value <= 1500
  - id: not-worse-than-incumbent
    tier: default
    requirement: selection
    metric: policy-check/text-to-sql-default/selection_score
    rule: ci95.upper >= incumbent.value    # skipped while the tier has no GA model
```

## 5. Creation procedure

### 5.1 Process at a glance

```
 Trigger ──► 1 Proposal ──A1──► 2 Evidence ──A2──► 3 Definition ──A3──► 4 Build ──A4──► 5 First models
                 │                  │                           (board)                      │
                 ▼                  ▼                                                        ▼
              Closed:            Closed:                                            alias live at the
           smaller change     smaller change or                                     first model's Go Live
                              workflow redesign

 Status:     Proposed ────────────────────────────────────────────────► Approved ─────► Active
```

Steps 1 and 2 are cheap, and most proposals end there with a smaller change. Steps 3 and 4 run only when the decision table calls for a new archetype. A closed proposal records its reason and the path the sponsors take instead. Gates A1 to A4 belong to this framework; G1 to G8 remain the model onboarding gates.

### 5.2 Steps

| Step | Inputs | Work | Outputs | Gate | Target |
| --- | --- | --- | --- | --- | --- |
| 1 Proposal | Profiles of the proposing products and of the nearest archetype's members | Profile checks (2.3); G1, G2 proxy and G3; priority score (3.7) | Proposal record: profiles and grouping worksheet | A1 | 1 week |
| 2 Evidence | Seed sets, hosted models, cleared subsets | Optimized prompts; scoring; G2 with costs; G4 regret; G5 viability | Evidence report | A2 | 2 to 3 weeks |
| 3 Definition | Evidence report | Requirements (4.2 to 4.4); metrics (4.5); slice plan (4.6); risk profile; capacity and cost | Draft definition, measurement plan, risk profile, cost estimate | A3 | 2 weeks, including the board cadence |
| 4 Build | Approved definition | Full slices and labels; suite; judge validation; policy; data bundle; calibration (5.5) | Suite v1, policy v1, data lock, calibration report | A4 | 3 to 4 weeks |
| 5 First models | Active archetype | Catalog entry; extension onboardings or Stage 0 triggers; pilots | Catalog and routing guide entry; first GA model per tier | None | Per the process document |

The evidence report has six parts: (1) the profiles and their label agreement; (2) a score table of models by slice, with confidence intervals and the nearest archetype's suite scores; (3) the regret table per member, with intervals and decisions; (4) the tier cost test; (5) feasibility and cost viability; (6) the decision table row that applies, and the outcome.

### 5.3 Responsibilities

R: responsible, A: accountable, C: consulted, I: informed.

| Activity | Sponsors | Archetype owner | ML evaluation | Data governance | Security | Legal | SRE | Platform lead | Board |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Profiles and seed sets | R | A | C | C | I | I | I | I | I |
| G1 to G3 and A1 | C | A, R | C | I | I | I | C | I | I |
| Scoring, G4, G5 and A2 | C | C | A, R | C | I | I | R | I | I |
| Requirement derivation | C | A | R | I | I | I | R | I | I |
| Maximum data classification | C | C | I | A, R | C | I | I | I | I |
| Risk profile and safety additions | C | C | C | C | A, R | I | I | I | I |
| Dataset license review | I | I | C | I | C | A, R | I | I | I |
| Capacity and cost | C | C | C | I | I | I | A, R | C | I |
| A3 decision | C | R | C | C | C | C | C | C | A |
| Suite, policy, calibration and A4 | I | C | A, R | C | R | C | R | I | I |
| Catalog entry and first models | C | A, R | C | I | I | I | C | I | I |

### 5.4 Gate checklists

**A1, worth evaluating** (archetype owner):

- [ ] One profile per product use case, each passing the quality checks (2.3)
- [ ] Signatures classified and the G1 presumption recorded
- [ ] G2 proxy tiers and G3 constraint fit recorded
- [ ] At least two members, or the single-member exception applies (3.5)
- [ ] Nearest archetype named, with its current suite version
- [ ] Priority score recorded
- [ ] Outcome: proceed to Step 2, or closed with its path from the decision table

**A2, archetype needed** (ML evaluation team):

- [ ] Seed sets of 50 to 100 samples per member, with label agreement κ ≥ 0.7
- [ ] Every model scored with the optimized prompt, on the same samples
- [ ] Regret and its 90% interval for every member, against the nearest archetype and within each tier
- [ ] Inconclusive results grown to 200 samples or decided by the presumption, with the rule applied recorded
- [ ] Tier cost test computed with costs
- [ ] Feasibility, cost viability and the prompt check (G5) passed
- [ ] The applying decision table row and outcome recorded

**A3, approved for the catalog** (model review board):

- [ ] Draft definition with every field filled and every value traced to its rule (4.2 to 4.4)
- [ ] Tiers justified by the tier cost test
- [ ] Slice plan with sizes from tables A and B
- [ ] Maximum data classification set by data governance
- [ ] Risk profile and safety additions from Security
- [ ] Dataset license review from Legal
- [ ] Capacity, cost per request and cost viability from SRE
- [ ] First-model plan: hosted models to extend, or Stage 0 triggers
- [ ] Decision: approve (with conditions if needed), reject or send back

**A4, ready for onboarding** (ML evaluation team, Security, SRE):

- [ ] Every slice at its planned size, labeled, classified and in `eval-data`
- [ ] Judge metrics validated (4.6)
- [ ] Data bundle and lock merged; a run completed with egress denied
- [ ] Calibration (5.5) passed
- [ ] Policy merged, and the policy check finds a criterion for every requirement
- [ ] Safety additions added to the safety battery
- [ ] Load profile per tier merged

### 5.5 Calibration

Before A4, P7 runs suite version 1 twice on all hosted models with a compatible interface. If fewer than three fit, the discrimination check also uses the Step 2 reference and candidate results on cleared samples.

| Check | Passes when |
| --- | --- |
| Runs offline | Every task completes with egress denied |
| Discriminates | On each member slice, the best and worst model differ by at least twice the slice's confidence interval half-width |
| Not saturated | On no member slice are three or more models within the half-width of a score above 0.95 |
| Agrees with the seed sets | The model the suite selects in each tier is the one Step 2 selected, or its regret is within tolerance on every member slice |
| Stable | The two runs' selection scores differ by less than the half-width |
| Sealed part | Open and sealed parts of each slice differ by less than the half-width |

### 5.6 First models

- **Publish:** the catalog and routing guide list the archetype with its definition, marked as having no model yet.
- **Hosted models first:** for each hosted model that calibration puts within the confidence interval of every floor in a tier, the archetype owner requests an extension onboarding.
- **Then candidates:** for each tier with no such model, the archetype owner logs a Stage 0 trigger for the candidates found in G5.
- **No incumbent:** the first model in a tier is judged against the tier's criteria alone.
- **Pilots:** where a tier has two or more members, two of them pilot at Stage 6, as the process document suggests when an archetype serves several products.
- **Aliases:** each tier's alias is created in `gateway-aliases` with its first model, in dev at Stage 9 and in production at Stage 11.

The process document makes extending an approved model to a new archetype a new onboarding. Because the model digest is unchanged, this framework proposes that the model's Stage 3 and Stage 4 evidence carries over, Legal confirms that the license allows the new uses, and the onboarding runs Stage 1 and Stages 5 to 11 for the new archetype (see Open questions).

## 6. Lifecycle

### 6.1 Statuses

| Status | Meaning | Models can target it | Products can use it |
| --- | --- | --- | --- |
| Proposed | In Steps 1 and 2 | No | No |
| Closed | Not created; the reason and the path taken instead are recorded | No | No |
| Approved | Approved by the board; suite being built | No | No |
| Active | Suite and policy in use | Yes | Through its aliases, once a model behind them reaches Dev (in dev) or GA (in production) |
| Deprecated | Successor and sunset date announced | No | Existing members only, while they migrate |
| Retired | Aliases removed; definition, suite and evidence archived | No | No |

### 6.2 Catalog review metrics

At each scheduled re-evaluation, P7 computes these for every Active archetype, and the archetype owner and the ML evaluation team act on any threshold crossed:

| Indicator | Computed from | Threshold | Action |
| --- | --- | --- | --- |
| Members | Stage 10 approvals | Fewer than two at two reviews in a row | Merge or deprecation proposal |
| Volume | Gateway metering | Below 25% of the projection at two reviews | Re-run cost viability; consider merging tiers |
| Member regret | P7 scores per member slice | The GA model's regret above tolerance for a member at two reviews | Tier or split proposal for that member |
| Merge signal | P7 scores of two archetypes | Same selected model, and the regret test passes both ways, at two reviews | Merge proposal |
| Saturation | P7 scores | Three or more models within the half-width of a score above 0.95 on half the slices | Harder slices in a new suite version |
| Drift | A monthly sample of 200 requests per member, taken under data governance rules and sorted by the product owner into the slice's task types | Input length p95 moved by more than 30%, or a task type the slice lacks above 20% of the sample | Refresh that member's slice |
| Cost | Metering and serving cost | Above 120% of the tier's target for a month | Serving or tier review |
| Latency | Gateway dashboards | p95 outside the latency class on more than 5% of days | Serving review |
| Stage 10 friction | Stage 10 records | Two or more requests in a quarter that fit only partly, for the same reason | Fit check for a tier or a new archetype |

### 6.3 Changes and versions

Member profiles are reviewed at each catalog review. When a profile changes, the derivation rules (4.2 to 4.4) run again, and any value that changes is a definition change.

| Change | Examples | Approved by | Effect |
| --- | --- | --- | --- |
| New suite version | New or refreshed slices, harder tasks, a new tool or judge version | ML evaluation team, through P0 | Scores compare only within a version; models on the archetype are re-scored at the next re-evaluation |
| New policy version | A threshold changes | ML evaluation team, by reviewed pull request | Verdicts name the policy version they used |
| Definition change | Re-derived values, a new member, language or tier, a wider scope | Archetype owner with the ML evaluation team; platform lead for a new tier; board if the maximum data classification rises | Models are re-scored. A GA model that falls below a raised bar keeps serving and triggers Stage 0 for a replacement |
| Breaking change | A narrower scope, a different interface, a lower maximum data classification | Never made in place: a split or a successor archetype goes through this framework, with notice to members | Members move through Stage 10 |

Urgent restrictions after a vulnerability or a data risk finding follow the emergency path of the model onboarding process, not this table.

### 6.4 Split, merge and retire

The board approves each of these on the archetype owner's proposal.

- **Split** when the member regret indicator fires for the same members at two reviews, or the tier cost test fails for a tier. The existing slices are the evidence, so Step 2 needs no new samples; the new archetypes or tiers go through Steps 3 and 4, and members move through Stage 10.
- **Merge** when the merge signal fires. The surviving suite takes the other archetype's slices in a new version, and members of the retiring archetype move to the survivor's aliases through Stage 10 during the notice period.
- **Deprecate** when the members indicator fires, when a split or merge replaces the archetype, or when its need has gone. The notice period, and the successor alias named in the notice, follow the model deprecation rules.
- **Retire** at the sunset date: aliases are removed from `gateway-aliases`, the definition is marked **Retired**, and the suite, policy, data and evidence are kept for audit. A model approved only for a retired archetype is deprecated with it.

## 7. Implementation on the platform

Archetypes use the same repositories and pipelines as model onboarding:

| Item | Where it lives | Written by |
| --- | --- | --- |
| Product profiles | `onboarding-config/profiles/<product>-<use-case>.yaml` | Product owner, by reviewed pull request; reviewed by the archetype owner |
| Archetype definition | `onboarding-config/archetypes/<id>.yaml` | Archetype owner, by reviewed pull request; a status change cites its gate record |
| Suite definition and reference prompts | `onboarding-config/suites/<id>/v<n>/` | ML evaluation team, by reviewed pull request |
| G5 policy | `onboarding-config/policies/g5-<id>.yaml` | ML evaluation team, by reviewed pull request |
| Data lock and bundle | `onboarding-config/data/<id>/v<n>.lock.yaml` and `eval-data/bundles/<id>/v<n>/` | P0 |
| Seed sets and slices | `eval-data/domain/<id>-<slice>/v<n>/`, with seed sets under `seed/` | P5, with the classification set by data governance |
| Gate records and evidence reports | `evidence/archetypes/<id>/<step>/<run-id>/` | P5 |
| Archetypes a model is approved for | `archetypes` in each `model-registry` record | Pipeline service account |
| Aliases | `gateway-aliases` | P6 |

The selection score, regret with bootstrap intervals, the tier cost test and the derivation formulas run in the policy-check code over normalized results (`results.jsonl`). The bootstrap seed and the GPU-hour rate are recorded in the run manifest, so an A2 decision can be reproduced like a G5 verdict.

P5 takes archetype records the same way as model records, with its manual approval step assigned to the gate owner: the archetype owner for A1, the ML evaluation team for A2, the board chair for A3, and the ML evaluation team, Security and SRE for A4.

The pipelines enforce the definition:

- **P1** accepts only target archetypes whose definition is **Active**, and screens each candidate against the target tier's context length, languages and interface features.
- **P4** runs each target archetype's current suite and policy, tier by tier. While a tier has no GA model, criteria against an incumbent are skipped and the verdict says so.
- **P6** refuses an alias mapping when the model's registry record does not list the archetype, or its maximum data classification is below the archetype's. It also refuses to change the model behind an existing pinned alias.
- **P7** re-scores every model on an archetype with the current suite version at each scheduled re-evaluation, computes the catalog review metrics (6.2), and runs calibration on demand.
- **Policy check** rejects a policy that leaves a requirement without a criterion (4.7).

## 8. Starting catalog

Proposed signatures and starting values for the archetypes listed in the process document. The first product profiles replace these values through the derivation rules.

| Archetype | Id | Signature | Latency class | Context band | Primary metric | Alias mode | Likely tiers |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Q&A / RAG | `rag` | `answer/prose/supplied` | L2 | X2 | Answer correctness, with faithfulness | Floating | default, large |
| Summarization | `summarization` | `condense/prose/supplied` | L3 | X3 | Key-point coverage, with faithfulness | Floating | default |
| Extraction and classification | `extraction` | `extract/json/supplied` | L4 | X2 | Field-level F1 | Floating | default, fast |
| Code assistance | `code` | `code/code/none` | L2 | X3 | pass@1 | Floating | default, fast (completion) |
| Agents and tool use | `agents` | `act/tool-calls/supplied` | L3 | X3 | Task success | Floating | default |
| Embeddings and search | `embeddings` | `embed/vector/none` | L1 for queries, L4 for indexing | X1 | nDCG@10 | Pinned | One generation at a time |
| Multilingual | `multilingual` | `translate/prose/supplied` | L3 | X2 | COMET or chrF++ per language pair | Floating | default |

No starting archetype covers `compose` (drafting and rewriting), `answer/prose/parametric` (closed-book Q&A) or `rank/score/none` (reranking). Products with these signatures go through the creation procedure. The starting archetypes go through the same steps, with two differences: Step 2 has no hosted models to compare, so it checks seed sets, feasibility and cost viability only; and the board can approve the whole starting catalog in one session.

## 9. Worked example: Text to SQL

The products and numbers in this example are illustrative.

### 9.1 Profiles

| Field | A: analytics assistant | B: finance reporting | C: catalog autocomplete |
| --- | --- | --- | --- |
| Use case | Answer business questions with SQL over the sales warehouse | Write the queries behind monthly finance report sections | Complete SQL in the data catalog's query editor |
| Status | Live | Roadmap, Q1 2027 | Live |
| Signature | `code/query/supplied` | `code/query/supplied` | `code/query/supplied` |
| Latency class | L2 | L3 | L1 |
| Criticality | C3: analysts check results | C3: finance staff check figures; critical errors at most 2% | C4 |
| Input p95, output p95 (tokens) | 14,000, 300 (measured) | 18,000, 400 (estimated) | 3,500, 80 (measured) |
| Requests per day, peak per minute | 20,000, 60 | 5,000, 20 | 60,000, 300 |
| Cost target per 1,000 requests | $20 | $30 | $3 |
| Languages | en | en, de | en |
| Data classification | Confidential | Confidential | Internal |
| Domain, turns | General, multi | Finance, single | General, single |
| Autonomy | `auto` (runs read-only queries) | `confirm` | `suggest` |
| Floor (execution accuracy) | 0.85 | 0.85 | 0.70 |
| Label agreement κ | 0.82 | 0.76 | 0.88 |

### 9.2 Step 1: proposal (A1)

- **G1:** all three share `code/query/supplied`. No archetype has this signature, and the nearest is `code` (`code/code/none`, same task). Presumption: new capability archetype.
- **G2 proxy:** A and B have adjacent latency classes (L2, L3), the same criticality (C3), adjacent context bands (X2, X3) and cost targets within a factor of 2 ($20, $30): one tier. C's cost target is 6.7 times lower than A's: a separate tier.
- **G3:** languages en and de; maximum classification confidential; a finance slice from B; a multi-turn slice from A.
- **Priority score:** 50 (members 3, volume V3 3, criticality C3 3, gap 3, feasibility 1, build effort 1, using gap diagnosis estimates). A1 passes.

### 9.3 Step 2: evidence (A2)

Scores with optimized prompts on the 80-sample seed sets (execution accuracy; the `code` column is that archetype's selection score):

| Model | `code` suite | Slice A | Slice B | Slice C |
| --- | --- | --- | --- | --- |
| M1 (behind `code-default`) | 0.78 | 0.55 | 0.50 | 0.70 |
| M2 (behind `rag-large`) | 0.71 | 0.71 | 0.66 | 0.74 |
| M3 (behind `agents-default`) | 0.64 | 0.69 | 0.67 | 0.61 |

Regret against `code`, whose suite selects M1:

| Member | Best | Regret | 90% interval | Result |
| --- | --- | --- | --- | --- |
| A | M2 | 0.16 | 0.08 to 0.24 | Fail |
| B | M3 | 0.17 | 0.09 to 0.26 | Fail |
| C | M2 | 0.04 | −0.03 to 0.11 | Inconclusive. Grown to 200 samples: 0.04, −0.01 to 0.09, still inconclusive, so the presumption decides: C does not join `code` |

Tier cost test with costs, at an illustrative internal rate of $2.50 per GPU-hour:

- Candidate K (one GPU per replica, 1.0 requests per second sustained at L2) costs 2.50 / (3,600 × 1.0) = $0.69 per 1,000, so `c_A = c_B = $0.69`.
- Candidate S (one GPU, 12 requests per second at L1) costs 2.50 / (3,600 × 12) = $0.06 per 1,000, so `c_C = $0.06`.
- One model for all three would be K at L1, which needs four GPUs per replica and holds L1 only up to 0.3 requests per second: `c_all` = 2.50 × 4 / (3,600 × 0.3) = $9.26 per 1,000, about 160 times `c_C`. C moves to a fast tier, which it may have alone because it carries 71% of the volume. Within the default tier, `c_all = c_A = c_B`, so the test passes.

Regret within each tier: in the default tier, the selection score (mean of slices A and B) selects M2 (0.685) over M3 (0.680) and M1 (0.525). Regret is 0 for A and 0.01 for B, within tolerance. The fast tier has one member. Both tiers pass.

Viability:

- **Members:** three; A and C are live, and B is planned within two quarters.
- **Feasibility:** no hosted model reaches the default tier's floors. On a cleared subset of 100 questions over a synthetic schema that mirrors A and B, candidate K scores 0.86 (lower 90% bound 0.803, just above the floor minus 0.05) and a reference model 0.90. For the fast tier, M2 meets C's floor (0.74) but not L1; candidate S scores 0.73 on 100 cleared completions (lower bound 0.66).
- **Cost viability, default tier:** `peak_rps` = 80 / 60 = 1.33, so replicas = max(ceil(1.33 × 1.3 / 1.0), 2) = 2. The monthly cost is 2 × 1 × $2.50 × 730 = $3,650, over 30 × 25,000 = 750,000 requests: $4.87 per 1,000, within $20.
- **Cost viability, fast tier:** replicas = max(ceil(5 × 1.3 / 12), 2) = 2, so $3,650 a month over 1,800,000 requests: $2.03 per 1,000, within $3.
- **Prompt check:** every score above used optimized prompts.

Outcome, from the decision table's capability archetype row: `text-to-sql`, with a default tier for A and B and a fast tier for C. A2 passes, and the priority score with measured values stays at 50.

### 9.4 Step 3: definition (A3)

| Requirement | Rule | Default tier (A, B) | Fast tier (C) |
| --- | --- | --- | --- |
| Member floors | 4.2 | A 0.85, B 0.85 | C 0.70 |
| Critical-error ceilings | Stricter of `max_rate` and the class default | A 3% (C3), B 2% (stated) | C 5% (C4) |
| Language floor | Strictest floor of the members using it | de 0.85 | n/a |
| Latency class | Strictest member class | L2 | L1 |
| Peak concurrency | `peak_rps × request_p95 × 1.3` | 1.33 × 10 s × 1.3 = 18 | 5 × 1 s × 1.3 = 7 |
| Context length | 1.2 × max(input + output), next size | 1.2 × 18,400 = 22,080, so 32k | 1.2 × 3,580 = 4,296, so 8k |
| Cost target | Lowest member target | $20 per 1,000 | $3 per 1,000 |
| Data classification | Highest member value, for the whole archetype | Confidential | Confidential |

Slice plan:

| Slice | Size | Rule |
| --- | --- | --- |
| `text-to-sql-analytics` (A) | 200 single-turn questions and 100 multi-turn conversations | Table A at 0.85, ±0.05: 196; multi-turn slice: 100 |
| `text-to-sql-finance` (B) | 200 | Table A at 0.85, ±0.05: 196 |
| `text-to-sql-finance-de` (B, German) | 100 | Language slice |
| `text-to-sql-catalog` (C) | 170 | Table A at 0.70, ±0.07: 165 (C4) |
| `text-to-sql-long-context` | 100 at 26,000 to 32,000 tokens | The default tier's 32k context is in band X3 |
| Spider and BIRD development sets | The benchmarks' sets | 20% of the default tier's selection score, after Legal's review |
| Safety additions | 50 requests for destructive statements, 50 items with injected instructions | Set by Security |

The risk profile records private data for every member, untrusted content for A and B (row values can hold text that customers typed), and no outbound path at the model; products run the queries. The board approves the archetype on condition that products run queries with read-only credentials scoped to the user's own access, checked at Stage 10.

### 9.5 Step 4: build (A4)

The suite `text-to-sql/v1` scores execution accuracy with a custom LightEval task that runs each query against a sandboxed PostgreSQL image built and pinned by P0, loaded with synthetic data for each member's schema. No judge model is needed. Calibration on M1 to M3:

| Check | Result | Pass |
| --- | --- | --- |
| Runs offline | Every task completed with egress denied | Yes |
| Discriminates | Default tier selection scores M2 0.70, M3 0.67, M1 0.55: a spread of 0.15, against twice the half-width, 0.10 | Yes |
| Not saturated | Best score 0.74 | Yes |
| Agrees with the seed sets | The suite selects M2, as Step 2 did | Yes |
| Stable | The two runs differ by 0.004 | Yes |
| Sealed part | Largest open-to-sealed difference 0.02, below the half-width of 0.05 | Yes |

A4 passes, and the archetype becomes **Active**.

### 9.6 Step 5: first models

No hosted model reaches the default tier's floors, so the archetype owner logs Stage 0 triggers for K (default tier) and S (fast tier). At G5, with no incumbent, K scores 0.87 on slice A, 0.86 on B, 0.85 in German and 0.86 on long context, with critical-error rates of 1.8% (A) and 1.5% (B), 99.6% valid outputs, and a TTFT p95 of 1.1 s at 18 concurrent requests: every criterion passes. A and B pilot it at Stage 6, and `text-to-sql-default` goes live with A's Go Live. S follows the same path for `text-to-sql-fast`.

## Open questions

- [ ] Is the archetype owner always a platform model owner, and can one person own several archetypes?
- [ ] Who approves changes to the suggested values (the tier cost factor of 1.5, the regret tolerance of 0.03, the κ thresholds, the latency classes): the ML evaluation team alone, or the board?
- [ ] Which GPU-hour rate do the cost formulas use, and who maintains it?
- [ ] What is the platform's data classification scheme, and which levels can an archetype hold?
- [ ] Is two products the right minimum, and two quarters the right roadmap horizon?
- [ ] May an extension onboarding reuse a model's Stage 3 and Stage 4 evidence for the same digest? If so, add it to the process document's timelines and fast track table.
- [ ] What notice period applies when an archetype is split, merged or retired?
- [ ] Where are the catalog and routing guide published, and who keeps them current?
