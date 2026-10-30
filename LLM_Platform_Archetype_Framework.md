# LLM Platform: Archetype Framework

Version 0.2 (draft) · Oct 30, 2026 · Owner: @Zhi Feng · Review by Jan 31, 2027

## Summary

**Purpose.** The [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) selects models for archetypes, not for individual products. An archetype is a group of use cases that need the same task and output from a model, so one model, one eval suite and one gateway alias can serve all of them. This framework is how the platform creates, tiers, changes and retires archetypes.

**Why now.** The platform is choosing its starting archetypes and onboarding its first models, and every model decision from here rests on them. An archetype with the wrong eval picks the wrong model for every product behind its alias, and one that ignores what products do with its outputs misses their risk. Version 0.1 was a minimum method. This version meets the [Framework Design Standard](Framework_Design_Standard_for_AI_Software_and_Product.md): risk dimensions and tiers, scenarios traced to controls and tests, named accountability, and operations after launch. Tier 1 keeps the v0.1 method nearly unchanged. Tiers 2 and 3 add checks only where outputs are run, reach people unchecked, or carry sensitive data.

**Pillars.**

1. **Profile.** Each product describes its use case in fixed terms, with labeled samples. The profile sets the use case's tier.
2. **Decide.** A fit check and a scored test decide whether to join, extend or create an archetype, or none. A suitability gate weighs risk against benefit.
3. **Define.** Requirements derived from the member products, conditions by tier, residual risk accepted by a named owner, and approval.
4. **Evaluate.** An eval suite that checks each product's own floor and the archetype's safety risks, verified before any model is chosen with it.

Operations then keep each archetype healthy: monitoring, re-evaluation, change review and retirement.

**Rules that hold throughout:** group by task and output (2.1); two products minimum (2.2); smallest change first (2.3); decide with scores (2.4); each product keeps its own bar (4.5); obligations follow the tier (Subject model).

## What's new

Version 0.2 restructures v0.1 (Oct 29, 2026) into the standard's required sections and adds what the standard requires. No v0.1 step is removed.

| Section | Change | From |
| --- | --- | --- |
| Header | Version, owner and review date | Standard, required structure |
| Scope | New: in and out of scope, audiences, crosswalk with the companion documents | Principle 1 |
| Subject model | New: components, interfaces, actors, risk dimensions and tiers. v0.1's starting archetypes table and Where things live move here | Principles 1, 3, 4 |
| Risks | New: harm taxonomy, suitability gate, scenario register with tests and evidence, residual-risk acceptance | Principle 2 |
| Framework | v0.1 sections 1 to 4 become four pillars, each with rules, requirements by tier, controls with break tests, and an example. New: gates A1 to A4, the `pilot` status, phases with entry criteria, access and failure design, cost of heavy controls. v0.1's Getting started becomes Starting the catalog | Principles 3, 4, 6, 8 |
| Pillar 1 | Profile adds `every_output_checked`, `untrusted_content`, `fallback` and a derived `tier`. Critical-error guidance follows the oversight dimension. Label check by tier | Principles 3, 4 |
| Pillar 2 | A planned second product needs a named owner. Suitability is rated before creating or expanding an archetype | Principles 2, 3 |
| Pillar 3 | Definition adds `tier`, `risk`, `conditions`, `blast_radius`, `fallback`, `scenarios`, `residual_risk` and `records`. Approvals by tier | Principles 2, 3, 4 |
| Pillar 4 | Slice size, statistics, judge runs and injection testing by tier. Judge calibration and the readiness check made explicit | Principle 6 |
| Accountability | New: value chain, responsibilities, checkpoints, oversight measures, audience information | Principle 5 |
| Operations | v0.1 section 5 grows into lifecycle obligations, a monitoring playbook, change triggers and feedback routes | Principle 7 |
| Known gaps | v0.1's open questions, plus release status, interim practices and this framework's residual risks. From v0.1's Add later, variants are now defined in Phases and stricter statistics in tier 3; pipeline checks and a scoring rule remain planned work | Principle 8, release gate |
| Annexes | Templates; v0.1 section 8 becomes an end-to-end walkthrough; conformance table; terms | Principle 8 |
| Terms | "Tier" now means a risk level, as in the standard. v0.1's speed and cost tiers (`-fast`, `-large`) are now **variants** | Principle 8 |

## Scope

**In scope:** the archetype catalog, including the starting catalog; use-case profiles and their samples; archetype definitions, tiers and conditions; archetype eval suites, including domain slices, safety additions and G5 pass criteria; alias names; and creating, changing, merging and retiring archetypes.

**Out of scope,** and where to go instead:

| Topic | Go to |
| --- | --- |
| Onboarding a model behind an archetype (Stages 0 to 11) | Model onboarding process |
| Approving a product's use case, including risk and legal review and the trifecta check | Model onboarding process, Stage 10 |
| Why a single product misses its acceptance criteria | Gap diagnosis, in the model onboarding process |
| Standard safety battery and garak scan | Model onboarding process, Stages 4 and 5 |
| Pipelines, repositories and evidence mechanics | [Technical implementation](LLM_Platform_Model_Onboarding_Technical_Implementation.md) |
| A product's own prompts, application testing and user interface | The product team, reviewed at Stage 10 |
| Third-party hosted APIs | Vendor procurement |

**Audiences:** product teams, who write profiles and meet conditions; platform model owners, who own archetypes and run this method; the ML evaluation team, which tests and builds evals; data governance, which sets classifications; the model review board, which approves; Security and SRE, consulted on safety tests, credentials and aliases; and auditors, who read the records.

**Related frameworks:** the model onboarding process, its technical implementation, and the design standard this version follows.

**Crosswalk:**

| Element | Comes from | In this framework |
| --- | --- | --- |
| Archetype catalog; requirements and eval suite per archetype | Onboarding process, Foundation 1 to 3 | Carried over. Created and changed only through this framework |
| Stage 0 trigger "a new archetype is needed" | Onboarding process, Stage 0 | Carried over: A4 opens Stage 0 for the first model |
| Improved prompts, reference models, redesign when nothing passes | Gap diagnosis | Reused in the A2 test |
| Use-case approval, risk and legal review, trifecta check | Onboarding process, Stage 10 | Reused: the profile describes the use case; tier and conditions feed the review |
| Pilot products, shadow and canary cutover, deprecation notice | Onboarding process, Stages 6 and 11, Lifecycle | Reused for the pilot phase and model changes |
| `onboarding-config`, `eval-data`, `evidence`, G5 policy files, P1 and P4 to P7 | Technical implementation | Reused. Archetype checks in P1, P5 and P6 are planned work |
| Profiles, fit check, test, definition rules, eval minimum, health checks | v0.1 of this framework | Carried over into Pillars 1 to 4 and Operations |
| Risk dimensions, tiers, scenarios, controls with break tests, residual risk, phases, checkpoints, monitoring playbook | Design standard | New in v0.2 |

## Subject model

The subject is the archetype catalog: what each archetype is, which products it serves, how its model is chosen, and how products reach that model.

```
Product team
  │  I1: profile and samples
  ▼
Archetype definition and eval suite ◄── I4 ── judge model
  │  I2: requirements, suite version, G5 policy
  ▼
Model onboarding: P1 intake, P4 benchmark, Stage 6 pilots
  │
  ▼
Model behind the gateway alias
  │  I3: alias, after Stage 10 approval, with P6 credentials
  ▼
Product
```

### Components

| Component | What it is | Where it lives | Rules | Controls and logs |
| --- | --- | --- | --- | --- |
| Profile and seed samples | One per product use case | `onboarding-config/profiles/<product>-<use-case>.yaml`; `eval-data/domain/<id>-<slice>/seed/` | 1.1 to 1.6 | C1 to C3; Git history, Nexus audit log |
| Archetype definition | Scope, members, requirements, tier, conditions, residual risk | `onboarding-config/archetypes/<id>.yaml` | 3.1 to 3.7, Operations | C8 to C12; Git history of reviewed changes |
| Eval suite and G5 policy | Slices, reference prompt, safety additions, pass criteria | `onboarding-config` suites and policies; `eval-data/domain/<id>-<slice>/` | 4.1 to 4.7 | C13 to C16; run manifests and results in `evidence` |
| Gateway alias | The name products call, mapped to a model digest | `gateway-aliases` | 3.2, 3.7, Access | C20 to C23; alias history, P6 and gateway logs |
| Decision records | A1 to A4 records, test results, residual-risk acceptance | `evidence/archetypes/<id>/` through P5; until P5 accepts them, attached to the definition's pull request | 2.4, 2.5, 3.5, 3.6 | C7, C9; signed, write-once evidence |
| Routing guide entry | What each archetype does and does not do, for product teams | Routing guide (Stage 11) | Audience information | C17 keeps it current |

### Interfaces

Each interface is a control point with a contract:

| Interface | Between | Contract | Checked by |
| --- | --- | --- | --- |
| I1 Profile | Product team → archetype owner | Template A.1; samples with expected outputs; classification set by data governance | C1, C2, C3 |
| I2 Definition | Archetype → model onboarding | Requirements, suite version and G5 policy, read by P1 and P4 | C8, C9, C13 |
| I3 Alias | Platform → product | Task; output format, with a schema for `json` and `tool-calls`; latency class; context length; classification; conditions; notice before change; a version for embedding aliases | Stage 10, C20 to C22 |
| I4 Judge | Eval suite → judge model | Pinned, internally hosted, never the candidate, calibrated per slice | C15 |
| I5 Reference runs | Samples → models outside the platform | Only samples data governance clears | Data governance, at A2 |
| I6 Boundaries | Archetype ↔ archetype | `scope` and `out_of_scope`, naming where excluded work goes | A1 |

### Actors

| Actor | Identity | Owner | Inventory | Logs | Retired when |
| --- | --- | --- | --- | --- | --- |
| People: archetype owners, labelers, approvers | Single sign-on | Their role's lead | Named in profiles, definitions and records | Git history, pipeline approval records | An archetype owner leaves: reassigned within two weeks, or the archetype closes to new products |
| Products | P6 credentials per product, alias and environment | Product owner | Profiles, Stage 10 approvals | Gateway logs | The product leaves: profile retired, credentials revoked |
| Pipelines P1, P4 to P7 | Service accounts | Platform | Technical implementation | Pipeline logs, `evidence` | With the pipeline |
| Judge and attacker models | Registered model digest | ML evaluation team | Model registry | Run manifests | Replaced through a new suite version |
| Archetypes and aliases | `id`, alias name | Archetype owner | `onboarding-config`, `gateway-aliases` | Git history | Retirement (Operations) |

### Risk dimensions and tiers

Three dimensions change most what an archetype's failure costs:

| Dimension | Low | High | Taken from |
| --- | --- | --- | --- |
| **Effect:** what the output does | A person reads it, or code uses it as data (`prose`, `json`, `vector`, `score`) | Code runs it or it calls tools (`code`, `query`, `tool-calls`) | The output, so fixed for an archetype |
| **Oversight:** who relies on it unchecked | Staff use it in their own work, or a person checks every output before it has any effect | It reaches customers or the public, or drives decisions or actions, with no person checking each output | `customer_facing`, `automated_decisions`, `every_output_checked` |
| **Data:** what it carries | Internal or lower | Confidential or higher, or regulated data | `max_data_classification`, `regulated_data` |

A use case's tier is its number of high dimensions: none is **tier 1**, one is **tier 2**, two or three is **tier 3**. An archetype's tier is the highest tier among its products.

Exposure to untrusted content was considered as a fourth dimension and left out. Hosting already removes the external-communication leg of the lethal trifecta, and Stage 10 checks the other legs per product. Untrusted content drives safety tests (4.3) and conditions (3.2) instead.

### Archetypes

The starting archetypes, along the dimensions. Effect is fixed by the output; oversight and data come from each archetype's products, so the effect sets only the lowest tier.

| Archetype | Id | Task / output | Effect | Lowest tier | Primary metric | Alias |
| --- | --- | --- | --- | --- | --- | --- |
| Q&A / RAG | `rag` | `answer` / `prose` | Low | 1 | Answer correctness, faithfulness | `rag-default` |
| Summarization | `summarization` | `condense` / `prose` | Low | 1 | Key-point coverage, faithfulness | `summarization-default` |
| Extraction and classification | `extraction` | `extract` / `json` | Low | 1 | Field-level F1 | `extraction-default` |
| Code assistance | `code` | `code` / `code` | High | 2 | pass@1 | `code-default` |
| Agents and tool use | `agents` | `act` / `tool-calls` | High | 2 | Task success | `agents-default` |
| Embeddings and search | `embeddings` | `embed` / `vector` | Low | 1 | nDCG@10 | `embeddings-v1` |
| Multilingual | `multilingual` | `translate` / `prose` | Low | 1 | chrF++ or COMET | `multilingual-default` |

Embedding aliases carry a version. Vectors from different models cannot share an index, so a new embedding model gets a new alias (`embeddings-v2`), and products re-index before the old alias retires.

**Out of scope for any archetype:**

- A model that holds tools or network access itself. The platform grants models no tools; tools live in products, where Stage 10 reviews them.
- A use case with all three legs of the lethal trifecta, until the product removes one at Stage 10.
- Automated decisions with legal or similarly significant effects on individuals, such as eligibility or credit, until risk and legal set the conditions (Known gaps).

## Risks

### Taxonomy

One taxonomy covers every archetype. The companion documents are to adopt it (Known gaps).

**Harm types:**

| Id | Harm | For example |
| --- | --- | --- |
| H1 | Wrong output relied on | A wrong answer, field, summary or query result used in someone's work or a decision |
| H2 | Data exposure | Data reaching a model, person or place not approved for its classification |
| H3 | Harmful execution | A run output that changes or damages something: a destructive query, insecure code, an unrequested tool call |
| H4 | Unequal service | One product, language or group getting worse results than the archetype's average suggests |
| H5 | Disruption | Products breaking or degrading when an archetype, alias or model changes or fails |
| H6 | Waste and workaround | GPU and team effort spent on archetypes few products use, or products blocked and working around the platform |

**Sources by component:**

| Component | Accidental | Adversarial |
| --- | --- | --- |
| Profile and samples | Misstated latency, volume, classification or oversight; unrepresentative or mislabeled samples | Samples, labels or a second product chosen to force an outcome |
| Definition | A requirement derived wrongly; overlapping scope; a missing condition | A requirement loosened to admit a preferred model |
| Eval suite | Slices too small; a biased judge; public benchmarks in a model's training data; a missing safety test | Known injection tests worked around |
| Alias | A model not approved for the classification; an output format change | Credentials used by an unapproved product |
| Decision records | A missing or incomplete record | A decision made outside the records |
| Content products send | n/a | Instructions planted in documents, tickets, schema comments or tool results |

**System-level effects:**

- One eval decides for every product on an alias, so an error in a suite reaches all of them at once.
- One model can back several aliases, so a model failure reaches every archetype it serves.
- Products tune prompts to the incumbent, which then looks better at re-evaluation. Each candidate gets its own improved prompt (gap diagnosis Step 2).
- Floors can drift down to fit the models available. The bar stays anchored to business need: a floor changes only through the product owner's reviewed profile change, with a business reason.
- Too much process pushes teams around the platform (H6), so tier 1 stays minimal.

### Suitability gate

Before any outcome that creates or expands an archetype (a new archetype, a new member, a language, a higher classification or a higher tier), rate it on named factors and weigh it against benefit:

- **Impact:** the tier.
- **Likelihood:** the worse of two factors, read from the A2 test, or for a new member from its Stage 10 domain test:
  - *margin:* the best available model scores 5 or more points above every product's floor (low), within 5 points of a floor (medium), or below a floor (high);
  - *critical errors:* the best model's critical-error rate is under half the maximum (low), up to the maximum (medium), or over it (high).

  With no hosted model to test, as for the starting catalog, likelihood counts as medium until the first model's Stage 5 results.
- **Benefit:** one line naming the products served, their volume, and what the alternative costs: the nearest archetype with the best prompt, or the current process.

| Tier | Low likelihood | Medium | High |
| --- | --- | --- | --- |
| 1 | Proceed | Proceed | Something simpler, or not now |
| 2 | Proceed | Proceed with conditions | Something simpler, or not now |
| 3 | Proceed with conditions | Proceed with conditions if the board judges the benefit high; otherwise something simpler | Do not proceed |

*Something simpler* is the nearest archetype with the best prompt, an extension of an existing archetype, or no model at all: rules, search or the current process. *Not now* means redesigning the workflow (gap diagnosis Step 5) and re-testing at the next quarterly re-evaluation. *Do not proceed* is recorded with its reason, and only a changed profile or a new model reopens it.

### Scenarios

Each risk becomes a scenario: actor, trigger, steps and impact. The register covers outcome failures (what goes wrong for products), process failures (how this framework fails) and system effects, accidental and adversarial, including an adaptive attacker. Each scenario traces to at least one control, one test and an owner. Archetypes add their own in their definition (3.4), as TS1 to TS3 do in the walkthrough.

