## Homelab Kubernetes Migration
Migrating Docker Compose services to a Kubernetes cluster for better resilience, scalability, and operational experience.




### 🎯 Project Goals
Migrate existing Docker Compose services to Kubernetes

Learn Kubernetes operations in a production-like environment

Implement GitOps workflow for infrastructure as code

Achieve high availability and self-healing capabilities


### 🏗️ Architecture
### Cluster Setup
* Master Node: Old Linux laptop (x86_64)
* Worker Node: Raspberry Pi 4 (ARM64)
* Network: Multi-architecture K3s cluster with Longhorn storage




### Services to Migrate

| Service             | Current Docker Setup           | Target K8s Setup                     |
| ------------------- | ------------------------------ | ------------------------------------ |
| Grafana             | Container with Traefik labels  | Deployment + Ingress + PVC           |
| Home Assistant      | Container with host networking | DaemonSet with `hostNetwork`         |
| Nextcloud + MariaDB | Linked containers              | StatefulSet + Headless Service       |
| Jellyfin            | Container with volumes         | Deployment with `nodeSelector` (ARM) |
| WireGuard           | Container with host networking | DaemonSet with privileged access     |
| Vault               | Development container          | StatefulSet with production config   |
| Heimdall            | Container with Traefik         | Deployment + Ingress                 |
| Custom Apps         | Docker Compose services        | Deployments + Services               |

### 📁 Project Structure

homelab-k8s/
├── clusters/
│   └── production/
│       ├── flux-system/          # FluxCD bootstrap manifests
│       ├── infrastructure/       # Traefik, Longhorn, cert-manager
│       └── kustomization.yaml    # Main cluster kustomization
├── apps/
│   ├── monitoring/              # Grafana, Prometheus, Loki
│   │   ├── grafana/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── ingress.yaml
│   │   │   └── kustomization.yaml
│   │   └── kustomization.yaml
│   ├── storage/                 # Nextcloud, MariaDB
│   ├── media/                   # Jellyfin, *Arr suite
│   ├── home-automation/         # Home Assistant
│   ├── networking/              # Traefik, WireGuard
│   ├── security/                # Vault, cert-manager
│   └── tools/                   # Heimdall, custom apps
├── base/                        # Common configurations
├── overlays/                    # Environment-specific configs
├── scripts/                     # Setup and maintenance scripts
└── README.md                    # This file

### 🚀 Quick Start
1. Prerequisites

* Kubernetes cluster (K3s recommended)
* 
* kubectl configured to access the cluster
* 
* Helm v3+
* 
* Git

2. Cluster Setup (K3s)

Master node (laptop):

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --disable traefik \
  --node-taint CriticalAddonsOnly=true:NoExecute
```

Worker node (Raspberry Pi):
```bash
# Get token from master:
# sudo cat /var/lib/rancher/k3s/server/node-token

curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 \
  K3S_TOKEN=<TOKEN> sh -

