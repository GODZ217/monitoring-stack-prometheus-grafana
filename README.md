# Monitoring Stack — Prometheus & Grafana

Comprehensive monitoring solution with Prometheus, Grafana, Loki, and Tempo for metrics, logs, and traces with custom dashboards and alerting.

## Architecture

```
┌────────────────────────────────────────────────────────────┐
│                  Monitoring Stack                          │
├────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │                  Data Sources                       │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │   │
│  │  │Kubernetes│  │   Node   │  │  Applications    │ │   │
│  │  │ Metrics  │  │ Exporter │  │  / Services      │ │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘ │   │
│  └────────────────────────────────────────────────────┘   │
│                          │                                  │
│                          ▼                                  │
│  ┌────────────────────────────────────────────────────┐   │
│  │                 Prometheus Server                   │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │TSDB      │  │Alert     │  │  Service         │  │   │
│  │  │ Storage  │  │Manager   │  │  Discovery       │  │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  └────────────────────────────────────────────────────┘   │
│                    │          │          │                  │
│         ┌──────────┘          │          └──────────┐      │
│         ▼                     ▼                     ▼      │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐        │
│  │ Grafana  │      │   Loki   │      │  Tempo   │        │
│  │Dashboard │      │  Logs    │      │  Traces  │        │
│  └──────────┘      └──────────┘      └──────────┘        │
│         │                │                │               │
│         └────────────────┼────────────────┘               │
│                          ▼                                 │
│  ┌────────────────────────────────────────────────────┐   │
│  │             Alerting & Notifications               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │Alertman. │  │  Slack   │  │     PagerDuty    │  │   │
│  │  │ Routes   │  │ Webhook  │  │     / OpsGenie   │  │   │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │   │
│  └────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

## Features

- **Metrics**: Prometheus for time-series metrics collection
- **Dashboards**: Grafana with custom, reusable dashboards
- **Logs**: Loki for centralized log aggregation
- **Traces**: Tempo for distributed tracing
- **Alerting**: Alertmanager with Slack/PagerDuty integration
- **Exporters**: Node Exporter, Blackbox Exporter, cAdvisor
- **Kubernetes**: Auto-discovery of pods and services

## Prerequisites

- Kubernetes cluster (or Docker Compose for local)
- Helm 3.x installed
- kubectl configured

## Installation

### Option 1: Helm (Kubernetes)

```bash
# Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin \
  --set prometheus.prometheusSpec.replicas=2

# Verify installation
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

### Option 2: Docker Compose

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./dashboards:/etc/grafana/provisioning/dashboards
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'

  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki_data:/loki

  tempo:
    image: grafana/tempo:latest
    ports:
      - "3200:3200"
      - "4317:4317"
    volumes:
      - ./tempo-config.yaml:/etc/tempo/config.yaml
      - tempo_data:/tmp/tempo

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
  tempo_data:
```

## Configuration

### Prometheus Config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'alerts/*.yml'

scrape_configs:
  - job_name: 'kubernetes-nodes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        replacement: '${1}:9100'
        target_label: __address__

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### Alert Rules

```yaml
# alerts/kubernetes-alerts.yml
groups:
  - name: kubernetes
    rules:
      - alert: NodeDown
        expr: up{job="kubernetes-nodes"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Node {{ $labels.node }} is down"
          description: "Node has been unreachable for more than 5 minutes"

      - alert: HighPodRestartRate
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High pod restart rate detected"
          description: "Pod {{ $labels.pod }} has restarted frequently"

      - alert: DiskSpaceLow
        expr: node_filesystem_avail_bytes{mountpoint="/"}
          / node_filesystem_size_bytes{mountpoint="/"} < 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space is below 10% on {{ $labels.instance }}"

      - alert: KafkaConsumerLag
        expr: max(kafka_consumer_lag) by (consumergroup) > 1000
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Kafka consumer lag > 1000"
          description: "Consumer group {{ $labels.consumergroup }} has high lag"
```

## Grafana Dashboards

### Dashboard Provisioning

```yaml
# dashboards/dashboard-provider.yml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

### Sample Dashboard JSON

```json
{
  "dashboard": {
    "title": "Kubernetes Cluster Overview",
    "panels": [
      {
        "title": "Cluster CPU Usage",
        "type": "timeseries",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(rate(container_cpu_usage_seconds_total[5m]))",
            "legendFormat": "CPU Usage"
          }
        ]
      },
      {
        "title": "Memory Usage",
        "type": "timeseries",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "sum(container_memory_usage_bytes) / 1024 / 1024 / 1024",
            "legendFormat": "Memory (GB)"
          }
        ]
      },
      {
        "title": "Pod Status",
        "type": "stat",
        "datasource": "Prometheus",
        "targets": [
          {
            "expr": "count(kube_pod_status_phase{phase='Running'})",
            "legendFormat": "Running Pods"
          }
        ]
      }
    ]
  }
}
```

## Usage

### Accessing Services

```bash
# Prometheus
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090

# Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Loki API
kubectl port-forward -n monitoring svc/loki 3100:3100
```

## Best Practices

- Set appropriate retention periods for metrics, logs, and traces
- Use recording rules for expensive queries
- Implement multi-tenancy for Grafana
- Use dashboards as code (provisioning)
- Set up proper alert routing (critical vs warning)
- Regular backup of Grafana dashboards
- Monitor the monitoring stack itself

## Security Considerations

- Enable authentication for all monitoring components
- Use TLS for data in transit
- Implement RBAC for Grafana
- Network policies to restrict access to exporters
- Regular updates of monitoring components
- Audit logging for configuration changes
- Rate limiting on Loki ingestion
