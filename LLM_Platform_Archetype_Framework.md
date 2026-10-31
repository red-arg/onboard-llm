# LLM Platform: Archetype Framework

Version 0.3 · Oct 31, 2026 · Owner: @Zhi Feng · Next review: Jan 31, 2027

## Purpose

The [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) selects models for archetypes, not for individual products. An archetype groups use cases that need the same task and output from a model, so one model, one eval suite and one gateway alias can serve them all.

This document tells archetype owners, product teams and the ML evaluation team when to create, change or retire an archetype, and what to do at each step. Repositories and pipelines are described in the [technical implementation](LLM_Platform_Model_Onboarding_Technical_Implementation.md).

## When to do what

| Situation | Do this | See |
| --- | --- | --- |
| A product needs a model for a use case | Write a profile with labeled samples, then run the fit check | Steps 1 and 2 |
| The use case matches an existing archetype | Join it through Stage 10 | Step 2 |
| It matches, but fails its Stage 10 domain test | Run gap diagnosis | Step 2 |
| Two or more products need a task or output no archetype covers | Test models, then create an archetype | Steps 3 to 6 |
| Only one product needs it | Use the nearest archetype with the best prompt, and revisit when a second product needs it | Step 2 |
| A product needs another language or a higher data classification | Extend the archetype | Step 2, Changes |
| A product's latency, volume, oversight or data changes | Update the profile, and re-derive the requirements and tier | Changes |
| An archetype's scope or output must change | Create a new archetype and retire the old one | Changes |
| The quarterly re-evaluation is due | Run the quarterly review | Quarterly review |
| A monitoring signal fires | Follow the signal table | Signals |
| An archetype has had fewer than two products for two quarters | Merge or retire it | Changes |

## Rules

1. **Group by task and output.** Use cases with the same task and output share an archetype, unless the Step 3 test shows that one model cannot serve them all.
2. **Two products minimum.** Create an archetype only when at least two products need it, live or planned within two quarters, each with a named owner.
3. **Smallest change first.** Join, then extend, then onboard a stronger model. Create a new archetype last.
4. **Each product keeps its own bar.** Every product's floor is checked on its own samples, so an average never hides one product's failure.
5. **Decide with scores** on the products' own samples.
6. **Checks scale with the tier.** Higher-risk archetypes get more testing, approval and monitoring.
7. **No breaking changes in place.** Never narrow an archetype's scope or change its output; create a new archetype. Never reuse an id.

## Roles

| Role | Responsibilities |
| --- | --- |
| Product team | Writes profiles and labels samples; meets the archetype's conditions; reports incidents; labels live outputs at tiers 2 and 3 |
| Archetype owner (a platform model owner) | Runs the fit check; writes the definition; proposes pilots; runs the quarterly review; acts on signals |
| ML evaluation team | Runs the Step 3 test; builds and re-runs the eval; second-labels samples at tiers 2 and 3 |
| Data governance | Sets classifications; clears samples for outside models; spot-checks slices |
| Model review board | Approves new archetypes, variants, major changes and retirements; decides tier 3 suitability and residual risk |
| Platform lead | Accepts tier 2 residual risk |
| Security | Reviews safety tests; runs red-team rounds with the ML evaluation team |
| SRE | Credentials, metering, alerts and fallback drills |

The framework owner keeps this document current. Whoever proposes an archetype or a candidate model does not decide its Step 3 test or approve it at Step 4.

## Tiers

Work out each use case's tier from its profile by counting the dimensions that are high:

| Dimension | High when |
| --- | --- |
| Effect | The output is run: `code`, `query` or `tool-calls` |
| Oversight | Outputs reach customers or the public, or drive decisions or actions, with no person checking each one |
| Data | The classification is confidential or higher, or the data is regulated |

None high is tier 1, one is tier 2, and two or three is tier 3. An archetype's tier is the highest among its products.

