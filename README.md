# Log Analysis Agent

AI-powered root cause analysis (RCA) agent for Kubernetes microservice logs using LangChain, LangGraph, and Elasticsearch.

## Overview

This project implements an intelligent agent that performs automated root cause analysis on K3s/Kubernetes microservice logs stored in Elasticsearch. It uses LangGraph for agentic workflows and Ollama for local LLM inference.

### Key Features

- **Agentic RCA Workflow**: Multi-step reasoning with tool-equipped LLM
- **Elasticsearch Integration**: Native hybrid (BM25 + vector) search on K8s logs
- **Local LLM**: Ollama integration for on-premise deployment
- **K8s-Aware**: Understands namespaces, pods, services, and K8s log schema
- **Iterative Analysis**: Reflection loop for evidence gathering
- **FastAPI REST API**: Simple HTTP interface for queries

## Architecture

```
User Query → Router → Tool Node → LLM Reasoning → Reflection → RCA Report
                ↓                         ↓
          Elasticsearch            Hypothesis Loop
          (search_logs)         (needs_more_evidence?)
```

## Prerequisites

- Python 3.11+
- Elasticsearch 8.x+ with K8s logs indexed
- Ollama running with llama3 or mistral model
- K3s/K8s cluster (optional, for deployment)

## Installation

### 1. Clone Repository

```bash
git clone https://github.com/kapilgupta86/log_analysis_agent.git
cd log_analysis_agent
```

### 2. Install Dependencies

```bash
pip install -e .
# or with uv:
uv pip install -e .
```

### 3. Configure Environment

```bash
cp .env.example .env
# Edit .env with your credentials:
# ES_URL=https://172.27.1.10:300060
# ES_USERNAME=elastic
# ES_PASSWORD=XYX
# OLLAMA_URL=http://172.27.10.10:300123
```

### 4. Verify Connections

```python
python -c "from src.tools.elastic_logs import get_es_client; get_es_client()"
# Should print: ✓ Connected to Elasticsearch successfully
```

## Usage

### Start FastAPI Server

```bash
uvicorn src.main:app --host 0.0.0.0 --port 8000
```

### Query via HTTP

```bash
curl -X POST http://localhost:8000/rca \
  -H "Content-Type: application/json" \
  -d '{"query": "Why did payment-service crash in prod?"}'
```

### Example Response

```json
{
  "report": "Root cause hypothesis: OOMKilled due to memory leak in payment-service...",
  "supporting_evidence": [...],
  "remediation": "Increase memory limits to 2Gi and add heap dump on OOM"
}
```

## Configuration

### Elasticsearch Index Pattern

Logs should follow standard K8s/Fluentd ECS schema:

```json
{
  "@timestamp": "2026-01-23T10:30:00Z",
  "kubernetes": {
    "namespace_name": "prod",
    "pod_name": "payment-abc-123",
    "container_name": "app",
    "node_name": "node-1"
  },
  "service": {"name": "payment-service"},
  "log": {"level": "ERROR"},
  "message": "OutOfMemoryError: Java heap space"
}
```

### Ollama Models

Tested models:
- `llama3` (recommended, 8B)
- `mistral` (7B)
- `neural-chat` (7B)

Pull model:
```bash
ollama pull llama3
```

## Deployment to K3s

### 1. Build Docker Image

```bash
docker build -t your-registry/log-analysis-agent:0.1.0 .
docker push your-registry/log-analysis-agent:0.1.0
```

### 2. Create Secret

```bash
kubectl create secret generic logs-analysis-agent-secrets \
  --from-literal=ES_URL=https://172.27.1.10:300060 \
  --from-literal=ES_USERNAME=elastic \
  --from-literal=ES_PASSWORD=XYX \
  --from-literal=OLLAMA_URL=http://172.27.10.10:300123
```

### 3. Deploy

```bash
kubectl apply -f infra/k8s-deployment.yaml
kubectl apply -f infra/k8s-service.yaml
```

## Project Structure

```
logs_analysis_agent/
├── src/
│   ├── config.py          # Environment configuration
│   ├── schema.py          # Data models (LogEntry, RCAState)
│   ├── tools/
│   │   ├── elastic_logs.py   # ES query tools
│   │   └── metrics.py        # Prometheus integration
│   ├── graph/
│   │   ├── prompts.py        # LLM prompts
│   │   ├── nodes.py          # Graph nodes (tool, reason, reflect)
│   │   └── graph_builder.py  # LangGraph workflow
│   └── main.py            # FastAPI entrypoint
├── infra/
│   ├── k8s-deployment.yaml
│   └── k8s-service.yaml
├── pyproject.toml
├── .env.example
└── README.md
```

## Troubleshooting

### Elasticsearch Connection Fails

```bash
# Test connectivity:
curl -k -u elastic:XYX https://172.27.1.10:300060/_cluster/health

# Check ES_VERIFY_CERTS=false in .env if using self-signed certs
```

### Ollama Not Responding

```bash
# Test Ollama:
curl http://172.27.10.10:300123/api/tags

# Ensure model is pulled:
ollama list
```

### No Logs Found

```bash
# Verify index pattern:
curl -k -u elastic:XYX https://172.27.1.10:300060/_cat/indices/logs-*

# Check time range (default: last 60 minutes)
```

## Development

### Run Tests

```bash
pytest tests/
```

### Code Formatting

```bash
black src/
flake8 src/
```

## Roadmap

- [ ] Add Prometheus metrics correlation
- [ ] Multi-agent collaboration (retriever + analyzer + summarizer)
- [ ] Trace ID correlation with Jaeger/Tempo
- [ ] Slack/MS Teams notifications
- [ ] Web UI dashboard

## License

MIT

## Contributing

Contributions welcome! Please open an issue or PR.

---

**Author**: Kapil Gupta  
**Repository**: https://github.com/kapilgupta86/log_analysis_agent
