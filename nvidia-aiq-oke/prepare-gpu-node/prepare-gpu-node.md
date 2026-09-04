# Validate the Pre-Provisioned GPU Cluster

## Introduction

In this lab, you validate the pre-provisioned OKE GPU cluster before deploying RAG and AIRA. OKE GPU nodes can carry the taint `nvidia.com/gpu=present:NoSchedule`. That taint is useful in production because it protects GPU nodes from ordinary workloads, but it can block system pods and lab workloads when the cluster has only one node.

You will also verify that the 1 TB boot volume is visible to Kubernetes. If the host root filesystem did not grow, image pulls and RAG containers can fill the default root volume and trigger `DiskPressure`. In a managed lab, ask the instructor before running repair commands that change the node.

### Objectives

- Inspect GPU taints.
- Keep CoreDNS and the DNS autoscaler schedulable on the GPU node.
- Verify service DNS.
- Verify OCI service reachability through the Service Gateway.
- Confirm the OCI CSI node plugin can provision Block Volume PVCs.
- Confirm the 1 TB boot volume is visible to Kubernetes.
- Validate GPU resources.

Estimated Time: 15 minutes

## Task 1: Inspect Node Taints

1. Capture the node name from the cluster.

    ```bash
    export NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
    echo "$NODE_NAME"
    ```

