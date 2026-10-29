# LLM Platform: Model Onboarding Technical Implementation

Oct 29, 2026 · @Zhi Feng

## Purpose and scope

This document describes how the [model onboarding process](LLM_Platform_Model_Onboarding_Process.md) is implemented on the platform's internal Quay and Nexus repositories and DevOps pipeline, so that every onboarding is automated where it can be, and auditable and traceable end to end.

It covers the automated stages (intake, security scanning, garak, internal benchmarking, deployment and go-live), how manual records from the other stages enter the same audit trail, and how anyone can later answer the question: *why is this model serving this alias, and who approved it on what evidence?*

Assumptions:

- Servers, pipeline agents and model servers cannot reach the internet. Everything from outside arrives through one of two proxies:
  - **Quay** proxies container images from upstream registries (Docker Hub, GitHub Container Registry, NVIDIA NGC and others).
  - **Nexus** proxies Python packages from PyPI, models and datasets from Hugging Face, and a short list of other approved hosts.
- Quay holds every container image the platform runs. Nexus holds model weights, datasets, tool resources and evidence.
- The DevOps pipeline runs on self-hosted agents inside the network, including GPU agents, and supports manual approval steps, a secret store, and one pipeline triggering another.

## Design principles

- **One identity.** Every artifact, run, result and approval refers to the model's content digest, never to its name alone.
- **Pin everything.** Every run records the exact inputs that produced it, so it can be re-run and its results compared.
- **Proxies are for intake only.** A proxy returns whatever upstream holds today. Only the intake jobs (P0 and P2) read from proxies; they pin what they fetch and copy it into hosted repositories. Every other job reads hosted copies only.
- **Keep everything.** Raw outputs of every run are kept, failed runs included, and nothing is overwritten.
- **Automation drafts, people decide.** Pipelines compute a draft verdict against versioned policy. Automation may reject on a hard failure; only a person can approve a gate.
- **Separation of duties.** Only pipeline service accounts write to the approved-model and evidence repositories. People approve; they do not edit evidence.

## Architecture overview

```
 INTAKE (P0, P2 only)                                  HOSTED, PINNED (every other job)

 Docker Hub, GHCR, NGC ──► Quay proxy-cache orgs ──P0──► Quay: onboarding-base, onboarding-tools
 PyPI ───────────────────► Nexus: pypi-proxy ─────P0──► tool images (built from lock files)
 HF datasets, tool ─────► Nexus: hf-proxy, ──────P0──► Nexus: eval-data (data bundles per suite version)
 models, other hosts       web-raw-proxy-<host>
 Hugging Face models ────► Nexus: hf-proxy ───────P2──► Nexus: models-quarantine ──G3──► Nexus: models-approved

                         models-approved + onboarding-tools + eval-data
                                              │
                    Stage 4 garak ──► G4 ──► Stage 5 benchmarks ──► G5 ──► ... ──► dev ──► production
                                              │
                    Nexus: evidence (manifests, raw outputs, results, verdicts, approvals, signatures)

 Git: onboarding-config (suites, policies, image and data locks) ──► Git: model-registry ──► Git: gateway-aliases
```

## Quay organization layout

| Organization | Type | Pushed by | Pulled by | Contents |
| --- | --- | --- | --- | --- |
| `proxy-dockerhub`, `proxy-ghcr`, `proxy-nvcr`, ... | Proxy cache, one per upstream registry | Quay, on first pull | P0 robot account only | Cached upstream images, including the serving image (for example `vllm/vllm-openai`) and Python base images |
| `onboarding-base` | Hosted | P0 robot account | P0 builds, deploy pipeline | Upstream images that passed intake checks, copied by digest: base images and the serving image |
| `onboarding-tools` | Hosted | P0 robot account | Eval agents, deploy pipeline | Tool images built by P0: garak, LightEval, DeepEval, MTEB, GuideLLM, ModelScan, normalizer and policy check |

Quay rules:

- **References are by digest only.** Upstream tags move, and proxy-cache entries expire and are fetched again, so a tag never identifies an image in a manifest, lock file or deployment. Enable immutable tags on hosted organizations if your Quay version supports them.
- **Proxy caches are not a source for runs.** Eval agents and serving environments use a read-only robot account with access to the hosted organizations only. Only P0's robot account can pull from proxy-cache organizations.
- **Vulnerability scanning:** Clair scans every image in the hosted organizations. P0 stores the scan report for the exact digest with the image's intake record, and Security's policy decides which findings block intake.
- **Signatures** for hosted images are pushed to Quay next to the images (cosign), and the deploy pipeline verifies them before running an image.
- **Action logs** are exported to the central log store, so pushes, pulls, permission and robot-account changes are traceable.
- Each proxy-cache organization has its own upstream credentials, for example a Docker Hub service account to avoid anonymous rate limits.

