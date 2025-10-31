# Data Science MCP Toolset

A comprehensive, declaratively-orchestrated Kubernetes-based MCP toolchain for data sourcing, analysis, and visualization.

## Overview

This toolset demonstrates Toolhive's ability to declaratively orchestrate data science MCP servers in Kubernetes, providing a unified interface for:

- **Data Discovery**: Search and access datasets from Hugging Face, Data.gov, NASA Earthdata, Treasury, and more
- **Data Exploration**: Interactive analysis with pandas, statistical tools, and exploratory workflows
- **Metadata Governance**: Track lineage, catalog datasets, and manage metadata with DataHub
- **Mathematical Visualization**: Create publication-ready diagrams with Penrose

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Client Layer (Claude/Cline)                  │
└────────────────────────────────────────────────────────────────┬┘
                                  │
┌─────────────────────────────────┴──────────────────────────────┐
│           Toolhive Operator (K8s Control Plane)                │
│  - Declarative MCPServer CRDs                                  │
│  - Service Discovery & Health Checks                           │
│  - Transport: Streamable HTTP/SSE, Stdio Proxy                 │
│  - Observability: OTEL, Prometheus, Jaeger                     │
└────────┬──────────┬──────────┬──────────┬──────────┬───────────┘
         │          │          │          │          │
    ┌────▼─┐   ┌────▼─┐   ┌────▼─┐   ┌────▼─┐   ┌────▼─┐
    │  HF  │   │ Data │   │ Pand │   │Explor│   │ Data │
    │ MCP  │   │ View │   │ MCP  │   │ MCP  │   │ Hub  │
    │      │   │      │   │      │   │      │   │      │
    │ Pod  │   │ Pod  │   │ Pod  │   │ Pod  │   │ Pod  │
    └────┬─┘   └────┬─┘   └────┬─┘   └────┬─┘   └────┬─┘
         │          │          │          │          │
    ┌────▼──────────▼──────────▼──────────▼──────────▼─┐
    │         Kubernetes Service Mesh                   │
    │      (Ingress via Caddy/Nginx)                    │
    └───────────────────────────────────────────────────┘
         │          │          │          │
    ┌────▼─┐   ┌────▼─┐   ┌────▼─┐   ┌────▼─┐
    │ HF   │   │ NASA │   │ US   │   │ Urban│
    │ API  │   │ CMR  │   │Treas│   │Inst. │
    │      │   │      │   │      │   │      │
    └──────┘   └──────┘   └──────┘   └──────┘
```

## Included MCP Servers

### Discovery & Access
| Server | Purpose | Auth | Container |
|--------|---------|------|-----------|
| **Hugging Face MCP** | 900k+ models, datasets, papers | Optional HF Token | [shreyas/huggingface-mcp](https://github.com/shreyaskarnik/huggingface-mcp-server) |
| **Dataset Viewer** | Browse HF datasets without download | Optional HF Token | [privetin/dataset-viewer](https://github.com/privetin/dataset-viewer) |
| **Data.gov MCP** | US government datasets (CKAN) | None | [melaodoidao/datagov-mcp](https://github.com/melaodoidao/datagov-mcp-server) |
| **Treasury Data MCP** | US fiscal data & economics | None | [QuantGeekDev/fiscal-data](https://github.com/QuantGeekDev/fiscal-data-mcp) |
| **NASA CMR MCP** | Earth observation & climate data | None | [PO.DAAC/cmr-mcp](https://github.com/podaac/cmr-mcp) |

### Analysis & Exploration
| Server | Purpose | Runtime | Container |
|--------|---------|---------|-----------|
| **MCP Pandas** | Robust pandas analysis (Docker) | Python/FastAPI | [alistair/mcp-pandas](https://github.com/alistairwalsh/mcp_pandas) |
| **Data Exploration** | Full data science assistant (343⭐) | Python/SciPy/SKLearn | [reading-plus/mcp-data-explorer](https://github.com/reading-plus-ai/mcp-server-data-exploration) |

### Governance & Visualization
| Server | Purpose | Runtime | Container |
|--------|---------|---------|-----------|
| **DataHub MCP** | Metadata, lineage, governance | Python/GraphQL | [acryldata/mcp-datahub](https://github.com/acryldata/mcp-server-datahub) |
| **Penrose MCP** | Mathematical diagram generation | TypeScript/SVG | [bmorphism/penrose-mcp](https://github.com/bmorphism/penrose-mcp) |

## Deployment Modes

### Mode 1: Local Development (Docker Compose)
```bash
cd docker-compose
docker-compose up -d
# Servers available on localhost:8000+
```

### Mode 2: Single-Tenancy Kubernetes (KinD)
```bash
# Setup KinD cluster with Toolhive operator
kubectl apply -f k8s/setup-kind-cluster.yaml
kubectl apply -f k8s/toolhive-operator-install.yaml

