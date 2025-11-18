# How k8s.pod.network.io is Computed in Kubeletstats Receiver

## Overview

The `k8s.pod.network.io` metric is computed by the **kubeletstats receiver** in the OpenTelemetry Collector Contrib repository. This metric tracks network I/O (input/output) at the pod level in Kubernetes clusters.

**Repository**: [opentelemetry-collector-contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib)  
**Component**: `receiver/kubeletstatsreceiver`

## Metric Definition

- **Name**: `k8s.pod.network.io`
- **Description**: Pod network IO (bytes transferred)
- **Unit**: Bytes (By)
- **Type**: Sum (Cumulative, Monotonic)
- **Value Type**: Int64
- **Stability**: Development
- **Attributes**:
  - `interface`: Name of the network interface (e.g., "eth0", "sit0")
  - `direction`: Direction of data flow - either "receive" or "transmit"

## Data Source

The metric is derived from the Kubernetes kubelet `/stats/summary` API endpoint, which provides statistics about pods and nodes. The relevant data structure is:

```json
{
  "pods": [
    {
      "podRef": {...},
      "network": {
        "time": "2020-04-20T22:52:20Z",
        "name": "eth0",
        "rxBytes": 948305362,
        "rxErrors": 0,
        "txBytes": 12542068,
        "txErrors": 0,
        "interfaces": [
          {
            "name": "eth0",
            "rxBytes": 948305362,
            "rxErrors": 0,
            "txBytes": 12542068,
            "txErrors": 0
          },
          {
            "name": "sit0",
            "rxBytes": 0,
            "rxErrors": 0,
            "txBytes": 0,
            "txErrors": 0
          }
        ]
      }
    }
  ]
}
```

## Computation Flow

### 1. Data Collection Entry Point
**File**: `receiver/kubeletstatsreceiver/internal/kubelet/metrics.go`

```go
func MetricsData(...) []pmetric.Metrics {
    acc := &metricDataAccumulator{...}
    
    // Process each pod
    for i := range summary.Pods {
        pod := &summary.Pods[i]
        acc.podStats(pod)  // Line 34
        // ... process containers and volumes
    }
    
    return acc.m
}
```

### 2. Pod Statistics Processing
**File**: `receiver/kubeletstatsreceiver/internal/kubelet/accumulator.go`

```go
func (a *metricDataAccumulator) podStats(s *stats.PodStats) {
    if !a.metricGroupsToCollect[PodMetricGroup] {
        return
    }
    
    currentTime := pcommon.NewTimestampFromTime(a.time)
    
    // Add network metrics - Line 83
    addNetworkMetrics(
        a.mbs.PodMetricsBuilder, 
        metadata.PodNetworkMetrics, 
        s.Network,  // NetworkStats from Kubernetes API
        currentTime, 
        a.allNetworkInterfaces[PodMetricGroup]  // Boolean flag
    )
    
    // ... other metrics and resource building
}
```

### 3. Network Metrics Computation
**File**: `receiver/kubeletstatsreceiver/internal/kubelet/network.go`

The actual computation happens in the `addNetworkMetrics` function:

```go
func addNetworkMetrics(
    mb *metadata.MetricsBuilder, 
    networkMetrics metadata.NetworkMetrics, 
    s *stats.NetworkStats, 
    currentTime pcommon.Timestamp, 
    allInterfaces bool
) {
    if s == nil {
        return
    }

    if allInterfaces {
        // Mode 1: Collect metrics from ALL network interfaces
        for i := range s.Interfaces {
            recordInterfaceDataPoint(mb, networkMetrics.IO, &s.Interfaces[i], getInterfaceIO, currentTime)
            recordInterfaceDataPoint(mb, networkMetrics.Errors, &s.Interfaces[i], getInterfaceErrors, currentTime)
        }
        return
    }

    // Mode 2: Collect metrics from DEFAULT interface only (e.g., eth0)
    recordNetworkDataPoint(mb, networkMetrics.IO, s, getNetworkIO, currentTime)
    recordNetworkDataPoint(mb, networkMetrics.Errors, s, getNetworkErrors, currentTime)
}
```

### 4. Data Point Recording

**For Default Interface** (when `allInterfaces` is false):

```go
func recordNetworkDataPoint(
    mb *metadata.MetricsBuilder, 
    recordDataPoint metadata.RecordIntDataPointWithDirectionFunc, 
    s *stats.NetworkStats, 
    getData getNetworkDataFunc, 
    currentTime pcommon.Timestamp
) {
    rx, tx := getData(s)  // Extract rxBytes and txBytes

    if rx != nil {
        recordDataPoint(mb, currentTime, int64(*rx), s.Name, metadata.AttributeDirectionReceive)
    }

    if tx != nil {
        recordDataPoint(mb, currentTime, int64(*tx), s.Name, metadata.AttributeDirectionTransmit)
    }
}

func getNetworkIO(s *stats.NetworkStats) (*uint64, *uint64) {
    return s.RxBytes, s.TxBytes  // Returns (received bytes, transmitted bytes)
}
```

**For All Interfaces** (when `allInterfaces` is true):

