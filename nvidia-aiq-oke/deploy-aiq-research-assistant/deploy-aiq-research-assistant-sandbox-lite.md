# Lab 3: Deploy and Test AI-Q

## Introduction

Deploy AI-Q in FRAG mode in the same namespace and connect it to the RAG services from Lab 2. Then act as a cloud architect: use an Oracle document and web research to produce a cited OCI Supercluster evaluation report.

AI-Q supports both short research answers and longer, asynchronous deep-research jobs. A chat answer, an approved plan, or a successful health check is not a completed research report.

> **Validation status:** The detailed deep-research exercise below is a draft for the next end-to-end test, not a previously validated workflow. The October 6 test validated web-search chat after replacing two retired hosted-model references in the backend configuration. That compatible ConfigMap/Helm override still needs to be included in Task 1 before a fresh deployment can reproduce the tested model settings. Do not publish this draft as fully validated until that prerequisite and Tasks 3–4 pass.

### Objectives

- Install AI-Q and verify backend health.
- Distinguish a quick web-search answer from a deep-research report.
- Attach a public document, select research sources, and approve a focused plan.
- Inspect research progress, verify document and web citations, and export the final report.

Estimated Time: 45 minutes, including deployment, document ingestion, research and review. This is a planning estimate; record actual report-generation time during the next test.

## Prerequisites

Complete Labs 1 and 2 in this variant. RAG and its local models must be Ready. Keep the same Cloud Shell session, namespace, and credentials. Have the public OCI Supercluster PDF from Lab 2 available on the computer running your browser.

Use only public workshop material. The workshop portals use HTTP and are not intended for confidential documents or sensitive research questions. No additional GPU, network, or administrator access is needed for this exercise.

## Task 1: Deploy AI-Q 2.1.0 in FRAG mode

