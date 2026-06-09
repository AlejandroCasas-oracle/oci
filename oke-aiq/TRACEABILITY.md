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
| OCI OKE and networking command patterns from the existing `livelabs/nim-on-oke` workshop | VCN, subnet, OKE, node pool, kubeconfig, validation, and cleanup flow |
| Local `oke-nvidia` skill notes | GPU shape guidance, OKE Gen2 GPU image lessons, 1 TB boot volume requirement, GPU taint behavior, CoreDNS troubleshooting |
| Field deployment on OCI, San Jose, June 2026 | AIQ chart deployment sequence, `ns-aiq` namespace, `ngc-secret`, `aiq-credentials`, CoreDNS toleration fix, AIQ health validation |
| NVIDIA NGC AIQ Blueprint chart path | Helm chart URL and AIQ deployment structure |

## Field Lessons Included

- Use an OKE `ENHANCED_CLUSTER`.
- Use a fresh three-subnet network: API, workers, and load balancers.
- Use NAT gateway egress from private worker subnets so nodes can pull from `nvcr.io`.
- Use an OKE `Gen2-GPU` image OCID with `image_id`, not a generic compute image.
- Match the Kubernetes version exactly between cluster and node image.
- Use a 1 TB boot volume for GPU nodes.
- Verify Kubernetes sees the expanded root filesystem; repair host LVM if it reports about 37 GiB.
- On a single-node GPU cluster, patch CoreDNS and DNS autoscaler with the GPU toleration or remove the GPU taint for lab-only environments.
- Store NGC and Tavily credentials only in Kubernetes secrets.
- Patch AIQ backend, frontend, and PostgreSQL deployments with the GPU toleration when the GPU taint is present.
- Validate backend health from inside the cluster before exposing the frontend.
- Delete LoadBalancer services before deleting node pools, clusters, and VCN resources.

## Placeholders That Must Stay Generic

- `<oci_profile>`
- `<region>`
- `<compartment_ocid>`
- `<availability_domain>`
- `<node_image_ocid>`
- `<ngc_api_key>`
- `<tavily_api_key>`

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 4, 2026
