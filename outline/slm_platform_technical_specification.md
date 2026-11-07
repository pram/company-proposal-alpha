# Technical Architecture and Product Specification

## Hybrid Small Language Model (SLM) Factory and AI Application Platform

**Document status:** Initial technical specification\
**Version:** 0.1\
**Date:** 7 November 2026\
**Purpose:** Define the technical architecture, capabilities,
interfaces, deployment artefacts and implementation approach for a
platform that creates specialised Small Language Models (SLMs), deploys
them as consumable services, and uses them to power business-specific AI
applications.

------------------------------------------------------------------------

## 1. Executive Summary

The proposed product is a hybrid AI engineering platform with two
complementary product streams:

1.  **SLM Factory** -- a reusable capability for creating specialised,
    optimised language models from datasets generated from
    customer-provided materials.
2.  **AI Applications** -- business-specific applications that use SLMs,
    larger language models, conventional software and workflow
    automation to perform defined business functions.

A shared platform provides model management, orchestration, deployment,
monitoring, security and automation capabilities.

A central architectural principle is to separate **model creation**,
**model serving**, **deployment packaging** and **business
applications**. A model produced by the SLM Factory should be deployable
independently of the training process. The deployment capability should
also accept models produced elsewhere, including compatible Hugging Face
repositories and customer-supplied models.

The intended outcome is a repeatable system in which a trained model can
be inspected, validated, packaged, deployed and exposed through a
standard API with minimal manual engineering intervention.

The platform should support managed hosting, deployment into a
customer's cloud subscription and on-premises/private infrastructure.

## 2. Goals and Design Principles

### 2.1 Goals

-   Turn the existing semi-manual SLM creation workflow into a
    repeatable, automated product.
-   Convert customer-provided source materials into curated training
    datasets.
-   Fine-tune existing pre-trained models rather than train foundation
    models from scratch.
-   Produce versioned model artefacts, including adapters and merged
    models.
-   Evaluate models against defined quality and performance criteria.
-   Select an appropriate inference runtime and deployment
    configuration.
-   Generate standardised, installable deployment packages.
-   Expose models through stable, documented APIs.
-   Support private and customer-controlled deployment.
-   Build business applications that combine models, tools, integrations
    and workflow execution.
-   Minimise operational costs and human intervention through
    automation.

### 2.2 Design principles

1.  **Automation first:** Every repeatable activity should be
    represented as an executable, observable and retryable workflow.
2.  **Separation of concerns:** Training, model registry, serving,
    deployment and application logic are distinct capabilities.
3.  **Model portability:** Avoid unnecessary dependence on one model
    provider, runtime, GPU provider or cloud.
4.  **Reproducibility:** Every model release must be traceable to its
    base model, source data, dataset version, training configuration and
    evaluation results.
5.  **Customer isolation:** Customer data, models, credentials, logs and
    deployments must be appropriately isolated.
6.  **Security by design:** Private data and model artefacts must be
    protected in transit and at rest.
7.  **Evidence-based quality:** A model is not ready for deployment
    merely because training completed; it must pass defined evaluations.
8.  **Open interfaces:** Prefer documented HTTP APIs and portable
    packaging formats.
9.  **Human oversight:** Human approval remains available for sensitive,
    high-impact or uncertain operations.
10. **Incremental implementation:** Start with a working CLI and Docker
    deployment path, then add enterprise orchestration and user
    interfaces.

## 3. Product Scope and Logical Architecture

### 3.1 Product streams

#### Stream 1: SLM Factory

The SLM Factory accepts customer materials and produces specialised
models.

Core capabilities:

-   Source document ingestion.
-   Content extraction and normalisation.
-   Dataset generation.
-   Dataset quality checks and versioning.
-   Base-model selection.
-   Fine-tuning and adapter generation.
-   Adapter merging where appropriate.
-   Model optimisation and quantisation where appropriate.
-   Model evaluation and benchmarking.
-   Model registration and release.

#### Stream 2: AI Applications

AI Applications solve specific business problems using the platform's
model and orchestration capabilities.

An application may combine:

-   One or more SLMs.
-   Third-party foundation models.
-   Retrieval-augmented generation (RAG).
-   Conventional application code and business rules.
-   APIs and enterprise system integrations.
-   Workflow orchestration and agents.
-   Human approval and exception handling.
-   Audit and reporting capabilities.

The application layer must not assume that every task should use a
fine-tuned SLM. Model selection should be driven by task requirements,
quality, latency, privacy and cost.

### 3.2 Shared platform capabilities

-   Customer and tenant management.
-   Project and job orchestration.
-   Model registry and artefact metadata.
-   Dataset registry and provenance.
-   Runtime and deployment management.
-   Secrets and configuration management.
-   API gateway and access control.
-   Monitoring, logging and audit.
-   Evaluation and benchmark management.
-   Package generation and distribution.

### 3.3 Logical architecture

