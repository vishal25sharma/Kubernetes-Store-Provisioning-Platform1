# Kubernetes Store Provisioning Platform

A production-grade platform for dynamically provisioning ecommerce stores on Kubernetes. Currently supports **WooCommerce** with plans for MedusaJS.

## 🎯 Features

- **Dynamic Store Provisioning**: Create WooCommerce stores on-demand via REST API or web dashboard
- **Kubernetes-Native**: Uses official Kubernetes API client for reliable, idempotent operations
- **Complete Isolation**: Each store runs in its own namespace with NetworkPolicies
- **Resource Management**: Automatic ResourceQuotas and LimitRanges per store
- **Persistent Storage**: StatefulSets for databases with persistent volumes
- **Production-Ready**: Helm charts with environment-specific values (local vs production)
- **Security First**: RBAC, NetworkPolicies, no hardcoded secrets
- **Abuse Prevention**: Rate limiting and max stores per user
- **Auto-Cleanup**: Guaranteed resource deletion via namespace cascading

## 📋 Prerequisites

### Local Development
- **Kubernetes cluster**:
  - Docker Desktop with Kubernetes enabled, OR
  - Minikube, OR
  - Kind (Kubernetes in Docker)
- **kubectl** configured to access your cluster
- **Helm 3.x**
- **nginx-ingress controller**
- **Node.js 20+** and **npm** (for local development)
- **Docker** (for building images)

### Production (k3s on VPS)
- k3s installed on your VPS
- kubectl configured
- Helm 3.x
- Domain with DNS configured
- (Optional) cert-manager for TLS

---

## 🚀 Quick Start (Local)

### 1. Install nginx-ingress Controller

```bash
# For Docker Desktop / Minikube
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Wait for it to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### 2. Configure /etc/hosts

Add these entries to `/etc/hosts`:

```
127.0.0.1 dashboard.stores.localhost
127.0.0.1 api.stores.localhost
```

### 3. Build Docker Images

```bash
# Build backend
cd backend
docker build -t localhost:5000/store-platform/backend:latest .
docker push localhost:5000/store-platform/backend:latest

# Build dashboard
cd ../dashboard
docker build -t localhost:5000/store-platform/dashboard:latest .
docker push localhost:5000/store-platform/dashboard:latest
```

**Note**: If using Kind/Minikube, you may need to set up a local registry or use `docker save/load` to transfer images.

### 4. Install Helm Chart

```bash
cd ..
helm install store-platform ./helm/store-platform \
  -f ./helm/store-platform/values-local.yaml \
  --create-namespace
```

### 5. Wait for Pods

```bash
kubectl get pods -n store-platform --watch
```

Wait until all pods are Running (postgres, backend, dashboard).

### 6. Access Dashboard

Open http://dashboard.stores.localhost in your browser.

---

## 🏪 Creating a Store

### Via Dashboard

1. Open http://dashboard.stores.localhost
2. Click **"Create Store"**
3. Fill in:
   - **Store Name**: `my-shop` (lowercase, alphanumeric, hyphens only)
   - **Store Type**: WooCommerce
   - **Created By**: Your name
4. Click **"Create Store"**
5. Wait for status to change from `provisioning` to `ready` (~2-5 minutes)
6. Click the store URL to open your WooCommerce site

### Via API

```bash
curl -X POST http://api.stores.localhost/api/stores \
  -H "Content-Type: application/json" \
  -d '{
    "name": "test-shop",
    "type": "woocommerce",
    "createdBy": "admin"
  }'
```

---

## 🛒 Testing WooCommerce

Once your store is ready:

### 1. Access WordPress Admin

URL: `http://my-shop.stores.localhost/wp-admin`

**Default credentials**: 
- Username: `admin`
- Password: `admin` (set during first install)

### 2. Add a Product

1. Go to **Products → Add New**
2. Enter product name (e.g., "Test Product")
3. Set price (e.g., $9.99)
4. Click **Publish**

### 3. Place an Order

