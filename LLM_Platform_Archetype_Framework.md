# LLM Platform: Archetype Framework

Version 0.24 · Nov 17, 2026 · Owner: @Zhi Feng · Next review: Jan 31, 2027

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
| Business Owner (owns the product) | Identifies and classifies the product's data; approves the product's profiles; signs each acceptance result; donates cases to regression slices; attests to the archetype's conditions and accepts its residual risk |
| Developers (build the product) | Write profiles and label the acceptance set; choose the archetype; run the acceptance test on joining and on each new version; move the product within the notice period; report incidents |
| Platform Owner | Approves new archetypes, variants, major changes, retirements and end-date extensions; names archetype owners; keeps this document current |
| Platform data science team | Owns the reference evals; runs gap diagnoses, capability tests, re-evaluations, label audits and red-team rounds |
| Production Engineers (platform engineers) | Run the gateway and the versioned aliases: credentials, metering, alerts, and capacity for every serving version through each notice period |

Each archetype has an **archetype owner**, a named member of the platform data science team, who advises products on their choice of archetype, records scope, writes the definition, proposes pilots, sends release notices and runs the quarterly review.

Whoever proposes an archetype or a candidate model does not decide its Step 3 test or approve it at Step 4.

## Versions

Every alias names one version of one archetype: `<id>-v<n>`, such as `rag-v2`. A version is one model at one hash and quantization, and never changes. A new model is a new version.

| Status | Meaning | Who can call it |
| --- | --- | --- |
| `dev` | Serving in the dev environment | Products running their acceptance test |
| `current` | The version new products join | Products whose acceptance test passed on it |
| `deprecated` | Still serving, with its end date announced | Products already on it, until the end date |
| `retired` | Removed from serving | Nobody |

- **Notice period:** 90 days, from the new version's release date to the old version's end date.
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

Otherwise, in this order: join an existing archetype, extend it, or release a stronger model as its next version. Where no current version can produce the output, as before the platform's first release, condition 3 and the comparison in condition 4 cannot be tested, and the archetype proceeds without them. The safety bar still applies.

### Steps

| Step | Who | Done when |
| --- | --- | --- |
| 1. Use case and acceptance set | Developers, with the data science team; Business Owner approves | Acceptance set and floor approved |
| 2. Choose an archetype and test | Developers; the archetype's owner advises and records scope | Outcome recorded |
| 3. Capability test and reference eval | Platform data science team | Readiness check passed; decision and suitability recorded |
| 4. Define and approve | Archetype owner; Platform Owner approves | Definition approved |
| 5. Pilot and release version 1 | Archetype owner | Go-active criteria met |

The archetype's status is `proposed` once Step 3 decides to create it, `pilot` once Step 4 approves it and `active` after Step 5. Step 1's approval is kept in the profile, and the decisions of Steps 2 to 5 in step records (Templates).

### Step 1. Use case and acceptance set

The Developers talk to the platform data science team about what the platform offers and which archetype could fit.

They then start the profile (Templates), writing down only what the choice and the test need:

- **Use case, task and output.** One task per profile: a use case with two tasks is two profiles and calls two aliases.
- **Needs:** the latency class, and the input and output lengths.
- **Data and its classification.** The Business Owner identifies the data the product sends to the model, acceptance samples included, and defines its classification. The platform has one ceiling, the classification its environment is approved for, and data above it is not sent to the platform.
- **Acceptance set:** labeled samples from its real or planned work, each with an expected output. It stays with the product. The platform holds only the cases a Business Owner donates to a regression slice.
- **Metric, floor and critical-error maximum,** set before the test runs.
- **Untrusted content** (customer messages, emails, uploads, outside documents): the acceptance set adds a set of injection samples, vetted by the data science team, on which following an injected instruction is a critical error.

The Business Owner approves the profile so far. The rest of it is completed when the product joins (Step 2). Where the use case goes to Step 3, its cost target is added first, because the offer is built from it.

### Step 2. Choose an archetype and test