| Requirement | Tier 1 | Tier 2 | Tier 3 |
| --- | --- | --- | --- |
| Label check (Step 1) | 27 of 30 samples agree | 27 of 30; second labeler from outside the product team | 45 of 50; second labeler from outside the team |
| Test samples per product (Step 3) | 50 or more | 100 | 100, plus injection samples if content is untrusted |
| Suitability decided by (Step 3) | Archetype owner | Archetype owner and ML evaluation team | Board, at Step 4 |
| Conditions on products (Step 4) | None | For each high dimension | For each high dimension |
| Residual risk accepted by (Step 4) | Archetype owner; review yearly | Platform lead; review yearly | Board chair with the board's risk member; review every 6 months |
| Product slice before the first Stage 5 (Step 5) | 200 samples | 200 samples | 300 samples; floor checked on the lower end of the 95% Wilson interval |
| Judge metrics (Step 5) | One pass | Mean of three passes | Mean of three passes |
| Injection tests (Step 5) | The archetype's tests | Plus a held-out set, and a yearly red-team round if content is untrusted | Plus a held-out set, and a red-team round before going active and every quarter |
| Going active and moderate changes approved by | Archetype owner | Archetype owner and ML evaluation team | Board |
| Usage checked against profiles | Quarterly | Monthly | Monthly |
| Labeled live outputs per product | None | 30 a quarter | 50 a quarter |
| Fallback | Recorded | Recorded | Recorded; drilled before going active and yearly |

**Conditions for high dimensions.** Every product in the archetype meets these, checked at Stage 10:

| High dimension | Condition |
| --- | --- |
| Effect | Run outputs with the end user's own permissions, read-only where the task allows. A person confirms any destructive or outbound action |
| Oversight | Tell people at the point of use that the output is AI-generated and may be wrong. Route uncertain cases to a person. Keep critical errors at or below 1% |
| Data | Send nothing above the archetype's classification. Use masked or synthetic samples for outside reference runs |

**Untrusted content** (customer messages, emails, uploads, outside documents) does not change the tier. If any product sends it, the eval tests injection through it.

## Getting started

1. Name an owner for each starting archetype below. One platform model owner can own several.
2. Ask each product team on the roadmap for one profile per use case (Step 1).
3. Sort the profiles into the table by task and output, and work out their tiers. Postpone any archetype with fewer than two products.
4. For the two or three archetypes with the most products, write the definition (Step 4) and build the eval (Step 5). The board approves them together. They skip the Step 3 test, because no models are hosted yet; record that, and treat their likelihood as medium.
5. Onboard each archetype's first model through the model onboarding process, with two of its products as pilots (Step 6).
6. Handle every later need from the When to do what table, and review this document's thresholds after the first few archetypes.

| Archetype | Id | Task / output | Lowest tier | Primary metric | Alias |
| --- | --- | --- | --- | --- | --- |
| Q&A / RAG | `rag` | `answer` / `prose` | 1 | Answer correctness, faithfulness | `rag-default` |
| Summarization | `summarization` | `condense` / `prose` | 1 | Key-point coverage, faithfulness | `summarization-default` |
| Extraction and classification | `extraction` | `extract` / `json` | 1 | Field-level F1 | `extraction-default` |
| Code assistance | `code` | `code` / `code` | 2 | pass@1 | `code-default` |
| Agents and tool use | `agents` | `act` / `tool-calls` | 2 | Task success | `agents-default` |
| Embeddings and search | `embeddings` | `embed` / `vector` | 1 | nDCG@10 | `embeddings-v1` |
| Multilingual | `multilingual` | `translate` / `prose` | 1 | chrF++ or COMET | `multilingual-default` |

Embedding aliases carry a version. Vectors from different models cannot share an index, so a new embedding model gets a new alias (`embeddings-v2`), and products re-index before the old alias retires.

## Creating an archetype

### When to create one

Create a new archetype only when all of these hold:

1. At least two products need it, live or planned within two quarters, each with a named owner.
2. No archetype covers its task and output, or the Step 3 test shows that the nearest archetype's eval would pick the wrong model for these products.
3. At least one model, hosted or an open-weight candidate, meets every product's floor.
4. Suitability allows it (Step 3).

Otherwise, join or extend an existing archetype, or onboard a stronger model for it.

### Steps

| Step | Who | Done when | Target |
| --- | --- | --- | --- |
| 1. Profile | Product teams | Profiles complete | 1 week |
| 2. Fit check | Archetype owner | Outcome recorded | 2 days |
| 3. Test | ML evaluation team | Decision and suitability recorded | 2 weeks |
| 4. Define and approve | Archetype owner; board approves | Definition approved; residual risk accepted | Next board meeting |
| 5. Build the eval | ML evaluation team | Readiness check passed; Stage 0 opened for the first model | 2 to 3 weeks |
| 6. Pilot and go active | Archetype owner | Go-active criteria met | With the first model's onboarding |

