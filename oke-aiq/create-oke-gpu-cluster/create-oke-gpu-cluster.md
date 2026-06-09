# Create the OKE GPU Cluster

## Introduction

In this lab, you create an enhanced OKE cluster and a one-node A100 GPU node pool. The worker image is the most important choice: use an OKE Gen2 GPU image that matches the cluster Kubernetes version exactly, and pass the image OCID as `image_id`.

Do not use a standard compute image for GPU nodes. Do not rely on `node_image_name` when a valid OKE GPU image OCID is available.

### Objectives

- Create an enhanced OKE cluster.
- Fetch kubeconfig with the correct OCI profile.
- Find a matching OKE Gen2 GPU worker image.
- Create a one-node A100 GPU node pool.
- Confirm the node joins the cluster.

Estimated Time: 25 minutes

## Task 1: Create the Enhanced OKE Cluster

1. Create the cluster options file.

    ```bash
    cat > /tmp/aiq-cluster-options.json <<EOF
    {
      "serviceLbSubnetIds": ["$LB_SUBNET_OCID"],
      "kubernetesNetworkConfig": {
        "podsCidr": "$PODS_CIDR",
        "servicesCidr": "$SERVICES_CIDR"
      }
    }
    EOF
    ```

2. Create the enhanced cluster. This lab enables a public Kubernetes API endpoint for learner convenience. For production, use a private endpoint with Bastion or an admin host.

    ```bash
    export CLUSTER_NAME="${RESOURCE_PREFIX}-oke"

    export CLUSTER_WORK_REQUEST_OCID=$(oci ce cluster create \
      --compartment-id "$COMPARTMENT_OCID" \
      --name "$CLUSTER_NAME" \
      --vcn-id "$VCN_OCID" \
      --kubernetes-version "$KUBERNETES_VERSION" \
      --type ENHANCED_CLUSTER \
      --endpoint-subnet-id "$API_SUBNET_OCID" \
      --endpoint-public-ip-enabled true \
      --cluster-pod-network-options '[{"cniType":"FLANNEL_OVERLAY"}]' \
      --options file:///tmp/aiq-cluster-options.json \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query '"opc-work-request-id"' \
      --raw-output)

    echo "$CLUSTER_WORK_REQUEST_OCID"
    ```

3. Poll the work request until it succeeds.

    ```bash
    watch -n 30 "oci ce work-request get \
      --work-request-id $CLUSTER_WORK_REQUEST_OCID \
      --profile $OCI_CLI_PROFILE \
      --region $REGION \
      --query 'data.status' \
      --raw-output"
    ```

4. Capture the cluster OCID.

    ```bash
    export CLUSTER_OCID=$(oci ce cluster list \
      --compartment-id "$COMPARTMENT_OCID" \
      --name "$CLUSTER_NAME" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data[0].id' \
      --raw-output)

    echo "$CLUSTER_OCID"
    ```

## Task 2: Configure kubectl

1. Fetch kubeconfig for the cluster.

    ```bash
    mkdir -p "$HOME/.kube"

    oci ce cluster create-kubeconfig \
      --cluster-id "$CLUSTER_OCID" \
      --file "$HOME/.kube/config" \
      --region "$REGION" \
      --token-version 2.0.0 \
      --kube-endpoint PUBLIC_ENDPOINT \
      --profile "$OCI_CLI_PROFILE"
    ```

2. Confirm the kubeconfig exec plugin includes your OCI profile.

    ```bash
    kubectl config view --minify --raw | grep -A20 "exec:"
    ```

3. If the profile is missing, add it to the kubeconfig user exec args.

    ```bash
    python3 - <<'PY'
    import os
    import pathlib
    import yaml

    path = pathlib.Path.home() / ".kube" / "config"
    profile = os.environ["OCI_CLI_PROFILE"]
    data = yaml.safe_load(path.read_text())
    for user in data.get("users", []):
        exec_cfg = user.get("user", {}).get("exec")
        if not exec_cfg:
            continue
        args = exec_cfg.setdefault("args", [])
        if "--profile" not in args:
            args.extend(["--profile", profile])
    path.write_text(yaml.safe_dump(data, sort_keys=False))
    PY
    ```

