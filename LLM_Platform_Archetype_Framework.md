# LLM Platform: Archetype Framework

Oct 29, 2026 · @Zhi Feng

## Purpose and scope

The [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) selects models for use-case archetypes rather than for individual products. An archetype describes a set of products that need the same capability from a model, so that one set of requirements, one eval suite and one gateway alias can serve all of them.

This framework defines how a new archetype is proposed, justified, specified, built and approved, and how archetypes are changed, split, merged and retired afterwards. It runs on the repositories and pipelines described in the [technical implementation](LLM_Platform_Model_Onboarding_Technical_Implementation.md).

In scope: new capability archetypes, domain archetypes and tiers, changes to existing archetypes, and setting up the starting catalog. Out of scope: onboarding the models that serve an archetype and approving a product's use case, which follow Stages 0 to 11 of the model onboarding process.

## What an archetype is

An archetype groups products by the capability they need from a model, not by team or business area. Two products belong to the same archetype when the same model, measured by the same eval suite, is the right choice for both. A product that needs several capabilities calls several aliases, one per archetype; it does not need an archetype of its own.

Each archetype is a named, versioned contract between the platform and the products that build on it. It fixes:

- **Scope:** the task products ask the model to do, and where it ends.
- **Interface:** what goes in and what comes out (chat or embeddings, free text or structured output, tool calls), and typical prompt and output lengths.
- **Requirements:** minimum quality bar, latency target, context length, languages, maximum data classification and target cost per request.
- **Eval suite and policy:** how a model is measured against the requirements, versioned so scores stay comparable across models.
- **Aliases:** the gateway names products call, such as `rag-default`.

Products request an archetype's alias at Stage 10. Models are evaluated and admitted for one or more archetypes at Stages 5 to 8. Because one model can serve several archetypes, a new archetype does not always need a new deployment.

### Levels of change

Most new needs do not need a new archetype. From the smallest change to the largest:

| Change | When it fits | Example | Approved by |
| --- | --- | --- | --- |
| Domain slice | Same task and interface; only the content differs | Insurance claim forms added to the extraction suite | ML evaluation team |
| Language extension | Same task and interface in another language | German added to summarization | Archetype owner, ML evaluation team |
| Tier | Same task and interface at a different trade-off between quality, latency and cost | `rag-large`: a higher quality bar at a higher cost per request | Archetype owner, platform lead |
| Domain archetype | Same task and interface, but two or more products share a domain where general models fall short, and the parent's suite cannot pick the right model for it | `rag-legal`: answers over contracts and case law | Model review board |
| Capability archetype | A task or interface that no existing archetype covers | `text-to-sql` (see Worked example) | Model review board |

A domain slice produces a new suite version, and a language extension or tier changes the archetype definition (see Changing an archetype). Only domain and capability archetypes go through the creation flow below.

## Principles

- **Start from products, not models.** An archetype exists because products need it. A model release with a new capability is never a reason for one; horizon scanning feeds candidates into existing archetypes at Stage 0.
- **Shared need.** An archetype serves at least two products, live or committed on a roadmap. A need that only one product has is usually met inside an existing archetype, through its prompt, a domain slice or a tier.
- **Smallest change that works.** Every archetype costs an eval suite to maintain, models to host and a re-evaluation every cycle, and the catalog must stay short and stable. Prefer the smallest change in the table above.
- **Evidence before definition.** A new archetype is justified by scoring hosted models on the sponsors' own samples, not by how different the need sounds.
- **Measurable or it does not count.** Every requirement used to select models names the metric and threshold that check it. A requirement that no test can measure cannot select models.
- **Stable for products.** Products build on an archetype's scope, interface and aliases, so a change that would break a member product is never made in place.

## Roles and responsibilities

