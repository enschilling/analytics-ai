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

Only `sandbox-lite`, `sandbox`, and `tenancy` remain under `workshops/`.
Each variant contains only `index.html` and `manifest.json`. The removed
compatibility entry points must be replaced in any published workshop links.
The separate `tenancy` manifest retains the older bundle-based workflow.

### Content layout

Both Sandbox manifests follow Introduction, shared Get Started, Labs 1–4,
and shared Need Help. Lab 1 prepares the environment, Lab 2 deploys/tests
RAG, Lab 3 deploys/tests AI-Q, and Lab 4 troubleshoots and cleans up.

Markdown lives in the root-level `introduction`, `prepare-environment`,
`deploy-rag-blueprint`, `deploy-aiq-research-assistant`, and
`validate-and-clean-up` folders. The `-sandbox-lite.md` and `-sandbox.md`
suffixes distinguish commands and model profiles without duplicating the
HTML launcher. Reused screenshots live in root-level `shared-images/`.
The existing unsuffixed files still serve the legacy Tenancy workflow.

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

On October 6, 2026, the two-GPU profile passed attendee provisioning,
deployment, portal access, PDF ingestion and RAG question answering with
ranked document sources. The hosted VL reranker settings are included in
Lab 2. AI-Q web-search chat passed with a live ConfigMap model override;
that pinned-config override still needs to be encoded in the source Lab 3
instructions. Full deep-research report generation is not yet validated.
The seven-GPU profile still needs a full deployment and functional test.

Existing RAG collection/citation images are cropped and reused. Older AI-Q
illustrations are omitted. Capture fresh Resources-panel, model-readiness,
portal, document-upload, and completed-report screenshots during full tests.
Redact reservation identifiers and secrets; keep images within 1180x1180.

Terraform in `terraform/nvida-gpu` now defaults to two GPUs. A seven-GPU
environment must explicitly set `gpu_quota=7` and reserve sufficient
capacity before launch.

## Screenshot Capture Checklist

For both Sandbox variants, capture the Resources panel, assigned HTTP portals,
document upload, RAG answer/citations, and AI-Q web-search/completed-report
states. Redact reservation-specific identifiers and secrets.

For the seven-GPU test, also capture model readiness showing all four local
NIM roles. The two-GPU images must show the two-local-model profile rather
than the seven-GPU deployment. Keep fresh screenshots in `shared-images/`
and within 1180 pixels on either dimension.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
