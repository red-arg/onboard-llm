# LLM Platform: Model Onboarding Process

Oct 19, 2026 · @Zhi Feng

## Purpose and scope

The platform decides which open-weight models it hosts, and every product on the platform builds on that catalog. This process defines how a candidate model moves from proposal to production, and how it is eventually retired.

Three principles shape it:

- **Selection through archetypes.** Models are chosen for the use-case archetypes the platform serves (RAG, summarization, extraction, code, and so on), not for individual products.
- **Two layers of approval.** The platform approves a *model* as safe, legal and fit for named archetypes. Each product separately gets its *use case* approved when it goes to production.
- **Products depend on aliases, not models.** Products call archetype aliases such as `rag-default`, so the platform can swap models without forcing product changes.

In scope: open-weight LLMs and embedding models hosted on platform infrastructure, including new models, new major versions, fine-tunes and quantized variants. Out of scope: third-party hosted APIs, which follow vendor procurement.

## Process at a glance

&#91;embedded content: model onboarding flow · 12 stages, 8 gates\]

Legal and security reviews run side by side; everything else is sequential. Steps 0 to 10 are owned by the platform, step 11 by each product team, and scheduled re-evaluation feeds the next cycle.

## Roles and responsibilities

The platform model owner drives each onboarding end to end. The model review board is the only body that can admit a model to the catalog.

| Role | What they do in this process |
| --- | --- |
| Platform model owner | Runs the onboarding: intake, coordination, selection recommendation, model card. One named owner per model for its whole life. |
| ML evaluation team | Maintains the archetype eval suites and safety battery; supports the pilot product; runs safety, data-risk and performance tests; co-signs the eval report. |
| Security | Supply-chain vetting, artifact scanning, isolated load test, AI bill of materials sign-off. |
| Legal | License classification, obligations that flow down to products, export-control check. |
| Data governance / privacy | Sets the maximum data classification the model may process; reviews memorization and PII test results. |
| Platform SRE / infrastructure | Capacity, serving configuration, observability, runbook, on-call. |
| Model review board | Approves or rejects admission, replacement and retirement. Members: platform lead (chair), security, legal, data governance, risk. |
| Product teams | Propose archetype gaps; as pilot product, run the Stage 6 capability evaluation and co-sign G6; test in sandbox; get use cases approved before production. On the review board: Consulted, not approvers. |

## Foundation: set up once, reused by every onboarding

These must exist before the first model is onboarded. Without them, every evaluation is ad hoc and models cannot be compared.

1. **Archetype catalog.** A short, stable list of the use-case archetypes the platform serves, built from product teams' roadmaps. Starting point: Q&A / RAG, summarization, extraction and classification, code assistance, agents and tool use, embeddings and search, multilingual.
2. **Requirements per archetype.** Minimum quality bar, latency target, context length, languages, maximum data classification, and target cost per request.
3. **Eval suite per archetype.** Representative tasks on content like the platform's real workload: public benchmarks plus domain samples contributed by product teams (or synthetic data until those exist). Versioned, so scores stay comparable across models.
4. **Standard safety battery.** Jailbreak and prompt-injection resistance, harmful-content refusal, bias probes, memorization and PII leakage tests. Run on every model regardless of archetype.
5. **License policy.** Which license classes the platform accepts: commercial use allowed, internal use only, or not acceptable.
6. **Security baseline.** Approved sources, allowed file formats, scanning tools, and the isolated environment where untrusted weights are first loaded.
7. **Model registry and model card template.** One record per model version holding its status, owner, artifacts and approvals.
8. **Gateway with archetype aliases.** Authentication, logging, guardrails, rate limits, usage metering and a per-model kill switch, applied to every hosted model.

## Stage-by-stage flow

A model passes eight gates (G1 to G8) before products can use it in production. Legal and security reviews run in parallel; any failed gate stops the onboarding and records the reason in the registry.