The Developers choose the archetype that fits the use case best, from the routing guide entries: scope, out of scope, offer, conditions and known weaknesses. The archetype's owner advises if asked, and records in the step record whether the use case is in that archetype's scope, in another built archetype's scope or in none. A use case in another archetype's scope is tested on that archetype first.

A use case in scope that needs a longer context or a stricter latency class first has the archetype extended (Changes), or a variant added (Step 4).

The Developers then run the acceptance test on the archetype's current version.

| Result | Outcome |
| --- | --- |
| Pass | Join. Joining from outside the scope does not change the scope |
| Fail, in scope | Gap diagnosis. Close a catalog gap with the archetype's next version (Releasing a new version) |
| Fail, in no built archetype's scope | Gap diagnosis. Go to Step 3 if it finds a catalog gap |

Where no current version can produce the output, there is nothing to choose or test, and the use case goes to Step 3.

**The acceptance test.** The Developers run it themselves, in dev, on their own acceptance set. In scope, they score with the scorers, pinned judge and reference prompt of the archetype's reference eval, which the data science team gives them. Outside the scope, they score with their own method. The test passes when the score meets the product's floor and the critical-error rate is within its maximum. When a product's scoring method changes, as when it moves from its own method to a new archetype's reference eval, the Business Owner re-approves the floor first. The Business Owner signs the acceptance record (Templates). The platform records it at joining and does not re-run it.

**Joining.** A product joins once its acceptance test has passed. The Developers complete the profile with the alias, volume and cost target. The archetype's owner checks it against the offer, and a need the offer does not cover is handled by an extension or a variant before the product goes live. The Business Owner attests to the archetype's conditions and accepts its residual risk.

**Gap diagnosis** is the platform's check on a fail: a failed acceptance test, or a live product that reports quality below its floor. The data science team scores the product's samples, with its Developers, on the hosted models before anything is built. If another archetype's model passes, the product may join that archetype instead, from outside its scope. For a live product, a catalog gap is closed with the archetype's next version, or goes to Step 3 if the use case is in no built archetype's scope.

### Step 3. Capability test and reference eval

The platform data science team builds a **reference eval**, the suite, once, for the capability and not for the product. The suite decides this step. If the step creates an archetype, it becomes that archetype's reference eval and selects every later version (Selection policy, under Releasing a new version).

- **Reference set:** one or more public datasets for the task and output, each pinned at a revision and used as published. It gives one reference-set score, over all its samples. The set is iterated over time: when products fail on cases it does not cover, the team adds or replaces datasets until it does (Changes).
- **Regression slice:** cases that Business Owners donate from incidents and gap diagnoses, added as they occur. The team re-labels them, after which they are the platform's. The slice's score goes to review under the selection policy and is not a hard criterion.
- **Reference prompt:** one, used for every model.
- **Safety:** the standard safety battery, plus injection tests through every kind of content the capability takes, and tests for the archetype's critical errors. A held-out injection set is kept away from Developers, and a red-team round runs each year. Embedding and ranking archetypes have no injection tests.
- **Judge:** pinned, internally hosted and never the candidate. It agrees with human labels on at least 85% of 50 calibration samples per dataset. Judge metrics are the mean of three passes.

Each promising open-weight candidate enters the onboarding process. A candidate is run only after it has passed that process's intake and its legal and security reviews. The team then runs the suite on every hosted model that can produce the output and on those candidates. A hosted model is the model behind a current version. For a candidate, this run is also its benchmark result in the onboarding process. Keep every run.

**Readiness check,** before the decision: the suite runs end to end offline, and a deliberately weak model scores more than a tie below the best model.

The result decides:

| Result | Decision |
| --- | --- |
| No candidate meets the safety bar and beats the best hosted model by more than a tie, with a critical-error rate no higher | Not now. Redesign the workflow (gap diagnosis Step 5), and re-test at the next quarterly re-evaluation |
| Otherwise | Create an archetype, if suitability allows. A capability that matches an unbuilt starting archetype builds that one |