The archetype's status is `proposed` once Step 3 decides to create it, `pilot` after Step 5 and `active` after Step 6. Record each step's decision (template below) in `evidence/archetypes/<id>/`.

### Step 1. Profile

The product team fills in one profile per use case (template below), with 50 to 100 labeled samples from its real or planned work. The same profile describes the use case at Stage 10.

- **One task per profile.** A use case with two tasks, such as answering from documents and drafting emails, is two profiles and calls two aliases. Answers from the model's own knowledge, rather than from supplied sources, are not `answer`, and no starting archetype covers them.
- **Classification** is set by data governance.
- **Tier** comes from the output and risk fields (Tiers), never chosen.
- **Critical errors:** set `max_critical_errors` to 1% when oversight is high, and 3% otherwise.
- **Checked outputs:** if the tier is low only because a person checks every output, the product reports its reviewers' change rate each quarter and runs a seeded-error test each year (20 known-wrong outputs, at least 18 caught). Otherwise, it counts as unchecked.
- **Complete** when every field is filled, every sample has an expected output, and the label check passes (Tiers). If labelers agree less often, fix the criteria first, as in gap diagnosis Step 1.

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

### Step 2. Fit check

The archetype owner compares the profile with the existing archetypes.

Refuse at this step:

- a model that would hold tools or network access itself, because tools live in products;
- a use case with all three legs of the lethal trifecta, until the product removes one;
- automated decisions with legal or similar effects on individuals, until risk and legal set conditions.

| The use case | Outcome |
| --- | --- |
| Matches an archetype's task and output, and fits its latency, context length, languages and data classification | Join through Stage 10, and add the samples to its eval as a new slice. If the product's tier is higher than the archetype's, meet the new tier's requirements first (Changes) |
| Matches, but needs another language or a higher data classification | Extend the archetype (Changes), then re-test its models |
| Matches, but failed its Stage 10 domain test | Run gap diagnosis. If it finds a catalog gap that two or more products share, go to Step 3 |
| Has a task or output no archetype covers, and at least two products need it | Go to Step 3 |
| Has a task or output no archetype covers, and only one product needs it | Not now: use the nearest archetype with the best prompt. The board may accept a single product when no archetype can produce its output at all |

### Step 3. Test

On each product's samples, the ML evaluation team scores:

- the model behind the nearest archetype's alias, and the other hosted models that can produce the output;
- promising open-weight candidates, on samples that data governance clears for outside use.

It also runs the nearest archetype's own eval on the same hosted models. Use an improved prompt for every model (gap diagnosis Step 2), so that a prompt problem does not look like a model gap. On 100 samples, a score varies by about 7 points by chance: treat differences of 5 points or less as ties, and add samples before acting on a difference under 7. Keep every run.

The first row that matches decides:

| Result | Decision |
| --- | --- |
| The model behind the nearest archetype's alias meets every product's floor | Join that archetype |
| No model meets the floors, hosted or candidate | Not now: redesign the workflow (gap diagnosis Step 5), and re-test at the next quarterly re-evaluation |
| The model the nearest archetype's eval ranks first is within 5 points of the best hosted model on the samples | Onboard a stronger model for that archetype through Stage 0 |
| Otherwise | Create an archetype: the existing eval would pick the wrong model for these products |

**Suitability.** Before creating an archetype or onboarding a stronger model, rate the likelihood of failure from the best available model:

- **Low:** it clears every floor by 5 points or more, and its critical-error rate is under half the maximum.
- **High:** it misses a floor or the critical-error maximum.
- **Medium:** anything else.

| Tier | Low | Medium | High |
| --- | --- | --- | --- |
| 1 | Proceed | Proceed | Not now |
| 2 | Proceed | Proceed with extra conditions | Not now |
| 3 | Proceed with extra conditions | The board decides, weighing the products served, their volume and the cost of the alternative | Not now |

### Step 4. Define and approve

The archetype owner writes the definition (template below), deriving each requirement from the member products' profiles:

| Requirement | Rule |
| --- | --- |
| Quality | Each product keeps its own floor and maximum critical-error rate, checked on its own slice |
| Latency | The strictest product's latency class |
| Context length | 1.2 × the largest (input p95 + output p95), rounded up to 8k, 16k, 32k, 64k or 128k |
| Languages | Every language any product needs, each with its own slice |
| Data classification | The highest any product sends. Every model behind the alias must be approved for it |
| Cost target | The lowest product target, where products gave one |
| Tier | The highest product tier |

