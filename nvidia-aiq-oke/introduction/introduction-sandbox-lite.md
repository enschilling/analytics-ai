# Deploy NVIDIA RAG and AI-Q on OKE: 2-GPU Workshop

## Introduction

In this workshop, you deploy NVIDIA RAG Blueprint 2.6.0 and NVIDIA AI-Q 2.1.0
in the namespace assigned to your LiveLabs reservation. The shared OKE cluster,
GPU nodes, NVIDIA operators, ECK operator, Istio ingress, and public HTTP
routes are prepared by the workshop platform.

You deploy only namespaced application resources from Cloud Shell. Do not
create a VCN, load balancer, namespace, Istio `Gateway`, Istio
`VirtualService`, or DNS record.

### Objectives

- Connect Cloud Shell to your assigned OKE namespace.
- Deploy the two-GPU RAG profile and validate its services.
- Deploy AI-Q in FRAG mode and connect it to RAG.
- Open the pre-provisioned RAG and AI-Q `nip.io` HTTP URLs.
- Generate a research report using private and web sources.

Estimated Time: 110 minutes, including initial NIM model download and startup and the deep-research exercise. This is a planning estimate pending the next end-to-end timing test.

## Architecture and workshop constraints

- The reservation provides one namespace: `$LAB_NAMESPACE`.
- The workshop platform creates the public routes before you begin. The URLs
  resolve before the applications are ready; they can return a temporary 503
  response until their frontend Service has endpoints.
- RAG and AI-Q frontend Services must be `ClusterIP`. The platform-owned
  Istio routes expose them as HTTP URLs.
- This profile uses two local A100 GPUs: one for Nemotron 3.5 Lightning
  generation and one for embeddings. Reranking and AI-Q research model calls
  use the NVIDIA API. GPU quota caps usage; the platform must also provide
  two available GPUs when admitting the reservation.
- PDF ingestion uses CPU text extraction. Use a text-based PDF for this lab;
  OCR, table, chart, and page-layout models are disabled in this profile.
- Your account is intentionally namespace-scoped. Commands that list nodes,
  storage classes, or cluster operators may be forbidden; those checks are
  performed by the lab platform.

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

The LiveLabs environment must provide the following values in the **Resources**
tab or startup output:

- `LAB_NAMESPACE`
- `OKE_CLUSTER_ID`
- `RAG_APP_URL`
- `AIQ_APP_URL`

You also need an NGC API key with access to the required NVIDIA charts and
models, an NVIDIA API key, and a Tavily API key. Accept the required model
terms in NGC before deployment. Lab 1, Task 2 generates a password for the
PostgreSQL database deployed alongside AI-Q; no existing database is needed.

Do not paste API keys or passwords into source control, screenshots, or chat.

Before starting Lab 1, obtain the required API keys:

- Follow the [NVIDIA API key instructions (PDF)](https://dam-cdn.nvd.orangelogic.com/AssetLink/1ib702mh10i74mgxnl3j80kw23q40158.pdf)
  to obtain your NVIDIA API key.
- Visit [Tavily](https://www.tavily.com/), sign up or sign in, and obtain your
  API key from your account dashboard.

## Workshop sequence

Complete **Get Started**, then work through the following labs in order:

1. Prepare the environment and credentials.
2. Deploy RAG, upload a document, and ask a question.
3. Deploy AI-Q, approve a research plan, and generate, verify, and export an OCI Supercluster decision brief using document and web sources.
4. Troubleshoot issues and clean up workshop application resources.

Keep the same Cloud Shell session and namespace throughout. If Cloud Shell
reconnects with a fresh session, restore your namespace and kubeconfig as shown
in Lab 1 before continuing.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