2. Display node taints.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -i taints
    ```

3. If you see `nvidia.com/gpu=present:NoSchedule`, keep it for this lab and add tolerations to the pods that must run on the single GPU node.

4. For a simple demo-only environment, you can remove the GPU taint instead. The rest of this lab uses the toleration approach.

    ```bash
    # Optional demo-only shortcut:
    # kubectl taint node "$NODE_NAME" nvidia.com/gpu=present:NoSchedule- || true
    # kubectl taint node "$NODE_NAME" nvidia.com/gpu:NoSchedule- || true
    ```

## Task 2: Patch CoreDNS for a Single GPU Node

1. Check whether CoreDNS is running.

    ```bash
    kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
    ```
    ```bash
    kubectl get deployment coredns -n kube-system
    ```

2. If CoreDNS is Pending, patch the deployment with the GPU toleration.

    ```bash
    kubectl patch deployment coredns -n kube-system --type=json \
      -p='[{"op":"add","path":"/spec/template/spec/tolerations/-","value":{"key":"nvidia.com/gpu","operator":"Equal","value":"present","effect":"NoSchedule"}}]'
    ```

3. Patch the DNS autoscaler as well.

    ```bash
    kubectl patch deployment kube-dns-autoscaler -n kube-system --type=merge \
      -p='{"spec":{"template":{"spec":{"tolerations":[{"key":"nvidia.com/gpu","operator":"Equal","value":"present","effect":"NoSchedule"}]}}}}'
    ```

4. Wait for both rollouts.

    ```bash
    kubectl rollout status deployment/coredns -n kube-system --timeout=180s
    ```
    ```bash
    kubectl rollout status deployment/kube-dns-autoscaler -n kube-system --timeout=180s
    ```

5. Confirm the pods are running.

    ```bash
    kubectl get pods -n kube-system -o wide
    ```

## Task 3: Test Service DNS and OCI API Reachability

1. Run a short DNS check from a temporary pod.

    This lab uses a regional OCIR-hosted OKE image so the check works in restricted or offline clusters. The image does not include `nslookup`, so use `getent hosts` to validate Kubernetes service DNS.

    ```bash
    kubectl delete pod dns-check --ignore-not-found=true

    kubectl run dns-check \
      --image=ap-melbourne-1.ocir.io/id9y6mi8tcky/oke-public-cloud-provider-oci:v1.32-1dc23aae3b5-103-csi \
      --restart=Never \
      --overrides='{"spec":{"tolerations":[{"operator":"Exists"}]}}' \
      --command -- sh -c 'getent hosts kubernetes.default.svc.cluster.local && echo DNS_OK'
    ```

2. Read the logs.

    ```bash
    kubectl logs pod/dns-check
    ```

3. Remove the temporary pod.

    ```bash
    kubectl delete pod dns-check --ignore-not-found=true
    ```

4. Test the regional OCI IAAS endpoint.

    The RAG deployment creates PVCs backed by OCI Block Volume. The control plane and CSI plugin need to reach OCI APIs. A `404` response is healthy for this test because it proves DNS and network connectivity to the API endpoint.

    ```bash
    export OCI_REGION="ap-melbourne-1"

    kubectl run iaas-test \
      --rm -it \
      --restart=Never \
      --image=ap-melbourne-1.ocir.io/id9y6mi8tcky/oke-public-cloud-provider-oci:v1.32-1dc23aae3b5-103-csi \
      --env OCI_REGION="$OCI_REGION" \
      --overrides='{"spec":{"tolerations":[{"operator":"Exists"}]}}' \
      -- sh -lc 'curl -I --connect-timeout 10 "https://iaas.${OCI_REGION}.oraclecloud.com/20160918/"'
    ```

5. Continue if the command returns `HTTP/1.1 404 Not Found`. If it times out, check that the worker route table has a Service Gateway route for regional OCI services.

## Task 4: Validate OCI CSI for Block Volumes

1. Confirm the OCI CSI node plugin is running.

    RAG uses PersistentVolumeClaims for Milvus, etcd, MinIO, and Redis. Those PVCs will stay pending if the CSI node plugin is not healthy.

    ```bash
    kubectl get pods -n kube-system -o wide | egrep 'csi-oci-node|csi'
    ```

2. If `csi-oci-node` is in `CrashLoopBackOff`, inspect the previous logs.

    ```bash
    export CSI_POD=$(kubectl get pods -n kube-system -l app=csi-oci-node -o jsonpath='{.items[0].metadata.name}')
    kubectl logs -n kube-system "$CSI_POD" -c csi-node-driver --previous --tail=200
    ```

3. If the log says it cannot get the node availability domain, add Kubernetes-safe topology labels.

    OCI availability domain names can contain a colon, but Kubernetes label values cannot. Use the AD suffix without the tenancy prefix, such as `AP-MELBOURNE-1-AD-1`.

    ```bash
    kubectl label node "$NODE_NAME" \
      topology.kubernetes.io/region="$REGION" \
      topology.kubernetes.io/zone=AP-MELBOURNE-1-AD-1 \
      failure-domain.beta.kubernetes.io/region="$REGION" \
      failure-domain.beta.kubernetes.io/zone=AP-MELBOURNE-1-AD-1 \
      --overwrite
    ```

4. Restart the CSI node pod and wait until it is healthy.

    ```bash
    kubectl delete pod -n kube-system -l app=csi-oci-node
    ```
    ```bash
    kubectl get pods -n kube-system -w | egrep 'csi-oci-node|coredns|kube-dns'
    ```

5. Do not continue until CoreDNS, kube-dns-autoscaler, and csi-oci-node are all `Running`.

## Task 5: Verify Boot Volume Expansion

1. Confirm the node does not have disk pressure.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -i "DiskPressure"
    ```

