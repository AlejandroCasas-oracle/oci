# Prepare the GPU Node

## Introduction

In this lab, you prepare Kubernetes for a one-node GPU workshop. OKE GPU nodes can carry the taint `nvidia.com/gpu=present:NoSchedule`. That taint is useful in production because it protects GPU nodes from ordinary workloads, but it can block system pods and lab workloads when the cluster has only one node.

You will also verify that the 1 TB boot volume is visible to Kubernetes. If the host root filesystem did not grow, image pulls and AIQ containers can fill the default root volume and trigger `DiskPressure`.

### Objectives

- Inspect GPU taints.
- Keep CoreDNS and the DNS autoscaler schedulable on the GPU node.
- Verify service DNS.
- Confirm the 1 TB boot volume is visible to Kubernetes.
- Validate GPU resources.

Estimated Time: 15 minutes

## Task 1: Inspect Node Taints

1. Display node taints.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -i taints
    ```

2. If you see `nvidia.com/gpu=present:NoSchedule`, keep it for this lab and add tolerations to the pods that must run on the single GPU node.

3. For a simple demo-only environment, you can remove the GPU taint instead. The rest of this lab uses the toleration approach.

    ```bash
    # Optional demo-only shortcut:
    # kubectl taint node "$NODE_NAME" nvidia.com/gpu=present:NoSchedule- || true
    # kubectl taint node "$NODE_NAME" nvidia.com/gpu:NoSchedule- || true
    ```

## Task 2: Patch CoreDNS for a Single GPU Node

1. Check whether CoreDNS is running.

    ```bash
    kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
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
    kubectl rollout status deployment/kube-dns-autoscaler -n kube-system --timeout=180s
    ```

5. Confirm the pods are running.

    ```bash
    kubectl get pods -n kube-system -o wide
    ```

## Task 3: Test Service DNS

1. Run a short DNS check from a temporary pod.

    ```bash
    kubectl run dns-check \
      --image=curlimages/curl:8.10.1 \
      --restart=Never \
      --overrides='{"spec":{"tolerations":[{"key":"nvidia.com/gpu","operator":"Equal","value":"present","effect":"NoSchedule"}]}}' \
      --command -- sh -c 'nslookup kubernetes.default.svc.cluster.local && echo DNS_OK'
    ```

2. Read the logs.

    ```bash
    kubectl logs pod/dns-check
    ```

3. Remove the temporary pod.

    ```bash
    kubectl delete pod dns-check --ignore-not-found=true
    ```

## Task 4: Verify Boot Volume Expansion

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
    export REPAIR_IMAGE=$(kubectl -n kube-system get pod "$REPAIR_POD" -o jsonpath='{.spec.containers[0].image}')

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

    kubectl apply -f /tmp/oke-grow-root-job.yaml
    kubectl -n kube-system wait --for=condition=complete job/oke-grow-root --timeout=10m
    kubectl -n kube-system logs job/oke-grow-root
    kubectl -n kube-system delete job oke-grow-root
    ```

5. Recheck storage.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -i "DiskPressure"
    kubectl describe node "$NODE_NAME" | grep -A8 -i "ephemeral-storage"
    ```

## Task 5: Confirm GPU Resources

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
* **Last Updated By/Date** - Alejandro Casas, June 4, 2026
