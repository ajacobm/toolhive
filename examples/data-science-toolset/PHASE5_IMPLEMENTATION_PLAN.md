# Phase 5: Client Integration Implementation Plan

## Overview
Configure Claude Desktop with MCP server connections, test data discovery workflows, validate analysis pipelines, verify governance integration, and test visualization outputs.

## 1. Claude Desktop Configuration

### Prerequisites
- Claude Desktop installed and running
- ToolHive CLI (thv) installed and configured
- MCP servers accessible via ingress

### Configuration Steps

1. **Configure ToolHive Client**
```bash
# Initialize ToolHive client
thv client setup

# Register all MCP servers
thv client register --name huggingface --endpoint https://finetunings.ai/huggingface
thv client register --name data-explorer --endpoint https://finetunings.ai/data-explorer
thv client register --name pandas --endpoint https://finetunings.ai/pandas
thv client register --name datahub --endpoint https://finetunings.ai/datahub
thv client register --name penrose --endpoint https://finetunings.ai/penrose
thv client register --name gov-data --endpoint https://finetunings.ai/gov-data
thv client register --name dataset-viewer --endpoint https://finetunings.ai/dataset-viewer

# List registered clients
thv client list
```

2. **Configure Claude Desktop MCP Connections**
   
   In Claude Desktop settings, add MCP connections for each service:
   - Hugging Face MCP: `https://finetunings.ai/huggingface`
   - Data Exploration: `https://finetunings.ai/data-explorer`
   - MCP Pandas: `https://finetunings.ai/pandas`
   - DataHub MCP: `https://finetunings.ai/datahub`
   - Penrose MCP: `https://finetunings.ai/penrose`
   - Government Data: `https://finetunings.ai/gov-data`
   - Dataset Viewer: `https://finetunings.ai/dataset-viewer`

## 2. Data Discovery Workflows Testing

### Test Workflow 1: Hugging Face Model Discovery
```bash
# List available tools in Hugging Face MCP
thv mcp list tools --server huggingface

# Search for models
thv mcp list resources --server huggingface --type model --query "bert"

# Get model information
thv mcp list resources --server huggingface --type model --id "bert-base-uncased"
```

### Test Workflow 2: Dataset Discovery
```bash
# Browse datasets with Dataset Viewer
thv mcp list resources --server dataset-viewer --type dataset

# Search government datasets
thv mcp list resources --server gov-data --type dataset --query "climate"
```

### Expected Results
- All discovery operations should return relevant results
- No errors or timeouts
- Consistent response format across services

## 3. Analysis Pipelines Validation

### Test Pipeline 1: Data Exploration to Pandas
```bash
# Start data exploration session
thv run data-exploration --tool explore_dataset --params '{"dataset_name": "sample_data.csv"}'

# Process data with pandas
thv run pandas --tool analyze_dataframe --params '{"operation": "summary_statistics"}'

# Chain operations
thv run data-exploration --tool filter_data --params '{"column": "value", "threshold": 100}' | thv run pandas --tool visualize_data --params '{"chart_type": "bar"}'
```

### Test Pipeline 2: Hugging Face to Data Exploration
```bash
# Load model from Hugging Face
thv run huggingface --tool load_model --params '{"model_name": "bert-base-uncased"}'

# Process data with loaded model
thv run data-exploration --tool process_with_model --params '{"model_ref": "bert-ref-123", "data": "sample_text.txt"}'
```

### Expected Results
- Pipeline operations should complete successfully
- Data should flow correctly between services
- No compatibility issues between services

## 4. Governance Integration Testing

### Test DataHub Integration
```bash
# Register data asset in DataHub
thv run datahub --tool register_dataset --params '{"dataset_name": "experiment_results", "owner": "data-science-team"}'

# Add lineage information
thv run datahub --tool add_lineage --params '{"input_dataset": "raw_data", "output_dataset": "processed_data", "process": "data-cleaning"}'

# Get governance information
thv run datahub --tool get_dataset_info --params '{"dataset_name": "experiment_results"}'
```

### Expected Results
- Data assets should be registered in DataHub
- Lineage information should be captured
- Governance metadata should be accessible

## 5. Visualization Outputs Testing

### Test Penrose Diagram Generation
```bash
# Generate mathematical diagram
thv run penrose --tool generate_diagram --params '{"specification": "set-theory", "content": "A ⊆ B, B ⊆ C"}'

# Retrieve generated SVG
thv run penrose --tool get_diagram --params '{"diagram_id": "diagram-123"}' --output-format svg > diagram.svg

# Verify SVG content
file diagram.svg
```