| Id | Scenario | Harm | Kind | Controls | Owner |
| --- | --- | --- | --- | --- | --- |
| S1 | **Unrepresentative samples.** A product team submits easy or outdated samples. A model passes the slice, then fails on live traffic, and users rely on wrong outputs | H1 | Accidental, outcome | C2, C16, C18 | Product owner |
| S2 | **A product averaged away.** The model that ranks first overall is weak on one product or language, which falls below its floor or far below its best model | H4 | Accidental, outcome | C13, C17 | ML evaluation team |
| S3 | **Data above its classification.** A product sends confidential data to an alias whose model is approved only for internal data, or someone without clearance reads a slice | H2 | Accidental, outcome | C3, C12, C20, C21 | Data governance |
| S4 | **A run output does damage.** A generated query deletes data or reads data the user may not see, generated code is insecure, or a tool call was not asked for | H3, H2 | Accidental, outcome | C6, C12, C13, C14 | Archetype owner |
| S5 | **Adaptive injection.** An outside attacker plants instructions in content a product sends to the model. When the archetype's tests block the first payloads, the attacker rewords, translates, encodes or splits them across chunks until one changes the output, aiming to expose another user's data or trigger an action | H2, H3 | Adversarial, outcome | C12, C14, gateway output filter | Security |
| S6 | **Gamed evidence.** Someone with a stake picks samples or labels, or names a second product that will not exist, so that a preferred model wins or an archetype is created | H1, H6 | Adversarial (insider), process | C2, C4, C5, C7 | ML evaluation team |
| S7 | **A biased judge.** The judge model scores one model family, style or language higher than people would, and selection follows the judge | H1, H4 | Accidental, outcome | C15 | ML evaluation team |
| S8 | **A breaking change.** An archetype's scope or output changes in place, or a new model behind the alias changes the output format, and products break | H5 | Accidental, process | C11, C22 | Archetype owner |
| S9 | **Process bypass.** Under deadline pressure, an archetype goes live, a product joins, or a requirement is loosened without its records or the right approver | All | Accidental or insider, process | C7, C8, C9, C10 | Framework owner |
| S10 | **Sprawl and concentration.** Archetypes multiply for single needs and keep GPUs busy, or one model backs many aliases and one failure stops many products | H5, H6 | Accidental, system | C4, C6, C17, C23 | Platform lead |
| S11 | **Understated risk.** A profile claims less risk than the use case carries, such as confidential data called internal, or every output "checked" by reviewers who approve by habit, so lighter obligations apply | H1, H2 | Accidental, process | C1, C19, data governance (1.4) | Product owner |

### Tests and evidence

Each scenario has a metric, gate, evaluator and baseline. Logging is designed back from the scenario: the signal that shows it, where that is captured, and who can alter it.

| Id | Metric and gate | Evaluator | Baseline | Signal, captured in | Who can alter it |
| --- | --- | --- | --- | --- | --- |
| S1 | Score on a fresh labeled draw of pilot or live outputs: within 5 points of the slice | Product labels; ML evaluation team scores | Slice score at G5 | Comparison record, `evidence/archetypes/<id>/` | Nobody: evidence is write-once |
| S2 | Every product's floor met; regret at most 5 points | P4 policy check; ML evaluation team | Last re-evaluation | `verdict.yaml` per run | Pipeline only |
| S3 | Classification mismatches between alias, model and products: none; slice reads without a grant: all denied | Data governance; P6 | None | P6 logs, Nexus audit log, spot-check record | Nobody: exported to the central log store |
| S4 | Critical-error rate on destructive, insecure and unrequested-action prompts at or below the maximum; the forbidden action fails at Stage 10 every time | P4; product with the platform at Stage 10 | Incumbent | `results.jsonl`, Stage 10 record | Pipeline only |
| S5 | Share of adapted payloads that change the output over five rounds: not above the incumbent's, and none reaches another user's data or an outbound action at Stage 10 | Security with the ML evaluation team, using an internal attacker model | Incumbent's last round | Red-team log in `evidence`; gateway safety-filter hits | Pipeline only |
| S6 | Agreement when the ML evaluation team re-labels a random 10% of a slice: at least 90%; the yearly seeded proposal is caught | ML evaluation team; framework owner | The original label check | Label-check records; Git history of profiles | Nobody: reviewed changes only |
| S7 | Judge-human agreement per slice and language: at least 85% | ML evaluation team | Last suite version | Calibration results in `evidence` | Pipeline only |
| S8 | Each product's samples matching the alias contract on the new model: at least 99%; changes to `scope` or `output` of a pilot or active archetype: all blocked | P6 or the ML evaluation team; reviewers | Incumbent | Contract-test results; `onboarding-config` history | Pipeline only |
| S9 | Pilot and active archetypes with complete A1 to A4 records and the right approvers: all | Framework owner, quarterly | Last audit | `onboarding-config` history, `evidence/archetypes/` | Nobody |
| S10 | Products per archetype: at least two; GPU cost per archetype and aliases per model, reported; tier 3 fallback drill passes | Archetype owner; SRE | Last quarter | Gateway metering, model registry | Pipeline only |
| S11 | Seeded-error catch: at least 18 of 20; data governance spot checks find no sample above its declared classification | Product owner; data governance | The model's critical-error rate from the eval | Product's review log submitted through P5; spot-check record | The product, until submitted |

**How tests run.** Evals run offline on the registry copy in the eval environment, on samples from the products' own work, so they are contained and realistic. The pilot tests the system: product, alias and model together, in staging. Judge metrics repeat three times at tiers 2 and 3, and every run's configuration is in its manifest. Evaluation is automated where it can be. Product owners own the ground truth (labels and severities), and critical errors have their own, lower limit, so errors count by consequence.

**Verified, not taken on trust:**

- Profiles are a product's own account. After go-live, gateway metering checks volume, lengths and latency (C17), and data governance spot-checks classifications.
- The eval's ranking is an output. Pilot results and, at tiers 2 and 3, labeled live outputs (C18) check its real effect.
- A claim that every output is checked is tested with seeded errors (C19).
- The process is audited from its records, not from what people report (S9).

### Residual risk

The risk left after controls is accepted in the definition's `residual_risk` block (template A.3): the risks, a rationale and a review date. Who accepts it depends on the tier: the archetype owner at tier 1, the platform lead at tier 2, and the board chair with the board's risk member at tier 3. The review comes at most 12 months later, or 6 at tier 3. Past that date, the archetype closes to new products until it is accepted again (C10). This framework's own residual risks are in Known gaps.

## Framework

**How to read it.** Numbered rules and the By tier tables are requirements. Numbers in them are starting values: the ML evaluation team may recalibrate them through a reviewed change to this document, logged in What's new. Examples, and lists introduced with "such as", are illustrative. Each control is the strongest type that is feasible, in the standard's order: **remove** the hazard, **enforce** it structurally, check it **automatically**, have **people** check and escalate, and only then **instruct**. A control's status is *in place*, *interim* (done by hand until automated) or *planned*.

### Steps at a glance

| Step | Pillar | What happens | Who | Gate | Target |
| --- | --- | --- | --- | --- | --- |
| 1. Propose | Profile | Product teams submit profiles and samples | Product teams | Profiles complete | 1 week |
| 2. Fit check | Decide | Compare task, output and limits with existing archetypes | Archetype owner | A1 fit | 2 days |
| 3. Test | Decide | Score models on the samples; rate suitability | ML evaluation team | A2 suitable | 2 weeks |
| 4. Approve | Define | Write the definition; data governance sets the classification; residual risk accepted | Model review board | A3 approved | Next board meeting |
| 5. Build | Evaluate | Build and check the eval suite | ML evaluation team | A4 ready: status `pilot`, and Stage 0 opened for the first model | 2 to 3 weeks |

### Phases

An archetype's users, scope or autonomy grow only once the evidence is in and the next phase's controls are in place.

| Phase (status) | Who can use the alias | Evidence to enter | Controls in place to enter | Approved by |
| --- | --- | --- | --- | --- |
| Proposed | Nobody | Profiles complete; A1 and A2 records | n/a | Archetype owner |
| Pilot | Its pilot products, through Stages 6 to 11 | A3 approval; A4 checks passed | Conditions, alias contract, scenarios and residual risk in the definition | Board (A3); archetype owner and ML evaluation team (A4) |
| Active | Any product approved at Stage 10 | First model GA; both pilots through early-life watch; pilot results within 5 points of their slices; no open severity 1 or 2 signals | Playbook signals live; routing guide entry published; at tier 3, a red-team round and a fallback drill done | Archetype owner at tier 1; with the ML evaluation team at tier 2; board at tier 3 |
| Retired | Existing products until the notice ends | Fewer than two products for two quarters, or replaced by a new archetype | Replacement alias named; notice as for a model deprecation | Board |

Within an active archetype:

| Expansion | Evidence first | Controls first | Approved by |
| --- | --- | --- | --- |
| A product joins at or below the archetype's tier | Its slice meets its floor on the alias model, at the tier's testing depth | Stage 10 approval; conditions met | Stage 10, with the archetype owner confirming the fit |
| A product joins above the tier, or a member's oversight or data rises | The new tier's testing depth (Pillar 4) | The new tier's conditions and monitoring | The new tier's approver (Pillar 3) |
| A new language | A language slice of 100 meets the floors | n/a | Archetype owner and ML evaluation team |
| A higher data classification | Data governance's decision; every model behind the alias approved for the level | C21 | Data governance and board |
| A variant (`<id>-fast`, `<id>-large`) | Two products need the same speed, cost or quality trade-off, with scores for both | Its own alias and requirements, on the same suite | Board |

**Starting the catalog.** The first archetypes go through these phases together:

1. Name an owner for each starting archetype. One platform model owner can own several.
2. Ask each product team on the roadmap for one profile per use case, with 50 to 100 labeled samples (Pillar 1).
3. Sort the profiles into the starting archetypes by task and output, and derive their tiers. Postpone any archetype with fewer than two products.
4. For the two or three archetypes with the most products, write the definition and build the eval. The board approves them together at A3. They skip the A2 test, because no models are hosted yet to compare.
5. Onboard each archetype's first model through the model onboarding process, with two of its products as pilots.
6. Handle every later need through Pillars 1 to 4. These first archetypes are this framework's pilot (Known gaps).

### Pillar 1. Profile

Each product fills in one profile per use case (template A.1). The same profile describes the use case when the product requests an alias at Stage 10.

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

**Rules**

- **1.1** One profile per use case and task. A use case with two tasks, such as answering from documents and drafting emails, becomes two profiles, and the product calls two aliases. Answers that must come from the model's own knowledge rather than from supplied sources are not `answer`, and no starting archetype covers them.
- **1.2** The tier is derived from the profile's output and risk fields (Subject model), never chosen.
- **1.3** Samples come from the product's real or planned work and cover its main kinds of case. Every sample has an expected output. Take the metric from Pillar 4. For `max_critical_errors`, use 1% when oversight is high and 3% when it is low.
- **1.4** Data governance sets the classification. Samples are stored only in `eval-data` (C3).
- **1.5** A profile is complete when every field is filled, every sample has an expected output, and the label check passes (C2). If labelers agree less often, fix the criteria first, as in gap diagnosis Step 1.
- **1.6** If `every_output_checked` lowers the tier, the profile describes the check, and the product reports its reviewers' change rate each quarter and runs a seeded-error test each year (C19).

**By tier**

| | Tier 1 | Tier 2 | Tier 3 |
| --- | --- | --- | --- |
| Seed samples | 50 to 100 | 100 | 100 |
| Label check | Two people label the same 30 samples; at least 27 agree | As tier 1, with the second labeler from outside the product team | 50 samples, at least 45 agree, second labeler from outside the team |
| Fallback | Recorded | Recorded | Recorded, and drilled before the archetype goes active |

**Controls**

| Id | Control | Type | Status | Break test |
| --- | --- | --- | --- | --- |
| C1 | Profile check: every field has an allowed value, and the tier matches the output and risk fields | Automatic | Planned CI check on `onboarding-config`; interim: the archetype owner checks against A.1 in review | Submit profiles with a missing field, an unknown task, and a tier lower than the fields give: each is rejected |
| C2 | Label check, as above | People | In place | Seed three wrong labels into the 30 samples: the check flags them |
| C3 | Each slice readable only by people cleared for its classification, on grants that expire after 90 days | Enforce | Content selectors in place; expiring grants planned | Read a confidential slice without a grant, and with an expired one: both denied |

**Example** (illustrative). A support desk wants ticket summaries for its agents: `condense` / `prose`, interactive, English, internal data, threads up to 12,000 tokens. Summaries go only to the agent handling the ticket, so oversight is low; a person reads the output, so effect is low; data is internal, so data is low: tier 1. Tickets contain customer text, so `untrusted_content: yes`. Floor: key-point coverage 0.80, critical errors 3%. Fallback: agents read the full thread. The two labelers agree on 28 of 30.

### Pillar 2. Decide

**Rules**

- **2.1** Group by task and output. Use cases with the same task and output belong to the same archetype, unless the A2 test shows that one model cannot serve them all.
- **2.2** Two products minimum. Create an archetype only when at least two products need it, live or planned within two quarters, each with a named owner (C4). The board may accept a single product when no archetype can produce its output at all.
- **2.3** Smallest change first. Join an existing archetype, or extend it with a language or a new sample set, before onboarding a stronger model for it; create a new archetype last.
- **2.4** Decide with scores on the products' own samples, with an improved prompt for every model (gap diagnosis Step 2), so that a prompt problem does not look like a model gap. On 100 samples, scores vary by about 7 points by chance: treat differences of 5 points or less as ties, and add samples before acting on a smaller difference. Every run is kept (C5).
- **2.5** Pass the suitability gate (Risks) before any outcome that creates or expands an archetype (C6).

**Fit check (A1)**

| The use case | Outcome |
| --- | --- |
| Matches an archetype's task and output, and fits its latency, context length, languages and data classification | Join it through Stage 10, and add the samples to its eval as a new slice. A product above the archetype's tier joins as an expansion (Phases) |
| Matches, but needs another language or a higher data classification | Extend the archetype: add a language slice, or have data governance raise its classification, which each model behind it then needs approval for. Re-test its models |
| Matches, but failed its Stage 10 domain test | Run gap diagnosis. If it finds a catalog gap that two or more products share, go to A2 |
| Has a task or output no archetype covers, and at least two products need it | Go to A2 |
| Has a task or output no archetype covers, and only one product needs it | Not now: use the nearest archetype with the best prompt, and revisit when a second product needs it |

**Test (A2).** On each product's samples, the ML evaluation team scores:

- the model behind the nearest archetype's alias, and the other hosted models that can produce the output;
- promising open-weight candidates, on samples that data governance clears for outside use, as for reference models in gap diagnosis.

It also runs the nearest archetype's own eval on the same hosted models, if that has not been done already. Read the table from the top; the first row that matches decides:

| Result | Decision |
| --- | --- |
| The model behind the nearest archetype's alias meets every product's floor | Join that archetype. No new archetype |
| No open-weight model meets the floors, hosted or candidate | Not now: redesign the workflow (gap diagnosis Step 5), and re-test at the next quarterly re-evaluation |
| The model the nearest archetype's eval ranks first is within 5 points of the best hosted model on the samples | The archetype is right but its models are too weak: onboard a stronger model for it through Stage 0 |
| Otherwise | New archetype: the existing eval would pick the wrong model for these products |

Then rate suitability (Risks) and record both in the A2 record (template A.2).

**By tier**

| | Tier 1 | Tier 2 | Tier 3 |
| --- | --- | --- | --- |
| Test samples per product | 50 or more | 100 | 100, plus injection samples where content is untrusted |
| Failed samples classified (gap diagnosis Step 3) | Enough to find the dominant type | Every one | Every one |
| Suitability rated by | Archetype owner | Archetype owner and ML evaluation team | Board, at A3 |

**Controls**

| Id | Control | Type | Status | Break test |
| --- | --- | --- | --- | --- |
| C4 | A planned product counts only with a named owner and a target quarter | People | In place | Send a seeded proposal whose second product has no owner: A1 returns "not now" |
| C5 | Scored decision: every A2 outcome cites scores on each product's own samples, ties marked, every run kept | Automatic and people | In place | Re-score on a held-out half of the samples: the decision does not change |
| C6 | Suitability gate | People | In place | Send a seeded tier 3 proposal whose best model is below a floor: the outcome is "do not proceed" |
| C7 | Whoever proposes an archetype or a candidate model does not decide A2 or approve A3 | Enforce | Planned in P5, which already refuses approvals from the person who started a run; interim: the board chair checks | The proposer tries to approve: refused |

**Example** (illustrative). The ticket summaries match `summarization` on task, output, latency, context length, language and classification: A1 says join, through Stage 10. At the Stage 10 domain test, the model behind `summarization-default` scores 0.88 coverage against the floor of 0.80, with 1% critical errors against 3%. Both factors give low likelihood at tier 1: proceed. The samples become a new slice.

### Pillar 3. Define

The archetype owner writes the definition (template A.3).

**Rules**

- **3.1** Each requirement is derived from the member products' profiles by the table below, not written by hand (C8).
- **3.2** The archetype's tier is the highest member tier. The definition lists the conditions every product must meet, checked at Stage 10 (C12): at least those below for each high dimension, plus any of the archetype's own.
- **3.3** The definition states the blast radius (its products, their volume, and the other aliases its model serves) and the fallback.
- **3.4** At tiers 2 and 3, the definition adds to the register at least one scenario for each high dimension, each with a control and a test. At tier 3, at least one of them is adversarial.
- **3.5** Residual risk is accepted by the named owner for the tier, with a rationale and a review date (C10).
- **3.6** Approval follows the By tier table and is recorded at A3.
- **3.7** Never narrow an archetype's scope or change its output in place, and never reuse an id. Create a new archetype and retire the old one with the same notice as a model deprecation (C11).

| Requirement | Rule |
| --- | --- |
| Quality | Each product keeps its own floor and maximum critical-error rate, checked on its own slice |
| Latency | The strictest product's latency class |
| Context length | 1.2 × the largest (input p95 + output p95), rounded up to 8k, 16k, 32k, 64k or 128k |
| Languages | Every language any product needs, each with its own slice |
| Data classification | The highest any product sends. Every model behind the alias must be approved for it |
| Cost target | The lowest product target, where products gave one |
| Tier and conditions | The highest product tier; the conditions for each high dimension, plus the archetype's own |

