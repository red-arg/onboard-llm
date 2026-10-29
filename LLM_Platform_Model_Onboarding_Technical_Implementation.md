# LLM Platform: Model Onboarding Technical Implementation

Oct 29, 2026 · @Zhi Feng

## Purpose and scope

This document describes how the [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) is implemented on the platform's internal Nexus repository and DevOps pipeline, so that every onboarding is automated where it can be, and auditable and traceable end to end.

It covers the automated stages (intake, security scanning, garak, internal benchmarking, deployment and go-live), how manual records from the other stages enter the same audit trail, and how anyone can later answer the question: *why is this model serving this alias, and who approved it on what evidence?*

Assumptions:

- Servers, pipeline agents and model servers have no internet access. Only the controlled acquisition step can reach Hugging Face.
- Nexus is the only artifact store: packages, container images, model weights, datasets and evidence.
- The DevOps pipeline runs on self-hosted agents inside the network, including GPU agents, and supports manual approval steps, a secret store, and one pipeline triggering another.

## Design principles

- **One identity.** Every artifact, run, result and approval refers to the model's content digest, never to its name alone.
- **Pin everything.** Every run records the exact inputs that produced it, so it can be re-run and its results compared.
- **Keep everything.** Raw outputs of every run are kept, failed runs included, and nothing is overwritten.
- **Automation drafts, people decide.** Pipelines compute a draft verdict against versioned policy. Automation may reject on a hard failure; only a person can approve a gate.
- **Separation of duties.** Only pipeline service accounts write to the approved-model and evidence repositories. People approve; they do not edit evidence.

## Architecture overview

```
 Hugging Face ──► acquire job ──► Nexus: models-quarantine ──► Stage 3 scans ──► G3 ──► Nexus: models-approved
                  (only step with                                                              │
                   internet access)                                                            ▼
                                                     Stage 4 garak ──► G4 ──► Stage 5 benchmarks ──► G5
                                                          │                         │
                                                          ▼                         ▼
                                            Nexus: evidence  (manifests, raw outputs, results, verdicts, approvals, signatures)
                                                          ▲
 Git: onboarding-config  (suites, policies, pipelines) ───┘──► Git: model-registry ──► Git: gateway-aliases ──► dev ──► production
```

## Nexus repository layout

| Repository | Format | Deployment policy | Written by | Contents |
| --- | --- | --- | --- | --- |
| `pypi-group` | PyPI (proxy + hosted) | Disable redeploy (hosted) | Tooling pipeline | Pinned packages for garak, LightEval, DeepEval, MTEB, GuideLLM, scanners and their dependencies |
| `docker-tools` | Docker (hosted) | Disable redeploy | Tooling pipeline | Tool images and the serving image, always pulled by digest |
| `hf-proxy` (optional) | Hugging Face (proxy) | n/a | Nexus | Upstream cache for the acquire job only, if your Nexus version supports it and has controlled upstream access |
| `models-quarantine` | Raw (hosted) | Disable redeploy | Acquire job | Downloaded weights before G3 |
| `models-approved` | Raw (hosted) | Disable redeploy | Promotion job only | Weights after G3; the only source for any serving environment |
| `eval-data` | Raw (hosted) | Disable redeploy | Data-mirror pipeline | Benchmark datasets, MTEB tasks, tokenizers, garak resources, domain test sets |
| `evidence` | Raw (hosted) | Disable redeploy | Pipeline service accounts only | Everything the audit trail needs (see Evidence bundle) |

Repository rules:

- **Disable redeploy** on every hosted repository, so a path once written cannot be overwritten. New content always goes to a new path.
- **Delete privileges** on `models-approved`, `eval-data` and `evidence` are held by no person or pipeline in day-to-day roles. Retention and retirement deletes run through a reviewed pipeline that records what it removed.
- **Content selectors** restrict read access to domain test sets by data classification.
- **Nexus audit logging** is enabled and exported to the central log store, so repository and privilege changes are themselves traceable.
- **Blob store backups** cover `models-approved` and `evidence`.