```go
func recordInterfaceDataPoint(
    mb *metadata.MetricsBuilder, 
    recordDataPoint metadata.RecordIntDataPointWithDirectionFunc, 
    s *stats.InterfaceStats, 
    getData getInterfaceDataFunc, 
    currentTime pcommon.Timestamp
) {
    rx, tx := getData(s)  // Extract rxBytes and txBytes for specific interface

    if rx != nil {
        recordDataPoint(mb, currentTime, int64(*rx), s.Name, metadata.AttributeDirectionReceive)
    }

    if tx != nil {
        recordDataPoint(mb, currentTime, int64(*tx), s.Name, metadata.AttributeDirectionTransmit)
    }
}

func getInterfaceIO(s *stats.InterfaceStats) (*uint64, *uint64) {
    return s.RxBytes, s.TxBytes  // Returns (received bytes, transmitted bytes)
}
```

## Configuration

### Default Behavior
By default, the receiver collects network metrics only from the **default network interface** (typically `eth0`).

### Collecting from All Interfaces
To enable collection from all network interfaces:

```yaml
receivers:
  kubeletstats:
    collection_interval: 10s
    auth_type: "serviceAccount"
    endpoint: "${env:K8S_NODE_NAME}:10250"
    insecure_skip_verify: true
    collect_all_network_interfaces:
      pod: true   # Enable for pods
      node: true  # Enable for nodes
```

**Note**: Enabling `collect_all_network_interfaces` increases metric cardinality due to the additional `interface` attribute dimension.

## Resulting Metrics

For each pod, the receiver generates **two data points per interface**:

1. **Receive direction**: `k8s.pod.network.io{direction="receive", interface="eth0"}` = rxBytes value
2. **Transmit direction**: `k8s.pod.network.io{direction="transmit", interface="eth0"}` = txBytes value

### Example Output

```yaml
- description: Pod network IO
  name: k8s.pod.network.io
  sum:
    aggregationTemporality: 2  # Cumulative
    dataPoints:
      - asInt: "948305362"
        attributes:
          - key: direction
            value:
              stringValue: receive
          - key: interface
            value:
              stringValue: eth0
        startTimeUnixNano: "1000000"
        timeUnixNano: "2000000"
      - asInt: "12542068"
        attributes:
          - key: direction
            value:
              stringValue: transmit
          - key: interface
            value:
              stringValue: eth0
        startTimeUnixNano: "1000000"
        timeUnixNano: "2000000"
    isMonotonic: true
  unit: By
```

## Key Implementation Details

### 1. Data Extraction
- The kubeletstats receiver queries the kubelet API at `/stats/summary`
- For each pod, it extracts `PodStats.Network` which contains:
  - `RxBytes`: Total bytes received
  - `TxBytes`: Total bytes transmitted
  - `Name`: Default interface name
  - `Interfaces[]`: Array of per-interface statistics

### 2. Metric Type
- **Cumulative**: The values represent total bytes since pod start
- **Monotonic**: Values only increase (or reset on pod restart)
- **Int64**: Values are converted from `uint64` to `int64`

### 3. Resource Attributes
Each metric is associated with pod resource attributes:
- `k8s.pod.uid`: Unique identifier
- `k8s.pod.name`: Pod name
- `k8s.namespace.name`: Namespace

### 4. Two Modes of Operation

| Mode | Configuration | Behavior | Cardinality |
|------|--------------|----------|-------------|
| Default | `collect_all_network_interfaces.pod: false` | Only default interface (eth0) | Lower |
| All Interfaces | `collect_all_network_interfaces.pod: true` | All interfaces (eth0, sit0, etc.) | Higher |

## Source Code References

- **Main entry point**: `receiver/kubeletstatsreceiver/internal/kubelet/metrics.go:16-48`
- **Pod stats processing**: `receiver/kubeletstatsreceiver/internal/kubelet/accumulator.go:73-93`
- **Network metrics logic**: `receiver/kubeletstatsreceiver/internal/kubelet/network.go:17-74`
- **Metric definition**: `receiver/kubeletstatsreceiver/metadata.yaml` (lines for k8s.pod.network.io)
- **Test expectations**: `receiver/kubeletstatsreceiver/testdata/scraper/test_scraper_expected.yaml`

## Related Metrics

The same logic applies to:
- `k8s.pod.network.errors` - Network errors (RxErrors, TxErrors)
- `k8s.node.network.io` - Node-level network I/O
- `k8s.node.network.errors` - Node-level network errors

## Summary

The `k8s.pod.network.io` metric computation is straightforward:

1. **Source**: Kubelet `/stats/summary` API provides `rxBytes` and `txBytes` from the pod's network namespace
2. **Processing**: The receiver extracts these values from `PodStats.Network`
3. **Output**: Two separate metric data points are created:
   - One for received bytes (direction="receive")
   - One for transmitted bytes (direction="transmit")
4. **Labeling**: Each data point includes the interface name and direction as attributes
5. **Aggregation**: Values are cumulative counters representing total bytes since pod start

The metric directly reflects the underlying Linux network statistics from the pod's network namespace as reported by the Kubernetes kubelet.
