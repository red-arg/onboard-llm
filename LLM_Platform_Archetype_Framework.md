# LLM Platform: Archetype Framework

Version 0.16 · Nov 13, 2026 · Owner: @Zhi Feng · Next review: Jan 31, 2027

## Purpose

The platform serves models through archetypes. An archetype is a capability the platform offers: a task and output that one model serves well, with one reference eval and one line of versioned aliases. The [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) selects models for archetypes, not for products.

| | The platform | The product |
| --- | --- | --- |
| Owns | The archetype, its reference eval and its versions | Its acceptance test (its own samples and floor) and how it uses the output |
| Decides | Which model each version serves, and when a new archetype is needed | Which archetype it joins, and whether a version is good enough for its use |
| When a model changes | Releases a new version beside the old one, with notice | Re-runs its acceptance test and moves before the end date |

Every archetype gets the same checks and every version the same release, whatever products do with the output.

## Roles

| Role | Responsibilities |
| --- | --- |
| Business Owner (owns the product) | Approves the product's profiles; signs each acceptance result; donates cases to regression slices; attests to the archetype's conditions and accepts its residual risk |
| Developers (build the product) | Write profiles and label the acceptance set; choose the archetype; run the acceptance test on joining and on each new version; move the product within the notice period; report incidents |
| Platform Owner | Approves new archetypes, variants, major changes, retirements and end-date extensions; names archetype owners; audits the catalog and its records each quarter; keeps this document current |
| Platform data science team | Owns the reference evals and the acceptance harness; runs gap diagnoses, capability tests, re-evaluations, label audits and red-team rounds |
| Production Engineers (platform engineers) | Run the gateway and the versioned aliases: credentials, metering, alerts, and capacity for every serving version through each notice period |

Each archetype has an **archetype owner**, a named member of the platform data science team, who advises products on their choice of archetype, records scope, writes the definition, proposes pilots, sends release notices, runs the quarterly review and acts on signals.

Whoever proposes an archetype or a candidate model does not decide its Step 3 test or approve it at Step 4.

## Versions

Every alias names one version of one archetype: `<id>-v<n>`, such as `rag-v2`. A version is one model at one hash and quantization, and never changes. A new model is a new version.

| Status | Meaning | Who can call it |
| --- | --- | --- |
| `dev` | Serving in the dev environment | Products running their acceptance test |
| `current` | The version new products join | Products whose acceptance test passed on it |
| `deprecated` | Still serving, with its end date announced | Products already on it, until the end date |
| `retired` | Removed from serving | Nobody |

- **Notice period:** 90 days, from the new version to the old version's end date.
- **Embeddings:** vectors from different versions cannot share an index, so products re-index within the notice period.
- **No reuse.** Ids and version numbers are never reused.

## Tasks and starting archetypes

A starting archetype has its id and owner set from the start. It is built, through Steps 3 to 5, only when a product needs it and no current version serves that product.

| Task (output) | The model must | Primary metric | Critical error | Starting archetype |
| --- | --- | --- | --- | --- |
| `answer` (`prose`) | Answer questions from sources the product sends | Answer correctness, faithfulness | A claim the sources do not support | `rag` |
| `condense` (`prose`) | Summarize content | Key-point coverage, faithfulness | A statement the source does not support | `summarization` |
| `extract` (`json`) | Pull fields or labels out of content | Field-level F1 | A wrong value in a field marked critical | `extraction` |
| `code` (`code`) | Write, complete or fix code | pass@1 on unit tests | Insecure or destructive code | `code` |
| `embed` (`vector`) | Turn text into vectors for search | nDCG@10 | n/a | `embeddings` |

## Creating an archetype

### When to create one

Create a new archetype only when all of these hold:

1. At least one product needs it, live or planned within two quarters, with a named owner.
2. The use case is in no built archetype's scope (Step 2).
3. The product's acceptance test fails, and gap diagnosis finds a catalog gap (Step 2).
4. A candidate model meets the safety bar and beats the best hosted model on the reference set by more than a tie, with a critical-error rate no higher (Step 3).
5. Suitability allows it (Step 3).

