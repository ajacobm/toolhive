# Comprehensive Data Science MCP Toolset Deployment with ToolHive Kubernetes Operator

## Objective
Deploy a complete data science toolchain using ToolHive's Kubernetes operator to orchestrate Model Context Protocol (MCP) servers for data discovery, analysis, and visualization in a production-grade Kubernetes environment.

## Context
ToolHive is a Kubernetes operator that manages MCP servers and registries, providing declarative orchestration of AI/ML toolchains. The data science toolset consists of seven specialized MCP servers:
1. Hugging Face MCP - Access to 900k+ models, datasets, papers
2. Dataset Viewer - Browse HF datasets without download
3. Data Exploration - Interactive analysis with pandas, statistical tools (343⭐ GitHub project)
4. MCP Pandas - Robust pandas analysis with Docker runtime
5. DataHub MCP - Metadata, lineage, governance (LinkedIn/Acryl Data)
6. Penrose MCP - Mathematical diagram generation
7. Government Data MPs - Data.gov, Treasury Data, NASA CMR for climate/gov data

## Technical Requirements

### Infrastructure
- Kubernetes cluster (v1.19+) with ToolHive operator installed
- NGINX Ingress Controller for external access
- Cert-Manager for TLS certificate management
- Sealed Secrets or External Secrets Operator for secure credential management
- Prometheus and Grafana for monitoring
- Jaeger for distributed tracing

### ToolHive Operator Features Utilization
- MCPServer CRDs with streamable-http transport for efficient communication
- Custom resource definitions for declarative infrastructure management
- Built-in observability with OpenTelemetry integration
- Network policy enforcement for security isolation
- Resource management with requests/limits and HPA
- Secret management with Kubernetes Secrets mounting
- Health checks with liveness/readiness probes
- Security context with non-root execution and capability dropping

### Security Implementation
- Network policies restricting ingress to ToolHive operator namespace and authorized clients
- Egress rules allowing only necessary external connections (HTTP/HTTPS)
- Service accounts with RBAC for minimal privilege access
- Sealed secrets for HF_TOKEN and DataHub credentials
- Non-root container execution with dropped capabilities
- TLS encryption for all external communications

### Observability Configuration
- Prometheus metrics endpoint exposure with custom dashboards
- Jaeger distributed tracing for request flow visualization
- Health check endpoints for all MCP services
- Resource utilization monitoring with alerts
- Custom metrics for MCP tool invocation tracking

## Implementation Approach

### Phase 1: Environment Setup
1. Deploy ToolHive operator CRDs and controller in toolhive-system namespace
2. Configure ingress controller with TLS certificates
3. Set up monitoring stack (Prometheus, Grafana, Jaeger)
4. Create dedicated data-science namespace with RBAC
5. Configure sealed secrets for credential management

### Phase 2: MCP Server Deployment
1. Deploy all seven MCP servers as MCPServer CRDs with:
   - Appropriate resource requests/limits (2-8GB RAM, 1-4 CPU)
   - Streamable HTTP transport for efficient proxying
   - Health check endpoints (/health)
   - Security contexts with non-root execution
   - Volume mounts for temporary data storage
   - Environment variable configuration for API keys
2. Configure Kubernetes Services for each MCP server
3. Set up network policies for service isolation
4. Implement horizontal pod autoscaling for high-concurrency services

### Phase 3: Access Configuration
1. Configure NGINX ingress with path-based routing:
   - /huggingface → huggingface-mcp:8001
   - /data-explorer → data-exploration:8003
   - /pandas → mcp-pandas:8004
   - /datahub → datahub-mcp:8005
   - /penrose → penrose-mcp:8006
   - /gov-data → gov-data-mcp:8007
   - /dataset-viewer → dataset-viewer-mcp:8008
2. Configure TLS with Let's Encrypt certificates
3. We have a custom domain name with GoDaddy: finetunings.ai
   - Set up custom domain names for external access (finetunings.ai and *.finetunings.ai)
   - I may migrate to Cloudflare soon-ish
4. Configure proxy timeouts for long-running operations (3600 seconds for read/send timeouts)
5. Add routes to output paths for services that produce output:
   - Penrose: SVG output via /penrose path
   - Future extensions for matplotlib output from pandas or data exploration services

### Phase 4: Integration Testing
1. Verify all MCP servers are accessible via ingress
2. Test health check endpoints for each service
3. Validate network policy isolation
4. Confirm monitoring metrics are collected
5. Test tracing integration with sample requests

### Phase 5: Client Integration
1. Configure Claude Desktop with MCP server connections
2. Test data discovery workflows (Hugging Face → Dataset Viewer)
3. Validate analysis pipelines (Data Exploration → Pandas)
4. Verify governance integration (DataHub metadata tracking)
5. Test visualization outputs (Penrose diagram generation)

## Expected Outcomes
- Fully functional data science toolchain with declarative Kubernetes management
- Production-ready security with network isolation and credential protection
- Comprehensive observability with monitoring, alerting, and tracing
- Scalable architecture supporting high-concurrency workloads
- Integrated workflows for climate research, AI model development, and policy analysis
- Unified interface for data sourcing, analysis, and visualization through MCP protocol