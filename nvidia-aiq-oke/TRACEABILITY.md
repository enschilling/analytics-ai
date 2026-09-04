# Traceability

Estimated Time: 5 minutes

## Introduction

This file maps the workshop content to the source material and field validation used to author the labs. It is intended for maintainers and reviewers.

### Objectives

- Identify the source for each major technical decision.
- Record the field lessons that shaped the lab.
- Track placeholders that must remain generic for publication.

## Source Material

| Source | Used For |
| --- | --- |
| OCI OKE and networking command patterns from the existing `livelabs/nim-on-oke` workshop | Pre-provisioned cluster assumptions, kubeconfig validation, Service Gateway checks, and cleanup guardrails |
| Local `oke-nvidia` skill notes | GPU shape guidance, OKE Gen2 GPU image lessons, 1 TB boot volume requirement, GPU taint behavior, CoreDNS troubleshooting |
| Field deployment on OCI, June 2026 | RAG and AIRA deployment sequence, Object Storage bundles, public OCIR image mirror, Squid proxy runtime access, CoreDNS toleration fix, OCI CSI troubleshooting, RAG and AIRA health validation |
| NVIDIA RAG Blueprint and AIRA deployment artifacts | Rendered manifest deployment structure, RAG namespace, AIRA namespace, NGC runtime secret, Tavily backend configuration |

## Field Lessons Included

- Use an OKE `ENHANCED_CLUSTER`.
- Use a fresh three-subnet network: API, workers, and load balancers.
- Use public OCIR image mirrors and Object Storage bundles so Cloud Shell and private workers do not need to pull application images from public registries.
- Use Service Gateway access from worker subnets for OCI APIs such as IAAS and Block Volume.
- Use LLfirewall DRG and Squid proxy for selected runtime internet egress.
- Use an OKE `Gen2-GPU` image OCID with `image_id`, not a generic compute image.
- Match the Kubernetes version exactly between cluster and node image.
- Use a 1 TB boot volume for GPU nodes.
- Verify Kubernetes sees the expanded root filesystem; repair host LVM if it reports about 37 GiB.
- On a single-node GPU cluster, patch CoreDNS and DNS autoscaler with the GPU toleration or remove the GPU taint for lab-only environments.
- Store NGC and Tavily credentials only in Kubernetes secrets.
- Patch RAG and AIRA workloads with GPU tolerations when the GPU taint is present.
- Apply proxy environment variables only to workloads that need external runtime access; keep Milvus, etcd, MinIO, and Redis off the proxy.
- Validate NGC access through Squid before waiting for NIM startup.
- Validate AIRA through the nginx `/health` endpoint before exposing it more broadly.
- Learners delete only RAG and AIRA Kubernetes resources. The instructor or lab owner manages shared OKE, VCN, DRG, firewall, and proxy lifecycle.

## Placeholders That Must Stay Generic

- `<oci_profile>`
- `<region>`
- `<compartment_ocid>`
- `<availability_domain>`
- `<node_image_ocid>`
- `<llfirewall_drg_ocid>`
- `<ngc_api_key>`
- `<tavily_api_key>`

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 11, 2026