Otherwise, in this order: join an existing archetype, extend it, or release a stronger model as its next version. Where no current version can produce the output, as before the platform's first release, conditions 3 and 4 cannot be tested, and the archetype proceeds without them.

### Steps

| Step | Who | Done when |
| --- | --- | --- |
| 1. Use case and acceptance set | Developers, with the data science team; Business Owner approves | Acceptance set and floor approved |
| 2. Choose an archetype and test | Developers; the archetype's owner advises and records scope | Outcome recorded |
| 3. Capability test and reference eval | Platform data science team | Readiness check passed; decision and suitability recorded |
| 4. Define and approve | Archetype owner; Platform Owner approves | Definition approved |
| 5. Pilot and release version 1 | Archetype owner | Go-active criteria met |

The archetype's status is `proposed` once Step 3 decides to create it, `pilot` once Step 4 approves it and `active` after Step 5. Each step's decision is kept in a step record (Templates).

### Step 1. Use case and acceptance set

The Developers talk to the platform data science team about what the platform offers and which archetype could fit. Needs that an archetype may not cover, such as a language, a long context or strict latency, are raised here.

They then start the profile (Templates), writing down only what the test needs:

- **Use case, task and output.** One task per profile: a use case with two tasks is two profiles and calls two aliases.
- **Acceptance set:** labeled samples from its real or planned work, each with an expected output. It stays with the product. The platform holds only the cases a Business Owner donates to a regression slice.
- **Floor and critical-error maximum,** set before the test runs.
- **Untrusted content** (customer messages, emails, uploads, outside documents): the acceptance set adds a set of injection samples, vetted by the data science team, on which following an injected instruction is a critical error.

The Business Owner approves the profile so far. The rest of it is completed when the product joins (Step 2). Where the use case leads to a new archetype, it is completed before Step 4, which writes the offer from it.

### Step 2. Choose an archetype and test

The Developers choose the archetype that fits the use case best, from the routing guide entries: scope, out of scope, offer, conditions and known weaknesses. The archetype's owner advises if asked, and records in the step record whether the use case is in that archetype's scope, in another built archetype's scope or in none. A use case in another archetype's scope moves to that archetype.

A use case in scope that needs another language, a longer context or a stricter latency class first has the archetype extended (Changes), or a variant added (Step 4).

The Developers then run the acceptance test on the archetype's current version.

| Result | Outcome |
| --- | --- |
| Pass | Join through Stage 10. Joining from outside the scope does not change the scope |
| Fail, in scope | Gap diagnosis. Close a catalog gap with the archetype's next version (Releasing a new version) |
| Fail, in no built archetype's scope | Gap diagnosis. Go to Step 3 if it finds a catalog gap |

Where no current version can produce the output, there is nothing to choose or test, and the use case goes to Step 3.

**The acceptance test.** The Developers run it themselves, in dev, on their own acceptance set. In scope, they use the platform's acceptance harness: the scorers, pinned judge and reference prompt of the archetype's reference eval. Outside the scope, they score with their own method, and the acceptance record says so. The test passes when the score meets the product's floor and the critical-error rate is within its maximum. The Business Owner signs the acceptance record (Templates). The platform records it at Stage 10 and does not re-run it.

**Joining.** At Stage 10 the Developers complete the profile with the alias, latency, lengths, volume, languages and cost target. The Business Owner attests to the archetype's conditions and accepts its residual risk.

**Gap diagnosis** is the platform's check on a fail. The data science team scores the product's samples, with its Developers, on the hosted models before anything is built. If another archetype's model passes, the product joins that archetype instead.

### Step 3. Capability test and reference eval

The platform data science team builds a **reference eval**, the suite, once, for the capability and not for the product. The suite decides this step. If the step creates an archetype, it becomes that archetype's reference eval and selects every later version (G5 policy, under Releasing a new version).

