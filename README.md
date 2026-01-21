# CloudMemo - Kubernetes Multi-Tenant Deployment

## Overview

This project demonstrates a **multi-tenant Kubernetes deployment** of the CloudMemo application using:

- **Docker** for containerization and image management
- **Kubernetes** for orchestration with namespace isolation
- **Network Policies** for security and inter-namespace communication control
- **Redis** as a stateful service per namespace
- **Ansible** for infrastructure automation (optional)

The deployment includes two separate namespaces (`team-blue` and `team-green`), each with isolated CloudMemo and Redis instances, demonstrating proper multi-tenancy practices and network security policies.

---

## Architecture

### Project Structure

```
cloudmemo-k8s/
├── Dockerfile              # Docker image build for CloudMemo
├── docker-compose.yaml    # Local testing with Docker Compose
├── README.md               # This file
├── kubernetes/
│  ├── namespaces.yaml      # Namespace definitions (team-blue, team-green)
│  ├── cloudmemo.yaml      # CloudMemo Deployment & Service
│  ├── redis.yaml           # Redis StatefulSet & Service
│  ├── ingress.yaml         # Ingress routes (blue/green)
│  ├── network-policies/
│  └──    ├── default-deny.yaml
│     └── allow-app-redis.yaml
└── ansible/                # Automation playbooks (optional)
   └── deploy.yaml         # Kubernetes deployment automation
```

---

## Prerequisites

- **Docker** installed (for local testing)
- **Kubernetes** cluster (v1.20+)
- **kubectl** configured to access your cluster
- **Calico** CNI plugin installed (for NetworkPolicies)
- **Docker Hub account** (for pushing images)

---

## Step 1: Build and Test Locally with Docker

### Build the CloudMemo Docker image

```bash
docker build -t flitzouille/cloudmemo:1.0 .
```

### Test with Docker Compose

```bash
docker-compose up -d
```

This will start:
- **CloudMemo** on `http://localhost:5000`
- **Redis** on `localhost:6379`

Verify it works:

```bash
curl http://localhost:5000/
```

### Push image to Docker Hub

```bash
docker login
docker push flitzouille/cloudmemo:1.0
```

---

## Step 2: Deploy to Kubernetes

### 1. Create namespaces

```bash
kubectl apply -f kubernetes/namespaces.yaml
```

### 2. Deploy Redis in both namespaces

```bash
kubectl apply -n team-blue  -f kubernetes/redis.yaml
kubectl apply -n team-green -f kubernetes/redis.yaml
```

Verify Redis is running:

```bash
kubectl get pods -n team-blue
kubectl get pods -n team-green
```

### 3. Deploy CloudMemo in both namespaces

```bash
kubectl apply -n team-blue  -f kubernetes/cloudmemo.yaml
kubectl apply -n team-green -f kubernetes/cloudmemo.yaml
```

Verify deployment:

```bash
kubectl get all -n team-blue
kubectl get all -n team-green
```

### 4. Deploy Ingress (optional)

```bash
kubectl apply -f kubernetes/ingress.yaml
```

This creates:
- `blue.lucas.tp.kaas-mvp-ts.fr` → team-blue CloudMemo
- `green.lucas.tp.kaas-mvp-ts.fr` → team-green CloudMemo

---

## Step 3: Apply Network Policies

Network Policies enforce multi-tenant isolation:

### Default Deny (blocks all ingress traffic)

```bash
kubectl apply -n team-blue  -f kubernetes/network-policies/default-deny.yaml
kubectl apply -n team-green -f kubernetes/network-policies/default-deny.yaml
```

### Allow App → Redis (port 6379, same namespace)

```bash
kubectl apply -n team-blue  -f kubernetes/network-policies/allow-app-redis.yaml
kubectl apply -n team-green -f kubernetes/network-policies/allow-app-redis.yaml
```

**Result**: CloudMemo can only access Redis in its own namespace. Cross-namespace access is blocked.

---

## Testing

### Access the applications locally

Use `kubectl port-forward` for local access:

```bash
# Terminal 1: team-blue
kubectl port-forward -n team-blue deploy/cloudmemo 5001:5000

# Terminal 2: team-green
kubectl port-forward -n team-green deploy/cloudmemo 5002:5000
```

