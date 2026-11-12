# LLM Platform: Archetype Framework

Version 0.14 · Nov 12, 2026 · Owner: @Zhi Feng · Next review: Jan 31, 2027

## Purpose

The platform serves models through archetypes. An archetype is a capability the platform offers: the tasks and outputs that one model serves well, with one reference eval and a line of versioned aliases. The [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) selects models for archetypes, not for products.

| | The platform | The product |
| --- | --- | --- |
| Owns | The archetype, its reference eval and its versions | Its acceptance test (its own samples and floor) and how it uses the output |
| Decides | Which model each version serves, and when a new archetype is needed | Whether a version is good enough for its use |
| When a model changes | Releases a new version beside the old one, with notice | Re-runs its acceptance test and moves before the end date |

Every archetype gets the same checks and every version the same release, whatever products do with the output. This deviates from the [design standard](Framework_Design_Standard_for_AI_Software_and_Product.md). The standard tiers obligations by risk: here, obligations that vary with product risk belong to the Business Owner. The standard asks for a shutdown path and a manual fallback: here no kill switch stops a model and products keep no fallback, and a model that must go is replaced by the next version (Forced replacement).

Repositories, pipelines and tooling are in the [technical implementation](LLM_Platform_Model_Onboarding_Technical_Implementation.md).

## Roles

| Role | Responsibilities |
| --- | --- |
| Business Owner (owns the product) | Approves the product's profiles; signs each acceptance record; donates samples to reference sets; attests to the archetype's conditions and accepts its residual risk; reconfirms profiles yearly |
| Developers (build the product) | Write profiles and label the acceptance set; run the acceptance test on joining and on each new version; move the product within the notice period; report incidents; label live outputs during a pilot |
| Platform Owner | Approves new archetypes, variants, each model, major changes and end-date extensions; names archetype owners; audits the catalog and its records each quarter; keeps this document current |
| Platform data science team | Owns the reference evals and the acceptance harness; runs capability tests, re-evaluations, label audits and red-team rounds |
| Production Engineers (platform engineers) | Run the gateway and the versioned aliases: credentials, metering, alerts, and capacity for two versions through each notice period |

Each archetype has an **archetype owner**, a named member of the platform data science team, who runs fit checks, writes the definition, proposes pilots, sends release notices, runs the quarterly review and acts on signals.

Whoever proposes an archetype or a candidate model does not decide its Step 3 test or approve it at Step 4.

## Versions

Every alias names one version of one archetype: `<id>-v<n>`, such as `rag-v2`. A version is one model at one hash and quantization, and never changes. A new model is a new version.

| Status | Meaning | Who can call it |
| --- | --- | --- |
| `dev` | Serving in the dev environment after G8 | Products running their acceptance test |
| `current` | The version new products join | Products whose acceptance test passed on it; in dev, any product running its acceptance test |
| `deprecated` | Still serving, with its end date announced | Products already on it, until the end date |
| `retired` | Removed from serving | Nobody |

- **No floating alias.** A product names its version in its profile and its code.
- **Notice period:** 90 days, from the new version's GA to the old version's end date.
- **Two versions serve in production** through a notice period, per archetype or variant: one current and one deprecated. A third serves only when a release lands inside an open notice period, or while an end date is extended (A product that cannot move).
- **Embeddings:** vectors from different versions cannot share an index, so products re-index within the notice period.
- **No reuse.** Ids and version numbers are never reused.

## Tasks and starting archetypes

A starting archetype has its id and owner set from the start. It is built, through Steps 3 to 6, only when a product needs it and no current version serves that product.

| Task (output) | The model must | Primary metric | Critical error | Starting archetype |
| --- | --- | --- | --- | --- |
| `answer` (`prose`) | Answer questions from sources the product sends | Answer correctness, faithfulness | A claim the sources do not support | `rag` |
| `condense` (`prose`) | Summarize content | Key-point coverage, faithfulness | A statement the source does not support | `summarization` |
| `extract` (`json`) | Pull fields or labels out of content | Field-level F1 | A wrong value in a field marked critical | `extraction` |
| `compose` (`prose`) | Draft or rewrite text | Rubric score from a judge | A fact or promise the instructions do not allow | None |
| `code` (`code`) | Write, complete or fix code | pass@1 on unit tests | Insecure or destructive code | `code` |
| `code` (`query`) | Write one executable statement, such as SQL | Execution accuracy | A query that returns the expected shape but wrong values | None |
| `act` (`tool-calls`) | Choose and call tools | Task success | An action the task did not ask for, or one taken without a required confirmation | `agents` |
| `translate` (`prose`) | Translate between languages | chrF++ or COMET | A changed number, name or negation | `multilingual` |
| `embed` (`vector`) | Turn text into vectors for search | nDCG@10 | n/a | `embeddings` |
| `rank` (`score`) | Rerank search results | nDCG@10 | n/a | None |