## Nexus repository layout

| Repository | Format | Deployment policy | Written by | Read by | Contents |
| --- | --- | --- | --- | --- | --- |
| `pypi-proxy` | PyPI (proxy) | n/a | Nexus | P0 builds only | Upstream PyPI |
| `hf-proxy` | Hugging Face (proxy) | n/a | Nexus | P0 and P2 only | Upstream Hugging Face models and datasets, fetched by pinned revision |
| `hf-api-proxy` | Raw (proxy) | n/a | Nexus | P2 only | Hugging Face Hub API, for the security scan status of a model revision. Short cache lifetime, so the status is current |
| `web-raw-proxy-<host>` | Raw (proxy), one per approved host | n/a | Nexus | P0 only | Tool resources hosted outside Hugging Face and PyPI, for example NLTK data or spaCy model releases, added only after a discovery run finds them and Security approves the host |
| `models-quarantine` | Raw (hosted) | Disable redeploy | P2 | P2 | Candidate weights before G3 |
| `models-approved` | Raw (hosted) | Disable redeploy | Promotion job only | Eval agents, deploy pipeline | Weights after G3; the only source for any serving environment |
| `eval-data` | Raw (hosted) | Disable redeploy | P0 (public data), P5 (domain test sets) | Eval agents | Data bundles per suite version, and domain test sets (see Datasets and models used by the tools) |
| `evidence` | Raw (hosted) | Disable redeploy | Pipeline service accounts only | Everyone with audit access | Everything the audit trail needs (see Evidence bundle) |

Nexus rules:

- **Disable redeploy** on every hosted repository, so a path once written cannot be overwritten. New content always goes to a new path.
- **Delete privileges** on `models-approved`, `eval-data` and `evidence` are held by no person or pipeline in day-to-day roles. Retention and retirement deletes run through a reviewed pipeline that records what it removed.
- **Proxy read access** is granted only to the P0 and P2 service accounts. Eval agents and serving environments have no read access to any proxy repository.
- **Content selectors** restrict read access to domain test sets by data classification.
- **Gated upstream content:** `hf-proxy` authenticates to Hugging Face with a platform service account. That account accepts a gated model's or dataset's terms only after Legal approves them, and the approval is recorded in the data inventory.
- **Nexus audit logging** is enabled and exported to the central log store, so repository and privilege changes are themselves traceable.
- **Blob store backups** cover `models-approved`, `eval-data` and `evidence`.

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
| `onboarding-config` | Pipeline definitions, suite definitions, policies (pass criteria), image lock (`images.lock.yaml`), data inventory and lock (`data/inventory.yaml`, `data/<suite>/<version>.lock.yaml`), normalizer and policy-check code | Reviewed pull requests; ML evaluation team and Security own their suites and policies. Lock files are written by P0 and merged by review |
| `model-registry` | One YAML record per model version: status, owner, gate records and evidence links | Pipeline service account only, on a protected branch; every commit references the pipeline run that made it |
| `gateway-aliases` | Alias-to-model-digest mapping and serving configuration per environment | Reviewed pull requests, merged only by the deploy pipeline after its checks pass |

Git history of `model-registry` is the status history of every model, and git history of `onboarding-config` is the history of every threshold change.

## Evidence bundle

All evidence for a model version lives under one path in the `evidence` repository. Evidence for images, tool models and data bundles, which is shared by every model, lives under `evidence/tool-intake/<run-id>/`.

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
images:                                   # from images.lock.yaml, hosted Quay orgs only
  lighteval: quay.internal/onboarding-tools/lighteval@sha256:...
  deepeval:  quay.internal/onboarding-tools/deepeval@sha256:...
  guidellm:  quay.internal/onboarding-tools/guidellm@sha256:...
  serving:   quay.internal/onboarding-base/vllm-openai@sha256:...
data:
  bundle: eval-data/bundles/rag/v2/bundle.tar.zst
  bundle_sha256: ...
  lock: data/rag/v2.lock.yaml             # lists every dataset and tool model inside the bundle
  domain_sets:
    - path: eval-data/domain/rag-domain/v2/test.jsonl
      sha256: ...
judge_model:
  digest: sha256:...
  endpoint: http://judge.eval.internal/v1
  temperature: 0
