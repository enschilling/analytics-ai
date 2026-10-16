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
    kubectl auth can-i create configmaps --namespace "$LAB_NAMESPACE"
    kubectl auth can-i patch configmaps --namespace "$LAB_NAMESPACE"
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
    mkdir -p "$HOME/aiq-workshop"
    cd "$HOME/aiq-workshop"
    export AIQ_FILES_BASE="https://raw.githubusercontent.com/enschilling/analytics-ai/c0173f4bbcb757b7c9be206f9efcb70d2e915d09/nvidia-aiq-oke/deploy-aiq-research-assistant/files/sandbox-lite"
    curl --fail --location --retry 2 --max-time 60 \
      "$AIQ_FILES_BASE/config_web_frag.yml" -o config_web_frag.yml
    curl --fail --location --retry 2 --max-time 60 \
      "$AIQ_FILES_BASE/aiq-workshop-values.yaml" -o aiq-workshop-values.yaml
    ```

    If a download fails, stop and ask the facilitator for the published files.
    Do not continue with an empty or older configuration.

3. Create the ConfigMap in your namespace. The same name is safe for every
   attendee because ConfigMaps are namespace-scoped.

    ```bash
    kubectl --namespace "$LAB_NAMESPACE" create configmap aiq-workshop-config \
      --from-file=config_web_frag.yml=config_web_frag.yml \
      --dry-run=client -o yaml |
    kubectl --namespace "$LAB_NAMESPACE" apply -f -
    ```

    Expected result: `configmap/aiq-workshop-config created` or `configured`.

## Task 2: Deploy AI-Q in FRAG mode

1. Install AI-Q into the same assigned namespace. `aiq.appname` must equal
   `$LAB_NAMESPACE`; otherwise the chart attempts to create another namespace.
   The values file mounts your ConfigMap into the backend while preserving the
   chart's PostgreSQL initializer volume.

    ```bash
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
    ```

2. Correct the PostgreSQL initializer image, restart the backend so it loads
   the ConfigMap even when you repeat the lab, and wait for readiness:

    ```bash
    kubectl --namespace "$LAB_NAMESPACE" patch deployment aiq-backend --type=json \
      -p='[{"op":"replace","path":"/spec/template/spec/initContainers/0/image","value":"docker.io/bitnami/postgresql:latest"}]'
    ```
    ```bash
    kubectl rollout status deployment/aiq-postgres --timeout=20m
    ```
    ```bash
    kubectl rollout status deployment/aiq-backend --timeout=20m
    ```
    ```bash
    kubectl rollout status deployment/aiq-frontend --timeout=20m
    ```

3. Confirm the backend's mounted model settings without displaying credentials:

    ```bash
    kubectl --namespace "$LAB_NAMESPACE" exec deployment/aiq-backend -c backend -- \
      /app/.venv/bin/python -c 'from pathlib import Path; print("\n".join(line.strip() for line in Path("/app/configs/config_web_frag.yml").read_text().splitlines() if line.lstrip().startswith("model_name:")))'
    ```

    Use the backend's installed Python runtime; its container does not include
    `grep`. This command prints only model names, not credential settings.

    ```bash
    curl -sf http://127.0.0.1:8000/health
    ```
    ```bash
    curl -sf http://127.0.0.1:8000/v1/knowledge/health
    ```

## Task 3: Check health and generate a cited short answer

1. In Cloud Shell, start a local port-forward. This does not create a public
   endpoint or load balancer:

    ```bash
    kubectl --namespace "$LAB_NAMESPACE" port-forward service/aiq-backend 8000:8000
    ```

    Leave it running. Open another Cloud Shell terminal and run:

    ```bash
    curl --fail --show-error --max-time 30 http://127.0.0.1:8000/health
    curl --fail --show-error --max-time 30 http://127.0.0.1:8000/v1/knowledge/health
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
    In two or three sentences, what is OCI Supercluster? Use an official Oracle web source and cite it.
    ```

6. Open **Show Research**, then **Plan**, and review the proposed outline. It should include the document and web evidence, the training-versus-serving distinction, risks, and a recommendation. Answer any remaining clarification request before waiting for a job to start.

7. When the agent requests plan approval, select **Approve**, or reply `approve` in the chat input if the buttons are absent. If the plan ignores the document or expands into an unrelated comparison, request a correction before approving. Do not reject the plan unless you intend to cancel that request.

    Expected result: a **Starting Deep Research** banner with a job ID. Record the job ID and start time for troubleshooting. A short chat answer without an asynchronous job is not a report-generation pass. If necessary, follow up once: "Please perform deep research and produce the report requested above, rather than a short answer." If no job starts after clarification and approval, ask the facilitator to investigate.

## Task 4: Follow progress, verify the report, and export it

1. Select **View Progress** or open **Show Research**. Inspect the **Tasks** tab for research activity and the **Citations** tab for collected sources. Keep the report in the same session. Do not repeatedly resubmit the prompt or select **Stop Researching** while it is making progress.

    Research is asynchronous and can take several minutes. Source collection and drafting can be uneven; a temporarily quiet display does not by itself mean the job failed. If there is no visible progress for five minutes, or the run exceeds 15 minutes, check the status with the facilitator. These are workshop investigation thresholds, not measured completion guarantees or automatic cancellation times.

2. Wait for **Report Completed!**, then select **View Report**. Open the **Report** tab and read the full result. Intermediate research notes, a plan preview, and an error banner do not count as the final report.

3. Verify the result against the exercise, not just the presence of text:

    - The report has a summary, infrastructure discussion, training-versus-serving comparison, risks, and a proof-of-concept checklist. Exact headings and wording may vary.
    - At least one citation identifies the attached PDF. Compare its cited claim with the actual document.
    - At least two distinct official Oracle web pages are cited. Open the links and verify that they support the associated claims; a list of URLs alone is insufficient.
    - The report does not confuse this two-GPU workshop with a multi-node training deployment. GPU quota is a usage limit, not a reservation of physical GPU capacity.
    - Unsupported claims and differences between the dated PDF and current sources are disclosed. Generated citations still require human review.

    If the job completes but the PDF was not used, record a **document-grounding failure**, not a full pass. Similarly, missing web evidence is a **web-source failure**. Correct the source setup and retry once with the facilitator; avoid launching many concurrent reports.

4. Select **Markdown** in the report export footer. Open the downloaded file locally and confirm that it contains the complete report and its references. You may also select **PDF** and review that export; PDF export is an additional check, not a substitute for confirming research completion.

5. Record the job ID, elapsed research time, the exported report, and the three source checks (one document and two web pages). Before sharing screenshots, remove reservation-specific hostnames, personal information, and unrelated browser content. Never include API keys or Kubernetes Secret values.

### Completion checks and troubleshooting

The deep-research exercise passes only when an asynchronous job completes, the final report is visible, both document and web grounding are verified, and the Markdown export opens successfully. A successful chat response or health endpoint alone is not enough.

If **Report Failed to Complete** appears, record the job ID and error time. Inspect **Tasks** and the visible error details, then use these namespace-scoped checks in Cloud Shell:

```bash
kubectl --namespace "$LAB_NAMESPACE" get pods
```
```bash
kubectl --namespace "$LAB_NAMESPACE" logs deployment/aiq-backend --since=15m --tail=100
```

Treat logs as potentially sensitive; redact them before sharing. HTTP 410 from a hosted model indicates a retired endpoint: contact the facilitator rather than changing GPUs or network resources. For 401/403, check the configured credentials without displaying their values. For 429 or upstream timeouts, avoid immediate repeated submissions and ask the facilitator to inspect provider capacity or limits. Do not reinstall the charts, delete document collections, or increase namespace quotas as a first response to a report error.

<!-- Capture after live validation: AI-Q Connections and session file-ready state; clarification/approved Plan; running Tasks/job banner; completed Report with document and web citations; export footer. Keep sanitized screenshots in ../shared-images/ and at most 1180x1180. Exact deployed-image labels, runtime, session upload, report completion and exports remain to be verified. -->

### Learn from the result

Discuss which claims came from the PDF and which required newer web evidence. Why is a cited recommendation more useful than a fluent but unsupported answer? What remains a human decision even after the report completes?

For reference, see NVIDIA's [AI-Q 2.1.0 UI source and interface notes](https://github.com/NVIDIA-AI-Blueprints/aiq/blob/v2.1.0/frontends/ui/README.md) and [asynchronous job API](https://github.com/NVIDIA-AI-Blueprints/aiq/blob/v2.1.0/frontends/aiq_api/README.md). The task labels are based on the versioned UI source; confirm them against the deployed image during the next test.

## Conclusion

You deployed AI-Q on OKE using the workshop's model configuration, verified its
RAG service connection, and generated a short answer with a source you reviewed.
Together with Lab 2, you have exercised document-grounded RAG and tool-assisted
web research. You have not generated or validated a deep-research report.

Continue to [Lab 4: Troubleshoot and Clean Up Workshop Resources](../validate-and-clean-up/validate-and-clean-up-sandbox-lite.md).

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