## Model identity

A model version is identified by its upstream coordinates and a content digest computed the same way everywhere:

```
cd <model-dir>
LC_ALL=C find . -type f ! -name files.sha256 -print0 | LC_ALL=C sort -z | xargs -0 sha256sum > files.sha256
sha256sum files.sha256        # the model digest
```

- `files.sha256` is stored with the weights in Nexus and in the evidence bundle.
- The **model digest** (`sha256:...` of `files.sha256`) is the key for the registry entry, the evidence path and every manifest.
- Every job that loads weights recomputes the digest and refuses to continue on a mismatch: scanning, garak, benchmarking and every serving start.

## Git repositories

| Repository | Contents | Who changes it |
| --- | --- | --- |
| `onboarding-config` | Pipeline definitions, suite definitions, policies (pass criteria), tool image references, normalizer and policy-check code | Reviewed pull requests; ML evaluation team and Security own their suites and policies |
| `model-registry` | One YAML record per model version: status, owner, gate records and evidence links | Pipeline service account only, on a protected branch; every commit references the pipeline run that made it |
| `gateway-aliases` | Alias-to-model-digest mapping and serving configuration per environment | Reviewed pull requests, merged only by the deploy pipeline after its checks pass |

Git history of `model-registry` is the status history of every model, and git history of `onboarding-config` is the history of every threshold change.

## Evidence bundle

All evidence for a model version lives under one path in the `evidence` repository:

```
evidence/<model-digest>/
  intake/<run-id>/
  stage2-legal/<run-id>/          license memo (uploaded through the record pipeline)
  stage3-security/<run-id>/       hub scan response, local scan reports, format check, AIBOM, load-test log
  stage4-garak/<run-id>/
  stage5-benchmark/<run-id>/
  stage6-pilot/<run-id>/          eval report (uploaded)
  stage7-board/<run-id>/          board decision record (uploaded)
  stage8-selection/<run-id>/      model card
  stage9-readiness/<run-id>/
  stage10-adoption/<product>/<run-id>/
  stage11-go-live/<product>/<run-id>/
```

Each run directory holds:

| File | Purpose |
| --- | --- |
| `manifest.yaml` | Every input that produced the run, written before the run starts |
| `raw/` | Each tool's native output, unmodified |
| `results.jsonl` | Normalized scores (see Results, policy and verdicts) |
| `verdict.yaml` | Draft gate verdict computed by the policy check, for gate stages |
| `approval.yaml` | The approver's decision, for gate stages |
| `logs/` | Pipeline job logs, so evidence outlives the pipeline's own log retention |
| `SHA256SUMS` | Digest of every file in the run directory |
| `SHA256SUMS.sig` | Signature over `SHA256SUMS` |

Run IDs take the form `<stage>-<yyyymmddThhmmss>-<pipeline-run-number>` and are never reused.

### Run manifest

```yaml
run_id: stage5-20261102T091500-4812
stage: 5
pipeline_run: <pipeline URL or ID>
trigger: registry status change G4 -> pass (commit 3f9c2e1)
model:
  name: <publisher>/<model>
  upstream_revision: <hub commit hash>
  digest: sha256:...
suite: rag/v2                             # from onboarding-config
config_commit: 8d41a07                    # onboarding-config commit used
images:
  lighteval: nexus.internal/docker-tools/lighteval@sha256:...
  deepeval:  nexus.internal/docker-tools/deepeval@sha256:...
  guidellm:  nexus.internal/docker-tools/guidellm@sha256:...
  serving:   nexus.internal/docker-tools/vllm@sha256:...
data:
  - path: eval-data/rag-domain/v2/test.jsonl
    sha256: ...
judge_model:
  digest: sha256:...
  endpoint: http://judge.eval.internal/v1
  temperature: 0
serving_config_sha256: ...
hardware: { gpu: <model>, count: 2, driver: <version> }
seeds: { global: 1234 }
```

## Pipelines

