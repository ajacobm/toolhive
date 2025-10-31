# Visualization Architecture: Unified Web Interface

## Problem Statement

Multiple MCP servers generate different types of visual output:
- **Data Exploration**: Generates Claude artifacts (HTML/charts within chat)
- **Penrose MCP**: Generates SVG diagrams (stored to disk)
- **Future**: Code-graph-mcp Vue.js frontend (complex visualizations)

Current approach: Each has separate output store. Goal: Unified NGINX gateway with virtual paths.

---

## Architecture

### Option 1: Docker Compose (Current) - Local Development

```
┌────────────────────────────────────────────────────┐
│  Host Machine                                      │
│                                                    │
│  ├─ ~/.penrose/              (SVG output)         │
│  ├─ ~/.data-explorer/        (HTML artifacts)     │
│  └─ ~/.code-graph/           (Vue.js app)         │
│                                                    │
│  Docker network: data-science                     │
│  ├─ NGINX (8888)             ← Single gateway    │
│  │   ├─ /penrose   → ~/penrose/                  │
│  │   ├─ /explorer  → ~/data-explorer/            │
│  │   ├─ /code-graph → ~/code-graph/              │
│  │   ├─ /health    → Status page                 │
│  │   └─ /           → Dashboard                   │
│  │                                                │
│  ├─ Penrose MCP (8008)       → /penrose          │
│  ├─ Data Exploration (8003)  → generates artifacts
│  ├─ Code-graph MCP (8009)    → /code-graph       │
│  └─ ...other MCPs...                             │
│                                                    │
└────────────────────────────────────────────────────┘

Access: http://localhost:8888
  - List: http://localhost:8888
  - View diagrams: http://localhost:8888/penrose
  - View explorer artifacts: http://localhost:8888/explorer
  - View code graphs: http://localhost:8888/code-graph
```

### Option 2: Kubernetes + Ingress (Production)

```
┌──────────────────────────────────────────────────┐
│  Kubernetes Cluster                              │
│                                                  │
│  ┌──────────────────────────────────────────┐  │
│  │ Ingress Controller (NGINX)               │  │
│  │ mcp.example.com → 443 (TLS)              │  │
│  │                                          │  │
│  │ ├─ /penrose     → penrose-viewer pod    │  │
│  │ ├─ /explorer    → explorer-viewer pod   │  │
│  │ ├─ /code-graph  → code-graph-frontend   │  │
│  │ ├─ /api/*       → MCP services          │  │
│  │ └─ /            → Dashboard             │  │
│  └──────────────────────────────────────────┘  │
│           ↓                ↓          ↓        │
│  ┌─────────────┐  ┌──────────────┐  ┌────────┐│
│  │ Penrose     │  │ Data         │  │ Code   ││
│  │ Viewer Pod  │  │ Explorer Pod │  │ Graph  ││
│  │ (NGINX)     │  │ (NGINX)      │  │ Vue    ││
│  │ - read PVC  │  │ - read PVC   │  │ Frontend
│  │ - mount     │  │ - mount      │  │        ││
│  │   ~/.penrose│  │   ~/.explorer│  │        ││
│  └─────────────┘  └──────────────┘  └────────┘│
│           ↓                ↓          ↓        │
│  ┌─────────────┐  ┌──────────────┐  ┌────────┐│
│  │ Penrose MCP │  │ Data Explor. │  │ Code   ││
│  │ Pod         │  │ Pod          │  │ Graph  ││
│  │ write SVG   │  │ write HTML   │  │ MCP    ││
│  │ to PVC      │  │ to PVC       │  │ Pod    ││
│  └─────────────┘  └──────────────┘  └────────┘│
│                                                  │
│  Persistent Volumes:                            │
│  ├─ penrose-pvc   (~/.penrose)                 │
│  ├─ explorer-pvc  (~/.data-explorer)           │
│  └─ code-graph-pvc (~/.code-graph)             │
│                                                  │
└──────────────────────────────────────────────────┘
```

---

## Implementation: Docker Compose (Immediate)

### 1. Updated docker-compose.minimal.yml

