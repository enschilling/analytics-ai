# Validate and Clean Up

## Introduction

In this lab, you validate RAG and AIRA from inside the cluster. Then you clean up the Kubernetes application resources you created.

Cleanup order matters. Delete application namespaces and any load balancer services, but do not delete the shared OKE cluster, GPU node pool, VCN, DRG, firewall, or proxy.

### Objectives

- Run RAG and AIRA health checks.
- Test AIRA through a Cloud Shell port-forward.
- Review common troubleshooting signals.
- Delete RAG and AIRA application resources safely.

Estimated Time: 15 minutes

## Task 1: Validate RAG Runtime Components

1. Confirm the RAG and AIRA URL variables.

    Re-run these exports if you opened a new Cloud Shell session.

    ```bash
    export RAG_NAMESPACE="${RAG_NAMESPACE:-rag}"
    export AIRA_NAMESPACE="${AIRA_NAMESPACE:-aira}"

    export RAG_LLM_URL="http://nim-llm.${RAG_NAMESPACE}.svc.cluster.local:8000"
    export RAG_SERVER_URL="http://rag-server.${RAG_NAMESPACE}.svc.cluster.local:8081"
    export RAG_INGESTOR_URL="http://ingestor-server.${RAG_NAMESPACE}.svc.cluster.local:8082"
    export RAG_URL="$RAG_SERVER_URL"

    export AIRA_SERVICE_URL="http://aira-nginx.${AIRA_NAMESPACE}.svc.cluster.local:8051"
    export AIRA_URL="http://127.0.0.1:8051"
    export AIRA_HEALTH_URL="${AIRA_URL}/health"
    ```

2. Confirm the RAG pods, services, and PVCs.

    ```bash
    kubectl get pods -n "$RAG_NAMESPACE" -o wide
    kubectl get deploy,sts,svc,pvc -n "$RAG_NAMESPACE" -o wide
    ```

3. Confirm the LLM, RAG server, and ingestor rollouts.

    ```bash
    kubectl rollout status statefulset/rag-nim-llm -n "$RAG_NAMESPACE" --timeout=1200s
    kubectl rollout status deployment/rag-server -n "$RAG_NAMESPACE" --timeout=1200s
    kubectl rollout status deployment/ingestor-server -n "$RAG_NAMESPACE" --timeout=1200s
    ```

4. Confirm Milvus can reach etcd.

    Milvus depends on etcd for metadata. A healthy etcd response helps distinguish an internal service issue from a Milvus startup issue.

    ```bash
    kubectl delete pod etcd-test -n "$RAG_NAMESPACE" --ignore-not-found

    kubectl run etcd-test -n "$RAG_NAMESPACE" \
      --restart=Never \
      --image=ap-melbourne-1.ocir.io/id9y6mi8tcky/oke-public-cloud-provider-oci:v1.32-1dc23aae3b5-103-csi \
      --overrides='{"spec":{"tolerations":[{"operator":"Exists"}]}}' \
      -- sh -lc 'curl -sv --max-time 10 http://rag-etcd:2379/health'

    kubectl logs -n "$RAG_NAMESPACE" etcd-test
    ```

5. Expected etcd output:

    ```text
    HTTP/1.1 200 OK
    ```

6. Confirm the RAG API URLs from inside the cluster.

    The root path can return `404 Not Found`; that still proves the server answered, but `/docs` or `/openapi.json` is a clearer API validation.

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

7. Delete pending Helm test pods if they are not needed.

    These pods are chart test helpers. They are not part of the live application.

    ```bash
    kubectl delete pod -n "$RAG_NAMESPACE" \
      rag-nim-llm-test-completions \
      rag-nim-llm-test-models \
      rag-nim-llm-test-streaming-chat \
      --ignore-not-found
    ```

## Task 2: Validate AIRA

1. Check AIRA pods and services.

    ```bash
    kubectl get pods -n "$AIRA_NAMESPACE" -o wide
    kubectl get deploy,svc -n "$AIRA_NAMESPACE" -o wide
    ```

