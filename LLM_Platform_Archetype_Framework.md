# LLM Platform: Archetype Framework

Version 0.7 · Nov 10, 2026 · Owner: @Zhi Feng · Next review: Jan 31, 2027

## Purpose

The [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) selects models for archetypes, not for individual products. An archetype groups use cases that need the same task and output from a model, so one model, one eval suite and one gateway alias can serve them all.

This document tells each role (Roles) when to create, change or retire an archetype, and what to do at each step. Repositories and pipelines are described in the [technical implementation](LLM_Platform_Model_Onboarding_Technical_Implementation.md).

## When to do what

| Situation | Do this | See |
| --- | --- | --- |
| A product needs a model for a use case | Write a profile with labeled samples, then run the fit check | Steps 1 and 2 |
| The use case matches an existing archetype | Join it through Stage 10 | Step 2 |
| It matches, but fails its Stage 10 domain test | Run gap diagnosis | Step 2 |
| A product needs a task or output no archetype covers | Test models, then join, onboard a stronger model or create an archetype, as the test decides | Steps 3 to 6 |
| A product needs another language or a higher data classification | Extend the archetype | Step 2, Changes |
| A product's latency, volume, oversight or data changes | Update the profile, and re-derive the requirements | Changes |
| An archetype's scope or output must change | Create a new archetype and retire the old one | Changes |
| The quarterly re-evaluation is due | Run the quarterly review | Quarterly review |
| A monitoring signal fires | Follow the signal table | Signals |
| An archetype has had no products for two quarters | Retire it | Changes |

## Rules

1. **Group by task and output.** Use cases with the same task and output share an archetype, unless the Step 3 test shows that one model cannot serve them all.
2. **Create on evidence.** Send any use case no archetype serves to the Step 3 test and follow its decision. One product is enough, live or planned within two quarters, with a named owner.
3. **Smallest change first.** Join, then extend, then onboard a stronger model. Create a new archetype last.
4. **Each product keeps its own bar.** Every product's floor is checked on its own samples, so an average never hides one product's failure.
5. **Decide with scores** on the products' own samples.
6. **One standard path.** Every archetype gets the same checks. The platform answers for its measurements, data classification and the shared service; each Business Owner answers for how their product uses the output.
7. **No breaking changes in place.** Never narrow an archetype's scope or change its output; create a new archetype. Never reuse an id.

## Roles

| Role | Responsibilities |
| --- | --- |
| Business Owner (owns the product) | Approves the product's profiles, including floors, critical-error maximums and fallback; sets the data classification; clears samples for outside models; attests to the product responsibilities and accepts the product's residual risk; reconfirms profiles yearly |
| Developers (build the product) | Write profiles and label samples; build the product to meet the archetype's conditions and the product responsibilities; switch to the fallback and report incidents; label live outputs during the pilot |
| Platform Owner | Approves new archetypes, variants, major changes and retirements; reviews the catalog each quarter; keeps this document current |
| Platform data science team | Runs the Step 3 test; builds and re-runs the eval; audits slice labels each quarter; runs red-team rounds |
| Production Engineers (platform engineers) | Run the gateway and aliases: credentials, metering, alerts, kill switch and capacity |

Each archetype has an **archetype owner**: a named member of the platform data science team, who runs the fit check for the use cases nearest that archetype, writes the definition, proposes pilots, runs the quarterly review and acts on signals. One person can own several archetypes.

The Platform Owner also checks every archetype's records each quarter and runs the yearly seeded proposal. Whoever proposes an archetype or a candidate model does not decide its Step 3 test or approve it at Step 4.

## Standard checks

The platform provides models; it does not run products. Every archetype therefore gets the same checks, whatever its products do with the output.

