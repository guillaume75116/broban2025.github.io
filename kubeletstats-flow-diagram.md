# k8s.pod.network.io Metric Computation Flow Diagram

## Visual Flow Representation

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Kubelet API                        │
│                    Endpoint: /stats/summary                      │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ HTTP GET Request
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    JSON Response Structure                       │
│  {                                                               │
│    "pods": [{                                                    │
│      "podRef": {...},                                            │
│      "network": {                                                │
│        "name": "eth0",                                           │
│        "rxBytes": 948305362,      ◄─── Bytes Received           │
│        "txBytes": 12542068,       ◄─── Bytes Transmitted        │
│        "interfaces": [...]                                       │
│      }                                                           │
│    }]                                                            │
│  }                                                               │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ Parse & Process
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│              metrics.go:MetricsData()                            │
│  • Entry point for metric collection                            │
│  • Loops through all pods in summary                            │
│  • Calls acc.podStats(pod) for each pod                         │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ For each pod
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│           accumulator.go:podStats()                              │
│  • Checks if PodMetricGroup is enabled                          │
│  • Calls addNetworkMetrics() with pod network data              │
│  • Parameters:                                                   │
│    - PodMetricsBuilder                                           │
│    - PodNetworkMetrics metadata                                  │
│    - s.Network (NetworkStats)                                    │
│    - currentTime                                                 │
│    - allNetworkInterfaces flag                                   │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ Extract network data
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│           network.go:addNetworkMetrics()                         │
│                                                                  │
│  ┌────────────────────────────────────────────────────┐        │
│  │ Decision: allNetworkInterfaces flag?               │        │
│  └──────────┬────────────────────────────┬────────────┘        │
│             │ false (default)            │ true                 │
│             ▼                            ▼                      │
│  ┌──────────────────────┐    ┌──────────────────────┐         │
│  │ Default Interface    │    │ All Interfaces       │         │
│  │ (e.g., eth0)         │    │ (eth0, sit0, ...)    │         │
│  │                      │    │                      │         │
│  │ recordNetworkDataPoint│    │ Loop through        │         │
│  │ - Uses s.RxBytes     │    │ s.Interfaces[]      │         │
│  │ - Uses s.TxBytes     │    │                      │         │
│  │                      │    │ recordInterfaceDataPoint│      │
│  │                      │    │ - Uses RxBytes per    │         │
│  │                      │    │   interface           │         │
│  │                      │    │ - Uses TxBytes per    │         │
│  │                      │    │   interface           │         │
│  └──────────┬───────────┘    └──────────┬───────────┘         │
│             │                            │                      │
│             └────────────┬───────────────┘                      │
└──────────────────────────┼──────────────────────────────────────┘
                           │
                           │ Create data points
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              Metric Data Points Creation                         │
│                                                                  │
│  For each interface:                                             │
│                                                                  │
│  ┌───────────────────────────────────────────────────┐          │
│  │ Data Point 1: RECEIVE                             │          │
│  │ - Metric: k8s.pod.network.io                      │          │
│  │ - Value: rxBytes (e.g., 948305362)                │          │
│  │ - Attributes:                                      │          │
│  │   * direction: "receive"                          │          │
│  │   * interface: "eth0"                             │          │
│  │ - Type: Cumulative, Monotonic                     │          │
│  └───────────────────────────────────────────────────┘          │
│                                                                  │
│  ┌───────────────────────────────────────────────────┐          │
│  │ Data Point 2: TRANSMIT                            │          │
│  │ - Metric: k8s.pod.network.io                      │          │
│  │ - Value: txBytes (e.g., 12542068)                 │          │
│  │ - Attributes:                                      │          │
│  │   * direction: "transmit"                         │          │
│  │   * interface: "eth0"                             │          │
│  │ - Type: Cumulative, Monotonic                     │          │
│  └───────────────────────────────────────────────────┘          │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ Attach resource attributes
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│              Resource Attributes Added                           │
│  • k8s.pod.uid: "abc123..."                                     │
│  • k8s.pod.name: "my-app-pod"                                   │
│  • k8s.namespace.name: "production"                             │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          │ Emit metrics
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│              OpenTelemetry Metrics Output                        │
│  Ready for export to observability backends                      │
│  (e.g., Prometheus, Jaeger, Grafana, etc.)                      │
└─────────────────────────────────────────────────────────────────┘
```

## Configuration Impact

### Default Configuration
```yaml
# collect_all_network_interfaces not set or pod: false
```
**Result:**
- 2 data points per pod (receive + transmit for eth0)
- Lower cardinality

### All Interfaces Configuration
```yaml
collect_all_network_interfaces:
  pod: true
```
**Result:**
- 2 data points per interface per pod
- If pod has 3 interfaces → 6 data points
- Higher cardinality

## Key Source Files

| File | Purpose | Key Function |
|------|---------|-------------|
| `metrics.go` | Entry point | `MetricsData()` - Main loop |
| `accumulator.go` | Pod processing | `podStats()` - Per-pod handler |
| `network.go` | Network extraction | `addNetworkMetrics()` - Metric creation |
| `network.go` | Data extraction | `getNetworkIO()` - Returns (rx, tx) |

## Data Flow Summary

1. **Query** → Kubelet API provides pod network stats
2. **Extract** → Parse JSON to get rxBytes and txBytes
3. **Process** → Loop through pods and interfaces
4. **Create** → Generate metric data points with attributes
5. **Export** → Send to OpenTelemetry pipeline

## Metric Characteristics

- **Monotonic**: Values only increase (never decrease)
- **Cumulative**: Total bytes since pod start
- **Labeled**: Interface and direction attributes for filtering
- **Int64**: Sufficient for bytes counting (max ~9 exabytes)

---

*This diagram represents the complete flow of how `k8s.pod.network.io` metrics are computed and emitted by the kubeletstats receiver.*