The definition also records:

- **Conditions:** those for each high dimension (Tiers), plus any the archetype needs, such as read-only database credentials.
- **Blast radius and fallback:** the products, their volume, any other alias the same model serves, and what products do when the alias is down.
- **Risks:** at tiers 2 and 3, at least one archetype-specific risk per high dimension, each with a control, a test and an owner. At tier 3, at least one of them is an attack.
- **Residual risk:** what is left after controls, why it is acceptable, and a review date, accepted by the owner for the tier (Tiers). Past the review date, the archetype takes no new products until the risk is accepted again.

Data governance sets the classification. The board approves, with conditions if needed, and consults Security at tier 3.

If one product needs a much faster or cheaper model than the rest, note it in the definition. A variant alias, such as `<id>-fast`, comes when at least two products need the same trade-off; the board approves it.

### Step 5. Build the eval

The ML evaluation team builds the archetype's eval suite:

- **One slice per product:** 100 samples to start, grown before the first model's Stage 5 benchmark (Tiers). **One slice per extra language:** 50 to 100 samples.
- **One reference prompt,** starting from the improved prompt used in Step 3.
- **Safety:** the standard safety battery, plus archetype tests, such as instructions injected into retrieved passages (`rag`), source documents (`summarization`) or tool results (`agents`), or destructive statements (`text-to-sql`). Test injection through every kind of untrusted content a product sends, and keep the held-out injection set away from product teams.
- **Public benchmarks:** optional, worth at most 20% of the ranking.
- **G5 policy:** each product's floor and maximum critical-error rate are hard criteria; at tier 3, the floor rule is `ci95.lower >= <floor>`. Latency and cost go to review. Rank the models that pass by the average of the product slices, with each product weighted equally.
- **Judge:** pinned, internally hosted and never the candidate. It must agree with human labels on at least 85% of each slice's calibration samples (30, 50 or 100, by tier).

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

Run unit tests and sandbox databases inside the eval job, loaded from the data bundle, so that the eval stays offline.

**Readiness check,** before setting status to `pilot`:

- the suite runs end to end offline;
- its scores differ between models;
- a deliberately weak model fails the floors;
- the model it ranks first is the one Step 3 found best, or within 5 points of it.

Then open Stage 0 for the archetype's first model.

### Step 6. Pilot and go active

Propose two products to the first model's owner as pilots. They take the model through Stages 6 to 11.

The archetype goes active when:

- the first model is GA;
- both pilots are through early-life watch;
- each pilot's labeled live results are consistent with its slice, meaning the slice score is inside the live score's 95% interval;
- no severity 1 or 2 signal is open;
- the routing guide entry is published, with the archetype's scope, out of scope, floors, tier, conditions and known weaknesses;
- at tier 3, a red-team round and the products' fallback drills are done.

The approver depends on the tier (Tiers). Once the status is `active`, any product can join through Stage 10.

## Running archetypes

### Quarterly review

At each quarterly re-evaluation, the archetype owner:

1. Re-runs the eval on every model behind the alias (P7), with incident and gap diagnosis cases added to the regression slice.
2. Re-derives the requirements and tiers for changed profiles, and asks owners to reconfirm profiles older than 12 months. Credentials for unconfirmed profiles lapse after notice.
3. Runs Step 3 for any product whose best model has scored more than 5 points above the archetype's model at two re-evaluations in a row.
4. Merges or retires any archetype that has had fewer than two products for two quarters.
5. Checks residual-risk review dates, the owner and the routing guide entry, and that every pilot or active archetype has its step records with the right approvers.
6. Reviews decisions that were later reversed or that overrode a rule, and their reasons.

Once a year, the framework owner sends a seeded proposal, such as a profile with too low a tier, through the steps to check that it is caught.

### Signals

Severity 1: act the same day. Severity 2: within two weeks. Severity 3: at the next quarterly review.