| Who | Answers for |
| --- | --- |
| The platform | Its measurements (a model that passes really meets the product's floor), the data classification each model is approved for, and the shared service |
| The Business Owner | How the product uses the output: what it runs, who sees it and who checks it. The Business Owner sets the floor and the critical-error maximum, and accepts the product's residual risk |

**Platform checks.** The same for every archetype. Each step below states its own.

**Data classification gate.** Every model behind an alias is approved for the highest classification any of its products sends. A product sends nothing above the archetype's classification, and uses masked or synthetic samples for outside reference runs. This is checked at Stage 10 and G8.

**Product responsibilities.** The Business Owner attests to these in the profile when they apply. The platform records the attestation at Stage 10 and does not test it:

| When the product | The Business Owner attests that |
| --- | --- |
| Runs the output (`code`, `query` or `tool-calls`) | Outputs run with the end user's own permissions, read-only where the task allows, and a person confirms any destructive or outbound action |
| Sends outputs to customers or the public, or lets them drive decisions or actions, with no person checking each one | People are told at the point of use that the output is AI-generated and may be wrong, and uncertain cases go to a person |

**Untrusted content** (customer messages, emails, uploads, outside documents) adds injection tests to the eval, through every kind of content a product sends.

**Deviation from the design standard.** The [design standard](Framework_Design_Standard_for_AI_Software_and_Product.md) asks frameworks to tier obligations by risk. This framework does not: the platform's obligations are the same for every product, and the obligations that vary with product risk belong to the Business Owner.

## Starting archetypes

The platform starts with these archetypes. Each one is built only when a product needs it.

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

## Creating an archetype

### When to create one

Create a new archetype only when all of these hold:

1. At least one product needs it, live or planned within two quarters, with a named owner.
2. No archetype covers its task and output, or the Step 3 test shows that the nearest archetype's eval would pick the wrong model for these products.
3. At least one model, hosted or an open-weight candidate, meets every product's floor.
4. Suitability allows it (Step 3).

Otherwise, join or extend an existing archetype, or onboard a stronger model for it.

### Steps

| Step | Who | Done when |
| --- | --- | --- |
| 1. Profile | Developers; Business Owner approves | Profiles complete |
| 2. Fit check | Nearest archetype's owner | Outcome recorded |
| 3. Test | Platform data science team | Decision and suitability recorded |
| 4. Define and approve | Archetype owner; Platform Owner approves; Business Owners accept residual risk | Definition approved; residual risk accepted |
| 5. Build the eval | Platform data science team | Readiness check passed; Stage 0 opened for the first model |
| 6. Pilot and go active | Archetype owner | Go-active criteria met |

The archetype's status is `proposed` once Step 3 decides to create it, `pilot` after Step 5 and `active` after Step 6. Record each step's decision (template below) in `evidence/archetypes/<id>/`.

### Step 1. Profile

The product's Developers fill in one profile per use case (template below), with at least 100 labeled samples from its real or planned work, and its Business Owner approves it. The same profile describes the use case at Stage 10.

- **One task per profile.** A use case with two tasks, such as answering from documents and drafting emails, is two profiles and calls two aliases. Answers from the model's own knowledge, rather than from supplied sources, are not `answer`, and no starting archetype covers them.
- **Classification** is set by the Business Owner, who owns the product's data. Samples are stored in `eval-data`, readable only by people cleared for that classification.
- **Critical errors:** the Business Owner sets `max_critical_errors`. As a guide, use 1% where outputs reach customers or drive actions with no person checking each one, and 3% otherwise.
- **Product responsibilities:** the Business Owner attests to those that apply (Standard checks).
- **Complete** when every field is filled, every sample has an expected output, and the label check passes: two labelers agree on 27 of 30 samples. If labelers agree less often, fix the criteria first, as in gap diagnosis Step 1.

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

The owner of the nearest archetype, by task and output, compares the profile with the existing archetypes.

Refuse at this step:

- a model that would hold tools or network access itself, because tools live in products;
- a use case with all three legs of the lethal trifecta, until the product removes one;
- automated decisions with legal or similar effects on individuals, until the Platform Owner and the product's Business Owner set conditions.

| The use case | Outcome |
| --- | --- |
| Matches an archetype's task and output, and fits its latency, context length, languages and data classification | Join through Stage 10, and add the samples to its eval as a new slice |
| Matches, but needs another language or a higher data classification | Extend the archetype (Changes), then re-test its models |
| Matches, but failed its Stage 10 domain test | Run gap diagnosis. If it finds a catalog gap, go to Step 3 |
| Has a task or output no archetype covers | Go to Step 3 |

### Step 3. Test

On at least 100 of each product's samples, plus at least 20 injection samples if its content is untrusted, the platform data science team scores:

- the model behind the nearest archetype's alias, and the other hosted models that can produce the output;
- promising open-weight candidates, on samples that the Business Owner clears for outside use.

The team writes the injection samples, and also runs the nearest archetype's own eval on the same hosted models. Use an improved prompt for every model (gap diagnosis Step 2), so that a prompt problem does not look like a model gap. Scores within 5 points of each other are a **tie**, because smaller differences can be chance at this sample size. Keep every run.

While the platform hosts no models, a starting archetype skips this test, because there is nothing to compare. Record that in the step record, and treat its likelihood as medium.

The first row that matches decides:

| Result | Decision |
| --- | --- |
| The model behind the nearest archetype's alias meets every product's floor | Join that archetype |
| No model meets the floors, hosted or candidate | Not now: redesign the workflow (gap diagnosis Step 5), and re-test at the next quarterly re-evaluation |
| The model the nearest archetype's eval ranks first ties with the best hosted model on the samples | Onboard a stronger model for that archetype through Stage 0 |
| Otherwise | Create an archetype: the existing eval would pick the wrong model for these products |

When the decision is to create an archetype, the Platform Owner names its archetype owner from the platform data science team.

**Suitability.** Before creating an archetype or onboarding a stronger model, the archetype owner rates the likelihood of failure from the best available model:

- **Low:** it clears every floor by more than a tie, and its critical-error rate is under half the maximum.
- **High:** it misses a floor or the critical-error maximum.
- **Medium:** anything else.

| Likelihood | Decision |
| --- | --- |
| Low | Proceed |
| Medium | Proceed. Tell each Business Owner the margin, and record it as a known weakness in the routing guide |
| High | Not now |

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

The definition also records:

- **Scope and out of scope:** written as the capability, never as a product, so that a later product with the same need joins or extends the archetype instead of creating another.
- **Conditions:** any the archetype needs, such as read-only database credentials. Business Owners attest to them with the product responsibilities (Standard checks).
- **Blast radius and fallback:** the products, their volume, any other alias the same model serves, and what products do when the alias is down.
- **Risks:** at least one archetype-specific risk, each with a control, a test and an owner. If any product sends untrusted content, at least one of them is an attack.
- **Residual risk:** what is left after controls, why it is acceptable, and a review date within a year, accepted by each product's Business Owner. Past the review date, the product's credentials lapse after notice, as for an unconfirmed profile.

The Platform Owner approves, with conditions if needed.

If one product needs a much faster or cheaper model than the rest, it can have a variant alias, such as `<id>-fast`, so that the others keep the stronger model. One product is enough; the Platform Owner approves it.

### Step 5. Build the eval

The platform data science team builds the archetype's eval suite:

- **One slice per product:** 100 samples to start, grown to 200 before the first model's Stage 5 benchmark; the product's Developers supply and label the added samples. **One slice per extra language:** 50 to 100 samples.
- **One reference prompt,** starting from the improved prompt used in Step 3.
- **Safety:** the standard safety battery, plus archetype tests, such as instructions injected into retrieved passages (`rag`), source documents (`summarization`) or tool results (`agents`), or destructive statements (`text-to-sql`). Test injection through every kind of untrusted content a product sends. If content is untrusted, also build a held-out injection set, kept away from Developers, and run a red-team round each year.
- **Public benchmarks:** optional, worth at most 20% of the ranking.
- **G5 policy:** each product's floor and maximum critical-error rate are hard criteria. Latency and cost go to review. Rank the models that pass by the average of the product slices, with each product weighted equally.
- **Judge:** pinned, internally hosted and never the candidate. It must agree with human labels on at least 85% of each slice's 50 calibration samples. Judge metrics are the mean of three passes.

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
- the model it ranks first is the one Step 3 found best, or ties with it (Step 3). This does not apply when Step 3 was skipped.

Then open Stage 0 for the archetype's first model.

### Step 6. Pilot and go active

The archetype owner proposes one product as the pilot for the first model's onboarding, and a second if the archetype has one. The pilots take the model through Stages 6 to 11.

The archetype goes active when:

- the first model is GA;
- every pilot is through early-life watch;
- each pilot's labeled live results, on at least 50 outputs, are consistent with its slice, meaning the slice score is inside the live score's 95% interval;
- no severity 1 or 2 signal is open;
- the routing guide entry is published, with the archetype's scope, out of scope, floors, conditions and known weaknesses.

The archetype owner and a second data science team member approve. Once the status is `active`, any product can join through Stage 10.

## Running archetypes

### Quarterly review

At each quarterly re-evaluation, the archetype owner:

1. Reviews the P7 re-evaluation of every model behind the alias. The data science team runs it, with incident and gap diagnosis cases added to the regression slice.
2. Checks each product's usage against its profile, re-derives the requirements for changed profiles, and asks Business Owners to reconfirm profiles older than 12 months. Credentials for unconfirmed profiles lapse after notice.
3. Asks the data science team to run Step 3 for any product whose best model has beaten the archetype's model by more than a tie (Step 3) at two re-evaluations in a row.
4. Proposes to the Platform Owner retiring the archetype if it has had no products for two quarters, or merging it with another archetype of the same task and output when one model meets every product's floor in both.
5. Updates the blast radius, including any other alias the archetype's model now serves, and checks the residual-risk review date and the routing guide entry.
6. Reviews decisions that were later reversed or that overrode a rule, and their reasons.

In the same review:

- The data science team re-labels a random 10% of one product slice per archetype, taking a different slice each quarter. If fewer than 90% of the labels agree, the slice is re-labeled before it is used again.
- The Platform Owner reviews products per archetype and aliases per model. Where one model backs several archetypes, the Platform Owner decides with the Production Engineers whether to add a fallback model (Stage 8).
- The Platform Owner checks that every pilot or active archetype has an owner, its step records with the right approvers and a current residual-risk acceptance. Once a year, the Platform Owner also sends a seeded proposal, such as a profile with too low a data classification, through the steps to check that it is caught.

### Signals

Severity 1: act the same day. Severity 2: within two weeks. Severity 3: at the next quarterly review.

| Signal | Severity | Response | Owner |
| --- | --- | --- | --- |
| Critical-error incident: a destructive or unrequested action, or data shown to the wrong person | 1 | The product switches to its fallback or disables the feature; kill switch or rollback if the model is at fault; review within 5 working days; cases join the regression slice | Business Owner, Production Engineers, archetype owner |
| A product sends data above the archetype's classification | 1 | Suspend its credentials; the Business Owner and Platform Owner review | Production Engineers |
| A product reports quality below its floor | 2 | Gap diagnosis; failures join the regression slice | Archetype owner |
| Labeled live outputs are not consistent with the slice | 2 | Check the samples for drift; refresh the slice | Developers, archetype owner |
| Red-team success is higher than in the last round | 2 | Add the payloads to the held-out set; tell the products' Business Owners | Platform data science team |
| Safety-filter hits on the alias double the four-week average for a week | 2 | Look for an injection campaign; run a red-team round | Production Engineers, platform data science team |
| Judge agreement falls below 85% on a slice | 2 | Block the suite version until fixed | Platform data science team |
| The owner, step records or residual-risk acceptance are missing | 2 | No new products until fixed | Platform Owner |
| Latency p95 is above the archetype's class for a week | 2 | Capacity review | Production Engineers |
| Volume reaches twice the profile, or input p95 is 25% above it | 3 | Update the profile; re-derive the requirements | Archetype owner |

### Changes

| Change | Such as | Approved by | Re-test |
| --- | --- | --- | --- |
| Minor | A new slice or sample refresh; a product joining; a profile change that leaves the requirements unchanged | Archetype owner | The changed slice |
| Moderate | A new language, longer context or stricter latency | Archetype owner and a second data science team member | Every model behind the alias |
| Major | A changed scope or output, which means a new archetype; a higher classification; a merge, split or retirement | Platform Owner | As for a new archetype |

For a higher classification, the Business Owner classifies the new data, and every model behind the alias must be approved for the new level.

**Changing the model behind an alias:** on the new model, at least 99% of each product's samples must match the alias's output format. Then cut over as the onboarding process describes: shadow traffic, then a canary, then full cutover.

**Retiring:** name the replacement alias and give the same notice as a model deprecation. For embeddings, products re-index before the old alias retires.

**Stopping fast:** use the gateway kill switch for a model, roll back the alias mapping in `gateway-aliases`, revoke a product's credentials, or stop new products from joining. Products then run on the fallback in their profile.

## Templates

**Profile**

```yaml
# onboarding-config/profiles/<product>-<use-case>.yaml
product: <product>
owner: <Business Owner>
use_case: <what the product asks the model to do, in one line>
status: live                          # live | planned, with the quarter
task: answer
output: prose
latency: interactive                  # real-time (first token within 300 ms) | interactive (1.5 s) | batch
lengths: { input_p95: 9000, output_p95: 600 }             # tokens
volume: { requests_per_day: 5000, peak_per_minute: 40 }
languages: [en]
max_data_classification: internal     # set by the Business Owner
risk:                                 # decides which product responsibilities apply (Standard checks)
  customer_facing: no
  automated_decisions: no
  regulated_data: no
  every_output_checked: no            # a person checks each output before it has any effect
  untrusted_content: yes              # customer messages, emails, uploads or outside documents reach the prompt
responsibilities_attested: <Business Owner, date>         # the product responsibilities that apply
acceptance: { metric: answer_correctness, floor: 0.85, max_critical_errors: 0.03 }
fallback: <what the product does when the alias is down>
cost_target_per_1k: <amount, if known>
samples: eval-data/domain/<archetype>-<slice>/seed/       # at least 100 labeled samples
```

**Definition** (filled in for the example below)

```yaml
# onboarding-config/archetypes/<id>.yaml
id: text-to-sql                       # lowercase, names the capability, never reused
owner: <archetype owner, from the platform data science team>
status: proposed                      # proposed | pilot | active | retired
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
conditions:                           # attested by the Business Owners at Stage 10
  - Queries run with the end user's own database permissions, read-only
  - Results render as plain tables, with no links or markdown
  - "finance-reporting: a person reviews every report before publication"
blast_radius: 2 products, about 6,500 requests a day; the model serves no other alias
fallback: Analysts write SQL themselves; finance runs its manual reports
risks:
  - risk: A query reads a table the user may not see
    control: Queries run with the user's own permissions
    test: At Stage 10, ask for a table outside the user's grants; the database refuses
    owner: Developers
  - risk: Instructions injected into column comments or row values, reworded when blocked
    control: Held-out injection set; yearly red-team round
    test: Red-team success no higher than in the last round, and no data read outside the user's grants
    owner: Platform data science team
  - risk: A query returns the right columns and row count but wrong values
    control: Critical-error maximums; finance review before publication
    test: Critical-error rate per slice
    owner: Archetype owner
eval:
  suite: text-to-sql/v1
  slices: [analytics (200), finance (200), finance-de (100)]
  safety_additions: [destructive statements, injections in column comments and row values (held out)]
alias: text-to-sql-default
residual_risk:
  accepted_by: <the Business Owners of both products>
  risks: [wrong values within each product's critical-error maximum]
  rationale: Analysts check figures against source reports; finance reviews every report before publication.
  review_by: 2027-11-30
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
suitability: { likelihood: medium, decision: proceed, Business Owners told the margin }
conditions: []
reason: <a few sentences>
```

## Example: text to SQL

The products and numbers are illustrative.

**Step 1.** Two products submit `code` / `query` profiles. An analytics assistant is live and interactive, with prompts up to 14,000 tokens, English only, a floor of 0.85, critical errors at most 3% and a cost target of $20 per 1,000 requests. A finance reporting tool is planned for Q1 2027 and runs in batch, with prompts up to 18,000 tokens, English and German, a floor of 0.85, critical errors at most 2% and a cost target of $30. The Business Owners set both to confidential, because schemas and sample rows reveal business data. Queries are run, so both Business Owners attest that they run with the end user's own permissions, read-only. Both products serve internal staff and make no automated decisions. Row values and column comments can hold customer text, so content is untrusted. The label checks agree on 28 and 27 of 30.

**Step 2.** No archetype has `code` / `query` (code assistance is `code` / `code`), and both products have named Business Owners: go to Step 3.

**Step 3.** Execution accuracy on 100 samples per product, with improved prompts:

| Model | Code assistance eval | Analytics samples | Finance samples |
| --- | --- | --- | --- |
| M1 (behind `code-default`) | 0.78 | 0.55 | 0.50 |
| M2 (behind `rag-large`) | 0.71 | 0.71 | 0.66 |
| M3 (behind `agents-default`) | 0.64 | 0.69 | 0.67 |

No hosted model meets the floor of 0.85. An open-weight candidate scores 0.86 on 100 cleared samples, with 2% critical errors, and passes each product's 20 injection samples. The code assistance eval ranks M1 first, but M1 is 16 to 17 points below the best model on both products' samples, so the existing eval would pick the wrong model: create an archetype. Likelihood is medium (the candidate clears the floor by 1 point, which is a tie, and its critical errors are at finance's maximum), so the archetype proceeds, and both Business Owners are told the margin.

**Step 4.** The definition is the template above. Context length is 1.2 × (18,000 + 400) = 22,080 tokens, rounded up to 32k. The Platform Owner approves the definition. Both Business Owners attest to its conditions and accept wrong values within their critical-error maximums, for review within a year.

## Where things live

- Profiles: `onboarding-config/profiles/<product>-<use-case>.yaml`
- Definitions: `onboarding-config/archetypes/<id>.yaml`
- Samples and slices: `eval-data/domain/<id>-<slice>/`
- Step records: `evidence/archetypes/<id>/`, through the record pipeline (P5)

## Known gaps

**Done by hand until automated:**

- Step records are attached to the definition's pull request until P5 accepts archetype records.
- Profile and definition checks (fields, attestations, derived requirements, linked records, no in-place scope or output change) happen in pull-request review. A reviewer other than the author recomputes derived values.
- The Platform Owner checks that the proposer does not approve, until P5 enforces it.
- Production Engineers check each alias against the archetype's classification at G8, until P6 does.
- The data science team runs the readiness check, and the output-format check before a model change.
- Stage 10 does not yet record the Business Owner's attestation, so the archetype owner collects it there.
- P6 does not yet scope credentials per alias and environment or lapse them after 12 months; the quarterly review tracks unconfirmed profiles.

**Companion documents:** the onboarding process's gap diagnosis says "tier" for what this document calls a variant (such as `rag-large`), and it assigns all domain slices to DeepEval, while slices scored by execution run on LightEval. Its gap diagnosis asks for two or more products before a new archetype, where this document needs one. Its Stage 10 sends higher-risk use cases to risk and legal review, where this document takes the Business Owner's attestation. Both still use the earlier roles: read the model review board and platform lead as the Platform Owner, the platform model owner and ML evaluation team as the platform data science team, SRE as Production Engineers, data governance as the Business Owner, and product teams as the Business Owner and Developers.

**Planned work:** review this document's thresholds after the first few archetypes.

**Open questions:**

- [ ] Which data science team member owns each starting archetype?
- [ ] Which Business Owners supply the first profiles and samples, and by when?
- [ ] Which data classification levels does the platform use?
- [ ] Does the Platform Owner accept the deviation from the design standard's tiering rule (Standard checks)?
- [ ] Who approves in the Platform Owner's place when the Platform Owner proposed the change?
- [ ] When an approved model is added to a new archetype, can its Stage 3 and Stage 4 results be reused, since its weights are unchanged?
- [ ] With no legal role, should automated decisions with legal or similar effects on individuals stay out of scope?
- [ ] Where do the metering and safety-filter signals live?

## Feedback and history

Send feedback as an issue or pull request in `red-arg/onboard-llm`. The Platform Owner reviews it each quarter.

| Version | Date | Change |
| --- | --- | --- |
| 0.1 | Oct 29, 2026 | Minimum method for the platform's start |
| 0.2 | Oct 30, 2026 | Added tiers, risks and controls, roles and monitoring |
| 0.3 | Oct 31, 2026 | Rewritten as a concise operator guide |
| 0.4 | Nov 1, 2026 | Folded the risks-and-controls table into the steps and reviews, trimmed the example, aligned owners |
| 0.5 | Nov 10, 2026 | For a low-volume platform: one product is enough to create an archetype or variant, the Step 3 test decides; one pilot; retire at no products; tier 1 Step 3 tests use 100 samples; scopes name the capability |
| 0.6 | Nov 10, 2026 | Roles reduced to the five the platform has: Business Owner, Developers, Platform Owner, Production Engineers and the platform data science team. The Platform Owner takes the board's approvals, and archetype owners come from the data science team |
| 0.7 | Nov 10, 2026 | Tiers replaced by one standard path: the platform answers for its measurements, data classification and the shared service, and Business Owners attest to product responsibilities and accept their product's residual risk. Each threshold sits in the step that uses it. Getting started reduced to the starting archetypes table. Step 3 defines a tie once, and sample counts are minimums. Step targets removed; profiles carry at least 100 samples; the nearest archetype's owner runs the fit check; pilots label at least 50 live outputs |
