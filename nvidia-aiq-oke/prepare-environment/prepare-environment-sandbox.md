# Lab 1: Prepare the Environment

## Introduction

Connect OCI Cloud Shell to your assigned namespace and prepare application credentials without exposing secrets.

### Objectives

- Download kubeconfig and verify namespaced access.
- Create the AI-Q credential Secret; let RAG Helm own its Secrets.

Estimated Time: 10 minutes.

## Prerequisites

Complete Get Started and obtain the Resources values and API keys listed in the Introduction.

> **Note for Mac users:** Commands must use straight double quotes (`"`), not curly or smart quotes (`“` and `”`), which can cause errors in the console. When editing commands before pasting them into OCI Cloud Shell, use a plain-text editor such as Notepad or TextPad, or TextEdit on macOS in plain-text mode with smart quotes disabled. Check that double quotes remain straight before running the command.

### Open OCI Cloud Shell

1. Sign in to the **OCI Console** using the login information described in
    the Introduction. For a LiveLabs reservation, use **View Login Info** and
    **Launch OCI** to open the console with your assigned credentials.

2. In the OCI Console header, select the region assigned to your workshop.

3. Click the **Cloud Shell/Code Editor** icon in the console header, then
    select **Cloud Shell** from the menu.

4. Wait for the terminal panel to open and display a command prompt. Enter
    the commands in this lab in that panel. Keep the same Cloud Shell session
    open as you work through the workshop.

For additional details, see Oracle's [Using Cloud Shell](https://docs.oracle.com/en-us/iaas/Content/API/Concepts/devcloudshellgettingstarted.htm).

## Task 1: Connect Cloud Shell to the assigned namespace

1. Copy the **Cloud Shell setup commands** from the LiveLabs Resources tab.
   If only individual values are shown, set the values supplied by the
   workshop environment below.

    ```bash
    export LAB_NAMESPACE="<your-assigned-namespace>"
    ```
    ```bash
    export OKE_CLUSTER_ID="<shared-oke-cluster-ocid>"
    ```
    ```bash
    export OCI_REGION="<assigned-oke-region>"
    ```
    ```bash
    export RAG_APP_URL="<RAG-application-URL-from-Resources>"
    ```
    ```bash
    export AIQ_APP_URL="<AI-Q-application-URL-from-Resources>"
    ```

2. If you copied the setup commands, kubeconfig is already configured.
   Otherwise, download it below. `--with-auth-context` preserves your OCI
   authentication settings; namespace access is enforced by IAM and RBAC.

    ```bash
    mkdir -p "$HOME/.kube"
    ```
    ```bash
    export KUBECONFIG="$HOME/.kube/${LAB_NAMESPACE}.yaml"
    ```

    ```bash
    oci ce cluster create-kubeconfig \
      --cluster-id "$OKE_CLUSTER_ID" \
      --file "$KUBECONFIG" \
      --region "$OCI_REGION" \
      --token-version 2.0.0 \
      --kube-endpoint PUBLIC_ENDPOINT \
      --with-auth-context
    ```
    ```bash
    kubectl config set-context --current --namespace="$LAB_NAMESPACE"
    ```

    Expected output (your context name will differ):

    ```text
    Context "context-NVIDIA-c6hijvwguia" modified.
    ```

    Check for pods in your assigned namespace:

    ```bash
    kubectl get pods
    ```

    Expected output for a new namespace before deploying the applications:

    ```text
    No resources found in aiq-241135 namespace.
    ```

    Your namespace name will match `$LAB_NAMESPACE`. This message is expected
    when no pods have been deployed yet; you can continue to the next step.

3. Verify the namespaced APIs used by the RAG chart. Each command must return
   `yes`.

    ```bash
    kubectl auth can-i create nimservices.apps.nvidia.com
    ```
    ```bash
    kubectl auth can-i create elasticsearches.elasticsearch.k8s.elastic.co
    ```
    ```bash
    kubectl auth can-i create networkpolicies.networking.k8s.io
    ```

## Task 2: Configure credentials securely

1. Enter your credentials using hidden prompts.

    ```bash
    read -rsp "NGC API key: " NGC_API_KEY; printf '\n'
    ```
    ```bash
    read -rsp "NVIDIA API key (press Enter to reuse NGC key): " NVIDIA_API_KEY; printf '\n'
    ```
    ```bash
    NVIDIA_API_KEY="${NVIDIA_API_KEY:-$NGC_API_KEY}"
    ```

    ```bash
    read -rsp "Tavily API key: " TAVILY_API_KEY; printf '\n'
    ```
    ```bash
    AIQ_DB_PASSWORD="$(openssl rand -base64 32)"
    ```
    ```bash
    export NGC_API_KEY NVIDIA_API_KEY TAVILY_API_KEY AIQ_DB_PASSWORD
    ```

2. Create only the AI-Q application Secret. RAG Helm creates and owns
   `ngc-secret` and `ngc-api` using the values passed in Lab 2, Task 1. Do not
   pre-create those names; doing so causes Helm ownership errors.

    ```bash
    kubectl create secret generic aiq-credentials \
      --from-literal=DB_USER_NAME=aiq \
      --from-literal=DB_USER_PASSWORD="$AIQ_DB_PASSWORD" \
      --from-literal=NVIDIA_API_KEY="$NGC_API_KEY" \
      --from-literal=TAVILY_API_KEY="$TAVILY_API_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

## Conclusion

Cloud Shell uses your assigned namespace and the AI-Q credential Secret is present.

Continue to [Lab 2: Deploy and Test RAG](?lab=deploy-rag-blueprint-sandbox), or select **Lab 2: Deploy and Test RAG** in the workshop navigation menu.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
