# Data Science MCP Toolset: Quick Start Guide

## 30-Second Overview

**9 MCP servers** delivering data sourcing, analysis, visualization, and governance via **Toolhive's Kubernetes orchestration**.

```
Discovery  → Analysis  → Visualization
   ↓           ↓            ↓
HF/Data.gov   Pandas    DataHub/Penrose
   ↓           ↓            ↓
Search/Download → Analyze → Visualize & Govern
```

---

## 5-Minute Setup (Docker Compose)

```bash
# 1. Clone and navigate
cd ~/GitHub/toolhive/examples/data-science-toolset/docker-compose

# 2. Configure (optional)
cp .env.example .env
# Edit .env if you have HF_TOKEN or DataHub credentials

# 3. Launch
docker-compose up -d

# 4. Verify
curl http://localhost:8001/health  # Hugging Face
curl http://localhost:8003/health  # Data Exploration
curl http://localhost:8004/health  # MCP Pandas

# Done! All 7 servers running locally on ports 8001-8007
```

---

## Access Points

| Port | Service | Purpose |
|------|---------|---------|
| 8001 | Hugging Face MCP | Search 900k+ models, datasets, papers |
| 8002 | Dataset Viewer | Browse HF datasets without download |
| 8003 | Data Exploration | Full data science workflows |
| 8004 | MCP Pandas | Robust pandas analysis |
| 8005 | DataHub MCP | Metadata, lineage, governance |
| 8006 | Data.gov MCP | US federal datasets |
| 8007 | Treasury Data MCP | Economic & fiscal data |

---

## Connect to Claude Desktop

Add to `~/.config/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "huggingface": {
      "command": "curl",
      "args": ["http://localhost:8001"]
    },
    "data-explorer": {
      "command": "curl",
      "args": ["http://localhost:8003"]
    }
  }
}
```

Restart Claude → use in chat!

---

## Three Use Case Recipes

### 1. Find & Analyze AI Models (5 minutes)
```
1. "Find me SOTA transformers for time series forecasting" 
   → Hugging Face MCP searches & returns results
2. "Analyze this dataset for missing values"
   → Data Exploration loads & analyzes
3. "Track the metadata and lineage"
   → DataHub catalogs the experiment
```

### 2. Compare Government Data (10 minutes)
```
1. "Get Treasury spending by department"
   → Treasury Data MCP retrieves fiscal data
2. "Find related datasets on Data.gov"
   → Data.gov MCP searches federal datasets
3. "Create a visualization comparing trends"
   → Data Exploration + Penrose generate charts
```

### 3. Explore Climate Data (15 minutes)
```
1. "Search NASA Earth observation datasets"
   → NASA CMR MCP finds climate datasets
2. "Browse without downloading first"
   → Dataset Viewer previews structure
3. "Analyze correlations with economic data"
   → Data Exploration correlates metrics
```

---

## Deployment Modes

| Mode | Time | Use Case | Command |
|------|------|----------|---------|
| **Docker Compose** | 5 min | Local dev | `docker-compose up` |
| **KinD Single** | 15 min | Integration tests | `kind create cluster && kubectl apply -f k8s/` |
| **KinD Multi** | 30 min | Multi-user testing | See DEPLOYMENT.md Mode 3 |
| **Production K8s** | Varies | Enterprise | See DEPLOYMENT.md Mode 4 |

---

## What's Where

```
examples/data-science-toolset/
├── README.md           ← Full documentation & architecture
├── DEPLOYMENT.md       ← All 4 deployment modes (detailed)
├── SUMMARY.md          ← Complete technical summary
├── QUICKSTART.md       ← This file!
├── docker-compose/     ← Local dev (7 MCPs + optional backend)
└── k8s/                ← Kubernetes CRDs (Toolhive operator)
    ├── namespace.yaml  ← data-science namespace + RBAC
    ├── ingress.yaml    ← Ingress + network policies
    └── mcpservers/     ← 5 MCPServer CRD definitions
```

---

## Servers at a Glance

