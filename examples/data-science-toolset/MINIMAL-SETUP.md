# Data Science MCP: Minimal Setup

## 5-Minute Local Test

### Prerequisites
```bash
# Verify HF token is loaded
echo $HF_TOKEN
# Should output: hf_HpPhCsZ...
```

### Start Services
```bash
source ~/.bashrc
mcp-compose up -f docker-compose.minimal.yml
```

### Services Running
| Port | Service | Purpose |
|------|---------|---------|
| 8002 | Dataset Viewer | Browse Hugging Face datasets |
| 8003 | Data Exploration | Full data science workflows |
| 8006 | Data.gov | US federal datasets |
| 8007 | Treasury Data | Economic data API |
| 8008 | Penrose MCP | Mathematical diagram generator |
| 8888 | Penrose Viewer | View SVG diagrams (web UI) |

### Quick Health Checks
```bash
curl http://localhost:8003/health  # Data Exploration
curl http://localhost:8002/health  # Dataset Viewer
curl http://localhost:8006/health  # Data.gov
curl http://localhost:8007/health  # Treasury
```

### View Penrose Diagrams
1. Open http://localhost:8888 in browser
2. Generated SVG files appear in ~/.penrose/
3. Penrose MCP stores output there automatically

### Connect to Claude
Add to `~/.config/Claude/claude_desktop_config.json`:
```json
{
  "mcpServers": {
    "data-explorer": {
      "command": "curl",
      "args": ["http://localhost:8003"]
    },
    "dataset-viewer": {
      "command": "curl",
      "args": ["http://localhost:8002"]
    },
    "penrose": {
      "command": "curl",
      "args": ["http://localhost:8008"]
    }
  }
}
```

## Essential Nice-to-Haves

### 1. Log Monitoring
```bash
# Watch all services
mcp-compose logs -f

# Watch specific service
mcp-compose logs -f data-exploration

# Last 100 lines
mcp-compose logs --tail 100
```

### 2. Environment Verification
```bash
# Check loaded environment
set -a; source ~/.envs; set +a
env | grep -E "HF_TOKEN|DATAHUB"
```

### 3. Storage Verification
```bash
# Penrose SVG store
ls -lah ~/.penrose/
du -sh ~/.penrose/

# Data Explorer temp
ls -lah /tmp/data-explorer/
du -sh /tmp/data-explorer/
```

### 4. Service Performance
```bash
# Check container resource usage
docker stats --no-stream

# Full compose status
mcp-compose ps
```

### 5. Clean Shutdown
```bash
# Stop all services
mcp-compose down

# Remove volumes (cleanup SVGs, temp files)
mcp-compose down -v
```

## Example Workflows

### Workflow 1: Treasury Trend Analysis (5 min)
```
1. Ask Claude: "Get treasury spending by department for last 30 days"
   → Treasury Data MCP retrieves data
2. Ask: "Analyze spending patterns"
   → Data Exploration loads & analyzes
3. Ask: "Show me a diagram of government structure"
   → Penrose creates diagram → View at http://localhost:8888
```

### Workflow 2: Dataset Exploration (10 min)
```
1. "Search for time series datasets on Hugging Face"
   → Dataset Viewer searches & lists
2. "Show first 100 rows without downloading"
   → Dataset Viewer retrieves preview
3. "Analyze data types and missing values"
   → Data Exploration profiles dataset
```

### Workflow 3: Government Data Research (15 min)
```
1. "Find climate-related datasets on Data.gov"
   → Data.gov MCP searches
2. "Search for EPA water quality data"
   → Data.gov API queries
3. "Load the most recent dataset"
   → Data Exploration processes
4. "Create a visualization of pollution trends"
   → Penrose generates diagram
```

## Troubleshooting

### Services won't start
```bash
# Check Docker running
docker ps

# Check docker-compose syntax
mcp-compose config

# See full startup logs
mcp-compose up (no -d flag)
```

### HF Token not loading
```bash
# Debug environment loading
bash -c 'set -a; source ~/.envs; set +a; echo $HF_TOKEN'

# Add explicit debug
mcp-compose ps
# Check penrose-mcp and dataset-viewer env vars
```

### Penrose viewer not loading
```bash
# Check NGINX is running
docker ps | grep penrose-viewer

# Check SVG files exist
ls ~/.penrose/

# View NGINX logs
docker logs penrose-viewer

# Check volume mount
docker inspect penrose-viewer | grep -A 5 "Mounts"
```

### Port conflicts
```bash
# Find process using port
lsof -i :8003
lsof -i :8888

# Kill if needed
kill -9 <PID>
```

## Helper Command Reference

```bash
# Start minimal setup
mcp-compose up

# Follow logs
mcp-compose logs -f

# List services
mcp-compose ps

# Execute command in container
mcp-compose exec data-exploration bash

# Stop all
mcp-compose down

# Full cleanup (remove volumes)
mcp-compose down -v

# View config (debug)
mcp-compose config
```

## Minimal + Optional Extensions

### Add Hugging Face Discovery
Uncomment in `docker-compose.minimal.yml`:
```yaml
  huggingface-mcp:
    image: shreyas/huggingface-mcp-server:latest
    # ... (already in full docker-compose.yml)
```
Then: `mcp-compose up`

### Add MCP Pandas
```yaml
  mcp-pandas:
    image: alistairwalsh/mcp-pandas:latest
    # ... (in full docker-compose.yml)
```

### Add NASA CMR (Earth Science)
```yaml
  nasa-cmr:
    image: podaac/cmr-mcp:latest
    # ... (in full docker-compose.yml)
```

## Performance Tips

### Reduce Resource Usage
```yaml
# In minimal compose, reduce data-exploration memory:
resource_limits:
  cpus: '1'
  memory: 2G
```

### Faster Startup
- Skip pulling: `docker-compose up --no-pull`
- Use local images: `docker images | grep mcp`

### Monitor Memory
```bash
watch -n 1 'docker stats --no-stream'
```

## Advanced: Custom Penrose Domains

Store custom Penrose `.sty`, `.dsl`, `.domain` files in `~/.penrose/`:
```
~/.penrose/
├── custom.domain
├── custom.sty
└── diagram_output.svg
```

Penrose MCP will auto-discover and use them!

## References

- **Quick Start**: This file
- **Full Docs**: README.md
- **Deployment**: DEPLOYMENT.md
- **Technical**: SUMMARY.md