### Stage 0: Trigger

An onboarding starts only from one of these:

- A product team reports an archetype gap: no hosted model meets the requirements, or a new archetype is needed.
- Horizon scanning finds a new open-weight release likely to beat an incumbent.
- The scheduled re-evaluation shows an incumbent falling behind.
- An incumbent must be replaced because of a vulnerability, license change or end of upstream support.

### Stage 1: Intake and screening (Gate G1)

- The platform model owner registers the candidate as **Proposed**: source, exact version and hash, license, parameter count, context length, languages, hardware needs, and target archetypes.
- Quick screen against archetype requirements: does it fit our hardware, context, language and license policy?
- **G1, worth evaluating:** platform model owner decides. Failing candidates are rejected with a reason, at near-zero cost.

### Stage 2: Legal and license review (Gate G2)

- Classify the license against policy: commercial use, internal only, or not acceptable.
- Record obligations that flow down to products, such as attribution, acceptable-use policies and user-count thresholds.
- Check known training-data and IP concerns, and export-control or regulatory constraints.
- **G2, legally usable:** Legal signs off with the license class and obligations.

### Stage 3: Security and supply chain (Gate G3)

Runs in parallel with Stage 2.

- **Check Hugging Face scan results first.** Review the Hub's security scans on the model page (malware, pickle and secrets scanning, plus JFrog and Protect AI results). Any flagged file stops the onboarding. A clean result is a first screen only: crafted files have evaded these scanners before.
- Download only from the verified publisher, pinned to an exact revision and hash, into a quarantine registry.
- **Safetensors is mandatory.** All weights must be in safetensors format. Pickle-based files (`.bin`, `.pt`, `.pth`, `.pkl`, `.ckpt`) are rejected. If the publisher ships no safetensors version, the model fails G3 unless Security converts and verifies the weights in the isolated environment.
- Review any custom code. `trust_remote_code` stays disabled unless that code passes review.
- Produce an AI bill of materials.
- Load test in an isolated environment with no network egress.
- **G3, safe to host:** Security signs off. The artifact moves from quarantine to the internal model registry.

### Stage 4: Garak model scan (Gate G4)

