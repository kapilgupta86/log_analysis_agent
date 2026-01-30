# Telco Network RCA Enhancements

## Extended Infrastructure Support

This document outlines enhancements to support comprehensive Telco network root cause analysis beyond Kubernetes pods.

### 1. Extended Resource Types Supported

#### Kubernetes Components
- Pods (containers)
- Nodes (physical/virtual machines)
- Services
- StatefulSets
- DaemonSets

#### Operating System Level
- Process monitoring
- System resource utilization (CPU, Memory, Disk I/O)
- Network interface metrics
- System event logs
- Kernel logs

#### Network Elements (Telco-Specific)
- OLT (Optical Line Terminal)
- ONU (Optical Network Unit)
- Switches (Access, Aggregation, Core)
- Spine/Leaf fabric elements
- BGP/OSPF routing nodes
- Load Balancers
- MPLS PE/P routers

### 2. Enhanced Elasticsearch Index Mapping

```json
{
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date"},
      "resource_type": {
        "type": "keyword",
        "index": true
      },
      "resource_id": {"type": "keyword"},
      "kubernetes": {
        "properties": {
          "namespace_name": {"type": "keyword"},
          "pod_name": {"type": "keyword"},
          "node_name": {"type": "keyword"},
          "container_name": {"type": "keyword"}
        }
      },
      "os": {
        "properties": {
          "hostname": {"type": "keyword"},
          "process_id": {"type": "integer"},
          "process_name": {"type": "keyword"},
          "cpu_usage_percent": {"type": "float"},
          "memory_usage_bytes": {"type": "long"},
          "disk_io_read_bytes": {"type": "long"},
          "disk_io_write_bytes": {"type": "long"},
          "network_interface": {"type": "keyword"}
        }
      },
      "network_element": {
        "properties": {
          "device_type": {
            "type": "keyword",
            "index": true
          },
          "device_id": {"type": "keyword"},
          "device_name": {"type": "keyword"},
          "port_id": {"type": "keyword"},
          "uptime_seconds": {"type": "long"},
          "cpu_usage_percent": {"type": "float"},
          "memory_usage_percent": {"type": "float"},
          "optical_power_dbm": {"type": "float"},
          "ber_value": {"type": "float"}
        }
      },
      "service": {
        "properties": {
          "name": {"type": "keyword"},
          "version": {"type": "keyword"},
          "status": {"type": "keyword"}
        }
      },
      "log": {
        "properties": {
          "level": {"type": "keyword"},
          "logger_name": {"type": "keyword"}
        }
      },
      "message": {"type": "text"},
      "error": {
        "properties": {
          "type": {"type": "keyword"},
          "stack_trace": {"type": "text"}
        }
      }
    }
  }
}
```

### 3. Resource Type Classification

```python
RESOURCE_TYPES = {
    "KUBERNETES": ["pod", "node", "service", "statefulset", "daemonset"],
    "OS_LEVEL": ["process", "host", "kernel", "syslog"],
    "NETWORK_ELEMENT": {
        "OPTICAL": ["olt", "onu", "transponder"],
        "ROUTING": ["bgp_peer", "ospf_peer", "mpls_pe", "mpls_p"],
        "SWITCHING": ["access_switch", "aggregation_switch", "core_switch", "spine_leaf"],
        "LOAD_BALANCING": ["load_balancer", "ha_proxy"],
        "SECURITY": ["firewall", "ddos_scrubber"]
    }
}
```

### 4. Enhanced Query Handler

#### Query Pattern Recognition

```
"Why did OLT-01 stop processing traffic?"
→ Device Type: OLT, Time-based query

"Node-5 CPU spike at 2026-01-23 14:30"
→ OS-level, Node monitoring, Specific time

"Payment service errors in prod namespace"
→ Kubernetes pod, Namespace-specific

"Spine-01 BGP session flaps last hour"
→ Network element, Routing, Time range
```

#### Automatic Classifier

The RCA agent will:
1. Parse user query for resource mentions
2. Identify resource type (K8s, OS, Network)
3. Extract time range (with +/-15 min buffer)
4. Build Elasticsearch query filters
5. Perform targeted evidence gathering

### 5. Multi-Step Interactive Queries

If initial query is ambiguous:

**Example: "Service degradation"

Agent asks:
```
1. Which service? (e.g., payment-service, order-service)
2. Which environment? (prod/staging/dev)
3. Time range? (specific time or duration)
4. Affected components? (K8s pods, VNF nodes, network elements)
```

User provides hints, agent refines Elasticsearch queries.

### 6. Timestamp and Date Format Hints