``` text
                         CUSTOMER / PLATFORM USERS
                                    |
                   +----------------+----------------+
                   |                                 |
             SLM FACTORY                       AI APPLICATIONS
                   |                                 |
        Document and data ingestion          Business application APIs
                   |                                 |
        Dataset generation and QA            Agents and workflow logic
                   |                                 |
        Base model selection                  Model and tool selection
                   |                                 |
        Fine-tuning and optimisation          Business integrations
                   |                                 |
        Evaluation and validation             Human approvals / audit
                   |                                 |
                   +----------------+----------------+
                                    |
                         MODEL REGISTRY
                                    |
                    DEPLOYMENT PACKAGE GENERATOR
                                    |
                     MODEL SERVING / RUNTIME
                                    |
              +---------------------+--------------------+
              |                     |                    |
         Managed cloud        Customer cloud       On-premises
              |                     |                    |
              +---------------------+--------------------+
                                    |
                        API / INFERENCE INTERFACE
```

## 4. SLM Factory: Technical Workflow

### 4.1 Source ingestion

The platform accepts customer-provided source materials.

Initial supported formats should include:

-   PDF.
-   DOCX.
-   TXT and Markdown.
-   HTML.
-   CSV and JSON.
-   Other structured formats where a defined parser is available.

Ingestion responsibilities:

-   Validate file type and size.
-   Scan uploads for malware where appropriate.
-   Extract text and relevant structure.
-   Preserve source identifiers and document metadata.
-   Identify extraction failures.
-   Record customer, project and source-document ownership.
-   Store original documents in controlled object storage.

The source material must be treated as customer data. Access, retention
and deletion policies must be configurable.

### 4.2 Content processing

Processing should include:

-   Text normalisation.
-   Removal of irrelevant boilerplate where safe.
-   Deduplication.
-   Chunking or segmentation where required.
-   Metadata extraction.
-   Language identification where relevant.
-   Detection of malformed or low-quality content.
-   Provenance links from generated examples back to source material.

The processing pipeline should retain enough information to reproduce a
dataset and investigate quality issues.

### 4.3 Dataset generation

The platform transforms source material into training examples suitable
for the selected training objective.

Potential dataset formats include JSONL instruction/response records,
conversational examples and task-specific structured examples.

Example:

``` json
{
  "messages": [
    {
      "role": "system",
      "content": "Classify banking operations exceptions."
    },
    {
      "role": "user",
      "content": "Transaction rejected because the beneficiary account is inactive."
    },
    {
      "role": "assistant",
      "content": "{\"category\":\"INVALID_BENEFICIARY_ACCOUNT\",\"severity\":\"MEDIUM\"}"
    }
  ]
}
```

Dataset generation may use deterministic transformations, templates,
rules and teacher-model-generated synthetic examples.

Quality controls should include:

-   Schema validation.
-   Duplicate detection.
-   Invalid or contradictory examples.
-   Output-format validation.
-   Sampling and human review.
-   Train/validation/test separation.
-   Leakage checks between splits.
-   Dataset versioning.

Synthetic examples should be labelled as synthetic and traceable to
their generation method and source context.

### 4.4 Model selection

The platform should maintain a catalogue of candidate base models.

Selection criteria should include:

-   Model architecture.
-   Parameter count.
-   Context length.
-   Supported languages.
-   Licence and permitted use.
-   Training framework compatibility.
-   Inference runtime compatibility.
-   Hardware requirements.
-   Baseline quality for the target task.

The platform should not assume that the smallest model is always best.
It should identify the smallest model that meets the customer's agreed
quality and operational requirements.

### 4.5 Fine-tuning

The initial training approach is fine-tuning an existing pre-trained
SLM.

Potential techniques:

-   LoRA.
-   QLoRA.
-   Full fine-tuning where justified.
-   Other parameter-efficient fine-tuning methods as they become
    relevant.

The training workflow should record:

-   Base model identifier and immutable revision.
-   Dataset identifier and version.
-   Training framework and version.
-   Training configuration and hyperparameters.
-   Random seed where applicable.
-   Hardware and compute configuration.
-   Training logs and checkpoints.
-   Output adapter or model artefact.
-   Training start/end times and status.

GPU compute may initially be supplied through RunPod. The platform
should abstract the compute provider so that alternative providers can
be added without changing the training workflow.

### 4.6 Model artefacts

The factory should support at least two output types.

**LoRA adapter**

An adapter contains fine-tuned weights and configuration and normally
requires the compatible base model at runtime.

Typical files:

``` text
adapter/
  adapter_config.json
  adapter_model.safetensors
```

**Merged model**

The adapter weights are merged into the base model to create a complete
model artefact.

Typical files may include:

``` text
model/
  config.json
  generation_config.json
  model.safetensors
  tokenizer.json
  tokenizer_config.json
  special_tokens_map.json
```

The exact files depend on the model architecture and framework.

The platform should preserve the original adapter and base-model
references even when a merged model is produced.

