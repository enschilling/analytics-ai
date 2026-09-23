# Deploy the NVIDIA Deep Research AI Agent with Local Models on Oracle Kubernetes Engine

## Introduction

In this workshop, you deploy the NVIDIA RAG Blueprint and NVIDIA AI-Q Research
Assistant in the namespace assigned to your LiveLabs reservation. Unlike the
API-oriented workshop, this profile runs the generation, embedding, reranking,
and document page-elements models locally on the shared OKE GPU cluster.

The workshop platform owns the OKE cluster, GPU nodes, NVIDIA and ECK
operators, Istio ingress, public HTTP routes, and network configuration. You
deploy only application resources in your assigned namespace. Do not create a
VCN, load balancer, namespace, Istio `Gateway`, Istio `VirtualService`, or DNS
record.

### Objectives

- Connect Cloud Shell to your assigned namespace.
- Deploy the seven-GPU local RAG profile.
- Validate the local NIM services and RAG document workflow.
- Deploy AI-Q 2.1.0 in FRAG mode, connected to the local RAG services.
- Open the platform-provided RAG and AI-Q `nip.io` HTTP URLs.

Estimated Time: 120 minutes. Initial image pulls, model downloads, and
TensorRT engine initialization are the longest steps.

## Local model profile and resource use

| Component | GPUs | Purpose |
| --- | ---: | --- |
| Nemotron Super 49B generation NIM | 4 | RAG answers and AI-Q synthesis through RAG |
| Embedding NIM | 1 | Document and query embeddings |
| Reranking NIM | 1 | Retrieval quality |
| Page-elements NIM | 1 | PDF text extraction |
| CPU services | 0 | RAG services, Milvus, AI-Q, PostgreSQL, and Phoenix |
| **Total** | **7** | One GPU remains spare on an eight-A100 node |

The reservation must have a seven-GPU quota and adequate CPU, memory, storage,
and PVC quota before the workshop begins. This profile cannot be used on the
two-GPU sandbox-api reservation.

## Prerequisites

The LiveLabs Resources tab or startup output supplies:

- `LAB_NAMESPACE`
- `OKE_CLUSTER_ID`
- `RAG_APP_URL`
- `AIQ_APP_URL`

You need an NGC API key accepted for the local NIM models and charts, a Tavily
API key, and a new PostgreSQL password. Do not place secret values in shell
history, source control, screenshots, or chat.

## Task 1: Connect Cloud Shell to the assigned namespace

1. Set the values supplied by the workshop environment.

    ```bash
    export LAB_NAMESPACE="<your-assigned-namespace>"
    export OKE_CLUSTER_ID="<shared-oke-cluster-ocid>"
    export RAG_APP_URL="http://rag-<reservation-id>.130.162.244.74.nip.io"
    export AIQ_APP_URL="http://aiq-<reservation-id>.130.162.244.74.nip.io"
    ```

2. Generate the kubeconfig for the shared OKE cluster. `--with-auth-context`
   is required so the generated credential uses your LiveLabs OCI profile.

    ```bash
    oci ce cluster create-kubeconfig \
      --cluster-id "$OKE_CLUSTER_ID" \
      --file "$HOME/.kube/config" \
      --region "$OCI_REGION" \
      --token-version 2.0.0 \
      --kube-endpoint PUBLIC_ENDPOINT \
      --with-auth-context

    kubectl config set-context --current --namespace="$LAB_NAMESPACE"
    kubectl get pods
    ```

3. Verify the namespaced APIs used by the RAG chart. Each command must return
   `yes`.

    ```bash
    kubectl auth can-i create nimservices.apps.nvidia.com
    kubectl auth can-i create elasticsearches.elasticsearch.k8s.elastic.co
    kubectl auth can-i create networkpolicies.networking.k8s.io
    ```

## Task 2: Configure credentials securely

1. Enter your credentials using hidden prompts.

    ```bash
    read -rsp "NGC API key: " NGC_API_KEY; printf '\n'
    read -rsp "Tavily API key: " TAVILY_API_KEY; printf '\n'
    read -rsp "AI-Q PostgreSQL password: " AIQ_DB_PASSWORD; printf '\n'
    export NGC_API_KEY TAVILY_API_KEY AIQ_DB_PASSWORD
    ```

2. Create the registry and application secrets. These commands do not print
   secret values.

    ```bash
    kubectl create secret docker-registry ngc-secret \
      --docker-server=nvcr.io \
      --docker-username='$oauthtoken' \
      --docker-password="$NGC_API_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -

    kubectl create secret generic ngc-api \
      --from-literal=NGC_CLI_API_KEY="$NGC_API_KEY" \
      --from-literal=NGC_API_KEY="$NGC_API_KEY" \
      --from-literal=NVIDIA_API_KEY="$NGC_API_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -

    kubectl create secret generic aiq-credentials \
      --from-literal=DB_USER_NAME=aiq \
      --from-literal=DB_USER_PASSWORD="$AIQ_DB_PASSWORD" \
      --from-literal=NVIDIA_API_KEY="$NGC_API_KEY" \
      --from-literal=TAVILY_API_KEY="$TAVILY_API_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

## Task 3: Deploy the seven-GPU RAG profile

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
    kubectl get nimservice
    kubectl get pods -o wide
    kubectl get service rag-frontend
    ```

## Task 4: Open and validate RAG

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

    ![Create a collection in the RAG Playground](../../deploy-rag-blueprint/images/rag-new-collection-dialog.png)

    ![Upload a document to the RAG Playground](../../deploy-rag-blueprint/images/rag-upload-source-files.png)

    ![Review RAG answer sources and citations](../../deploy-rag-blueprint/images/rag-citations-panel.png)

## Task 5: Deploy AI-Q Research Assistant

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
    kubectl rollout restart deployment/aiq-backend
    kubectl rollout status deployment/aiq-backend --timeout=20m
    kubectl rollout status deployment/aiq-frontend --timeout=20m
    ```

## Task 6: Open and test AI-Q

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

    ![AI-Q Research Assistant interface](../../deploy-aiq-research-assistant/images/aiq-home-step-one.png)

    ![Select a source in AI-Q](../../deploy-aiq-research-assistant/images/aiq-source-selection.png)

## Task 7: Troubleshoot and clean up

- A missing NIM or Elasticsearch API is a platform issue. Do not install or
  modify cluster-wide operators from the attendee account.
- If a local model does not become ready, inspect namespace-scoped status and
  events:

    ```bash
    kubectl get nimservice
    kubectl describe nimservice
    kubectl get pods -o wide
    kubectl get events --sort-by=.metadata.creationTimestamp | tail -50
    ```

- If an application URL resolves but is temporarily unavailable, verify its
  frontend Service endpoints. Do not create a load balancer or change Istio.
- To remove both applications before your reservation ends:

    ```bash
    helm uninstall aiq210-frag
    helm uninstall rag
    ```

  PersistentVolumeClaims can retain data. Delete them only with instructor
  approval.

## Screenshot follow-up

The existing RAG and AI-Q images are reused as workflow illustrations. During
the full seven-GPU end-to-end test, capture replacement screenshots for:

1. the Resources tab with the assigned URLs and identifiers redacted;
2. the local-model deployment status, showing the four local NIM roles without
   exposing usernames, hostnames, or API values;
3. the fully rendered RAG and AI-Q applications through their `nip.io` HTTP
   URLs;
4. an AI-Q report backed by the local seven-GPU RAG profile.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** September 2026
