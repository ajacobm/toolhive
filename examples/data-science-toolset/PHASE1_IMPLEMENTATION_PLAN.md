# Phase 1: Environment Setup Implementation Plan

## 1. Deploy ToolHive Operator CRDs and Controller

### Prerequisites
- Helm 3.10+ installed
- Kind cluster created and running
- Kubectl configured to use the correct context

### Commands

```bash
# Navigate to the ToolHive repository root
cd /home/adam/GitHub/toolhive

# Install CRDs into the K8s cluster
helm upgrade --install toolhive-operator-crds deploy/charts/operator-crds --namespace toolhive-system --create-namespace

# Deploy the operator into the K8s cluster
helm upgrade --install toolhive-operator deploy/charts/operator --namespace toolhive-system --create-namespace
```

### Verification
```bash
# Check CRDs are installed
kubectl get crds | grep toolhive

# Check operator deployment
kubectl get pods -n toolhive-system

# Check operator logs
kubectl logs -n toolhive-system -l app.kubernetes.io/name=toolhive-operator
```

## 2. Configure Ingress Controller with TLS Certificates

### Prerequisites
- ToolHive operator deployed
- Kubernetes cluster with networking capabilities

### Commands

```bash
# Apply the ingress controller configuration from the data science toolset
kubectl apply -f examples/data-science-toolset/k8s/ingress.yaml
```

### TLS Configuration
The ingress configuration includes:
- Cert-manager integration for automated certificate management
- LetsEncrypt staging/production issuers
- TLS annotations for automatic certificate provisioning

### Verification
```bash
# Check ingress controller status
kubectl get pods -n ingress-nginx

# Check certificate issuance
kubectl get certificates -n data-science
kubectl get certificaterequests -n data-science
```

## 3. Set Up Monitoring Stack (Prometheus, Grafana, Jaeger)

### Prerequisites
- Helm 3.10+ installed
- Kubernetes cluster with sufficient resources

### Commands

```bash
# Add required Helm repositories
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install Jaeger for distributed tracing
helm upgrade -i jaeger-all-in-one jaegertracing/jaeger \
  -f examples/otel/jaeger-values.yaml \
  -n monitoring

# Install Prometheus and Grafana stack
helm upgrade -i kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -f examples/otel/prometheus-stack-values.yaml \
  -n monitoring

# Install OpenTelemetry collector
helm upgrade -i otel-collector open-telemetry/opentelemetry-collector \
  -f examples/otel/otel-values.yaml \
  -n monitoring
```

### Verification
```bash
# Check monitoring components
kubectl get pods -n monitoring

# Check services
kubectl get svc -n monitoring

# Port-forward to access dashboards
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
kubectl port-forward -n monitoring svc/jaeger-all-in-one-query 16686:16686
```

## 4. Create Dedicated Data-Science Namespace with RBAC

### Prerequisites
- Kubernetes cluster with RBAC enabled
- kubectl configured to use the correct context

### Commands

```bash
# Apply the namespace and RBAC configuration from the data science toolset
kubectl apply -f examples/data-science-toolset/k8s/namespace.yaml
```

### Configuration Details
The namespace.yaml includes:
- Namespace creation for data-science workloads
- Service accounts for MCP servers
- Role-based access control for secure operations
- Resource quotas and limits for resource management
- Network policies for secure communication

### Verification
```bash
# Check namespace creation
kubectl get namespaces | grep data-science

# Check service accounts
kubectl get serviceaccounts -n data-science

# Check RBAC resources
kubectl get roles,rolebindings -n data-science
```

## 5. Configure Sealed Secrets for Credential Management

### Prerequisites
- Kubernetes cluster
- Helm 3.10+ installed
- kubeseal CLI tool installed

### Commands

```bash
# Install sealed-secrets controller (not part of ToolHive repo manifests)
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets -n kube-system

# Wait for controller to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=sealed-secrets -n kube-system --timeout=120s

# Create and seal secrets for Hugging Face credentials
echo -n "hf_xxxx" | kubectl create secret generic hf-credentials \
  --dry-run=client \
  --from-file=token=/dev/stdin \
  -o yaml | kubeseal > examples/data-science-toolset/k8s/sealed-hf-credentials.yaml

# Apply sealed secrets to the data-science namespace
kubectl apply -f examples/data-science-toolset/k8s/sealed-hf-credentials.yaml -n data-science

# Create and seal secrets for DataHub credentials (if needed)
echo -n "datahub_token" | kubectl create secret generic datahub-credentials \
  --dry-run=client \
  --from-file=token=/dev/stdin \
  -o yaml | kubeseal > examples/data-science-toolset/k8s/sealed-datahub-credentials.yaml

# Apply sealed secrets to the data-science namespace
kubectl apply -f examples/data-science-toolset/k8s/sealed-datahub-credentials.yaml -n data-science
```

### Verification
```bash
# Check sealed secrets controller
kubectl get pods -n kube-system | grep sealed-secrets

# Check sealed secrets
kubectl get sealedsecrets -n data-science

# Check that secrets were unsealed (may need to wait a moment)
kubectl get secrets -n data-science | grep credentials
```

## Next Steps
After completing Phase 1, proceed to Phase 2: MCP Server Deployment by applying the MCPServer CRDs from the examples/data-science-toolset/k8s/mcpservers/ directory.