# Phase 3: Access Configuration Implementation Plan

## Overview
Configure NGINX ingress with path-based routing, TLS certificates, custom domain names, and proxy timeouts for all MCP services in the data science toolset.

## 1. Ingress Controller Deployment

### Prerequisites
- Kubernetes cluster with ToolHive operator deployed
- cert-manager installed for TLS certificate management
- metallb or similar load balancer for bare metal clusters

### Commands

```bash
# Apply the ingress controller configuration
kubectl apply -f examples/data-science-toolset/k8s/ingress.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

### Verification
```bash
# Check ingress controller pods
kubectl get pods -n ingress-nginx

# Check ingress controller services
kubectl get svc -n ingress-nginx

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

## 2. TLS Certificate Configuration

### Prerequisites
- cert-manager installed in the cluster
- Domain name (finetunings.ai) configured to point to the cluster's load balancer IP

### Commands

```bash
# Apply the certificate configuration
kubectl apply -f examples/data-science-toolset/k8s/data-science-ingress.yaml

# Check certificate status
kubectl get certificates -n data-science
kubectl get certificaterequests -n data-science
```

### Configuration Details
The certificate configuration includes:
- Self-signed issuer for development/testing
- TLS certificate for finetunings.ai and *.finetunings.ai
- Automatic certificate renewal

## 3. Path-Based Routing Configuration

### Configuration Details
The ingress configuration provides path-based routing for all MCP services:

- `/huggingface` → huggingface-mcp:8001
- `/data-explorer` → data-exploration:8003
- `/pandas` → mcp-pandas:8004
- `/datahub` → datahub-mcp:8005
- `/penrose` → penrose-mcp:8006
- `/gov-data` → gov-data-mcp:8007
- `/dataset-viewer` → dataset-viewer-mcp:8008

### Commands

```bash
# Apply the ingress configuration
kubectl apply -f examples/data-science-toolset/k8s/data-science-ingress.yaml

# Check ingress status
kubectl get ingress -n data-science

# Describe ingress for detailed information
kubectl describe ingress data-science-ingress -n data-science
```

## 4. Custom Domain Configuration

### Configuration Details
The ingress is configured to handle requests for:
- `finetunings.ai` (primary domain)
- `*.finetunings.ai` (wildcard subdomains)

### DNS Configuration
To use the custom domain, configure DNS to point:
- `finetunings.ai` to the load balancer IP
- `*.finetunings.ai` to the load balancer IP

The load balancer IP can be found with:
```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

## 5. Proxy Timeout Configuration

### Configuration Details
The ingress configuration includes timeout settings for long-running operations:
- `proxy-read-timeout`: 3600 seconds (1 hour)
- `proxy-send-timeout`: 3600 seconds (1 hour)

These settings ensure that long-running MCP operations don't timeout prematurely.

## 6. Output Path Routing

### Configuration Details
The ingress configuration includes routes for services that produce output:

- `/penrose` → penrose-mcp:8006 (for SVG output)
- Future extensions for matplotlib output from pandas or data exploration services

## 7. Verification Commands

```bash
# Check all ingress resources
kubectl get ingress -A

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller

# Test connectivity to each service
curl -k https://finetunings.ai/huggingface/health
curl -k https://finetunings.ai/data-explorer/health
curl -k https://finetunings.ai/pandas/health
curl -k https://finetunings.ai/datahub/health
curl -k https://finetunings.ai/penrose/health
curl -k https://finetunings.ai/gov-data/health
curl -k https://finetunings.ai/dataset-viewer/health

# Test wildcard domain
curl -k https://test.finetunings.ai/huggingface/health
```

## 8. Troubleshooting

### Common Issues

1. **Certificate Not Ready**
   ```bash
   kubectl describe certificate toolhive-data-science-cert -n data-science
   kubectl describe certificaterequest -n data-science
   ```

2. **Ingress Not Routing**
   ```bash
   kubectl describe ingress data-science-ingress -n data-science
   kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
   ```

3. **Service Unreachable**
   ```bash
   kubectl get endpoints -n data-science
   kubectl describe service huggingface-mcp -n data-science
   ```

### Logs and Debugging
```bash
# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --follow

# Check cert-manager logs
kubectl logs -n cert-manager -l app.kubernetes.io/component=controller
```

## Next Steps
After completing Phase 3, proceed to Phase 4: Integration Testing by verifying all MCP servers are accessible via ingress and testing health check endpoints.