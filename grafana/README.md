# Grafana observability dashboard

This directory contains the exported Grafana dashboard for the RAG App.

Dashboard Definition: [`rag-dashboard.json`](./rag-dashboard.json)

![`dashboard.png`](./dashboard.png)

The dashboard uses Azure Managed Prometheus through an Azure Managed Grafana
workspace. It provides an operational view of the RAG App and Pods.

## Dashboard panels
The dashboard currently includes:
- HTTP request counts by route
- `/healthz` Avg response time
- `/query` P50, P95, and P99 latency
- Pod CPU utilization
- Pod memory utilization as a percentage of configured memory limits
- HTTP status counts by route and status code
- Pod restart counts
- A 5xx error-rate threshold panel

The request and latency panels use application metrics exposed by FastAPI
instrumentation. The CPU, memory, and restart panels use Kubernetes metrics
collected by Azure Managed Prometheus.

## Import into Grafana
1. Open the target Azure Managed Grafana workspace.
2. Select **Dashboards** and choose **Import**.
3. Upload `ras-backend-dashboard.json`.
4. Select the Azure Managed Prometheus data source when prompted.
5. Save the imported dashboard.

The exported JSON contains the datasource name from the development workspace.
When importing into another workspace, select the matching Prometheus data
source rather than relying on the exported datasource reference.

## PromQL notes

Counters are converted into rates or interval counts before they are displayed.
```promql
sum by (handler) (increase(http_requests_total[5m]))
```

Latency percentiles are calculated from Prometheus histogram buckets. 
```promql
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket{handler="/query"}[5m])))
```

CPU usage is calculated from the cumulative CPU-seconds counter:
```promql
100 * sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="raas" container!="",container!="POD"}[5m]))
```

Memory percentage compares the current working set with the configured pod
memory limit:

```promql
100 * sum by (namespace, pod) (container_memory_working_set_bytes{namespace="raas", container!="",container!="POD"})
/
sum by (namespace, pod) (kube_pod_container_resource_limits{namespace="raas", resource="memory",unit="byte"})
```

## Production considerations
The exported error-rate panel is set to >5% 5XX error. Before creating an alert, account for
- The endpoint's traffic volume
- The service availability SLO and error budget
- A pending duration, such as five minutes
- A minimum number of failed requests to avoid low-traffic noise
- Different criticality for `/query`, `/ingest`, and health endpoints

For example, a low-traffic service can exceed a percentage threshold after a
single failed request. Pairing an error-rate condition with a minimum request
or error count makes the alert more useful.