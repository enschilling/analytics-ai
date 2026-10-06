# Lab 3: Deploy and Test AI-Q

## Introduction

Deploy AI-Q in FRAG mode in the same namespace and connect it to the RAG services from Lab 2.

### Objectives

- Install AI-Q and verify backend health.
- Open the AI-Q portal and test web and document research.

Estimated Time: 30 minutes.

## Prerequisites

Complete Labs 1 and 2 in this variant. RAG and its local models must be Ready.

## Task 1: Deploy AI-Q Research Assistant

1. Deploy AI-Q 2.1.0 into the same namespace. AI-Q sends FRAG requests to the
   local RAG services. `aiq.appname` must be the assigned namespace to prevent
   the chart from attempting to use a second namespace.

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

2. Correct the PostgreSQL initializer image and wait for AI-Q.

    ```bash
    kubectl patch deployment aiq-backend --type=json \
      -p='[{"op":"replace","path":"/spec/template/spec/initContainers/0/image","value":"docker.io/bitnami/postgresql:latest"}]'

    kubectl rollout status deployment/aiq-postgres --timeout=20m
    kubectl rollout status deployment/aiq-backend --timeout=20m
    kubectl rollout status deployment/aiq-frontend --timeout=20m
    ```

## Task 2: Open and test AI-Q

1. Validate the backend from Cloud Shell without creating another public
   endpoint:

    ```bash
    kubectl port-forward service/aiq-backend 8000:8000
    ```

    In another terminal, run:

    ```bash
    curl -sf http://127.0.0.1:8000/health
    curl -sf http://127.0.0.1:8000/v1/knowledge/health
    ```

    Stop the port-forward with `Ctrl+C` after both checks succeed.

2. Open `$AIQ_APP_URL`. Enable **Web Search**, generate a report about OCI AI
   capabilities, then upload a born-digital PDF, DOCX, TXT, or Markdown file
   and ask a question that uses both the uploaded content and web search.

## Conclusion

AI-Q is reachable and the research workflow has been exercised. A health response alone does not verify report generation.

Continue to [Lab 4: Troubleshoot and Clean Up Workshop Resources](../validate-and-clean-up/validate-and-clean-up-sandbox.md).

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