4. Test API access.

    ```bash
    kubectl get namespaces
    ```

## Task 3: Find the Matching OKE Gen2 GPU Image

1. List node image sources that match the Kubernetes version and contain `Gen2-GPU`.

    ```bash
    oci ce node-pool-options get \
      --node-pool-option-id "$CLUSTER_OCID" \
      --compartment-id "$COMPARTMENT_OCID" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      | jq -r --arg k8s "$KUBERNETES_VERSION" '.data.sources[]? | select(."source-name" | contains("Gen2-GPU") and contains($k8s)) | [."source-name", ."image-id"] | @tsv'
    ```

2. Export the selected image OCID. Choose the newest matching `Oracle-Linux-8.*-Gen2-GPU-*` image.

    ```bash
    export NODE_IMAGE_OCID="<node_image_ocid>"
    echo "$NODE_IMAGE_OCID"
    ```

3. Stop if no image appears. The node image Kubernetes version must match the cluster version exactly.

## Task 4: Create the A100 GPU Node Pool

1. Create node pool source and placement JSON files.

    ```bash
    cat > /tmp/aiq-node-source.json <<EOF
    {
      "sourceType": "IMAGE",
      "imageId": "$NODE_IMAGE_OCID",
      "bootVolumeSizeInGBs": $BOOT_VOLUME_GB
    }
    EOF

    cat > /tmp/aiq-node-config.json <<EOF
    {
      "size": 1,
      "placementConfigs": [
        {
          "availabilityDomain": "$AVAILABILITY_DOMAIN",
          "subnetId": "$WORKER_SUBNET_OCID"
        }
      ],
      "nodePoolPodNetworkOptionDetails": {
        "cniType": "FLANNEL_OVERLAY"
      }
    }
    EOF
    ```

2. Create the node pool.

    ```bash
    export NODE_POOL_NAME="${RESOURCE_PREFIX}-a100-pool"

    export NODE_POOL_WORK_REQUEST_OCID=$(oci ce node-pool create \
      --compartment-id "$COMPARTMENT_OCID" \
      --cluster-id "$CLUSTER_OCID" \
      --name "$NODE_POOL_NAME" \
      --kubernetes-version "$KUBERNETES_VERSION" \
      --node-shape "$GPU_SHAPE" \
      --node-source-details file:///tmp/aiq-node-source.json \
      --node-config-details file:///tmp/aiq-node-config.json \
      --initial-node-labels '{"workload":"nvidia-aiq","accelerator":"a100"}' \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query '"opc-work-request-id"' \
      --raw-output)

    echo "$NODE_POOL_WORK_REQUEST_OCID"
    ```

3. Poll the work request.

    ```bash
    watch -n 30 "oci ce work-request get \
      --work-request-id $NODE_POOL_WORK_REQUEST_OCID \
      --profile $OCI_CLI_PROFILE \
      --region $REGION \
      --query 'data.status' \
      --raw-output"
    ```

4. Capture the node pool OCID.

    ```bash
    export NODE_POOL_OCID=$(oci ce node-pool list \
      --compartment-id "$COMPARTMENT_OCID" \
      --cluster-id "$CLUSTER_OCID" \
      --name "$NODE_POOL_NAME" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data[0].id' \
      --raw-output)

    echo "$NODE_POOL_OCID"
    ```

## Task 5: Wait for the Node

1. Wait for the node to appear.

    ```bash
    watch -n 20 kubectl get nodes -o wide
    ```

2. Capture the node name when it is `Ready`.

    ```bash
    export NODE_NAME=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
    echo "$NODE_NAME"
    ```

3. Check GPU capacity.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -A8 "Capacity:" | grep nvidia || true
    kubectl describe node "$NODE_NAME" | grep -A8 "Allocatable:" | grep nvidia || true
    ```

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 4, 2026
