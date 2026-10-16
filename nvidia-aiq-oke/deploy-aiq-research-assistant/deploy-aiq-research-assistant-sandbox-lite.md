# Lab 3: Deploy and Test AI-Q

## Introduction

Deploy NVIDIA AI-Q 2.1.0 in FRAG (Foundational RAG) mode on Oracle Kubernetes
Engine (OKE), connect it to the RAG services from Lab 2, and generate a short
answer with a public web citation. AI-Q adds an agent workflow that can choose
tools, search sources, and synthesize a response rather than only answer from
an uploaded document.

This lab demonstrates deployment, service connectivity, and short-answer web
research. AI-Q also supports longer, multi-step deep-research reports, but
report generation and export are outside this workshop's exercise scope.

### Objectives

- Create a namespace-scoped ConfigMap with the workshop's hosted-model settings.
- Deploy AI-Q on OKE and verify its connection to the RAG backend.
- Generate a short web-research answer and verify its citation.
- Explain which components run on OCI and which use NVIDIA-hosted APIs.

Estimated Time: 25 minutes. Deployment and hosted API response times can vary.

## Prerequisites

Complete Labs 1 and 2. RAG and its local models must be Ready. Keep the same
Cloud Shell session, namespace, kubeconfig, and credentials. The
`aiq-credentials` Secret from Lab 1 and `ngc-secret` from Lab 2 must exist.

Use only public workshop material. The portals use HTTP and are not intended
for confidential documents or sensitive questions. This lab does not require
additional GPUs or permission to change OCI networking or shared infrastructure.

## Task 1: Prepare the AI-Q model configuration

The original hosted model endpoints used by this version of AI-Q were no
longer available during workshop validation. The supplied configuration uses
`nvidia/nemotron-3-super-120b-a12b`, which produced a short answer with an Oracle
web citation in the October 12, 2026 test. It preserves AI-Q's existing agent
aliases and RAG integration; it does not require rebuilding container images.

The ConfigMap stores application settings, not credentials. AI-Q continues to
read the NVIDIA and Tavily API keys from the Secret created in Lab 1.

1. Confirm that you can create and patch ConfigMaps in your assigned namespace:

    ```bash
    <copy>
    kubectl auth can-i create configmaps --namespace "$LAB_NAMESPACE"
    kubectl auth can-i patch configmaps --namespace "$LAB_NAMESPACE"
    </copy>
    ```

    Both commands must return `yes`. If either returns `no`, ask the facilitator
    to check your namespace role; do not switch to an administrator account.

2. Download the [workshop configuration](files/sandbox-lite/config_web_frag.yml)
   and [Helm values](files/sandbox-lite/aiq-workshop-values.yaml) into Cloud Shell.
   Copy the complete block:

    <!-- PRE-MERGE: Temporary verified fork URL while these assets await upstream publication.
    Before merging upstream, restore AIQ_FILES_BASE to
    https://raw.githubusercontent.com/oracle-livelabs/analytics-ai/main/nvidia-aiq-oke/deploy-aiq-research-assistant/files/sandbox-lite
    and include config_web_frag.yml plus aiq-workshop-values.yaml in the same change.
    Check both canonical downloads after publication, then remove this reminder. -->

    ```bash
    <copy>
    mkdir -p "$HOME/aiq-workshop"
    cd "$HOME/aiq-workshop"
    export AIQ_FILES_BASE="https://raw.githubusercontent.com/enschilling/analytics-ai/c0173f4bbcb757b7c9be206f9efcb70d2e915d09/nvidia-aiq-oke/deploy-aiq-research-assistant/files/sandbox-lite"
    curl --fail --location --retry 2 --max-time 60 \
      "$AIQ_FILES_BASE/config_web_frag.yml" -o config_web_frag.yml
    curl --fail --location --retry 2 --max-time 60 \
      "$AIQ_FILES_BASE/aiq-workshop-values.yaml" -o aiq-workshop-values.yaml
    </copy>
    ```

    If a download fails, stop and ask the facilitator for the published files.
    Do not continue with an empty or older configuration.

3. Create the ConfigMap in your namespace. The same name is safe for every
   attendee because ConfigMaps are namespace-scoped.

    ```bash
    <copy>
    kubectl --namespace "$LAB_NAMESPACE" create configmap aiq-workshop-config \
      --from-file=config_web_frag.yml=config_web_frag.yml \
      --dry-run=client -o yaml |
    kubectl --namespace "$LAB_NAMESPACE" apply -f -
    </copy>
    ```

    Expected result: `configmap/aiq-workshop-config created` or `configured`.