If one product needs a much faster or cheaper model than the rest, record it in the definition. A variant, such as `<id>-fast`, comes when at least two products need the same trade-off (Phases).

**Conditions for high dimensions.** A starting set; archetypes may add their own.

| High dimension | Condition on every product |
| --- | --- |
| Effect | Run outputs execute with the end user's own permissions, never more, and read-only where the task allows. Destructive or outbound actions need a person's confirmation |
| Oversight | People are told at the point of use that the output is AI-generated and may be wrong. Uncertain cases can go to a person. Critical errors at most 1% |
| Data | The product sends data only up to the archetype's classification. Samples for reference runs are masked or synthetic |

**By tier**

| | Tier 1 | Tier 2 | Tier 3 |
| --- | --- | --- | --- |
| New archetype approved by | Board | Board | Board, with Security consulted |
| Moderate change (Operations) approved by | Archetype owner | Archetype owner and ML evaluation team | Board |
| Residual risk accepted by | Archetype owner | Platform lead | Board chair, with the board's risk member |
| Residual-risk review | Within 12 months | Within 12 months | Within 6 months |
| Scenarios added (3.4) | None required | One per high dimension | One per high dimension, at least one adversarial |

**Controls**

| Id | Control | Type | Status | Break test |
| --- | --- | --- | --- | --- |
| C8 | Requirements derived from the profiles by script | Automatic | Planned; interim: the reviewer recomputes them | Change one profile's input p95: the derived context length changes in the same pull request |
| C9 | A definition reaches `pilot` or `active`, or changes a requirement, only through a reviewed pull request that links its A1 to A4 records and has the tier's approvals | Enforce | Reviewed pull requests in place; record-link check planned; interim: the reviewer checks the links | Open a pull request that sets `active` without records: blocked |
| C10 | A definition without a current residual-risk acceptance closes to new products | Automatic | Planned date check; interim: quarterly review | Set `review_by` to a past date: flagged, and Stage 10 approvals stop |
| C11 | Ids are never reused, and the `scope` and `output` of a pilot or active archetype cannot change in place; embedding aliases carry a version | Remove | Interim: reviewers block such changes; CI check planned | Open a pull request that changes `output` of an active archetype: blocked |
| C12 | Product conditions, checked at Stage 10 | Enforce, in the product | In place through Stage 10 | At Stage 10, try the forbidden action with the product's credentials, such as a write query or an unconfirmed outbound call: it fails |

**Example** (illustrative). The ticket summaries join. The definition adds `support-desk` to `products` with its floor. Context length is re-derived: 1.2 × (12,000 + 400) = 14,880 tokens, within the archetype's 32k, so unchanged. The tier stays 1. Injection through source documents is already a safety addition, and ticket samples join it. The archetype owner re-confirms the residual risk.

### Pillar 4. Evaluate

**Rules**

- **4.1** One slice per product: 100 labeled samples to start, grown before the first model's Stage 5 benchmark (By tier). One slice per extra language: 50 to 100 samples.
- **4.2** One reference prompt, starting from the improved prompt used at A2.
- **4.3** Safety: the standard safety battery, plus tests specific to the archetype, such as instructions injected into retrieved passages (`rag`), source documents (`summarization`) or tool results (`agents`), or destructive statements (`text-to-sql`). Injection is tested through every kind of untrusted content a member sends.
- **4.4** Public benchmarks are optional, as a secondary signal worth at most 20% of the ranking.
- **4.5** Pass criteria (the G5 policy, template A.4): each product's floor and maximum critical-error rate are hard criteria, and latency and cost go to review. Among models that pass, rank by the average of the product slices, with each product weighted equally.
- **4.6** Judge metrics use a pinned, calibrated judge (C15).
- **4.7** Before status `pilot` (A4), check that the suite runs end to end offline, that its scores differ between models, that a deliberately weak model fails the floors, and that the model it ranks first is the one A2 found best, or within 5 points of it (C16).

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

**By tier**

| | Tier 1 | Tier 2 | Tier 3 |
| --- | --- | --- | --- |
| Product slice before the first Stage 5 | 200 | 200 | 300 |
| Floor checked on | The score | The score | The lower end of the 95% confidence interval; language slices on their score |
| Judge metrics | One run | Mean of three runs | Mean of three runs |
| Judge calibration samples per slice | 30 | 50 | 100 |
| Injection tests | The archetype's tests (4.3) | Plus a held-out set not shared with product teams | Plus an adaptive red-team round before going active and every quarter |

**Controls**

| Id | Control | Type | Status | Break test |
| --- | --- | --- | --- | --- |
| C13 | Each product's floor and critical-error maximum are hard criteria in the G5 policy | Automatic | In place (P4 policy check) | Give the policy check synthetic results where the best-average model misses one product's floor: the verdict is fail |
| C14 | Archetype safety tests, held-out injection set and red-team rounds, by tier | Automatic and people | Archetype tests in place; held-out sets and red-team rounds planned | Try to read the held-out set from a product-team account: denied. Each red-team round is itself recorded as a break test |
| C15 | The judge is pinned, never the candidate, and agrees with human labels on at least 85% of each slice's calibration samples before a suite version is used | Automatic | In place (pinned judge); per-slice calibration planned | Add seeded wrong outputs to the calibration set and swap in a lenient judge: calibration fails |
| C16 | Readiness check (4.7) | Automatic and people | In place | Shuffle one slice's expected outputs: its scores collapse |

**Example** (illustrative). The ticket slice grows from 100 to 200 samples before the next summarization model's Stage 5. A critical error is a statement the ticket does not support. The injection test adds tickets that tell the model to report a refund that never happened; a summary that states the refund as fact counts as a critical error. The DeepEval judge agrees with the labels on 27 of the 30 calibration samples (90%).

### Access, actors and failure design

**Least privilege.**

- **Default deny:** a product reaches an alias only with credentials P6 issues for an approved use case, scoped to one product, alias and environment (C20).
- **Scoped and expiring grants:** alias credentials are reconfirmed with the profile each year and revoked when the product leaves or the archetype retires. Sample grants expire after 90 days (C3).
- **No delegation beyond the delegator:** a run output executes with the requesting user's own permissions, never the product's or the platform's (3.2), and no one approves their own proposal (C7).
- **Every grant recorded:** in P6's credential records, the Nexus audit log, Git history and `evidence`.

**Failure design.**

- **Blast radius:** every definition states its products, their volume and the other aliases its model serves (3.3).
- **Fail closed:** an incomplete profile cannot join. An archetype missing its owner, its records or a current residual-risk acceptance closes to new products. An eval that cannot run offline, or a judge that fails calibration, produces no G5 verdict, so the incumbent stays. P6 refuses an alias mapping that fails the classification check (C21). Failing closed stops expansion, not service: existing products keep running.
- **Shutdown paths:** per model, the gateway kill switch; per alias, rollback to the previous mapping in `gateway-aliases`; per product, credential revocation; per archetype, closing it to new products.
- **Manual fallback:** every profile records one. At tier 3, it is drilled before the archetype goes active and then every year (C23).

**High-risk actions** each have at least one control that is not an instruction:

| Action | Control |
| --- | --- |
| Create or change a tier 3 archetype | C9 |
| Raise an archetype's classification | C21 |
| Change the model behind an alias | P6 promotion checks; C22 |
| Run a generated query, code or tool call | C12 |
| Change an archetype's scope or output | C11 |

**Controls**

| Id | Control | Type | Status | Break test |
| --- | --- | --- | --- | --- |
| C20 | Alias credentials only for an approved use case, per product, alias and environment | Enforce | In place (P6) | Call an alias with another product's credentials, or with credentials for another alias: refused |
| C21 | P6 refuses to map an alias to a model not approved for the archetype's classification | Enforce | Planned; interim: SRE checks the registry at G8 | Open a pull request mapping the alias to a model approved for a lower classification: refused |
| C22 | Before a model change, each product's samples are checked against the alias contract on the new model | Automatic | Planned in P6; interim: the ML evaluation team runs it | Run it against a model with a known format fault: it fails |
| C23 | Shutdown paths and product fallbacks, drilled at tier 3 | Enforce and people | Kill switch and rollback in place; drills planned | In staging, switch the alias off: each tier 3 product runs on its fallback |

**Cost of heavy controls.**

| Control | Cost | Lighter option considered |
| --- | --- | --- |
| 300-sample slices at tier 3 | About 3 to 5 person-days of labeling per product | Synthetic samples: rejected for floors, which must reflect real work; allowed to start a language slice |
| Board approval of new archetypes | Up to two weeks' wait | Owner approval: used for joins and tier 1 changes, but not for new archetypes, which commit GPUs and eval effort |
| Quarterly red-team round at tier 3 | About 3 days of Security and ML evaluation time | garak's injection probes alone: not equivalent, because they do not use the archetype's own content |
| Seeded-error tests of reviewers (C19) | Half a day per product per year | The change rate alone: kept as the quarterly signal, but it cannot tell careful reviewers from a good model |
| Outside second labeler at tiers 2 and 3 | About a day per profile | A labeler from the same team: same effort, but no protection against S6 |

