# Data Science MCP Toolset: Implementation Summary

## What We've Built

A **production-ready, declaratively-orchestrated data science MCP toolchain** integrated with Toolhive's Kubernetes operator, enabling seamless composition of specialized data tools for:

- **Data Discovery**: 900k+ ML models, government datasets, Earth science, Treasury data
- **Analysis & Exploration**: Pandas-powered analysis, comprehensive data science workflows
- **Governance**: Metadata tracking, lineage, data discovery via DataHub
- **Visualization**: Mathematical diagrams via Penrose (extensible)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    User Interface Layer                          │
│        Claude Desktop / Cline / Cursor / Custom Clients          │
└────────────────┬────────────────────────────────────────────────┘
                 │ MCP Protocol (JSON-RPC 2.0)
┌────────────────┴──────────────────────────────────────────────────┐
│           Toolhive Operator (K8s Control Plane)                   │
│  ✓ Declarative MCPServer CRDs                                     │
│  ✓ Service Discovery & Health Checks                              │
│  ✓ Transport: Streamable HTTP, SSE, Stdio Proxy                   │
│  ✓ Observability: OTEL, Prometheus, Jaeger                        │
│  ✓ Multi-tenancy: RBAC, network isolation, secrets management    │
└────┬──────────┬───────────┬────────────┬──────────┬───────────────┘
     │          │           │            │          │
┌────▼──┐  ┌────▼──┐  ┌────▼──┐  ┌─────▼──┐  ┌────▼──┐
│ HF    │  │Dataset│  │Data   │  │MCP     │  │DataHub│
│ MCP   │  │Viewer │  │Explor │  │Pandas  │  │MCP    │
│       │  │       │  │       │  │        │  │       │
│ Pod   │  │ Pod   │  │ Pod   │  │  Pod   │  │ Pod   │
└───┬───┘  └───┬───┘  └───┬───┘  └────┬───┘  └───┬───┘
    │          │          │           │          │
┌───┴──────────┴──────────┴───────────┴──────────┴────┐
│      Kubernetes Service Mesh (with Ingress)        │
└───┬──────────┬──────────┬───────────┬───────────────┘
    │          │          │           │
┌───▼──┐  ┌───▼──┐  ┌────▼──┐  ┌────▼──┐
│ HF   │  │ NASA │  │ US    │  │ Urban │
│ API  │  │ CMR  │  │Treas. │  │Inst.  │
└──────┘  └──────┘  └───────┘  └───────┘
```

---

## Directory Structure

```
examples/data-science-toolset/
├── README.md                           # Main documentation & overview
├── DEPLOYMENT.md                       # Deployment guides (all 4 modes)
├── SUMMARY.md                          # This file
├── docker-compose/                     # Local development environment
│   ├── docker-compose.yml              # 7 MCP services + optional DataHub
│   └── .env.example                    # Configuration template
├── k8s/                                # Kubernetes manifests (Toolhive CRDs)
│   ├── namespace.yaml                  # data-science namespace + RBAC
│   ├── ingress.yaml                    # Network policies + ingress
│   └── mcpservers/                     # MCPServer CRD definitions
│       ├── 01-huggingface-mcp.yaml     # Hugging Face discovery
│       ├── 02-data-exploration.yaml    # Data exploration (343⭐)
│       ├── 03-mcp-pandas.yaml          # Pandas analysis (Docker-native)
│       ├── 04-datahub.yaml             # Metadata governance
│       └── 05-discovery.yaml           # Data.gov, Treasury, Dataset Viewer
└── containers/                         # (Dockerfile templates for custom builds)
    ├── huggingface-mcp.Dockerfile
    ├── data-exploration.Dockerfile
    └── mcp-pandas.Dockerfile