| Role | What they do in this framework |
| --- | --- |
| Sponsor product teams | Propose the archetype; supply seed samples, acceptance criteria and expected volumes; present the need at A3 (Consulted, not approvers); pilot its first models at Stage 6. |
| Archetype owner | One named person on the platform for the archetype's whole life, usually a platform model owner. Runs the proposal, writes and maintains the definition, leads catalog reviews, and proposes changes, splits, merges and retirement. |
| ML evaluation team | Runs the evidence step; translates requirements into metrics; builds, versions and calibrates the eval suite; writes the G5 policy; keeps the reference prompts. |
| Data governance / privacy | Sets the archetype's maximum data classification; approves the classification of seed samples and domain slices. |
| Security | Writes the archetype's risk profile, adds archetype-specific tests to the safety battery, and approves any new tool image or host the suite needs. |
| Legal | Reviews the licenses of datasets the suite adds; flags model license restrictions that conflict with the archetype's uses. |
| Platform SRE / infrastructure | Load profile, capacity and cost estimate, alias and gateway configuration. |
| Platform lead | Assigns archetype owners; approves tiers for the capacity they add. |
| Model review board | Admits new archetypes to the catalog; approves splits, merges and retirements. |
| Member product teams | Contribute domain slices; receive notice of changes; migrate when an archetype is split, merged or retired. |

## Process at a glance

```
 Trigger ──► 1 Proposal ──A1──► 2 Evidence ──A2──► 3 Definition ──A3──► 4 Build ──A4──► 5 First models
                 │                  │                           (board)                      │
                 ▼                  ▼                                                        ▼
              Closed:            Closed:                                            alias live at the
           smaller change     smaller change or                                     first model's Go Live
                              workflow redesign

 Status:     Proposed ────────────────────────────────────────────────► Approved ─────► Active
```

Steps 1 and 2 are cheap and close most proposals with a smaller change: an existing archetype, a domain slice, a language extension, a tier or a new model for an existing archetype. Steps 3 and 4 run only when the evidence shows that a new archetype is needed. A closed proposal records its reason and the path the sponsors take instead. Gates A1 to A4 belong to this framework; G1 to G8 remain the model onboarding gates.

## When a new archetype is justified

### Triggers

A proposal starts from one of these:

- Gap diagnosis finds a catalog gap that two or more products share.
- A product team reports that no archetype covers its task or interface (a Stage 0 archetype gap).
- Roadmap planning shows two or more products about to need a capability the catalog lacks.
- A catalog review finds an archetype whose members need different models (a split), or two archetypes that keep selecting the same model (a merge).

### Fit check

The archetype owner compares the need with the nearest existing archetypes, one dimension at a time. Each dimension that differs calls for the change in the last column. Only task, interface and, with evidence, content can lead to a new archetype.

| Dimension | Question | If it differs |
| --- | --- | --- |
| Task | What does the model do: answer from sources, summarize, extract to a schema, write code, call tools, embed? | Capability archetype |
| Interface | What goes in and what comes out: chat, structured output, tool calls or vectors? | Capability archetype |
| Content | Which domain does the content come from? | Domain slice; a domain archetype only if the evidence step shows the parent's suite cannot pick the right model |
| Languages | Which languages must work? | Language extension. The multilingual archetype is for tasks where crossing languages is the task itself, such as translation |
| Requirement values | Quality bar, latency, context length, cost | Tier, or a change to the archetype's requirements when it does not raise cost or latency for the other members |
| Data classification | What is the highest classification the product will send? | Data governance raises the archetype's maximum. It takes effect once the models behind its aliases are approved at the new level, which is a new onboarding |
| Risk | Is it customer-facing, does it make automated decisions, which trifecta legs does it have? | None: risk is reviewed per product at Stage 10 |

### Qualifying criteria

A new archetype is created only when all of these hold. A1 checks the first two on paper, A2 confirms them with evidence, and A3 checks the rest.

1. **Shared:** at least two sponsor products, live or with the need on a committed roadmap for the next two quarters. The board may admit an archetype with one sponsor when no archetype offers the interface the product needs; it is retired or merged if no second product adopts it within two catalog reviews.
2. **Distinct:** it differs from every existing archetype in task or interface, or the evidence step shows that the nearest archetype's suite would select the wrong model for it.
3. **Measurable:** every requirement used to select models has a metric and a threshold, and the sponsors can supply samples that two people label consistently.
4. **Feasible:** at least one open-weight model, hosted or candidate, reaches the bar on the seed samples.
5. **Affordable:** expected volume justifies the hosting cost, or an existing deployment can serve the archetype at its target cost per request.
6. **Governable:** data governance has set a maximum data classification, and Security has reviewed the risk profile.
7. **Owned:** a named archetype owner, and sponsors committed to supplying test data and piloting the first model.