### 4.7 Optimisation

Potential optimisation stages include:

-   Weight quantisation.
-   Runtime-specific conversion.
-   Memory optimisation.
-   Inference configuration tuning.
-   Batch and concurrency tuning.
-   Context-length configuration.
-   CPU/GPU placement decisions.

Optimisation must be evaluated against model quality. A smaller or
faster artefact is not acceptable if it falls below the agreed quality
threshold.

### 4.8 Evaluation and release approval

Evaluation should compare the fine-tuned model with the base model and
any relevant baseline.

Metrics depend on the use case. Potential measures include:

-   Accuracy.
-   Precision and recall.
-   F1 score.
-   Structured-output validity.
-   Exact match.
-   Task-specific quality.
-   Hallucination or unsupported-claim rate.
-   Latency.
-   Throughput.
-   Tokens per second.
-   Memory consumption.
-   Cost per request or task.

A release should include:

-   Evaluation dataset and version.
-   Evaluation methodology.
-   Results and thresholds.
-   Comparison with baseline.
-   Known limitations.
-   Approval status.

Evaluation data must be controlled so that test examples are not
inadvertently included in training data.

## 5. Model Registry and Provenance

The model registry is the hand-off point between model creation and
deployment.

It should store model metadata and references to immutable artefacts.

### 5.1 Required metadata

-   Model identifier.
-   Customer and project identifiers.
-   Model name and semantic version.
-   Base model and revision.
-   Adapter or merged-model status.
-   Model architecture.
-   Parameter count.
-   Artefact format.
-   Artefact location.
-   Checksum.
-   Tokenizer information.
-   Dataset version.
-   Training run identifier.
-   Training configuration reference.
-   Evaluation report reference.
-   Licence and usage restrictions.
-   Creation date and release status.

### 5.2 Artefact storage

Hugging Face may be used as a source and distribution repository,
including private repositories.

For production, the platform should also support controlled artefact
storage such as S3-compatible object storage or customer-controlled
storage.

Large model weights should generally be stored separately from
deployment configuration packages. Deployment packages should reference
an immutable model revision and verify its checksum.

### 5.3 Versioning

A model release must be immutable. Changes to weights, tokenizer,
configuration or relevant runtime assumptions should result in a new
release or deployment revision.

The platform should retain relationships between:

-   Source documents.
-   Dataset versions.
-   Training runs.
-   Model versions.
-   Evaluation reports.
-   Deployment packages.
-   Active deployments.

## 6. Model Serving and Inference

### 6.1 Purpose

A trained model is not directly usable by most customer applications. It
must be loaded by an inference engine, which exposes an interface
through which applications can submit prompts and receive generated
responses.

The inference engine is responsible for:

1.  Loading model artefacts.
2.  Managing CPU/GPU memory.
3.  Tokenising inputs.
4.  Executing model inference.
5.  Generating outputs.
6.  Returning responses.
7.  Supporting concurrency and request management.

### 6.2 Candidate inference runtimes

  -----------------------------------------------------------------------
  Runtime                 Intended use            Considerations
  ----------------------- ----------------------- -----------------------
  vLLM                    Primary production GPU  High-throughput
                          serving                 serving,
                                                  OpenAI-compatible API,
                                                  model and adapter
                                                  compatibility must be
                                                  checked

  Ollama                  Simple local and        Convenient local
                          private deployments     runtime and model
                                                  management; validate
                                                  model format and
                                                  supported features

  llama.cpp               Lightweight and         GGUF ecosystem, CPU and
                          quantised deployments   supported GPU execution

  Hugging Face Inference  Managed hosted          Useful for managed
  Endpoints               deployment              hosting and
                                                  prototyping; introduces
                                                  provider dependency and
                                                  cost
  -----------------------------------------------------------------------

The platform should use a runtime abstraction rather than hard-code one
engine into the model registry.

### 6.3 Initial runtime recommendation

-   Use vLLM as the preferred production GPU runtime.
-   Support Ollama for straightforward local deployments.
-   Support llama.cpp for selected quantised/GGUF deployment scenarios.
-   Consider managed Hugging Face endpoints as an optional hosting
    route.

Runtime selection must be based on model architecture, model format,
supported features, licence, hardware and performance requirements.

### 6.4 Runtime configuration

Runtime configuration should define:

-   Model path or repository reference.
-   Model revision.
-   Runtime and version.
-   Tokenizer configuration.
-   Context length.
-   Quantisation.
-   GPU utilisation and memory settings.
-   Maximum concurrency and batching settings.
-   Port and network binding.
-   Authentication integration.
-   Logging and telemetry settings.

The model weights should normally be mounted or downloaded from
controlled storage at deployment time rather than baked into a generic
runtime container image.

## 7. Standard Model Deployment Package

### 7.1 Purpose

The Model Deployment Kit is a standardised product that accepts a
pre-generated model and produces a validated, installable deployment
package.