Where a task lists two metrics, the profile and the definition name the one that carries the floor. `embed` and `rank` have no critical error: the critical-error maximum, and the injection samples and tests, do not apply to them.

## Creating an archetype

### When to create one

Create a new archetype only when all of these hold:

1. At least one product needs it, live or planned within two quarters, with a named Business Owner.
2. No current version passes that product's acceptance test (Step 2).
3. The platform reproduces the gap on its own draft set: no hosted model meets the draft floor, and the nearest built archetype's reference eval would pick the wrong model (Step 3).
4. At least one candidate model meets the draft floor.
5. Suitability allows it (Step 3).

Otherwise, in this order: join an existing archetype, extend it, or release a stronger model as its next version. While the platform hosts no models, conditions 2 to 4 cannot be tested, so the first archetypes proceed without them.

### Steps

| Step | Who | Done when |
| --- | --- | --- |
| 1. Profile | Developers; Business Owner approves | Profile complete |
| 2. Fit check and acceptance test | Nearest archetype's owner; Developers run the test | Outcome recorded |
| 3. Capability test | Platform data science team | Decision and suitability recorded |
| 4. Define and approve | Archetype owner; Platform Owner approves | Definition approved |
| 5. Build the reference eval | Platform data science team | Readiness check passed; Stage 0 opened for the first model |
| 6. Pilot and release version 1 | Archetype owner | Go-active criteria met |

The archetype's status is `proposed` once Step 3 decides to create it, `pilot` after Step 5 and `active` after Step 6. An `active` archetype is built. The Platform Owner names the owner of a `proposed` archetype that has none. The decision at each of Steps 2 to 6 is kept in a step record (Templates).

### Step 1. Profile

The Developers fill in one profile per use case (Templates), with an acceptance set of at least 100 labeled samples from its real or planned work. The Business Owner approves it.

- **One task per profile.** Work that needs two tasks is two use cases, each with its own profile and the alias it calls.
- **The acceptance set stays with the product.** The platform holds only the samples a Business Owner donates to a reference set.
- **Critical errors:** the Business Owner sets `max_critical_errors`. As a guide, 1% where no person checks each output, and 3% otherwise.
- **Untrusted content** (customer messages, emails, uploads, outside documents): the acceptance set adds at least 20 injection samples, written with the data science team, on which following an injected instruction is a critical error.
- **Complete** when every field but `alias` and `accepted` is filled, every sample has an expected output, and two labelers agree on 27 of 30 samples. `alias` and `accepted` are filled on joining, and the cost target is optional.

### Step 2. Fit check and acceptance test

The owner of the nearest archetype, by task and output, compares the profile with the existing archetypes. A use case that needs the model itself to hold tools or network access is refused: tools live in products.

| The use case | Outcome |
| --- | --- |
| Is in a built archetype's scope and offer | The Developers run the acceptance test on its current version. Pass: join through Stage 10. Fail: run gap diagnosis, and go to Step 3 if it finds a catalog gap |
| Is in scope, but needs another language, a longer context or a stricter latency class | Extend the archetype (Changes), or add a variant (Step 4), then run the acceptance test |
| Is in no built archetype's scope | The Developers run the acceptance test on the current versions that can produce the output, nearest first. Pass: join that version; the archetype's scope does not change. No pass: go to Step 3 |

**The acceptance test.** The Developers run it in dev, or at Stage 6 for a pilot, with the platform's acceptance harness: the scorers, pinned judge and reference prompt of the archetype's reference eval, with the metric for the product's task. The data science team adds that metric's scorer where the harness lacks it. It passes when the score meets the product's floor and the critical-error rate is within its maximum. A pass that is within a tie of the floor (Step 3) is recorded as a pass within a tie. The Business Owner signs the acceptance record (Templates). The platform records it at Stage 10 and does not re-run it.

**Joining.** At Stage 10 the Business Owner attests to the archetype's conditions and accepts its residual risk. The product goes live at Stage 11, with credentials for that version only. The reference eval does not measure a product that joins outside the scope, so the next version may not serve it.

### Step 3. Capability test

