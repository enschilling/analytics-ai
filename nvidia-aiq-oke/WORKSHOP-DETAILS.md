# Workshop Details

Estimated Time: 5 minutes

## Short Description

Deploy NVIDIA RAG Blueprint and AIRA from OCI Cloud Shell on a pre-provisioned Oracle Kubernetes Engine GPU cluster.

## Long Description

This workshop teaches OCI practitioners how to deploy NVIDIA RAG Blueprint and AIRA from OCI Cloud Shell on a pre-provisioned Oracle Kubernetes Engine GPU cluster. Learners validate the existing cluster, download offline deployment bundles from Object Storage, deploy manifests that reference public OCIR images, configure Squid proxy variables only for pods that need runtime internet access, validate RAG and AIRA, and clean up the application resources safely.

The lab emphasizes the operational details that made the deployment successful: validating boot volume expansion, handling the `nvidia.com/gpu=present:NoSchedule` taint on single-node GPU clusters, keeping CoreDNS schedulable, verifying Service Gateway access to OCI APIs, fixing OCI CSI topology labels when needed, keeping internal Milvus/etcd/MinIO traffic off the proxy, and storing API keys only in Kubernetes secrets.

## Workshop Outline

1. Introduction
2. Lab 1 - Prepare Your Environment
3. Lab 2 - Validate the Pre-Provisioned GPU Cluster
4. Lab 3 - Deploy NVIDIA RAG Blueprint and AIRA
5. Lab 4 - Validate and Clean Up

## Workshop Prerequisites

- OCI Cloud Shell access in the target tenancy and region.
- A pre-provisioned OKE Enhanced cluster with one Ready A100 GPU node.
- `kubectl` configured in Cloud Shell for the target cluster.
- Worker networking already configured for OCI Service Gateway access and LLfirewall Squid proxy egress.
- NVIDIA NGC API key with access to the selected NIM models.
- Tavily API key for AIQ web-search integrations.
- Public Object Storage URLs for the RAG and AIRA deployment bundles.
- Public OCIR repositories that contain the mirrored RAG and AIRA images.

## Notes

- This workshop uses placeholders such as `<compartment_ocid>`, `<region>`, `<ngc_api_key>`, and `<tavily_api_key>`. Replace them with values from your tenancy. The RAG and AIRA Object Storage bundle URLs are already populated in the lab instructions.
- Do not store real API keys, private keys, or kubeconfig contents in the workshop repository.
- GPU resources can be expensive. The instructor or lab environment owner manages the OKE cluster lifecycle.
- This workshop is command-driven. Learners deploy and validate Kubernetes resources with `kubectl` from Cloud Shell.

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 11, 2026