### Discovery (Search & Download)
- **Hugging Face**: 900k+ models, 200k+ datasets, papers | ⭐52
- **Dataset Viewer**: Browse HF datasets | ⭐15
- **Data.gov**: US federal/state/local datasets | ⭐3
- **Treasury Data**: US economic data | ⭐1

### Analysis (Process & Understand)
- **Data Exploration**: Full data science workflows | ⭐343 (recommended)
- **MCP Pandas**: Containerized pandas + FastAPI | ⭐3

### Governance (Manage & Visualize)
- **DataHub**: Metadata, lineage, governance | ⭐27
- **Penrose**: Mathematical diagrams | ⭐custom

---

## Common Commands

### Docker Compose
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f data-exploration

# Stop all
docker-compose down

# Clean volumes
docker volume prune -f
```

### Kubernetes (KinD)
```bash
# Deploy to existing cluster
kubectl apply -f k8s/

# Watch pods
kubectl get pods -n data-science -w

# Port-forward
kubectl port-forward svc/huggingface-mcp 8001:8001 -n data-science

# View logs
kubectl logs -f deployment/data-exploration -n data-science

# Delete toolset
kubectl delete namespace data-science
```

---

## Troubleshooting

### Service not responding?
```bash
# Check health
curl http://localhost:8001/health

# Check logs
docker-compose logs huggingface-mcp
# or
kubectl logs deployment/huggingface-mcp -n data-science
```

### Want to rebuild from source?
```bash
# Kubernetes: CRD already references upstream images (no rebuild needed)

# Custom build (if needed):
docker build -f containers/my-server.Dockerfile -t localhost:5000/my-server:latest .
docker push localhost:5000/my-server:latest
# Update CRD image: value in k8s/mcpservers/*.yaml
```

### Need to scale for higher concurrency?
```bash
# Kubernetes
kubectl scale deployment data-exploration --replicas=3 -n data-science

# Or use HPA (see DEPLOYMENT.md)
kubectl apply -f k8s/hpa.yaml
```

---

## Next Steps

1. **Local Test**: Run docker-compose (this page, 5 min)
2. **Explore Features**: Use with Claude Desktop (10 min)
3. **Try KinD**: Set up local K8s cluster (DEPLOYMENT.md Mode 2, 15 min)
4. **Go Multi-User**: Test multi-tenancy (DEPLOYMENT.md Mode 3, 30 min)
5. **Deploy to Production**: Scale to managed K8s (DEPLOYMENT.md Mode 4)

---

## Resources

- **Full Docs**: README.md (architecture, features, configuration)
- **Deployment Guide**: DEPLOYMENT.md (all 4 modes with troubleshooting)
- **Technical Details**: SUMMARY.md (complete server inventory, integration points)
- **Source Article**: [Snyk: 11 Data Science MCP Servers](https://snyk.io/articles/11-data-science-mcp-servers-for-sourcing-analyzing-and-visualizing-data/)
- **Toolhive Docs**: ~/GitHub/toolhive/docs/arch/

---

## Quick Decision Tree

**"Should I use this?"**
- ✓ Working with datasets? YES
- ✓ Need to analyze data? YES
- ✓ Want repeatable workflows? YES
- ✓ Need multi-user isolation? Use Kubernetes mode
- ✓ Just exploring? Start with docker-compose

**"What do I deploy first?"**
1. Data Exploration (343⭐, most mature)
2. Hugging Face (richest dataset catalog)
3. MCP Pandas (production-ready analysis)
4. Others as needed (Data.gov, Treasury, etc.)

**"How much infrastructure?"**
- **Local**: Docker Compose (laptop friendly)
- **Team**: KinD on shared workstation
- **Organization**: Managed K8s (EKS/GKE/AKS)

---

**Ready? Run this:**
```bash
cd ~/GitHub/toolhive/examples/data-science-toolset/docker-compose
docker-compose up -d && echo "✓ All 7 MCPs running on localhost:8001-8007"
```

Then ask Claude:
> "Search for time series forecasting models on Hugging Face and analyze their performance"