## Task 2: Deploy AI-Q in FRAG mode

1. Install AI-Q into the same assigned namespace. `aiq.appname` must equal
   `$LAB_NAMESPACE`; otherwise the chart attempts to create another namespace.
   The values file mounts your ConfigMap into the backend while preserving the
   chart's PostgreSQL initializer volume.

    ```bash
    <copy>
    helm upgrade --install aiq210-frag \
      https://helm.ngc.nvidia.com/nvidia/blueprint/charts/aiq2-web-2.1.0.tgz \
      --namespace "$LAB_NAMESPACE" \
      --username '$oauthtoken' \
      --password "$NGC_API_KEY" \
      --reset-values \
      --values aiq-workshop-values.yaml \
      --set aiq.appname="$LAB_NAMESPACE" \
      --set aiq.project.deploymentTarget=kind \
      --set 'aiq.apps.backend.imagePullSecrets[0].name=ngc-secret' \
      --set 'aiq.apps.frontend.imagePullSecrets[0].name=ngc-secret' \
      --set-string aiq.apps.backend.env.CONFIG_FILE=configs/config_web_frag.yml \
      --set-string aiq.apps.backend.env.RAG_SERVER_URL=http://rag-server:8081/v1 \
      --set-string aiq.apps.backend.env.RAG_INGEST_URL=http://ingestor-server:8082/v1 \
      --set-string aiq.apps.backend.env.COLLECTION_NAME=aiq_frag_workshop \
      --set aiq.apps.postgres.image.repository=docker.io/bitnami/postgresql \
      --set aiq.apps.frontend.service.type=ClusterIP \
      --set aiq.apps.backend.ingress.enabled=false \
      --set aiq.apps.frontend.ingress.enabled=false \
      --set aiq.apps.postgres.ingress.enabled=false \
      --wait=false
    </copy>
    ```

2. Correct the PostgreSQL initializer image, restart the backend so it loads
   the ConfigMap even when you repeat the lab, and wait for readiness:

    ```bash
    <copy>
    kubectl --namespace "$LAB_NAMESPACE" patch deployment aiq-backend --type=json \
      -p='[{"op":"replace","path":"/spec/template/spec/initContainers/0/image","value":"docker.io/bitnami/postgresql:latest"}]'
    </copy>
    ```
    ```bash
    <copy>
    kubectl --namespace "$LAB_NAMESPACE" rollout restart deployment/aiq-backend
    </copy>
    ```
    ```bash
    <copy>
    kubectl --namespace "$LAB_NAMESPACE" rollout status deployment/aiq-postgres --timeout=20m
    </copy>
    ```
    ```bash
    <copy>
    kubectl --namespace "$LAB_NAMESPACE" rollout status deployment/aiq-backend --timeout=20m
    </copy>
    ```
    ```bash
    <copy>
    kubectl --namespace "$LAB_NAMESPACE" rollout status deployment/aiq-frontend --timeout=20m
    </copy>
    ```

3. Confirm the backend's mounted model settings without displaying credentials:

    ```bash
    <copy>
    kubectl --namespace "$LAB_NAMESPACE" exec deployment/aiq-backend -c backend -- \
      /app/.venv/bin/python -c 'from pathlib import Path; print("\n".join(line.strip() for line in Path("/app/configs/config_web_frag.yml").read_text().splitlines() if line.lstrip().startswith("model_name:")))'
    </copy>
    ```

    Use the backend's installed Python runtime; its container does not include
    `grep`. This command prints only model names, not credential settings.

    Expected result: three `model_name` entries, all using
    `nvidia/nemotron-3-super-120b-a12b`. The internal alias names are retained for
    compatibility; they no longer identify the actual hosted models being called.

## Task 3: Check health and generate a cited short answer

1. In Cloud Shell, start a local port-forward. This does not create a public
   endpoint or load balancer:

    ```bash
    <copy>
    kubectl --namespace "$LAB_NAMESPACE" port-forward service/aiq-backend 8000:8000
    </copy>
    ```

    Leave it running. Open another Cloud Shell terminal and run:

    ```bash
    <copy>
    curl --fail --show-error --max-time 30 http://127.0.0.1:8000/health
    curl --fail --show-error --max-time 30 http://127.0.0.1:8000/v1/knowledge/health
    </copy>
    ```

    Expected results: `status` is `healthy` in both responses, and the knowledge
    response identifies `foundational_rag`. This confirms service connectivity,
    not a completed research request. Return to the port-forward terminal and
    press `Ctrl+C` when both checks pass.