**Suitability.** Before creating an archetype, the archetype owner rates the likelihood that the best candidate fails the products. The best candidate is the highest-scoring one that meets the safety bar and beats the best hosted model by more than a tie, with a critical-error rate no higher. The rating rests on two numbers:

- **Lead:** the candidate's margin over the best hosted model on the reference set.
- **Shortfall:** the largest gap, among the products, between a product's floor and the best score a hosted model reached on its acceptance set in gap diagnosis.

| Likelihood | When | Decision |
| --- | --- | --- |
| Low | The lead is more than a tie above the shortfall | Proceed |
| Medium | Anything else | Proceed. Tell each Business Owner the lead and the shortfall, and record them as a known weakness in the routing guide |
| High | The lead is more than a tie below the shortfall | Not now |

Where no hosted model can produce the output, there is nothing to compare a candidate with. The team still builds the suite and runs the candidates, and the best candidate is the highest-scoring one that meets the safety bar. If none meets it, the decision is not now. Otherwise record the decision to create the archetype in the step record, treat the likelihood as medium and tell each Business Owner that nothing was compared.

### Step 4. Define and approve

The archetype owner writes the definition (Templates):

- **Scope and out of scope:** written as the capability, never as a product.
- **Offer:** what every version provides (table below). It changes only through Changes.
- **Reference eval:** the datasets, their pinned revisions and the metric from Step 3.
- **Conditions:** any the archetype needs from its products, such as read-only database credentials.
- **Blast radius:** the products on each version, their volume, and any other alias the same model serves.
- **Risks:** at least one archetype-specific risk, each with a control, a test and an owner. At least one is an attack.
- **Residual risk:** what is left after controls, and why it is acceptable.

The Platform Owner approves, with conditions if needed. The status becomes `pilot`, and the best candidate completes its benchmarking in the onboarding process.

**Variants.** If one product needs a much faster or cheaper model than the rest, the Platform Owner can approve a variant, such as `<id>-fast-v<n>`. A variant has its own offer and line of versions, and shares the archetype's scope and reference eval.

### Step 5. Pilot and release version 1

The archetype owner proposes one product as the pilot. The pilot runs its acceptance test on the candidate, and takes the first model through the rest of the onboarding process to go-live.

The archetype goes active when:

- the first model is released as `<id>-v1`, with status `current`;
- the pilot is through early-life watch;
- the pilot's acceptance score is inside the 95% Wilson interval of its live score, on a set of labeled live outputs;
- no critical-error incident is open;
- the routing guide entry is published: scope, out of scope, offer, conditions, known weaknesses and version 1's reference scores.

The archetype owner and a second data science team member approve. Any product can then join (Step 2).

## Running archetypes

### Releasing a new version

Release a new version only when a candidate:

- beats the current version on the reference set by more than a tie;
- ties with it at a clearly lower cost or latency; or
- replaces a model that must go, because of a critical-error incident where the model is at fault, a vulnerability, a license change or the end of upstream support.

**Selection policy.** One policy selects the model for every version, on the reference eval:

- **Safety:** every candidate meets the safety bar of the standard safety battery. A first version is no exception.
- **Hard criteria:** on the reference set, the candidate scores no more than a tie below the current version, and its critical-error rate is no higher. The critical-error rate counts critical errors on the reference set and failures on the injection and critical-error tests.
- **First version of an archetype:** the best hosted model at Step 3 stands in for the current version, and the candidate beats it by more than a tie. Where Step 3 had nothing to compare, candidates are only ranked.
- **First version of a variant:** there is no comparison. The candidate meets the variant's offer, and the product that needs the variant runs its acceptance test on it, as the pilot.
- **For review:** the regression slice score, and latency and cost against the offer.
- **Ranking:** candidates that pass are ranked by reference-set score.
- **Known weakness:** a version that scores below the one it replaces has that recorded in the routing guide.