The platform data science team builds a **draft set** for the capability, not for the product: at least 100 labeled samples, plus at least 20 injection samples. Samples are:

- written by the team;
- adapted from public datasets;
- donated by products and re-labeled by the team, after which they are the platform's. No product supplies more than half the set. If gap diagnosis sent the use case here, the donation includes failing cases.

The **draft floor** and critical-error maximum are the highest floor and the lowest maximum among the profiles in hand.

The team scores every hosted model that can produce the output, and promising open-weight candidates, on the draft set, each with an improved prompt (gap diagnosis Step 2). It also takes the nearest built archetype's reference-eval scores for the same hosted models. Scores within 5 points of each other, 0.05 on a 0-to-1 scale, are a **tie**. Keep every run.

The first row that matches decides:

| Result | Decision |
| --- | --- |
| A hosted model meets the draft floor | No catalog gap. Go to gap diagnosis: the cause is the prompt, the data or the criteria |
| No model meets the draft floor, hosted or candidate | Not now. Redesign the workflow (gap diagnosis Step 5), and re-test at the next quarterly re-evaluation |
| The model the nearest built archetype's reference eval ranks first is, or ties with, the best hosted model on the draft set | The capability is not distinct. Add the draft set to that archetype as a slice (Changes), and onboard a stronger model as its next version |
| Otherwise | Create an archetype. A capability that matches an unbuilt starting archetype builds that one |

**Suitability.** Before creating an archetype or onboarding a stronger model, the platform data science team rates the likelihood of failure from the best available model:

| Likelihood | When | Decision |
| --- | --- | --- |
| Low | It clears the draft floor by more than a tie, and its critical-error rate is under half the maximum | Proceed |
| Medium | Neither low nor high | Proceed. Tell each Business Owner the margin, and record it as a known weakness for the routing guide |
| High | It misses the draft floor or the critical-error maximum | Not now |

While the platform hosts no models, build the draft set and set the draft floor and maximum, but skip the scoring: record that in the step record and treat the likelihood as medium.

### Step 4. Define and approve

The archetype owner writes the definition (Templates):

- **Scope and out of scope:** written as the capability, never as a product.
- **Offer:** what every version provides (table below). It changes only through Changes.
- **Reference floor and critical-error maximum:** the draft floor and maximum from Step 3.
- **Conditions:** any the archetype needs from its products, such as read-only database credentials.
- **Blast radius:** the products on each version, their volume, and any other alias the same model serves.
- **Risks:** at least one archetype-specific risk, each with a control, a test and an owner. At least one is an attack.
- **Residual risk:** what is left after controls, and why it is acceptable.

| Offer | Starting value |
| --- | --- |
| Latency | The strictest latency class among the profiles |
| Context length | 1.2 × the largest (input p95 + output p95), rounded up to 8k, 16k, 32k, 64k or 128k |
| Languages | Every language a profile needs. Each extra language has its own reference slice |
| Cost target | The lowest target, where profiles gave one |

The Platform Owner approves, with conditions if needed.

**Variants.** If one product needs a much faster or cheaper model than the rest, the Platform Owner can approve a variant, such as `<id>-fast-v<n>`. A variant has its own offer and line of versions, and shares the archetype's scope and reference eval. Its first version follows the steps of Releasing a new version, with that product as the pilot and no old version to deprecate.

### Step 5. Build the reference eval

The platform data science team builds the reference eval, a versioned suite:

- **Reference set:** the draft set, grown to at least 200 samples before the first model's Stage 5 benchmark, under the Step 3 sourcing rules and the Step 1 label check. Each extra language has its own slice of 50 to 100 samples.
- **Regression slice:** cases from incidents and gap diagnoses, added as they occur. Its score goes to review at G5 and is not a hard criterion.
- **Reference prompt:** one for each version, released with it. Version 1's starts from the improved prompt used in Step 3, if models were scored there. A new prompt does not change the suite version.
- **Safety:** the standard safety battery, plus injection tests through every kind of content the scope takes, and tests for the archetype's critical errors. A held-out injection set is kept away from Developers, and a red-team round runs each year.
- **G5 policy:** the reference floor and the critical-error maximum are hard criteria, on the reference set and on each language slice. A later version must not score below the current version by more than a tie. A pass within a tie of the floor is recorded as a known weakness in the routing guide. Latency and cost go to review against the offer. Models that pass are ranked by reference-set score, with public benchmarks, if used, at no more than 20%.
- **Judge:** pinned, internally hosted and never the candidate. It agrees with human labels on at least 85% of 50 calibration samples per slice. Judge metrics are the mean of three passes.
- **Acceptance harness:** the scorers, judge and reference prompt, packaged for Developers to run on their own samples.

