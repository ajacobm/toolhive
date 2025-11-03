# Phase 4: Integration Testing Implementation Plan

## Overview
Verify all MCP servers are accessible via ingress, test health check endpoints, validate network policy isolation, confirm monitoring metrics collection, and test tracing integration with sample requests.

## 1. MCP Server Accessibility Testing

### Prerequisites
- All MCP servers deployed and running
- Ingress controller configured and accessible
- TLS certificates provisioned and valid

### Commands

```bash
# Test connectivity to each MCP server via ingress
curl -k https://finetunings.ai/huggingface/health
curl -k https://finetunings.ai/data-explorer/health
curl -k https://finetunings.ai/pandas/health
curl -k https://finetunings.ai/datahub/health
curl -k https://finetunings.ai/penrose/health
curl -k https://finetunings.ai/gov-data/health
curl -k https://finetunings.ai/dataset-viewer/health

# Test wildcard domain access
curl -k https://test.finetunings.ai/huggingface/health
```

### Expected Results
- All health endpoints should return HTTP 200 OK
- Response should contain valid JSON with status information
- No connection timeouts or errors

## 2. Health Check Endpoint Testing

### Commands

```bash
# Detailed health check for each service
kubectl exec -n data-science -l app=huggingface-mcp -- wget -qO- http://localhost:8001/health
kubectl exec -n data-science -l app=data-exploration -- wget -qO- http://localhost:8003/health
kubectl exec -n data-science -l app=mcp-pandas -- wget -qO- http://localhost:8004/health
kubectl exec -n data-science -l app=datahub-mcp -- wget -qO- http://localhost:8005/health
kubectl exec -n data-science -l app=penrose-mcp -- wget -qO- http://localhost:8006/health
kubectl exec -n data-science -l app=gov-data-mcp -- wget -qO- http://localhost:8007/health
kubectl exec -n data-science -l app=dataset-viewer-mcp -- wget -qO- http://localhost:8008/health
```

### Expected Results
- All health checks should return valid JSON with status information
- No errors or timeouts
- Consistent response format across all services

## 3. Network Policy Isolation Testing

### Commands

```bash
# Test that MCP servers are isolated from unauthorized access
# This should fail (no access from default namespace)
kubectl run test-pod --rm -it --image=curlimages/curl -- curl -v http://huggingface-mcp.data-science:8001/health

# This should succeed (access from toolhive-system namespace)
kubectl run test-pod -n toolhive-system --rm -it --image=curlimages/curl -- curl -v http://huggingface-mcp.data-science:8001/health
```

### Expected Results
- Unauthorized access should be blocked (connection timeout or refused)
- Authorized access should succeed (HTTP 200 OK)

## 4. Monitoring Metrics Collection Testing

### Prerequisites
- Prometheus and Grafana deployed and configured
- OpenTelemetry collector deployed and configured

### Commands

```bash
# Check that metrics are being collected
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# In another terminal, check metrics
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.labels.job | contains("data-science"))'

# Check specific metrics for each MCP server
curl http://localhost:9090/api/v1/query?query=up{namespace="data-science"}
```

### Expected Results
- All MCP server targets should be up in Prometheus
- Metrics should be collected regularly
- No scrape errors

## 5. Tracing Integration Testing

### Prerequisites
- Jaeger deployed and accessible
- OpenTelemetry collector configured for tracing

### Commands

```bash
# Port forward to Jaeger UI
kubectl port-forward -n monitoring svc/jaeger-all-in-one-query 16686:16686

# Generate sample requests to create traces
for i in {1..10}; do
  curl -k https://finetunings.ai/huggingface/health
  curl -k https://finetunings.ai/data-explorer/health
  sleep 1
done

# Check traces in Jaeger UI at http://localhost:16686
```

### Expected Results
- Traces should appear in Jaeger UI
- Each request should generate a trace with proper spans
- Trace data should include service names and operation details

## 6. Load Testing

### Commands

```bash
# Install hey (load testing tool)
go install github.com/rakyll/hey@latest

# Run load test against each service
hey -z 30s -c 10 -H "Authorization: Bearer $(thv config get token)" https://finetunings.ai/huggingface/health
hey -z 30s -c 5 -H "Authorization: Bearer $(thv config get token)" https://finetunings.ai/data-explorer/health
hey -z 30s -c 5 -H "Authorization: Bearer $(thv config get token)" https://finetunings.ai/pandas/health
```

### Expected Results
- Services should handle concurrent requests without errors
- Response times should be within acceptable limits
- No service crashes or restarts during testing

## 7. Auto Scaling Testing

### Commands

```bash
# Monitor HPA status during load test
watch kubectl get hpa -n data-science

# Run intensive load test to trigger scaling
hey -z 60s -c 50 -H "Authorization: Bearer $(thv config get token)" https://finetunings.ai/huggingface/health
```

### Expected Results
- HPA should scale up pods when CPU/memory utilization exceeds thresholds
- Services should continue to respond during scaling events
- HPA should scale down pods after load decreases

## 8. Comprehensive Verification Script

```bash
#!/bin/bash

# integration-test.sh
echo "Starting integration tests..."

# Test 1: Service availability
echo "Testing service availability..."
services=("huggingface" "data-explorer" "pandas" "datahub" "penrose" "gov-data" "dataset-viewer")
for service in "${services[@]}"; do
  echo "Testing $service..."
  response=$(curl -k -s -o /dev/null -w "%{http_code}" https://finetunings.ai/$service/health)
  if [ "$response" == "200" ]; then
    echo "✓ $service is accessible"
  else
    echo "✗ $service is not accessible (HTTP $response)"
  fi
done

# Test 2: Health checks
echo "Testing health endpoints..."
for service in "${services[@]}"; do
  echo "Checking $service health..."
  health=$(kubectl exec -n data-science -l app=$service -- wget -qO- http://localhost:8001/health 2>/dev/null || echo "error")
  if [ "$health" != "error" ]; then
    echo "✓ $service health check passed"
  else
    echo "✗ $service health check failed"
  fi
done

# Test 3: Metrics collection
echo "Checking metrics collection..."
metrics_up=$(curl -s http://localhost:9090/api/v1/query?query=up{namespace="data-science"} | jq '.status' 2>/dev/null || echo "error")
if [ "$metrics_up" == "\"success\"" ]; then
  echo "✓ Metrics are being collected"
else
  echo "✗ Metrics collection issue"
fi

echo "Integration tests completed."
```

## 9. Test Results Documentation

Create a test results document to track:
- Test date and time
- ToolHive operator version
- Kubernetes version
- Test environment details
- Results for each test case
- Any issues discovered and their resolutions

## 10. Cleanup After Testing

```bash
# Remove test pods
kubectl delete pod test-pod
kubectl delete pod -n toolhive-system test-pod

# Reset any test data if needed
```

## Next Steps
After completing Phase 4, proceed to Phase 5: Client Integration by configuring Claude Desktop with MCP server connections and testing data discovery workflows.