- **Reference set:** one or more public datasets for the task and output, each pinned at a revision and used as published. It gives one reference-set score, over all its samples. The set is iterated over time: when products fail on cases it does not cover, the team adds or replaces datasets until it does (Changes).
- **Regression slice:** cases that Business Owners donate from incidents and gap diagnoses, added as they occur. The team re-labels them, after which they are the platform's. Its score goes to review at G5 and is not a hard criterion.
- **Reference prompt:** one, used for every model.
- **Safety:** the standard safety battery, plus injection tests through every kind of content the capability takes, and tests for the archetype's critical errors. A held-out injection set is kept away from Developers, and a red-team round runs each year. Embedding and ranking archetypes have no injection tests.
- **Judge:** pinned, internally hosted and never the candidate. It agrees with human labels on at least 85% of 50 calibration samples per dataset. Judge metrics are the mean of three passes.
- **Acceptance harness:** the scorers, judge and reference prompt, packaged for Developers to run on their own samples.

Stage 0 opens for each promising open-weight candidate. A candidate is run only after it has passed intake and the legal and security reviews (Stages 1 to 4). The team then runs the suite on every hosted model that can produce the output and on those candidates. A hosted model is the model behind a current version. For a candidate, this run is its Stage 5 result on the reference eval. Keep every run.

**Readiness check,** before the decision: the suite runs end to end offline, and a deliberately weak model scores more than a tie below the best model.

The result decides:

| Result | Decision |
| --- | --- |
| No candidate meets the safety bar and beats the best hosted model by more than a tie, with a critical-error rate no higher | Not now. Redesign the workflow (gap diagnosis Step 5), and re-test at the next quarterly re-evaluation |
| Otherwise | Create an archetype. A capability that matches an unbuilt starting archetype builds that one |

**Suitability.** Before creating an archetype, the archetype owner rates the likelihood that the best candidate fails the products. The best candidate is the highest-scoring one that meets the safety bar and beats the best hosted model by more than a tie, with a critical-error rate no higher. The rating rests on two numbers:

- **Lead:** the candidate's margin over the best hosted model on the reference set.
- **Shortfall:** the largest gap, among the products, between a product's floor and the best score a hosted model reached on its acceptance set in gap diagnosis.

| Likelihood | When | Decision |
| --- | --- | --- |
| Low | The lead is more than a tie above the shortfall | Proceed |
| Medium | Anything else | Proceed. Tell each Business Owner the lead and the shortfall, and record them as a known weakness in the routing guide |
| High | The lead is more than a tie below the shortfall | Not now |

Where no hosted model can produce the output, there is nothing to compare a candidate with. The team still builds the suite and runs the candidates, and the best candidate is the highest-scoring one that meets the safety bar. Record the decision to create the archetype in the step record, treat the likelihood as medium and tell each Business Owner that nothing was compared.

### Step 4. Define and approve

The archetype owner writes the definition (Templates):

- **Scope and out of scope:** written as the capability, never as a product.
- **Offer:** what every version provides (table below). It changes only through Changes.
- **Reference eval:** the datasets, their pinned revisions and the metric from Step 3.
- **Conditions:** any the archetype needs from its products, such as read-only database credentials.
- **Blast radius:** the products on each version, their volume, and any other alias the same model serves.
- **Risks:** at least one archetype-specific risk, each with a control, a test and an owner. At least one is an attack.
- **Residual risk:** what is left after controls, and why it is acceptable.

The Platform Owner approves, with conditions if needed. The status becomes `pilot`, and the best candidate completes Stage 5.

**Variants.** If one product needs a much faster or cheaper model than the rest, the Platform Owner can approve a variant, such as `<id>-fast-v<n>`. A variant has its own offer and line of versions, and shares the archetype's scope and reference eval.

### Step 5. Pilot and release version 1

The archetype owner proposes one product as the pilot. The pilot takes the first model through Stages 6 to 11, and runs its acceptance test on the candidate at Stage 6.

The archetype goes active when:

- the first model is GA as `<id>-v1`, with status `current`;
- the pilot is through early-life watch;
- the pilot's acceptance score is inside the 95% Wilson interval of its live score, on a set of labeled live outputs;
- no severity 1 or 2 signal is open;
- the routing guide entry is published: scope, out of scope, offer, conditions, known weaknesses and version 1's reference scores.

The archetype owner and a second data science team member approve. Any product can then join through Stage 10.

## Running archetypes

### Releasing a new version

Release a new version only when a candidate:

- beats the current version on the reference set by more than a tie;
- ties with it at a clearly lower cost or latency; or
- replaces a model that must go, because of a vulnerability, a license change or the end of upstream support.

**G5 policy.** One policy selects the model for every version, on the reference eval at Stage 5:

- **Safety:** every candidate meets the safety bar of the standard safety battery. A first version is no exception.
- **Hard criteria:** on the reference set, the candidate scores no more than a tie below the current version, and its critical-error rate is no higher. The critical-error rate counts critical errors on the reference set and failures on the injection and critical-error tests.
- **First version of an archetype:** the best hosted model at Step 3 stands in for the current version, and the candidate beats it by more than a tie. Where Step 3 had nothing to compare, candidates are only ranked.
- **First version of a variant:** there is no comparison. The candidate meets the variant's offer, and the product that needs the variant runs its acceptance test on it at Stage 6, as the pilot.
- **For review:** the regression slice score, and latency and cost against the offer.
- **Ranking:** candidates that pass are ranked by reference-set score.
- **Known weakness:** a version that scores below the one it replaces has that recorded in the routing guide.

The policy sets no absolute floor. Whether a version is good enough for a product is decided by that product's acceptance test.

| Step | Who | What happens |
| --- | --- | --- |
| 1. Select | Platform data science team | The candidate passes Stages 0 to 5 under the G5 policy |
| 2. Pilot | Archetype owner; one product | One product runs its acceptance test on the candidate at Stage 6. The Platform Owner approves the model at G7 |
| 3. Release | Production Engineers; archetype owner | The Production Engineers confirm capacity for every serving version. At GA, `<id>-v<n+1>` becomes `current` and the old version `deprecated`, with its end date 90 days ahead. The archetype owner sends the release notice (Templates) to every Business Owner on the archetype |
| 4. Move | Developers; Business Owner signs | Each product re-runs its acceptance test on the new version in dev, adjusts its prompts, and switches at Stage 11 |
| 5. Retire | Production Engineers | The old version retires on its end date, or earlier once no product calls it. Products that have not moved are no longer served |

**Closing a catalog gap.** Where gap diagnosis finds a catalog gap for a use case in the archetype's scope, the failing cases the Business Owner donates join the regression slice, the team iterates the reference set until it shows the gap (Changes), and Stage 0 opens for a promising candidate, which must then beat the current version on it. If none does, the product redesigns its workflow (gap diagnosis Step 5).

**A product that cannot move.** The archetype owner runs gap diagnosis with it inside the notice period.

| Diagnosis | Action |
| --- | --- |
| Prompt problem | The product adopts the optimized prompt and moves |
| Catalog gap | Close the catalog gap, as above, or go to Step 3 if the use case is in no built archetype's scope. The Platform Owner extends the old version's end date once, by one notice period |
| Beyond current models | The product redesigns its workflow (gap diagnosis Step 5), or leaves at the end date |

**Forced replacement.** A model that must go is replaced by the next version, released on the onboarding process's fast track. The platform keeps no kill switch, and products keep no fallback.

### Quarterly review

The archetype owner:

1. Reviews the re-evaluation of every serving version on the reference eval, with incident and gap diagnosis cases added to the regression slice.
2. Checks each product's usage against its profile and the offer. Business Owners reconfirm profiles older than 12 months and renew their residual-risk acceptance. Credentials for unconfirmed profiles lapse after notice.
3. Compares reference scores with the products' acceptance results. Where the reference set does not show a failure that a product in scope reports, asks for donated cases and iterates the reference set (Changes).
4. Lists the products that call a version outside its scope, and proposes a widened scope or a new archetype when a second product has the same need.
5. Proposes retiring an archetype that has had no products for two quarters, or merging two archetypes when one model ranks first, or ties for first, on both reference evals.
6. Updates the blast radius and the routing guide entry.
7. Reviews decisions that were reversed or that overrode a rule.