It must not depend on the model having been created by the SLM Factory.

Supported inputs should include:

-   Private or public Hugging Face model repositories.
-   Customer-supplied model directories.
-   Models in supported object storage.
-   LoRA adapters with their compatible base models.
-   Merged model artefacts.

### 7.2 Logical package structure

An example package:

``` text
acme-banking-slm-v1.0.0/
|
+-- manifest.yaml
+-- checksums.sha256
+-- LICENSE
|
+-- runtime/
|   +-- Dockerfile
|   +-- entrypoint.sh
|   +-- runtime-config.yaml
|
+-- deployment/
|   +-- docker-compose.yaml
|   +-- .env.example
|   +-- kubernetes/
|   |   +-- deployment.yaml
|   |   +-- service.yaml
|   |   +-- configmap.yaml
|   +-- helm/
|       +-- acme-slm/
|
+-- api/
|   +-- openapi.yaml
|   +-- examples/
|       +-- curl.sh
|       +-- java-client.java
|
+-- evaluation/
|   +-- evaluation-report.pdf
|   +-- benchmark.json
|
+-- documentation/
    +-- installation.md
    +-- configuration.md
    +-- operations.md
    +-- troubleshooting.md
```

The model weights may be distributed separately, particularly for large
models. The package must contain an immutable reference to the artefact
and a mechanism to verify its checksum.

### 7.3 Model manifest

The manifest is the contract between the model registry, package
generator and deployment manager.

Illustrative YAML:

``` yaml
apiVersion: slm.example.com/v1
kind: ModelDeployment

metadata:
  name: acme-banking-slm
  version: 1.0.0
  customer: acme

model:
  source:
    type: huggingface
    repository: company/acme-banking-slm
    revision: "immutable-commit-or-revision"
  format: safetensors
  architecture: qwen3
  parameterCount: 3B
  artefactChecksum: "sha256:..."

runtime:
  engine: vllm
  version: "pinned-version"
  quantization: none
  apiProtocol: openai-compatible

resources:
  minimum:
    gpuMemory: 8Gi
    systemMemory: 16Gi
    cpu: 4

network:
  port: 8000

deployment:
  profile: production
  mode: docker
```

This is a proposed schema, not an existing standard. It must be
validated against the chosen runtime and hardware. Resource values are
illustrative and must be derived from actual model profiling.

### 7.4 Runtime container

A runtime container packages the inference engine and its dependencies.

Example concept:

``` dockerfile
FROM vllm/vllm-openai:PINNED_VERSION

ENV MODEL_PATH=/models/acme-banking-slm

EXPOSE 8000

ENTRYPOINT ["vllm", "serve"]

CMD [
  "/models/acme-banking-slm",
  "--host", "0.0.0.0",
  "--port", "8000"
]
```

The actual image and command must be tested against the selected vLLM
release and model architecture.

Model weights should normally be mounted from storage or downloaded at
startup using controlled credentials. Avoid embedding large customer
models in a reusable runtime image.

### 7.5 Docker Compose profile

A simple single-server deployment may use Docker Compose.

Illustrative configuration:

``` yaml
services:
  model-server:
    image: registry.example.com/slm-runtime:1.0.0
    container_name: acme-banking-slm
    ports:
      - "8000:8000"
    volumes:
      - ./model:/models/acme-banking-slm:ro
    environment:
      MODEL_PATH: /models/acme-banking-slm
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
```

A production configuration must additionally address health checks,
secrets, logging, resource limits, network exposure, GPU runtime
requirements and persistent storage.

### 7.6 Kubernetes and Helm profile

For enterprise deployments, provide:

-   Kubernetes Deployment or an appropriate workload controller.
-   Service.
-   ConfigMap.
-   Secret references.
-   GPU resource requests and limits.
-   Node selectors or tolerations where required.
-   Readiness and liveness probes.
-   Network policies.
-   Pod security settings.
-   Persistent or object storage integration.
-   Horizontal scaling policy where supported.
-   Helm chart and values schema.

The customer should be able to deploy into an existing Kubernetes
environment without requiring the platform provider to own the cluster.

### 7.7 Package integrity and supply chain

Each package should include:

-   SHA-256 checksums for distributed artefacts.
-   Cryptographic package signature.
-   Runtime image digest.
-   Software Bill of Materials (SBOM).
-   Licence and usage information.
-   Model provenance record.
-   Supported runtime and hardware matrix.

Offline installation should be supported for customers that cannot
download images or model files directly from public repositories.

## 8. Deployment Package Generator

### 8.1 Purpose

The Deployment Package Generator automates conversion of a registered
model into a validated deployment deliverable.

### 8.2 Workflow