### Test Pandas Visualization (if available)
```bash
# Generate plot with pandas
thv run pandas --tool plot_data --params '{"data": "[1,2,3,4,5]", "type": "line", "title": "Sample Plot"}' --output-format png > plot.png

# Verify PNG content
file plot.png
```

### Expected Results
- Visualization services should generate valid output files
- Output formats should match requested types
- Files should be usable (valid SVG/PNG)

## 6. End-to-End Workflow Testing

### Complete Data Science Workflow
```bash
# 1. Discover and select a dataset
dataset_id=$(thv mcp list resources --server gov-data --type dataset --query "climate" | jq -r '.[0].id')

# 2. View dataset sample
thv run dataset-viewer --tool view_sample --params "{\"dataset_id\": \"$dataset_id\"}"

# 3. Explore dataset
thv run data-exploration --tool explore_dataset --params "{\"source\": \"dataset-viewer\", \"dataset_id\": \"$dataset_id\"}"

# 4. Process data with pandas
thv run pandas --tool clean_data --params "{\"input\": \"exploration-results-123\", \"operations\": [\"remove_nulls\", \"normalize\"]}"

# 5. Register in DataHub
thv run datahub --tool register_dataset --params "{\"dataset_name\": \"cleaned_climate_data\", \"source\": \"pandas-processing-456\"}"

# 6. Generate visualization
thv run penrose --tool generate_diagram --params "{\"specification\": \"bar-chart\", \"data_ref\": \"cleaned_climate_data\"}"
```

## 7. Performance and Reliability Testing

### Concurrent Operations
```bash
# Run multiple operations concurrently
thv run huggingface --tool search_models --params '{"query": "nlp"}' &
thv run data-exploration --tool explore_dataset --params '{"dataset": "sample.csv"}' &
thv run pandas --tool summarize_data --params '{"data_ref": "exp-789"}' &
wait
```

### Error Handling
```bash
# Test error handling with invalid parameters
thv run huggingface --tool load_model --params '{"model_name": "non-existent-model"}' || echo "Error handled correctly"
```

## 8. Security and Access Testing

### Test Authentication
```bash
# Test with valid credentials
thv run huggingface --tool search_models --params '{"query": "bert"}' --token "$HF_TOKEN"

# Test without credentials (should have limited access)
thv run huggingface --tool search_models --params '{"query": "bert"}'
```

### Test Authorization
```bash
# Test access to restricted operations
thv run datahub --tool admin_operation --params '{"action": "delete_dataset"}' || echo "Access correctly restricted"
```

## 9. Client Integration Verification Script

```bash
#!/bin/bash

# client-integration-test.sh
echo "Starting client integration tests..."

# Test 1: Client registration
echo "Testing client registration..."
thv client list | grep -q "huggingface" && echo "✓ Hugging Face client registered"
thv client list | grep -q "pandas" && echo "✓ Pandas client registered"

# Test 2: Basic operations
echo "Testing basic operations..."
thv mcp list tools --server huggingface >/dev/null && echo "✓ Hugging Face tools accessible"
thv mcp list tools --server pandas >/dev/null && echo "✓ Pandas tools accessible"

# Test 3: End-to-end workflow
echo "Testing end-to-end workflow..."
# Simple discovery workflow
result=$(thv mcp list resources --server gov-data --type dataset --query "test" 2>&1)
if [ $? -eq 0 ]; then
  echo "✓ End-to-end workflow successful"
else
  echo "✗ End-to-end workflow failed: $result"
fi

echo "Client integration tests completed."
```

## 10. Troubleshooting and Debugging

### Common Issues

1. **Connection Refused**
   ```bash
   # Check ingress status
   kubectl get ingress -n data-science
   
   # Check service endpoints
   kubectl get endpoints -n data-science
   ```

2. **Authentication Errors**
   ```bash
   # Check secret configuration
   kubectl get secrets -n data-science
   
   # Verify secret content
   kubectl get secret hf-credentials -n data-science -o yaml
   ```

3. **Performance Issues**
   ```bash
   # Check resource usage
   kubectl top pods -n data-science
   
   # Check HPA status
   kubectl get hpa -n data-science
   ```

## 11. Documentation and Handoff

Create documentation for end users including:
- Connection details for each MCP service
- Example workflows and use cases
- Troubleshooting guide
- Best practices for data science workflows

## Next Steps
After completing all five phases, the data science MCP toolset deployment is ready for production use. Consider setting up monitoring alerts, backup procedures, and regular maintenance schedules.