2. Wait for AIRA rollouts.

    ```bash
    kubectl rollout status deployment/aira-aira-backend -n "$AIRA_NAMESPACE" --timeout=600s
    kubectl rollout status deployment/aira-aira-frontend -n "$AIRA_NAMESPACE" --timeout=600s
    kubectl rollout status deployment/aira-nginx -n "$AIRA_NAMESPACE" --timeout=600s
    kubectl rollout status deployment/aira-phoenix -n "$AIRA_NAMESPACE" --timeout=600s
    ```

3. Port-forward AIRA nginx from Cloud Shell.

    The AIRA nginx service listens on port `8051` in this bundle. Running the port-forward in the background lets you curl from the same Cloud Shell window.

    ```bash
    export AIRA_NAMESPACE="${AIRA_NAMESPACE:-aira}"
    export AIRA_URL="${AIRA_URL:-http://127.0.0.1:8051}"
    export AIRA_HEALTH_URL="${AIRA_HEALTH_URL:-${AIRA_URL}/health}"

    kubectl port-forward -n "$AIRA_NAMESPACE" svc/aira-nginx 8051:8051 > /tmp/aira-port-forward.log 2>&1 &
    ```

4. Test the health endpoint.

    ```bash
    echo "$AIRA_HEALTH_URL"
    sleep 2; curl -i --max-time 20 "$AIRA_HEALTH_URL"
    ```

5. A healthy AIRA nginx service returns `HTTP/1.1 200 OK`.

    If the curl command times out, inspect the port-forward log.

    ```bash
    cat /tmp/aira-port-forward.log
    ```

6. Stop the port-forward.

    ```bash
    pkill -f "kubectl port-forward -n $AIRA_NAMESPACE svc/aira-nginx" || true
    ```

## Task 3: Troubleshoot Common Issues

1. If CoreDNS is Pending, patch the GPU toleration as shown in Lab 4.

    ```bash
    kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
    kubectl describe node "$NODE_NAME" | grep -i taints
    ```

2. If PVCs stay Pending, check the OCI CSI node plugin and IAAS API reachability.

    ```bash
    kubectl get pods -n kube-system -o wide | egrep 'csi-oci-node|csi'
    kubectl get pvc -n "$RAG_NAMESPACE" -o wide
    ```

3. If NIM pods fail with NGC `builder error`, verify the runtime secret and proxy variables.

    ```bash
    kubectl get secret ngc-api -n "$RAG_NAMESPACE" -o jsonpath='{.data.NGC_API_KEY}' | base64 -d | wc -c
    kubectl set env statefulset/rag-nim-llm -n "$RAG_NAMESPACE" --list | egrep 'HTTP_PROXY|HTTPS_PROXY|NO_PROXY|NGC|NVIDIA'
    ```

4. If Milvus restarts while etcd is healthy, confirm proxy variables were not added to Milvus.

    Milvus should call `rag-etcd` and `rag-minio` directly inside Kubernetes. It should not use Squid for those internal services.

    ```bash
    MILVUS_DEPLOY=$(kubectl get deployment -n "$RAG_NAMESPACE" -o name | grep milvus | head -1)
    kubectl set env "$MILVUS_DEPLOY" -n "$RAG_NAMESPACE" --list | egrep -i 'proxy|etcd|minio' || true
    ```

5. If the node reports disk pressure, verify host root growth before pulling more images.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -i "DiskPressure"
    kubectl describe node "$NODE_NAME" | grep -A8 -i "ephemeral-storage"
    ```

## Task 4: Clean Up Kubernetes Resources

1. Delete any load balancer services first.

    ```bash
    kubectl get svc -A | grep LoadBalancer || true
    ```

2. Delete the AIRA namespace.

    ```bash
    kubectl delete namespace "$AIRA_NAMESPACE" --ignore-not-found=true
    ```

3. Delete the RAG namespace.

    ```bash
    kubectl delete namespace "$RAG_NAMESPACE" --ignore-not-found=true
    ```

4. Wait until both namespaces are gone.

    ```bash
    kubectl get namespace "$AIRA_NAMESPACE" "$RAG_NAMESPACE"
    ```

5. If namespaces are stuck terminating, look for remaining PVCs, finalizers, or load balancer services.

    ```bash
    kubectl get pvc -A
    kubectl get svc -A | grep LoadBalancer || true
    ```

The OKE cluster, GPU node pool, VCN, DRG, firewall, and proxy are shared lab infrastructure. Do not delete them as part of the learner cleanup.

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 11, 2026