2. Check the ephemeral storage reported by kubelet.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -A8 -i "ephemeral-storage"
    ```

3. Continue if Kubernetes reports roughly 1 TB of ephemeral storage and `DiskPressure=False`.

4. If Kubernetes reports roughly 37 GiB, the OCI boot volume was created at 1 TB but the Oracle Linux root filesystem did not finish expanding. Create a privileged maintenance job that runs on the host.

    ```bash
    export REPAIR_POD=$(kubectl -n kube-system get pods -o name | grep oke-node-problem-detector | head -1 | cut -d/ -f2)
    ```
    ```bash
    export REPAIR_IMAGE=$(kubectl -n kube-system get pod "$REPAIR_POD" -o jsonpath='{.spec.containers[0].image}')
    ```
    
    ```bash
    kubectl delete job -n kube-system oke-grow-root --ignore-not-found=true

    cat > /tmp/oke-grow-root-job.yaml <<EOF
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: oke-grow-root
      namespace: kube-system
    spec:
      backoffLimit: 0
      template:
        spec:
          restartPolicy: Never
          priorityClassName: system-node-critical
          hostPID: true
          tolerations:
          - operator: Exists
          containers:
          - name: grow-root
            image: ${REPAIR_IMAGE}
            securityContext:
              privileged: true
            command:
            - /bin/bash
            - -lc
            - |
              chroot /host growpart /dev/sda 3 || true
              chroot /host partprobe /dev/sda || true
              chroot /host pvresize /dev/sda3
              chroot /host lvextend -r -l +100%FREE /dev/mapper/ocivolume-root
              chroot /host xfs_growfs /
              chroot /host df -hT /
              chroot /host systemctl restart crio
              chroot /host systemctl restart kubelet
            volumeMounts:
            - name: host-root
              mountPath: /host
          volumes:
          - name: host-root
            hostPath:
              path: /
    EOF
    ```
    ```bash
    kubectl apply -f /tmp/oke-grow-root-job.yaml
    ```
    ```bash
    kubectl -n kube-system logs job/oke-grow-root -f
    ```

    The job can remain `Running` after restarting `crio` or `kubelet`. That is acceptable if the log shows that the logical volume grew. Look for output similar to:

    ```bash
    Size of logical volume ocivolume/root changed from 35.50 GiB to <988.90 GiB
    ```

5. Confirm the host filesystem grew.

    If the job log still shows `/dev/mapper/ocivolume-root` near `36G`, run a second privileged job that grows the XFS filesystem explicitly.

    ```bash
    kubectl delete job -n kube-system oke-grow-xfs --ignore-not-found=true

    cat > /tmp/oke-grow-xfs-job.yaml <<EOF
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: oke-grow-xfs
      namespace: kube-system
    spec:
      backoffLimit: 0
      template:
        spec:
          restartPolicy: Never
          priorityClassName: system-node-critical
          hostPID: true
          tolerations:
          - operator: Exists
          containers:
          - name: grow-xfs
            image: ${REPAIR_IMAGE}
            securityContext:
              privileged: true
            command:
            - /bin/bash
            - -lc
            - |
              set -eux
              chroot /host df -hT /
              chroot /host lvs
              chroot /host xfs_growfs /
              chroot /host df -hT /
              chroot /host systemctl restart kubelet
            volumeMounts:
            - name: host-root
              mountPath: /host
          volumes:
          - name: host-root
            hostPath:
              path: /
    EOF
    ```
    ```bash
    kubectl apply -f /tmp/oke-grow-xfs-job.yaml
    ```
    ```bash
    kubectl -n kube-system logs job/oke-grow-xfs -f
    ```

    Continue when the log shows the root filesystem near `989G`, for example:

    ```text
    /dev/mapper/ocivolume-root xfs 989G 35G 955G 4% /
    ```

6. Recheck storage after kubelet republishes node capacity.

    Wait about one minute after the grow job restarts kubelet, then run:

    ```bash
    kubectl describe node "$NODE_NAME" | grep -i "DiskPressure"
    ```
    ```bash
    kubectl describe node "$NODE_NAME" | grep -A8 -i "ephemeral-storage"
    ```

    Kubernetes should now report roughly 1 TB of ephemeral storage, for example:

    ```text
    ephemeral-storage:  1036916992Ki
    ```

7. Remove the maintenance jobs.

    ```bash
    kubectl delete job -n kube-system oke-grow-xfs --ignore-not-found=true
    kubectl delete job -n kube-system oke-grow-root --ignore-not-found=true
    ```

## Task 6: Confirm GPU Resources

1. Check NVIDIA system pods.

    ```bash
    kubectl get pods -n kube-system | grep -i nvidia || true
    ```

2. Confirm allocatable GPUs.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -A12 "Allocatable:" | grep nvidia || true
    ```

3. For a `BM.GPU4.8` node, expect `nvidia.com/gpu: 8`. If you use a different A100 shape, match the expected count to that shape.

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 11, 2026