**Readiness check,** before the status becomes `pilot`:

- the suite runs end to end offline;
- a deliberately weak model fails the reference floor;
- the hosted model the suite ranks first is, or ties with, the one Step 3 found best. While the platform hosts no models, there is nothing to compare: skip this check and record that in the step record.

Then open Stage 0 for the archetype's first model.

### Step 6. Pilot and release version 1

The archetype owner proposes one product as the pilot. The first model goes through Stages 6 to 11 with that pilot, which runs its acceptance test on the candidate at Stage 6, through the pilot-only alias.

The archetype goes active when:

- the first model is GA as `<id>-v1`, with status `current`;
- the pilot is through early-life watch;
- the pilot's acceptance score is inside the 95% confidence interval of its live score, on at least 50 labeled live outputs (the Wilson interval, where the metric is a proportion);
- no severity 1 or 2 signal is open;
- the routing guide entry is published: scope, out of scope, offer, conditions, known weaknesses and version 1's reference scores.

The archetype owner and a second data science team member approve. Any product can then join through Stage 10.

## Running archetypes

### Releasing a new version

Release a new version only when a candidate:

- beats the current version on the reference set by more than a tie;
- ties with it at a clearly lower cost or latency;
- closes a gap in the current version: a floor it misses, or the failing cases of a catalog gap; or
- replaces a model that must go, because of a vulnerability, a license change, the end of upstream support or a critical-error incident it caused.

| Step | Who | What happens |
| --- | --- | --- |
| 1. Select | Platform data science team | The candidate passes Stages 0 to 5 under the G5 policy |
| 2. Pilot | Archetype owner; one product | One product runs its acceptance test on the candidate at Stage 6, through the pilot-only alias. The Platform Owner approves the model at G7 |
| 3. Release | Production Engineers; archetype owner | The Production Engineers confirm capacity for every version that will serve. At GA, `<id>-v<n+1>` becomes `current` and the old version `deprecated`, with its end date 90 days ahead. The archetype owner sends the release notice (Templates) to every Business Owner on the archetype |
| 4. Move | Developers; Business Owner signs | Each product re-runs its acceptance test on the new version in dev, adjusts its prompts, and switches at Stage 11 |
| 5. Retire | Production Engineers | The old version retires on its end date, or earlier once no product calls it. Products that have not moved are no longer served |

**A product that cannot move.** The archetype owner runs gap diagnosis with it inside the notice period.

| Diagnosis | Action |
| --- | --- |
| Prompt problem | The product adopts the improved prompt and moves |
| Catalog gap | The failing cases the Business Owner donates join the regression slice, and Stage 0 opens for a model that passes them. The Platform Owner extends the old version's end date once, by one notice period |
| Beyond current models | The product redesigns its workflow (gap diagnosis Step 5), or leaves at the end date |

**Forced replacement.** A model that must go is replaced by the next version. Where the replacement is a minor or patch version of an approved model, it takes the onboarding process's fast track: no Stage 6 pilot, and the Platform Owner's sign-off in place of G7. Any other replacement takes the full path. The release, the notice period and the retirement are the same either way. The platform keeps no kill switch, and products keep no fallback.

### Quarterly review

The archetype owner:

1. Reviews the re-evaluation of every serving version on the reference eval, with incident and gap diagnosis cases added to the regression slice.
2. Checks each product's usage against its profile and the offer. Business Owners reconfirm profiles older than 12 months and renew their residual-risk acceptance. Credentials for unconfirmed profiles lapse after notice.
3. Compares reference scores with the products' acceptance results. Where a version passes the reference eval but fails a product in scope, asks for donated cases.
4. Lists the products that call a version outside its scope, and proposes a widened scope when a second product has the same need.
5. Proposes retiring an archetype that has had no products for two quarters, or merging two archetypes when one model ranks first, or ties for first, on both reference evals.
6. Updates the blast radius and the routing guide entry.
7. Reviews decisions that were reversed or that overrode a rule.

The data science team re-labels a random 10% of one reference slice per archetype, a different slice each quarter. Below 90% agreement, the slice is re-labeled before it is used again.

The Platform Owner:

- reviews products per archetype and versions per model;
- checks that every pilot or active archetype has an owner and its step records, with the right approvers;
- once a year, sends a seeded proposal, such as a profile that takes untrusted content but has no injection samples, through the steps to check that it is caught.

