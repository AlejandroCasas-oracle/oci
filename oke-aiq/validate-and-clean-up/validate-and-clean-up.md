# Validate and Clean Up

## Introduction

In this lab, you validate AIQ from inside the cluster and through a public OCI Load Balancer. Then you clean up the Kubernetes and OCI resources in a safe order.

Cleanup order matters. Delete load balancer services and workloads before deleting the node pool, cluster, and network. This avoids orphaned load balancers, block volumes, and VNIC dependencies.

### Objectives

- Run an in-cluster health check.
- Expose the AIQ frontend through an OCI Load Balancer.
- Test the public frontend endpoint.
- Review common troubleshooting signals.
- Delete AIQ, OKE, and network resources safely.

Estimated Time: 15 minutes

## Task 1: Run an Internal Health Check

1. Create a temporary health-check job.

    ```bash
    cat > /tmp/aiq-healthcheck.yaml <<EOF
    apiVersion: batch/v1
    kind: Job
    metadata:
      name: aiq-healthcheck
      namespace: ${AIQ_NAMESPACE}
    spec:
      ttlSecondsAfterFinished: 300
      backoffLimit: 0
      template:
        spec:
          restartPolicy: Never
          tolerations:
          - key: nvidia.com/gpu
            operator: Equal
            value: present
            effect: NoSchedule
          containers:
          - name: curl
            image: curlimages/curl:8.10.1
            command: ["sh", "-c"]
            args:
            - |
              set -e
              echo backend-health
              curl -fsS http://aiq-backend:8000/health
              echo
              echo frontend-root
              curl -fsSI http://aiq-frontend:3000/ | head -20
    EOF

    kubectl apply -f /tmp/aiq-healthcheck.yaml
    ```

2. Wait for the job to complete.

    ```bash
    kubectl wait --for=condition=complete job/aiq-healthcheck \
      -n "$AIQ_NAMESPACE" \
      --timeout=120s
    ```

3. Read the logs.

    ```bash
    kubectl logs job/aiq-healthcheck -n "$AIQ_NAMESPACE"
    ```

4. Expected backend output:

    ```json
    {"status":"healthy"}
    ```

5. Delete the temporary job.

    ```bash
    kubectl delete job aiq-healthcheck -n "$AIQ_NAMESPACE" --ignore-not-found=true
    ```

## Task 2: Expose the AIQ Frontend

1. Create an OCI LoadBalancer service for the frontend.

    ```bash
    cat > /tmp/aiq-frontend-lb.yaml <<EOF
    apiVersion: v1
    kind: Service
    metadata:
      name: aiq-frontend-lb
      namespace: ${AIQ_NAMESPACE}
      annotations:
        oci.oraclecloud.com/load-balancer-type: "lb"
    spec:
      type: LoadBalancer
      selector:
        app: aiq-frontend
      ports:
      - name: http
        protocol: TCP
        port: 80
        targetPort: 3000
    EOF

    kubectl apply -f /tmp/aiq-frontend-lb.yaml
    ```

2. Wait for the external IP.

    ```bash
    watch -n 10 "kubectl get svc aiq-frontend-lb -n $AIQ_NAMESPACE -o wide"
    ```

3. Export the endpoint.

    ```bash
    export AIQ_FRONTEND_IP=$(kubectl get svc aiq-frontend-lb \
      -n "$AIQ_NAMESPACE" \
      -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

    echo "http://$AIQ_FRONTEND_IP"
    ```

## Task 3: Test the Public Frontend

1. Test from your local terminal.

    ```bash
    curl -I "http://$AIQ_FRONTEND_IP"
    ```

2. A healthy frontend returns `HTTP/1.1 200 OK`.

3. Open the frontend in a browser.

    ```text
    http://<aiq_frontend_ip>
    ```

4. If the browser cannot connect, check the load balancer security list and confirm inbound TCP 80 is open to the load balancer subnet.

## Task 4: Troubleshoot Common Issues

1. If the backend waits in init and repeats `aiq-postgres:5432 - no response`, check CoreDNS.

    ```bash
    kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
    kubectl get endpoints aiq-postgres -n "$AIQ_NAMESPACE" -o wide
    ```

2. If CoreDNS is Pending, patch the GPU toleration as shown in Lab 4.