```

### 3. Infrastructure setup

# Enable Longhorn (storage)
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml

# Install Traefik
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik -n traefik --create-namespace \
  -f infrastructure/traefik-values.yaml

# Create namespaces
kubectl create ns monitoring
kubectl create ns storage
kubectl create ns media
kubectl create ns home-automation
kubectl create ns security
kubectl create ns tools
4. Deploy Applications
# Deploy all applications
kubectl apply -k apps/monitoring/
kubectl apply -k apps/storage/
kubectl apply -k apps/media/

# Or deploy individually
kubectl apply -f apps/monitoring/grafana/
🔧 Configuration
Environment Variables
Create a .env file in the project root:

# Domain configuration
DOMAIN=instaworks.ng
CLOUDFLARE_API_EMAIL=your-email@example.com
CLOUDFLARE_API_KEY=your-api-key

# Database passwords (generate with: openssl rand -base64 32)
MYSQL_ROOT_PASSWORD=generated-password
MYSQL_PASSWORD=generated-password

# Storage settings
STORAGE_CLASS=longhorn
Ingress Configuration
All ingress resources use Traefik with TLS termination:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: service-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  rules:
    - host: service.instaworks.ng
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service
                port:
                  number: 80
  tls:
    - hosts:
        - service.instaworks.ng
      secretName: service-tls
Storage Configuration
Using Longhorn for persistent volumes:

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  storageClassName: longhorn
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
🤖 GitOps with FluxCD
Bootstrap Flux
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux to your repository
flux bootstrap github \
  --owner=your-username \
  --repository=homelab-k8s \
  --branch=main \
  --path=./clusters/production \
  --personal
Add Applications
# Add Grafana from Helm repository
flux create source helm grafana \
  --url=https://grafana.github.io/helm-charts \
  --interval=10m

flux create helmrelease grafana \
  --source=HelmRepository/grafana \
  --chart=grafana \
  --interval=10m \
  --target-namespace=monitoring \
  --values=apps/monitoring/grafana/values.yaml
# Or add from local manifests
flux create kustomization apps \
  --source=GitRepository/homelab-k8s \
  --path="./apps" \
  --prune=true \
  --interval=5m
📊 Monitoring
Access Dashboards
Grafana: https://grafana.instaworks.ng (admin/dugneonadmin)

Traefik: https://traefik.instaworks.ng

Longhorn: https://longhorn.instaworks.ng

Kubernetes Dashboard:

kubectl proxy
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

Alerts
Slack / Discord webhooks

Email notifications

Resource threshold alerts (CPU, memory, disk)

🔒 Security
Secrets Management
kubectl create secret generic cloudflare-api \
  --from-literal=email=$CLOUDFLARE_API_EMAIL \
  --from-literal=api-key=$CLOUDFLARE_API_KEY \
  -n traefik
# SealedSecrets (Git-safe secrets)
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.1/controller.yaml
Network Policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: grafana-allow-ingress
spec:
  podSelector:
    matchLabels:
      app: grafana
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 3000
🚨 Troubleshooting
Common Issues
Pods stuck in Pending:

kubectl describe pod <pod-name>
kubectl get events --sort-by='.lastTimestamp'
PVC not binding:

kubectl get pvc
kubectl get pv
kubectl describe pvc <pvc-name>
Ingress not routing:

kubectl describe ingress <ingress-name>
kubectl logs -l app.kubernetes.io/name=traefik -n traefik
Mixed ARM/x86 images:

kubectl get nodes -o wide
nodeSelector:
  kubernetes.io/arch: arm64
Useful Commands
./scripts/status.sh
./scripts/backup.sh
./scripts/restore.sh <backup-file>

kubectl rollout restart deployment --all
kubectl get pods -w

kubectl run -it --rm debug \
  --image=nicolaka/netshoot -- /bin/bash
📈 Performance Optimization
Resource Requests / Limits
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
Node Affinity
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/arch
              operator: In
              values:
                - arm64
🔄 Update Process
kubectl apply -f updated-manifest.yaml
kubectl rollout status deployment/<deployment-name>
Rollback:

kubectl rollout undo deployment/<deployment-name>
kubectl rollout history deployment/<deployment-name>
📚 Learning Resources
K3s Documentation

Kubernetes Documentation

FluxCD Documentation

Traefik Documentation

Longhorn Documentation

🤝 Contributing
Fork the repository

Create a feature branch

Test changes in your homelab

Submit a pull request with documentation

📄 License
MIT License — see LICENSE for details.

🎯 Success Criteria
K3s cluster running on both nodes

All Docker Compose services migrated

Persistent storage working with Longhorn

Ingress routing with TLS certificates

Monitoring stack operational

GitOps workflow established

Backup & recovery tested

Documentation complete

Happy Homelabbing! 🚀
Break things. Fix them. Learn fast. Your homelab is the perfect sandbox.