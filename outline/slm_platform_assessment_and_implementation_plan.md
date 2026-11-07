# SLM Platform: Assessment, Open Questions and Implementation Plan

**Reviews:** `slm_platform_technical_specification.md` v0.1 (7 November 2026)
**Date:** 7 November 2026
**Planning assumptions (confirmed with the founder):**

| Assumption | Answer |
|---|---|
| Team | Solo founder or 1–2 engineers |
| First product to reach paying customers | Model Deployment Kit |
| Design partner / launch customer | None yet |
| Current factory workflow | Python scripts and notebooks on RunPod, outputs pushed to Hugging Face by hand |

---

## Part A: Assessment

### A.1 Overall verdict

The specification is technically sound and unusually disciplined for a v0.1. It
picks the right first engineering milestone (a stable contract between model
creation and deployment), treats evaluation as a release gate rather than an
afterthought, and is honest that schemas, resource figures and CLI commands are
illustrative.

Its main weakness is **not technical but proportional**: it describes the end
state of a company with three products (Factory, Deployment Kit, AI
Applications) plus a shared multi-tenant platform. With 1–2 engineers and no
design partner, roughly 80% of the spec must be deferred, and the 20% that is
built first must be chosen for *learning from customers*, not for architectural
completeness.

### A.2 Strengths to keep

1. **Contract-first architecture** (§5, §7.3, §21). The manifest as the
   hand-off between registry, generator and deployment is the right spine.
2. **Deployment Kit independent of the Factory** (§7.1). This lets the Kit
   be sold on its own and lets you onboard customer-supplied models.
3. **Evidence-based release** (§4.8, §8.2 steps 12–14). "Training finished" is
   not "ready". This is a genuine differentiator if the evidence is good.
4. **Explicit compatibility catalogue with clear failure** (§8.3). This is
   where most of the real engineering value of the Kit lies.
5. **Incremental roadmap** (§17): CLI → Compose → evaluation → Kubernetes →
   platform → application. The order is broadly right.
6. **Sensible security posture**: no inbound access to customer networks,
   no prompt logging by default, immutable releases, offline install.

### A.3 Issues and recommended changes

#### 1. Scope is far larger than the team (high impact)

The spec covers ingestion, dataset generation, training orchestration,
registry, packaging, three runtimes, three deployment profiles, API gateway,
multi-tenancy, monitoring, remote management, and an application layer. Each
of these is a product in its own right.

**Recommendation:** Build the Deployment Kit as a single Python CLI. Run
the Factory as a *service you perform* with your existing scripts, made
reproducible but not yet automated. Defer the management platform, the API
gateway, multi-tenancy, remote management and AI Applications until a paying
customer asks for them.

#### 2. The Kit's commercial differentiation is unproven (high impact)

Serving an open-weights model is already easy: `vllm serve <repo>`, Ollama,
Hugging Face Inference Endpoints, NVIDIA NIM, BentoML, KServe, and hosted
fine-tune-and-serve providers all cover parts of it. A Kit that "generates a
Docker Compose file for vLLM" will not justify a purchase by itself.

The defensible value lies in what those tools do *not* bundle together:

- A **validated compatibility matrix** (this model + this runtime version + this
  GPU = known to work, with evidence).
- **Signed, checksummed, offline-installable bundles** for restricted networks.
- **Evaluation and performance evidence** shipped with the package.
- **Provenance** from source data to deployed model (when combined with the
  Factory).

These matter most to regulated and security-conscious buyers, who also need
the heaviest features (offline bundles, signing, Kubernetes). So there is a
tension: the buyers who value the Kit most are the most expensive to serve.

**Recommendations:**

- Treat the first 3 months as **customer discovery plus MVP**. Aim for one
  design partner before building Kubernetes support.
- Pull **offline bundle, signing and validation report** earlier (Phase 3
  below) because they are the differentiators, and push Helm later.