1. Install AI-Q into the same assigned namespace. `aiq.appname` must equal the
   assigned namespace; the chart otherwise attempts to create a different
   namespace, which a workshop user is not permitted to use.

    ```bash
    helm upgrade --install aiq210-frag \
      https://helm.ngc.nvidia.com/nvidia/blueprint/charts/aiq2-web-2.1.0.tgz \
      --namespace "$LAB_NAMESPACE" \
      --username '$oauthtoken' \
      --password "$NGC_API_KEY" \
      --reset-values \
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

2. Correct the PostgreSQL initializer image and wait for the frontend and
   backend to become ready:

    ```bash
    kubectl patch deployment aiq-backend --type=json \
      -p='[{"op":"replace","path":"/spec/template/spec/initContainers/0/image","value":"docker.io/bitnami/postgresql:latest"}]'

    kubectl rollout status deployment/aiq-postgres --timeout=20m
    kubectl rollout status deployment/aiq-backend --timeout=20m
    kubectl rollout status deployment/aiq-frontend --timeout=20m
    ```

## Task 2: Check health and test a short answer

1. Validate the application from Cloud Shell without exposing a new public
   endpoint:

    ```bash
    kubectl port-forward service/aiq-backend 8000:8000
    ```

    In another terminal:

    ```bash
    curl -sf http://127.0.0.1:8000/health
    curl -sf http://127.0.0.1:8000/v1/knowledge/health
    ```

    Stop the port-forward with `Ctrl+C` after the checks succeed.

2. Open `$AIQ_APP_URL` and confirm the Research Assistant loads. Open **Data Sources**, select **Connections**, and enable **Web Search**. Leave unrelated connections disabled.

3. Submit this short question:

    ```text
    In two or three sentences, what is OCI Supercluster? Use an official Oracle web source and cite it.
    ```

    Confirm that an answer and a web citation appear. This checks the basic chat, model, and web-search path. It does not satisfy the deep-research completion checks below. If this prompt unexpectedly produces a plan instead, do not start an extra report; proceed with the exercise in Task 3.

## Task 3: Prepare sources and approve a deep-research plan

1. Start a **New Session** for the report. Use this same session through Task 4; attached files belong to a session and should not be assumed available in a different one.

2. Open **Data Sources**. Under **Connections**, enable **Web Search** and **Knowledge Base**. The FRAG configuration connects the knowledge source to the RAG services in your assigned namespace. Leave other sources disabled; this exercise does not require academic-paper search or another API key.

3. Select **Files** and use the upload area to attach the [OCI Supercluster PDF](https://www.oracle.com/a/ocom/docs/cloud/accelerate-ai-with-oci-supercluster.pdf) used in Lab 2. If files already exist, use **+ Add File**. Wait until the file has finished processing and is available for research, with no upload or ingestion error.

    Upload the PDF into this AI-Q session even if you already uploaded it into the RAG portal. Lab 2's `OCI Documentation` collection and an AI-Q session's file collection are not interchangeable merely because both applications use the same RAG backend. Do not change the deployment's `COLLECTION_NAME`, create an ingress, or request extra permissions to attach the document.

    If the Files panel says the backend must be set up, or **Knowledge Base** is unavailable, stop and check `/v1/knowledge/health` using Task 2. Ask the facilitator for help rather than proceeding with a web-only report and treating it as a document-research pass.

4. Close the source panel and submit the following focused request. The scenario asks for analysis and a recommendation, rather than a simple definition, to exercise the deep-research path:

    ```text
    Perform deep research and prepare a decision brief titled
    "OCI Supercluster for a Small AI Research Team".

    Scenario: A team is evaluating OCI for multi-node GPU model training
    and later serving a document-based research assistant on OKE.

    Use the attached OCI Supercluster PDF and official Oracle web sources.
    Treat the PDF as background, not proof that its hardware examples,
    availability, or scale figures are still current. Distinguish dated
    document claims from current web evidence.

    Aim for about 800-1,200 words in these sections:
    1. Executive summary and recommendation.
    2. What Supercluster provides: compute, networking, and storage.
    3. How multi-node training differs from hosting this workshop's
       two-GPU RAG and AI-Q applications on a shared OKE cluster.
    4. Three prerequisites or risks to verify before adoption, including
       GPU capacity and tenancy limits.
    5. A short proof-of-concept checklist and references.

    Cite at least one specific passage from the attached PDF and two
    distinct official web pages. If you cannot retrieve the document or
    verify a claim, say so; do not invent citations, pricing, availability,
    benchmark results, or guarantees. Keep the scope to this brief.
    ```

5. If the agent asks clarifying questions, answer in the chat input. For example:

    ```text
    The audience is a cloud architect planning a small proof of concept.
    Cover training infrastructure and OKE application serving separately.
    Use only the attached public PDF and official Oracle web sources.
    Exclude pricing, regional availability promises, and benchmark comparisons.
    Keep the report to about 800-1,200 words.
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
kubectl --namespace "$LAB_NAMESPACE" logs deployment/aiq-backend --since=15m --tail=100
```

Treat logs as potentially sensitive; redact them before sharing. HTTP 410 from a hosted model indicates a retired endpoint: contact the facilitator rather than changing GPUs or network resources. For 401/403, check the configured credentials without displaying their values. For 429 or upstream timeouts, avoid immediate repeated submissions and ask the facilitator to inspect provider capacity or limits. Do not reinstall the charts, delete document collections, or increase namespace quotas as a first response to a report error.

<!-- Capture after live validation: AI-Q Connections and session file-ready state; clarification/approved Plan; running Tasks/job banner; completed Report with document and web citations; export footer. Keep sanitized screenshots in ../shared-images/ and at most 1180x1180. Exact deployed-image labels, runtime, session upload, report completion and exports remain to be verified. -->

### Learn from the result

Discuss which claims came from the PDF and which required newer web evidence. Why is a cited recommendation more useful than a fluent but unsupported answer? What remains a human decision even after the report completes?

For reference, see NVIDIA's [AI-Q 2.1.0 UI source and interface notes](https://github.com/NVIDIA-AI-Blueprints/aiq/blob/v2.1.0/frontends/ui/README.md) and [asynchronous job API](https://github.com/NVIDIA-AI-Blueprints/aiq/blob/v2.1.0/frontends/aiq_api/README.md). The task labels are based on the versioned UI source; confirm them against the deployed image during the next test.

## Conclusion

After completing all checks, you have exercised AI-Q deployment, short-answer research, session document ingestion, plan approval, asynchronous report generation, citation review, and report export. Keep the exported brief as your workshop outcome. If any check failed, record it and work with the facilitator before claiming full completion.

Continue to [Lab 4: Troubleshoot and Clean Up Workshop Resources](../validate-and-clean-up/validate-and-clean-up-sandbox-lite.md).

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
