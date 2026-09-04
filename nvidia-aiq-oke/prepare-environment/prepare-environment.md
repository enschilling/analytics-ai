# Prepare Your Cloud Shell Environment

## Introduction

In this lab, you prepare OCI Cloud Shell for the RAG and AIRA deployment. The OKE cluster, GPU node, VCN, DRG, Service Gateway, and Squid proxy are already provisioned by the lab environment. Your job is to confirm that Cloud Shell can reach the cluster, set the variables used by the deployment commands, and capture the required runtime API keys without printing them.

The deployment uses Object Storage bundles and public OCIR image mirrors. Runtime calls to NVIDIA NGC, Tavily, and selected package endpoints go through the Squid proxy.

### Objectives

- Confirm Cloud Shell is in the correct region.
- Verify `kubectl` access to the pre-provisioned OKE cluster.
- Set namespace, bundle, and proxy variables.
- Capture NVIDIA NGC and Tavily keys safely.
- Confirm the GPU node is visible before deployment.

Estimated Time: 10 minutes

## Task 1: Confirm Cloud Shell and Kubernetes Access

1. Confirm the Cloud Shell region.

    The region shown in the Cloud Shell prompt should match the workshop region.

    ```bash
    oci iam region-subscription list --query 'data[]."region-name"' --output table
    ```

2. Confirm `kubectl` is configured.

    ```bash
    kubectl config current-context
    kubectl cluster-info
    ```

3. Confirm the pre-provisioned GPU node is visible.

    ```bash
    kubectl get nodes -o wide
    ```

4. Stop and ask the instructor for help if `kubectl get nodes` returns no nodes or an authentication error.

## Task 2: Set Deployment Variables

1. Set the region and namespaces.

    ```bash
    export OCI_REGION="ap-melbourne-1"
    export RAG_NAMESPACE="rag"
    export AIRA_NAMESPACE="aira"
    ```

2. Set the RAG and AIRA service URLs.

    The RAG URLs are internal Kubernetes service URLs. They are useful for AIRA configuration and in-cluster troubleshooting. The AIRA URL points to the local Cloud Shell port-forward you create later in the workshop.

    ```bash
    export RAG_LLM_URL="http://nim-llm.${RAG_NAMESPACE}.svc.cluster.local:8000"
    export RAG_SERVER_URL="http://rag-server.${RAG_NAMESPACE}.svc.cluster.local:8081"
    export RAG_INGESTOR_URL="http://ingestor-server.${RAG_NAMESPACE}.svc.cluster.local:8082"
    export RAG_URL="$RAG_SERVER_URL"

    export AIRA_SERVICE_URL="http://aira-nginx.${AIRA_NAMESPACE}.svc.cluster.local:8051"
    export AIRA_URL="http://127.0.0.1:8051"
    export AIRA_HEALTH_URL="${AIRA_URL}/health"
    ```

3. Set the Object Storage bundle URLs.

    These bundles contain rendered Kubernetes manifests. The manifests point to images already mirrored into public OCIR repositories.

    ```bash
    export RAG_BUNDLE_URL="https://objectstorage.ap-melbourne-1.oraclecloud.com/p/uROJzcKb7krH778RifnPhUa8zq4BA-jqyN7_1CAyKSEAmubFv-Q2hHIxlpbAQ4dd/n/yzrh1ull1ess/b/bucket-aiq/o/rag-ocir-cloudshell-bundle.tgz"
    export AIRA_BUNDLE_URL="https://objectstorage.ap-melbourne-1.oraclecloud.com/p/TC-jEXiGGqTEUiYMPNP8t_JykZHS_xKV1o1oFfusdBCn0ZosSj5oH7UUMNkf6d0A/n/yzrh1ull1ess/b/bucket-aiq/o/aira-ocir-cloudshell-bundle.tgz"
    ```

4. Set the Squid proxy values.

    The proxy handles selected runtime internet access. The `NO_PROXY` list keeps Kubernetes service traffic, pod traffic, metadata, VCN traffic, and Oracle service traffic off the proxy.

    ```bash
    export PROXY_URL="http://10.255.255.18:3128"
    export PROXY_NO_PROXY="localhost,127.0.0.1,169.254.169.254,10.96.0.0/16,10.244.0.0/16,20.85.99.0/24,.svc,.cluster.local,.oraclevcn.com"
    ```

5. Confirm the variables are set.

    ```bash
    env | egrep 'OCI_REGION|RAG_NAMESPACE|AIRA_NAMESPACE|RAG_URL|RAG_LLM_URL|RAG_SERVER_URL|RAG_INGESTOR_URL|AIRA_URL|AIRA_SERVICE_URL|AIRA_HEALTH_URL|RAG_BUNDLE_URL|AIRA_BUNDLE_URL|PROXY_URL|PROXY_NO_PROXY'
    ```

## Task 3: Set Runtime Credentials

1. Export the NVIDIA NGC API key into environment variables.

    Replace the placeholder with your real key in Cloud Shell. Do not commit API keys to source control or paste them into shared documents.

    ```bash
    export NVIDIA_API_KEY="<ngc_api_key>"
    export NGC_API_KEY="$NVIDIA_API_KEY"
    ```

2. Export the Tavily API key.

    ```bash
    export TAVILY_API_KEY="<tavily_api_key>"
    ```

3. Confirm only that the variables are populated.

    ```bash
    test -n "$NVIDIA_API_KEY" && echo "NVIDIA_API_KEY is set"
    test -n "$NGC_API_KEY" && echo "NGC_API_KEY is set"
    test -n "$TAVILY_API_KEY" && echo "TAVILY_API_KEY is set"
    ```

## Task 4: Confirm Basic Cluster Readiness

1. Confirm the node is Ready.

    ```bash
    kubectl get nodes
    kubectl describe nodes | egrep -i "Name:|Ready|Taints:|DiskPressure|nvidia.com/gpu|Allocatable|Capacity" -A10
    ```

2. Confirm the expected Kubernetes namespaces exist.

    ```bash
    kubectl get ns
    ```

3. Continue when the node is Ready and `DiskPressure=False`.

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 11, 2026