3. If pods are Pending, inspect taints and events.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -i taints
    kubectl describe pod -n "$AIQ_NAMESPACE" -l app=aiq-backend
    ```

4. If image pulls fail, verify the NGC pull secret.

    ```bash
    kubectl describe secret ngc-secret -n "$AIQ_NAMESPACE"
    kubectl get events -n "$AIQ_NAMESPACE" --sort-by=.metadata.creationTimestamp | tail -30
    ```

5. If the node reports disk pressure, verify host root growth before pulling more images.

    ```bash
    kubectl describe node "$NODE_NAME" | grep -i "DiskPressure"
    kubectl describe node "$NODE_NAME" | grep -A8 -i "ephemeral-storage"
    ```

## Task 5: Clean Up Kubernetes Resources

1. Delete the public frontend load balancer service first.

    ```bash
    kubectl delete svc aiq-frontend-lb -n "$AIQ_NAMESPACE" --ignore-not-found=true
    ```

2. Wait until the OCI load balancer is removed. This can take a few minutes.

    ```bash
    kubectl get svc -n "$AIQ_NAMESPACE"
    ```

3. Uninstall the AIQ release.

    ```bash
    helm uninstall "$AIQ_RELEASE" -n "$AIQ_NAMESPACE" || true
    ```

4. Delete remaining AIQ secrets and PVCs.

    ```bash
    kubectl delete secret ngc-secret aiq-credentials -n "$AIQ_NAMESPACE" --ignore-not-found=true
    kubectl delete pvc --all -n "$AIQ_NAMESPACE" --ignore-not-found=true
    ```

5. Delete the namespace.

    ```bash
    kubectl delete namespace "$AIQ_NAMESPACE" --ignore-not-found=true
    ```

## Task 6: Delete OKE and OCI Resources

1. Delete the node pool.

    ```bash
    export DELETE_NODE_POOL_WR=$(oci ce node-pool delete \
      --node-pool-id "$NODE_POOL_OCID" \
      --force \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query '"opc-work-request-id"' \
      --raw-output)

    echo "$DELETE_NODE_POOL_WR"
    ```

2. Poll the node pool deletion work request until it succeeds.

    ```bash
    watch -n 30 "oci ce work-request get \
      --work-request-id $DELETE_NODE_POOL_WR \
      --profile $OCI_CLI_PROFILE \
      --region $REGION \
      --query 'data.status' \
      --raw-output"
    ```

3. Delete the cluster.

    ```bash
    export DELETE_CLUSTER_WR=$(oci ce cluster delete \
      --cluster-id "$CLUSTER_OCID" \
      --force \
      --profile "$OCI_CLI_PROFILE" \
      --region "$REGION" \
      --query '"opc-work-request-id"' \
      --raw-output)

    echo "$DELETE_CLUSTER_WR"
    ```

4. Poll cluster deletion until it succeeds.

    ```bash
    watch -n 30 "oci ce work-request get \
      --work-request-id $DELETE_CLUSTER_WR \
      --profile $OCI_CLI_PROFILE \
      --region $REGION \
      --query 'data.status' \
      --raw-output"
    ```

5. Delete subnets.

    ```bash
    oci network subnet delete --subnet-id "$API_SUBNET_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    oci network subnet delete --subnet-id "$WORKER_SUBNET_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    oci network subnet delete --subnet-id "$LB_SUBNET_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    ```

6. Delete route tables and security lists.

    ```bash
    oci network route-table delete --rt-id "$PUBLIC_RT_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    oci network route-table delete --rt-id "$PRIVATE_RT_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"

    oci network security-list delete --security-list-id "$API_SL_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    oci network security-list delete --security-list-id "$WORKER_SL_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    oci network security-list delete --security-list-id "$LB_SL_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    ```

7. Delete gateways and the VCN.

    ```bash
    oci network internet-gateway delete --ig-id "$IGW_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    oci network nat-gateway delete --nat-gateway-id "$NAT_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    oci network service-gateway delete --service-gateway-id "$SGW_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    oci network vcn delete --vcn-id "$VCN_OCID" --force --profile "$OCI_CLI_PROFILE" --region "$REGION"
    ```

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 4, 2026
