# Deploy NVIDIA RAG and AI-Q on OKE: 7-GPU Workshop

## Introduction

In this workshop, you deploy the NVIDIA RAG Blueprint and NVIDIA AI-Q Research
Assistant in the namespace assigned to your LiveLabs reservation. Unlike the
two-GPU sandbox-lite workshop, this profile runs the generation, embedding, reranking,
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
| Nemotron Super 49B generation NIM | 4 | RAG answer generation |
| Embedding NIM | 1 | Document and query embeddings |
| Reranking NIM | 1 | Retrieval quality |
| Page-elements NIM | 1 | PDF text extraction |
| CPU services | 0 | RAG services, Milvus, AI-Q, PostgreSQL, and Phoenix |
| **Total** | **7** | One GPU remains spare on an eight-A100 node |

The reservation must have a seven-GPU quota and adequate CPU, memory, storage,
and PVC quota before the workshop begins. This profile cannot be used on the
two-GPU sandbox-lite reservation. The reservation owner must explicitly set
`gpu_quota=7` when provisioning this variant; Terraform now defaults to two.

This guide retains the RAG 2.2.0 local-model profile. A full seven-GPU test
is still required; the September validation covered the two-GPU variant's
deployment and portal rendering.

## Get your reservation login information

1. In the LiveLabs workshop page, click **View Login Info** in the upper-left
   corner, above the workshop title. This opens **Reservation Information**.

2. Expand **Login Credentials** if it is collapsed. Click **Copy** next to
   **Username**, then use the **Copy** button next to **Password** when you
   need to sign in. Click **View** to reveal the password if needed.

3. Under **Tenancy Information**, note the **Tenancy Name** and **Region**
   assigned to your reservation. Use these reservation values throughout the
   workshop.

4. Click **Launch OCI** to open the OCI Console. Alternatively, click
   **Copy OCI Link** and paste that link into your browser. Sign in with the
   username and password from **Login Credentials**.

5. After signing in, select the reservation's region in the OCI Console and
   open **Cloud Shell**. Continue with **Get Started** and Lab 1 to configure
   access to your assigned Kubernetes namespace.

You can return to **View Login Info** whenever you need the credentials or OCI
link. **Reservation Time Remaining** shows how much lab time remains. Use the
credentials from your own reservation; example values shown in screenshots
will not sign you in.

## Prerequisites

The LiveLabs Resources tab or startup output supplies:

- `LAB_NAMESPACE`
- `OKE_CLUSTER_ID`
- `RAG_APP_URL`
- `AIQ_APP_URL`

You need an NGC API key accepted for the local NIM models and charts, a Tavily
API key. Lab 1, Task 2 generates a password for the PostgreSQL database installed
with AI-Q. Do not place secret values in shell
history, source control, screenshots, or chat.

Before starting Lab 1, obtain the required API keys:

- Follow the [NVIDIA API key instructions (PDF)](https://dam-cdn.nvd.orangelogic.com/AssetLink/1ib702mh10i74mgxnl3j80kw23q40158.pdf)
  to obtain your NVIDIA API key.
- Visit [Tavily](https://www.tavily.com/), sign up or sign in, and obtain your
  API key from your account dashboard.

## Workshop sequence

Complete **Get Started**, then work through the following labs in order:

1. Prepare the environment and credentials.
2. Deploy RAG, upload a document, and ask a question.
3. Deploy AI-Q and test research with web and document sources.
4. Troubleshoot issues and clean up workshop application resources.

Keep the same Cloud Shell session and namespace throughout. If Cloud Shell
reconnects with a fresh session, restore your namespace and kubeconfig as shown
in Lab 1 before continuing.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
