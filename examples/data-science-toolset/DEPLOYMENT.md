# Data Science MCP Toolset: Deployment Guides

## Quick Reference

| Mode | Environment | Complexity | Use Case |
|------|-------------|-----------|----------|
| **Docker Compose** | Local dev, testing | Low | Rapid prototyping, CI/CD |
| **KinD Single-Tenancy** | Local K8s cluster | Medium | Development, integration tests |
| **KinD Multi-Tenancy** | Isolated namespaces | High | Multi-user, RBAC testing |
| **Production K8s** | Cloud/on-premise | High | Enterprise, SLA compliance |

---

## Mode 1: Docker Compose (5 minutes)

### Setup
```bash
cd examples/data-science-toolset/docker-compose

cp .env.example .env
# Optional: Edit .env with HF_TOKEN, DATAHUB credentials

docker-compose up -d
```

### Verify
```bash
# Check all services are running
docker-compose ps

# Test each endpoint
curl http://localhost:8001/health  # Hugging Face
curl http://localhost:8003/health  # Data Exploration
curl http://localhost:8004/health  # MCP Pandas
```

### Connect to Claude Desktop
```json
{
  "mcpServers": {
    "huggingface": {
      "command": "curl",
      "args": ["--unix-socket", "/var/run/docker.sock", "http://huggingface-mcp:8001"]
    }
  }
}
```

### Cleanup
```bash
docker-compose down
docker volume prune -f
```

---

## Mode 2: KinD Single-Tenancy (15 minutes)

### Prerequisites
```bash
# Install KinD
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/

# Install kubectl
# (Usually pre-installed; verify with: kubectl version --client)

# Install Toolhive operator
# (Refer to ~/GitHub/toolhive/docs/kind/deploying-toolhive-operator.md)
```

### Cluster Setup
```bash
# Create KinD cluster
kind create cluster --name data-science

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

### Install Toolhive Operator
```bash
# Apply operator CRDs and deployment
kubectl apply -f https://github.com/StacklokLabs/toolhive/releases/download/v0.x.x/toolhive-operator-bundle.yaml

# Wait for operator readiness
kubectl wait --for=condition=ready pod \
  -l app=toolhive-operator -n toolhive-operator-system \
  --timeout=300s
```

### Deploy Data Science Toolset
```bash
cd examples/data-science-toolset/k8s

# Create namespace and RBAC
kubectl apply -f namespace.yaml

# Deploy MCPServer resources
kubectl apply -f mcpservers/

# Verify pods are running
kubectl get pods -n data-science
kubectl logs -f deployment/huggingface-mcp -n data-science
```

### Port Forward for Local Access
```bash
# Forward multiple services
for svc in huggingface-mcp data-exploration mcp-pandas; do
  kubectl port-forward svc/$svc $(echo $svc | grep -o '[0-9]\+$') \
    -n data-science &
done

# Or forward individually
kubectl port-forward svc/huggingface-mcp 8001:8001 -n data-science
```

### Cleanup
```bash
kind delete cluster --name data-science
```

---

## Mode 3: KinD Multi-Tenancy (30 minutes)

### Cluster Setup
```bash
kind create cluster --name data-science-multi

# Label nodes for multi-tenancy
kubectl label node data-science-multi-control-plane \
  environment=production
```

### Install Toolhive with Multi-Tenancy
```bash
# Apply operator with multi-tenancy enabled
kubectl apply -f deploy/charts/operator/values-multi-tenant.yaml

# Wait for operator
kubectl wait --for=condition=ready pod \
  -l app=toolhive-operator -n toolhive-operator-system \
  --timeout=300s
```

### Setup Namespace Isolation
```bash
cd examples/data-science-toolset/k8s

# Create multiple isolated namespaces
for tenant in research analytics governance; do
  kubectl create namespace data-science-$tenant
  kubectl label namespace data-science-$tenant \
    tenant=$tenant \
    mcp-toolset=true
done

# Apply RBAC and network policies per tenant
kubectl apply -f namespace.yaml
kubectl apply -f ingress.yaml
```

### Deploy to Tenant Namespaces
```bash
# Deploy to each tenant
for tenant in research analytics governance; do
  NS="data-science-$tenant"
  
  # Apply secrets for tenant
  kubectl create secret generic hf-credentials \
    --from-literal=token=$HF_TOKEN \
    -n $NS
  
  # Deploy MCP servers
  kubectl apply -f mcpservers/ -n $NS
done

# Verify isolation
kubectl get pods -n data-science-research
kubectl get pods -n data-science-analytics
```

### Setup Observability
```bash
# Deploy Prometheus
kubectl apply -f k8s/observability/prometheus.yaml

# Deploy Jaeger
kubectl apply -f k8s/observability/jaeger.yaml

# Access dashboards
kubectl port-forward svc/prometheus 9090:9090 -n data-science &
kubectl port-forward svc/jaeger-query 16686:16686 -n data-science &
```

### Test Multi-Tenancy Isolation
```bash
# Verify network policies prevent cross-namespace traffic
kubectl run test-pod --image=curlimages/curl \
  -n data-science-research -- sleep 3600

