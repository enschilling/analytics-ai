# Lab 4: Troubleshoot and Clean Up Workshop Resources

## Introduction

Diagnose namespace-scoped issues and remove the workshop applications when you
have finished. This lab does not delete shared OCI or Kubernetes infrastructure.

### Objectives

- Recognize issues that require facilitator assistance.
- Uninstall AI-Q and RAG while preserving the platform and retained data.

Estimated Time: 10 minutes.

## Prerequisites

Use the same namespace and kubeconfig from Lab 1. Perform cleanup only after
finishing the application tests or when the instructor asks you to stop.

## Task 1: Troubleshoot Workshop Applications

- A missing `Elasticsearch` or `NIMService` API is a platform issue. Do not try
  to install cluster-wide operators from the attendee account.
- If a NIM model is not ready, inspect its status, pods, and namespace events:

    ```bash
    kubectl get nimservice
    kubectl describe nimservice
    kubectl get pods -o wide
    kubectl get events --sort-by=.metadata.creationTimestamp | tail -50
    ```

- If either portal URL resolves but returns a temporary error, verify the
  matching frontend deployment and Service endpoints. Do not create a new
  load balancer or alter Istio routing.
- If kubeconfig download returns `NotAuthorizedOrNotFound` immediately
  after provisioning, wait a minute and retry. If it persists, ask the
  facilitator to check the reservation IAM policy and assigned cluster.
- Namespaced NetworkPolicy permissions are required for the chart's Redis
  policy. They do not allow changes to OCI VCNs, subnets, or load balancers.

## Task 2: Clean Up Workshop Resources

- To remove application resources before your reservation ends:

    ```bash
    helm uninstall aiq210-frag
    helm uninstall rag
    ```

  PersistentVolumeClaims can retain data. Delete them only if the lab
  administrator confirms that the data is no longer needed.

## Conclusion

The Helm application releases have been removed. PersistentVolumeClaims may
retain data; do not delete the shared namespace, cluster, load balancer, or
network resources. Use **Need Help?** for additional assistance.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
