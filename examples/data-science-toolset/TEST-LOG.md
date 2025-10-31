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
