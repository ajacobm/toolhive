# Phase 2: MCP Server Deployment Implementation Plan

## Overview
Deploy all seven MCP servers as MCPServer CRDs with appropriate configurations, services, network policies, and horizontal pod autoscaling.

## 1. MCP Server Deployments

### 1.1 Hugging Face MCP Server
File: `examples/data-science-toolset/k8s/mcpservers/01-huggingface-mcp.yaml`

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: huggingface-mcp
  namespace: data-science
  labels:
    mcp-toolset: "true"
    discovery: "true"
spec:
  image: shreyas/huggingface-mcp-server:latest
  imagePullPolicy: IfNotPresent
  transport:
    type: streamable-http
  env:
    - name: HF_TOKEN
      valueFrom:
        secretKeyRef:
          name: hf-credentials
          key: token
          optional: true
  ports:
    - name: http
      containerPort: 8001
      protocol: TCP
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "8Gi"
      cpu: "4000m"
  livenessProbe:
    httpGet:
      path: /health
      port: 8001
    initialDelaySeconds: 30
    periodSeconds: 30
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: 8001
    initialDelaySeconds: 10
    periodSeconds: 10
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    capabilities:
      drop:
        - ALL
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchExpressions:
                - key: app
                  operator: In
                  values:
                    - huggingface-mcp
            topologyKey: kubernetes.io/hostname
```

### 1.2 Data Exploration MCP Server
File: `examples/data-science-toolset/k8s/mcpservers/02-data-exploration.yaml`

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: data-exploration
  namespace: data-science
  labels:
    mcp-toolset: "true"
    analysis: "true"
spec:
  image: reading-plus/mcp-server-data-exploration:latest
  imagePullPolicy: IfNotPresent
  transport:
    type: streamable-http
  ports:
    - name: http
      containerPort: 8003
      protocol: TCP
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "8Gi"
      cpu: "4000m"
  livenessProbe:
    httpGet:
      path: /health
      port: 8003
    initialDelaySeconds: 30
    periodSeconds: 30
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: 8003
    initialDelaySeconds: 15
    periodSeconds: 15
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    capabilities:
      drop:
        - ALL
  volumeMounts:
    - name: work
      mountPath: /tmp/data-explorer
  volumes:
    - name: work
      emptyDir: {}
```

### 1.3 MCP Pandas Server
File: `examples/data-science-toolset/k8s/mcpservers/03-mcp-pandas.yaml`

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: mcp-pandas
  namespace: data-science
  labels:
    mcp-toolset: "true"
    analysis: "true"
spec:
  image: ghcr.io/stacklok/mcp-pandas:latest
  imagePullPolicy: IfNotPresent
  transport:
    type: streamable-http
  ports:
    - name: http
      containerPort: 8004
      protocol: TCP
  resources:
    requests:
      memory: "2Gi"
      cpu: "1000m"
    limits:
      memory: "8Gi"
      cpu: "4000m"
  livenessProbe:
    httpGet:
      path: /health
      port: 8004
    initialDelaySeconds: 30
    periodSeconds: 30
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: 8004
    initialDelaySeconds: 15
    periodSeconds: 15
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    capabilities:
      drop:
        - ALL
  volumeMounts:
    - name: work
      mountPath: /tmp/pandas
  volumes:
    - name: work
      emptyDir: {}
```

### 1.4 DataHub MCP Server
File: `examples/data-science-toolset/k8s/mcpservers/04-datahub.yaml`

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: datahub-mcp
  namespace: data-science
  labels:
    mcp-toolset: "true"
    governance: "true"
spec:
  image: ghcr.io/stacklok/datahub-mcp:latest
  imagePullPolicy: IfNotPresent
  transport:
    type: streamable-http
  env:
    - name: DATAHUB_TOKEN
      valueFrom:
        secretKeyRef:
          name: datahub-credentials
          key: token
          optional: true
  ports:
    - name: http
      containerPort: 8005
      protocol: TCP
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  livenessProbe:
    httpGet:
      path: /health
      port: 8005
    initialDelaySeconds: 30
    periodSeconds: 30
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: 8005
    initialDelaySeconds: 15
    periodSeconds: 15
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    capabilities:
      drop:
        - ALL
```

### 1.5 Penrose MCP Server
File: `examples/data-science-toolset/k8s/mcpservers/05-penrose-mcp.yaml`

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: penrose-mcp
  namespace: data-science
  labels:
    mcp-toolset: "true"
    visualization: "true"