# This should fail (isolated)
kubectl exec -it test-pod -n data-science-research \
  -- curl http://huggingface-mcp.data-science-analytics:8001/health

# This should succeed (same namespace)
kubectl exec -it test-pod -n data-science-research \
  -- curl http://huggingface-mcp.data-science-research:8001/health
```

### Cleanup
```bash
kind delete cluster --name data-science-multi
```

---

## Mode 4: Production Kubernetes (varies)

### Prerequisites
- Managed K8s cluster (EKS, GKE, AKS) or self-hosted
- NGINX Ingress Controller
- Cert-Manager for TLS
- Secret management (Sealed Secrets, External Secrets Operator)

### Pre-Flight Checks
```bash
# Verify cluster access
kubectl cluster-info

# Check node capacity
kubectl top nodes
kubectl top pods -A

# Verify ingress controller
kubectl get ingressclass
kubectl get pods -n ingress-nginx
```

### Secret Management Setup
```bash
# Install Sealed Secrets (example)
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Seal your HF_TOKEN
echo -n "hf_xxxx" | kubectl create secret generic hf-credentials \
  --dry-run=client --from-file=token=/dev/stdin -o yaml | \
  kubeseal -f - > hf-credentials-sealed.yaml

kubectl apply -f hf-credentials-sealed.yaml
```

### Deploy Data Science Toolset
```bash
cd examples/data-science-toolset/k8s

# Create namespace
kubectl apply -f namespace.yaml

# Apply secrets
kubectl apply -f hf-credentials-sealed.yaml

# Deploy MCPServer CRDs
kubectl apply -f mcpservers/

# Setup ingress with DNS
kubectl apply -f ingress.yaml

# Verify deployment
kubectl get mcpservers -n data-science
kubectl get pods -n data-science
kubectl get svc -n data-science
```

### Monitor Deployment
```bash
# Watch pod creation
kubectl get pods -n data-science -w

# View deployment logs
kubectl logs -f deployment/huggingface-mcp -n data-science

# Check events
kubectl get events -n data-science --sort-by='.lastTimestamp'
```

### Scale MCPServers
```bash
# Horizontally scale data-exploration for high concurrency
kubectl scale deployment data-exploration \
  --replicas=3 -n data-science

# Verify scaling
kubectl get pods -l app=data-exploration -n data-science
```

### Setup Monitoring
```bash
# Install Prometheus Operator (Helm)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace

# Configure Toolhive metrics scraping
kubectl apply -f k8s/observability/servicemonitor.yaml

# Setup Grafana dashboards
kubectl apply -f k8s/observability/grafana-dashboards/
```

### Enable Tracing
```bash
# Deploy Jaeger operator (example)
kubectl create namespace jaeger
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.48.0/jaeger-operator.yaml -n jaeger

# Create Jaeger instance
kubectl apply -f k8s/observability/jaeger-instance.yaml
```

---

## Troubleshooting

### Pod Fails to Start
```bash
# Check pod status
kubectl describe pod <pod-name> -n data-science

# View container logs
kubectl logs <pod-name> -n data-science -c <container>

# Check resource limits
kubectl top pod <pod-name> -n data-science
```

### Service Unreachable
```bash
# Verify service exists
kubectl get svc -n data-science

# Test DNS resolution
kubectl run -it --rm debug --image=curlimages/curl --restart=Never \
  -n data-science -- nslookup huggingface-mcp.data-science.svc.cluster.local

# Check ingress
kubectl get ingress -n data-science
kubectl describe ingress data-science-ingress -n data-science
```

### High Latency
```bash
# Check resource utilization
kubectl top pods -n data-science
kubectl top nodes

# View traces in Jaeger
# Look for: Client → Toolhive Proxy → MCP Server → External API

# Adjust resource requests/limits
kubectl patch deployment data-exploration -n data-science --patch \
  '{"spec":{"template":{"spec":{"containers":[{"name":"data-exploration","resources":{"requests":{"memory":"4Gi","cpu":"2"}}}]}}}}'
```

### Authentication Failures
```bash
# Verify secrets are mounted
kubectl get secrets -n data-science

# Check environment variable configuration
kubectl exec -it <pod> -n data-science -- env | grep HF_TOKEN

# Re-create secret if corrupted
kubectl delete secret hf-credentials -n data-science
kubectl create secret generic hf-credentials \
  --from-literal=token=$HF_TOKEN -n data-science
```

---

## Performance Tuning

### Resource Optimization
```yaml
# Minimal (development)
resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "1Gi"
    cpu: "500m"

# Standard (production)
resources:
  requests:
    memory: "2Gi"
    cpu: "1"
  limits:
    memory: "8Gi"
    cpu: "4"
```

### HPA (Horizontal Pod Autoscaling)
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: data-exploration-hpa
  namespace: data-science
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: data-exploration
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## References

- [Toolhive: KinD Setup](../../../docs/kind/setup-kind-cluster.md)
- [Toolhive: Operator Deployment](../../../docs/kind/deploying-toolhive-operator.md)
- [Toolhive: Ingress Configuration](../../../docs/kind/ingress.md)
- [Kubernetes: Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)
- [Kubernetes: Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