## Step-by-step flow

Each gate leaves a record in the archetype's evidence path. A failed A1 or A2, or a rejection at A3, closes the proposal with its reason; a suite that fails A4 is fixed and checked again.

### Step 1: Proposal and fit check (Gate A1)

- A sponsor submits a proposal with:
  - the need in business terms, and the sponsor products, each with a named owner;
  - acceptance criteria per product, with error types and their severities;
  - five to ten example inputs with expected outputs;
  - languages, the highest data classification each product will send, expected volumes and latency;
  - the nearest archetypes, and why they do not fit.
- The platform lead assigns an archetype owner, who registers the proposal as **Proposed** under a working id and runs the fit check.
- **A1, worth evaluating:** the archetype owner decides. A need that fits an existing archetype, a domain slice, a language extension or a tier is closed here and redirected, at near-zero cost.

### Step 2: Evidence (Gate A2)

This step tests the claim that existing archetypes cannot serve the need, on the sponsors' own samples, with the same method as gap diagnosis.

- **Seed samples:** each sponsor contributes 50 to 100 samples of its real task with expected outputs and error severities, or synthetic samples where real data cannot be used yet. Data governance approves their classification. Two people label a subset independently; if they often disagree, the criteria go back to the sponsor.
- **Nearest archetype:** the ML evaluation team scores the hosted models of the nearest archetype on the seed samples, with its reference prompt and with an optimized prompt written within a fixed time box. It also compares how the nearest archetype's suite ranks those models with how the seed samples rank them.
- **Feasibility:** a reference model and any promising open-weight candidates run on non-sensitive or synthetic samples, as in gap diagnosis.

Read the results against this table from the top; the first row that matches decides. When no archetype offers the interface the need requires, only the feasibility row applies: if an open-weight model meets the bar, the conclusion is a new capability archetype.

| Result | Conclusion | Path |
| --- | --- | --- |
| A hosted model meets the bar through the nearest archetype's aliases, as is or with the optimized prompt | Fits an existing archetype | Sponsors adopt through Stage 10 with the optimized prompt, kept as a reference prompt; seed samples join that suite as domain slices |
| A hosted model meets the quality bar, but only outside the sponsors' latency or cost target | Different trade-off | A new tier in the nearest archetype, with a Stage 0 trigger if no hosted model fits the tier |
| No open-weight model meets the bar, hosted or candidate | Beyond current open models | Redesign the workflow as in gap diagnosis Step 5, and re-test at the next scheduled re-evaluation |
| The nearest archetype's suite ranks the hosted models as the seed samples do | Catalog gap in an existing archetype | Stage 0 trigger for a stronger model or a domain adapter on that archetype; seed samples join its suite |
| The nearest archetype's suite ranks the hosted models differently from the seed samples | The existing suite cannot select models for this need | New archetype: a domain archetype if task and interface match the parent, otherwise a capability archetype |

**A2, archetype needed:** the ML evaluation team signs off the evidence report. Any other conclusion closes the proposal with its path.

### Step 3: Definition and board review (Gate A3)

- The archetype owner drafts the definition with the sponsors (see The archetype definition).
- The ML evaluation team writes the measurement plan: metrics per requirement, slices and their sizes, and the judge model, if any.
- Data governance sets the maximum data classification.
- Security writes the risk profile and the safety additions.
- Legal reviews the licenses of public datasets the suite will add and flags model license restrictions that conflict with the archetype's uses.
- Platform SRE writes the load profile and a capacity and cost estimate, including whether an existing deployment can serve the archetype.
- The board reviews the evidence report, draft definition, risk profile and cost estimate.
- **A3, approved for the catalog:** the board approves (with conditions if needed), rejects or sends back. Status becomes **Approved**.

