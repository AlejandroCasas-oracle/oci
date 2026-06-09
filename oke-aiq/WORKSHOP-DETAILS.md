# Workshop Details

Estimated Time: 5 minutes

## Short Description

Deploy NVIDIA AIQ Blueprint on Oracle Kubernetes Engine with an OCI A100 GPU node, NVIDIA NGC secrets, a public frontend endpoint, and practical single-node GPU troubleshooting.

## Long Description

This workshop teaches OCI practitioners how to deploy the NVIDIA AIQ Blueprint on Oracle Kubernetes Engine. Learners create a fresh three-subnet OCI network, provision an enhanced OKE cluster, add a one-node A100 GPU node pool with a matching OKE Gen2 GPU image and 1 TB boot volume, configure NVIDIA NGC and Tavily credentials, deploy the AIQ Helm chart, expose the frontend, validate the application, and clean up the environment safely.

The lab emphasizes the operational details that made the deployment successful: using an OKE Gen2 GPU image by `image_id`, matching the Kubernetes version exactly, validating boot volume expansion, handling the `nvidia.com/gpu=present:NoSchedule` taint on single-node GPU clusters, keeping CoreDNS schedulable, and storing API keys only in Kubernetes secrets.

## Workshop Outline

1. Introduction
2. Lab 1 - Prepare Your Environment
3. Lab 2 - Build the OCI Network
4. Lab 3 - Create the OKE GPU Cluster
5. Lab 4 - Prepare the GPU Node
6. Lab 5 - Deploy NVIDIA AIQ Blueprint
7. Lab 6 - Validate and Clean Up

## Workshop Prerequisites

- OCI tenancy with permission to create VCN, subnet, gateway, route table, security list, OKE, compute, load balancer, and block volume resources.
- GPU quota and regional capacity for an A100-capable OCI GPU shape such as `BM.GPU4.8`.
- OCI CLI configured for the target tenancy profile.
- `kubectl`, Helm 3, and `jq` installed locally.
- NVIDIA NGC API key with access to the AIQ Blueprint images and chart.
- Tavily API key for AIQ web-search integrations.

## Notes

- This workshop uses placeholders such as `<compartment_ocid>`, `<region>`, `<ngc_api_key>`, and `<tavily_api_key>`. Replace them with values from your tenancy.
- Do not store real API keys, private keys, or kubeconfig contents in the workshop repository.
- GPU resources can be expensive. Complete the cleanup lab after testing.
- This workshop is command-driven. Learners create OCI resources with OCI CLI and deploy Kubernetes resources with `kubectl` and Helm.

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 4, 2026