| Pipeline | Process stage | Trigger | Gate approver |
| --- | --- | --- | --- |
| P0 `tooling` | Foundation | Change to tool versions or mirrored data in `onboarding-config` | Reviewed pull request |
| P1 `intake` | Stage 1 | Platform model owner submits a candidate | Platform model owner (G1) |
| P2 `acquire-and-scan` | Stage 3 | G1 pass | Security (G3) |
| P3 `garak` | Stage 4 | G3 pass | Security (G4) |
| P4 `benchmark` | Stage 5 | G4 pass | ML evaluation team (G5) |
| P5 `record` | Stages 2, 6, 7, 8, 10 | Owner uploads a record | The record's owner (G2, G6, G7) |
| P6 `deploy` | Stages 9 and 11 | Pull request to `gateway-aliases` | SRE and platform model owner (G8); go-live sign-off |
| P7 `re-evaluate` | Lifecycle | Schedule | Reuses P3 and P4 with the current suites |

Every pipeline follows the same pattern:

1. Write `manifest.yaml` to the evidence path before any work starts.
2. Run the jobs, pulling images and data from Nexus by digest and verifying the model digest on load.
3. Upload raw outputs and logs, whether the run passed or failed.
4. Normalize results and run the policy check to produce `verdict.yaml`.
5. Pause at a manual approval step for the gate owner.
6. Write `approval.yaml`, `SHA256SUMS` and the signature.
7. Commit the gate record to `model-registry` and queue the next pipeline.

### P0: tooling

- Builds each tool image from a lock file installed with `pip install --require-hashes` from `pypi-group`, and pushes it to `docker-tools`. Manifests always reference images by digest, not tag.
- Mirrors external datasets, MTEB tasks, tokenizers and garak resources into `eval-data` at a new versioned path, recording each file's sha256 in `onboarding-config`.
- Tool images run with network egress denied except to Nexus, the model under test and the judge model, plus these environment settings:

| Setting | Why |
| --- | --- |
| `HF_HUB_OFFLINE=1`, `HF_DATASETS_OFFLINE=1` | Hugging Face libraries fail fast instead of trying to download |
| `HF_HOME` pointing at a cache populated from `eval-data` | Tools find mirrored datasets and tokenizers locally |
| `DEEPEVAL_TELEMETRY_OPT_OUT=YES` | Stops DeepEval sending telemetry |

### P1: intake

- The platform model owner submits upstream coordinates, the pinned revision, license, target archetypes and hardware needs.
- The pipeline checks the submission against archetype requirements in `onboarding-config` and creates the registry record with status **Proposed**.
- G1 is a manual approval step in the pipeline.

### P2: acquire and scan (Stage 3)

The only job with access to Hugging Face, through `hf-proxy` or a dedicated egress-allowed agent.

1. Fetch the Hub's security scan status for the pinned revision and store the response. Any flagged file fails the run.
2. Download the pinned revision only, compute `files.sha256` and the model digest, and upload to `models-quarantine`.
3. **Format check:** every file must match the allowlist in the security baseline. Any pickle-based file (`.bin`, `.pt`, `.pth`, `.pkl`, `.ckpt`) fails the run.
4. **Local scans:** run the security baseline's scanners (for example ModelScan) on the quarantine copy.
5. **Custom code:** flag any Python file in the repository for Security review; `trust_remote_code` stays off unless that review passes.
6. **AI bill of materials:** generate a CycloneDX ML-BOM from the model files, config and upstream metadata.
7. **Isolated load test:** load the weights on an agent with no egress and record the log.
8. **G3 approval** by Security. On approval, the promotion job copies the files to `models-approved`, re-verifies the digest, and sets the registry record's `artifact` field to the approved path.

### P3: garak (Stage 4)

- Deploys the model from `models-approved` on the isolated eval serving environment with the pinned serving image and configuration.
- Runs garak from its pinned image with the suite's probe and detector selection, generations per probe and seed, against the OpenAI-compatible endpoint. Probes that need an attacker or judge model point at an internally hosted model recorded in the manifest.
- Keeps garak's report, hit log and HTML report in `raw/`.
- The normalizer computes failure rates per probe family; the policy check compares them with Security's limits and the incumbent's latest result on the same suite version.