Access from your PC:
- **team-blue**: `http://localhost:5001`
- **team-green**: `http://localhost:5002`

### Test data isolation

1. Create a memo in `http://localhost:5001` (team-blue)
2. Create a different memo in `http://localhost:5002` (team-green)
3. Verify that each namespace only sees its own data

### Test the "hack" scenario

Prove that NetworkPolicies prevent cross-tenant access:

```bash
# Modify team-blue CloudMemo to point to team-green Redis
kubectl edit deploy cloudmemo -n team-blue
```

Change `REDIS_HOST` from `redis` to `redis.team-green.svc.cluster.local`

Expected result:
- CloudMemo blue will fail to connect (connection timeout / DNS error)
- NetworkPolicy blocks the cross-namespace traffic
- Restore `REDIS_HOST` to `redis` to fix it

---

## Environment Variables

### CloudMemo

| Variable | Default | Description |
|----------|---------|-------------|
| `REDIS_HOST` | `redis` | Redis hostname (same namespace) |
| `REDIS_PORT` | `6379` | Redis port |
| `FLASK_ENV` | `production` | Flask environment |

### Docker Compose Override

Edit `docker-compose.yaml` to customize ports, Redis settings, etc.

---

## Troubleshooting

### Pods stuck in `ContainerCreating`

**Cause**: CNI plugins not found  
**Solution**: Ensure `/usr/lib/cni` contains the Calico binaries:

```bash
ln -s /opt/cni/bin/* /usr/lib/cni/
kubectl restart kubelet
```

### Redis PVC in `Pending`

**Cause**: No StorageClass / PV available  
**Solution**: Use `emptyDir` volume instead of `volumeClaimTemplates` (data lost on restart)

### CloudMemo can't reach Redis

**Cause 1**: NetworkPolicy blocking traffic  
**Solution**: Verify `allow-app-redis.yaml` is applied

**Cause 2**: Wrong DNS name  
**Solution**: Verify pod is using `REDIS_HOST=redis` (intra-namespace) or `redis.team-green.svc.cluster.local` (inter-namespace)

### Port-forward not accessible from Windows PC

**Cause**: Port-forward only listens on 127.0.0.1  
**Solution**: Use SSH tunnel from Windows:

```powershell
ssh -L 5001:127.0.0.1:5001 -L 5002:127.0.0.1:5002 lucas@192.168.20.150
```

Then access `http://localhost:5001` and `http://localhost:5002` on your PC.

---

## Validation Checklist

- [ ] Docker image builds and runs locally
- [ ] Both namespaces created in Kubernetes
- [ ] CloudMemo pods running in both namespaces
- [ ] Redis pods running in both namespaces
- [ ] CloudMemo blue accessible and functional
- [ ] CloudMemo green accessible and functional
- [ ] Data isolation confirmed (different memos in each namespace)
- [ ] NetworkPolicies applied successfully
- [ ] Default-deny policy blocks unknown traffic
- [ ] App-to-Redis policy allows intra-namespace access
- [ ] Cross-namespace access attempt fails as expected
- [ ] Restoring correct `REDIS_HOST` fixes the app

---

## Key Concepts

### Multi-Tenancy

Each namespace (`team-blue`, `team-green`) is a separate tenant with:
- Isolated pods
- Isolated services
- Isolated storage (Redis)
- No data leakage between tenants

### Network Policies

**Default Deny**: All ingress traffic to all pods is blocked by default.  
**Explicit Allow**: Only specified traffic (app → Redis on port 6379) is permitted.  
**Same Namespace Only**: Cross-namespace communication is blocked.

### Kubernetes Objects Used

- **Namespace**: Logical cluster partitioning
- **Deployment**: Manages CloudMemo replicas
- **StatefulSet**: Manages Redis with stable identity
- **Service**: ClusterIP for inter-pod communication
- **NetworkPolicy**: Ingress traffic control

---

## Author

**Lucas LESENS** - IT Student at CEFAP  
Project: Kubernetes Multi-Tenant Infrastructure (TP)

---

## License

None specified (for educational purposes).