### Step 4: Build and calibrate (Gate A4)

- **Suite version 1**, owned by the ML evaluation team:
  - public benchmark tasks that measure the capability, where good ones exist;
  - one domain slice per sponsor, grown to 100 to 300 samples;
  - one slice per required language, and a long-context slice at the required context length;
  - metrics for every requirement used to select models, and any judge prompts checked against human labels on a sample;
  - for embedding archetypes, MTEB tasks for the archetype's languages and task types, plus domain retrieval sets.
- **Held-out samples:** part of each slice is never used to write prompts or tune models, so scores stay honest.
- **Reference prompts:** the prompt templates every model is scored with, starting from the optimized prompt in Step 2.
- **Safety additions:** Security's archetype-specific tests, run with the standard safety battery on every model that targets the archetype.
- **Load profile:** prompt and output length distributions and request rates for GuideLLM, from the definition's interface and volumes.
- **Policy:** `g5-<id>` version 1, with at least one criterion for every requirement used to select models, a floor for every sponsor slice and every language so no member's need is averaged away, and hard criteria for critical errors and safety. Criteria against an incumbent apply once a first model is GA.
- **Data and tools:** P0 runs discovery, applies the intake rules, builds the data bundle and merges the lock for `<id>/v1`. A new tool image or host goes through P0 and Security as usual.
- **Calibration:** a P7 run, started on demand, scores the hosted models with suite version 1. The ML evaluation team checks that the suite runs fully offline, that scores spread across models instead of bunching at the floor or ceiling, that the ranking agrees with the seed samples, that judge scores agree with human labels, and that confidence intervals are narrow enough to decide each threshold.
- **A4, ready for onboarding:** the ML evaluation team signs off the suite, policy and calibration report, Security the safety additions, and SRE the load profile. Status becomes **Active**, and P1 accepts the archetype as a target.

### Step 5: First models and publication

- **Publish:** the archetype appears in the catalog and routing guide with its definition, marked as having no model yet, so product teams can plan.
- **Hosted models first:** for each hosted model that calibration puts near or above the bar, the archetype owner requests an extension onboarding (below).
- **Then new candidates:** if no hosted model qualifies, the archetype owner logs a Stage 0 trigger for the new archetype.
- **No incumbent:** the first model is judged against the archetype's requirements alone.
- **Pilots:** with two or more sponsors, two of them pilot at Stage 6, as the process document suggests when an archetype serves several products.
- **Alias:** the first alias (`<id>-default`, or `<id>-v1` for a pinned archetype) is created in `gateway-aliases` with the first model, in dev at Stage 9 and in production at Stage 11.

The process document makes extending an approved model to a new archetype a new onboarding. Because the model digest is unchanged, this framework proposes that the model's Stage 3 and Stage 4 evidence carries over, Legal confirms that the license allows the new uses, and the onboarding runs Stage 1 and Stages 5 to 11 for the new archetype (see Open questions).

### Setting up the starting catalog

The archetypes the platform starts with go through the same steps, with two differences: Step 2 has no hosted models to compare, so it covers seed samples and feasibility only; and the board can approve the whole starting catalog in one session. Proposed ids for the starting list in the process document:

| Archetype | Id | Interface (in → out) | Alias mode |
| --- | --- | --- | --- |
| Q&A / RAG | `rag` | Question and retrieved passages → grounded answer | Floating |
| Summarization | `summarization` | Document or thread → summary | Floating |
| Extraction and classification | `extraction` | Text → JSON valid against a schema | Floating |
| Code assistance | `code` | Code and instructions → code | Floating |
| Agents and tool use | `agents` | Task and tool definitions → tool calls | Floating |
| Embeddings and search | `embeddings` | Text → vectors | Pinned |
| Multilingual | `multilingual` | Text in one language → output in another | Floating |

## The archetype definition

Each archetype has one definition, a versioned file in `onboarding-config`. Publish the catalog from these files, so what product teams read matches what the pipelines enforce.