# Deploy data science toolset
kubectl apply -f k8s/data-science-namespace.yaml
kubectl apply -f k8s/mcpservers/

# Port-forward or expose via ingress
kubectl port-forward svc/huggingface-mcp 8001:8001 -n data-science
```

### Mode 3: Multi-Tenancy Kubernetes (Production)
```bash
# Setup production cluster with OIDC, RBAC, observability
kubectl apply -f k8s/multi-tenancy/namespace-isolation.yaml
kubectl apply -f k8s/multi-tenancy/rbac.yaml

# Deploy with ingress
kubectl apply -f k8s/multi-tenancy/ingress-config.yaml
kubectl apply -f k8s/mcpservers/

# Enable observability
kubectl apply -f k8s/observability/prometheus.yaml
kubectl apply -f k8s/observability/jaeger.yaml
```

## Quick Start

### 1. Prerequisites
```bash
# Install Toolhive CLI
# (Installation instructions in ../../../docs/cli/thv.md)

# Or use Docker Compose for local dev
docker --version  # 20.10+
docker-compose --version  # 2.0+
```

### 2. Local Development
```bash
cd docker-compose
cp .env.example .env
# Edit .env with your API keys (optional: HF_TOKEN, DATAHUB_GMS_URL)

docker-compose up -d

# Check services
curl http://localhost:8001/health  # Hugging Face MCP
curl http://localhost:8002/health  # Dataset Viewer
curl http://localhost:8003/health  # Data Exploration
curl http://localhost:8004/health  # MCP Pandas
curl http://localhost:8005/health  # DataHub
```

### 3. Connect with Claude Desktop
Add to `~/.config/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "huggingface": {
      "command": "npx",
      "args": ["@mcp-servers/huggingface"],
      "env": {
        "HF_TOKEN": "YOUR_HF_TOKEN"
      }
    },
    "data-explorer": {
      "command": "npx",
      "args": ["@mcp-servers/data-explorer"]
    },
    "pandas": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "localhost:5000/mcp-pandas:latest"]
    },
    "datahub": {
      "command": "npx",
      "args": ["@mcp-servers/datahub"],
      "env": {
        "DATAHUB_GMS_URL": "http://localhost:8080",
        "DATAHUB_GMS_TOKEN": "YOUR_TOKEN"
      }
    }
  }
}
```

## Integrated Workflows

### Climate & Environmental Research
```
1. NASA CMR MCP → discover Earth observation datasets
2. Dataset Viewer → inspect without downloading
3. Data Exploration → statistical analysis
4. DataHub → track lineage and metadata
5. Penrose MCP → visualize relationships
```

### AI Model Research & Development
```
1. Hugging Face MCP → find SOTA models/datasets
2. Dataset Viewer → examine dataset structure
3. MCP Pandas → prepare data for training
4. Data Exploration → analyze model inputs/outputs
5. DataHub → version experiments and track provenance
```

### Government Policy Analysis
```
1. Data.gov MCP → discover federal datasets
2. Treasury Data MCP → retrieve economic data
3. Data Exploration → correlate policy metrics
4. Penrose MCP → create policy impact visualizations
5. DataHub → publish findings with governance
```

## Configuration

### Environment Variables

**Discovery & Access:**
```bash
HF_TOKEN=hf_xxxxxxxxxxxxx              # Hugging Face (optional, higher rate limits)
```

**Analysis:**
```bash
# Data Exploration defaults (no config needed)
# MCP Pandas defaults (no config needed)
```

**Governance:**
```bash
DATAHUB_GMS_URL=http://datahub:8080   # DataHub instance
DATAHUB_GMS_TOKEN=xxxxxxxxxxxxx       # DataHub authentication
```

### Toolhive CRD Examples

**Basic Hugging Face MCP Server:**
```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: huggingface-mcp
  namespace: data-science
spec:
  image: shreyas/huggingface-mcp-server:latest
  transport: streamable-http  # or: stdio, sse
  port: 8001
  env:
    - name: HF_TOKEN
      valueFrom:
        secretKeyRef:
          name: hf-credentials
          key: token
  resources:
    limits:
      memory: "2Gi"
      cpu: "1000m"
```

**Data Exploration with Observability:**
```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: data-exploration
  namespace: data-science
spec:
  image: reading-plus/mcp-data-exploration:latest
  transport: streamable-http
  port: 8003
  observability:
    enabled: true
    tracing:
      jaegerCollectorUrl: http://jaeger-collector:14268/api/traces
    metrics:
      prometheusUrl: http://prometheus:9090
  resources:
    requests:
      memory: "4Gi"
      cpu: "2000m"
    limits:
      memory: "8Gi"
      cpu: "4000m"
  healthCheck:
    enabled: true
    path: /health
    initialDelaySeconds: 10
    periodSeconds: 30