The policy sets no absolute floor. Whether a version is good enough for a product is decided by that product's acceptance test.

| Step | Who | What happens |
| --- | --- | --- |
| 1. Select | Platform data science team | The candidate passes the onboarding process's reviews and the selection policy |
| 2. Pilot | Archetype owner; one product | One product runs its acceptance test on the candidate. The Platform Owner approves the model |
| 3. Release | Production Engineers; archetype owner | The Production Engineers confirm capacity for every serving version. On the release date, `<id>-v<n+1>` becomes `current` and the old version `deprecated`, with its end date 90 days ahead. The archetype owner sends the release notice (Templates) to every Business Owner on the archetype |
| 4. Move | Developers; Business Owner signs | Each product re-runs its acceptance test on the new version in dev, adjusts its prompts, and switches to it in production |
| 5. Retire | Production Engineers | The old version retires on its end date, or earlier once no product calls it. Products that have not moved are no longer served |

**Closing a catalog gap.** Where gap diagnosis finds a catalog gap for a use case in the archetype's scope, the failing cases the Business Owner donates join the regression slice, the team iterates the reference set until it shows the gap (Changes), and a promising candidate enters the onboarding process, which must then beat the current version on it. If none does, the product redesigns its workflow (gap diagnosis Step 5).

**A product that cannot move.** The archetype owner runs gap diagnosis with it inside the notice period.

| Diagnosis | Action |
| --- | --- |
| Prompt problem | The product adopts the optimized prompt and moves |
| Catalog gap | Close the catalog gap, as above, or go to Step 3 if the use case is in no built archetype's scope. The Platform Owner extends the old version's end date once, by one notice period |
| Beyond current models | The product redesigns its workflow (gap diagnosis Step 5), or leaves at the end date |

**Forced replacement.** A model that must go is replaced by the next version, released on the onboarding process's fast track. When a product reports a critical-error incident, the archetype owner acts the same day and reviews it within 5 working days. If the model is at fault it must go, and the cases the Business Owner donates join the regression slice. The platform keeps no kill switch, and products keep no fallback.

### Quarterly review

The archetype owner:

1. Reviews the re-evaluation of every serving version on the reference eval, with incident and gap diagnosis cases added to the regression slice.
2. Updates the archetype's model: where a candidate model qualifies, proposes a new version (Releasing a new version).
3. Proposes retiring an archetype that has had no products for two quarters, or merging two archetypes when one model ranks first, or ties for first, on both reference evals.

The data science team re-labels a random 10% of each regression slice. Below 90% agreement, the slice is re-labeled before it is used again. Public datasets are used as published.

### Changes

| Change | Such as | Approved by | Re-test |
| --- | --- | --- | --- |
| Minor | A new revision of a reference dataset; new regression cases; a profile change within the offer | Archetype owner | The changed part of the suite, on every serving version |
| Moderate | A longer context or stricter latency; a widened scope; an added or replaced reference dataset | Archetype owner and a second data science team member | Every serving version |
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

**Step 1.** Two products talk to the data science team and start `code` / `query` profiles, each with 100 acceptance samples plus 20 injection samples, because row values and column comments can hold customer text. Each Business Owner identifies the product's data and defines its classification. They set their floors and state these needs:

| | Analytics assistant | Finance reporting |
| --- | --- | --- |
| Status | Live | Planned, Q1 2027 |
| Data | Warehouse schemas and sample rows, user questions | Ledger schemas and sample rows, report requests |
| Classification | Within the platform's ceiling | Within the platform's ceiling |
| Latency | Interactive | Batch |
| Input p95, tokens | 14,000 | 18,000, with outputs up to 400 |
| Floor / critical-error maximum | 0.85 / 3% | 0.85 / 2% |
| Cost target per 1,000 requests | $20 | $30 |

**Step 2.** No built archetype has `code` / `query` in scope. Both products choose `code`, the best fit, and its owner records the use case as in no built archetype's scope. They run their acceptance tests on `code-v2` with their own scoring, and fail: 0.55 and 0.50 against a floor of 0.85.