- Package the Kit with a done-for-you fine-tune as the first commercial offer
  ("we fine-tune a model on your data and deliver a validated, signed,
  installable package"). This uses your existing RunPod workflow for revenue
  while the Kit matures, and every engagement adds to the compatibility matrix.

#### 3. Two-language stack is too expensive for a solo build (high impact)

§14 proposes Java/Spring Boot for the platform API and Python for ML, with
JobRunr for jobs. With 1–2 engineers, two stacks double the CI, dependency
management, deployment and context switching. Everything the MVP needs
(Hugging Face Hub client, safetensors inspection, vLLM, eval tooling) is
Python-native.

**Recommendation:** Python only until there is a team and a reason. Build the
Kit as an importable library with a thin CLI (`slm_kit` + Typer). When the
management platform arrives (Phase 5), expose the same library through
FastAPI. Revisit Spring Boot only if an enterprise customer or new hire
makes it worthwhile. The Java client example in §7.2 can stay as generated
documentation, since the API is OpenAI-compatible.

#### 4. Don't build a registry, gateway or workflow engine yet (medium)

§19 asks "build or adopt" for the registry, gateway and orchestration.

| Component | MVP recommendation | Revisit when |
|---|---|---|
| Model registry | Manifest + provenance files stored next to weights in a private HF repo or S3 bucket; a local index file | More than ~20 models or multiple users. Then consider MLflow Model Registry or a Postgres-backed service |
| Artefact storage | Hugging Face private repos (already used) **plus** S3-compatible storage for offline bundles | A customer forbids HF |
| API gateway | Reverse proxy in the Compose package (e.g. Caddy or Traefik) for TLS and API-key auth; vLLM `--api-key` as a minimum | Managed hosting with metering. Then evaluate an established gateway (e.g. Envoy-based, Kong, LiteLLM proxy) rather than building one |
| Job orchestration | None. The CLI runs steps synchronously and writes state to disk, so a failed step can be re-run | Factory automation in Phase 5. Then choose between a Postgres-backed queue and a durable workflow engine (Temporal, Prefect) |

#### 5. Consider existing packaging standards before inventing one (medium)

The manifest in §7.3 is a reasonable start. Before freezing it, evaluate:

- **OCI artifacts for model weights** (pushed with ORAS, or the KitOps
  "ModelKit" format). Weights then live in any container registry, mirror into
  air-gapped registries with standard tooling, and can be signed with the same
  tooling as images. This could resolve several §19 decisions (artefact
  storage, offline installation, signing) at once.
- **Sigstore/cosign** for signing images and bundles, **syft** for SBOMs,
  **trivy/grype** for scanning, and **SLSA-style provenance** attestations.
- **Model cards / CycloneDX ML-BOM** for the provenance record.

Keep your own manifest as the deployment contract, but have it *reference*
these standard artefacts rather than duplicate them.

#### 6. Avoid custom runtime images in the MVP (medium)

§7.4 builds a custom Dockerfile on top of `vllm/vllm-openai`. The upstream image
already contains the server. Referencing the **upstream image by digest** and
passing configuration as arguments removes an image build, a registry and a
patching obligation from the MVP. Build custom images only when you need to
add something (e.g. a weight-fetch-and-verify step can be a separate init
container).

#### 7. Gaps not covered by the spec

| Gap | Why it matters |
|---|---|
| **Base-model licences on redistribution** | Merged models inherit the base licence. Some licences (e.g. the Llama community licence) carry attribution, naming and usage-policy conditions; Gemma has its own terms; many Qwen models are Apache-2.0 but not all. The Kit redistributes weights, so the compatibility catalogue needs a licence field that can block packaging. |
| **Teacher-model terms for synthetic data** (§4.3) | Several commercial LLM providers restrict using their outputs to train competing models. Check the terms of any teacher model before building dataset generation on it. |
| **Ownership of fine-tuned weights and customer data rights** | Who owns the adapter? Can you reuse learnings across customers? This must be in the contract template before the first Factory engagement. |
| **Structured output / guided decoding** | The banking example outputs JSON. vLLM supports schema-guided decoding, which makes "structured-output validity" close to 100% and should be a manifest option and an eval dimension. |
| **Multi-LoRA serving economics** | vLLM can serve many LoRA adapters on one base model. For managed hosting this is the main cost lever and is the practical answer to the §19 multi-tenancy question. Merge by default for customer-installed packages; serve adapters for managed hosting. |
| **GPU CI cost** | Validation needs real GPUs. Budget for on-demand GPUs (e.g. RunPod) per validation run and keep the support matrix small. |
| **Patch and support model** | Shipped packages pin vLLM versions that will get CVEs. Decide on support windows and how customers receive updated bundles. |
| **Regulation** | If customers use models for decisions such as creditworthiness, EU AI Act high-risk obligations may apply to them and indirectly to you as a supplier. Get advice before selling into banking. |
| **Commercial model** | Pricing (per package, per model, subscription, managed per-token) affects what must be metered and built. |
| **Hardware estimation method** (§19) | This can be computed rather than guessed: weight bytes + KV cache (2 × layers × KV heads × head dim × bytes × context × concurrency) + runtime overhead, then confirmed by a measured benchmark. |

#### 8. Smaller points

- `evaluation-report.pdf` (§7.2): generate Markdown/HTML plus machine-readable
  JSON. PDF can be rendered from HTML later.
- `apiVersion: slm.example.com/v1`: choose the real domain early, because it
  is hard to change once packages ship.
- MVP criterion 9 ("supported GPU host") needs a concrete definition: list
  the GPUs, driver versions and container toolkit versions.
- Phase 1 exit criterion should include "manifest JSON Schema published and
  versioned", so the contract is machine-checkable.
- The spec has no success metrics for the business (e.g. time to package a
  new model, number of validated combinations). Add them so progress is
  measurable.

---

## Part B: Open Questions

Questions are grouped by when they block work. Items marked ⛔ block the phase
named.

### B.1 Business and commercial

1. ⛔ *Phase 0:* Which buyer segment is the first target (regulated
   enterprise on-prem, mid-market managed hosting, ISVs embedding models)?
   This decides whether offline/Kubernetes or managed hosting comes first.
2. What is the pricing model: per-package licence, subscription with updates,
   fine-tuning engagement fee, managed per-token/per-instance?
3. Is the Kit open source (open core with paid validated packages and support)
   or proprietary? An open-source CLI is a credible distribution strategy in
   this market, but changes what you can charge for.
4. What is the budget and runway for GPU compute (dev, CI validation, demos)?
5. What is the time to first revenue you are aiming for?

### B.2 Legal and data

6. ⛔ *First Factory engagement:* Who owns fine-tuned weights, datasets and
   synthetic data? What reuse rights do you retain?
7. Which teacher model(s) will generate synthetic data, and do their terms
   allow it?
8. Where may customer data be processed (RunPod regions, data-residency
   requirements)? Does RunPod meet the security bar of your target customers?
9. Will you pursue SOC 2 / ISO 27001, and when?

### B.3 Technical

10. ⛔ *Phase 2:* Initial support matrix: which 2–3 model families, which
    sizes, which GPUs (e.g. 24 GB L4/A10G class and 80 GB A100/H100 class)?
11. ⛔ *Phase 1:* Do you adopt OCI artefacts (ORAS/KitOps) for weights, or
    keep HF + S3? (Recommended: run a one-week spike in Phase 1.)
12. Is the Ollama/llama.cpp (GGUF) profile needed for the MVP, or only vLLM?
    (Recommended: vLLM only until a customer needs CPU or edge deployment.)
13. Should the first package support LoRA adapters directly, or merged models
    only? (Recommended: merged only for customer packages.)
14. What does the current training workflow use (Unsloth, Axolotl, TRL,
    custom), and which hyperparameters/configs are already captured?
15. Which domain/name will the manifest `apiVersion` use?

---

## Part C: Implementation Plan

### C.1 Principles for this plan

- **One language, one repo, one CLI.** Python 3.12, `uv`, Typer, Pydantic v2,
  Jinja2 templates, pytest.
- **Library first.** All logic lives in `slm_kit` so the later FastAPI service
  calls the same code.
- **Data-driven compatibility.** The compatibility catalogue and GPU
  profiles are YAML files with tests, not code branches.
- **Every phase ends in a demo** you can show a prospective customer.
- **Customer discovery runs alongside engineering** throughout Phases 0–3.

Indicative timeline assumes one full-time engineer. With two, Phases 2 and 3
overlap and the timeline shortens by roughly a third.

```text
Week:     0    2    4    6    8   10   12   14   16   18   20   22   24+
Phase 0   ████
Phase 1        ██████
Phase 2              ██████████
Phase 3                        ████████
Discovery ██████████████████████████████████  (target: 1 design partner by wk 14)
Phase 4                                 ░░░░░░░░░░  (only with a partner who needs it)
Phase 5+                                          ░░░░░░░░░░░░░░░░░░
Factory   ░░░░ reproducible service engagements alongside, from Phase 1 ░░░░
```

### C.2 Phase 0: Foundations and discovery (weeks 0–2)

**Engineering**

- Create the repository layout (below), CI (GitHub Actions: lint, type-check,
  unit tests), pre-commit, licence decision for the code.
- Add `docs/adr/` and record the first decisions: Python-only, manifest as
  contract, vLLM as first runtime, no custom runtime image.
- Set up accounts: Hugging Face org with private repos, S3-compatible bucket,
  RunPod API key in a secrets manager, a container registry.
- Pick the initial support matrix (question 10). Suggested starting point:
  two or three current open-weights families with permissive licences at
  1–8B parameters, on one 24 GB GPU class and one 80 GB class.

**Business**

- List 30 target companies in one segment; run 10–15 discovery conversations.
  Test the offer: "validated, signed, installable SLM package + optional
  fine-tune on your data".
- Get legal advice on base-model licence redistribution, teacher-model terms
  and a standard data/IP clause (questions 6–7).

**Exit:** repo and CI running; ADRs written; support matrix chosen; discovery
calls booked.

```text
slm-kit/
  pyproject.toml
  src/slm_kit/
    cli.py                 # Typer entry point: inspect | package | validate | install | bundle
    manifest/              # Pydantic models -> JSON Schema (versioned)
    sources/               # huggingface.py, local.py, s3.py  (resolve immutable revision, fetch, checksum)
    inspect/               # architecture, params, dtype, tokenizer, chat template, licence
    compat/                # catalogue loader + rule engine
    hardware/              # memory estimator, GPU profiles
    generate/              # Jinja renderers for compose, docs, openapi
    validate/              # smoke tests, benchmarks, task evals, report
    bundle/                # checksums, signing, SBOM, offline bundle
  catalogue/
    runtimes/vllm-<ver>.yaml
    architectures/*.yaml
    gpus/*.yaml
    licences/*.yaml
  templates/compose/ ...   templates/docs/ ...
  schemas/manifest.v1alpha1.json
  tests/  (unit + golden-file tests for generated output)
  factory/                 # existing training scripts, made reproducible (see C.8)
  docs/adr/
```

### C.3 Phase 1: Standardise the model output (weeks 2–5)

Maps to spec §17 Phase 1 and §5.

1. **Manifest schema `v1alpha1`** as Pydantic models, exported to JSON Schema
   and checked into `schemas/`. Fields from §5.1 and §7.3, plus `licence`,
   `chatTemplate`, `structuredOutput`, `provenance` (dataset, training run,
   evaluation refs) and `artefacts[]` with SHA-256 checksums.
2. **Source resolvers**: Hugging Face (resolve branch/tag to commit SHA, fetch
   via `huggingface_hub` with token from env/secret), local directory, S3.
3. **`slm-kit inspect <source>`**: read `config.json`, safetensors headers
   (parameter count, dtype without loading weights), tokenizer and chat
   template, licence from the model card, file checksums. Output a draft
   manifest.
4. **Factory hook**: modify the existing RunPod scripts to write a
   `provenance.json` (base model + revision, dataset hash, config,
   hyperparameters, seed, framework versions, GPU type, timings) and push it
   alongside the adapter/merged model.
5. **Spike (1 week): OCI artefacts for weights** (question 11). Push a model
   with ORAS or KitOps, pull it in an isolated network, sign with cosign.
   Record the decision as an ADR.

**Exit:** every existing model in your HF org can be inspected and produces a
schema-valid manifest; new training runs emit provenance automatically.

### C.4 Phase 2: Deployment CLI MVP (weeks 5–10)

Maps to spec §17 Phase 2 and MVP acceptance criteria §18.

1. **Compatibility catalogue** for the chosen vLLM version: supported
   architectures, dtypes, quantisation, LoRA support, minimum compute
   capability, known issues. Rule engine returns `supported | unsupported(reason)
   | needs-conversion(steps)`.
2. **Hardware estimator**: weights + KV cache + overhead per GPU profile;
   recommend `max-model-len`, `gpu-memory-utilization` and max concurrent
   sequences. Mark estimates as estimates until Phase 3 measures them.
3. **`slm-kit package`** renders:
   - `manifest.yaml`, `runtime-config.yaml`
   - `docker-compose.yaml` with: upstream vLLM image **by digest**; an init
     container that fetches weights and verifies checksums; reverse proxy with
     TLS and API-key auth; health checks using vLLM's `/health`;
     restart policy; read-only model volume; resource reservations; log
     settings; no model port exposed outside the internal network.
   - `.env.example`, OpenAPI spec (subset of the OpenAI chat/completions API
     the runtime supports), `curl` and Python/Java examples.
   - `installation.md`, `configuration.md`, `operations.md`,
     `troubleshooting.md` from templates.
   - `checksums.sha256`.
4. **`slm-kit install --target docker`**: pre-flight checks (NVIDIA driver,
   container toolkit, GPU memory, disk space), then `docker compose up`.
5. **`slm-kit validate`**: start the package on a RunPod GPU (via its API) or
   locally, wait for readiness, run smoke tests (chat completion, streaming,
   JSON-schema-guided output if configured, auth rejection without key), tear
   down. Fail with diagnostics.
6. **Golden-file tests** for generated output, and a nightly GPU job that
   packages and validates each catalogue combination.

**Exit (spec criterion):** a supported model is packaged and runs on a
supported GPU host with no manual edits, and all 15 MVP criteria in §18 pass.
**Demo:** "HF repo → validated package → running authenticated API in
under 15 minutes."

### C.5 Phase 3: Evidence, integrity and offline (weeks 10–14)

Combines spec Phase 3 with the supply-chain items from §7.7, which are moved
earlier because they are the differentiators.

1. **Performance benchmark**: latency percentiles, throughput and tokens/s at
   several concurrency levels; GPU memory peak. Store in `benchmark.json` and
   use to correct the hardware estimator.
2. **Task evaluation harness**: run a customer- or Factory-supplied eval set;
   metrics: exact match, JSON validity, field-level accuracy/F1, and comparison
   with the base model. Thresholds in the manifest; packaging fails if they
   are not met.
3. **Evaluation report**: Markdown/HTML + JSON (intended use, limitations,
   method, results, baseline, approval status).
4. **Supply chain**: SBOM (syft), image scan (trivy/grype), cosign signatures
   for the bundle and image digests, `slm-kit verify`.
5. **`slm-kit bundle --offline`**: tarball with saved images, weights (or OCI
   layout), signatures and installer; tested on a host with no outbound
   network.

**Exit:** each package ships with reproducible validation evidence and can be
installed and verified offline. **Demo for design partners.**

### C.6 Phase 4: Kubernetes (weeks 14–20, gated)

Start only when a design partner needs it; otherwise extend the catalogue,
add the GGUF/Ollama profile or invest in the Factory instead.

- Helm chart with values schema; GPU requests, node selectors/tolerations,
  probes, NetworkPolicy, pod security context, secret references,
  PVC or object-storage weight loading via init container.
- Test on k3s/kind with a GPU node (RunPod or a cloud GPU node) and on the
  partner's distribution (EKS/AKS/GKE/OpenShift).
- Rollback by Helm revision; document upgrade procedure.

**Exit:** the partner installs a validated package into their cluster.

### C.7 Phase 5 and beyond: platform, Factory automation, applications

Only after the Kit has paying users:

- **Management service**: FastAPI wrapping `slm_kit`; Postgres for
  customers, projects, models, packages, deployments; object storage; job
  execution via the workflow engine chosen then (§19).
- **Factory automation**: ingestion → dataset generation → QA → RunPod
  training via API → evaluation gate → registration → packaging, each as a
  retryable step with human approval gates (spec §15).
- **Managed hosting**: multi-LoRA vLLM per base model, gateway with metering.
- **Web UI**, then the **first AI application** (spec Phase 6), chosen from
  problems seen in Factory engagements.

### C.8 Factory track (alongside Phases 1–3)

Keep the Factory as a service, made reproducible rather than automated:

1. Move scripts and notebooks into `factory/` as parameterised scripts with
   config files (no hidden notebook state); pin framework versions; set seeds.
2. Add dataset hashing, train/val/test split with leakage checks, and record
   synthetic-example origin (spec §4.3).
3. Every run emits `provenance.json` and pushes to a private HF repo with a
   manifest, so its output goes straight into `slm-kit package`.
4. Hold out an evaluation set per engagement; that becomes the Phase 3 task
   evaluation input.

### C.9 Success metrics

| Metric | Target by end of Phase 3 |
|---|---|
| Time from HF repo to validated running package | < 15 minutes (excluding download) |
| Validated model × runtime × GPU combinations | ≥ 6 |
| Packages requiring manual edits after generation | 0 |
| Hardware estimate error vs measured | < 15% GPU memory |
| Design partners signed | ≥ 1 |
| Discovery conversations | ≥ 15 |

### C.10 Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Kit is seen as "a wrapper around vLLM" | High | High | Lead with evidence, offline/signed bundles and done-for-you fine-tuning; validate in discovery before Phase 4 |
| vLLM releases break compatibility | High | Medium | Pin by digest; nightly matrix validation; catalogue per runtime version |
| GPU spend exceeds budget | Medium | Medium | Small matrix; on-demand GPUs only during validation; cache weights |
| Licence breach when redistributing merged models | Medium | High | Licence field in catalogue blocks packaging; legal review in Phase 0 |
| Solo-founder bandwidth (sales + engineering) | High | High | Strict deferral list; time-box discovery (e.g. 2 days/week) |
| Regulated buyer requires certifications you lack | Medium | High | Target a less-regulated first partner, or plan SOC 2 readiness from Phase 3 |

### C.11 Explicitly deferred (from the spec)

Web UI · Spring Boot API · Postgres registry service · API gateway product ·
multi-tenancy · remote management agent · autoscaling · multi-cloud
provisioning · Ollama/llama.cpp profile · automated ingestion and dataset
generation · AI Applications stream.

Each moves into scope only when a paying customer needs it.
