# Broban 2025 GitHub Pages

This repository hosts documentation and analysis related to various technical topics.

## Contents

### Kubeletstats Receiver Analysis

Documentation on how the `k8s.pod.network.io` metric is computed in the OpenTelemetry Collector's kubeletstats receiver:

- **[kubeletstats-network-metric-analysis.md](kubeletstats-network-metric-analysis.md)** - Comprehensive technical documentation in Markdown format
- **[kubeletstats-analysis.html](kubeletstats-analysis.html)** - Formatted HTML page for web viewing
- **[kubeletstats-flow-diagram.md](kubeletstats-flow-diagram.md)** - Visual ASCII flow diagram showing the complete computation process

#### Key Topics Covered:
- Metric definition and data source
- Complete computation flow with code references
- Configuration options (default vs. all interfaces)
- Data extraction from Kubernetes kubelet API
- Example outputs and related metrics

## GitHub Pages Site

This site is hosted at: https://guillaume75116.github.io/broban2025.github.io/

## About the Kubeletstats Analysis

The analysis documents how network I/O metrics are collected at the pod level in Kubernetes clusters using OpenTelemetry's kubeletstats receiver. The metric tracks both received (rx) and transmitted (tx) bytes through the pod's network interfaces.

### Quick Summary

The `k8s.pod.network.io` metric:
- **Source**: Kubernetes kubelet `/stats/summary` API
- **Data**: `rxBytes` and `txBytes` from pod network stats
- **Output**: Two data points per interface (receive and transmit)
- **Type**: Cumulative monotonic counter
- **Attributes**: `interface` (e.g., "eth0") and `direction` ("receive" or "transmit")

## License

See individual files for licensing information.