1.  Receive model identifier or source repository.
2.  Resolve an immutable model revision.
3.  Retrieve model metadata and artefacts.
4.  Inspect architecture, tokenizer, format and dependencies.
5.  Verify licence and distribution restrictions.
6.  Check runtime compatibility.
7.  Estimate hardware requirements.
8.  Select a deployment profile.
9.  Generate manifest and runtime configuration.
10. Generate Docker Compose and/or Helm deployment files.
11. Build or select a compatible runtime image.
12. Deploy into an isolated validation environment.
13. Run smoke tests and inference tests.
14. Run performance and resource benchmarks.
15. Verify package checksums and signatures.
16. Generate documentation and package metadata.
17. Publish the versioned deployment package.

### 8.3 Compatibility handling

The generator must not assume that all Hugging Face models can be served
by all runtimes.

It should maintain a compatibility catalogue covering:

-   Model architecture.
-   Model format.
-   Runtime and version.
-   Adapter support.
-   Quantisation support.
-   Hardware requirements.
-   Required conversion steps.
-   Known limitations.

Unsupported combinations should fail with a clear diagnostic rather than
produce an apparently valid but unusable package.

### 8.4 Initial interface

Start with a CLI and a service API.

Illustrative CLI:

``` bash
slm-deploy package \
  --source huggingface:company/acme-banking-slm \
  --revision a83f92d \
  --profile production \
  --runtime vllm \
  --output ./acme-banking-slm-v1.0.0
```

Illustrative validation:

``` bash
slm-deploy validate ./acme-banking-slm-v1.0.0
```

Illustrative local deployment:

``` bash
slm-deploy install ./acme-banking-slm-v1.0.0 \
  --target docker
```

These are proposed command interfaces, not implemented commands.

The same core functionality can later be exposed through a web UI and
Spring Boot API.

## 9. API and Application Integration

### 9.1 Standard inference API

Prefer an OpenAI-compatible HTTP interface where supported by the
runtime.

Example:

``` http
POST /v1/chat/completions
Authorization: Bearer <API_KEY>
Content-Type: application/json
```

Example request:

``` json
{
  "model": "acme-banking-slm",
  "messages": [
    {
      "role": "user",
      "content": "Classify this banking exception."
    }
  ],
  "temperature": 0.1
}
```

The response should follow the selected API specification and include
appropriate request identifiers and usage information where available.

### 9.2 API gateway

The gateway or equivalent service should provide:

-   Authentication.
-   Authorisation.
-   Tenant and model access control.
-   Rate limiting and quotas.
-   Request size limits.
-   TLS termination.
-   API versioning.
-   Request correlation identifiers.
-   Usage metering.
-   Optional routing between model versions.
-   Configurable logging and redaction.

The gateway must not expose internal model-serving ports directly to
untrusted networks.

### 9.3 Application integration

Customer applications should interact with the model through the
documented API rather than depend on the underlying runtime.

Example Java integration concept:

``` java
// Pseudocode: use a compatible OpenAI Java SDK or HTTP client.
var request = ChatCompletionRequest.builder()
    .model("acme-banking-slm")
    .userMessage("Classify this transaction exception")
    .build();

var response = inferenceClient.complete(request);
```

The application should be able to change model endpoint or model version
through configuration, without requiring code changes.

### 9.4 Model versus application responsibilities

A model service accepts inputs and returns model outputs.

A complete AI application additionally handles:

-   Business process context.
-   Retrieval and source data.
-   Tool and API invocation.
-   Business rules.
-   Workflow state.
-   Validation.
-   Exception handling.
-   Human approvals.
-   Audit records.
-   User experience and reporting.

Do not place critical deterministic business rules solely inside a
language model.

## 10. Deployment Profiles

The same model and API contract should support three deployment
profiles.

### 10.1 Lightweight / local

Target: development, small deployments and customers with modest
infrastructure.

Potential technologies:

-   Ollama.
-   llama.cpp.
-   Docker Compose.
-   GGUF or other compatible quantised model formats.

Deliverables:

-   Compatible model artefact.
-   Runtime configuration.
-   Docker deployment.
-   Local API endpoint.
-   Basic health checks and documentation.

### 10.2 Production GPU

Target: dedicated GPU server or managed cloud GPU infrastructure.

Potential technologies:

-   vLLM.
-   Docker.
-   API gateway.
-   Prometheus and Grafana or equivalent monitoring.

Deliverables:

-   Production model artefact.
-   Runtime container.
-   Deployment configuration.
-   API endpoint.
-   Health checks.
-   Monitoring and operational documentation.
-   Rollback procedure.

### 10.3 Enterprise Kubernetes

Target: customer cloud subscription or on-premises Kubernetes.

Potential technologies:

-   vLLM or another compatible runtime.
-   Kubernetes.
-   Helm.
-   Customer-managed secrets and storage.
-   Enterprise identity and observability integrations.

Deliverables:

-   Model artefact or immutable model reference.
-   Runtime image.
-   Helm chart.
-   Kubernetes configuration.
-   Network and security policies.
-   Monitoring integration.
-   Installation, operations and troubleshooting guides.
-   Offline installation bundle where required.