| Signal | Severity | Response | Owner |
| --- | --- | --- | --- |
| Critical-error incident: a destructive or unrequested action, or data shown to the wrong person | 1 | The product switches to its fallback or disables the feature; kill switch or rollback if the model is at fault; review within 5 working days; cases join the regression slice | Product owner, SRE, archetype owner |
| A product sends data above the archetype's classification | 1 | Suspend its credentials; data governance reviews | Data governance |
| A product reports quality below its floor | 2 | Gap diagnosis; failures join the regression slice | Archetype owner |
| Labeled live outputs are not consistent with the slice | 2 | Check the samples for drift; refresh the slice | Product owner |
| Red-team success is higher than in the last round | 2 | Add the payloads to the held-out set; re-check products' Stage 10 conditions | Security |
| Safety-filter hits on the alias double the four-week average for a week | 2 | Look for an injection campaign; run a red-team round | Security |
| Judge agreement falls below 85% on a slice | 2 | Block the suite version until fixed | ML evaluation team |
| A seeded-error test catches fewer than 18 of 20 | 2 | The product counts as unchecked; re-derive its tier | Product owner |
| The owner, step records or residual-risk acceptance are missing | 2 | No new products until fixed | Framework owner |
| Latency p95 is above the archetype's class for a week | 2 | Capacity review | SRE |
| Volume reaches twice the profile, or input p95 is 25% above it | 3 | Update the profile; re-derive the requirements | Archetype owner |

### Changes

| Change | Such as | Approved by | Re-test |
| --- | --- | --- | --- |
| Minor | A new slice or sample refresh; a product joining at or below the tier; a profile change that leaves the requirements unchanged | Archetype owner | The changed slice |
| Moderate | A new language, longer context or stricter latency; the tier rising by one | By tier (Tiers) | Every model behind the alias |
| Major | A changed scope or output, which means a new archetype; a higher classification; reaching tier 3; a merge, split or retirement | Board | As for a new archetype |

A higher classification also needs data governance's decision, and every model behind the alias must be approved for the new level.

**Retiring:** name the replacement alias and give the same notice as a model deprecation. For embeddings, products re-index before the old alias retires.

**Stopping fast:** use the gateway kill switch for a model, roll back the alias mapping in `gateway-aliases`, revoke a product's credentials, or stop new products from joining. Products then run on the fallback in their profile.

## Risks and controls

| Risk | Control | How it is checked | Owner |
| --- | --- | --- | --- |
| Samples do not reflect live work, so a model passes its slice and fails in use | Samples from real work; label check; pilots before going active | Labeled live outputs are consistent with the slice | Product owner |
| One product or language is averaged away | Each product's floor is a hard G5 criterion | Quarterly comparison of each product's best model with the archetype's model | ML evaluation team |
| Data reaches a model or person not approved for it | Data governance sets classifications; slices are readable by classification; every model behind an alias is approved for its classification | Spot checks; alias check at G8 | Data governance |
| A run output does damage: a destructive query, insecure code, an unrequested tool call | Effect conditions; critical-error maximums; safety tests | At Stage 10, try the forbidden action with the product's credentials: it fails | Archetype owner |
| Injected instructions are reworded until they pass the tests | No egress at the hosting layer; trifecta check at Stage 10; held-out injection set; red-team rounds | Red-team success is no higher than in the last round | Security |
| Someone games the samples, or names a second product that will not exist | Outside labeler at tiers 2 and 3; named owners for planned products; the proposer does not approve; every run kept | The ML evaluation team re-labels 10% of a slice: at least 90% agree | ML evaluation team |
| The judge favors a model family, style or language | Pinned judge, never the candidate, calibrated per slice | At least 85% agreement with human labels | ML evaluation team |
| A change breaks products | No in-place scope or output change; versioned embedding aliases; shadow and canary cutover | Before a model change, at least 99% of each product's samples match the alias's output format | Archetype owner |
| Steps are skipped under deadline pressure | Reviewed pull requests that link the step records | Quarterly records check | Framework owner |
| Too many archetypes, or one model behind many aliases | Two-product rule; quarterly review; blast radius in each definition; tier 3 fallbacks | Products per archetype; aliases per model; fallback drills | Platform lead |
| A profile understates its risk | Tier derived from the fields; data governance sets the classification; seeded errors for "checked" claims | At least 18 of 20 seeded errors caught | Product owner |

## Templates

**Profile**

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
max_data_classification: internal     # set by data governance
risk:                                 # any of the first three yes: risk and legal review at Stage 10
  customer_facing: no
  automated_decisions: no
  regulated_data: no
  every_output_checked: no            # a person checks each output before it has any effect
  untrusted_content: yes              # customer messages, emails, uploads or outside documents reach the prompt
