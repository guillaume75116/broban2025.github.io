# Cumulative Monotonic Sum Behavior for k8s.pod.network.io

## Answer to Follow-up Questions

### Question 1: Does the sum stop at a certain value before going back to 0?

**Answer: Yes, the counter can theoretically wrap around, but it's extremely unlikely in practice.**

#### Technical Details:

1. **Data Type in Kubernetes API**: The source data (`RxBytes` and `TxBytes`) is defined as `*uint64` in the Kubernetes kubelet stats API:
   ```go
   // From kubernetes/pkg/kubelet/apis/stats/v1alpha1/types.go
   type InterfaceStats struct {
       Name string `json:"name"`
       // Cumulative count of bytes received.
       RxBytes *uint64 `json:"rxBytes,omitempty"`
       // Cumulative count of bytes transmitted.
       TxBytes *uint64 `json:"txBytes,omitempty"`
   }
   ```

2. **Maximum Value**: 
   - `uint64` maximum: **18,446,744,073,709,551,615 bytes** (2^64 - 1)
   - This equals approximately **18.4 exabytes** (18.4 million terabytes)

3. **Conversion to OpenTelemetry**: The value is converted from `uint64` to `int64` when recorded:
   ```go
   // From network.go line 40
   recordDataPoint(mb, currentTime, int64(*rx), s.Name, metadata.AttributeDirectionReceive)
   ```
   
4. **Practical Implications**:
   - At 10 Gbps (gigabits per second) sustained transfer rate:
     - 1.25 GB/s = 1,250,000,000 bytes/sec
     - Time to overflow: 18,446,744,073,709,551,615 / 1,250,000,000 = **14,757,395,258 seconds**
     - This is approximately **467 years** of continuous maximum throughput

5. **What Happens at Rollover**:
   - When a `uint64` counter reaches its maximum value and increments again, it wraps around to 0 (this is standard unsigned integer behavior)
   - The Linux kernel network counters follow this same behavior
   - Monitoring systems typically detect this wrap-around by observing a decrease in a monotonic counter and handle it appropriately

6. **Pod Lifetime**:
   - In practice, pods are ephemeral and typically restarted long before reaching anywhere near the overflow limit
   - When a pod restarts, the counters reset to 0 anyway
   - The metric has a `StartTime` associated with it, so monitoring systems can detect pod restarts

### Question 2: Show me the precise part of the code where you can see that it is a cumulative sum

**Answer: The cumulative sum behavior is defined in multiple places in the code:**

#### Location 1: Metadata Definition (`metadata.yaml`)

**File**: `receiver/kubeletstatsreceiver/metadata.yaml`

```yaml
k8s.pod.network.io:
    enabled: true
    description: "Pod network IO"
    unit: By
    stability:
      level: development
    sum:
      value_type: int
      monotonic: true
      aggregation_temporality: cumulative    # ← CUMULATIVE DEFINED HERE
    attributes: ["interface", "direction"]
```

**Line reference**: The `aggregation_temporality: cumulative` explicitly defines this as a cumulative metric.

#### Location 2: Generated Metrics Code (`generated_metrics.go`)

**File**: `receiver/kubeletstatsreceiver/internal/metadata/generated_metrics.go`

The metric initialization sets up the cumulative behavior:

```go
func (m *metricK8sPodNetworkIo) init() {
	m.data.SetName("k8s.pod.network.io")
	m.data.SetDescription("Pod network IO")
	m.data.SetUnit("By")
	m.data.SetEmptySum()
	m.data.Sum().SetIsMonotonic(true)                                            // ← MONOTONIC
	m.data.Sum().SetAggregationTemporality(pmetric.AggregationTemporalityCumulative)  // ← CUMULATIVE
	m.data.Sum().DataPoints().EnsureCapacity(m.capacity)
}
```

**Key Lines**:
- `SetIsMonotonic(true)`: Marks the metric as monotonically increasing
- `SetAggregationTemporality(pmetric.AggregationTemporalityCumulative)`: Marks it as cumulative (not delta)

#### Location 3: Data Point Recording (`generated_metrics.go`)

**File**: `receiver/kubeletstatsreceiver/internal/metadata/generated_metrics.go`

The actual data point recording preserves the raw cumulative value:

```go
func (m *metricK8sPodNetworkIo) recordDataPoint(start pcommon.Timestamp, ts pcommon.Timestamp, val int64, interfaceAttributeValue string, directionAttributeValue string) {
	if !m.config.Enabled {
		return
	}
	dp := m.data.Sum().DataPoints().AppendEmpty()
	dp.SetStartTimestamp(start)      // ← Start time (pod start time)
	dp.SetTimestamp(ts)              // ← Current observation time
	dp.SetIntValue(val)              // ← Raw cumulative value from kubelet
	dp.Attributes().PutStr("interface", interfaceAttributeValue)
	dp.Attributes().PutStr("direction", directionAttributeValue)
}
```