```yaml
services:
  # Writer: Penrose MCP
  penrose-mcp:
    image: bmorphism/penrose-mcp:latest
    volumes:
      - ~/.penrose:/work/diagrams

  # Writer: Data Exploration 
  data-exploration:
    image: mcp-data-exploration:latest
    volumes:
      - ~/.data-explorer:/app/artifacts

  # Gateway: Unified NGINX
  nginx-gateway:
    image: nginx:alpine
    ports:
      - "8888:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ~/.penrose:/srv/penrose:ro
      - ~/.data-explorer:/srv/explorer:ro
      - ./web-dashboard:/srv/www:ro
    depends_on:
      - penrose-mcp
      - data-exploration
```

### 2. nginx.conf (Smart Gateway)

```nginx
http {
  # Upstream services
  upstream penrose_api {
    server penrose-mcp:8008;
  }

  upstream explorer_api {
    server data-exploration:8003;
  }

  # Main server
  server {
    listen 80;
    server_name localhost;

    # Root: Dashboard
    location / {
      root /srv/www;
      try_files $uri $uri/ /index.html;
    }

    # Penrose SVG store
    location /penrose {
      alias /srv/penrose;
      autoindex on;
      autoindex_format json;
      
      location ~ \.svg$ {
        add_header Content-Type image/svg+xml;
        expires 1h;
      }
    }

    # Data Explorer artifacts
    location /explorer {
      alias /srv/explorer;
      autoindex on;
      
      location ~ \.html$ {
        add_header Content-Type text/html;
        expires 1h;
      }
    }

    # Code-graph frontend (when available)
    location /code-graph {
      alias /srv/code-graph;
      try_files $uri $uri/ /index.html;
    }

    # API proxies (optional: for direct access)
    location /api/penrose {
      proxy_pass http://penrose_api;
    }

    location /api/explorer {
      proxy_pass http://explorer_api;
    }

    # Health check
    location /health {
      access_log off;
      return 200 "Gateway OK\n";
      add_header Content-Type text/plain;
    }

    # Status dashboard
    location /status {
      default_type application/json;
      return 200 '{
        "nginx": "ok",
        "penrose": "pending",
        "explorer": "pending"
      }';
    }
  }
}
```

### 3. Web Dashboard (index.html)

```html
<!DOCTYPE html>
<html>
<head>
  <title>MCP Visualization Gateway</title>
  <style>
    body { font-family: monospace; margin: 2rem; background: #1e1e1e; color: #e0e0e0; }
    .container { max-width: 1200px; margin: 0 auto; }
    h1 { color: #4ec9b0; }
    .card { 
      border: 1px solid #444; 
      padding: 1rem; 
      margin: 1rem 0; 
      border-radius: 4px;
      background: #252526;
    }
    .card h3 { color: #569cd6; margin-top: 0; }
    a { color: #ce9178; text-decoration: none; }
    a:hover { text-decoration: underline; }
    .status { font-size: 0.8rem; color: #858585; }
  </style>
</head>
<body>
  <div class="container">
    <h1>MCP Visualization Gateway</h1>
    <p>Unified access to all MCP server outputs</p>

    <div class="card">
      <h3>Penrose Diagrams</h3>
      <p><a href="/penrose/">/penrose</a> - Mathematical SVG diagrams</p>
      <p class="status">Auto-generated by Penrose MCP, stored as .svg files</p>
    </div>

    <div class="card">
      <h3>Data Explorer Artifacts</h3>
      <p><a href="/explorer/">/explorer</a> - Analysis HTML & charts</p>
      <p class="status">Generated from CSV analysis, includes visualizations</p>
    </div>

    <div class="card">
      <h3>Code Graph Visualizations</h3>
      <p><a href="/code-graph/">/code-graph</a> - Interactive code structure (coming soon)</p>
      <p class="status">Vue.js frontend for code dependency graphs</p>
    </div>

    <div class="card">
      <h3>API Access</h3>
      <p>Direct MCP access (for debugging):</p>
      <ul>
        <li><a href="/api/penrose">/api/penrose</a> - Penrose API</li>
        <li><a href="/api/explorer">/api/explorer</a> - Data Explorer API</li>
      </ul>
    </div>

    <div class="card">
      <h3>Health</h3>
      <p><a href="/health">/health</a> | <a href="/status">/status</a></p>
    </div>
  </div>
</body>
</html>
```

---

## Storage Layers

