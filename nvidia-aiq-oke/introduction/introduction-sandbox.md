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