**Key Points**:
- `SetStartTimestamp(start)`: Records when the counter started (pod start time)
- `SetTimestamp(ts)`: Records the current observation time
- `SetIntValue(val)`: Stores the **raw cumulative value** directly from the kubelet API without any calculation or differencing

#### Location 4: Value Extraction (`network.go`)

**File**: `receiver/kubeletstatsreceiver/internal/kubelet/network.go`

The code extracts the raw cumulative values directly:

```go
func getNetworkIO(s *stats.NetworkStats) (*uint64, *uint64) {
	return s.RxBytes, s.TxBytes    // ← Returns raw cumulative counters
}

func recordNetworkDataPoint(mb *metadata.MetricsBuilder, recordDataPoint metadata.RecordIntDataPointWithDirectionFunc, s *stats.NetworkStats, getData getNetworkDataFunc, currentTime pcommon.Timestamp) {
	rx, tx := getData(s)           // ← Gets raw cumulative values

	if rx != nil {
		recordDataPoint(mb, currentTime, int64(*rx), s.Name, metadata.AttributeDirectionReceive)  // ← Records raw value
	}

	if tx != nil {
		recordDataPoint(mb, currentTime, int64(*tx), s.Name, metadata.AttributeDirectionTransmit) // ← Records raw value
	}
}
```

**Key Point**: The code does **NOT**:
- Calculate deltas between observations
- Reset counters
- Perform any mathematical operations on the values

It simply passes through the raw cumulative byte counter from the Linux kernel network statistics, as reported by the kubelet.

#### Location 5: Kubernetes API Source

**File**: `kubernetes/pkg/kubelet/apis/stats/v1alpha1/types.go` (upstream Kubernetes)

The source data structure explicitly documents the cumulative nature:

```go
type InterfaceStats struct {
	Name string `json:"name"`
	// Cumulative count of bytes received.    // ← EXPLICIT COMMENT: "Cumulative"
	RxBytes *uint64 `json:"rxBytes,omitempty"`
	// Cumulative count of receive errors encountered.
	RxErrors *uint64 `json:"rxErrors,omitempty"`
	// Cumulative count of bytes transmitted.  // ← EXPLICIT COMMENT: "Cumulative"
	TxBytes *uint64 `json:"txBytes,omitempty"`
	// Cumulative count of transmit errors encountered.
	TxErrors *uint64 `json:"txErrors,omitempty"`
}
```

## Summary

### Cumulative Behavior Chain:

1. **Linux Kernel** → Maintains cumulative network byte counters per interface (uint64)
2. **Kubelet** → Reads these counters and exposes them via `/stats/summary` API
3. **Kubeletstats Receiver** → Extracts raw values without modification
4. **OpenTelemetry Metric** → Stores as cumulative monotonic sum with start timestamp

### Why Cumulative?

Cumulative metrics have several advantages:
- **Resilient to missed samples**: If you miss a collection interval, you still have the total
- **Easy rate calculation**: Rate = (value_now - value_before) / (time_now - time_before)
- **Handles restarts**: Start timestamp allows detecting when counter resets
- **Aligned with source**: Matches the semantics of underlying Linux kernel counters

### Code Flow Diagram:

```
Linux Kernel Network Stats (uint64 cumulative counter)
    ↓
Kubelet /stats/summary API
    {rxBytes: 948305362, txBytes: 12542068}
    ↓
network.go:getNetworkIO()
    Returns raw uint64* values unchanged
    ↓
network.go:recordNetworkDataPoint()
    Converts uint64 → int64, no other math
    ↓
generated_metrics.go:recordDataPoint()
    dp.SetIntValue(val) ← Stores raw cumulative value
    With aggregation_temporality: cumulative
    With monotonic: true
    ↓
OpenTelemetry Pipeline
    Exports cumulative monotonic sum metric
```

## Practical Example

If a pod has transferred:
- 1st observation at t=0: 1,000,000 bytes
- 2nd observation at t=10s: 2,500,000 bytes
- 3rd observation at t=20s: 4,200,000 bytes

The metric reports:
- t=0: value=1,000,000 (cumulative total since pod start)
- t=10s: value=2,500,000 (cumulative total since pod start)
- t=20s: value=4,200,000 (cumulative total since pod start)

To calculate the rate between t=10s and t=20s:
- Rate = (4,200,000 - 2,500,000) / (20 - 10) = 170,000 bytes/second

The consumer of the metric (like Prometheus) is responsible for calculating rates/deltas from the cumulative values.