serving_config_sha256: ...
hardware: { gpu: <model>, count: 2, driver: <version> }
seeds: { global: 1234 }
```

## Datasets and models used by the tools

The scanning and evaluation tools need more than the model under test. Many fetch datasets, classifier models or other resources on first use, and in this network that fetch either fails or, worse, silently reads whatever a proxy returns that day. Every such resource is treated as a pinned, licensed, scanned input, like the model itself.

### What each tool needs

Examples below are typical; the authoritative list for a pinned tool version and suite is whatever the discovery run finds (next section).

| Tool | Needs at run time | Comes from | Points to watch |
| --- | --- | --- | --- |
| ModelScan (Stage 3) | Nothing beyond its package | Tool image | Its rule set is part of the pinned package version, so record the version with each scan |
| Clair (in Quay) | Vulnerability databases for OS packages and language ecosystems | Clair's updaters, through Quay's upstream access, or an offline import | Record the database update time with each scan report; a stale database passes images it should flag |
| garak (Stage 4) | Probe prompts and payloads (mostly bundled in the package); Hugging Face datasets for some probes, such as package lists for package-hallucination probes; Hugging Face classifier models used by detectors, such as toxicity and refutation detectors; NLTK data for some probes; attacker and judge models for generative attack probes | Package; `hf-proxy`; `web-raw-proxy-<host>`; internal models | Detector models are models: they follow the safetensors and scanning rules. Probes that call an external service (for example the Perspective API) are left out of the pinned probe set. Attacker and judge models are internally hosted models |
| LightEval (Stage 5) | A dataset per benchmark task (for example MMLU, GSM8K, ARC, HellaSwag, MATH, GPQA and multilingual sets); helper models and data for some metrics, such as a model for BERTScore or NLTK tokenizer data | `hf-proxy`; `web-raw-proxy-<host>` | Some datasets are gated (for example GPQA) and need Legal's approval of their terms. Pin every task's dataset revision in the suite |
| DeepEval (Stage 5) | Our domain test sets; the judge model. The metrics themselves need no public datasets | `eval-data/domain/` (uploaded through P5); internal judge model | Do not use DeepEval's built-in public benchmarks: LightEval covers them, and a second copy of the same datasets is a second thing to pin and audit. If DeepEval's synthesizer is used to draft test cases, its embedding model is an internal model too |
| MTEB (Stage 5) | Corpus, queries and relevance judgments for each selected task, mostly from the `mteb` organization on Hugging Face | `hf-proxy` | Each MTEB task names its dataset revision in its metadata; the lock records it. Retrieval corpora can be large, so select tasks by the archetype's languages and task types. Embedding models that need custom code follow the Stage 3 custom-code review |
| GuideLLM (Stage 5, 9) | The candidate's tokenizer, to size prompts in tokens; a prompt source | `models-approved`; built-in synthetic data or our own prompt sets | Prefer synthetic prompts sized per archetype, or prompts shaped like our traffic, over public chat datasets |
| Safety battery (Stage 5) | Memorization and canary sets; PII detection models if used (for example a spaCy model for Presidio) | `eval-data/domain/`; `web-raw-proxy-<host>` or `pypi-proxy` | Owned by data governance; canary sets are classified data |

### Discovery run

A tool version change can add a new download, so the inventory is discovered, not written from memory. P0 runs discovery whenever a tool version or suite changes:

1. Run the full pinned suite once in an isolated environment, against a small reference model, with `HF_ENDPOINT` pointing at `hf-proxy`, the package index at `pypi-proxy`, and every other outbound connection blocked and logged.
2. Read two logs: the Nexus request log for the proxy repositories (what was fetched), and the firewall deny log (what was missing).
3. For each blocked host, Security either approves a new `web-raw-proxy-<host>` repository or the probe or task is dropped from the suite. Repeat until the deny log is empty.
4. Write the fetched list to the suite's lock file. Every Hugging Face entry is pinned to a commit hash, never a branch such as `main`.

### Intake rules for datasets and tool models

| Rule | Why |
| --- | --- |
| Pin every Hugging Face dataset and model to a commit hash | A branch can change between runs |
| No dataset loading scripts; use the Parquet revision, or Security reviews the script | A loading script is remote code; `trust_remote_code` stays off for datasets as for models |
| No pickle-based files in tool models or datasets (`.bin`, `.pt`, `.pth`, `.pkl`, `.ckpt`, `.joblib`); safetensors or Parquet only, unless Security converts and verifies them | The same rule Stage 3 applies to candidate weights |
| ModelScan runs over every tool model and every file with a model-like extension in a dataset | A detector model is executed like any other model |
| Each entry has a license, and gated entries have Legal's recorded approval | Public benchmarks have their own terms, some non-commercial or research-only |
| Each entry has a data classification; public benchmarks are public, domain sets carry their owner's classification | Decides who can read it in `eval-data` |
| Internal models used by tools (judge, attacker, embedding for synthesis) are registered models that passed at least G2, G3 and G4 | Their outputs decide gate verdicts |

### Data inventory and lock

`data/inventory.yaml` lists every external resource the platform has approved; each suite's lock file lists exactly what one suite version uses:

```yaml
# data/inventory.yaml
- id: hf-dataset:cais/mmlu
  revision: <commit hash>
  used_by: [lighteval]
  license: MIT
  gated: false
  classification: public
  approved_by: { legal: <user>, security: <user>, at: 2026-10-30 }
