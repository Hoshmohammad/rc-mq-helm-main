# MQ_AKS — IBM MQ on AKS with Helm

Deploy IBM MQ 9.4 as a Native HA container on Azure Kubernetes Service (AKS) using Helm charts and a GitHub Actions CI/CD pipeline.

---

## Project Overview

This repository provides a production-ready Helm chart and GitHub Actions pipeline for deploying [IBM MQ 9.4](https://www.ibm.com/products/mq) to Kubernetes with Native High Availability (Native HA). The container image is sourced from an Azure Container Registry (ACR) where the IBM MQ image has been mirrored — no image is pulled directly from IBM Container Registry and no custom image is built or pushed.

### Features

- IBM MQ 9.4.x LTS with Native HA (3 pods per queue manager: 1 active + 2 replicas)
- Helm chart with per-environment value overrides (QA, Stage, Prod)
- Automated validation (helm lint + template dry-run) triggered manually via `workflow_dispatch`
- Sequential environment promotion: QA → Stage → Prod
- Manual approval gate for Production via GitHub Environment protection rules
- ACR image pull via AKS managed identity (no imagePullSecret required)
- StatefulSet with per-pod dedicated PersistentVolumes (Azure Premium SSD, 200 Gi each)
- MQSC and INI configuration via ConfigMap

---

## Native HA Topology

IBM MQ Native HA uses three pods per queue manager that form a Raft-based replication group.

| Component | Description |
|---|---|
| MQ pods per queue manager | 3 (1 active + 2 standby) |
| Internal replication port | 9414 (TCP, Raft consensus) |
| Storage per pod | 200 Gi dedicated PersistentVolume (ReadWriteOnce) |
| Client connectivity | Kubernetes Service on port 1414 |
| Web Console | Kubernetes Service on port 9443 |
| Metrics endpoint | Kubernetes Service on port 9157 |

At any given time one pod acts as the active queue manager; if it fails a standby automatically becomes active with no data loss. Native HA peer discovery is handled automatically by the IBM MQ container image via the `MQ_NATIVE_HA=true` environment variable together with the headless Kubernetes service — no manual replication address configuration is required.

This deployment supports **two queue managers** (QM1 and QM2) deployed as separate Helm releases, totalling 6 MQ pods and 1.2 TB of storage (6 × 200 Gi).

---

## Project Structure

```
.
├── .github/
│   └── workflows/
│       └── deployment.yml          # CI/CD pipeline
├── helm/
│   └── ibm-mq/
│       ├── Chart.yaml
│       ├── values.yaml             # Base defaults
│       ├── values-qa.yaml          # QA overrides
│       ├── values-stage.yaml       # Stage overrides
│       ├── values-prod.yaml        # Production overrides
│       └── templates/
│           ├── _helpers.tpl
│           ├── statefulset.yaml
│           ├── service.yaml
│           ├── headless-service.yaml
│           ├── configmap.yaml
│           ├── secret.yaml
│           ├── serviceaccount.yaml
│           └── ingress.yaml
└── README.md
```

---

## Prerequisites

| Tool | Version |
|---|---|
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | v1.28+ |
| [Helm 3](https://helm.sh/docs/intro/install/) | v3.x |
| [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli) | Latest |

**AKS Cluster Requirements** (must be provisioned before MQ deployment):

| Parameter | Recommended |
|---|---|
| Node pool OS | Linux |
| Node pool type | Dedicated user node pool for MQ workloads |
| Node vCPU | 4–6 vCPU per node |
| Node memory | 16–64 GB RAM per node |
| Disk type | Premium SSD |
| Minimum nodes | 3 (recommended 6 for better isolation) |
| Container runtime | containerd |
| Container registry | Azure Container Registry (ACR) |

> The AKS platform team must grant ACR pull access via managed identity:
> ```bash
> az aks update --name <AKS_CLUSTER> --resource-group <RG> --attach-acr <ACR_NAME>
> ```
> No imagePullSecret is required when using managed identity.

---

## GitHub Secrets Configuration

Configure the following secrets in your GitHub repository (**Settings → Secrets and variables → Actions**):

| Secret | Description |
|---|---|
| `KUBECONFIG_QA` | Base64-encoded kubeconfig for the QA AKS cluster |
| `KUBECONFIG_STAGE` | Base64-encoded kubeconfig for the Stage AKS cluster |
| `KUBECONFIG_PROD` | Base64-encoded kubeconfig for the Production AKS cluster |
| `ACR_LOGIN_SERVER` | ACR login server hostname, e.g. `myacr.azurecr.io` |
| `MQ_ADMIN_PASSWORD` | IBM MQ administrator password |
| `MQ_APP_PASSWORD` | IBM MQ application password |

### Encoding a kubeconfig as base64

```bash
cat ~/.kube/config | base64 -w 0
```

Paste the output as the value for the corresponding `KUBECONFIG_*` secret.

---

## Running Helm Lint Locally

```bash
# Lint with default values
helm lint ./helm/ibm-mq

# Lint with environment-specific values
helm lint ./helm/ibm-mq -f ./helm/ibm-mq/values-qa.yaml
helm lint ./helm/ibm-mq -f ./helm/ibm-mq/values-stage.yaml
helm lint ./helm/ibm-mq -f ./helm/ibm-mq/values-prod.yaml
```

---

## Manual Deployment

### Deploy QM1 to QA

```bash
kubectl create namespace ibm-mq-qa --dry-run=client -o yaml | kubectl apply -f -
helm upgrade --install ibm-mq-qm1 ./helm/ibm-mq \
  -f ./helm/ibm-mq/values-qa.yaml \
  --namespace ibm-mq-qa \
  --set image.repository=<ACR_LOGIN_SERVER>/ibm-messaging/mq \
  --set mq.qmgrName=QM1 \
  --set mq.adminPassword=<ADMIN_PASSWORD> \
  --set mq.appPassword=<APP_PASSWORD> \
  --wait --timeout 10m
```

### Deploy QM2 to QA

```bash
helm upgrade --install ibm-mq-qm2 ./helm/ibm-mq \
  -f ./helm/ibm-mq/values-qa.yaml \
  --namespace ibm-mq-qa \
  --set image.repository=<ACR_LOGIN_SERVER>/ibm-messaging/mq \
  --set mq.qmgrName=QM2 \
  --set service.ports.mqListener=1415 \
  --set service.ports.webConsole=9444 \
  --set service.ports.metrics=9158 \
  --set mq.adminPassword=<ADMIN_PASSWORD> \
  --set mq.appPassword=<APP_PASSWORD> \
  --wait --timeout 10m
```

### Deploy to Stage

```bash
kubectl create namespace ibm-mq-stage --dry-run=client -o yaml | kubectl apply -f -
helm upgrade --install ibm-mq-qm1 ./helm/ibm-mq \
  -f ./helm/ibm-mq/values-stage.yaml \
  --namespace ibm-mq-stage \
  --set image.repository=<ACR_LOGIN_SERVER>/ibm-messaging/mq \
  --set mq.qmgrName=QM1 \
  --set mq.adminPassword=<ADMIN_PASSWORD> \
  --set mq.appPassword=<APP_PASSWORD> \
  --wait --timeout 10m
```

### Deploy to Production

```bash
kubectl create namespace ibm-mq-prod --dry-run=client -o yaml | kubectl apply -f -
helm upgrade --install ibm-mq-qm1 ./helm/ibm-mq \
  -f ./helm/ibm-mq/values-prod.yaml \
  --namespace ibm-mq-prod \
  --set image.repository=<ACR_LOGIN_SERVER>/ibm-messaging/mq \
  --set mq.qmgrName=QM1 \
  --set mq.adminPassword=<ADMIN_PASSWORD> \
  --set mq.appPassword=<APP_PASSWORD> \
  --wait --timeout 10m
```

### Uninstall

```bash
helm uninstall ibm-mq-qm1 --namespace ibm-mq-prod
helm uninstall ibm-mq-qm2 --namespace ibm-mq-prod
```

---

## Network Ports

| Port | Protocol | Purpose |
|---|---|---|
| 1414 | TCP | MQ client connections (QM1) |
| 1415 | TCP | MQ client connections (QM2 — override via `--set service.ports.mqListener=1415`) |
| 9443 | TCP | MQ Web Console (QM1) |
| 9444 | TCP | MQ Web Console (QM2 — override via `--set service.ports.webConsole=9444`) |
| 9157 | TCP | MQ metrics (QM1) |
| 9158 | TCP | MQ metrics (QM2 — override via `--set service.ports.metrics=9158`) |
| 9414 | TCP | Native HA internal replication (Raft, pod-to-pod only) |

Port 9414 is used internally between the three MQ pods via the headless Kubernetes service. It is **not** exposed outside the cluster.

Each queue manager is deployed as a separate Helm release. The default port values in `values.yaml` apply to QM1. When deploying QM2, pass `--set service.ports.*` overrides to avoid port clashes if both releases share the same namespace.

---

## Configuring MQ Queues via MQSC

MQ queue, channel, and listener configuration is managed via the `mqsc` value in each values file. The content is mounted into the container at `/etc/mqm/config.mqsc` and applied automatically on startup.

To add or modify queues, edit the `mqsc` block in the appropriate values file:

```yaml
# values-qa.yaml
mqsc: |
  DEFINE QLOCAL('MY.NEW.QUEUE') REPLACE DEFPSIST(YES)
  DEFINE CHANNEL('MY.APP.SVRCONN') CHLTYPE(SVRCONN) REPLACE
  SET CHLAUTH('MY.APP.SVRCONN') TYPE(BLOCKUSER) USERLIST('nobody')
```

After editing, redeploy using `helm upgrade`:

```bash
helm upgrade ibm-mq-qm1 ./helm/ibm-mq \
  -f ./helm/ibm-mq/values-qa.yaml \
  --namespace ibm-mq-qa \
  --set image.repository=<ACR_LOGIN_SERVER>/ibm-messaging/mq \
  --set mq.adminPassword=<ADMIN_PASSWORD> \
  --set mq.appPassword=<APP_PASSWORD>
```

### MQSC Reference

| Command | Description |
|---|---|
| `DEFINE QLOCAL(name)` | Define a local queue |
| `DEFINE CHANNEL(name) CHLTYPE(SVRCONN)` | Define a server-connection channel |
| `SET CHLAUTH(name) TYPE(BLOCKUSER)` | Block specific users from a channel |
| `START LISTENER(name)` | Start a TCP listener |
| `DISPLAY QLOCAL(name)` | Display queue attributes |

---

## Environment Configuration Summary

| Setting | QA | Stage | Production |
|---|---|---|---|
| `replicaCount` | 3 | 3 | 3 |
| `resources.limits.cpu` | 2 | 4 | 6 |
| `resources.limits.memory` | 4 Gi | 16 Gi | 64 Gi |
| `resources.requests.cpu` | 1 | 2 | 4 |
| `resources.requests.memory` | 2 Gi | 8 Gi | 16 Gi |
| `persistence.size` | 200 Gi | 200 Gi | 200 Gi |
| `persistence.storageClass` | managed-premium | managed-premium | managed-premium |
| `persistence.accessMode` | ReadWriteOnce | ReadWriteOnce | ReadWriteOnce |

Each queue manager deploys 3 pods, each with its own 200 Gi PersistentVolume. With 2 queue managers per environment, total storage per environment is **1.2 TB** (6 pods × 200 Gi).

---

## Storage Requirements

| Parameter | Value |
|---|---|
| Storage type | Azure Managed Disk |
| Storage class | `managed-premium` (Premium SSD) |
| Size per pod | 200 Gi |
| Access mode | ReadWriteOnce (each pod has its own volume) |
| Pods per queue manager | 3 |
| Queue managers | 2 (QM1, QM2) |
| **Total storage per environment** | **1.2 TB** |

---

## Security Notes

- MQ passwords are stored as Kubernetes Secrets and injected as environment variables
- The container runs as non-root user (UID 1001)
- AKS clusters pull images from ACR via managed identity — no imagePullSecret required
- GitHub environment protection rules enforce manual approval before Production deployments
- Sensitive values (`mq.adminPassword`, `mq.appPassword`) must be passed via `--set` at deploy time and never committed to source control
- TLS encryption is recommended for MQ channels in production environments
- HTTPS certificates should be configured for the MQ Web Console