Gap diagnosis scores both acceptance sets on every hosted model that can write a query:

| Version | Analytics | Finance |
| --- | --- | --- |
| `code-v2` | 0.55 | 0.50 |
| `rag-large-v1` | 0.71 | 0.66 |
| `agents-v1` | 0.69 | 0.67 |

An optimized prompt lifts the best hosted score to 0.74 on the analytics set and 0.70 on the finance set, and only the reference model passes: a catalog gap. The use case goes to Step 3, so both products add their cost targets to their profiles.

**Step 3.** The reference set is one public text-to-SQL dataset, pinned at a revision. The team adds injection tests through column comments and row values, and tests for destructive statements. One open-weight candidate passes the legal and security reviews and is run with the hosted models. A small internal model scores 0.31, more than a tie below every other model.

| Model | Reference set |
| --- | --- |
| M1 (behind `code-v2`) | 0.57 |
| M2 (behind `rag-large-v1`) | 0.72 |
| M3 (behind `agents-v1`) | 0.70 |
| Open-weight candidate | 0.87 |

The candidate meets the safety bar and beats the best hosted model, M2, by 15 points, more than a tie, and its 2% critical-error rate is no higher than M2's: create an archetype. Likelihood is medium, because the 15-point lead is within a tie of the largest shortfall, the finance tool's 15 points (0.85 against 0.70), so both Business Owners are told the lead and the shortfall.

**Step 4.** The definition is the one filled in under [Templates](LLM_Platform_Archetype_Framework_Templates.md), shown there as it stands once the archetype is active. Context length is 1.2 × (18,000 + 400) = 22,080 tokens, rounded up to 32k.

**Step 5.** The candidate is released as `text-to-sql-v1`, with the analytics assistant as pilot. Its acceptance score is 0.86, a pass within a tie. Of 50 labeled live outputs, 44 are correct: the 95% Wilson interval is 0.76 to 0.94, and 0.86 is inside it. The archetype goes active.

**Version 2.** A new model scores 0.93 on the reference set against 0.87 for version 1: more than a tie. `text-to-sql-v2` is released on Aug 3, 2027, and version 1's end date is Nov 1. The analytics assistant scores 0.83 with its old prompt and 0.90 once it adopts the reference prompt, and moves in September. The finance tool scores 0.91 and moves in October. Version 1 then retires before its end date.

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
| 0.17 | Nov 13, 2026 | Step 1 also records the latency class and lengths, and the Business Owner re-approves the floor when the scoring method changes. Languages removed from the profile, the offer and the example. Coherence fixes: the completed profile is checked against the offer at joining; a use case that goes to Step 3 completes its needs first; the safety bar applies to the first archetype; joining is no longer listed as a change the archetype owner approves |
| 0.18 | Nov 17, 2026 | In Step 1 the Business Owner identifies the product's data and defines its classification in the profile, against the platform's one ceiling |
| 0.19 | Nov 17, 2026 | Acceptance harness removed: in scope, Developers score with the archetype's reference eval, and the acceptance record no longer names a harness |
| 0.20 | Nov 17, 2026 | Onboarding stage and gate numbers removed: the framework names its own events (join, select, pilot, release, move) and refers to the onboarding process as a whole. The G5 policy is renamed the selection policy |
| 0.21 | Nov 17, 2026 | GA replaced by release date |
| 0.22 | Nov 17, 2026 | Signals section removed. Two of its rules are kept: a critical-error incident where the model is at fault triggers a forced replacement, and a live product below its floor triggers gap diagnosis |
| 0.23 | Nov 17, 2026 | Known gaps section removed: interim practices, differences from the companion documents, planned work and open questions |
| 0.24 | Nov 17, 2026 | Quarterly review reduced to three tasks for the archetype owner: review the re-evaluation, update the archetype's model, and propose retiring or merging archetypes. The Platform Owner's review and the yearly profile reconfirmation are removed |