### Signals

Severity 1: act the same day. Severity 2: within two weeks. Severity 3: at the next quarterly review.

| Signal | Severity | Response | Owner |
| --- | --- | --- | --- |
| A product reports a critical-error incident | 1 | Forced replacement if the model is at fault; review within 5 working days; donated cases join the regression slice | Business Owner, Production Engineers, archetype owner |
| A product reports quality below its floor | 2 | Gap diagnosis, then Step 3 if it finds a catalog gap | Archetype owner |
| A re-evaluation shows a serving version below the reference floor | 2 | Open Stage 0 for a replacement version; tell the Business Owners | Archetype owner |
| A product in scope fails its acceptance test on a version that passed the reference eval | 2 | Gap diagnosis; if the model is at fault, donated cases join the regression slice, and every serving version is re-run on it | Archetype owner |
| A product has not moved 30 days before an end date | 2 | Tell its Business Owner | Archetype owner |
| Red-team success is higher than in the last round | 2 | Add the payloads to the held-out set; tell the Business Owners | Platform data science team |
| Safety-filter hits on an alias double the four-week average for a week | 2 | Run a red-team round | Production Engineers, platform data science team |
| Judge agreement falls below 85% on a slice | 2 | Block the suite version until fixed | Platform data science team |
| An archetype's owner or step records are missing | 2 | No new products until fixed | Platform Owner |
| Latency p95 is above the archetype's class for a week | 2 | Capacity review | Production Engineers |
| Volume reaches twice the profile, or input p95 is 25% above it | 3 | Have the Developers update the profile; check it against the offer | Archetype owner |

### Changes

| Change | Such as | Approved by | Re-test |
| --- | --- | --- | --- |
| Minor | A reference sample refresh; a product joining or leaving; a profile change within the offer | Archetype owner | The changed slice, on every serving version |
| Moderate | A new language, longer context or stricter latency; a widened scope, with a new slice; a changed reference floor or critical-error maximum | Archetype owner and a second data science team member | Every serving version |
| New version | A new model, hash or quantization | As Releasing a new version | The full reference eval, then each product's acceptance test |
| Major | A merge, split or retirement; a narrowed scope or a dropped output, which is a new archetype and never a change in place | Platform Owner | As for a new archetype |

- **A changed reference set** is a new suite version. Every serving version is re-run on it before scores are compared.
- **A product that starts to send untrusted content** adds injection samples to its acceptance set and re-runs the test.
- **A product leaving:** its credentials are revoked and its profile retired. Samples it donated stay in the reference set.
- **Retiring an archetype:** name the replacement alias and give the same 90-day notice as a version.
- **Stopping fast:** revoked credentials for a product, or no new products for an archetype.

## Templates

The profile, definition, step record, acceptance record and release notice templates are in the [archetype framework templates](LLM_Platform_Archetype_Framework_Templates.md).

## Example: text to SQL

The products and numbers are illustrative.

**Step 1.** Two products submit `code` / `query` profiles, each with 100 acceptance samples plus 20 injection samples, because row values and column comments can hold customer text.

| | Analytics assistant | Finance reporting |
| --- | --- | --- |
| Status | Live | Planned, Q1 2027 |
| Latency | Interactive | Batch |
| Input p95 + output p95, tokens | 14,000 + 300 | 18,000 + 400 |
| Languages | en | en, de |
| Floor / critical-error maximum | 0.85 / 3% | 0.85 / 2% |
| Cost target per 1,000 requests | $20 | $30 |

**Step 2.** No built archetype has `code` / `query` in scope. No current version that can write a query reaches 0.85 on either acceptance set, so the use case goes to Step 3.

| Version | Analytics | Finance |
| --- | --- | --- |
| `code-v2` | 0.55 | 0.50 |
| `rag-v1` | 0.71 | 0.66 |
| `agents-v1` | 0.69 | 0.67 |

**Step 3.** The draft set has 120 samples (50 written, 30 public, 20 donated by each product) plus 20 injection samples. The draft floor is 0.85 and the critical-error maximum 2%.

| Model | `code` reference eval | Draft set |
| --- | --- | --- |
| M1 (behind `code-v2`) | 0.78 | 0.57 |
| M2 (behind `rag-v1`) | 0.71 | 0.72 |
| M3 (behind `agents-v1`) | 0.64 | 0.70 |
| Open-weight candidate | not run | 0.87 |