```

## Observability

### Prometheus Metrics
- Request/response times per MCP server
- Tool invocation counts and latencies
- Transport efficiency (HTTP vs SSE vs Stdio)
- DataHub metadata operation metrics

```bash
kubectl port-forward svc/prometheus 9090:9090 -n data-science
# Open http://localhost:9090
```

### Jaeger Distributed Tracing
```bash
kubectl port-forward svc/jaeger-query 16686:16686 -n data-science
# Open http://localhost:16686
# Trace data flow: Client → Toolhive → MCP → External API
```

### Logs
```bash
# View specific MCP server logs
kubectl logs -f deployment/huggingface-mcp -n data-science

# Stream all data science toolset logs
kubectl logs -f -l app=mcp-toolset -n data-science
```

## Building Custom Images

If stable upstream images don't exist, build from source:

```bash
cd containers/

# Build Hugging Face MCP
docker build -t localhost:5000/huggingface-mcp:latest \
  -f huggingface-mcp.Dockerfile .
docker push localhost:5000/huggingface-mcp:latest

# Build Data Exploration MCP
docker build -t localhost:5000/data-exploration:latest \
  -f data-exploration.Dockerfile .
docker push localhost:5000/data-exploration:latest

# Build MCP Pandas (already containerized)
docker build -t localhost:5000/mcp-pandas:latest \
  -f mcp-pandas.Dockerfile .
docker push localhost:5000/mcp-pandas:latest
```

## Security Considerations

### Authentication
- **HF Token**: Store in K8s Secret, not ConfigMap
- **DataHub Token**: Use sealed-secrets or external secret operator
- **OAuth2 Flow**: Toolhive supports OIDC for multi-tenant deployments

Example with sealed-secrets:
```bash
echo -n "hf_xxxx" | kubectl create secret generic hf-credentials \
  --dry-run=client --from-file=token=/dev/stdin -o yaml | \
  kubeseal -f - > hf-credentials-sealed.yaml
kubectl apply -f hf-credentials-sealed.yaml
```

### Network Isolation
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: data-science-ingress
  namespace: data-science
spec:
  podSelector:
    matchLabels:
      app: mcp-toolset
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: toolhive-operator
      ports:
        - protocol: TCP
          port: 8080
```

### Data Privacy
- External API calls go through egress proxy (optional)
- Sensitive datasets can stay in Kubernetes (not downloaded)
- Audit logging via Toolhive audit middleware

## Ingress & Service Exposure

### Caddy Ingress (Recommended)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: data-science-ingress
  namespace: data-science
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - mcp.example.com
      secretName: data-science-tls
  rules:
    - host: mcp.example.com
      http:
        paths:
          - path: /huggingface
            pathType: Prefix
            backend:
              service:
                name: huggingface-mcp
                port:
                  number: 8001
          - path: /dataset-viewer
            pathType: Prefix
            backend:
              service:
                name: dataset-viewer
                port:
                  number: 8002
          - path: /data-explorer
            pathType: Prefix
            backend:
              service:
                name: data-explorer
                port:
                  number: 8003
```

### Local Development (Port Forward)
```bash
# Create local tunnel to K8s services
for svc in huggingface-mcp dataset-viewer data-explorer mcp-pandas datahub; do
  kubectl port-forward svc/$svc $((8000 + $(kubectl get svc $svc -o json | jq '.spec.ports[0].port'))) \
    -n data-science &
done
```

## Troubleshooting

### Service Not Ready
```bash
# Check pod status
kubectl get pods -n data-science

# Describe failing pod
kubectl describe pod huggingface-mcp-xxx -n data-science

# Check container logs
kubectl logs huggingface-mcp-xxx -n data-science
```

### Connection Issues
```bash
# Test service endpoint
kubectl exec -it data-explorer-xxx -n data-science \
  -- curl -v http://huggingface-mcp:8001/health

# Check network policies
kubectl get networkpolicies -n data-science
```

### High Latency
```bash
# Check metrics
kubectl top pods -n data-science
kubectl top nodes

# View traces in Jaeger
# Look for slow operations in: Client → Toolhive Proxy → MCP → External API
```

## References

- [Snyk: 11 Data Science MCP Servers](https://snyk.io/articles/11-data-science-mcp-servers-for-sourcing-analyzing-and-visualizing-data/)
- [Toolhive Operator Docs](../../../docs/arch/)
- [MCP Specification](https://modelcontextprotocol.io)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

## Contributing

To add new data science MCP servers:

1. Create server CRD in `k8s/mcpservers/new-server.yaml`
2. Build container image (if needed)
3. Add to `docker-compose/docker-compose.yml`
4. Update this README with workflow examples
5. Submit PR

## License

Same as Toolhive (per /home/adam/GitHub/toolhive/LICENSE)