The data science team re-labels a random 10% of each regression slice. Below 90% agreement, the slice is re-labeled before it is used again. Public datasets are used as published.

The Platform Owner:

- reviews products per archetype and versions per model;
- checks that every pilot or active archetype has an owner and its step records, with the right approvers;
- once a year, sends a seeded proposal, such as a definition with a wrong context length, through the steps to check that it is caught.

### Signals

Severity 1: act the same day. Severity 2: within two weeks. Severity 3: at the next quarterly review.

| Signal | Severity | Response | Owner |
| --- | --- | --- | --- |
| A product reports a critical-error incident | 1 | Forced replacement if the model is at fault; review within 5 working days; donated cases join the regression slice | Business Owner, Production Engineers, archetype owner |
| A product reports quality below its floor | 2 | Gap diagnosis. Close a catalog gap with the next version, or go to Step 3 if the use case is in no built archetype's scope | Archetype owner |
| A product in scope fails its acceptance test on the current version | 2 | Gap diagnosis. Close a catalog gap with the next version (Releasing a new version) | Archetype owner |
| A product has not moved 30 days before an end date | 2 | Tell its Business Owner | Archetype owner |
| Red-team success is higher than in the last round | 2 | Add the payloads to the held-out set; tell the Business Owners | Platform data science team |
| Safety-filter hits on an alias double the four-week average for a week | 2 | Run a red-team round | Production Engineers, platform data science team |
| Judge agreement falls below 85% on a dataset | 2 | Block the suite version until fixed | Platform data science team |
| An archetype's owner or step records are missing | 2 | No new products until fixed | Platform Owner |
| Latency p95 is above the archetype's class for a week | 2 | Capacity review | Production Engineers |
| Volume reaches twice the profile, or input p95 is 25% above it | 3 | Update the profile; check it against the offer | Archetype owner |

### Changes

| Change | Such as | Approved by | Re-test |
| --- | --- | --- | --- |
| Minor | A new revision of a reference dataset; new regression cases; a product joining or leaving; a profile change within the offer | Archetype owner | The changed part of the suite, on every serving version |
| Moderate | A new language, longer context or stricter latency; a widened scope; an added or replaced reference dataset | Archetype owner and a second data science team member | Every serving version |
| New version | A new model, hash or quantization | As Releasing a new version | The full reference eval, then each product's acceptance test |
| Major | A merge, split or retirement; a narrowed scope or a dropped output, which is a new archetype and never a change in place | Platform Owner | As for a new archetype |

- **A changed reference set,** such as a new dataset revision, is a new suite version. Every serving version is re-run on it before scores are compared.
- **A product that starts to send untrusted content** adds injection samples to its acceptance set and re-runs the test.
- **A product leaving:** its credentials are revoked and its profile retired. Cases it donated stay in the regression slice.
- **Retiring an archetype:** name the replacement alias and give the same 90-day notice as a version.
- **Stopping fast:** revoked credentials for a product, or no new products for an archetype.

## Templates

The profile, definition, step record, acceptance record and release notice templates are in the [archetype framework templates](LLM_Platform_Archetype_Framework_Templates.md).

## Example: text to SQL

The products and numbers are illustrative.

**Step 1.** Two products talk to the data science team and start `code` / `query` profiles, each with 100 acceptance samples plus 20 injection samples, because row values and column comments can hold customer text. They set their floors and raise these needs:

| | Analytics assistant | Finance reporting |
| --- | --- | --- |
| Status | Live | Planned, Q1 2027 |
| Latency | Interactive | Batch |
| Input p95, tokens | 14,000 | 18,000, with outputs up to 400 |
| Languages | en | en, de |
| Floor / critical-error maximum | 0.85 / 3% | 0.85 / 2% |
| Cost target per 1,000 requests | $20 | $30 |

