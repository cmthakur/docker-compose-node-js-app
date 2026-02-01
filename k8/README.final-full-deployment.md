# Kubernetes manifest: final-full-deployment

This README documents the single-file Kubernetes manifest in [final-full-deployment.yml](final-full-deployment.yml). It deploys a Redis data tier, an application tier, and a Horizontal Pod Autoscaler in the `deployment` namespace.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    deployment namespace                         │
│                                                                 │
│   ┌───────────────┐         ┌───────────────────────────────┐   │
│   │  data-tier    │         │     app-tier Deployment       │   │
│   │  (Pod)        │◄────────│     (5 replicas, HPA 1-5)     │   │
│   │  Redis:6379   │         │     nginx:8080                │   │
│   └───────────────┘         └───────────────────────────────┘   │
│         ▲                              ▲                        │
│         │                              │                        │
│   ┌─────┴─────┐                  ┌─────┴─────┐                  │
│   │ Service   │                  │ Service   │                  │
│   │ ClusterIP │                  │ NodePort  │ ◄── External     │
│   │ :6379     │                  │ :8080     │     Traffic      │
│   └───────────┘                  └───────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

## Resources Created

| Kind | Name | Type/Details |
|------|------|--------------|
| Namespace | `deployment` | Isolated namespace for the app |
| Service | `data-tier` | ClusterIP, port 6379 (Redis) |
| Pod | `data-tier` | Redis container with probes |
| Service | `app-tier` | NodePort, port 8080 |
| Deployment | `app-tier` | 5 replicas, nginx-based server |
| HPA | `app-tier` | min 1, max 5, target 70% CPU |

## Validate the Manifest

**Client-side validation:**
```bash
kubectl apply -f k8/final-full-deployment.yml --dry-run=client
```

**Server-side validation (requires cluster access):**
```bash
kubectl apply -f k8/final-full-deployment.yml --dry-run=server
```

**Diff against the cluster:**
```bash
kubectl diff -f k8/final-full-deployment.yml
```

## Deploy

```bash
kubectl apply -f k8/final-full-deployment.yml
```

## Verify Deployment

```bash
# Check all pods are running
kubectl get pods -n deployment

# Check services
kubectl get svc -n deployment

# Wait for deployment rollout
kubectl rollout status deployment/app-tier -n deployment

# Check HPA status
kubectl get hpa -n deployment
```

**Expected output:**
- `data-tier` pod: `1/1 Running`
- `app-tier-*` pods: `5/5 Running` (scaled by HPA)
- `app-tier` service: NodePort with external port assigned

## Access the Application

**Option 1: Port-forward (recommended for local testing)**
```bash
kubectl port-forward svc/app-tier 8080:8080 -n deployment
# Open http://localhost:8080
```

**Option 2: Minikube service**
```bash
minikube service app-tier -n deployment
```

**Option 3: NodePort (if you know the node IP)**
```bash
NODE_PORT=$(kubectl get svc app-tier -n deployment -o jsonpath='{.spec.ports[0].nodePort}')
NODE_IP=$(minikube ip)
curl http://$NODE_IP:$NODE_PORT
```

## Endpoints

| Path | Description |
|------|-------------|
| `/` | Main app response |
| `/health` | Health check endpoint |

## Troubleshooting

**Pods not starting:**
```bash
kubectl describe pod -l tier=app -n deployment
kubectl logs -l tier=app -n deployment --tail=50
```

**Service not reachable:**
```bash
kubectl get endpoints -n deployment
kubectl exec -it data-tier -n deployment -- redis-cli ping
```

**HPA showing `<unknown>` CPU:**
```bash
# Ensure metrics-server is running
kubectl get pods -n kube-system | grep metrics-server
minikube addons enable metrics-server
```

## Cleanup

```bash
kubectl delete -f k8/final-full-deployment.yml
```

## Notes

- The `app-tier` uses `nginx:alpine` (multi-arch) for ARM64/x86 compatibility
- Redis probes use TCP socket checks on port 6379
- App probes use HTTP GET on `/health`
- HPA scales based on 70% CPU utilization
