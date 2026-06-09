# Deploy NVIDIA AIQ Blueprint

## Introduction

In this lab, you deploy NVIDIA AIQ Blueprint into the OKE cluster. AIQ uses a frontend, backend, and PostgreSQL database. The deployment needs two secrets: an NGC image pull secret for `nvcr.io` and an application credential secret that holds the NVIDIA API key, Tavily API key, and database credentials.

The lab deploys the chart first, then applies the tolerations needed for a single-node GPU cluster.

### Objectives

- Create the `ns-aiq` namespace.
- Create NGC image pull and AIQ application secrets.
- Deploy the NVIDIA AIQ Blueprint Helm chart.
- Patch workloads with GPU-node tolerations.
- Verify frontend, backend, and PostgreSQL rollouts.

Estimated Time: 25 minutes

## Task 1: Create the Namespace

1. Set the namespace.

    ```bash
    export AIQ_NAMESPACE="ns-aiq"
    export AIQ_RELEASE="aiq"
    export AIQ_CHART_URL="https://helm.ngc.nvidia.com/nvidia/blueprint/charts/aiq2-web-2.0.0.tgz"
    ```

2. Create the namespace.

    ```bash
    kubectl create namespace "$AIQ_NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
    ```

## Task 2: Create NGC and AIQ Secrets

1. Create the NGC image pull secret. Use the literal username `$oauthtoken`.

    ```bash
    kubectl create secret docker-registry ngc-secret \
      --namespace "$AIQ_NAMESPACE" \
      --docker-server=nvcr.io \
      --docker-username='$oauthtoken' \
      --docker-password="$NGC_API_KEY" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

2. Choose lab database credentials.

    ```bash
    export AIQ_DB_USER="aiq"
    read -rsp "AIQ database password: " AIQ_DB_PASSWORD
    echo
    export AIQ_DB_PASSWORD
    ```

3. Create the application secret. Do not print the secret values.

    ```bash
    kubectl create secret generic aiq-credentials \
      --namespace "$AIQ_NAMESPACE" \
      --from-literal=NVIDIA_API_KEY="$NGC_API_KEY" \
      --from-literal=TAVILY_API_KEY="$TAVILY_API_KEY" \
      --from-literal=DB_USER_NAME="$AIQ_DB_USER" \
      --from-literal=DB_USER_PASSWORD="$AIQ_DB_PASSWORD" \
      --dry-run=client -o yaml | kubectl apply -f -
    ```

4. Confirm the expected keys exist without displaying values.

    ```bash
    kubectl describe secret aiq-credentials -n "$AIQ_NAMESPACE"
    ```

## Task 3: Deploy the AIQ Helm Chart

1. Render the chart first. Rendering catches authentication and values errors before resources are applied.

    ```bash
    helm template "$AIQ_RELEASE" "$AIQ_CHART_URL" \
      --namespace "$AIQ_NAMESPACE" \
      --username='$oauthtoken' \
      --password "$NGC_API_KEY" \
      --set aiq.apps.backend.imagePullSecrets[0].name=ngc-secret \
      --set aiq.apps.frontend.imagePullSecrets[0].name=ngc-secret \
      > /tmp/aiq-rendered.yaml
    ```

2. Inspect the rendered resource names.

    ```bash
    grep -E '^kind:|^  name:' /tmp/aiq-rendered.yaml | head -80
    ```

3. Install or upgrade the chart.

    ```bash
    helm upgrade --install "$AIQ_RELEASE" "$AIQ_CHART_URL" \
      --namespace "$AIQ_NAMESPACE" \
      --username='$oauthtoken' \
      --password "$NGC_API_KEY" \
      --set aiq.apps.backend.imagePullSecrets[0].name=ngc-secret \
      --set aiq.apps.frontend.imagePullSecrets[0].name=ngc-secret
    ```

4. Check the initial resources.

    ```bash
    kubectl get deploy,svc,pvc,pods -n "$AIQ_NAMESPACE" -o wide
    ```

## Task 4: Add GPU Tolerations to AIQ Workloads

1. Patch the backend deployment.

    ```bash
    kubectl patch deployment aiq-backend -n "$AIQ_NAMESPACE" --type=merge \
      -p='{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"ngc-secret"}],"tolerations":[{"key":"nvidia.com/gpu","operator":"Equal","value":"present","effect":"NoSchedule"}]}}}}'
    ```

2. Patch the frontend deployment.

    ```bash
    kubectl patch deployment aiq-frontend -n "$AIQ_NAMESPACE" --type=merge \
      -p='{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"ngc-secret"}],"tolerations":[{"key":"nvidia.com/gpu","operator":"Equal","value":"present","effect":"NoSchedule"}]}}}}'
    ```

3. Patch the PostgreSQL deployment.

    ```bash
    kubectl patch deployment aiq-postgres -n "$AIQ_NAMESPACE" --type=merge \
      -p='{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"ngc-secret"}],"tolerations":[{"key":"nvidia.com/gpu","operator":"Equal","value":"present","effect":"NoSchedule"}]}}}}'
    ```

4. If old pods remain Pending from before the patch, remove them and let deployments recreate pods.

    ```bash
    kubectl delete pod -n "$AIQ_NAMESPACE" \
      -l app=aiq-backend \
      --field-selector=status.phase=Pending \
      --ignore-not-found=true || true
    ```

## Task 5: Wait for Rollout

1. Wait for PostgreSQL.

    ```bash
    kubectl rollout status deployment/aiq-postgres -n "$AIQ_NAMESPACE" --timeout=300s
    ```

2. Wait for the frontend.

    ```bash
    kubectl rollout status deployment/aiq-frontend -n "$AIQ_NAMESPACE" --timeout=300s
    ```

3. Wait for the backend.

    ```bash
    kubectl rollout status deployment/aiq-backend -n "$AIQ_NAMESPACE" --timeout=600s
    ```

4. If the backend waits in `Init:0/1`, check the init logs.

    ```bash
    export AIQ_BACKEND_POD=$(kubectl get pod -n "$AIQ_NAMESPACE" -l app=aiq-backend -o jsonpath='{.items[0].metadata.name}')
    kubectl logs "$AIQ_BACKEND_POD" -n "$AIQ_NAMESPACE" -c db-init --tail=120
    ```

5. The backend init container should eventually show that PostgreSQL accepts connections and the database initialization script completed. If it repeats `aiq-postgres:5432 - no response`, return to Lab 4 and verify CoreDNS is running.

## Task 6: Confirm AIQ Pods and Services

1. List pods.

    ```bash
    kubectl get pods -n "$AIQ_NAMESPACE" -o wide
    ```

2. List services.

    ```bash
    kubectl get svc -n "$AIQ_NAMESPACE" -o wide
    ```

3. Expected services:

    ```text
    aiq-backend    ClusterIP   port 8000
    aiq-frontend   ClusterIP   port 3000
    aiq-postgres   ClusterIP   port 5432
    ```

## Acknowledgements

* **Author** - Alejandro Casas, Sr. Principal Product Marketing Manager, OCI
* **Last Updated By/Date** - Alejandro Casas, June 4, 2026
