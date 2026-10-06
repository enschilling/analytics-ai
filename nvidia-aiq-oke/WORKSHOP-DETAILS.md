# Workshop Details

## Short Description

Deploy NVIDIA RAG and AI-Q from OCI Cloud Shell in one attendee namespace
on a shared Oracle Kubernetes Engine GPU cluster.

## Workshop Variants

| Entry point | GPU allocation | Application profile |
| --- | ---: | --- |
| `workshops/sandbox-lite/index.html` | 2 | RAG 2.6.0 with local Nemotron 3.5 Lightning generation and embedding NIMs; NVIDIA API reranking; AI-Q 2.1.0 FRAG. |
| `workshops/sandbox/index.html` | 7 | RAG 2.2.0 with local generation, embedding, reranking and page-elements models; AI-Q 2.1.0 FRAG. |

The two-GPU variant is the default deployment path. Estimated workshop time
is 90 minutes for lite and 120 minutes for the seven-GPU variant, including
initial model downloads and startup.

The earlier `sandbox-api` and `sandbox-local` entry points route to the
canonical lite and seven-GPU guides respectively. The separate `tenancy`
manifest still describes the older bundle-based workflow.

## Long Description

Attendees use their LiveLabs identity to download kubeconfig for a shared OKE
cluster, configure API credentials, and install RAG and AI-Q charts into one
assigned namespace. The platform provisions namespace RBAC, GPU/workload
quotas, the kubeconfig IAM policy, and two deterministic HTTP nip.io routes.

RAG and AI-Q keep their frontend Services internal as ClusterIP. Both portal
URLs use the existing shared Istio Gateway and load balancer; attendees need
no OCI network, DNS, ingress, or load-balancer administration rights.

The lite profile uses two local GPUs, CPU PDF text extraction, NVIDIA API
reranking, and NVIDIA API model calls for AI-Q research. It is not an API-only
RAG deployment. The seven-GPU profile keeps additional RAG model workloads
local and requires a separately admitted seven-GPU reservation.

## Prerequisites

- OCI Cloud Shell and the supplied namespace, OKE cluster/region, kubeconfig
  setup commands, and two HTTP application URLs.
- Available GPU capacity and enough CPU, memory and storage on the shared
  cluster; a namespace quota alone does not reserve hardware.
- Platform-installed operators, storage, Istio HTTP Gateway and public ingress.
- NGC model/chart entitlement, NVIDIA API access and a Tavily API key.
- Helm 3; the tested RAG chart failed with Helm 4.
- A text-based PDF for the lite document workflow.

AI-Q deploys PostgreSQL within the attendee namespace. The guide generates
its password securely and stores application credentials in a Kubernetes
Secret. RAG Helm creates and owns its registry and NGC Secrets.

## Validation Status and Image Follow-up

On September 30, 2026, the two-GPU profile deployed through attendee Cloud
Shell, both NIMs became Ready, and both HTTP portals fully rendered. Initial
model startup took about 16 minutes. Document ingestion, question answering,
and research report generation still need functional validation. The
seven-GPU profile still needs a full deployment and functional test.

Existing RAG collection/citation images are cropped and reused. Older AI-Q
illustrations are omitted. Each canonical guide lists the new
Resources-panel, model-readiness, portal and completed-report screenshots to
capture during the next full test.

Terraform in `terraform/nvida-gpu` now defaults to two GPUs. A seven-GPU
environment must explicitly set `gpu_quota=7` and reserve sufficient
capacity before launch.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
