# Lab 3: Deploy and Test AI-Q

## Introduction

Deploy AI-Q in FRAG mode in the same namespace and connect it to the RAG services from Lab 2.

### Objectives

- Install AI-Q and verify backend health.
- Open the AI-Q portal and test web and document research.

Estimated Time: 25 minutes.

## Prerequisites

Complete Labs 1 and 2 in this variant. RAG and its local models must be Ready.

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

## Task 2: Open and test AI-Q

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

2. Open `$AIQ_APP_URL` and confirm the Research Assistant loads. Enable **Web
   Search**, ask a research question, and confirm the completed report includes
   web sources and RAG knowledge where applicable.

## Conclusion

AI-Q is reachable and the research workflow has been exercised. A health response alone does not verify report generation.

Continue to [Lab 4: Troubleshoot and Clean Up Workshop Resources](../validate-and-clean-up/validate-and-clean-up-sandbox-lite.md).

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