1. Visit store frontend: `http://my-shop.stores.localhost`
2. Add product to cart
3. Go to **Checkout**
4. Fill in billing details
5. Select **Cash on Delivery** payment method
6. Click **Place Order**

### 4. Verify Order

1. Go back to `/wp-admin`
2. Navigate to **WooCommerce → Orders**
3. See your order listed

---

## 🗑️ Deleting a Store

### Via Dashboard

1. Click the **delete icon** on the store card
2. Confirm deletion
3. Wait for store to be removed (~30 seconds to 2 minutes)

### Via API

```bash
curl -X DELETE http://api.stores.localhost/api/stores/{store-id}
```

### Verification

```bash
# Check namespace is deleted
kubectl get namespaces | grep store-

# Check PVCs are deleted
kubectl get pvc --all-namespaces | grep store-
```

All resources (Deployments, Services, Secrets, PVCs, Ingress) are automatically removed via namespace deletion.

---

## ⚙️ Configuration

### Environment Variables (Backend)

Create `backend/.env` from `backend/.env.example`:

```bash
DB_HOST=postgres
DB_PORT=5432
DB_NAME=store_platform
DB_USER=postgres
DB_PASSWORD=<your-password>

PORT=3000
NODE_ENV=development

MAX_STORES_PER_USER=10
MAX_CONCURRENT_PROVISIONING=3
PROVISIONING_TIMEOUT_MS=600000

DOMAIN=stores.localhost
ENVIRONMENT=local
STORAGE_CLASS=hostpath
```

### Helm Values

Key configuration in `values-local.yaml`:

```yaml
storeDefaults:
  maxStoresPerUser: 10
  maxConcurrentProvisioning: 3
  storageClass: "hostpath"
  resourceQuota:
    limits.cpu: "2"
    limits.memory: "2Gi"
```

---

## 🌐 Local vs Production Differences

| Aspect | Local (`values-local.yaml`) | Production (`values-prod.yaml`) |
|--------|----------------------------|--------------------------------|
| **Domain** | `stores.localhost` | `stores.yourdomain.com` |
| **Storage Class** | `hostpath` | `fast-ssd` |
| **Replicas** | 1 (backend, dashboard) | 3 (HA) |
| **TLS** | Disabled | Enabled with cert-manager |
| **Max Stores/User** | 10 | 100 |
| **Concurrency** | 3 | 10 |
| **Resources** | Minimal | Production-grade |

---

## 🔄 Helm Operations

### Install

```bash
helm install store-platform ./helm/store-platform \
  -f ./helm/store-platform/values-local.yaml
```

### Upgrade

```bash
# Make changes to values or code
# Rebuild Docker images

helm upgrade store-platform ./helm/store-platform \
  -f ./helm/store-platform/values-local.yaml
```

### Rollback

```bash
# List releases
helm history store-platform

# Rollback to previous version
helm rollback store-platform

# Rollback to specific revision
helm rollback store-platform 2
```

### Uninstall

```bash
helm uninstall store-platform -n store-platform
```

---

## 🚀 Production Deployment (k3s)

### 1. Prepare VPS with k3s

```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# Copy kubeconfig
sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/config
# Edit server URL to point to your VPS IP
```

### 2. Set Up Docker Registry

```bash
# Example: Use Docker Hub or private registry
docker tag localhost:5000/store-platform/backend:latest \
  registry.yourdomain.com/store-platform/backend:latest

docker push registry.yourdomain.com/store-platform/backend:latest
```

### 3. Configure DNS

Point these domains to your VPS IP:

- `dashboard.stores.yourdomain.com`
- `api.stores.yourdomain.com`
- `*.stores.yourdomain.com` (wildcard for stores)

### 4. Install cert-manager (Optional)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml

# Create ClusterIssuer for Let's Encrypt
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

### 5. Update values-prod.yaml

```yaml
global:
  domain: stores.yourdomain.com

dashboard:
  image:
    repository: registry.yourdomain.com/store-platform/dashboard
    
backend:
  image:
    repository: registry.yourdomain.com/store-platform/backend

ingress:
  tls:
    enabled: true
```

### 6. Deploy

