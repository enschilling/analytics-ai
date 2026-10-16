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
   They contain the assigned namespace, cluster OCID, region, kubeconfig
   download command, and both HTTP application URLs. If only individual
   values are displayed, set them below using the supplied values.

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

2. If you used the setup commands, the kubeconfig is already downloaded.
   Otherwise, run the following commands. Access is scoped by your IAM and
   Kubernetes permissions; the kubeconfig selects the shared cluster.
   `--with-auth-context` preserves the OCI authentication settings used when
   downloading it.

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

3. Verify the chart-required namespaced APIs. All authorization checks must
   return `yes`.

    ```bash
    kubectl auth can-i create nimservices.apps.nvidia.com
    ```
    ```bash
    kubectl auth can-i create nimcaches.apps.nvidia.com
    ```
    ```bash
    kubectl auth can-i create elasticsearches.elasticsearch.k8s.elastic.co
    ```
    ```bash
    kubectl auth can-i create networkpolicies.networking.k8s.io
    ```
    ```bash
    kubectl get resourcequota
    ```

## Task 2: Configure credentials securely

1. Use hidden prompts so secret values are not echoed.

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
    AIQ_DB_PASSWORD="$(openssl rand -hex 32)"
    ```

    Check that the generated password contains exactly 64 lowercase
    hexadecimal characters without printing the password:

    ```bash
    if [[ "$AIQ_DB_PASSWORD" =~ ^[0-9a-f]{64}$ ]]; then
        echo "Password format OK"
    else
        echo "Password generation failed"
    fi
    ```

    Expected output:

    ```text
    Password format OK
    ```

    If the check prints `Password generation failed`, repeat the generation
    command and validation check above.

    After the check succeeds, export the variables before creating the Secret.

    ```bash
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

Continue to [Lab 2: Deploy and Test RAG](?lab=deploy-rag-blueprint-sandbox-lite), or select **Lab 2: Deploy and Test RAG** in the workshop navigation menu.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
