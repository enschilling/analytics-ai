# Lab 1: Prepare the Environment

## Introduction

Connect OCI Cloud Shell to your assigned namespace and prepare application credentials without exposing secrets.

### Objectives

- Download kubeconfig and verify namespaced access.
- Create the AI-Q credential Secret; let RAG Helm own its Secrets.

Estimated Time: 10 minutes.

## Prerequisites

Complete Get Started and obtain the Resources values and API keys listed in the Introduction.

## Task 1: Connect Cloud Shell to the assigned namespace

1. Copy the **Cloud Shell setup commands** from the LiveLabs Resources tab.
   They contain the assigned namespace, cluster OCID, region, kubeconfig
   download command, and both HTTP application URLs. If only individual
   values are displayed, set them below using the supplied values.

    ```bash
    export LAB_NAMESPACE="<your-assigned-namespace>"
    export OKE_CLUSTER_ID="<shared-oke-cluster-ocid>"
    export OCI_REGION="<assigned-oke-region>"
    export RAG_APP_URL="<RAG-application-URL-from-Resources>"
    export AIQ_APP_URL="<AI-Q-application-URL-from-Resources>"
    ```

2. If you used the setup commands, the kubeconfig is already downloaded.
   Otherwise, run the following commands. Access is scoped by your IAM and
   Kubernetes permissions; the kubeconfig selects the shared cluster.
   `--with-auth-context` preserves the OCI authentication settings used when
   downloading it.

    ```bash
    mkdir -p "$HOME/.kube"
    export KUBECONFIG="$HOME/.kube/${LAB_NAMESPACE}.yaml"
    oci ce cluster create-kubeconfig \
      --cluster-id "$OKE_CLUSTER_ID" \
      --file "$KUBECONFIG" \
      --region "$OCI_REGION" \
      --token-version 2.0.0 \
      --kube-endpoint PUBLIC_ENDPOINT \
      --with-auth-context

    kubectl config set-context --current --namespace="$LAB_NAMESPACE"
    kubectl get pods
    ```

3. Verify the chart-required namespaced APIs. All authorization checks must
   return `yes`.

    ```bash
    kubectl auth can-i create nimservices.apps.nvidia.com
    kubectl auth can-i create nimcaches.apps.nvidia.com
    kubectl auth can-i create elasticsearches.elasticsearch.k8s.elastic.co
    kubectl auth can-i create networkpolicies.networking.k8s.io
    kubectl get resourcequota
    ```

## Task 2: Configure credentials securely

1. Use hidden prompts so secret values are not echoed.

    ```bash
    read -rsp "NGC API key: " NGC_API_KEY; printf '\n'
    read -rsp "NVIDIA API key (press Enter to reuse NGC key): " NVIDIA_API_KEY; printf '\n'
    NVIDIA_API_KEY="${NVIDIA_API_KEY:-$NGC_API_KEY}"
    read -rsp "Tavily API key: " TAVILY_API_KEY; printf '\n'
    AIQ_DB_PASSWORD="$(openssl rand -base64 32)"
    export NGC_API_KEY NVIDIA_API_KEY TAVILY_API_KEY AIQ_DB_PASSWORD
    ```

2. Create only the AI-Q application Secret. RAG Helm creates and owns
   `ngc-secret` and `ngc-api` in Lab 2, Task 1 from the supplied chart values.
   Pre-creating those names causes a Helm release-ownership error.
   The generated database password stays in your shell and the Secret.

    ```bash
    kubectl create secret generic aiq-credentials \
      --from-literal=DB_USER_NAME=aiq \
      --from-literal=DB_USER_PASSWORD="$AIQ_DB_PASSWORD" \
      --from-literal=NVIDIA_API_KEY="$NVIDIA_API_KEY" \
      --from-literal=TAVILY_API_KEY="$TAVILY_API_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

## Conclusion

Cloud Shell uses your assigned namespace and the AI-Q credential Secret is present.

Continue to [Lab 2: Deploy and Test RAG](../deploy-rag-blueprint/deploy-rag-blueprint-sandbox-lite.md).

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