### Docker Compose

```
Host Machine:
├─ ~/.penrose/                    (shared volume)
│   ├─ diagram_1.svg
│   ├─ diagram_2.svg
│   └─ index.html (auto-generated)
│
├─ ~/.data-explorer/              (shared volume)
│   ├─ analysis_2024-10-31.html
│   ├─ chart_correlation.svg
│   └─ index.html (auto-generated)
│
└─ ~/.code-graph/                 (for future)
    ├─ graph.json
    ├─ index.html (Vue app)
    └─ styles.css
```

### Kubernetes

```yaml
# Persistent Volumes
penrose-pvc:
  capacity: 10Gi
  mountPath: /data/penrose

explorer-pvc:
  capacity: 10Gi
  mountPath: /data/explorer

code-graph-pvc:
  capacity: 5Gi
  mountPath: /data/code-graph

# Viewers (read-only mount PVC, serve via NGINX)
# Writers (MCPs, write to PVC)
```

---

## How Data Flows

### Current (Separate):
```
Penrose MCP → ~/.penrose/ → NGINX /penrose → Browser
Data Explor → Claude Chat → Manual copy?
```

### Improved (Unified):
```
Penrose MCP → ~/.penrose/ ──┐
                             ├─ NGINX Gateway (8888)
Data Explorer → artifacts ──┤  ├─ /penrose
                             ├─ /explorer
Code-graph → Vue.js ────────┤  └─ /code-graph
                             │
                             Browser (single port)
```

---

## Implementation Steps (No Rabbit Hole)

### Phase 1: Docker Compose (Today)
1. Create `nginx.conf` (route /penrose, /explorer, placeholder /code-graph)
2. Create `web-dashboard/index.html` (simple directory listing)
3. Update `docker-compose.minimal.yml` to use unified NGINX
4. Add volume mounts for ~/.data-explorer/ (for future Data Explorer output)
5. Test: http://localhost:8888

### Phase 2: K8s Adaptation (Optional)
1. Separate Viewer pods (read-only NGINX + PVC mount)
2. Ingress controller routes /penrose, /explorer, /code-graph
3. Same PVC volumes, multi-reader pattern

### Phase 3: Code-Graph Integration (Future)
1. code-graph-mcp outputs JSON to ~/.code-graph/
2. Add Vue.js frontend to serve from /code-graph
3. Update NGINX to reverse-proxy Vue app

---

## K8s Example (If You Want)

```yaml
# Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mcp-visualization
  namespace: data-science
spec:
  rules:
    - host: mcp.local
      http:
        paths:
          - path: /penrose
            backend:
              service:
                name: penrose-viewer
                port: 80
          - path: /explorer
            backend:
              service:
                name: explorer-viewer
                port: 80

# Viewer Service (read-only)
apiVersion: v1
kind: Service
metadata:
  name: penrose-viewer
spec:
  selector:
    app: penrose-viewer
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: penrose-viewer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: penrose-viewer
  template:
    metadata:
      labels:
        app: penrose-viewer
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: penrose-data
          mountPath: /srv/penrose
          readOnly: true
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          readOnly: true
      volumes:
      - name: penrose-data
        persistentVolumeClaim:
          claimName: penrose-pvc
      - name: nginx-config
        configMap:
          name: viewer-nginx-config
```

---

## Decision: Which Path?

### Keep Simple (Recommended for Now)
- **Docker Compose**: Update `docker-compose.minimal.yml` with unified NGINX
- **Single port**: http://localhost:8888
- **Virtual paths**: /penrose, /explorer, /code-graph (placeholder)
- **Extensible**: Easy to add new services later

### Go Full K8s
- Use KinD cluster you have enabled
- Deploy to test multi-service pattern
- Real Ingress controller
- PVC pattern (same volumes, multi-reader)

---

## TL;DR

**Simplest approach** (Docker Compose):
1. Create `nginx.conf` with `/penrose`, `/explorer`, `/code-graph` routes
2. Create `web-dashboard/index.html` (navigation)
3. Update compose: NGINX on 8888, route to all viewers
4. Access everything via `http://localhost:8888`

**Bonus** (K8s):
- Same pattern with Ingress + viewer pods
- Real production setup
- Ready to scale

Which would you prefer to implement?
