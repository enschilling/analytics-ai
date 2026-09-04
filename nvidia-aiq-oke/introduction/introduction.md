# Deploy NVIDIA AIQ Blueprint on Oracle Kubernetes Engine

## Introduction

NVIDIA RAG Blueprint and AIRA, the AIQ Research Assistant, give you a ready-to-run agentic AI application pattern with retrieval, NIM inference services, ingestion, a frontend, and integrations for NVIDIA and external search services. Oracle Kubernetes Engine gives the blueprint a managed Kubernetes foundation, and OCI GPU shapes provide the accelerator capacity for larger AI workloads.

In this workshop, you deploy RAG and AIRA from OCI Cloud Shell on a pre-provisioned OKE GPU cluster. You validate the existing cluster, prepare Kubernetes for a single-node GPU lab, deploy offline-ready manifests from Object Storage, pull mirrored images from OCIR, configure Squid proxy runtime access for pods that need internet, and test the application.

The workshop is designed for builders who need the deployment flow after the platform is ready. It focuses on the operational details learners must handle from Cloud Shell: GPU taints, CoreDNS scheduling, OCI CSI, Service Gateway access to OCI APIs, NGC runtime secrets, Tavily credentials, proxy-aware pod configuration, and application cleanup.

### Architecture

You will use this environment:

```text
Cloud Shell or learner browser
  |
  | kubectl port-forward or private load balancer
  v
OKE Enhanced Cluster
  |
  +-- rag / RAG services, Milvus, MinIO, Redis, etcd
  +-- rag / NVIDIA NIM model services
  +-- aira / AIRA frontend, backend, nginx, Phoenix
  |
  v
GPU worker subnet
  |
  v
One OCI A100 GPU node with 1 TB boot volume
  |
  v
LLfirewall Squid proxy for selected runtime internet egress
```

The OCI network uses three subnets:

```text
VCN: <resource_prefix>-vcn
  |
  +-- API subnet      public or private, Kubernetes API endpoint
  +-- Worker subnet   private, GPU node, Service Gateway plus DRG/proxy path
  +-- LB subnet       private or public, depending on validation mode
  |
  +-- Service Gateway OCI service access for APIs and Block Volume
  +-- DRG             path to LLfirewall and Squid proxy
  +-- Internet GW     optional public Kubernetes API or public load balancer
```

The deployment uses this security pattern:

```text
Images:
  Object Storage bundle -> rendered manifests
  Public OCIR           -> mirrored RAG and AIRA container images

Secrets:
  ngc-api               -> NVIDIA/NGC runtime access for NIM model files
  Tavily key            -> AIRA backend search integration

Runtime traffic:
  RAG/AIRA pods needing internet -> Squid proxy -> external services
  Milvus/etcd/MinIO/Redis        -> internal Kubernetes service traffic
```

### Objectives

- Prepare Cloud Shell, `kubectl`, NVIDIA NGC, and Tavily credentials.
- Validate a pre-provisioned enhanced OKE cluster with one A100 GPU node.
- Confirm the GPU node has the expected storage, DNS, CSI, and GPU resources.
- Fix common single-node GPU cluster scheduling blockers.
- Deploy NVIDIA RAG Blueprint and AIRA from offline-ready Object Storage bundles.
- Configure proxy variables only on pods that require runtime internet access.
- Validate NGC access, backend health, frontend access, service DNS, and PVC provisioning.
- Clean up the RAG and AIRA Kubernetes resources in the correct order.

Estimated Workshop Time: 70 minutes

### Prerequisites

- OCI Cloud Shell access in the target tenancy and region.
- A pre-provisioned OKE Enhanced cluster with one Ready A100 GPU node.
- `kubectl` configured in Cloud Shell for the target cluster.
- NVIDIA NGC API key with entitlement to the selected NIM models.
- Tavily API key.
- Public Object Storage URLs for the RAG and AIRA offline deployment bundles.

### Cost and Cleanup

This workshop uses a pre-provisioned GPU cluster. GPU nodes can generate significant cost, but learners do not create or delete OCI infrastructure in this lab. Complete the cleanup lab to remove the RAG and AIRA application resources when you finish testing.

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 11, 2026