| Section | Contents |
| --- | --- |
| Identity | Id, name, kind (capability, or domain with its parent), version, status, archetype owner, sponsors |
| Scope | One-sentence summary; tasks in scope; tasks out of scope, each with the archetype or product code that covers it; two to five example use cases |
| Interface | Model type, API, input, output and its schema, required serving features (structured output, tool calling, streaming), typical and p95 prompt and output lengths |
| Requirements | Quality bar, latency, context length, languages, maximum data classification, target cost per request, expected volume |
| Evaluation | Current suite and policy versions; slices and who contributed them; judge model, if any; reference prompts |
| Risk profile | Trifecta legs products in this archetype usually have; what Stage 10 checks first; which uses go to risk and legal review; safety additions; gateway guardrails beyond the platform default |
| Gateway | Alias mode; aliases (`<id>-default` and any tiers with their requirement overrides) |
| Lifecycle | Review cadence; related archetypes (parent, neighbors, successor) |

Member products are not listed in the definition; they come from Stage 10 use-case approvals. A domain archetype starts from its parent's definition and suite and changes only what the domain needs; from then on it is versioned on its own.

### Setting requirement values

| Requirement | How to set it | Measured by |
| --- | --- | --- |
| Quality bar | Translate each sponsor's acceptance criteria into metrics, with error types weighted by severity (critical, moderate, minor) as in gap diagnosis | Domain slices, with public benchmarks as a secondary signal |
| Latency | From how the output is used: interactive work needs time to first token and request latency at p95 under expected concurrency; batch work needs throughput only | GuideLLM sweep with the archetype's load profile |
| Context length | The p95 prompt length in sponsors' traffic, plus headroom. This is the length the model must handle well, not the one it advertises | Long-context slice at that length |
| Languages | Every language a sponsor serves now or has on its roadmap | One slice per language, each with its own floor |
| Maximum data classification | The highest classification any sponsor will send, set by data governance. Every model behind the archetype's aliases must be approved for at least this level, so swapping models never breaks a product's approval | Model card at Stage 8; checked again by P6 when an alias is mapped |
| Target cost per request | From sponsors' business cases and volumes, at the typical prompt and output lengths | Throughput at the latency target, from the GuideLLM sweep |
| Expected volume | Requests per day and peak per minute, per sponsor. Used for capacity, not for selecting models | Capacity plan at Stage 9 |

Set each value at the strictest sponsor's need, unless meeting it would materially raise cost or latency for the other members; in that case, make it a tier. Interface features such as structured output or tool calling are checked functionally in the suite, for example as the share of outputs valid against the schema.

### Alias modes

- **Floating** (the default, for LLM archetypes): the alias follows the archetype's selected model, and the platform replaces models behind it through shadow traffic, canary and cutover at Stage 11. Products do not change.
- **Pinned** (embedding archetypes, and any archetype whose outputs products store and compare later): vectors from different models cannot be mixed in one index, so each model generation gets its own alias, such as `embeddings-v1`. A replacement model, including a quantized variant, gets `embeddings-v2`, and `embeddings-v1` is deprecated with notice so products can re-index before it retires.

### Naming

- **Archetype id:** lowercase and hyphenated, naming the capability (`rag`, `text-to-sql`). Domain archetypes are `<parent>-<domain>` (`rag-legal`). No model, vendor, team or product names. Ids are never reused, including those of closed proposals.
- **Aliases:** `<id>-default`, plus `<id>-<tier>` for tiers (`rag-large`). Pinned archetypes use `<id>-v<n>` (`embeddings-v2`).
- **Suite, policy and slices:** suite `<id>/v<n>`, G5 policy `g5-<id>`, domain slices `<id>-<slice>`.

### Example definition