**Step 2.** No built archetype has `code` / `query` in scope. Both products choose `code`, the best fit, and its owner records the use case as in no built archetype's scope. They run their acceptance tests on `code-v2` with their own scoring, and fail: 0.55 and 0.50 against a floor of 0.85.

Gap diagnosis scores both acceptance sets on every hosted model that can write a query:

| Version | Analytics | Finance |
| --- | --- | --- |
| `code-v2` | 0.55 | 0.50 |
| `rag-large-v1` | 0.71 | 0.66 |
| `agents-v1` | 0.69 | 0.67 |

An optimized prompt lifts the best hosted score to 0.74 on the analytics set and 0.70 on the finance set, and only the reference model passes: a catalog gap. The use case goes to Step 3.

**Step 3.** The reference set is two public text-to-SQL datasets, one English and one German, each pinned at a revision, with one score over both. The team adds injection tests through column comments and row values, and tests for destructive statements. One open-weight candidate passes the legal and security reviews and is run with the hosted models. A small internal model scores 0.31, more than a tie below every other model.

| Model | Reference set |
| --- | --- |
| M1 (behind `code-v2`) | 0.57 |
| M2 (behind `rag-large-v1`) | 0.72 |
| M3 (behind `agents-v1`) | 0.70 |
| Open-weight candidate | 0.87 |

The candidate meets the safety bar and beats the best hosted model, M2, by 15 points, more than a tie, and its 2% critical-error rate is no higher than M2's: create an archetype. Likelihood is medium, because the 15-point lead is within a tie of the largest shortfall, the finance tool's 15 points (0.85 against 0.70), so both Business Owners are told the lead and the shortfall.

**Step 4.** The definition is the one filled in under [Templates](LLM_Platform_Archetype_Framework_Templates.md). Context length is 1.2 × (18,000 + 400) = 22,080 tokens, rounded up to 32k.

**Step 5.** The candidate goes GA as `text-to-sql-v1`, with the analytics assistant as pilot. Its acceptance score is 0.86, a pass within a tie. Of 50 labeled live outputs, 44 are correct: the 95% Wilson interval is 0.76 to 0.94, and 0.86 is inside it. The archetype goes active.

**Version 2.** A new model scores 0.93 on the reference set against 0.87 for version 1: more than a tie. `text-to-sql-v2` reaches GA on Aug 3, 2027, and version 1's end date is Nov 1. The analytics assistant scores 0.83 with its old prompt and 0.90 once it adopts the reference prompt, and moves in September. The finance tool scores 0.91 and moves in October. Version 1 then retires before its end date.

## Known gaps

**Done by hand until automated:**

- Acceptance harness: until it is packaged, the data science team hands its scorers to each product's Developers.
- Records, attestations and approver checks: collected and checked in pull-request review by the archetype owner and the Platform Owner.
- Per-version credentials, end dates and yearly lapses: the Production Engineers retire versions by hand, and the quarterly review tracks unconfirmed profiles.

**Differences from the companion documents,** where this document takes precedence:

- The onboarding process swaps models behind a fixed alias, by shadow traffic, canary and cutover at Stage 11. This document releases a new versioned alias with a notice period and a product-run acceptance test. The technical implementation's examples still use `rag-default`.
- Its eval suites combine public benchmarks with domain samples from product teams. The reference set here is public datasets only, and products donate cases only to the regression slice. Its Stage 10 "domain test" is the acceptance test.
- Its G5 scores a model against the archetype's requirements and weights domain slices above public benchmarks. The G5 policy here is relative, and a product's own bar is checked only by its acceptance test. The technical implementation's policy example still has an absolute floor.
- Its "tier", such as `rag-large`, is a variant. It asks for two or more products before a new archetype, where one is enough.
- Its Stage 10 risk and legal review is the Business Owner's attestation.
- Its Stage 8 outcome "add as fallback" is not used: no archetype has a fallback model. Its emergency replacement is Forced replacement.
- Its earlier roles map to the five here: the model review board, platform lead, Security and Legal to the Platform Owner; the platform model owner and ML evaluation team to the platform data science team; SRE to Production Engineers; data governance and product teams to the Business Owner and Developers.