```bash
helm install store-platform ./helm/store-platform \
  -f ./helm/store-platform/values-prod.yaml
```

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
├─────────────────────────────────────────────────────────┤
│  Namespace: store-platform                              │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Dashboard  │  │   Backend    │  │  PostgreSQL  │  │
│  │  (React)    │  │  (Node.js)   │  │ (Metadata)   │  │
│  └─────────────┘  └──────────────┘  └──────────────┘  │
│         │                  │                 │          │
│         └──────────Ingress─┴─────────────────┘          │
└─────────────────────────────────────────────────────────┘
                           │
                    Provisions ↓
┌─────────────────────────────────────────────────────────┐
│  Namespace: store-{uuid}  (Per Store)                   │
│  ┌──────────────┐         ┌─────────────────┐          │
│  │   MariaDB    │ ←───────│   WordPress +   │          │
│  │ (StatefulSet)│         │   WooCommerce   │          │
│  └──────────────┘         └─────────────────┘          │
│         │                          │                    │
│         └──────────Ingress─────────┘                    │
│           (my-shop.stores.localhost)                    │
└─────────────────────────────────────────────────────────┘
```

---

## 🔒 Security

- **RBAC**: Minimal permissions via ServiceAccount + ClusterRole
- **NetworkPolicies**: Default deny, explicit allow rules
- **Secrets**: Generated dynamically, never committed to git
- **Resource Limits**: ResourceQuota and LimitRange per namespace
- **Non-root**: Containers run as non-root where possible
- **Rate Limiting**: Prevents API abuse

---

## 📈 Scaling

### Current Limits

- **Max stores per user**: 10 (local) / 100 (prod)
- **Concurrent provisioning**: 3 (local) / 10 (prod)
- **Backend replicas**: 1 (local) / 3 (prod)

### To Scale to 100s of Stores

1. **Database**: Use PostgreSQL replication or managed DB
2. **Provisioning**: Increase concurrency, add message queue
3. **Storage**: Use cloud storage (EBS, Persistent Disk)
4. **Multi-cluster**: Distribute stores across multiple K8s clusters

See `docs/system-design.md` for detailed scaling architecture.

---

## 🐛 Troubleshooting

### Backend pods not starting

```bash
kubectl logs -n store-platform deployment/backend
kubectl describe pod -n store-platform -l app=backend
```

Common issues:
- Database not ready → Wait for postgres pod
- Secrets missing → Check `kubectl get secrets -n store-platform`

### Store stuck in "provisioning"

```bash
# Check backend logs
kubectl logs -n store-platform deployment/backend

# Check if namespace was created
kubectl get namespace | grep store-

# Check for events
kubectl get events -n store-{uuid}
```

### Ingress not working

```bash
# Check ingress controller
kubectl get pods -n ingress-nginx

# Check ingress resource
kubectl get ingress -n store-platform

# Verify /etc/hosts entries
ping dashboard.stores.localhost
```

---

## 📚 API Documentation

### Endpoints

- `GET /api/stores` - List all stores
- `GET /api/stores/:id` - Get store details
- `POST /api/stores` - Create new store
- `DELETE /api/stores/:id` - Delete store
- `GET /health` - Health check

### Example Responses

**GET /api/stores**
```json
{
  "stores": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "my-shop",
      "type": "woocommerce",
      "namespace": "store-my-shop",
      "status": "ready",
      "url": "http://my-shop.stores.localhost",
      "createdBy": "admin",
      "createdAt": "2024-01-15T10:30:00Z"
    }
  ],
  "count": 1
}
```

---

## 🤝 Contributing

This is a demo project. Key areas for contribution:

1. **MedusaJS Support**: Implement the stubbed engine
2. **Testing**: Add integration tests
3. **Monitoring**: Add Prometheus metrics
4. **UI Enhancements**: Improve dashboard with more features

---

## 📄 License

MIT License - See LICENSE file for details

---

## 🙏 Credits

Built with:
- React + Material UI
- Node.js + TypeScript
- Kubernetes Official Client
- PostgreSQL
- Helm
- WordPress + WooCommerce