```yaml
# onboarding-config/archetypes/text-to-sql.yaml (see Worked example)
id: text-to-sql
name: Text to SQL
kind: capability                         # capability | domain
parent: null                             # a domain archetype names its parent here
version: 1
status: Active                           # Proposed | Closed | Approved | Active | Deprecated | Retired
owner: <archetype owner>
sponsors: [<analytics assistant>, <finance reporting>]
scope:
  summary: Write one read-only SQL query that answers a business question over a given schema.
  in:
    - SELECT queries with joins, filters and aggregations
    - a one-sentence explanation of the query
  out:
    - { task: running the query or acting on its results, covered_by: product code }
    - { task: general code generation, covered_by: code }
    - { task: answers from documents, covered_by: rag }
  examples: [<two to five use cases>]
interface:
  model_type: llm
  api: chat-completions
  input: question, schema as DDL, up to five sample rows per table
  output: "JSON {sql, explanation}, valid against a published schema"
  features: [structured-output]
  lengths: { prompt_p50: 6000, prompt_p95: 18000, output_p95: 400 }   # tokens
requirements:
  quality:
    - id: execution-accuracy             # the query returns the same rows as the reference query
      target: ">= 0.85 on each sponsor slice"
    - id: critical-errors                # the query runs but answers a different question
      target: "<= 2%"
  latency: { ttft_p95_ms: 1500, request_p95_ms: 6000, concurrency: 10 }
  context_length: 32768
  languages: [en]
  max_data_classification: confidential
  cost_per_request: <target>
  volume: { requests_per_day: 4000, peak_per_minute: 60 }
evaluation:
  suite: text-to-sql/v1
  policy: g5-text-to-sql@1
  judge_model: none                      # scored by running each query, not by a judge
risk:
  trifecta: { private_data: present, untrusted_content: possible, outbound: products run the query }
  stage10_focus: read-only database credentials scoped to the user's own access
  safety_additions: [destructive statements, injected instructions in comments and sample rows]
gateway:
  alias_mode: floating                   # floating | pinned
  aliases: [text-to-sql-default]
lifecycle:
  review: quarterly                      # with the scheduled re-evaluation
  related: { neighbors: [code, rag] }
```

## Implementation on the platform

Archetypes use the same repositories and pipelines as model onboarding:

| Item | Where it lives | Written by |
| --- | --- | --- |
| Archetype definition | `onboarding-config/archetypes/<id>.yaml` | Archetype owner, by reviewed pull request; a status change cites its gate record |
| Suite definition and reference prompts | `onboarding-config/suites/<id>/v<n>/` | ML evaluation team, by reviewed pull request |
| G5 policy | `onboarding-config/policies/g5-<id>.yaml` | ML evaluation team, by reviewed pull request |
| Data lock and bundle | `onboarding-config/data/<id>/v<n>.lock.yaml` and `eval-data/bundles/<id>/v<n>/` | P0 |
| Seed samples and domain slices | `eval-data/domain/<id>-<slice>/v<n>/` | P5, with the classification set by data governance |
| Gate records: proposal, evidence report, board decision, readiness sign-off | `evidence/archetypes/<id>/<step>/<run-id>/` | P5 |
| Archetypes a model is approved for | `archetypes` in each `model-registry` record | Pipeline service account |
| Aliases | `gateway-aliases` | P6 |

P5 takes archetype records the same way as model records, with its manual approval step assigned to the gate owner: the archetype owner for A1, the ML evaluation team for A2, the board chair for A3, and the ML evaluation team, Security and SRE for A4.

The pipelines enforce the definition:

- **P1** accepts only target archetypes whose definition is **Active**, and screens the candidate against the definition's context length, languages and interface features.
- **P4** runs each target archetype's current suite and policy. When no model on the archetype is GA yet, criteria against an incumbent are skipped and the verdict says so.
- **P6** refuses an alias mapping when the model's registry record does not list the archetype, or its maximum data classification is below the archetype's. It also refuses to change the model behind an existing pinned alias.
- **P7** re-scores every model on an archetype with the current suite version at each scheduled re-evaluation, and runs calibration on demand.
- **Policy check:** a check in `onboarding-config` rejects a policy that leaves a quality, latency, context length, language or cost requirement without a criterion. Data classification is checked by P6, and volume is used for capacity only.

Each criterion names the requirement it checks:

```yaml
# onboarding-config/policies/g5-text-to-sql.yaml (excerpt)
policy: g5-text-to-sql
version: 1
suite: text-to-sql/v1
criteria:
  - id: execution-accuracy-analytics
    requirement: quality.execution-accuracy
    metric: lighteval/text-to-sql-analytics/execution_accuracy
    rule: value >= 0.85                  # one floor per sponsor slice
    hard: true
  - id: execution-accuracy-finance
    requirement: quality.execution-accuracy
    metric: lighteval/text-to-sql-finance/execution_accuracy
    rule: value >= 0.85
    hard: true
  - id: ttft-p95
    requirement: latency.ttft_p95_ms
    metric: guidellm/text-to-sql-profile/ttft_ms_p95
    rule: value <= 1500
```

## Archetype status and lifecycle

| Status | Meaning | Models can target it | Products can use it |
| --- | --- | --- | --- |
| Proposed | In Steps 1 and 2 | No | No |
| Closed | Not created; the reason and the path taken instead are recorded | No | No |
| Approved | Approved by the board; suite being built | No | No |
| Active | Suite and policy in use | Yes | Through its aliases, once a model behind them reaches Dev (in dev) or GA (in production) |
| Deprecated | Successor and sunset date announced | No | Existing members only, while they migrate |
| Retired | Aliases removed; definition, suite and evidence archived | No | No |

### Catalog review

At each scheduled re-evaluation, the archetype owner and the ML evaluation team review every Active archetype:

- **Members and use:** member products, traffic, and Stage 10 requests that only partly fit the scope.
- **Models:** GA models per alias, their results on the current suite version, and open Stage 0 triggers.
- **Suite health:** scores bunching at the ceiling, public benchmarks likely to be in new models' training data, domain slices that no longer look like members' traffic, and judge agreement with human labels.
- **Requirements:** whether production latency and cost meet the targets, and whether members' needs have moved.

The outcome is no change, a new suite version, a definition change, or a proposal to split, merge or deprecate.

### Changing an archetype

| Change | Examples | Approved by | Effect |
| --- | --- | --- | --- |
| New suite version | New or refreshed slices, harder tasks, a new tool or judge version | ML evaluation team, through P0 | Scores compare only within a version; models on the archetype are re-scored on it at the next re-evaluation |
| New policy version | A threshold changes | ML evaluation team, by reviewed pull request | Verdicts name the policy version they used |
| Definition change | A new language or tier, a wider scope, tighter or looser targets | Archetype owner with the ML evaluation team; platform lead for a tier; board if the maximum data classification rises | Models are re-scored. A GA model that falls below a raised bar keeps serving and triggers Stage 0 for a replacement |
| Breaking change | A narrower scope, a different interface, a lower maximum data classification | Never made in place: a split or a successor archetype goes through this framework, with notice to members | Members move through Stage 10 |

Urgent restrictions after a vulnerability or a data risk finding follow the emergency path of the model onboarding process, not this table.

### Split, merge and retire

The board approves each of these on the archetype owner's proposal.

- **Split** when member products' slices keep selecting different best models, or their requirements pull so far apart that one model cannot serve them economically. The existing slices are the evidence, so Step 2 needs no new samples; the new archetypes go through Steps 3 and 4, and members move through Stage 10.
- **Merge** when two archetypes are served by the same models, their suites select the same model at successive reviews, and their requirements overlap. The surviving suite takes the other's slices in a new version, and members of the retiring archetype move to the survivor's aliases through Stage 10 during the notice period.
- **Deprecate** when an archetype has had no member products for two catalog reviews, is replaced by a split or merge, or its need has gone. The notice period, and the successor alias named in the notice, follow the model deprecation rules.
- **Retire** at the sunset date: aliases are removed from `gateway-aliases`, the definition is marked **Retired**, and the suite, policy, data and evidence are kept for audit. A model approved only for a retired archetype is deprecated with it.

## Records produced

