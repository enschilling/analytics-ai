# Deploy NVIDIA RAG Blueprint and AIRA

## Introduction

In this lab, you deploy the NVIDIA RAG Blueprint and AIRA, also known as the AIQ Research Assistant, onto the OKE GPU cluster. The deployment uses offline-ready Kubernetes manifests from Object Storage and container images mirrored into public OCIR repositories. This pattern lets Cloud Shell and private worker nodes deploy the application without pulling container images directly from the public internet.

The application still needs runtime access to external APIs. NVIDIA NIM containers download model manifests and signed model files from NGC, and AIRA uses Tavily for search. You will route those runtime calls through the Squid proxy while keeping internal Kubernetes traffic, such as Milvus to etcd and MinIO, inside the cluster.

### Objectives

- Download the RAG and AIRA deployment bundles from Object Storage.
- Verify the rendered manifests point to OCIR images.
- Deploy the RAG Blueprint into the `rag` namespace.
- Add GPU taint tolerations for a single-node lab cluster.
- Configure proxy variables only on pods that require internet access.
- Validate NGC access through Squid before waiting for NIM startup.
- Deploy AIRA into the `aira` namespace.
- Confirm AIRA responds through an in-cluster port-forward.

Estimated Time: 35 minutes

## Task 1: Download and Inspect the Offline Bundles

The variables used in this lab were set in the Prepare Environment lab. Keep the same Cloud Shell session open, or re-run the export commands from that lab before continuing.

1. Create a local working directory in Cloud Shell.

    ```bash
    mkdir -p "$HOME/aiq-offline"
    cd "$HOME/aiq-offline"
    ```

2. Download the RAG and AIRA bundles.

    ```bash
    curl -L -o rag-ocir-cloudshell-bundle.tgz "$RAG_BUNDLE_URL"
    curl -L -o aira-ocir-cloudshell-bundle.tgz "$AIRA_BUNDLE_URL"

    ls -lh *.tgz
    ```

3. Extract the bundles.

    ```bash
    tar -xzf rag-ocir-cloudshell-bundle.tgz -C "$HOME"
    ```
    ```bash
    tar -xzf aira-ocir-cloudshell-bundle.tgz -C "$HOME"
    ```
    ```bash
    export RAG_MANIFEST="$HOME/rag-ocir-bundle/aiq-rendered-ocir.yaml"
    ```
    ```bash
    export AIRA_MANIFEST="$HOME/aira-ocir-bundle/aiq-rendered-ocir.yaml"
    ```

## Task 2: Deploy the RAG Blueprint

1. Create the RAG namespace.

    ```bash
    kubectl create namespace "$RAG_NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
    ```

2. Apply the rendered RAG manifest.

    ```bash
    kubectl apply -n "$RAG_NAMESPACE" -f "$RAG_MANIFEST"
    ```

