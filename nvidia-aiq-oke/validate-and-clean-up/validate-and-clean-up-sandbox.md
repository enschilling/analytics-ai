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

- A missing NIM or Elasticsearch API is a platform issue. Do not install or
  modify cluster-wide operators from the attendee account.
- If a local model does not become ready, inspect namespace-scoped status and
  events:

    ```bash
    kubectl get deployments,statefulsets
    kubectl describe pods
    kubectl get pods -o wide
    kubectl get events --sort-by=.metadata.creationTimestamp | tail -50
    ```

- If an application URL resolves but is temporarily unavailable, verify its
  frontend Service endpoints. Do not create a load balancer or change Istio.

## Task 2: Clean Up Workshop Resources

- To remove both applications before your reservation ends:

    ```bash
    helm uninstall aiq210-frag
    helm uninstall rag
    ```

  PersistentVolumeClaims can retain data. Delete them only with instructor
  approval.

## Conclusion

The Helm application releases have been removed. PersistentVolumeClaims may
retain data; do not delete the shared namespace, cluster, load balancer, or
network resources. Use **Need Help?** for additional assistance.

## Acknowledgements

- **Authors:** Oracle and NVIDIA workshop team
- **Last Updated:** October 2026