## 11. Managed, Customer-Cloud and On-Premises Deployment

### 11.1 Managed hosting

The platform provider operates the model-serving infrastructure.

``` text
Customer application
       |
       v
Public/private API endpoint
       |
       v
API gateway and authentication
       |
       v
Model-serving runtime
       |
       v
Customer-specific model
       |
       v
GPU infrastructure
```

Responsibilities include infrastructure, runtime availability, model
loading, monitoring, patching and agreed service levels.

### 11.2 Customer cloud

The customer deploys the model in its own cloud subscription.

The provider supplies signed images, model artefacts or registry
references, Helm charts and installation support.

The customer controls the cloud account, network, data and
infrastructure.

### 11.3 On-premises

The model runs on customer-owned hardware.

The deployment bundle should support controlled transfer into restricted
environments. The customer should not need live connectivity to Hugging
Face or the provider's cloud to run inference once the required
artefacts have been installed.

### 11.4 Remote management

Remote management is an optional future capability and must be designed
carefully.

Possible approaches:

-   Customer-initiated outbound connection to a management service.
-   Customer-hosted management agent.
-   Offline management through signed update bundles.

Do not require inbound access from the provider to a customer's private
network.

Remote telemetry must be configurable, documented and subject to
customer approval. Prompts, responses and source documents should not be
transmitted by default.

## 12. Security, Privacy and Governance

Security requirements apply to both model creation and deployment.

### 12.1 Data protection

-   Encrypt data in transit and at rest.
-   Isolate customer data and artefacts.
-   Apply role-based access control.
-   Manage secrets through a secrets manager or customer-controlled
    equivalent.
-   Define retention and deletion policies.
-   Avoid logging sensitive prompts and responses by default.
-   Provide configurable redaction and audit policies.
-   Ensure temporary training and evaluation data is cleaned up
    according to policy.

### 12.2 Model and software supply chain

-   Verify model source and revision.
-   Validate model licences and usage rights.
-   Pin runtime and dependency versions.
-   Scan container images and dependencies.
-   Generate SBOMs.
-   Sign deployment packages and images.
-   Verify artefact checksums.
-   Maintain provenance from source data through to deployed model.

### 12.3 Operational security

-   TLS for API traffic.
-   Strong authentication and authorisation.
-   Network segmentation.
-   Least-privilege service accounts.
-   Rate limits and request limits.
-   Secure handling of customer credentials.
-   Audit events for deployment and model changes.
-   Vulnerability and patch management.

### 12.4 Model risk

The platform should document:

-   Intended use.
-   Known limitations.
-   Evaluation methodology.
-   Performance boundaries.
-   Model and dataset provenance.
-   Human oversight requirements.
-   Appropriate escalation behaviour.

A model should not autonomously execute high-impact actions unless the
application has appropriate controls, authorisation and validation.

## 13. Monitoring and Operations

### 13.1 Runtime monitoring

Monitor:

-   Request count.
-   Request success and failure rates.
-   Latency percentiles.
-   Tokens per second.
-   Queue time.
-   GPU utilisation.
-   GPU memory.
-   CPU and system memory.
-   Model load failures.
-   Container restarts.
-   API availability.

Potential tools:

-   Prometheus.
-   Grafana.
-   OpenTelemetry.
-   Centralised logging platform.

The final choice should be compatible with customer infrastructure and
observability standards.

### 13.2 Model quality monitoring

Where permitted and appropriately privacy-controlled, monitor:

-   Output schema validity.
-   Task-specific success metrics.
-   Human correction rates.
-   Escalation rates.
-   Drift indicators.
-   Evaluation results across releases.

Do not assume that collecting all prompts and responses is acceptable.
Data collection must be configurable and governed by customer policy.

### 13.3 Model lifecycle

Support:

-   Model registration.
-   Release approval.
-   Deployment.
-   Health validation.
-   Version promotion.
-   Rollback.
-   Retirement.
-   Deletion.

A deployment should reference an immutable model version. Updating a
model should be an explicit release operation rather than silently
replacing weights in place.

## 14. Proposed Technology Stack

The following is a proposed starting point based on the existing
technical direction. It is not a mandatory final selection.

  -----------------------------------------------------------------------
  Layer                   Candidate technology    Responsibility
  ----------------------- ----------------------- -----------------------
  Platform API            Java / Spring Boot      Core APIs,
                                                  customer/project
                                                  management

  ML processing           Python                  Data processing,
                                                  training, evaluation
                                                  and packaging

  Database                PostgreSQL              Metadata, projects,
                                                  jobs and deployment
                                                  state

  Object storage          S3-compatible storage   Documents, datasets and
                                                  model artefacts

  Training compute        RunPod initially        GPU training workloads

  Fine-tuning             LoRA / QLoRA tooling    Model adaptation

  Model repository        Hugging Face            Model source and
                                                  distribution

  Model serving           vLLM                    Production GPU
                                                  inference

  Local serving           Ollama                  Simple local deployment

  Alternative serving     llama.cpp               Selected GGUF/quantised
                                                  deployments

  Containers              Docker                  Reproducible runtime

  Orchestration           Kubernetes / Helm       Enterprise deployment

  Background jobs         JobRunr or equivalent   Asynchronous platform
                                                  tasks

  Monitoring              Prometheus / Grafana    Metrics and dashboards

  API contract            OpenAPI                 Documented API
                                                  interface
  -----------------------------------------------------------------------