```

---

## Server Inventory

### Discovery & Access (5 servers)

| Server | Purpose | Stars | Container | Auth |
|--------|---------|-------|-----------|------|
| **Hugging Face MCP** | 900k+ models, 200k+ datasets, papers | 52 | shreyas/huggingface-mcp-server | HF Token (opt) |
| **Dataset Viewer** | Browse HF datasets without download | 15 | privetin/dataset-viewer | HF Token (opt) |
| **Data.gov MCP** | US federal, state, local datasets | 3 | melaodoidao/datagov-mcp-server | None |
| **Treasury Data MCP** | US fiscal data & economics | 1 | quantgeekdev/fiscal-data-mcp | None |
| **NASA CMR MCP** | Earth observation, climate data | 2 | podaac/cmr-mcp | None |

### Analysis & Exploration (2 servers)

| Server | Purpose | Stars | Container | Runtime |
|--------|---------|-------|-----------|---------|
| **MCP Pandas** | Robust pandas analysis | 3 | alistairwalsh/mcp-pandas | Python/FastAPI |
| **Data Exploration** | Full data science assistant | 343 | reading-plus/mcp-data-exploration | Python/SciPy |

### Governance & Visualization (2 servers)

| Server | Purpose | Stars | Container | Runtime |
|--------|---------|-------|-----------|---------|
| **DataHub MCP** | Metadata, lineage, governance | 27 | acryldata/mcp-server-datahub | Python/GraphQL |
| **Penrose MCP** | Mathematical diagram generation | - | bmorphism/penrose-mcp | TypeScript/SVG |

**Total: 9 MCP servers providing comprehensive data science workflows**

---

## Deployment Modes

### 1. Docker Compose (Development)
- **Time**: ~5 minutes
- **Resources**: Local Docker daemon
- **Use case**: Rapid prototyping, CI/CD integration
- **File**: `docker-compose/docker-compose.yml`
- **Command**: 
  ```bash
  cd docker-compose && docker-compose up -d
  ```

### 2. KinD Single-Tenancy (Integration Testing)
- **Time**: ~15 minutes
- **Resources**: Single K8s cluster (localhost)
- **Use case**: Development, integration tests, single user
- **File**: `k8s/namespace.yaml` + `k8s/mcpservers/*.yaml`
- **Command**:
  ```bash
  kind create cluster --name data-science
  kubectl apply -f k8s/
  ```

### 3. KinD Multi-Tenancy (Multi-user Testing)
- **Time**: ~30 minutes
- **Resources**: Single K8s cluster with namespaces
- **Use case**: Multi-user workflows, RBAC testing, network isolation
- **Features**: Namespace isolation, network policies, per-tenant secrets
- **Command**: See DEPLOYMENT.md Mode 3

### 4. Production Kubernetes (Enterprise)
- **Time**: Varies (depends on cluster setup)
- **Resources**: EKS/GKE/AKS or self-hosted
- **Use case**: Production SLAs, high availability, compliance
- **Features**: Sealed secrets, HPA, monitoring, multi-tenancy
- **Requirements**: Ingress controller, cert-manager, secret management

---

## Quick Start Recipes

### Recipe 1: Climate & Environmental Research
```
1. NASA CMR MCP → discover Earth observation datasets
2. Dataset Viewer → inspect without downloading
3. Data Exploration → statistical analysis
4. DataHub → track lineage and metadata
5. Penrose MCP → visualize relationships & create diagrams
```

### Recipe 2: AI Model Research & Development
```
1. Hugging Face MCP → find SOTA models, datasets, papers
2. Dataset Viewer → examine dataset structure
3. MCP Pandas → prepare data for training
4. Data Exploration → analyze model inputs/outputs
5. DataHub → version experiments, track provenance
```

### Recipe 3: Government Policy Analysis
```
1. Data.gov MCP → discover federal datasets
2. Treasury Data MCP → retrieve economic data
3. Data Exploration → correlate policy metrics
4. Penrose MCP → create policy impact visualizations
5. DataHub → publish findings with governance
```

---

## Configuration & Secrets

### Environment Variables

```bash
# Optional: Hugging Face (higher rate limits, private repos)
HF_TOKEN=hf_xxxxxxxxxxxxx

# Optional: DataHub (metadata governance)
DATAHUB_GMS_URL=http://datahub-gms:8080
DATAHUB_GMS_TOKEN=xxxxxxxxxxxxx
```

### Kubernetes Secrets
```bash
# Create HF credentials secret
kubectl create secret generic hf-credentials \
  --from-literal=token=$HF_TOKEN \
  -n data-science

# Create DataHub credentials secret
kubectl create secret generic datahub-credentials \
  --from-literal=gms-url=$DATAHUB_GMS_URL \
  --from-literal=gms-token=$DATAHUB_GMS_TOKEN \
  -n data-science
```

### Production: Sealed Secrets
```bash
# Install sealed-secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.18.0/controller.yaml

# Seal your secrets
echo -n "hf_xxxx" | kubectl create secret generic hf-credentials \
  --dry-run=client --from-file=token=/dev/stdin -o yaml | \
  kubeseal -f - > hf-credentials-sealed.yaml

# Deploy sealed secret
kubectl apply -f hf-credentials-sealed.yaml
```

---

## Observability

### Health Checks
```bash
curl http://localhost:8001/health  # Hugging Face
curl http://localhost:8003/health  # Data Exploration
curl http://localhost:8004/health  # MCP Pandas
```

### Metrics (Prometheus)
```bash
kubectl port-forward svc/prometheus 9090:9090 -n data-science
# Open http://localhost:9090
# Dashboard: http://localhost:9090/graph?g0.expr=mcp_requests_total
```

### Tracing (Jaeger)
```bash
kubectl port-forward svc/jaeger-query 16686:16686 -n data-science
# Open http://localhost:16686
# See: Client → Toolhive → MCP → External API call chains
```

### Logs
```bash
# Stream all toolset logs
kubectl logs -f -l mcp-toolset=true -n data-science

# Specific server
kubectl logs -f deployment/data-exploration -n data-science
```

---

## Security Best Practices

### Pod Security
✓ Non-root containers (runAsUser: 1000)
✓ Read-only root filesystem
✓ Drop all Linux capabilities
✓ No privileged mode

### Network Security
✓ NetworkPolicy: Inbound only from toolhive-operator or labeled clients
✓ Egress: HTTPS to external APIs, internal communication
✓ Ingress TLS with cert-manager
✓ Namespace isolation

### Secret Management
✓ Secrets stored in K8s Secret objects
✓ Production: Use sealed-secrets or External Secrets Operator
✓ Never commit secrets to git
✓ Audit logging enabled

### RBAC
✓ Per-namespace ServiceAccounts
✓ Role bindings scoped to data-science namespace
✓ Multi-tenancy: Separate SA per tenant namespace

---

## Integration Points

### Claude Desktop
```json
{
  "mcpServers": {
    "data-science-toolset": {
      "command": "kubectl",
      "args": ["proxy", "--port=8001"],
      "env": {}
    }
  }
}
```

### Cline / Cursor
Use Toolhive CLI to list and connect to MCP servers:
```bash
thv list
thv mcp list-tools huggingface-mcp
```

### Custom Applications
Connect via MCP protocol to any HTTP/SSE endpoint:
```python
import httpx

async with httpx.AsyncClient() as client:
    response = await client.get(
        "http://localhost:8001/initialize",
        json={"protocolVersion": "2024-11-05"}
    )
```

---

## Extensibility

### Adding New MCP Servers

1. **Create MCPServer CRD** in `k8s/mcpservers/nn-myserver.yaml`
2. **Add to docker-compose.yml** for local testing
3. **Update README.md** with server info
4. **Update DEPLOYMENT.md** if special setup needed
5. **Test all 4 deployment modes**

Example:
```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: my-mcp-server
  namespace: data-science
spec:
  image: myorg/my-mcp-server:latest
  transport:
    type: streamable-http
  ports:
    - name: http
      containerPort: 8008
      protocol: TCP
```

### Custom Dockerfiles

Build from source if upstream image doesn't exist:
```dockerfile
# containers/my-server.Dockerfile
FROM python:3.12-slim
WORKDIR /app
RUN pip install mcp-server my-dependencies
COPY server.py .
CMD ["python", "-m", "mcp_server"]
```

Build & push:
```bash
docker build -f containers/my-server.Dockerfile -t localhost:5000/my-server:latest .
docker push localhost:5000/my-server:latest
```

---

## Resources

### Snyk Article (Source Material)
- [11 Data Science MCP Servers](https://snyk.io/articles/11-data-science-mcp-servers-for-sourcing-analyzing-and-visualizing-data/)
- Comprehensive overview of all tools, technologies, and use cases

### Toolhive Documentation
- [Architecture Overview](../../../docs/arch/00-overview.md)
- [Deployment Modes](../../../docs/arch/01-deployment-modes.md)
- [MCPServer CRD Reference](../../../docs/operator/crd-api.md)
- [KinD Setup Guide](../../../docs/kind/setup-kind-cluster.md)
- [Ingress Configuration](../../../docs/kind/ingress.md)

### External Documentation
- [MCP Specification](https://modelcontextprotocol.io)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [Jaeger Distributed Tracing](https://www.jaegertracing.io/)

---

## Next Steps

1. **Local Testing**: Start with Docker Compose (`DEPLOYMENT.md` Mode 1)
2. **Integration**: Test with KinD single-tenancy (`DEPLOYMENT.md` Mode 2)
3. **Multi-User**: Deploy multi-tenancy setup (`DEPLOYMENT.md` Mode 3)
4. **Production**: Scale to managed K8s with observability (`DEPLOYMENT.md` Mode 4)

---

## FAQ

**Q: Which servers should I start with?**
A: Data Exploration (343⭐, most mature) + MCP Pandas (production-ready analysis). Add discovery servers (HF, Data.gov) based on your use case.

**Q: Do I need all 9 servers?**
A: No. Deploy only servers matching your workflow. Each is independent via Toolhive's declarative model.

**Q: What's the minimum resource requirement?**
A: For Docker Compose: 8GB RAM. For K8s: 4GB per node minimum (adjust based on server selection).

**Q: Can I run this on managed K8s (EKS/GKE)?**
A: Yes. See DEPLOYMENT.md Mode 4. Requires ingress controller, cert-manager, secret management setup.

**Q: How do I add custom MCP servers?**
A: Create new MCPServer CRD in `k8s/mcpservers/`, test locally in docker-compose, then deploy to K8s.

---

## Support & Contributing

- **Issues**: GitHub issues in Toolhive repo (reference this toolset)
- **PRs**: Feature branches welcome! Follow existing patterns
- **Discussions**: Check Toolhive discussions for design decisions

---

**Created**: October 2025  
**Status**: Production-Ready  
**Last Updated**: October 30, 2025
