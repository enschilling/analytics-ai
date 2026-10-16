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
    <copy>
    export LAB_NAMESPACE="<your-assigned-namespace>"
    export OKE_CLUSTER_ID="<shared-oke-cluster-ocid>"
    export OCI_REGION="<assigned-oke-region>"
    export RAG_APP_URL="<RAG-application-URL-from-Resources>"
    export AIQ_APP_URL="<AI-Q-application-URL-from-Resources>"
    </copy>
    ```

2. If you used the setup commands, the kubeconfig is already downloaded.
   Otherwise, run the following commands. Access is scoped by your IAM and
   Kubernetes permissions; the kubeconfig selects the shared cluster.
   `--with-auth-context` preserves the OCI authentication settings used when
   downloading it.

    ```bash
    <copy>
    mkdir -p "$HOME/.kube"
    </copy>
    ```
    ```bash
    <copy>
    export KUBECONFIG="$HOME/.kube/${LAB_NAMESPACE}.yaml"
    </copy>
    ```
    ```bash
    <copy>
    oci ce cluster create-kubeconfig \
      --cluster-id "$OKE_CLUSTER_ID" \
      --file "$KUBECONFIG" \
      --region "$OCI_REGION" \
      --token-version 2.0.0 \
      --kube-endpoint PUBLIC_ENDPOINT \
      --with-auth-context
    </copy>
    ```
    ```bash
    <copy>
    kubectl config set-context --current --namespace="$LAB_NAMESPACE"
    </copy>
    ```

    Expected output (your context name will differ):

    ```text
    <copy>
    Context "context-NVIDIA-c6hijvwguia" modified.
    </copy>
    ```

    Check for pods in your assigned namespace:

    ```bash
    <copy>
    kubectl get pods
    </copy>
    ```

    Expected output for a new namespace before deploying the applications:

    ```text
    <copy>
    No resources found in aiq-241135 namespace.
    </copy>
    ```

    Your namespace name will match `$LAB_NAMESPACE`. This message is expected
    when no pods have been deployed yet; you can continue to the next step.

3. Verify the chart-required namespaced APIs. All authorization checks must
   return `yes`.

    ```bash
    <copy>
    kubectl auth can-i create nimservices.apps.nvidia.com
    </copy>
    ```
    ```bash
    <copy>
    kubectl auth can-i create nimcaches.apps.nvidia.com
    </copy>
    ```
    ```bash
    <copy>
    kubectl auth can-i create elasticsearches.elasticsearch.k8s.elastic.co
    </copy>
    ```
    ```bash
    <copy>
    kubectl auth can-i create networkpolicies.networking.k8s.io
    </copy>
    ```
    ```bash
    <copy>
    kubectl get resourcequota
    </copy>
    ```

## Task 2: Configure credentials securely

1. Use hidden prompts so secret values are not echoed.

    ```bash
    <copy>
    read -rsp "NGC API key: " NGC_API_KEY; printf '\n'
    </copy>
    ```
    ```bash
    <copy>
    read -rsp "NVIDIA API key (press Enter to reuse NGC key): " NVIDIA_API_KEY; printf '\n'
    </copy>
    ```
    ```bash
    <copy>
    NVIDIA_API_KEY="${NVIDIA_API_KEY:-$NGC_API_KEY}"
    </copy>
    ```

    ```bash
    <copy>
    read -rsp "Tavily API key: " TAVILY_API_KEY; printf '\n'
    </copy>
    ```
    ```bash
    <copy>
    AIQ_DB_PASSWORD="$(openssl rand -hex 32)"
    </copy>
    ```
    ```bash
    <copy>
    export NGC_API_KEY NVIDIA_API_KEY TAVILY_API_KEY AIQ_DB_PASSWORD
    </copy>
    ```

    The database password uses 32 random bytes represented as hexadecimal.
    This keeps it secure and URL-safe: the AI-Q chart inserts the password into
    database connection URLs without escaping it. Do not substitute a Base64
    password, which can contain URL delimiters such as `/`.

2. Create only the AI-Q application Secret. RAG Helm creates and owns
   `ngc-secret` and `ngc-api` in Lab 2, Task 1 from the supplied chart values.
   Pre-creating those names causes a Helm release-ownership error.
   The generated database password stays in your shell and the Secret.
    Run this in the same Cloud Shell session. The checks prevent creating an
    empty password or overwriting API keys with empty shell variables. If the
    checks fail, return to step 1; do not enable empty database passwords.

    ```bash
    <copy>
    if [[ -n "$AIQ_DB_PASSWORD" && -n "$NVIDIA_API_KEY" && -n "$TAVILY_API_KEY" ]]; then
    kubectl create secret generic aiq-credentials \
      --from-literal=DB_USER_NAME=aiq \
      --from-literal=DB_USER_PASSWORD="$AIQ_DB_PASSWORD" \
      --from-literal=NVIDIA_API_KEY="$NVIDIA_API_KEY" \
      --from-literal=TAVILY_API_KEY="$TAVILY_API_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -
    else
      printf '%s\n' 'Missing credentials: complete Task 2, step 1 in this shell before continuing.' >&2
    fi
    </copy>
    ```

## Conclusion

Cloud Shell uses your assigned namespace and the AI-Q credential Secret is present.

Continue to [Lab 2: Deploy and Test RAG](?lab=deploy-rag-blueprint-sandbox-lite), or select **Lab 2: Deploy and Test RAG** in the workshop navigation menu.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
