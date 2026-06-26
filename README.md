# Resyze — Kubernetes Deployment

Deploys all Resyze services to a k3s cluster running on 3 bare-metal Hetzner servers.

## Directory Structure

Each app and infrastructure component lives in its own folder so ArgoCD can target it independently as an Application.

```
deployment/k8s/
├── apps/
│   ├── resyze-api/
│   │   └── resyze-api.yaml           # Deployment + Service (port 8081)
│   ├── resyze-analytics/
│   │   └── resyze-analytics.yaml     # Deployment + Service (port 8082)
│   ├── resyze-mailer/
│   │   └── resyze-mailer.yaml        # Deployment + Service (port 8083)
│   └── resyze-app/
│       └── resyze-app.yaml           # Deployment + Service (nginx, port 80)
├── infrastructure/
│   ├── namespace/
│   │   └── namespace.yaml            # resyze namespace
│   ├── postgres/
│   │   └── postgres.yaml             # StatefulSet + PVC + Service
│   ├── nats/
│   │   └── nats.yaml                 # Deployment + Service (JetStream enabled)
│   ├── elasticsearch/
│   │   └── elasticsearch.yaml        # StatefulSet + PVC + Service
│   └── ingress/
│       └── ingress.yaml              # Traefik Ingress + cert-manager ClusterIssuer
├── secrets/
│   └── secrets.yaml                  # Secret templates — fill in before applying
└── README.md
```

## Architecture

```
Internet
   │
   ▼
Traefik (k3s built-in ingress, ports 80/443)
   │
   ├── /api/*         → resyze-api:8081
   ├── /cdn/*         → resyze-api:8081
   ├── /analytics/*   → resyze-analytics:8082
   └── /*             → resyze-app:80 (SPA)

Internal cluster communication:
  resyze-api      → postgres:5432
  resyze-api      → nats:4222
  resyze-analytics → nats:4222
  resyze-analytics → elasticsearch:9200
  resyze-mailer   → nats:4222
```

## Prerequisites

### 1. k3s Cluster

Install k3s on each of the 3 Hetzner nodes. On the first node (control plane):

```bash
curl -sfL https://get.k3s.io | sh -
# Get the node token for joining workers
cat /var/lib/rancher/k3s/server/node-token
```

On each of the 2 worker nodes:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<control-plane-ip>:6443 K3S_TOKEN=<node-token> sh -
```

Copy the kubeconfig from the control plane to your local machine:

```bash
scp root@<control-plane-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/config
# Replace 127.0.0.1 with the control plane public IP
sed -i 's/127.0.0.1/<control-plane-ip>/' ~/.kube/config
```

### 2. cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
# Wait for cert-manager pods to be ready
kubectl wait --namespace cert-manager --for=condition=ready pod --selector=app.kubernetes.io/instance=cert-manager --timeout=120s
```

### 3. DNS

Point an A record for `resyze.aoilabs.net` to the public IP of your Hetzner nodes (or a floating IP / load balancer in front of them). Traefik listens on ports 80 and 443 on every node.

### 4. Private Registry Secret

Create the image pull secret for `docker.aoilabs.net`:

```bash
kubectl create secret docker-registry registry-secret \
  --namespace resyze \
  --docker-server=docker.aoilabs.net \
  --docker-username=<username> \
  --docker-password=<token>
```

## Deployment

### Step 1 — Namespace

```bash
kubectl apply -f deployment/k8s/infrastructure/namespace/
```

### Step 2 — Secrets

Fill in all `<base64>` placeholders in `secrets/secrets.yaml`. Encode values with:

```bash
echo -n "my-value" | base64
```

Then apply:

```bash
kubectl apply -f deployment/k8s/secrets/secrets.yaml
```

> For production, consider [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) or [External Secrets Operator](https://external-secrets.io) instead of committing encoded values.

### Step 3 — Infrastructure

Apply in order to respect dependencies:

```bash
kubectl apply -f deployment/k8s/infrastructure/postgres/
kubectl apply -f deployment/k8s/infrastructure/nats/
kubectl apply -f deployment/k8s/infrastructure/elasticsearch/
```

Wait for infrastructure pods to be ready before deploying apps:

```bash
kubectl wait --namespace resyze --for=condition=ready pod --selector=app=postgres --timeout=120s
kubectl wait --namespace resyze --for=condition=ready pod --selector=app=nats --timeout=60s
kubectl wait --namespace resyze --for=condition=ready pod --selector=app=elasticsearch --timeout=180s
```

### Step 4 — Apps

```bash
kubectl apply -f deployment/k8s/apps/resyze-api/
kubectl apply -f deployment/k8s/apps/resyze-analytics/
kubectl apply -f deployment/k8s/apps/resyze-mailer/
kubectl apply -f deployment/k8s/apps/resyze-app/
```

### Step 5 — Ingress

```bash
kubectl apply -f deployment/k8s/infrastructure/ingress/
```

cert-manager will automatically provision a Let's Encrypt TLS certificate. Check the status:

```bash
kubectl get certificate -n resyze
kubectl describe certificate resyze-tls -n resyze
```

## Database Migrations

Migrations are not run automatically. After deploying resyze-api, run migrations manually from the `resyze-api/migrations/` directory using a temporary pod:

```bash
kubectl run -it --rm migrate \
  --namespace resyze \
  --image=postgres:16-alpine \
  --env="PGPASSWORD=<password>" \
  -- psql -h postgres -U <user> -d resyze -f /migrations/<file>.sql
```

## Image Tags

All app Deployments use `:latest` by default. For production, pin to a specific image tag and update via:

```bash
kubectl set image deployment/resyze-api resyze-api=docker.aoilabs.net/resyze/resyze-api:<tag> -n resyze
kubectl set image deployment/resyze-analytics resyze-analytics=docker.aoilabs.net/resyze/resyze-analytics:<tag> -n resyze
kubectl set image deployment/resyze-mailer resyze-mailer=docker.aoilabs.net/resyze/resyze-mailer:<tag> -n resyze
kubectl set image deployment/resyze-app resyze-app=ghcr.io/aifaniyi/resyze-app:<tag> -n resyze
```

## Useful Commands

```bash
# Check pod status
kubectl get pods -n resyze

# Stream logs for a service
kubectl logs -f -l app=resyze-api -n resyze

# Describe a pod to see events / errors
kubectl describe pod <pod-name> -n resyze

# Open a shell in a running pod
kubectl exec -it <pod-name> -n resyze -- sh

# Check ingress and TLS certificate
kubectl get ingress -n resyze
kubectl get certificate -n resyze
```

## Storage

| Component     | PVC Name           | Size  | Notes                              |
|---------------|--------------------|-------|------------------------------------|
| Postgres      | postgres-pvc       | 10 Gi | local-path provisioner             |
| Elasticsearch | elasticsearch-pvc  | 20 Gi | local-path provisioner             |
| NATS          | —                  | —     | emptyDir (replace with PVC for HA) |

The default k3s `local-path` provisioner binds PVCs to a single node. If you need PVCs to survive node failure, install [Longhorn](https://longhorn.io) and change `storageClassName: local-path` to `storageClassName: longhorn` in the PVC specs.