| Record | Produced at | Owner |
| --- | --- | --- |
| Proposal | Step 1 | Sponsor product teams |
| Fit check and A1 decision | Step 1 | Archetype owner |
| Evidence report (seed samples, scores, suite comparison, feasibility, conclusion) | Step 2 | ML evaluation team |
| Archetype definition and every later version | Step 3, then lifecycle | Archetype owner |
| Data classification decision | Step 3 | Data governance |
| Risk profile and safety additions | Step 3 | Security |
| Dataset license review | Step 3 and P0 | Legal |
| Load profile, capacity and cost estimate | Step 3 | Platform SRE |
| Board decision record | Step 3 | Model review board |
| Suite, policy, data lock and calibration report | Step 4 | ML evaluation team |
| Readiness sign-off | Step 4 | ML evaluation team, Security, Platform SRE |
| Catalog and routing guide entry | Step 5 | Archetype owner |
| Catalog review notes | Lifecycle | Archetype owner |
| Change, split, merge and deprecation notices | Lifecycle | Archetype owner |

## Worked example: Text to SQL

This example is illustrative. Two product teams, a self-service analytics assistant and a finance reporting tool, fail their Stage 10 domain tests on `code-default`, and gap diagnosis finds a catalog gap for both.

- **Step 1:** the fit check finds a different task (one read-only query from a question and a schema, rather than writing or editing programs) and a different interface (schema-heavy prompts of 6,000 to 18,000 tokens, with JSON output). A1 passes.
- **Step 2:** each team contributes 80 questions with reference queries, over a sandbox copy of its schema filled with synthetic rows. The metric is execution accuracy: the share of questions where the generated query returns the same rows as the reference query. The best hosted model reaches 0.62, or 0.71 with an optimized prompt, against a bar of 0.85. The code assistance suite ranks the hosted models in a different order from the seed samples. On a subset with a synthetic schema, which data governance clears for outside runs, a reference model reaches 0.90 and an open-weight candidate 0.86. A2 passes: the code suite cannot select models for this need, so a capability archetype is needed.
- **Step 3:** the definition is the example above. Data governance sets the maximum classification to confidential, because schemas and sample rows reveal business data. Security's risk profile notes that sample rows and column comments can carry injected instructions, and that products, not the model, run the query. The board approves the archetype on condition that products run queries with read-only credentials scoped to the user's own access, checked at Stage 10.
- **Step 4:** suite `text-to-sql/v1` holds a public text-to-SQL benchmark such as Spider or BIRD after Legal's license review, two sponsor slices of 200 questions each, and a custom LightEval task that runs each query against a pinned sandbox database image built by P0. No judge model is needed. Calibration spreads the hosted models from 0.55 to 0.71, all below the bar. A4 passes and the archetype becomes Active.
- **Step 5:** no hosted model is near the bar, so the archetype owner logs a Stage 0 trigger for the open-weight candidate. It passes G5 against `g5-text-to-sql` version 1 with no incumbent, both sponsors pilot it at Stage 6, and `text-to-sql-default` goes live with the first sponsor's Go Live.

## Timelines

The targets below are suggested starting points; adjust them after the first few archetypes. A3 waits only for the board's fixed cadence.

| Path | Applies to | Steps run | Suggested target |
| --- | --- | --- | --- |
| Closed proposal | A need that fits an existing archetype or a smaller change | Step 1, or Steps 1 and 2 | 1 week at A1, 3 weeks at A2 |
| Domain slice | New content for an existing archetype | Samples, label check, next suite version | 1 to 2 weeks |
| Language extension or tier | A new language or trade-off in an existing archetype | Definition change, suite and policy versions, re-scoring | 2 to 4 weeks |
| New archetype | Domain or capability archetype | Steps 1 to 4 | 6 to 8 weeks, proposal to Active |
| First model | After the archetype is Active | Extension onboarding for a hosted model, or full onboarding for a new candidate | Per the model onboarding process |

## Open questions

- [ ] Is the archetype owner always a platform model owner, and can one person own several archetypes?
- [ ] Is two sponsor products the right minimum, and how far ahead may a roadmap commitment count?
- [ ] Do tiers and language extensions need the board, or is approval by the archetype owner and platform lead enough?
- [ ] May an extension onboarding reuse a model's Stage 3 and Stage 4 evidence for the same digest? If so, add the path to the process document's timelines and fast track table.
- [ ] What notice period applies when an archetype is split, merged or retired?
- [ ] Where are the catalog and routing guide published, and who keeps them current?