#### Accepted Formats
```
- ISO 8601: 2026-01-23T14:30:00Z
- Unix timestamp: 1674432600
- Human readable: "2026-01-23 14:30 UTC"
- Relative: "last 15 minutes", "past 2 hours", "today"
- Date only: "2026-01-23" (assumes 00:00 UTC)
```

#### User Input Hints

When user doesn't provide timestamp:
```
Agent: "No time specified. Using current time (2026-01-23 15:00 UTC)
         with ±15 minute buffer for searching related events.
         To specify exact time, use formats:
         - '2026-01-23 14:30 UTC'
         - 'last 30 minutes'
         - '2026-01-23T10:00:00Z'"
```

### 7. Implementation Architecture

```
User Query
    ↓
Query Classifier
├── Resource Type Detector (K8s/OS/Network)
├── Time Range Extractor (with defaults)
├── Ambiguity Detector (triggers multi-step)
    ↓
ES Query Builder
├── Filter by resource type
├── Filter by time range
├── Full-text + vector search
    ↓
LLM Reasoning with Tools
├── Fetch logs
├── Correlate metrics
├── Analyze patterns
├── Multi-resource correlation
    ↓
RCA Report with:
├── Root cause hypothesis
├── Evidence from all resource types
├── Remediation steps
└── Preventive measures
```

### 8. RCA Workflow for Telco

```python
class TelcoRCAAgent:
    def analyze(self, query: str) -> RCAReport:
        # Step 1: Parse query
        parsed = self.query_parser.parse(query)
        
        # Step 2: Clarify if needed
        if parsed.ambiguity_score > 0.7:
            clarifications = self.interactive_clarifier.get_inputs(parsed)
            parsed.update(clarifications)
        
        # Step 3: Build ES query
        es_filters = self.build_es_query(
            resource_type=parsed.resource_type,
            resource_id=parsed.resource_id,
            time_range=parsed.time_range,
            namespace=parsed.namespace
        )
        
        # Step 4: Fetch evidence
        evidence = self.fetch_evidence(es_filters)
        
        # Step 5: Correlate across resources
        correlations = self.correlate_resources(evidence)
        
        # Step 6: LLM reasoning
        report = self.llm_agent.reason(
            query=query,
            evidence=evidence,
            correlations=correlations
        )
        
        return report
```

### 9. Example Telco Queries

1. **OLT Failure**
   - "Why did OLT-Mumbai-01 fail processing GPON traffic?"
   - Agent identifies: Device type=OLT, Location=Mumbai, Time=now-15min
   - Correlates: Optical power levels, BER, BGP sessions

2. **Node Resource Exhaustion**
   - "Node-7 high CPU at 2026-01-23 14:30"
   - Agent identifies: OS-level process monitoring, specific timestamp
   - Correlates: Kernel logs, process list, container metrics on that node

3. **Service Cascade**
   - "Payment service cascade failure in prod"
   - Agent asks: Time range? Affected pods? Upstream services?
   - Correlates: Pod logs, network policies, database connections

4. **Network Element Flap**
   - "Spine-01 BGP flaps last hour"
   - Agent identifies: Network element, routing, time=last 60min
   - Correlates: BGP logs, interface stats, neighbor states

### 10. Configuration for Telco Setup

```env
# Elasticsearch
ES_URL=https://172.27.1.10:300060
ES_USERNAME=elastic
ES_PASSWORD=XYX
ES_VERIFY_CERTS=false

# Index patterns for different resource types
ES_K8S_LOGS_INDEX=logs-k8s-*
ES_OS_LOGS_INDEX=logs-os-*
ES_NETWORK_LOGS_INDEX=logs-network-*
ES_SYSLOG_INDEX=logs-syslog-*

# Ollama
OLLAMA_URL=http://172.27.10.10:300123
OLLAMA_MODEL=llama3

# RCA Configuration
DEFAULT_TIME_RANGE_MINUTES=60
DEFAULT_BUFFER_MINUTES=15
AMBIGUITY_THRESHOLD=0.7
ENABLE_INTERACTIVE_MODE=true
```

### 11. Benefits for Telco Operations

1. **Unified RCA** across K8s, OS, and network elements
2. **Faster MTTR** with automatic multi-step clarification
3. **Comprehensive evidence** from all infrastructure layers
4. **Proactive insights** on network element health
5. **Familiar patterns** for Telco operations teams
6. **Extensible model** for additional device types

---

**Next Steps:**
- Implement QueryClassifier with resource type detection
- Extend Elasticsearch index templates
- Build InteractiveClarifier for multi-step queries
- Add timestamp format validator
- Test with real Telco logs