2. Open `$AIQ_APP_URL` from the LiveLabs Resources tab. Confirm the Research
   Assistant loads. Open **Data Sources**, select **Connections**, and enable
   **Web Search**. Leave unrelated connections disabled.

3. Submit the short question used in workshop validation:

    ```text
    <copy>
    In two or three sentences, what is OCI Supercluster? Use an official Oracle web source and cite it.
    </copy>
    ```

    Expected result: a short answer with an Oracle source citation. This request
    uses web search; it does not test retrieval from the PDF you uploaded in the
    RAG portal. The two applications' document collections are not automatically
    interchangeable.

4. Open the citation and check that the Oracle page or PDF supports the answer.
   Notice the source date: a dated document is not a guarantee of current GPU
   availability, pricing, or performance. A fluent answer alone is not sufficient
   evidence. Exact wording and sources can vary between runs.

    If the request stalls for five minutes, fails, or produces a plan instead of
    a short answer, ask the facilitator for help. Do not approve a report or
    repeatedly resubmit the question. Hosted-model availability and rate limits
    can affect responses even when all Kubernetes pods are Ready.

## Understand the value: Oracle + NVIDIA

This exercise combines Oracle's managed Kubernetes and GPU infrastructure with
NVIDIA's application blueprints, inference microservices, and research-agent
software. It illustrates the integration described in the
[Oracle + NVIDIA partnership](https://www.oracle.com/cloud/nvidia/).

| Component | Where it runs in this workshop | What it provides |
| --- | --- | --- |
| OKE, namespace controls, and persistent storage | OCI | A shared platform for deploying and operating the applications with scoped attendee access. |
| NVIDIA RAG and local NIM models | OKE on OCI, including two NVIDIA A100 GPUs | Document retrieval, local generation, and embeddings exercised in Lab 2. |
| AI-Q frontend, backend, and PostgreSQL | OKE on OCI | The research-assistant application and agent orchestration. |
| AI-Q model inference and web search | NVIDIA-hosted API and Tavily | Hosted reasoning and public-source search for the short answer in this lab. |

The value is not simply another chat screen: NVIDIA supplies reusable agent and
inference components, while OCI provides the platform on which you deploy,
connect, and operate them. This two-GPU profile uses hosted AI-Q inference to
avoid deploying an additional large research model locally. It does not mean
that all inference runs on OCI or that this sandbox is production-hardened.

Discuss how Lab 2's document-grounded answer differs from this web-research
answer. Which source supports each response, and what would you need to verify
before using it to make a business decision?

### Beyond this lab: deep research

AI-Q also supports longer workflows that plan research, gather evidence, and
produce citation-backed reports. This workshop does not ask you to generate or
export one: the replacement configuration passed the short-answer test, but
full report completion was not validated. A healthy deployment or cited short
answer should not be presented as a completed deep-research exercise.

Explore NVIDIA's [AI-Q 2.1.0 overview and architecture](https://github.com/NVIDIA-AI-Blueprints/aiq/tree/v2.1.0#overview)
and [asynchronous deep-research documentation](https://github.com/NVIDIA-AI-Blueprints/aiq/tree/v2.1.0#async-deep-research-jobs)
for further learning outside this timed workshop.

<!-- Before publication: publish both files/sandbox-lite assets with this guide
and run the revised copy-and-paste path in a fresh reservation. Capture sanitized
screenshots of mounted model settings, healthy endpoints, and the short answer
with its Oracle citation; use ../shared-images/ and dimensions <=1180x1180.
October 12 validated the underlying live configuration, not this rewritten guide.
Group concurrency and hosted API capacity still require event testing. -->

## Conclusion

You deployed AI-Q on OKE using the workshop's model configuration, verified its
RAG service connection, and generated a short answer with a source you reviewed.
Together with Lab 2, you have exercised document-grounded RAG and tool-assisted
web research. You have not generated or validated a deep-research report.

Continue to [Lab 4: Troubleshoot and Clean Up Workshop Resources](../validate-and-clean-up/validate-and-clean-up-sandbox-lite.md).

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