[NVIDIA's garak](https://github.com/NVIDIA/garak) is an open-source LLM vulnerability scanner. It attacks the model with a library of known techniques, including prompt injection, jailbreaks, encoding tricks, data leakage, toxicity and hallucination, and records which attacks succeed.

- **When:** straight after G3, against the registry copy served in the isolated environment.
- **Pinned configuration:** the garak version and the probe and detector selection are pinned in the internal benchmarking project, so results compare across models. New model families get the full probe set; fast-track versions get a pinned subset.
- **Set up for our network:** our servers cannot reach the internet, and some garak probes and detectors download datasets or models on first use. Mirror garak and everything it downloads internally and pre-install it. Point any probe that needs an attacker model at an internally hosted model.
- **Pass criteria:** Security sets a maximum failure rate per probe family and compares results with the incumbent. A higher failure rate than the incumbent on prompt-injection or jailbreak probes needs Security's review.
- **Records:** garak's report and hit log are attached to the registry entry.
- **Scope:** garak tests the bare model against known attacks. It does not replace the trifecta check at Stage 11 or product-level testing.
- **G4, scan passed:** Security signs off the garak results. Models that fail are rejected before any benchmarking.

### Stage 5: Internal benchmarking (Gate G5)

Every candidate runs through the internal benchmarking project before any product team spends time on it. The project holds the platform's versioned benchmark harness, the archetype eval suites and the standard safety battery from the Foundation section, so every model is scored the same way and results stay comparable over time.

The harness is built on four open-source tools, each covering a different part of the benchmark:

| Tool | What it measures | Used for |
| --- | --- | --- |
| [GuideLLM](https://github.com/vllm-project/guidellm) | Load tests the served endpoint across request rates, from a single user up to saturation: time to first token, inter-token latency, request latency and throughput | Operational baseline |
| [LightEval](https://github.com/huggingface/lighteval) | Runs public benchmarks (knowledge, reasoning, math, code, multilingual) and custom tasks with the same prompts and scoring for every model | Public-benchmark part of each archetype suite |
| [DeepEval](https://github.com/confident-ai/deepeval) | Scores outputs on our own test cases with task metrics and LLM-as-judge metrics: RAG faithfulness and relevance, hallucination, summarization, JSON and tool-call correctness, bias and toxicity | Domain slices of each archetype suite; bias and toxicity in the safety battery |
| [MTEB](https://github.com/embeddings-benchmark/mteb) | Embedding quality across retrieval, reranking, classification, clustering and semantic similarity, in many languages | Embeddings and search archetype |

- **Isolated and product-free:** runs on the registry copy in a non-production environment. No product team is involved yet.
- **Capability:** each target archetype's eval suite, including its domain slices, scored against the archetype requirements and the incumbent. LightEval covers the public benchmarks and DeepEval the domain slices; embedding models run MTEB in place of both, on the task types and languages the archetype needs.
- **Safety and data risk:** the standard safety battery, red-teaming, and memorization and PII tests, reviewed by data governance.
- **Operational baseline:** a GuideLLM sweep for latency, throughput and memory on target hardware, including any quantized variants, with prompt and output lengths matched to each archetype. Cost per request is derived from throughput at the latency target.
- **Pinned configuration:** tool versions, task lists, metrics, thresholds and the judge model are pinned in the benchmarking project. Changing any of them creates a new suite version, and scores are compared only within the same version.
- **Set up for our network:** as with garak, mirror each tool and everything it downloads (benchmark datasets, MTEB tasks, tokenizers) internally and pre-install it. DeepEval's judge metrics need a judge model: point them at a fixed, internally hosted model, never the candidate itself, and check judge scores against human labels on a sample before relying on them. Turn off DeepEval telemetry (`DEEPEVAL_TELEMETRY_OPT_OUT=YES`) and do not connect it to the vendor's cloud.
- **Published results:** scores are recorded in the benchmarking project and the registry entry, with each tool's raw output attached, so product teams and the board can compare models.
- **G5, worth piloting:** the ML evaluation team signs off that the model meets the safety bar and the requirements of at least one target archetype, and matches or beats the incumbent there. Models that fail are rejected without using a pilot team's time.

### Stage 6: Pilot evaluation (Gate G6)

A pilot product evaluates the model on its real use case, so selection rests on actual work rather than benchmarks alone. Garak, benchmark, safety and data-risk results come from Stages 4 and 5.

- **Choose the pilot:** the platform model owner picks one product per target archetype, usually the team that reported the gap. The pilot commits a named owner, representative data within a classification limit agreed with data governance, and success criteria set before testing starts.
- **Run outside production:** the pilot integrates the registry copy in staging through a pilot-only gateway alias. No production traffic or end users until G8.
- **Capability, run by the pilot:** task success against the agreed criteria, compared with the incumbent on the same tasks.
- **Operational, run by the platform:** latency, throughput, memory and cost per request on target hardware, using the pilot's traffic pattern.
- **G6, meets the bar:** the pilot product owner confirms fit for its use case, and the platform confirms performance under the pilot's traffic. Both sign the eval report.

One pilot can tilt the catalog toward one product's needs. When an archetype serves several products, run two pilots or give the Stage 5 benchmark results equal weight in Stage 8.

### Stage 7: Model review board (Gate G7)

- The board reviews the legal memo, security report and eval report.
- Decision: approve (with conditions if needed), reject, or send back for more evidence.
- **G7, admitted to the catalog:** status becomes **Approved** in the registry.

### Stage 8: Selection decision

For a board-approved model, the platform model owner decides how it enters the catalog, weighing quality against GPU cost and catalog size:

- **Add:** fills a gap or a new archetype.
- **Replace:** clearly beats the incumbent on an archetype; the incumbent moves to deprecation.
- **Add as fallback:** for resilience or a specialized need within an archetype.
- **Reject:** no meaningful gain over the current catalog.

The owner drafts the model card: approved archetypes, maximum data classification, license class and obligations, eval results, known weaknesses.

### Stage 9: Production readiness (Gate G8)

- Serving configuration, capacity plan and autoscaling.
- Gateway integration: alias mapping, guardrails, rate limits, metering, kill switch.
- Dashboards and alerts for latency, errors, safety-filter hits and usage.
- Runbook, rollback plan and on-call ownership.
- Load test at expected peak.
- **G8, ready to serve:** SRE and the platform model owner sign off.

### Stage 10: Release

1. **Sandbox:** opt-in access for product teams, non-sensitive data only. Collect feedback and real-world issues.
2. **General availability:** published in the catalog and routing guide with its model card.
3. **If replacing an incumbent:** shadow traffic, then a canary on the alias, then full cutover. Announce the incumbent's deprecation date to affected products.

### Stage 11: Product adoption

This is the second approval layer, owned by each product team.

- A product requests production access to an archetype alias, describing its use case and the data it will send.
- The platform checks the use case against the model card (archetype, data classification, license obligations).
- Higher-risk use cases, such as customer-facing, regulated data or automated decisions, go to risk and legal review.
- The product passes the trifecta check (see Lethal trifecta and network isolation). Production credentials are issued only after approval. Each product runs its own application-level evaluation.

## Lethal trifecta and network isolation

The [lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) (Simon Willison, June 2025) names the combination that lets a prompt injection steal data. When an LLM system has access to private data, is exposed to untrusted content, and can communicate externally, an attacker can plant instructions in the content that make it read the data and send it out. Removing any one leg breaks the attack. The platform removes the external-communication leg at the hosting layer; every product must keep it removed.

&#91;embedded content: lethal trifecta · hosting layer removes one leg\]

### The three legs on our platform

| Leg | Where it appears | Status at the hosting layer |
| --- | --- | --- |
| Access to private data | Prompts, retrieved documents and tool results that products send to models | Present by design: products send internal data to models |
| Exposure to untrusted content | Customer messages, inbound emails, uploaded files, tickets and outside documents fed in through retrieval | Present whenever a product feeds outside-sourced content. Network isolation does not remove it: injected instructions arrive inside internal data |
| Ability to communicate externally | Web search, outbound requests, tools that send data out | **Removed:** model servers cannot search the web or reach beyond the internal subnet |

### Why model hosting conforms

- **No outbound path.** Model servers have no web search and no network route beyond the internal subnet, so an injected instruction cannot make a hosted model fetch a URL or send data to an outside server.
- **Models only generate text.** A hosted model cannot act on its own. Any action happens in a product's code and tools, outside the hosting layer.
- **Weights cannot call out.** No egress, plus the safetensors-only rule in Stage 3, means model files cannot run code that contacts the outside.
- **Internal-only gateway.** The gateway that fronts every model is reachable only from the internal subnet.

### Where the leg can come back

Isolating the model servers covers the hosting layer only. A product built on top can reintroduce external communication, so the check continues at Stage 11:

- **Outbound tools.** Sending email (internal mail servers relay outside), posting to chat, webhooks, or creating tickets, files or pull requests that outside parties can see.
- **Rendering in the user's browser.** If a product renders model output as markdown or HTML, an injected image or link can carry data in its URL to an outside server when the browser loads it. The browser sits outside the server subnet's restrictions.
- **Internal oversharing.** Data from one user's access scope written where another internal user can read it. Network isolation does not stop this; it is the same pattern inside the organization.
- **Hosting outside the subnet.** Products or integrations that run outside the internal subnet.

### Controls

**Platform, for every model:**

- Model servers and the gateway stay without internet egress. Any exception needs Security approval and a Stage 9 readiness review.
- The gateway output filter removes or flags external URLs and markdown images in model output by default.
- The platform grants models no tools. Tools live in products, where Stage 11 can review them.

**Product, at Stage 11 (trifecta check):**

- The product declares its private data sources, untrusted content sources and outbound actions.
- If all three legs are present, the product removes one before approval: drop the outbound action, require a person to confirm every outbound action, or keep untrusted content out of any context that holds private data.
- Model output is rendered as plain text or sanitized markdown with external URLs blocked.
- Prompt-injection filters and guardrails are an extra layer only. They miss some attacks, so they never count as removing a leg.

## Gap diagnosis: when a product misses its acceptance criteria

When a business owner reports that hosted models don't meet their acceptance criteria, the platform runs this diagnosis before anyone concludes the model is at fault. It separates three possible causes (the prompt, the hosted catalog, or the limits of current models) by changing one variable at a time on the same test set.

It runs when:

- A new project's domain test fails during Stage 11.
- A live product reports quality falling short of its criteria.
- A business owner asks for a new model or an exception to the catalog.

### Step 1: Confirm the criteria and the test set

- **Acceptance criteria** are written by the business owner in business terms, with every error type given a severity (critical, moderate or minor). The ML evaluation team translates them into measurable metrics.
- **Domain test set:** real samples with expected answers, around 100 to 300 to start.
- **Label check:** two people label a sample independently. If they often disagree, fix the criteria or labels first, because no model can meet a bar that people can't agree on.
- **Baseline:** score the current process (people, rules or a legacy system) on the same test set, so results can be compared with today.

### Step 2: Run the 2×2

Four runs on the same test set, varying one thing at a time:

|  | Business owner's prompt | Optimized prompt |
| --- | --- | --- |
| **Best hosted model** | Starting point | Does better prompting close the gap? |
| **Reference model** | Does a stronger model alone close the gap? | The ceiling of current models |

- **Best hosted model:** the best-scoring model and tier in the archetype, not just the default.
- **Optimized prompt:** written by the ML evaluation team within a fixed time box (suggested 3 to 5 working days), using structured output, worked examples, task decomposition and better retrieval. If scores plateau across techniques, stop.
- **Reference model:** the strongest model available anywhere, including a frontier API if no open model is stronger. Use only non-sensitive or synthetic samples for these runs, as approved by data governance.

### Step 3: Classify the failures

Scores show whether there's a gap; failure types show why. Classify every failed sample, and use the dominant type to confirm the diagnosis.

| Failure type | Likely cause |
| --- | --- |
| Ignored instructions, wrong format | Prompt |
| Missing facts or context | Retrieval or data, not the model |
| Reasoning or accuracy errors that persist across prompts and models | Model capability |
| Expected answer is disputable | Test set or criteria |

### Step 4: Decide

| Result | Diagnosis | Action |
| --- | --- | --- |
| Hosted model passes with the optimized prompt | Prompt problem | Product adopts the optimized prompt. The platform keeps it as a reference prompt for the archetype. |
| Hosted model passes only at a higher tier | Wrong tier | Product moves to that tier through Stage 11. |
| Only the reference model passes | Catalog gap | Platform model owner logs a Stage 0 trigger. Options, cheapest first: domain adapter, new candidate model, new domain archetype if two or more products share the need. |
| Nothing passes, even the reference model with the optimized prompt | Beyond current models | Redesign the workflow (Step 5). |

### Step 5: When nothing passes

The bar stays anchored to business need; it is not lowered to fit the models. Instead, the business owner and the platform redesign the workflow:

- Split the task into smaller steps that models do reliably.
- Let the model abstain on uncertain cases and route them to a person. Criteria then apply to the answered cases, plus a maximum abstention rate.
- Narrow the scope to the subset that passes, and handle the rest manually.
- Park the use case and re-test it at the next scheduled re-evaluation.

### Roles

- **Business owner:** criteria, severities, labels, and the final call on any redesign.
- **ML evaluation team:** metric translation, optimized prompt, the four runs, failure classification and the gap diagnosis report.
- **Platform model owner:** catalog actions, including any Stage 0 trigger.
- **Data governance:** approves the samples used for reference-model runs.

## Registry status and lifecycle

Every model version carries exactly one status in the registry, and only Sandbox and GA models are reachable through the gateway.

| Status | Meaning | Who can use it |
| --- | --- | --- |
| Proposed | Registered at intake, under review | Nobody |
| Rejected | Failed a gate; reason recorded | Nobody |
| Approved | Passed the review board, not yet serving | Nobody |
| Sandbox | Serving, limited support | Opted-in teams, non-sensitive data only |
| GA | Serving with SLA and support | Products with an approved use case |
| Deprecated | Still serving, sunset date announced | Existing products only, while they migrate |
| Retired | Removed from serving; artifacts archived | Nobody |

**Ongoing obligations while a model is in Sandbox or GA:**

- Monitor quality, latency, safety-filter hits and incidents.
- Re-run archetype eval suites on a fixed schedule, for example quarterly, alongside new candidates.
- Watch upstream for vulnerabilities, license changes and end of support. A critical vulnerability can trigger the kill switch.
- Give products a fixed notice period before retirement, with the replacement alias named in the deprecation notice.

**Changes that need a new onboarding:** a new major version, a fine-tune, a new quantized variant, or a request to extend the model to a new archetype or a higher data classification.

## Records produced

Each gate leaves a record in the registry entry, so any model's admission can be audited later.

| Record | Produced at | Owner |
| --- | --- | --- |
| Intake record | Stage 1 | Platform model owner |
| License memo (class + obligations) | Stage 2 | Legal |
| Security report + AI bill of materials | Stage 3 | Security |
| Garak report and hit log | Stage 4 | Security |
| Benchmark report (archetype suites, safety, data risk, operational baseline) with GuideLLM, LightEval, DeepEval and MTEB outputs | Stage 5 | ML evaluation team |
| Eval report (archetypes, safety, data risk, performance) | Stage 6 | Pilot product, ML evaluation team |
| Board decision record | Stage 7 | Model review board |
| Model card | Stage 8 | Platform model owner |
| Runbook + readiness checklist | Stage 9 | Platform SRE |
| Routing guide update + release notes | Stage 10 | Platform model owner |
| Use-case approval | Stage 11 | Product team, platform |
| Gap diagnosis report (criteria, 2×2 scores, failure breakdown, decision) | Gap diagnosis | ML evaluation team, business owner |
| Deprecation notice | Lifecycle | Platform model owner |

## Timelines and fast track

Set target durations per stage so the process doesn't become a bottleneck that pushes teams to work around it. The figures below are suggested starting points; adjust them after the first few onboardings.

| Path | Applies to | Stages run | Suggested target |
| --- | --- | --- | --- |
| Full onboarding | New model family or new publisher | All stages, 0 to 10 | 4 to 6 weeks, intake to GA |
| Fast track | Minor or patch version of an approved model, same license and architecture | Security (Stage 3), garak scan (Stage 4), eval regression on approved archetypes (Stage 5), readiness (Stage 9); owner and security sign-off replace the board | 1 to 2 weeks |
| Emergency replacement | Critical vulnerability in a GA model | Kill switch or rollback first; replacement follows fast track if available, else full onboarding | Mitigation same day |

The board should meet on a fixed cadence, for example every two weeks, so approvals don't wait on scheduling.

### Open questions

- [ ] Who chairs the model review board, and how often does it meet?
- [ ] Which archetypes do we start with, and which product teams contribute eval samples?
- [ ] What notice period do products get before a model is retired?
- [ ] Which license classes does Legal accept?