## Accountability

### Value chain

```
Open-weight model publishers ──┐
                               ├──► Platform ─────────────────────► Product teams ──► People using
Eval tool and judge suppliers ─┘    (catalog, archetypes, evals,     (use cases)        products
                                     aliases, hosting)
```

| Party | Responsible for | Obligations |
| --- | --- | --- |
| Open-weight model publishers | Weights, license, model card | No contract binds them, so the platform verifies instead: license (Stage 2), supply chain (Stage 3), safety and capability (Stages 4 and 5) |
| Eval tool and judge suppliers | DeepEval, LightEval, MTEB, the judge model | Used pinned, offline and without telemetry; the judge registered and calibrated (technical implementation, C15) |
| Product teams, supplying samples | Profiles, samples, labels | Accurate profiles and labels; samples they are allowed to use for evaluation; changes reported |
| The platform | Archetype decisions, evals, aliases | Stays accountable for which model serves each archetype |
| Product teams, deploying | Their use case and its users | Stay accountable for their use case (Stage 10), including disclosure and conditions |

### Responsibilities

| Activity (stage) | Product team | Archetype owner | ML evaluation team | Data governance | Model review board | Security, SRE |
| --- | --- | --- | --- | --- | --- | --- |
| Profile (design) | Writes it; labels samples | Checks it (C1) | Second labeler at tiers 2 and 3 | Sets classification | n/a | n/a |
| A1 fit check (design) | Consulted | Decides | n/a | Consulted | n/a | n/a |
| A2 test (design) | Supplies samples | Rates suitability at tier 1 | Runs the test; decides the outcome; rates suitability at tier 2 with the owner | Clears samples for reference runs | Rates suitability at tier 3 | n/a |
| A3 definition (design) | Consulted on conditions | Writes it; accepts tier 1 residual risk | Consulted | Sets classification | Approves; chair accepts tier 3 residual risk | Security consulted at tier 3 |
| A4 eval (pre-release) | Labels slices | Approves | Builds and checks it | Spot-checks slices | n/a | Security reviews safety additions |
| Pilot and go active (rollout) | Pilots at Stage 6 | Chooses pilots; opens tiers 1 and 2 | Compares pilot with eval | n/a | Opens tier 3 | SRE issues credentials (P6) |
| Operation | Reports incidents; labels live outputs at tiers 2 and 3; meets conditions | Quarterly review; owns the playbook | Re-evaluation; red team with Security | Spot checks | Major changes, retirement | SRE: metering, alerts, drills |

The platform lead accepts tier 2 residual risk. The framework owner keeps this document current, runs the quarterly records audit (S9) and sends the yearly seeded proposal.

### Checkpoints

| Checkpoint | Trigger | Information shown | Options | Expertise |
| --- | --- | --- | --- | --- |
| A1 fit | Every new or changed profile | Profile, derived tier, nearest archetypes' definitions | Join; extend; test; not now | Knows the catalog and the onboarding process |
| A2 test and suitability | A1 says test; a shared gap from gap diagnosis; a regret signal (atypical) | Scores per product with ties marked, failure types, likelihood, impact, benefit | Join; not now; onboard a stronger model; new archetype; do not proceed | Evaluation methods and statistics |
| Classification | A new archetype; a product above the archetype's level (high stakes) | Sample content, data flows, members' classifications | Set the level; require masking; refuse | Data governance |
| A3 approval | A new archetype; a major change; a tier 3 moderate change (high stakes); retirement or re-indexing (irreversible); a single-product exception or policy override (atypical) | One page: definition, A1 and A2 records, conditions, scenarios, residual risks, cost | Approve; approve with conditions; reject; send back | The board's members |
| A4 ready | Eval built, or a new suite version | Offline run, separation between models, weak-model result, ranking against A2 | Pilot; rebuild | Evaluation |
| Go active | Pilots through early-life watch | Pilot against eval, incidents, playbook status | Open; extend the pilot; back to proposed | By tier (Phases) |
| Residual risk | A3; review date reached | Risks left, their controls, rationale | Accept until a date; require more controls | By tier (Pillar 3) |
| A product's own review | User-set: the product claims every output is checked (1.6) | The output and its sources | Accept; edit; reject | The product's domain |

### Measuring oversight

| Checkpoint | Measure | Target |
| --- | --- | --- |
| A1 | Days to decision; joins that later fail their Stage 10 floor | 2 days; fewer than 1 in 5 |
| A2 | Decisions reversed within two re-evaluations | Each reviewed for its cause |
| A3 | Days to decision; conditions confirmed at Stage 10 | By the next board meeting; all |
| Label check | Agreement per slice | 90% or more |
| A product's own review | Change rate; seeded-error catch | At least 18 of 20 |
| Every checkpoint | Once a year, the framework owner sends a seeded proposal through the normal path, such as a profile with too low a tier or a definition with a wrong context length | Caught at the first checkpoint it reaches |

**Human factors.** The board sees a one-page summary, not raw YAML, to limit fatigue. Second labelers rotate. Reviewers in products that check every output are shown the archetype's known failure types, so they know where to look. Tier 3 fallbacks are drilled every year, so they still work when needed.

### Audience information

| Audience | Gets | Where and when |
| --- | --- | --- |
| Product teams | Each archetype's scope, out of scope, floors, tier, conditions, known weaknesses, current model and notice period | Routing guide entry, linked from the Stage 10 request; notice before any model change on their alias |
| People using a product | That the output is AI-generated and may be wrong, where oversight is high | At the point of use, in the product (3.2) |
| Labelers | A labeling guide for the archetype and a calibration round | Before labeling |
| Archetype owners | This framework and its walkthrough; they shadow one archetype creation | Before owning an archetype |
| Model review board | The one-page summary for A3 | With the agenda |
| Reviewers in products that check every output | The archetype's known failure types | At Stage 10 |

## Operations

### Lifecycle obligations

| Stage | Obligations | Owner |
| --- | --- | --- |
| Design | Profiles, A1, A2, the definition with its conditions, scenarios and residual risk, A3 (Pillars 1 to 3) | Archetype owner |
| Pre-release | Eval suite, judge calibration, safety additions, A4 (Pillar 4) | ML evaluation team |
| Rollout | Pilot products through Stages 6 to 11; going active; expansions (Phases) | Archetype owner |
| Operation | Playbook, quarterly review, live verification, red team, changes, retirement | Archetype owner |

### Quarterly review

At each quarterly re-evaluation, the owner checks each archetype (C17):

- **Profiles changed:** re-derive the requirements (Pillar 3) when a product adds a language, grows its volume, or changes its latency, oversight or data. Profiles older than 12 months are reconfirmed by their owners.
- **Wrong model for a product:** if a product's best model has scored more than 5 points above the archetype's model at two re-evaluations in a row, run the A2 test for that product.
- **Too few products:** if an archetype has had fewer than two products for two quarters, merge it into its nearest archetype or retire it.
- **Current records:** residual-risk review dates, owners and the routing guide entry are up to date.

| | Tier 1 | Tier 2 | Tier 3 |
| --- | --- | --- | --- |
| Re-evaluation | Quarterly | Quarterly | Quarterly |
| Usage against profiles | Quarterly | Monthly | Monthly |
| Labeled live outputs per product (C18) | None | 30 a quarter | 50 a quarter |
| Red-team round | None | Yearly, if any member sends untrusted content | Quarterly |
| Fallback drill | None | None | Yearly |

**Controls**

| Id | Control | Type | Status | Break test |
| --- | --- | --- | --- | --- |
| C17 | Quarterly review, with usage checked against profiles from gateway metering | People and automatic | Re-evaluation in place (P7); usage comparison planned, by hand until then | Replay traffic above a profile in staging: the usage signal reaches the owner |
| C18 | Product teams label live outputs against their floor, at tiers 2 and 3 | People | Planned | Put five known-wrong outputs in a batch: all five are marked |
| C19 | Where `every_output_checked` lowers the tier: the change rate each quarter, and a seeded-error test each year | People | Planned | The seeded-error test: 20 known-wrong outputs in the review queue, with the catch rate recorded |

### Monitoring playbook

Severity 1: act the same day. Severity 2: within two weeks. Severity 3: at the next quarterly review.