### P4: benchmark (Stage 5)

Runs one job per tool in parallel, against the same deployment and manifest:

| Job | Runs for | Keeps in `raw/` |
| --- | --- | --- |
| LightEval | LLM candidates | Aggregate results and per-sample details for every task |
| DeepEval | LLM candidates | Per-test-case results, including every judge prompt, score and reason |
| MTEB | Embedding candidates | Per-task result files |
| GuideLLM | All candidates | Benchmark output for the full sweep, plus the load profile used |
| Safety battery | All candidates | Memorization and PII test outputs, for data governance review |

- DeepEval's judge is the internally hosted judge model in the manifest, at temperature 0, never the candidate itself.
- GuideLLM runs a sweep with prompt and output lengths set per archetype in the suite.
- The policy check compares each criterion with the archetype requirements and with the incumbent on the same suite version. When the suite version is newer than the incumbent's latest result, the pipeline re-runs the incumbent first.

### P5: record (manual stages)

Stages 2, 6, 7 and 8, and the use-case approval in Stage 10, produce documents rather than tool output. The record pipeline takes the uploaded document, writes it into the evidence path with a manifest naming the uploader and the gate, signs it, and commits the gate record to the registry. The manual approval step is assigned to the gate owner from the process document (Legal for G2, the pilot product owner and platform for G6, the board chair for G7).

### P6: deploy (Stages 9 and 11)

- **Dev (Stage 9):** a pull request to `gateway-aliases` maps the alias to the model digest for dev. The pipeline checks that the registry status is **Approved**, deploys from `models-approved`, verifies the digest at serving start, runs the readiness checks including a GuideLLM load test at expected peak, and pauses for G8. On approval, status becomes **Dev**.
- **Production (Stage 11):** the pipeline promotes the exact dev deployment. It refuses to run if the model digest, serving image digest or serving configuration hash differ from what G8 approved. It runs production smoke tests and kill-switch and rollback checks, pauses for the go-live sign-off, then sets status to **GA** on a model's first go-live.
- Production credentials for a product are issued only by this pipeline, and only for a use case with an approval record from Stage 10.

## Results, policy and verdicts

### Normalized results

The normalizer maps every tool's output to one row format, so results compare across tools, models and runs:

```json
{"run_id": "stage5-20261102T091500-4812", "model_digest": "sha256:...", "suite": "rag/v2",
 "tool": "deepeval", "task": "rag-domain", "metric": "faithfulness",
 "value": 0.88, "n": 300, "ci95": [0.85, 0.91]}
```

### Policy

Pass criteria live in `onboarding-config` as versioned files, one per gate and suite:

```yaml
policy: g5-rag
version: 4
suite: rag/v2
criteria:
  - id: faithfulness-floor
    metric: deepeval/rag-domain/faithfulness
    rule: value >= 0.85
    hard: true                     # failing a hard criterion lets automation reject
  - id: faithfulness-vs-incumbent
    metric: deepeval/rag-domain/faithfulness
    rule: ci95.upper >= incumbent.value
  - id: ttft-p95
    metric: guidellm/rag-profile/ttft_ms_p95
    rule: value <= 800
```

Changing a threshold is a reviewed pull request by the gate owner, so every threshold change has an author, reviewer and date.

### Verdicts

```yaml
gate: G5
model_digest: sha256:...
policy: g5-rag@4
runs:
  - run_id: stage5-20261102T091500-4812
    sha256sums: sha256:...
outcome: pass | fail | needs-review
criteria:
  - id: faithfulness-floor
    measured: 0.88
    threshold: ">= 0.85"
    outcome: pass
generated_at: 2026-11-02T11:42:10Z
```

- **fail** on a hard criterion: the pipeline records the model as **Rejected** with the failed criteria as the reason. The gate owner can override with a written reason.
- **pass** or **needs-review**: the pipeline pauses for the gate owner. A pass is a recommendation, not an approval.
- **No best-of-N.** The verdict lists every completed run for that model, gate and policy version. Re-runs are allowed only for infrastructure failures, and the reason is recorded in the new run's manifest.

