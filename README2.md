# Kubernetes Monitoring Using Prometheus & Grafana

This README describes how to run Prometheus and Grafana on a local Kubernetes cluster using **Minikube**, **Helm**, and **kubectl**. Commands below are written for **Windows PowerShell** where applicable.

## Prerequisites

Install and verify the following tools:

- Docker Desktop (running)
- Minikube
- kubectl
- Helm

Check installations:

```powershell
docker --version
minikube version
kubectl version --client
helm version
```

## 1. Start Minikube

Start a local Kubernetes cluster using the Docker driver and allocate 4 GB of memory:

```powershell
minikube start --memory=4098 --driver=docker
```

Check the cluster:

```powershell
kubectl get nodes
```

## 2. Install Prometheus with Helm

Add the Prometheus Community chart repository and install Prometheus:

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus
```

Check the deployed resources:

```powershell
kubectl get pods
kubectl get svc
```

Wait until the Prometheus pods are ready before proceeding.

## 3. Expose Prometheus

Expose the Prometheus service through a NodePort service:

```powershell
kubectl expose service prometheus-kube-prometheus-prometheus `
  --type=NodePort `
  --target-port=9090 `
  --name=prometheus-server-ext
```

Verify the service and open it through Minikube:

```powershell
kubectl get services prometheus-server-ext
minikube service prometheus-server-ext
```

The `minikube service` command normally opens the service URL in your default browser. If it does not, use the URL printed in the terminal.

## 4. Install Grafana with Helm

Add the Grafana chart repository, refresh chart metadata, and install Grafana:

```powershell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm repo list
helm install grafana grafana/grafana
```

Check the pods and services:

```powershell
kubectl get pods
kubectl get svc
```

Wait until the Grafana pod is ready.

## 5. Retrieve Grafana Login Credentials

### Get the admin password

Run in **PowerShell**:

```powershell
$encoded = kubectl get secret grafana -n default -o jsonpath='{.data.admin-password}'
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```

### Get the admin username

```powershell
kubectl get secret grafana -n default -o jsonpath='{.data.admin-user}' |
    ForEach-Object { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

These commands decode the credentials stored in the Kubernetes Secret. Treat the output as sensitive and do not commit credentials to source control.

## 6. Expose Grafana

Create a NodePort service for Grafana:

```powershell
kubectl expose service grafana `
  --type=NodePort `
  --target-port=3000 `
  --name=grafana-ext
```

Verify it and open the service:

```powershell
kubectl get svc
minikube service grafana-ext
```

Log in with the username and password retrieved in the previous section.

## 7. Reset the Grafana Admin Password (Optional)

To reset the admin password to the example value below:

```powershell
kubectl exec -n default deploy/grafana -- grafana cli admin reset-admin-password 'Grafana@12345'
```

**Security note:** `Grafana@12345` is a sample password from this guide. Use a unique, strong password for your environment, and do not use this example password in production. After resetting the password, use the new password to log in.

## 8. Configure Prometheus as a Grafana Data Source

1. Open Grafana using the URL printed by `minikube service grafana-ext`.
2. Go to **Connections → Data sources** (the menu wording may vary by Grafana version).
3. Select **Add data source → Prometheus**.
4. For the URL, use the in-cluster Prometheus service address:

   ```text
   http://prometheus-kube-prometheus-prometheus:9090
   ```

5. Select **Save & test**.

Because Grafana and Prometheus are installed in the same Kubernetes namespace (`default`), Kubernetes service DNS should resolve this service name from Grafana.

## 9. Verify Monitoring

In Grafana, open **Explore**, select the Prometheus data source, and run a query such as:

```promql
up
```

This returns the availability status of targets Prometheus is scraping. A value of `1` indicates a target is up; `0` indicates it is down.

You can also open the Prometheus UI and enter the same query.

## Troubleshooting

- **Helm repository command fails:** Ensure the repository URL has no spaces. Correct command:
  ```powershell
  helm repo add grafana https://grafana.github.io/helm-charts
  ```
- **Pods are pending or restarting:** Check `kubectl get pods` and inspect a pod with `kubectl describe pod <pod-name>`.
- **Service does not open:** Run `minikube status`, confirm the cluster is running, and retry `minikube service <service-name>`.
- **Grafana cannot connect to Prometheus:** Confirm both services exist with `kubectl get svc`, verify the data-source URL, and check that the Prometheus pods are ready.
- **Grafana deployment name differs:** Check `kubectl get deployments`. The password-reset command assumes the deployment is named `grafana`, as in this guide.

## Cleanup (Optional)

Remove the Helm releases and the Minikube cluster when finished:

```powershell
helm uninstall grafana
helm uninstall prometheus
minikube delete
```

> **Note:** These commands remove the installed releases and local cluster resources. Back up any configuration or dashboards you need before cleanup.