tier: 1                               # from the output and risk fields (Tiers)
acceptance: { metric: answer_correctness, floor: 0.85, max_critical_errors: 0.03 }
fallback: <what the product does when the alias is down>
cost_target_per_1k: <amount, if known>
samples: eval-data/domain/<archetype>-<slice>/seed/       # 50 to 100 labeled samples
```

**Definition** (filled in for the example below)

```yaml
# onboarding-config/archetypes/<id>.yaml
id: text-to-sql                       # lowercase, names the capability, never reused
owner: <platform model owner>
status: pilot                         # proposed | pilot | active | retired
task: code
output: query
scope: Write one read-only SQL query that answers a question over a supplied schema.
out_of_scope: [running the query (product code), general code (code), answers from documents (rag)]
products: [analytics-assistant, finance-reporting]
tier: 3
requirements:
  floors:                             # each product, on its own slice
    analytics-assistant: { execution_accuracy: 0.85, max_critical_errors: 0.03 }
    finance-reporting: { execution_accuracy: 0.85, max_critical_errors: 0.02 }
  latency: interactive
  context_length: 32k
  languages: [en, de]
  max_data_classification: confidential
  cost_target_per_1k: 20
conditions:                           # checked at Stage 10
  - Queries run with the end user's own database permissions, read-only
  - Results render as plain tables, with no links or markdown
  - "finance-reporting: a person reviews every report before publication"
blast_radius: 2 products, about 6,500 requests a day; the model serves no other alias
fallback: Analysts write SQL themselves; finance runs its manual reports
risks:
  - risk: A query reads a table the user may not see
    control: Queries run with the user's own permissions
    test: At Stage 10, ask for a table outside the user's grants; the database refuses
    owner: Product owner
  - risk: Instructions injected into column comments or row values, reworded when blocked
    control: Held-out injection set; quarterly red-team round
    test: Red-team success no higher than in the last round, and no data read outside the user's grants
    owner: Security
  - risk: A query returns the right columns and row count but wrong values
    control: Critical-error maximums; finance review before publication
    test: Critical-error rate per slice
    owner: Archetype owner
eval:
  suite: text-to-sql/v1
  slices: [analytics (300), finance (300), finance-de (100)]
  safety_additions: [destructive statements, injections in column comments and row values (held out)]