## Approvals and signatures

The pipeline's manual approval step authenticates the approver through single sign-on. The pipeline then writes:

```yaml
gate: G5
verdict_sha256: sha256:...          # the exact verdict the approver saw
decision: approve | reject | send-back
approver: <user id>
role: ML evaluation team
pipeline_run: <pipeline URL or ID>
decided_at: 2026-11-02T14:05:33Z
reason: <required when the decision differs from the verdict>
```

- The pipeline rejects an approval where the approver also started the run or authored the last policy change used in it.
- Each run directory's `SHA256SUMS` is signed with a key held in the pipeline's secret store (for example `cosign sign-blob` with `--tlog-upload=false`, since no public transparency log is reachable). The public key is published in `onboarding-config`, so anyone can verify a bundle.
- Signing proves the evidence has not changed since the pipeline wrote it. The approver's identity comes from the pipeline's single sign-on record, which is copied into `approval.yaml` and the run's logs.

## Registry record

```yaml
model: <publisher>/<model>
upstream_revision: <hub commit hash>
digest: sha256:...
artifact: models-approved/<publisher>/<model>/<digest>/
status: Approved
owner: <platform model owner>
archetypes: [rag]
max_data_classification: internal
gates:
  G3: { decision: approve, by: <user>, at: 2026-10-30, evidence: evidence/sha256:.../stage3-security/stage3-20261030T101200-4790/ }
  G4: { decision: approve, by: <user>, at: 2026-10-31, evidence: evidence/sha256:.../stage4-garak/stage4-20261031T083000-4801/ }
  G5: { decision: approve, by: <user>, at: 2026-11-02, evidence: evidence/sha256:.../stage5-benchmark/stage5-20261102T091500-4812/ }
runs:                                 # every run, including failed and re-run ones
  - stage5-20261101T160000-4808       # infrastructure failure: GPU node lost
  - stage5-20261102T091500-4812
```

## Answering audit questions

| Question | Where the answer is |
| --- | --- |
| Why is `rag-default` serving this model? | `gateway-aliases` history → model digest → registry record → gate evidence paths |
| Who approved G4, and what did they see? | `approval.yaml` in the G4 run, and the verdict it names by digest |
| Were these results changed after approval? | Verify `SHA256SUMS.sig`, then `SHA256SUMS` against the files |
| Were the pass criteria changed to let a model through? | Policy version in the verdict, then `onboarding-config` history for that policy file |
| Was a failed run hidden? | `runs` in the registry record and the verdict's run list, against the evidence folder listing |
| A mirrored dataset is found to be wrong. Which decisions relied on it? | Search all manifests in `evidence` for the dataset's sha256 |
| Is production running exactly what was approved? | Digest check at serving start, plus P6's comparison of production and the G8-approved deployment |
| Can we reproduce this result? | Re-run with the same manifest: same images, data, model and judge digests |

## Reproducibility limits

Pinning removes most variation, but not all of it. Policies should allow for what remains:

- **LLM-as-judge scores** vary slightly even at temperature 0. Policies compare confidence intervals, not single values, and high-stakes DeepEval metrics run more than once.
- **garak** failure rates depend on generations per probe, which is fixed in the suite version.
- **GuideLLM** results depend on hardware and the serving stack. They are compared only across runs with the same recorded hardware and serving image.
- **Public benchmarks** in LightEval and MTEB may be in a model's training data, so domain slices carry more weight at G5.

## Open decisions

- [ ] How long is evidence kept after a model is retired, and where do expired bundles go?
- [ ] Who holds the signing key, and how is it rotated?
- [ ] Is the pipeline's single sign-on record enough for approvals, or must approvers sign personally?
- [ ] Is the registry Git-backed as described, or does an existing tool (for example MLflow) hold it?
- [ ] Does our Nexus version support Hugging Face proxy repositories, or does acquisition run on a dedicated egress agent?