- id: hf-model:<org>/<toxicity-detector>
  revision: <commit hash>
  used_by: [garak]
  format: safetensors
  modelscan: evidence/tool-intake/<run-id>/
  license: Apache-2.0
  gated: false
  classification: public
  approved_by: { security: <user>, at: 2026-10-30 }
```

```yaml
# data/rag/v2.lock.yaml
suite: rag/v2
tools:
  lighteval: quay.internal/onboarding-tools/lighteval@sha256:...
  deepeval:  quay.internal/onboarding-tools/deepeval@sha256:...
entries:
  - id: hf-dataset:<org>/<dataset>
    revision: <commit hash>
    files_sha256: sha256:...
  - id: nltk:punkt_tab
    source: web-raw-proxy-nltk
    sha256: ...
bundle: eval-data/bundles/rag/v2/bundle.tar.zst
bundle_sha256: sha256:...
```

### Data bundles

P0 packs each suite version's resources into one bundle in `eval-data`: a pre-filled Hugging Face cache plus any other files, such as NLTK data. Eval jobs download the bundle, check its sha256 against the lock, unpack it, and run fully offline:

| Setting | Why |
| --- | --- |
| `HF_HOME` pointing at the unpacked bundle | Tools find datasets, tokenizers and detector models locally |
| `HF_HUB_OFFLINE=1`, `HF_DATASETS_OFFLINE=1` | Hugging Face libraries fail fast instead of trying to download |
| `NLTK_DATA` pointing at the unpacked bundle | NLTK finds its data locally |
| `DEEPEVAL_TELEMETRY_OPT_OUT=YES` | Stops DeepEval sending telemetry |
| Egress denied except to the model under test, the judge and attacker models, and the hosted Nexus repositories | Anything missing from the bundle fails the run instead of being fetched |

A run that fails because something is missing from the bundle is a discovery gap: fix it through P0 and a new suite version, never by allowing a download during the run.

## Pipelines

| Pipeline | Process stage | Trigger | Gate approver |
| --- | --- | --- | --- |
| P0 `tooling` | Foundation | Change to tool versions, upstream images or suites in `onboarding-config` | Security (images, tool models, new hosts); Legal (gated or restricted licenses); reviewed pull request (lock files) |
| P1 `intake` | Stage 1 | Platform model owner submits a candidate | Platform model owner (G1) |
| P2 `acquire-and-scan` | Stage 3 | G1 pass | Security (G3) |
| P3 `garak` | Stage 4 | G3 pass | Security (G4) |
| P4 `benchmark` | Stage 5 | G4 pass | ML evaluation team (G5) |
| P5 `record` | Stages 2, 6, 7, 8, 10 | Owner uploads a record | The record's owner (G2, G6, G7) |
| P6 `deploy` | Stages 9 and 11 | Pull request to `gateway-aliases` | SRE and platform model owner (G8); go-live sign-off |
| P7 `re-evaluate` | Lifecycle | Schedule | Reuses P3 and P4 with the current suites |

Every pipeline follows the same pattern:

1. Write `manifest.yaml` to the evidence path before any work starts.
2. Run the jobs, pulling images from hosted Quay organizations and data from hosted Nexus repositories by digest, and verifying the model digest on load.
3. Upload raw outputs and logs, whether the run passed or failed.
4. Normalize results and run the policy check to produce `verdict.yaml`.
5. Pause at a manual approval step for the gate owner.
6. Write `approval.yaml`, `SHA256SUMS` and the signature.
7. Commit the gate record to `model-registry` and queue the next pipeline.

### P0: tooling

P0 is the only pipeline that reads from the Quay proxy caches and the Nexus PyPI and raw proxies. Its evidence goes under `evidence/tool-intake/<run-id>/`.

**Upstream images** (base images and the serving image):

1. Pull the image through its Quay proxy-cache organization by digest, resolving the tag once and recording both.
2. Wait for Clair's scan of that digest and store the report.
3. Apply Security's image policy to the report. Images that pass go to Security's manual approval.
4. Copy the image by digest into `onboarding-base` (for example with `skopeo copy --preserve-digests`), sign it, and record it in `images.lock.yaml` with its upstream reference, digest, scan report and approver.

**Tool images:**

1. Build each tool image from an `onboarding-base` image and a pip lock file, installed with `pip install --require-hashes` from `pypi-proxy`.
2. Push to `onboarding-tools`, wait for Clair's scan, apply the image policy, sign, and record the digest in `images.lock.yaml`.

**Data bundles**, for each new or changed suite version:

1. Run the discovery run and update the suite's lock file.
2. Fetch each entry through `hf-proxy` or its `web-raw-proxy-<host>` repository at the pinned revision.
3. Apply the intake rules for datasets and tool models; new entries go to Security, and gated or restricted-license entries to Legal.
4. Pack the bundle, upload it to `eval-data/bundles/<suite>/<version>/`, and record its sha256 in the lock file.
5. Open a pull request to `onboarding-config` with the updated lock files. A suite version is usable only after that pull request is merged.

### P1: intake

- The platform model owner submits upstream coordinates, the pinned revision, license, target archetypes and hardware needs.
- The pipeline checks the submission against archetype requirements in `onboarding-config` and creates the registry record with status **Proposed**.
- G1 is a manual approval step in the pipeline.

### P2: acquire and scan (Stage 3)

The only job that reads candidate models from `hf-proxy`, and the only reader of `hf-api-proxy`.

1. Fetch the Hub's security scan status for the pinned revision through `hf-api-proxy` and store the response. Any flagged file fails the run.
2. Download the pinned revision only through `hf-proxy` (with `HF_ENDPOINT` set to the proxy and the revision set to the commit hash), compute `files.sha256` and the model digest, and upload to `models-quarantine`.
3. **Format check:** every file must match the allowlist in the security baseline. Any pickle-based file (`.bin`, `.pt`, `.pth`, `.pkl`, `.ckpt`) fails the run.
4. **Local scans:** run the security baseline's scanners (for example ModelScan) on the quarantine copy.
5. **Custom code:** flag any Python file in the repository for Security review; `trust_remote_code` stays off unless that review passes.
6. **AI bill of materials:** generate a CycloneDX ML-BOM from the model files, config and upstream metadata.
7. **Isolated load test:** load the weights on an agent with no egress and record the log.
8. **G3 approval** by Security. On approval, the promotion job copies the files to `models-approved`, re-verifies the digest, and sets the registry record's `artifact` field to the approved path.

### P3: garak (Stage 4)

- Deploys the model from `models-approved` on the isolated eval serving environment with the pinned serving image and configuration.
- Runs garak from its pinned image with the suite's probe and detector selection, generations per probe and seed, against the OpenAI-compatible endpoint, with the suite's data bundle unpacked for detector models and probe datasets. Probes that need an attacker or judge model point at an internally hosted model recorded in the manifest.
- Keeps garak's report, hit log and HTML report in `raw/`.
- The normalizer computes failure rates per probe family; the policy check compares them with Security's limits and the incumbent's latest result on the same suite version.

### P4: benchmark (Stage 5)

Runs one job per tool in parallel, against the same deployment, manifest and data bundle:

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
- **Serving image:** both environments pull the serving image from `onboarding-base` by digest and verify its signature before starting.
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
| A mirrored dataset or tool model is found to be wrong. Which decisions relied on it? | Find the suite lock files that list it, then search all manifests in `evidence` for those bundles' sha256 |
| Where did this container image come from, and who let it in? | `images.lock.yaml` entry → upstream reference, Clair report and approver in `evidence/tool-intake/`, Quay action logs |
| Did any run download something at run time? | Eval agents have no proxy access and run offline; firewall deny logs for eval agents show any attempt |
| Under what terms do we use this benchmark dataset? | `data/inventory.yaml` entry: license, gated status and Legal's approval |
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
- [ ] Does our Nexus version's Hugging Face proxy serve datasets as well as models, and does `hf-api-proxy` pass query strings through for the scan-status call? If not, the platform model owner records the Hub scan status by hand and uploads it through P5.
- [ ] Does our Quay version support immutable tags on hosted organizations?
- [ ] How does Clair's vulnerability database get updated: through Quay's upstream access, or by offline import, and how often?
- [ ] Which hosts outside Hugging Face and PyPI does Security accept for `web-raw-proxy-<host>` repositories?