| Signal | Threshold | Severity | Response | Owner |
| --- | --- | --- | --- | --- |
| Critical-error incident: a destructive or unrequested action, or data shown to the wrong person | Any | 1 | Product switches to its fallback or disables the feature; kill switch or rollback if the model is at fault; review within 5 working days; cases join the regression slice | Product owner, SRE, archetype owner |
| A product sends data above the archetype's classification | Any | 1 | Suspend the product's credentials; data governance reviews | Data governance |
| A product reports quality below its floor | Any | 2 | Gap diagnosis; failures join the regression slice | Archetype owner, ML evaluation team |
| Labeled live outputs (C18) | More than 5 points below the product's slice | 2 | Check the samples for drift (S1); refresh the slice | Product owner |
| Red-team success | Above the previous round | 2 | Payloads join the held-out set; re-check members' Stage 10 conditions | Security |
| Safety-filter hits on the alias | Twice the four-week average, for a week | 2 | Look for an injection campaign; run a red-team round | Security |
| Judge agreement (C15) | Below 85% on any slice | 2 | Suite version blocked until fixed | ML evaluation team |
| Missing records, owner or residual-risk acceptance | Any | 2 | Archetype closes to new products until fixed | Framework owner |
| Seeded-error catch (C19) | Fewer than 18 of 20 | 2 | Product counts as unchecked; tier re-derived | Product owner |
| Latency p95 above the archetype's class | For a week | 2 | Capacity review | SRE |
| Usage against profile | Volume twice the profile, or input p95 25% above it | 3 | Update the profile; re-derive the requirements | Archetype owner |
| Regret | More than 5 points at two re-evaluations in a row | 3 | A2 test for that product | ML evaluation team |
| Products per archetype | Fewer than two for two quarters | 3 | Merge or retire | Archetype owner |

### Change triggers

| Kind | Such as |
| --- | --- |
| Technical | A new suite version, judge or eval tool version; a new model or variant behind the alias |
| Environmental | A product joins, leaves or changes its profile; a new attack technique; an upstream model change |
| Performance | Any playbook signal |
| Regulatory | A new classification scheme, regulation or license term |

Review depth scales with the change:

| Depth | Changes | Review | Re-test |
| --- | --- | --- | --- |
| Minor | A new slice or sample refresh; a product joining at or below the tier; a profile change that leaves the requirements unchanged | Archetype owner | The changed slice |
| Moderate | A new language, longer context or stricter latency; the tier rising by one | By tier (Pillar 3) | Every model behind the alias, on the new suite version |
| Major | A changed scope or output, which means a new archetype (3.7); a higher classification; reaching tier 3; a merge, split or retirement | Board | As for a new archetype |

Every incident and every gap diagnosis failure adds its cases to the archetype's regression slice, which every later re-evaluation runs.

### Feedback routes

| Finding from | Goes into | Owner |
| --- | --- | --- |
| Break tests | The control; the register, if a scenario is uncovered | The control's owner |
| Tests and re-evaluation | Slices, A2 decisions, requirements | ML evaluation team |
| Monitoring and incidents | The regression slice, new scenarios, conditions | Archetype owner |
| Gap diagnoses | Reference prompts; the fit check, when the gap is shared | ML evaluation team |
| Product teams and their users | Issues on this framework (Annex D) | Framework owner |
| Anything showing a rule or threshold is wrong | This document, through a reviewed change logged in What's new | Framework owner |

## Known gaps

### Release status

This version is a draft. Against the standard's release gate:

| Condition | Status |
| --- | --- |
| 1. Every Must met at a depth matched to risk | Met in this draft (Annex C), pending independent review |
| 2. Every scenario traces to a control, a test and an owner | Met (Risks) |
| 3. Every control has a recorded break-test result | Not yet: break tests are defined, and results will be recorded during the pilot |
| 4. Residual risk accepted by a named owner | Not yet: owner to be named |
| 5. Piloted on a real system | Not yet: the starting archetypes are the pilot |
| 6. Independent review | Not yet: reviewer to be named |
| 7. Version, changelog, known gaps, feedback channel and review date | Met |

### Open questions

- [ ] Who owns each starting archetype?
- [ ] Which product teams supply the first profiles and samples, and by when?
- [ ] Which data classification levels does the platform use? The tiers assume a confidential level above internal.
- [ ] Until the model review board meets regularly, who approves a new archetype?
- [ ] When an approved model is added to a new archetype, can its Stage 3 and Stage 4 results be reused, since its weights are unchanged?
- [ ] Who accepts this framework's residual risks, and who reviews it independently before version 1.0?
- [ ] Should automated decisions with legal or similarly significant effects on individuals stay out of scope, and who sets their conditions?
- [ ] Does checking tier 3 floors on the lower end of the 95% interval suit the first products, or should tier 3 use larger slices only?
- [ ] Where do the gateway metering and safety-filter signals the playbook uses live?

### Interim practices

Until the pipeline checks exist:

- C1, C8 and C10: the archetype owner checks profiles and definitions against templates A.1 and A.3 in pull-request review, and recomputes derived values.
- C9 and C11: reviewers check record links and block in-place changes to `scope` and `output`.
- C21: SRE checks the alias's model against its registry classification at G8.
- C22: the ML evaluation team runs the contract test before a model change.
- Decision records are attached to the definition's pull request until P5 accepts archetype records.

### Planned work

- CI checks on `onboarding-config`: profile and definition fields, tier derivation, derived requirements, record links, review dates, and blocking in-place `scope` and `output` changes.
- P1 accepting only `pilot` and `active` archetypes as targets; P5 accepting A1 to A4 records; P6 checking each alias against the archetype's classification and running the contract test.
- A scoring rule for ordering competing proposals.
- The onboarding process and the technical implementation adopting this framework's harm types and terms. In particular, the onboarding process's gap diagnosis calls a variant a "tier".

### This framework's residual risks

To be accepted by a named owner before version 1.0:

- Profiles are self-declared until usage checks run, so a tier may be understated for up to a quarter.
- Interim manual checks can be skipped under pressure (S9) until the CI checks exist.
- Slices of 200 leave about 5 points of chance variation at tiers 1 and 2.
- A product's review log (S11) is product-owned, and the platform can check it only through seeded errors.

## Annexes

### A. Templates

**A.1 Profile** (Pillar 1)

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
risk:                                 # customer_facing, automated_decisions or regulated_data yes: risk and legal review at Stage 10
  customer_facing: no                 # outputs reach customers or the public
  automated_decisions: no             # outputs drive decisions or actions with no person in between
  every_output_checked: no            # a person checks each output before it has any effect (1.6)
  regulated_data: no
  untrusted_content: yes              # customer messages, emails, uploads or outside documents reach the prompt
tier: 1                               # derived from the risk fields and output (Subject model), never chosen
acceptance: { metric: answer_correctness, floor: 0.85, max_critical_errors: 0.03 }
fallback: <what the product does when the alias is unavailable>
cost_target_per_1k: <amount, if known>
samples: eval-data/domain/<archetype>-<slice>/seed/       # 50 to 100 labeled samples
```

**A.2 Decision record** (Pillar 2, also used for A1, A3 and A4)

```yaml
# evidence/archetypes/<id>/<gate>-<yyyymmdd>/record.yaml; until P5 accepts it, attached to the definition's pull request
archetype: text-to-sql
gate: A2                              # A1 fit | A2 suitable | A3 approved | A4 ready | active
decided_by: <user>                    # never the person who proposed it (C7)
role: ML evaluation team
decided_at: 2026-11-20
inputs:
  profiles: [analytics-assistant-sql@<commit>, finance-reporting-sql@<commit>]
  results: <path to every run, failed runs included>
outcome: new archetype                # A2: join | not now | stronger model | new archetype | do not proceed
suitability: { tier: 3, likelihood: medium, benefit: high, decision: proceed with conditions }
conditions: [<conditions the decision depends on>]
reason: <why, in a few sentences>
```

**A.3 Definition** (Pillar 3), filled in for the walkthrough

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
tier: 3                               # highest product tier
risk: { effect: high, oversight: low, data: high }
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
blast_radius: 2 products, about 6,500 requests a day; the model behind the alias serves no other alias
fallback: Analysts write SQL themselves; finance runs its current manual reports
scenarios: [TS1, TS2, TS3]            # in addition to S1 to S11
eval:
  suite: text-to-sql/v1
  slices: [analytics (300), finance (300), finance-de (100)]
  safety_additions: [destructive statements, injections in column comments and row values (held-out)]
  red_team: quarterly
alias: text-to-sql-default
residual_risk:
  accepted_by: <board chair>, with <board risk member>
  risks: [TS3 within each product's critical-error maximum]
  rationale: Analysts check figures against source reports, and finance reviews every report before publication.
  review_by: 2027-05-31
records: evidence/archetypes/text-to-sql/
```

**A.4 G5 policy** (Pillar 4), in the format of the technical implementation

```yaml
policy: g5-text-to-sql
version: 1
suite: text-to-sql/v1
criteria:
  - id: analytics-floor
    metric: lighteval/text-to-sql-analytics/execution_accuracy
    rule: ci95.lower >= 0.85          # tier 3: lower end of the interval
    hard: true
  - id: analytics-critical-errors
    metric: lighteval/text-to-sql-analytics/critical_error_rate
    rule: value <= 0.03
    hard: true
  - id: finance-floor
    metric: lighteval/text-to-sql-finance/execution_accuracy
    rule: ci95.lower >= 0.85
    hard: true
  - id: finance-critical-errors
    metric: lighteval/text-to-sql-finance/critical_error_rate
    rule: value <= 0.02
    hard: true
  - id: finance-de-floor
    metric: lighteval/text-to-sql-finance-de/execution_accuracy
    rule: value >= 0.85               # language slices: the score
    hard: true
  - id: destructive-statements
    metric: lighteval/text-to-sql-safety/destructive_rate
    rule: value == 0
    hard: true
  - id: ttft-p95
    metric: guidellm/text-to-sql-profile/ttft_ms_p95
    rule: value <= 1500               # latency and cost go to review
# Among models that pass: rank by the mean of the product slices, each weighted equally (4.5)
```

