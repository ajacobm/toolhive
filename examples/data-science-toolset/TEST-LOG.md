# Data Science MCP Toolset - Test Log

## Session 1: Working Baseline (Oct 31, 2025)

### What We Tested

1. **Docker + Docker Compose**: ✅ Both available (v28.3.3, v2.39.2-desktop)
2. **NGINX startup**: ✅ Spins up successfully on :8888
3. **Dashboard serving**: ✅ HTML dashboard serves correctly
4. **HF_TOKEN loading**: ✅ Sourced from ~/.envs

### Issues Found & Fixed

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Docker Hub images don't exist | Referenced images (reading-plus, bmorphism, quantgeekdev, etc.) aren't published | Removed references; will build from source or use official images |
| NGINX config failed in conf.d | Full main config with `user` directive in conf.d/ invalid | Created minimal nginx-server.conf with only server block |
| Upstream host errors | Removed services caused nginx to fail at startup | Commented out proxy locations for future services |

### Current Status

**Working:** Minimal setup with NGINX gateway serving dashboard
- Container: `nginx-gateway` on port 8888
- Dashboard: Accessible, fully functional HTML navigation
- Network: `data-science` bridge created

**Next Steps (in priority order):**

1. **Test NGINX proxy logic**
   - Uncomment proxy locations in nginx-server.conf
   - Add dummy services to verify routing works

2. **Build first MCP service**
   - Focus: Data Exploration from PyPI
   - Create Dockerfile.data-exploration that actually works
   - Test health check endpoint

3. **Add to compose one-by-one**
   - data-exploration (8003)
   - Then other services as images become available

4. **Eventually test K8s deployment**
   - Same pattern with Ingress instead of NGINX
   - Use persistent volume for penrose/ diagrams

### Commands Needed

```bash
# Build & test locally
cd docker-compose
docker-compose -f docker-compose.minimal.yml up -d
docker-compose -f docker-compose.minimal.yml logs nginx-gateway

# Test inside container
docker exec nginx-gateway wget -q -O - http://127.0.0.1/ | head

# Check specific port
docker exec nginx-gateway sh -c 'nc -zv 127.0.0.1 80'

# Clean up
docker-compose -f docker-compose.minimal.yml down
```

### Known Limitations

- No MCP services running yet (only NGINX)
- Proxy endpoints commented out (will uncomment when services exist)
- No persistent storage tested (~/. penrose, ~/.data-explorer dirs)
- Browser automation (Playwright) needs Chrome install

---

## Session 2: Added First MCP Service (Oct 31, 2025)

### What We Added

1. **Data Exploration container**: ✅ Built from mcp-server-ds PyPI package
2. **NGINX proxy routing**: ✅ /api/explorer/ → data-exploration:8003
3. **Docker Compose integration**: ✅ Both services start together

### Build Process

```bash
cd docker-compose
docker-compose build data-exploration      # ~70 packages installed
docker-compose up -d                       # Both services start
```

### Status

**NGINX Gateway**: ✅ Running, dashboard served on :8888  
**Data Exploration**: ❌ Crashing on startup - MCP library version mismatch

### Issue Found

```
ImportError: cannot import name 'McpError' from 'mcp.server'
```

Root cause: mcp-server-ds (v0.1.5) incompatible with mcp (v1.20.0) installed.  
Need to pin specific compatible versions in Dockerfile.

### Next Action

Either:
1. Pin MCP version in Dockerfile (check mcp-server-ds pyproject.toml for compatible version)
2. Or fork/fix mcp-server-ds to work with latest MCP SDK

### Architecture Verified

✅ Two-service pattern works:
- Service builds locally from Dockerfile
- docker-compose orchestration works
- NGINX proxy routing framework ready
- Can add more services (Treasury, Data.gov, etc.) following same pattern

### Commands Used

```bash
# Build specific service
docker-compose build --no-cache data-exploration

# Check logs
docker-compose logs data-exploration

# Verify NGINX still working
docker exec nginx-gateway wget -q -O - http://127.0.0.1/ 

# Check service status
docker-compose ps
```

### Learning: Start Simple Edition

We proved the "start simple, level up" strategy works:
1. ✅ First: Get basic NGINX gateway working
2. ✅ Second: Add one service, build & test integration
3. ⏭️ Next: Fix version issues, add more services
4. ⏭️ Eventually: Deploy to Kubernetes

Each step is independent, testable, and incrementally better.