spec:
  image: ghcr.io/stacklok/penrose-mcp:latest
  imagePullPolicy: IfNotPresent
  transport:
    type: streamable-http
  ports:
    - name: http
      containerPort: 8006
      protocol: TCP
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  livenessProbe:
    httpGet:
      path: /health
      port: 8006
    initialDelaySeconds: 30
    periodSeconds: 30
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: 8006
    initialDelaySeconds: 15
    periodSeconds: 15
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    capabilities:
      drop:
        - ALL
  volumeMounts:
    - name: work
      mountPath: /tmp/penrose
  volumes:
    - name: work
      emptyDir: {}
```

### 1.6 Government Data MCP Server
File: `examples/data-science-toolset/k8s/mcpservers/06-gov-data-mcp.yaml`

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: gov-data-mcp
  namespace: data-science
  labels:
    mcp-toolset: "true"
    discovery: "true"
spec:
  image: ghcr.io/stacklok/gov-data-mcp:latest
  imagePullPolicy: IfNotPresent
  transport:
    type: streamable-http
  ports:
    - name: http
      containerPort: 8007
      protocol: TCP
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  livenessProbe:
    httpGet:
      path: /health
      port: 8007
    initialDelaySeconds: 30
    periodSeconds: 30
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: 8007
    initialDelaySeconds: 15
    periodSeconds: 15
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    capabilities:
      drop:
        - ALL
```

### 1.7 Dataset Viewer MCP Server
File: `examples/data-science-toolset/k8s/mcpservers/07-dataset-viewer-mcp.yaml`

```yaml
apiVersion: toolhive.stacklok.dev/v1alpha1
kind: MCPServer
metadata:
  name: dataset-viewer-mcp
  namespace: data-science
  labels:
    mcp-toolset: "true"
    discovery: "true"
spec:
  image: ghcr.io/stacklok/dataset-viewer-mcp:latest
  imagePullPolicy: IfNotPresent
  transport:
    type: streamable-http
  ports:
    - name: http
      containerPort: 8008
      protocol: TCP
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "4Gi"
      cpu: "2000m"
  livenessProbe:
    httpGet:
      path: /health
      port: 8008
    initialDelaySeconds: 30
    periodSeconds: 30
    failureThreshold: 3
  readinessProbe:
    httpGet:
      path: /health
      port: 8008
    initialDelaySeconds: 15
    periodSeconds: 15
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    capabilities:
      drop:
        - ALL
```

## 2. Kubernetes Services Configuration

Each MCP server will have a corresponding service defined in the same YAML file as the MCPServer CRD.

Example service configuration:
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: huggingface-mcp
  namespace: data-science
  labels:
    mcp-toolset: "true"
spec:
  type: ClusterIP
  selector:
    app: huggingface-mcp
  ports:
    - name: http
      port: 8001
      targetPort: 8001
      protocol: TCP
```

## 3. Network Policies for Service Isolation

File: `examples/data-science-toolset/k8s/network-policies.yaml`

```yaml
apiVersion: batch/v1
kind: NetworkPolicy
metadata:
  name: mcp-server-isolation
  namespace: data-science
spec:
  podSelector:
    matchLabels:
      mcp-toolset: "true"
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: toolhive-system
    ports:
    - protocol: TCP
      port: 8001
    - protocol: TCP
      port: 8003
    - protocol: TCP
      port: 8004
    - protocol: TCP
      port: 8005
    - protocol: TCP
      port: 8006
    - protocol: TCP
      port: 8007
    - protocol: TCP
      port: 8008
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
```

## 4. Horizontal Pod Autoscaling

File: `examples/data-science-toolset/k8s/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: huggingface-mcp-hpa
  namespace: data-science
spec:
  scaleTargetRef:
    apiVersion: toolhive.stacklok.dev/v1alpha1
    kind: MCPServer
    name: huggingface-mcp
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

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: data-exploration-hpa
  namespace: data-science
spec:
  scaleTargetRef:
    apiVersion: toolhive.stacklok.dev/v1alpha1
    kind: MCPServer
    name: data-exploration
  minReplicas: 1
  maxReplicas: 5
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

## 5. Deployment Commands

```bash
# Apply all MCP server configurations
kubectl apply -f examples/data-science-toolset/k8s/mcpservers/

# Apply network policies
kubectl apply -f examples/data-science-toolset/k8s/network-policies.yaml

# Apply horizontal pod autoscaling
kubectl apply -f examples/data-science-toolset/k8s/hpa.yaml
```

## 6. Verification Commands

```bash
# Check MCP server deployments
kubectl get mcpservers -n data-science

# Check services
kubectl get services -n data-science

# Check pods
kubectl get pods -n data-science

# Check network policies
kubectl get networkpolicies -n data-science

# Check horizontal pod autoscalers
kubectl get hpa -n data-science

# Check MCP server status
kubectl describe mcpservers -n data-science

# Check logs for a specific MCP server
kubectl logs -n data-science -l app=huggingface-mcp
```