3. Create the NGC runtime secret.

    The chart uses several key names across NIM containers. Store the same value under all expected names so each container can find it.

    ```bash
    kubectl create secret generic ngc-api \
      -n "$RAG_NAMESPACE" \
      --from-literal NGC_CLI_API_KEY="$NVIDIA_API_KEY" \
      --from-literal NGC_API_KEY="$NVIDIA_API_KEY" \
      --from-literal NVIDIA_API_KEY="$NVIDIA_API_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

4. Confirm the secret is not empty.

    This command prints a byte count, not the secret value. A zero byte count means the NIM pods will authenticate as an empty user and fail with NGC `401` errors.

    ```bash
    kubectl get secret ngc-api -n "$RAG_NAMESPACE" \
      -o jsonpath='{.data.NGC_API_KEY}' | base64 -d | wc -c
    ```

## Task 3: Patch Scheduling and Image Pull Settings

1. Add broad tolerations to RAG deployments and stateful sets.

    The lab uses a single GPU node. Some supporting pods, such as Redis, MinIO, Milvus, and test pods, may not request GPUs, but they still need to run on the GPU-tainted node.

    ```bash
    for DEPLOY in $(kubectl get deployment -n "$RAG_NAMESPACE" -o name); do
      kubectl patch "$DEPLOY" -n "$RAG_NAMESPACE" --type=merge \
        -p '{"spec":{"template":{"spec":{"tolerations":[{"operator":"Exists"}]}}}}'
    done
    ```
    ```bash 
    for STS in $(kubectl get statefulset -n "$RAG_NAMESPACE" -o name); do
      kubectl patch "$STS" -n "$RAG_NAMESPACE" --type=merge \
        -p '{"spec":{"template":{"spec":{"tolerations":[{"operator":"Exists"}]}}}}'
    done
    ```

2. Remove image pull secret references.

    The mirrored OCIR repositories for this lab are public, so learners do not need an OCIR username, auth token, or Kubernetes image pull secret.

    ```bash
    for DEPLOY in $(kubectl get deployment -n "$RAG_NAMESPACE" -o name); do
      kubectl patch "$DEPLOY" -n "$RAG_NAMESPACE" --type=merge \
        -p '{"spec":{"template":{"spec":{"imagePullSecrets":[]}}}}'
    done
    ```
    ```bash
    for STS in $(kubectl get statefulset -n "$RAG_NAMESPACE" -o name); do
      kubectl patch "$STS" -n "$RAG_NAMESPACE" --type=merge \
        -p '{"spec":{"template":{"spec":{"imagePullSecrets":[]}}}}'
    done
    ```

3. Set `Recreate` rollout strategy on GPU-heavy NIM deployments.

    A one-node GPU lab can run out of allocatable GPUs during rolling updates if old and new NIM pods overlap. `Recreate` avoids that temporary double booking.

    ```bash
    for DEPLOY in \
      rag-text-reranking-nim \
      rag-nvidia-nim-llama-32-nv-embedqa-1b-v2 \
      rag-nemoretriever-page-elements-v2
    do
      if kubectl get deployment "$DEPLOY" -n "$RAG_NAMESPACE" >/dev/null 2>&1; then
        kubectl patch deployment "$DEPLOY" -n "$RAG_NAMESPACE" --type=merge \
          -p '{"spec":{"strategy":{"type":"Recreate"}}}'
      fi
    done
    ```

## Task 4: Configure Proxy Access for Internet-Facing RAG Pods

1. Patch proxy variables only on RAG workloads that need internet access.

    Do not set proxy variables on internal infrastructure components such as Milvus, etcd, MinIO, or Redis. Those pods talk to Kubernetes service names like `rag-etcd` and `rag-minio`; forcing that traffic through Squid can break startup.

    ```bash
    for NAME_PATTERN in \
      'rag-nv-ingest' \
      'ingestor-server' \
      'rag-server' \
      'rag-text-reranking-nim' \
      'rag-nvidia-nim' \
      'rag-nemoretriever'
    do
      for DEPLOY in $(kubectl get deployment -n "$RAG_NAMESPACE" -o name | grep "$NAME_PATTERN" || true); do
        kubectl set env "$DEPLOY" -n "$RAG_NAMESPACE" \
          HTTP_PROXY="$PROXY_URL" \
          HTTPS_PROXY="$PROXY_URL" \
          http_proxy="$PROXY_URL" \
          https_proxy="$PROXY_URL" \
          NO_PROXY="$PROXY_NO_PROXY,rag-etcd,rag-minio,rag-redis-master,rag-redis-replicas,milvus" \
          no_proxy="$PROXY_NO_PROXY,rag-etcd,rag-minio,rag-redis-master,rag-redis-replicas,milvus"
      done
    done
    ```
    ```bash
    if kubectl get statefulset/rag-nim-llm -n "$RAG_NAMESPACE" >/dev/null 2>&1; then
      kubectl set env statefulset/rag-nim-llm -n "$RAG_NAMESPACE" \
        HTTP_PROXY="$PROXY_URL" \
        HTTPS_PROXY="$PROXY_URL" \
        http_proxy="$PROXY_URL" \
        https_proxy="$PROXY_URL" \
        NO_PROXY="$PROXY_NO_PROXY,rag-etcd,rag-minio,rag-redis-master,rag-redis-replicas,milvus" \
        no_proxy="$PROXY_NO_PROXY,rag-etcd,rag-minio,rag-redis-master,rag-redis-replicas,milvus"
    fi
    ```

2. Remove proxy variables from internal infrastructure components if they were added accidentally.

    This cleanup protects Milvus, etcd, MinIO, and Redis from using the Squid proxy for internal cluster calls.

    ```bash
    for NAME_PATTERN in 'milvus' 'etcd' 'minio' 'redis'; do
      for RESOURCE in $(kubectl get deployment,statefulset -n "$RAG_NAMESPACE" -o name | grep "$NAME_PATTERN" || true); do
        kubectl set env "$RESOURCE" -n "$RAG_NAMESPACE" \
          HTTP_PROXY- HTTPS_PROXY- http_proxy- https_proxy- \
          NO_PROXY- no_proxy- CONDA_HTTP_PROXY- CONDA_HTTPS_PROXY- PIP_PROXY-
      done
    done
    ```

3. Patch Conda and PIP proxy settings for `nv-ingest`.

    The exact deployment name can vary by bundle. Discover the deployment first, then patch it. This avoids failures such as `Failed to connect to conda.anaconda.org`.

    ```bash
    NV_INGEST_DEPLOY=$(kubectl get deployment -n "$RAG_NAMESPACE" -o name | grep 'nv-ingest' | head -1 || true)
    ```
    ```bash
    if [ -n "$NV_INGEST_DEPLOY" ]; then
      kubectl set env "$NV_INGEST_DEPLOY" -n "$RAG_NAMESPACE" \
        CONDA_HTTP_PROXY="$PROXY_URL" \
        CONDA_HTTPS_PROXY="$PROXY_URL" \
        PIP_PROXY="$PROXY_URL"
    else
      echo "No nv-ingest deployment found; skipping Conda proxy patch."
    fi
    ```

4. Attach the NGC secret to the NVIDIA/NIM workloads.

    Keep API keys scoped to the workloads that need them. Supporting infrastructure such as Redis and Milvus should not receive external API credentials.

    ```bash
    for DEPLOY in \
      rag-text-reranking-nim \
      rag-nvidia-nim-llama-32-nv-embedqa-1b-v2 \
      rag-nemoretriever-page-elements-v2
    do
      if kubectl get deployment "$DEPLOY" -n "$RAG_NAMESPACE" >/dev/null 2>&1; then
        kubectl set env deployment/"$DEPLOY" -n "$RAG_NAMESPACE" --from=secret/ngc-api
      fi
    done
    ```
    ```bash
    if kubectl get statefulset/rag-nim-llm -n "$RAG_NAMESPACE" >/dev/null 2>&1; then
      kubectl set env statefulset/rag-nim-llm -n "$RAG_NAMESPACE" --from=secret/ngc-api
    fi
    ```

## Task 5: Validate NGC Through the Proxy

1. Create a temporary NGC test pod.

    This test proves that the pod can reach NGC through Squid and that the NGC key is valid. A `200` response means the network path and credentials are working. A `401` response means the key is missing, invalid, or not entitled to the model. A timeout means the proxy path is still broken.

    ```bash
    kubectl delete pod ngc-auth-test -n "$RAG_NAMESPACE" --ignore-not-found
    ```
    ```bash
    kubectl apply -n "$RAG_NAMESPACE" -f - <<EOF
    apiVersion: v1
    kind: Pod
    metadata:
      name: ngc-auth-test
    spec:
      restartPolicy: Never
      tolerations:
      - operator: Exists
      containers:
      - name: test
        image: ap-melbourne-1.ocir.io/id9y6mi8tcky/oke-public-cloud-provider-oci:v1.32-1dc23aae3b5-103-csi
        env:
        - name: NGC_API_KEY
          valueFrom:
            secretKeyRef:
              name: ngc-api
              key: NGC_API_KEY
        - name: HTTPS_PROXY
          value: ${PROXY_URL}
        - name: HTTP_PROXY
          value: ${PROXY_URL}
        command:
        - sh
        - -lc
        - |
          curl -sS -o /tmp/ngc-response.json -w "HTTP_STATUS=%{http_code}\n" \
            --proxy "${PROXY_URL}" \
            -H "Authorization: Bearer \${NGC_API_KEY}" \
            --connect-timeout 20 \
            https://api.ngc.nvidia.com/v2/org/nim/team/nvidia/models/llama-3.2-nv-rerankqa-1b-v2/tokenizer-8192-3fe66485/files
          head -c 500 /tmp/ngc-response.json
          echo
    EOF
    ```

2. Read the test result.

    ```bash
    kubectl logs -n "$RAG_NAMESPACE" ngc-auth-test
    ```

3. Restart RAG workloads after patching their pod templates.

    ```bash
    kubectl rollout restart deployment -n "$RAG_NAMESPACE"
    kubectl rollout restart statefulset -n "$RAG_NAMESPACE"
    ```

4. Watch the RAG pods.

    First startup can take 20 to 60 minutes because NIM containers initialize large models.

    ```bash
    kubectl get pods -n "$RAG_NAMESPACE" -w
    ```

## Task 6: Validate RAG Services

1. Check workload and PVC status.

    ```bash
    kubectl get pods -n "$RAG_NAMESPACE" -o wide
    kubectl get deploy,sts,svc,pvc -n "$RAG_NAMESPACE" -o wide
    ```

2. Wait for the primary services.

    ```bash
    kubectl rollout status statefulset/rag-nim-llm -n "$RAG_NAMESPACE" --timeout=1200s
    ```
    ```bash 
    kubectl rollout status deployment/rag-server -n "$RAG_NAMESPACE" --timeout=1200s
    ```
    ```bash 
    kubectl rollout status deployment/ingestor-server -n "$RAG_NAMESPACE" --timeout=1200s
    ```

3. Delete Helm test pods if they remain pending.

    The `rag-nim-llm-test-*` pods are test helpers, not runtime services. On a one-node GPU lab, they can remain pending because the GPU is already allocated to the LLM.

    ```bash
    kubectl delete pod -n "$RAG_NAMESPACE" \
      rag-nim-llm-test-completions \
      rag-nim-llm-test-models \
      rag-nim-llm-test-streaming-chat \
      --ignore-not-found
    ```

4. Confirm the RAG API URLs from inside the cluster.

    Cloud Shell cannot call `ClusterIP` services directly, so run the test from a temporary pod. The root path can return `404 Not Found`; that still proves the server answered, but `/docs` or `/openapi.json` is a clearer API validation.

    ```bash
    export RAG_NAMESPACE="${RAG_NAMESPACE:-rag}"
    export RAG_URL="${RAG_URL:-http://rag-server.${RAG_NAMESPACE}.svc.cluster.local:8081}"
    echo "$RAG_URL"

    kubectl delete pod -n "$RAG_NAMESPACE" rag-url-test rag-url-test-health --ignore-not-found

    kubectl run rag-url-test-health -n "$RAG_NAMESPACE" \
      --restart=Never \
      --image=ap-melbourne-1.ocir.io/id9y6mi8tcky/oke-public-cloud-provider-oci:v1.32-1dc23aae3b5-103-csi \
      --env RAG_URL="$RAG_URL" \
      --overrides='{"spec":{"tolerations":[{"operator":"Exists"}]}}' \
      -- sh -lc 'for p in /health /docs /openapi.json; do echo "== $p =="; curl -i --max-time 20 "$RAG_URL$p" || true; done'

    kubectl logs -n "$RAG_NAMESPACE" rag-url-test-health
    ```

## Task 7: Deploy AIRA

1. Create the AIRA namespace.

    ```bash
    kubectl create namespace "$AIRA_NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
    ```

2. Apply the rendered AIRA manifest.

    ```bash
    test -s "$AIRA_MANIFEST" || { echo "Missing AIRA manifest: $AIRA_MANIFEST"; find "$HOME" -maxdepth 3 -name aiq-rendered-ocir.yaml; exit 1; }
    ```
    ```bash 
    kubectl apply -n "$AIRA_NAMESPACE" -f "$AIRA_MANIFEST"
    ```

3. Set the Tavily key on the backend deployment.

    ```bash
    kubectl set env deployment/aira-aira-backend \
      -n "$AIRA_NAMESPACE" \
      "TAVILY_API_KEY=$TAVILY_API_KEY"
    ```

4. Patch proxy environment variables on AIRA deployments.

    AIRA calls external APIs at runtime, so it must also use Squid.

    ```bash
    for DEPLOY in $(kubectl get deployment -n "$AIRA_NAMESPACE" -o name); do
      kubectl set env "$DEPLOY" -n "$AIRA_NAMESPACE" \
        HTTP_PROXY="$PROXY_URL" \
        HTTPS_PROXY="$PROXY_URL" \
        http_proxy="$PROXY_URL" \
        https_proxy="$PROXY_URL" \
        NO_PROXY="$PROXY_NO_PROXY" \
        no_proxy="$PROXY_NO_PROXY"
    done
    ```

5. Patch AIRA tolerations and remove image pull secret references.

    ```bash
    for DEPLOY in $(kubectl get deployment -n "$AIRA_NAMESPACE" -o name); do
      kubectl patch "$DEPLOY" -n "$AIRA_NAMESPACE" --type=merge \
        -p '{"spec":{"template":{"spec":{"tolerations":[{"operator":"Exists"}],"imagePullSecrets":[]}}}}'
    done
    ```

6. Restart and watch AIRA.

    ```bash
    kubectl rollout restart deployment -n "$AIRA_NAMESPACE"
    ```
    ```bash
    kubectl get pods -n "$AIRA_NAMESPACE" -w
    ```

## Task 8: Test AIRA

1. List AIRA services.

    ```bash
    kubectl get svc -n "$AIRA_NAMESPACE" -o wide
    ```

2. Port-forward the AIRA nginx service.

    If Cloud Shell cannot open a second tab, run the port-forward in the background.

    ```bash
    export AIRA_NAMESPACE="${AIRA_NAMESPACE:-aira}"
    export AIRA_URL="${AIRA_URL:-http://127.0.0.1:8051}"
    export AIRA_HEALTH_URL="${AIRA_HEALTH_URL:-${AIRA_URL}/health}"

    kubectl port-forward -n "$AIRA_NAMESPACE" svc/aira-nginx 8051:8051 > /tmp/aira-port-forward.log 2>&1 &
    ```

3. Test the health endpoint.

    A `200 OK` response from `/health` proves that the AIRA nginx service is reachable.

    ```bash
    echo "$AIRA_HEALTH_URL"
    sleep 2; curl -i --max-time 20 "$AIRA_HEALTH_URL"
    ```

    If the curl command times out, inspect the port-forward log.

    ```bash
    cat /tmp/aira-port-forward.log
    ```

4. Stop the port-forward when you finish testing.

    ```bash
    pkill -f "kubectl port-forward -n $AIRA_NAMESPACE svc/aira-nginx" || true
    ```

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 11, 2026