alias: text-to-sql-default
residual_risk:
  accepted_by: <board chair>, with <board risk member>
  risks: [wrong values within each product's critical-error maximum]
  rationale: Analysts check figures against source reports; finance reviews every report before publication.
  review_by: 2027-05-31
```

**Step record**

```yaml
# evidence/archetypes/<id>/step<n>-<yyyymmdd>.yaml
archetype: text-to-sql
step: 3                               # 2 fit check | 3 test | 4 approval | 5 readiness | 6 go active
decided_by: <user>                    # not the person who proposed it
decided_at: 2026-11-20
inputs: [profiles/analytics-assistant-sql.yaml@<commit>, profiles/finance-reporting-sql.yaml@<commit>, <every run, failed runs included>]
outcome: create an archetype
suitability: { tier: 3, likelihood: medium, decision: board decides at Step 4 }
conditions: []
reason: <a few sentences>
```

## Example: text to SQL

The products and numbers are illustrative.

**Step 1.** Two products submit `code` / `query` profiles. An analytics assistant is live and interactive, with prompts up to 14,000 tokens, English only, a floor of 0.85, critical errors at most 3% and a cost target of $20 per 1,000 requests. A finance reporting tool is planned for Q1 2027 and runs in batch, with prompts up to 18,000 tokens, English and German, a floor of 0.85, critical errors at most 2% and a cost target of $30. Data governance sets both to confidential, because schemas and sample rows reveal business data. Queries run (effect high) on confidential data (data high), so both are tier 3; oversight is low, because they serve internal staff and make no automated decisions. Row values and column comments can hold customer text, so content is untrusted. With an outside second labeler, the label checks agree on 47 and 46 of 50.

**Step 2.** No archetype has `code` / `query` (code assistance is `code` / `code`), and both products have named owners: go to Step 3.

**Step 3.** Execution accuracy on 100 samples per product, with improved prompts:

| Model | Code assistance eval | Analytics samples | Finance samples |
| --- | --- | --- | --- |
| M1 (behind `code-default`) | 0.78 | 0.55 | 0.50 |
| M2 (behind `rag-large`) | 0.71 | 0.71 | 0.66 |
| M3 (behind `agents-default`) | 0.64 | 0.69 | 0.67 |

No hosted model meets the floor of 0.85. An open-weight candidate scores 0.86 on 100 cleared samples, with 2% critical errors, and passes each product's 20 injection samples. The code assistance eval ranks M1 first, but M1 is 16 to 17 points below the best model on both products' samples, so the existing eval would pick the wrong model: create an archetype. Likelihood is medium (the candidate is within 5 points of the floor, and its critical errors are at finance's maximum), so at tier 3 the board decides.

**Step 4.** The definition is the template above. Context length is 1.2 × (18,000 + 400) = 22,080 tokens, rounded up to 32k. The board judges the benefit high, because analysts wait days for the data team and finance builds its reports by hand, and approves with the conditions in the definition. The board chair, with the risk member, accepts wrong values within the critical-error maximums, for review in six months.

**Step 5.** Slices of 300 samples per product and 100 in German, scored by running each query against a sandbox database inside the eval job. A deliberately weak model fails every floor, and among hosted models the suite ranks M2 first, as Step 3 did. Status becomes `pilot`, and the candidate enters Stage 0.

**Step 6.** At Stage 5, the candidate scores 0.91 on analytics (95% Wilson interval 0.87 to 0.94), 0.90 on finance (0.86 to 0.93) and 0.87 on finance-de, with 1.0% and 1.3% critical errors. In the first red-team round, 2 of 250 adapted payloads change a query, and neither reads data outside the user's grants; both join the held-out set. Both products pilot the model and go live. After early-life watch, 50 labeled live questions per product score 0.88 (0.76 to 0.94) and 0.87 (0.75 to 0.94), consistent with their slices, and both fallback drills work. The board approves, and the status becomes `active`.

**First quarter.** Analytics input p95 rises to 19,000 tokens (severity 3). The owner updates the profile: 1.2 × (19,000 + 400) = 23,280 tokens, still within 32k, so it is a minor change. Finance's review catches a total with the right columns and row count but wrong values. The question and nine variants join the regression slice, and the critical-error rate stays within 2%.

## Where things live

- Profiles: `onboarding-config/profiles/<product>-<use-case>.yaml`
- Definitions: `onboarding-config/archetypes/<id>.yaml`
- Samples and slices: `eval-data/domain/<id>-<slice>/`
- Step records: `evidence/archetypes/<id>/`, through the record pipeline (P5)

## Known gaps

**Done by hand until automated:**

- Step records are attached to the definition's pull request until P5 accepts archetype records.
- Profile and definition checks (fields, tier, derived requirements, linked records, no in-place scope or output change) happen in pull-request review. A reviewer other than the author recomputes derived values.
- The board chair checks that the proposer does not approve, until P5 enforces it.
- SRE checks each alias against the archetype's classification at G8, until P6 does.
- The ML evaluation team runs the readiness check, and the output-format check before a model change.
- Stage 10 does not yet check archetype conditions, so the archetype owner reviews them there and witnesses the forbidden-action test.
- P6 does not yet scope credentials per alias and environment or lapse them after 12 months; the quarterly review tracks unconfirmed profiles.

**Companion documents:** the onboarding process's gap diagnosis says "tier" for what this document calls a variant (such as `rag-large`), and it assigns all domain slices to DeepEval, while slices scored by execution run on LightEval.

**Open questions:**

- [ ] Who owns each starting archetype?
- [ ] Which product teams supply the first profiles and samples, and by when?
- [ ] Which data classification levels does the platform use? The tiers assume a confidential level above internal.
- [ ] Until the model review board meets regularly, who approves a new archetype?
- [ ] When an approved model is added to a new archetype, can its Stage 3 and Stage 4 results be reused, since its weights are unchanged?
- [ ] Should automated decisions with legal or similar effects on individuals stay out of scope, and who sets their conditions?
- [ ] Does checking tier 3 floors on the lower end of the 95% interval suit the first products?
- [ ] Where do the metering and safety-filter signals live?

## Feedback and history

Send feedback as an issue or pull request in `red-arg/onboard-llm`. The framework owner reviews it each quarter.

| Version | Date | Change |
| --- | --- | --- |
| 0.1 | Oct 29, 2026 | Minimum method for the platform's start |
| 0.2 | Oct 30, 2026 | Added tiers, risks and controls, roles and monitoring |
| 0.3 | Oct 31, 2026 | Rewritten as a concise operator guide |
