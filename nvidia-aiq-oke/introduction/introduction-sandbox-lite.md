# Deploy NVIDIA RAG and AI-Q on OKE: 2-GPU Workshop

## Introduction

In this workshop, you deploy NVIDIA RAG Blueprint 2.6.0 and NVIDIA AI-Q 2.1.0
in the namespace assigned to your LiveLabs reservation. Ingest a public document
and ask a grounded question through RAG, then use AI-Q to generate a short
answer with a web citation that you verify.

This is a hands-on example of the [Oracle + NVIDIA partnership](https://www.oracle.com/cloud/nvidia/):
Oracle provides managed Kubernetes, GPU compute, and storage on OCI; NVIDIA
provides reusable application blueprints, NIM inference microservices, and
research-agent software. You connect these components without building the
application pipelines from scratch.

The shared OKE cluster, GPU nodes, NVIDIA operators, ECK operator, Istio ingress,
and public HTTP routes are prepared by the workshop platform. You deploy only
namespaced application resources from Cloud Shell. Do not create a VCN, load
balancer, namespace, Istio `Gateway`, Istio `VirtualService`, or DNS record.

### Objectives

- Connect Cloud Shell to your assigned OKE namespace.
- Deploy the two-GPU RAG profile, ingest a document, and verify a cited answer.
- Deploy AI-Q in FRAG mode and verify its connection to the RAG services.
- Open the pre-provisioned RAG and AI-Q `nip.io` HTTP URLs.
- Generate a short AI-Q web-research answer and review its source citation.
- Explain the roles of OCI infrastructure and NVIDIA software and hosted models.

Estimated Time: 90 minutes, including initial NIM model download and startup.
This is a planning estimate; startup and hosted API response times can vary.

## Architecture and workshop constraints

- The reservation provides one namespace: `$LAB_NAMESPACE`.
- The workshop platform creates the public routes before you begin. The URLs
  resolve before the applications are ready; they can return a temporary 503
  response until their frontend Service has endpoints.
- RAG and AI-Q frontend Services must be `ClusterIP`. The platform-owned
  Istio routes expose them as HTTP URLs.
- This profile uses two local A100 GPUs: one for Nemotron 3.5 Lightning
  generation and one for embeddings. Reranking and AI-Q research model calls
  use the NVIDIA API. AI-Q's application and orchestration run on OKE, but its
  model inference is hosted by NVIDIA, not on the two local GPUs. GPU quota
  caps usage; the platform must also provide two available GPUs when admitting
  the reservation.
- Lab 3 creates a namespace-scoped ConfigMap with a replacement hosted-model
  configuration because the original endpoints were no longer available during
  testing. Credentials remain in Kubernetes Secrets. No administrator or
  additional GPU access is needed for this configuration.
- PDF ingestion uses CPU text extraction. Use a public, text-based PDF for
  this lab; OCR, table, chart, and page-layout models are disabled in this profile.
- Your account is intentionally namespace-scoped. Commands that list nodes,
  storage classes, or cluster operators may be forbidden; those checks are
  performed by the lab platform.
- The portals use HTTP. Do not upload private or confidential documents or
  submit sensitive research questions. This is a learning environment, not a
  production security configuration.
- AI-Q supports longer deep-research reports, but report generation and export
  are outside this workshop's scope. The short-answer exercise does not validate
  the full report workflow.

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
2. Deploy RAG, upload a public document, and verify a document-grounded answer.
3. Configure and deploy AI-Q, check its health, and generate a short web-research
   answer with an Oracle citation.
4. Troubleshoot issues and clean up workshop application resources.

Keep the same Cloud Shell session and namespace throughout. If Cloud Shell
reconnects with a fresh session, restore your namespace and kubeconfig as shown
in Lab 1 before continuing.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