The platform should avoid unnecessary coupling between Spring Boot and
Python. Use explicit APIs and job contracts between services.

## 15. Automation and Orchestration

Automation is a core operating principle.

Each significant operation should be a versioned workflow step with:

-   Defined inputs and outputs.
-   Validation.
-   Status and progress.
-   Retry behaviour.
-   Timeout.
-   Logging and correlation identifier.
-   Idempotency where practical.
-   Failure and escalation handling.

Examples:

-   Document ingestion.
-   Dataset generation.
-   Dataset validation.
-   Training job submission.
-   Training result collection.
-   Evaluation.
-   Model registration.
-   Package generation.
-   Deployment.
-   Smoke testing.
-   Release approval.

Human approval should be configurable at defined gates, especially for
data quality, model release and production deployment.

The platform should be able to recover from a failed stage without
repeating every previous stage.

## 16. Non-Functional Requirements

Initial requirements to be refined through implementation and customer
discovery:

  -----------------------------------------------------------------------
  Area                                Requirement
  ----------------------------------- -----------------------------------
  Reproducibility                     Training and deployment must be
                                      traceable and repeatable

  Portability                         Support managed, customer-cloud and
                                      on-premises targets

  Extensibility                       Add model architectures and
                                      runtimes through adapters/plugins

  Security                            Tenant isolation, encrypted storage
                                      and secure APIs

  Reliability                         Retryable jobs, health checks and
                                      rollback

  Observability                       Metrics, logs, traces and audit
                                      records

  Performance                         Benchmark latency, throughput and
                                      resource use per model

  Cost control                        Measure compute and inference cost
                                      per job and deployment

  Maintainability                     Pin and test runtime and dependency
                                      versions

  Interoperability                    Standard HTTP APIs and documented
                                      artefact formats

  Offline operation                   Support restricted-network
                                      installation where required
  -----------------------------------------------------------------------

Specific service-level objectives, throughput targets, recovery
objectives and availability targets must be established with reference
to actual workloads.

## 17. Implementation Roadmap

### Phase 1: Standardise the model output

-   Define model metadata and manifest schema.
-   Define model registry records.
-   Standardise adapter and merged-model outputs.
-   Record dataset, training and evaluation provenance.
-   Publish a model from the existing training workflow to a repository.

**Exit criterion:** A model produced by the current process can be
identified, retrieved and inspected using a consistent specification.

### Phase 2: Build the deployment CLI

-   Accept a Hugging Face repository or local model directory.
-   Inspect model architecture and format.
-   Validate runtime compatibility.
-   Select vLLM as the initial production runtime.
-   Generate Docker Compose deployment files.
-   Generate API documentation.
-   Build and run the deployment.
-   Execute an inference smoke test.
-   Produce a versioned package.

**Exit criterion:** A compatible model can be packaged and run on a
supported GPU host without manual editing of generated deployment files.

### Phase 3: Add evaluation and validation

-   Integrate benchmark execution.
-   Record latency and throughput.
-   Record resource consumption.
-   Verify output behaviour.
-   Produce evaluation reports.
-   Add checksums and package verification.
-   Add failure diagnostics.

**Exit criterion:** A deployment package is accompanied by reproducible
validation evidence.

### Phase 4: Add Kubernetes deployment

-   Generate Helm charts.
-   Support GPU scheduling.
-   Add probes and resource configuration.
-   Integrate customer secrets and storage.
-   Support rollback.
-   Test in representative Kubernetes environments.

**Exit criterion:** A customer can install a validated model package
into a supported Kubernetes environment.

### Phase 5: Build the management platform

-   Expose CLI capabilities through APIs.
-   Add model and deployment registry.
-   Add job orchestration.
-   Add customer and project management.
-   Add deployment status and monitoring.
-   Add user interface.
-   Add approval and release workflows.

**Exit criterion:** Users can manage the model lifecycle through a
platform rather than manually invoking deployment commands.

### Phase 6: Develop the first AI application

Select one narrow business problem with measurable value.

The application should reuse:

-   Model registry.
-   Inference APIs.
-   Agent and workflow orchestration.
-   Authentication.
-   Monitoring.
-   Audit.
-   Deployment capabilities.

**Exit criterion:** A production-oriented application demonstrates
measurable improvement against a defined business-process baseline.

## 18. Initial MVP Acceptance Criteria

The first Model Deployment Kit MVP should satisfy the following:

1.  Accept a compatible Hugging Face model repository and immutable
    revision.