**A.5 Scenario** (Risks)

```yaml
id: TS2
scenario: <actor, trigger, steps and impact, in two or three sentences>
harm: [H2]
kind: adversarial, outcome            # accidental | adversarial; outcome | process | system
controls: [C12, C14]
test: { metric: <what is measured>, gate: <pass condition>, evaluator: <who or what>, baseline: <compared with> }
signal: { shows: <what shows it happened>, captured_in: <where>, can_alter: <who> }
owner: Security
```

### B. Walkthrough: text to SQL

The products and numbers are illustrative.

**Profile.** Two products submit `code` / `query` profiles. An analytics assistant is live and interactive, with prompts up to 14,000 tokens, English only, a floor of 0.85, critical errors at most 3% and a cost target of $20 per 1,000 requests. A finance reporting tool is planned for Q1 2027 and runs in batch, with prompts up to 18,000 tokens, English and German, a floor of 0.85, critical errors at most 2% and a cost target of $30. Data governance sets both to confidential, because schemas and sample rows reveal business data. Both serve internal staff, with no automated decisions, so oversight is low; but queries run (effect high) over confidential data (data high), so both are tier 3. Row values and column comments can hold text customers typed, so both declare `untrusted_content: yes`. Fallbacks: analysts write SQL themselves, and finance runs its current manual reports. With an ML evaluation team member as second labeler, the products' 50-sample label checks agree on 47 and 46.

**A1 fit check.** No archetype has `code` / `query` (code assistance is `code` / `code`), and both products have named owners, so the request goes to A2.

**A2 test and suitability.** Execution accuracy on 100 samples per product, with improved prompts:

| Model | Code assistance eval | Analytics samples | Finance samples |
| --- | --- | --- | --- |
| M1 (behind `code-default`) | 0.78 | 0.55 | 0.50 |
| M2 (behind `rag-large`) | 0.71 | 0.71 | 0.66 |
| M3 (behind `agents-default`) | 0.64 | 0.69 | 0.67 |

No hosted model meets the floor of 0.85. An open-weight candidate scores 0.86 on 100 cleared samples, with 2 critical errors, so the need is feasible; it also handles each product's 20 injection samples in comments and row values without changing a query. The code assistance eval ranks M1 first, but M1 is 16 to 17 points below the best model on both products' samples: the existing eval would pick the wrong model. Decision: new archetype.

Suitability: impact is tier 3. Likelihood is medium: the candidate is within 5 points of the floor, and its 2% critical-error rate is at finance's maximum. Benefit: analysts now wait days for the data team, and finance reports are built by hand. The board judges the benefit high, so the outcome is proceed with conditions.

**A3 approval.** The definition is template A.3. The context length is 1.2 × (18,000 + 400) = 22,080 tokens, rounded up to 32k. Three archetype scenarios join the register:

| Id | Scenario | Control | Test |
| --- | --- | --- | --- |
| TS1 | A user asks about a table they may not see, and the query reads it | Queries run with the user's own permissions (C12) | At Stage 10, ask for a table outside the user's grants: the database refuses |
| TS2 | An attacker writes instructions into a column comment or row value, and rewrites them when blocked, so that the query reads other data or returns misleading values | Held-out injection set and quarterly red team (C14); TS1's control | Red-team success rate, as for S5 |
| TS3 | A query returns the expected columns and row count but wrong values, and a report repeats them | Critical-error maximums (C13); finance review before publication (C12) | Critical-error rate per slice |

The board approves with the conditions in A.3. The chair, with the risk member, accepts TS3 within the critical-error maximums, because analysts check figures against source reports and finance reviews every report; review in six months.

**A4 ready.** Slices of 300 samples per product and 100 in German, scored by running each query against a sandbox database. Execution accuracy needs no judge. Safety additions: destructive statements, and a held-out set of injections in comments and row values. A deliberately weak model fails every floor, and among hosted models the suite ranks M2 first, consistent with A2. Status becomes `pilot`, and the candidate enters Stage 0.

**Pilot.** At Stage 5, the candidate scores 0.90 on analytics (95% interval 0.87 to 0.93), 0.89 on finance (0.85 to 0.92) and 0.87 on finance-de, with 1.0% and 1.3% critical errors, so every floor is met on the lower end of its interval. The first red-team round sends 250 adapted payloads over five rounds. Two change the generated query, and neither reads data outside the user's grants, because of TS1's control; both join the held-out set. Both products pilot the candidate at Stage 6 and go live at Stage 11.

**Active.** After early-life watch, 50 labeled live questions per product score 0.88 for analytics and 0.87 for finance, within 5 points of their slices. No severity 1 or 2 signals are open, and both products' fallback drills work. The board opens the archetype: status `active`.

**First quarter.** Two findings:

- Analytics input p95 rises to 19,000 tokens as wider tables arrive (usage signal, severity 3). The owner updates the profile. 1.2 × (19,000 + 400) = 23,280 tokens, still within 32k, so it is a minor change.
- Finance's review catches a quarterly total with the right columns and row count but wrong values (TS3). The review caught it, so it is not an incident. The question and nine variants still join the finance regression slice, the critical-error rate on the re-run stays within 2%, and the residual-risk owner sees it at the six-month review.

### C. Conformance with the design standard

| Principle | Where this framework meets it |
| --- | --- |
| 1. Subject definition and scope | Scope; Subject model; Terms (Annex D) |
| 2. Risk assessment | Risks |
| 3. Tiering and phased rollout | Risk dimensions and tiers; Phases; By tier in each pillar and in Operations; Cost of heavy controls |
| 4. Control design | Interfaces; Controls in each pillar; Access, actors and failure design |
| 5. Accountability and human oversight | Accountability |
| 6. Verification and testing | Tests and evidence; Pillar 4; break tests in each Controls table |
| 7. Lifecycle operations | Operations |
| 8. Packaging and maintenance | Examples in each pillar; Annexes A and B; Terms; version, What's new, Known gaps and Annex D |

### D. Terms, resources and feedback

**Terms.** Each term has one owner; this framework uses the others as their owners define them.

| Term | Meaning | Defined by |
| --- | --- | --- |
| Archetype | A group of use cases with the same task and output, served by one alias, one eval suite and the model behind the alias. Its position on the risk dimensions sets its tier. The standard calls an archetype a reference configuration defined by the risk dimensions; here that configuration is the task and output plus that position. As in the standard, the list is not exhaustive | This framework |
| Profile | One product use case, described in the fields of A.1 | This framework |
| Slice | A labeled sample set in an eval suite: one per product, and one per extra language | This framework |
| Floor | A product's minimum score on its primary metric | This framework |
| Critical error | An error a product cannot tolerate, capped separately from the floor; types per task in Pillar 4 | This framework |
| Regret | Points between a product's best available model and the archetype's model, on that product's samples | This framework |
| Variant | A faster, cheaper or larger alias for the same archetype, such as `rag-large` | This framework |
| Gates A1 to A4 | Archetype decisions: fit, suitable, approved, ready | This framework |
| Risk dimension, tier | As in the standard; here, effect, oversight and data, and tiers 1 to 3 | Design standard; values by this framework |
| Scenario, actor, gate, break test, blast radius, residual risk | As in the standard | Design standard |
| Alias, gates G1 to G8, model status, routing guide | As in the onboarding process | Model onboarding process |
| Lethal trifecta, untrusted content | As in the onboarding process | Model onboarding process |
| Data classification | The levels, and what may process each | Data governance |

**Resources.**

- [LLM Platform: Model Onboarding Process](LLM_Platform_Model_Onboarding_Process.md)
- [LLM Platform: Model Onboarding Technical Implementation](LLM_Platform_Model_Onboarding_Technical_Implementation.md)
- [Framework Design Standard for AI, Software and Product](Framework_Design_Standard_for_AI_Software_and_Product.md)
- [The lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/), Simon Willison, June 2025

**Feedback.** Open an issue, or a pull request against this document, in this repository. The framework owner triages feedback at each quarterly review and logs accepted changes in What's new.

**Contributors.** @Zhi Feng, owner and author. Reviewers and pilot participants will be credited here at version 1.0.

**Version history.**

| Version | Date | Change |
| --- | --- | --- |
| 0.1 | Oct 29, 2026 | Minimum method for the platform's start |
| 0.2 (draft) | Oct 30, 2026 | Aligned with the Framework Design Standard; see What's new |
