# Lab 2: Deploy and Test RAG

## Introduction

Deploy the seven-GPU RAG profile and validate document ingestion and question answering through the platform-owned HTTP route.

### Objectives

- Install RAG and wait for the local model services.
- Upload the OCI PDF and inspect the answer's document sources.

Estimated Time: 70 minutes, including model startup.

## Prerequisites

Complete Lab 1 in this variant. Keep its Cloud Shell session and exported variables available.

## Task 1: Deploy the seven-GPU RAG profile

1. Confirm Cloud Shell is using Helm 3.

    ```bash
    helm version --short
    ```

    Continue only when the client major version is `v3`.

2. Deploy RAG into the assigned namespace. The frontend must remain
   `ClusterIP`; the workshop platform exposes it through the pre-created
   `nip.io` route.

    ```bash
    helm upgrade --install rag \
      https://helm.ngc.nvidia.com/nvidia/blueprint/charts/nvidia-blueprint-rag-v2.2.0.tgz \
      --namespace "$LAB_NAMESPACE" \
      --username '$oauthtoken' \
      --password "$NGC_API_KEY" \
      --reset-values \
      --set-string imagePullSecret.password="$NGC_API_KEY",ngcApiSecret.password="$NGC_API_KEY" \
      --set frontend.service.type=ClusterIP \
      --set nim-llm.enabled=true \
      --set nim-llm.image.repository=nvcr.io/nim/nvidia/llama-3.3-nemotron-super-49b-v1 \
      --set nim-llm.image.tag=1.8.5 \
      --set 'nim-llm.resources.limits.nvidia\.com/gpu=4,nim-llm.resources.requests.nvidia\.com/gpu=4' \
      --set nvidia-nim-llama-32-nv-embedqa-1b-v2.enabled=true \
      --set nvidia-nim-llama-32-nv-embedqa-1b-v2.image.tag=1.9.0 \
      --set 'nvidia-nim-llama-32-nv-embedqa-1b-v2.resources.limits.nvidia\.com/gpu=1,nvidia-nim-llama-32-nv-embedqa-1b-v2.resources.requests.nvidia\.com/gpu=1' \
      --set text-reranking-nim.enabled=true \
      --set text-reranking-nim.image.tag=1.7.0 \
      --set 'text-reranking-nim.resources.limits.nvidia\.com/gpu=1,text-reranking-nim.resources.requests.nvidia\.com/gpu=1' \
      --set ingestor-server.enabled=true \
      --set ingestor-server.envVars.APP_VECTORSTORE_ENABLEGPUINDEX=False,ingestor-server.envVars.APP_VECTORSTORE_ENABLEGPUSEARCH=False \
      --set ingestor-server.envVars.APP_NVINGEST_EXTRACTTEXT=True,ingestor-server.envVars.APP_NVINGEST_EXTRACTTABLES=False,ingestor-server.envVars.APP_NVINGEST_EXTRACTCHARTS=False,ingestor-server.envVars.APP_NVINGEST_EXTRACTIMAGES=False,ingestor-server.envVars.APP_NVINGEST_EXTRACTINFOGRAPHICS=False \
      --set ingestor-server.envVars.APP_NVINGEST_ENABLEPDFSPLITTER=True,ingestor-server.envVars.APP_NVINGEST_CHUNKSIZE=1024,ingestor-server.envVars.APP_NVINGEST_CHUNKOVERLAP=150 \
      --set ingestor-server.nv-ingest.nemoretriever-page-elements-v2.deployed=true \
      --set ingestor-server.nv-ingest.nemoretriever-page-elements-v2.image.tag=1.4.0 \
      --set 'ingestor-server.nv-ingest.nemoretriever-page-elements-v2.resources.limits.nvidia\.com/gpu=1,ingestor-server.nv-ingest.nemoretriever-page-elements-v2.resources.requests.nvidia\.com/gpu=1' \
      --set ingestor-server.nv-ingest.nemoretriever-graphic-elements-v1.deployed=false \
      --set ingestor-server.nv-ingest.nemoretriever-table-structure-v1.deployed=false \
      --set ingestor-server.nv-ingest.paddleocr-nim.deployed=false \
      --set-string ingestor-server.nv-ingest.milvus.image.all.repository=docker.io/milvusdb/milvus \
      --set-string ingestor-server.nv-ingest.milvus.image.all.tag=v2.5.3 \
      --set-string ingestor-server.nv-ingest.milvus.image.tools.repository=docker.io/milvusdb/milvus-config-tool \
      --set-string ingestor-server.nv-ingest.milvus.minio.image.repository=docker.io/minio/minio \
      --set-string ingestor-server.nv-ingest.redis.image.repository=docker.io/library/redis \
      --set-string ingestor-server.nv-ingest.redis.image.tag=8.2.1 \
      --set-string envVars.ENABLE_RERANKER=True \
      --wait=false
    ```

3. Wait for the local models and application services. The four-GPU generation
   model can take 10–15 minutes to initialize after its images and model files
   download; allow additional time on a cold node.

    ```bash
    kubectl get pods --watch
    ```

    Press `Ctrl+C` when the pods are stable, then run:

    ```bash
    kubectl get deployments,statefulsets
    kubectl get pods -o wide
    kubectl get service rag-frontend
    ```

## Task 2: Open and validate RAG

1. Open the pre-provisioned RAG URL:

    ```bash
    printf '%s\n' "$RAG_APP_URL"
    ```

2. Create a collection named `OCI Documentation`. Download the [OCI
   Supercluster PDF](https://www.oracle.com/a/ocom/docs/cloud/accelerate-ai-with-oci-supercluster.pdf), upload it, and ask:

    ```text
    What is OCI Supercluster and what makes it unique?
    ```

    Confirm that the answer includes citations from the document.

    ![Create a collection in the RAG Playground](../shared-images/rag-new-collection-dialog.png)

    ![Review RAG answer sources and citations](../shared-images/rag-citations-panel.png)

## Conclusion

The RAG application is reachable and can answer a question using the uploaded document.

Continue to [Lab 3: Deploy and Test AI-Q](../deploy-aiq-research-assistant/deploy-aiq-research-assistant-sandbox.md).

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