2.  Accept a local model directory as an alternative input.
3.  Inspect model metadata and identify supported runtime options.
4.  Validate model/runtime compatibility.
5.  Generate a manifest.
6.  Generate a Docker Compose deployment.
7.  Configure a compatible inference runtime.
8.  Support secure retrieval of private model artefacts.
9.  Start the model-serving container on a supported GPU host.
10. Expose an inference API.
11. Pass a defined inference smoke test.
12. Record runtime and model versions.
13. Produce checksums for package artefacts.
14. Generate installation and configuration documentation.
15. Fail clearly when a model or hardware combination is unsupported.

The MVP does not need to include a full web interface, multi-cloud
provisioning, remote management or sophisticated autoscaling.

## 19. Key Technical Decisions Still to Resolve

The following require proof-of-concept work or explicit design
decisions:

  -----------------------------------------------------------------------
  Topic                               Decision required
  ----------------------------------- -----------------------------------
  Model manifest                      Final schema and versioning
                                      approach

  Model registry                      Build internally or adopt an
                                      existing registry

  Artefact storage                    Hugging Face, object storage, or
                                      both

  Runtime selection                   Initial supported model
                                      architectures and vLLM versions

  Adapter deployment                  Serve adapters directly or merge by
                                      default

  Quantisation                        Supported methods and quality
                                      thresholds

  Hardware profiling                  How GPU memory and compute
                                      requirements are estimated

  API gateway                         Build in Spring Boot or use an
                                      established gateway

  Job orchestration                   JobRunr versus a dedicated workflow
                                      engine

  Kubernetes                          Supported versions and GPU
                                      environments

  Secrets                             Integration with customer and cloud
                                      secret managers

  Remote management                   Whether to provide an optional
                                      customer-side management agent

  Offline installation                Exact signed bundle and update
                                      process

  Evaluation                          Standard benchmarks and
                                      customer-specific acceptance tests

  Multi-tenancy                       Shared infrastructure versus
                                      dedicated model instances
  -----------------------------------------------------------------------

## 20. Glossary

  -----------------------------------------------------------------------
  Term                                Meaning
  ----------------------------------- -----------------------------------
  SLM                                 Small Language Model

  LLM                                 Large Language Model

  Fine-tuning                         Adapting a pre-trained model using
                                      additional training data

  LoRA                                Low-Rank Adaptation, a
                                      parameter-efficient fine-tuning
                                      technique

  QLoRA                               Quantised Low-Rank Adaptation

  Adapter                             Additional trained weights used
                                      with a compatible base model

  Merged model                        Base model with adapter weights
                                      merged into its weights

  Inference                           Executing a model to generate an
                                      output

  Inference engine                    Runtime software that loads and
                                      executes a model

  RAG                                 Retrieval-Augmented Generation

  Model registry                      Catalogue of model versions and
                                      associated metadata

  Model provenance                    Traceability of model origin, data,
                                      training and evaluation

  GGUF                                Model file format commonly used in
                                      llama.cpp deployments

  SBOM                                Software Bill of Materials

  Helm                                Kubernetes package manager

  OpenAI-compatible API               API interface following the
                                      relevant OpenAI API conventions
  -----------------------------------------------------------------------

## 21. Summary

The proposed platform consists of three major technical capabilities:

1.  **SLM Factory:** Creates specialised models from customer-provided
    materials and produces versioned, evaluated model artefacts.
2.  **Model Deployment Kit:** Accepts pre-generated models, selects a
    compatible inference runtime, generates deployment configurations
    and produces validated installation packages.
3.  **AI Application Platform:** Uses deployed models alongside other AI
    services, conventional software and workflow orchestration to solve
    specific business problems.

The most important early engineering milestone is to establish a
standard contract between model creation and model deployment.

A model should be registered with complete metadata and provenance. The
deployment product should then be able to consume that model
independently, generate a reproducible package and prove that the
resulting service works.

The initial implementation should focus on a CLI-based deployment
generator, a pinned vLLM runtime, Docker Compose deployment and
automated inference validation. Kubernetes, a management UI, remote
management and application-specific products can follow once the core
process is reliable.

The long-term objective is a repeatable, automated lifecycle:

``` text
CUSTOMER MATERIALS
        |
        v
DATASET GENERATION
        |
        v
MODEL TRAINING
        |
        v
MODEL EVALUATION
        |
        v
MODEL REGISTRY
        |
        v
DEPLOYMENT PACKAGE GENERATION
        |
        v
AUTOMATED VALIDATION
        |
        v
CUSTOMER DEPLOYMENT
        |
        v
MODEL SERVING API
        |
        v
AI APPLICATIONS AND BUSINESS WORKFLOWS
        |
        v
MONITORING, FEEDBACK AND IMPROVEMENT
```

This architecture keeps model creation independent from deployment,
supports models from multiple sources, and provides a common technical
foundation for both product streams.
