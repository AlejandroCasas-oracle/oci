# Prepare Your Environment

## Introduction

In this lab, you prepare the command-line environment for the AIQ deployment. You will set reusable variables, verify your OCI profile, check tool versions, and confirm that the target region can run the selected GPU shape.

The commands use placeholders. Replace them with values from your tenancy and keep secrets out of shell history when possible.

### Objectives

- Configure reusable environment variables.
- Verify OCI CLI, `kubectl`, Helm, and `jq`.
- Confirm the OCI profile, region, and compartment.
- Prepare NVIDIA NGC and Tavily credentials for Kubernetes secrets.
- Check GPU shape availability before creating the cluster.

Estimated Time: 15 minutes

## Task 1: Set Workshop Variables

1. Set the OCI profile, region, and compartment.

    ```bash
    export OCI_CLI_PROFILE="<oci_profile>"
    export REGION="<region>"
    export COMPARTMENT_OCID="<compartment_ocid>"
    export RESOURCE_PREFIX="aiq-a100-lab"
    export KUBERNETES_VERSION="v1.32.1"
    ```

2. Set the GPU shape. This lab uses one A100 GPU node. On many OCI tenancies, `BM.GPU4.8` provides A100 40 GB GPUs.

    ```bash
    export GPU_SHAPE="BM.GPU4.8"
    export GPU_EXPECTED_COUNT="8"
    export BOOT_VOLUME_GB="1000"
    ```

3. Select the availability domain after checking capacity.

    ```bash
    export AVAILABILITY_DOMAIN="<availability_domain>"
    ```

4. Set the network CIDR plan.

    ```bash
    export VCN_CIDR="10.91.0.0/16"
    export API_SUBNET_CIDR="10.91.0.0/28"
    export WORKER_SUBNET_CIDR="10.91.1.0/24"
    export LB_SUBNET_CIDR="10.91.2.0/24"
    export PODS_CIDR="10.244.0.0/16"
    export SERVICES_CIDR="10.96.0.0/16"
    ```

## Task 2: Verify Local Tools

1. Confirm OCI CLI can authenticate.

    ```bash
    oci iam region list \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --output table
    ```

2. Confirm the required local tools.

    ```bash
    oci --version
    kubectl version --client=true
    helm version
    jq --version
    ```

3. Confirm the target compartment is visible.

    ```bash
    oci iam compartment get \
      --compartment-id "$COMPARTMENT_OCID" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data."name"' \
      --raw-output
    ```

## Task 3: Capture Credentials Safely

1. Read the NVIDIA NGC API key into an environment variable.

    ```bash
    read -rsp "NGC API key: " NGC_API_KEY
    echo
    export NGC_API_KEY
    ```

2. Read the Tavily API key into an environment variable.

    ```bash
    read -rsp "Tavily API key: " TAVILY_API_KEY
    echo
    export TAVILY_API_KEY
    ```

3. Do not print these values. Confirm only that they are set.

    ```bash
    test -n "$NGC_API_KEY" && echo "NGC_API_KEY is set"
    test -n "$TAVILY_API_KEY" && echo "TAVILY_API_KEY is set"
    ```

## Task 4: Check GPU Shape and OKE Version

1. List availability domains.

    ```bash
    oci iam availability-domain list \
      --compartment-id "$COMPARTMENT_OCID" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query 'data[].name' \
      --output table
    ```

2. Check whether the GPU shape appears in the region.

    ```bash
    oci compute shape list \
      --compartment-id "$COMPARTMENT_OCID" \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --all \
      | jq -r --arg shape "$GPU_SHAPE" '.data[] | select(.shape == $shape) | [.shape, ."processor-description", ."gpus", ."gpu-description"] | @tsv'
    ```

3. If the selected shape does not appear, stop and select a supported A100 shape in your region. Do not silently switch to a non-GPU shape.

4. Check supported Kubernetes versions.

    ```bash
    oci ce cluster-options get \
      --cluster-option-id all \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      | jq -r '.data."kubernetes-versions"[]'
    ```

5. Confirm the requested version is listed.

    ```bash
    oci ce cluster-options get \
      --cluster-option-id all \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      | jq -e --arg version "$KUBERNETES_VERSION" '.data."kubernetes-versions"[] | select(. == $version)'
    ```

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 4, 2026