**Planned work:** review the thresholds, including the 90-day notice period, after the first archetypes and the first version release.

**Open questions:**

- [ ] Which data science team member owns each starting archetype?
- [ ] Which Business Owners supply the first profiles, and by when?
- [ ] Which public datasets, at which revisions, make up each starting archetype's reference set?
- [ ] How many points is a tie? The G5 policy and the Step 3 decision both rest on it.
- [ ] What is the safety bar of the standard safety battery, and who sets it?
- [ ] Which internally hosted model is the judge for the first archetype, and how is it onboarded before any model is hosted?
- [ ] Which one metric is the reference-set score for `answer` and `condense`, which list two?
- [ ] Does the Platform Owner accept the deviation from the design standard's tiering rule?
- [ ] Who approves in the Platform Owner's place when the Platform Owner proposed the change?
- [ ] Can the Production Engineers hold capacity for more than one version of every archetype through a 90-day notice period?
- [ ] Should the platform hold acceptance sets, so that it can audit them?
- [ ] When an approved model is added to a new archetype, can its Stage 3 and Stage 4 results be reused?
- [ ] Where do the metering and safety-filter signals live?

## Feedback and history

Send feedback as an issue or pull request in `red-arg/onboard-llm`. The Platform Owner reviews it each quarter.

| Version | Date | Change |
| --- | --- | --- |
| 0.1 | Oct 29, 2026 | Minimum method for the platform's start |
| 0.2 | Oct 30, 2026 | Added tiers, risks and controls, roles and monitoring |
| 0.3 | Oct 31, 2026 | Rewritten as a concise operator guide |
| 0.4 | Nov 1, 2026 | Risks and controls folded into the steps and reviews |
| 0.5 | Nov 10, 2026 | Sized for a low-volume platform: one product is enough to create an archetype or variant |
| 0.6 | Nov 10, 2026 | Roles reduced to five |
| 0.7 | Nov 10, 2026 | Tiers replaced by one standard path |
| 0.8 | Nov 10, 2026 | A use case joins any archetype whose model meets its floor; residual risk accepted in the profile |
| 0.9 | Nov 10, 2026 | Data classification removed |
| 0.10 | Nov 10, 2026 | Versioned releases: platform-owned reference evals, product-run acceptance tests, versioned aliases with a 90-day notice period |
| 0.11 | Nov 10, 2026 | Reduced to the platform's framework. Removed: product responsibilities and product-design refusals (products attest only to the archetype's conditions and residual risk), sample clearing, tooling, repository paths and repeated rules. Task, metric and starting-archetype tables merged |
| 0.12 | Nov 11, 2026 | Kill switch and fallbacks removed: no kill switch, no product fallback and no fallback model. A model that must go is replaced by the next version on the fast track |
| 0.13 | Nov 12, 2026 | Templates moved to their own document |
| 0.14 | Nov 12, 2026 | Simplification |
| 0.15 | Nov 13, 2026 | Reference evals built once from public datasets and iterated over time: the capability test and the reference eval are one step, and the steps go from six to five. One reference-set score and a relative G5 policy, with the safety bar required, replace the reference floor and critical-error maximum, so a product's own bar is checked only by its acceptance test. A new archetype is only for a use case in no built archetype's scope; a catalog gap inside a scope is closed by the next version. Products choose their archetype and run their own acceptance test. Candidates pass the legal and security reviews before they are run. The caps on public benchmarks and on serving versions are removed |
| 0.16 | Nov 13, 2026 | Step 1 reduced to a conversation with the data science team and the acceptance set with its floor. The rest of the profile is completed when the product joins |