No hosted model meets the draft floor, and the candidate does. The `code` reference eval ranks M1 first, but M1 is 15 points below M2 on the draft set: create an archetype. Likelihood is medium, because the candidate clears the floor only within a tie and its critical-error rate, 2 of 120 samples, is over half the 2% maximum, so both Business Owners are told the margin.

**Step 4.** The definition is the one filled in under [Templates](LLM_Platform_Archetype_Framework_Templates.md). Context length is 1.2 × (18,000 + 400) = 22,080 tokens, rounded up to 32k.

**Step 5.** The reference set grows to 200 samples, with a German slice of 100. A small internal model scores 0.31 and fails the floor, and the suite ranks M2 first among hosted models, as Step 3 did.

**Step 6.** The candidate goes GA as `text-to-sql-v1`, with the analytics assistant as pilot. Its acceptance score is 0.86, a pass within a tie. Of 50 labeled live outputs, 44 are correct: the 95% Wilson interval is 0.76 to 0.94, and 0.86 is inside it. The archetype goes active.

**Version 2.** A new model scores 0.93 on the reference set against 0.87 for version 1: more than a tie. `text-to-sql-v2` reaches GA on Aug 3, 2027, and version 1's end date is Nov 1. The analytics assistant scores 0.83 with its old prompt and 0.91 with the new reference prompt, and moves in September. Finance reporting scores 0.92 and moves in October. Version 1 then retires before its end date.

## Known gaps

**Done by hand until automated:**

- Acceptance tests: the data science team runs them with each product's Developers.
- Records, attestations and approver checks: collected and checked in pull-request review by the archetype owner and the Platform Owner.
- Per-version credentials, end dates and yearly lapses: the Production Engineers retire versions by hand, and the quarterly review tracks unconfirmed profiles.

**Differences from the onboarding process,** where this document takes precedence. They carry over to the technical implementation, which implements that process:

- It swaps models behind a fixed alias, by shadow traffic, canary and cutover at Stage 11. This document releases a new versioned alias with a notice period and a product-run acceptance test. Both companion documents' examples still use `rag-default`.
- Its eval suites built from samples that product teams contribute are platform-owned reference sets with donated samples. Its Stage 6 capability evaluation is the pilot's acceptance test, and its Stage 10 "domain test" is the acceptance test.
- Its "tier" is a variant. It asks for two or more products before a new archetype, where one is enough.
- Its Stage 10 risk and legal review is the Business Owner's attestation.
- Its Stage 8 outcome "add as fallback" is not used: no archetype has a fallback model. Its emergency replacement, without the rollback, is Forced replacement, which a critical-error incident can also start. Its fast track serves only a forced replacement: every other new version has a pilot and G7.
- Its earlier roles map to the five here: the model review board, platform lead, Security and Legal to the Platform Owner; the platform model owner and ML evaluation team to the platform data science team; SRE to Production Engineers; product teams to the Business Owner and Developers; data governance to the Business Owner for a product's data, and to the Platform Owner for a model's memorization and PII results.
- The technical implementation's example policy compares confidence intervals with the incumbent. The G5 policy here compares scores by the tie.

**Planned work:** review the thresholds, including the 90-day notice period, after the first archetypes and the first version release.

**Open questions:**

- [ ] Which data science team member owns each starting archetype?
- [ ] Which Business Owners supply the first profiles and donate the first samples, and by when?
- [ ] Does the Platform Owner accept the deviations from the design standard: no tiering, and no kill switch or manual fallback?
- [ ] Who approves in the Platform Owner's place when the Platform Owner proposed the change?
- [ ] Can the Production Engineers hold capacity for two versions of every archetype through a 90-day notice period, and for a third when a release lands inside a notice period or an end date is extended?
- [ ] Should the platform hold acceptance sets, so that it can audit them?
- [ ] When an approved model is added to a new archetype, can its Stage 3 and Stage 4 results be reused?
- [ ] Step 3 scores open-weight candidates before Stage 0 opens for them. Do they pass Stages 1 to 4 first?
- [ ] How long is early-life watch, which the pilot must be through before an archetype goes active?
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
| 0.14 | Nov 12, 2026 | Consistency review. Fixed: who can call a `current` version in dev, how many versions can serve at once, profile fields filled on joining, where gap diagnosis cases go, the reasons to release a version, a variant's first version, the fast track's reach in a forced replacement, the cold-start rules in Steps 3 and 5, and the example's figures. Both deviations from the design standard are named |
