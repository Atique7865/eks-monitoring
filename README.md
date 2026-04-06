# 📊 Production-Grade EKS Monitoring with Prometheus & Grafana

> Complete observability stack for Amazon EKS with alerting via Alertmanager, Slack, PagerDuty, and email notifications.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Directory Structure](#directory-structure)
- [Step 1 — Prepare EKS Cluster](#step-1--prepare-eks-cluster)
- [Step 2 — Install Helm & Add Repos](#step-2--install-helm--add-repos)
- [Step 3 — Create Monitoring Namespace](#step-3--create-monitoring-namespace)
- [Step 4 — Configure Persistent Storage](#step-4--configure-persistent-storage)
- [Step 5 — Deploy kube-prometheus-stack](#step-5--deploy-kube-prometheus-stack)
- [Step 6 — Instrument Your Application](#step-6--instrument-your-application)
- [Step 7 — Configure ServiceMonitor](#step-7--configure-servicemonitor)
- [Step 8 — Create Alerting Rules](#step-8--create-alerting-rules)
- [Step 9 — Configure Alertmanager Notifications](#step-9--configure-alertmanager-notifications)
- [Step 10 — Import Grafana Dashboards](#step-10--import-grafana-dashboards)
- [Step 11 — Expose via Ingress (HTTPS)](#step-11--expose-via-ingress-https)
- [Step 12 — Backup & Retention Policy](#step-12--backup--retention-policy)
- [Verification & Testing](#verification--testing)
- [Troubleshooting](#troubleshooting)
- [Security Hardening](#security-hardening)
- [Upgrade & Maintenance](#upgrade--maintenance)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         EKS Cluster                             │
│                                                                 │
│  ┌──────────────┐    scrape     ┌─────────────────────────┐    │
│  │  Your App    │◄──────────────│      Prometheus          │    │
│  │  (metrics)   │               │  (metrics collection)   │    │
│  └──────────────┘               └────────────┬────────────┘    │
│                                              │                  │
│  ┌──────────────┐               ┌────────────▼────────────┐    │
│  │  Node        │               │     Alertmanager         │    │
│  │  Exporter    │               │  (alert routing)         │    │
│  └──────────────┘               └────────────┬────────────┘    │
│                                              │                  │
│  ┌──────────────┐               ┌────────────▼────────────┐    │
│  │  kube-state  │               │        Grafana           │    │
│  │  -metrics    │               │  (visualization)         │    │
│  └──────────────┘               └─────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
         │                              │
         ▼                              ▼
  ┌─────────────┐              ┌────────────────┐
  │   Slack /   │              │  PagerDuty /   │
  │   Email     │              │  OpsGenie      │
  └─────────────┘              └────────────────┘
```

### Components

| Component | Purpose |
|-----------|---------|
| **Prometheus** | Metrics collection, storage, and alerting engine |
| **Alertmanager** | Alert deduplication, grouping, and routing to receivers |
| **Grafana** | Dashboards and visualization |
| **kube-state-metrics** | Kubernetes object state metrics |
| **node-exporter** | Host-level metrics (CPU, memory, disk) |
| **Prometheus Operator** | Manages Prometheus/Alertmanager via CRDs |

---

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| AWS CLI | ≥ 2.x | Configured with EKS permissions |
| kubectl | ≥ 1.28 | Matching your EKS version |
| Helm | ≥ 3.12 | Package manager for Kubernetes |
| eksctl | ≥ 0.170 | Optional, for cluster setup |
| EKS Cluster | ≥ 1.28 | Running with worker nodes |
| EBS CSI Driver | Latest | For persistent volumes |

### Required IAM Permissions

Your EKS node role or service account must have:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:DeleteVolume",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumeStatus",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Directory Structure

```
eks-monitoring/
├── README.md
├── values/
│   ├── kube-prometheus-stack-values.yaml   # Main Helm values
│   └── grafana-dashboards/                 # Custom dashboards (JSON)
├── manifests/
│   ├── namespace.yaml
│   ├── storage-class.yaml
│   ├── service-monitor.yaml               # Scrape your app
│   ├── prometheus-rules.yaml              # Alert rules
│   ├── alertmanager-config.yaml           # Notification config
│   └── ingress.yaml                       # Grafana/Prometheus ingress
└── scripts/
    ├── install.sh
    └── uninstall.sh
```

---

## Step 1 — Prepare EKS Cluster

### 1.1 Verify cluster access

```bash
aws eks update-kubeconfig --region <AWS_REGION> --name <CLUSTER_NAME>
kubectl cluster-info
kubectl get nodes
```

### 1.2 Install EBS CSI Driver (required for persistent volumes)

```bash
# Add the EBS CSI driver addon
aws eks create-addon \
  --cluster-name <CLUSTER_NAME> \
  --addon-name aws-ebs-csi-driver \
  --region <AWS_REGION>

# Verify
kubectl get pods -n kube-system -l app=ebs-csi-controller
```

### 1.3 Create StorageClass

```yaml
# manifests/storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-monitoring
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain        # IMPORTANT: Retain data on PVC delete
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
```

```bash
kubectl apply -f manifests/storage-class.yaml
```

---

## Step 2 — Install Helm & Add Repos

```bash
# Install Helm (if not present)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add required Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana             https://grafana.github.io/helm-charts
helm repo update

# Verify
helm search repo prometheus-community/kube-prometheus-stack --versions | head -5
```

---

## Step 3 — Create Monitoring Namespace

```yaml
# manifests/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring
    environment: production
```

```bash
kubectl apply -f manifests/namespace.yaml
```

---

## Step 4 — Configure Persistent Storage

Persistent volumes ensure metrics and dashboards survive pod restarts.

| Component | Recommended Size | Notes |
|-----------|-----------------|-------|
| Prometheus | 100Gi | ~15 days retention at moderate scale |
| Alertmanager | 2Gi | Alert history |
| Grafana | 10Gi | Dashboards and plugins |

> **Production tip:** Size Prometheus storage using this formula:
> `needed_bytes = retention_seconds × ingested_samples_per_second × bytes_per_sample`
> Average `bytes_per_sample` ≈ 1–2 bytes with compression.

---

## Step 5 — Deploy kube-prometheus-stack

### 5.1 Create Helm values file

```yaml
# values/kube-prometheus-stack-values.yaml

# ─────────────────────────────────────────
# Global
# ─────────────────────────────────────────
fullnameOverride: "monitoring"

# ─────────────────────────────────────────
# Prometheus
# ─────────────────────────────────────────
prometheus:
  prometheusSpec:
    replicas: 2                    # HA — two Prometheus instances
    retention: 15d
    retentionSize: "90GB"

    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3-monitoring
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi

    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 8Gi

    # Allow Prometheus to discover ServiceMonitors across all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}

    # Allow PodMonitors across all namespaces
    podMonitorSelectorNilUsesHelmValues: false
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}

    # Security context
    securityContext:
      runAsNonRoot: true
      runAsUser: 65534
      fsGroup: 65534

    # Tolerations / node affinity (optional — pin to monitoring nodes)
    # affinity:
    #   nodeAffinity:
    #     requiredDuringSchedulingIgnoredDuringExecution:
    #       nodeSelectorTerms:
    #         - matchExpressions:
    #             - key: node-role
    #               operator: In
    #               values: ["monitoring"]

  service:
    type: ClusterIP

# ─────────────────────────────────────────
# Alertmanager
# ─────────────────────────────────────────
alertmanager:
  alertmanagerSpec:
    replicas: 2                    # HA

    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3-monitoring
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 2Gi

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

  service:
    type: ClusterIP

# ─────────────────────────────────────────
# Grafana
# ─────────────────────────────────────────
grafana:
  enabled: true
  replicas: 1

  adminUser: admin
  # Reference a Kubernetes secret instead of plain text
  adminPassword: ""              # Leave empty — set via secret below
  admin:
    existingSecret: grafana-admin-secret
    userKey: admin-user
    passwordKey: admin-password

  persistence:
    enabled: true
    storageClassName: gp3-monitoring
    size: 10Gi
    accessModes: ["ReadWriteOnce"]

  resources:
    requests:
      cpu: 200m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

  grafana.ini:
    server:
      root_url: "https://grafana.yourdomain.com"
    security:
      allow_embedding: false
      cookie_secure: true
      strict_transport_security: true
    auth:
      disable_login_form: false
    analytics:
      reporting_enabled: false
      check_for_updates: false

  # Pre-load dashboards from ConfigMaps
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: ALL
    datasources:
      enabled: true

  # Built-in datasource pointing to Prometheus
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://monitoring-prometheus:9090
          isDefault: true
          access: proxy

  service:
    type: ClusterIP

# ─────────────────────────────────────────
# kube-state-metrics
# ─────────────────────────────────────────
kube-state-metrics:
  enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

# ─────────────────────────────────────────
# node-exporter
# ─────────────────────────────────────────
prometheus-node-exporter:
  enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi

# ─────────────────────────────────────────
# Default alert rules
# ─────────────────────────────────────────
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: true
    configReloaders: true
    general: true
    k8s: true
    kubeApiserver: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubeApiserverHistogram: true
    kubeApiserverSlos: true
    kubelet: true
    kubeProxy: true
    kubePrometheusGeneral: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    kubeScheduler: true
    kubeStateMetrics: true
    network: true
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    prometheus: true
    prometheusOperator: true
```

### 5.2 Create Grafana admin secret

```bash
kubectl create secret generic grafana-admin-secret \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<STRONG_PASSWORD_HERE>'
```

### 5.3 Install the stack

```bash
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values values/kube-prometheus-stack-values.yaml \
  --atomic \
  --timeout 10m \
  --version 58.2.2        # Pin to a specific chart version

# Verify all pods are running
kubectl get pods -n monitoring
```

Expected output:
```
NAME                                                   READY   STATUS    RESTARTS
alertmanager-monitoring-alertmanager-0                 2/2     Running   0
alertmanager-monitoring-alertmanager-1                 2/2     Running   0
monitoring-grafana-xxxxxxxxxx-xxxxx                    3/3     Running   0
monitoring-kube-prometheus-operator-xxxxxxxxxx-xxxxx  1/1     Running   0
monitoring-kube-state-metrics-xxxxxxxxxx-xxxxx        1/1     Running   0
monitoring-prometheus-node-exporter-xxxxx             1/1     Running   0
prometheus-monitoring-prometheus-0                    2/2     Running   0
prometheus-monitoring-prometheus-1                    2/2     Running   0
```

---

## Step 6 — Instrument Your Application

Your application must expose a `/metrics` endpoint in **Prometheus format**.

### Option A — Application already exposes metrics

Ensure your app serves metrics on a dedicated port (e.g., `9090` or `8080/metrics`).

### Option B — Add a metrics client library

**Node.js (prom-client):**
```javascript
const client = require('prom-client');
const register = client.register;
client.collectDefaultMetrics();

// Custom metric example
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
});

app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

**Python (prometheus_client):**
```python
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
import time

REQUEST_COUNT   = Counter('app_requests_total', 'Total requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('app_request_latency_seconds', 'Request latency', ['endpoint'])

@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}
```

### 6.1 Expose metrics port in your Deployment

```yaml
# Your application Deployment — add metrics port
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-application
  namespace: default          # Change to your app namespace
  labels:
    app: my-application
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-application
  template:
    metadata:
      labels:
        app: my-application
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port:   "8080"
        prometheus.io/path:   "/metrics"
    spec:
      containers:
        - name: my-application
          image: your-image:tag
          ports:
            - name: http
              containerPort: 8080
            - name: metrics           # Named port for ServiceMonitor
              containerPort: 8080
          # Readiness & liveness probes are REQUIRED for proper alerting
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
            failureThreshold: 3
```

### 6.2 Create a Service for your app

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-application
  namespace: default
  labels:
    app: my-application
    monitoring: "true"         # Label used by ServiceMonitor selector
spec:
  selector:
    app: my-application
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: metrics
      port: 8080
      targetPort: 8080
```

---

## Step 7 — Configure ServiceMonitor

A `ServiceMonitor` tells Prometheus Operator which services to scrape.

```yaml
# manifests/service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-application
  namespace: monitoring         # Deploy in monitoring namespace
  labels:
    release: kube-prometheus-stack   # Must match Prometheus serviceMonitorSelector
spec:
  namespaceSelector:
    matchNames:
      - default                  # Namespace where your app lives
  selector:
    matchLabels:
      app: my-application
      monitoring: "true"
  endpoints:
    - port: metrics              # Must match Service port name
      path: /metrics
      interval: 30s              # Scrape every 30 seconds
      scrapeTimeout: 10s
      honorLabels: true
```

```bash
kubectl apply -f manifests/service-monitor.yaml

# Verify Prometheus discovers the target (wait ~1 min)
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090 &
# Open http://localhost:9090/targets — look for your app
```

---

## Step 8 — Create Alerting Rules

```yaml
# manifests/prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-application-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:

    # ──────────────────────────────────────────
    # Application Availability
    # ──────────────────────────────────────────
    - name: application.availability
      interval: 30s
      rules:

        - alert: AppDown
          expr: |
            absent(up{job="my-application"}) == 1
            OR
            up{job="my-application"} == 0
          for: 1m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "Application {{ $labels.instance }} is DOWN"
            description: >
              The application instance {{ $labels.instance }} has been unreachable
              for more than 1 minute. Immediate action required.
            runbook_url: "https://wiki.yourcompany.com/runbooks/app-down"

        - alert: AppHighErrorRate
          expr: |
            sum(rate(http_request_duration_seconds_count{status_code=~"5.."}[5m]))
            /
            sum(rate(http_request_duration_seconds_count[5m])) > 0.05
          for: 5m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "High error rate on application"
            description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"

        - alert: AppHighLatency
          expr: |
            histogram_quantile(0.95,
              sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
            ) > 2
          for: 5m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "High p95 latency on {{ $labels.job }}"
            description: "p95 latency is {{ $value | humanizeDuration }} (threshold: 2s)"

        - alert: AppNoTraffic
          expr: |
            sum(rate(http_request_duration_seconds_count[10m])) == 0
          for: 10m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "No traffic to application for 10 minutes"
            description: "Application may be down or disconnected from load balancer."

    # ──────────────────────────────────────────
    # Pod Health
    # ──────────────────────────────────────────
    - name: application.pods
      rules:

        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total{
              namespace="default",
              pod=~"my-application.*"
            }[15m]) > 3
          for: 5m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "Pod {{ $labels.pod }} is crash-looping"
            description: "Pod {{ $labels.pod }} restarted {{ $value }} times in 15 minutes."

        - alert: PodNotReady
          expr: |
            kube_pod_status_ready{
              namespace="default",
              pod=~"my-application.*",
              condition="true"
            } == 0
          for: 5m
          labels:
            severity: critical
            team: backend
          annotations:
            summary: "Pod {{ $labels.pod }} is not ready"
            description: "Pod {{ $labels.pod }} has been in a non-ready state for more than 5 minutes."

        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_spec_replicas{namespace="default", deployment="my-application"}
            !=
            kube_deployment_status_available_replicas{namespace="default", deployment="my-application"}
          for: 10m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "Deployment replicas mismatch for {{ $labels.deployment }}"
            description: >
              Desired replicas: {{ $value }}. Available replicas may be less.
              Check for pending pods or resource pressure.

    # ──────────────────────────────────────────
    # Resource Utilization
    # ──────────────────────────────────────────
    - name: application.resources
      rules:

        - alert: PodHighMemoryUsage
          expr: |
            sum(container_memory_working_set_bytes{
              namespace="default",
              pod=~"my-application.*",
              container!=""
            }) by (pod)
            /
            sum(kube_pod_container_resource_limits{
              namespace="default",
              pod=~"my-application.*",
              resource="memory"
            }) by (pod) > 0.85
          for: 10m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "Pod {{ $labels.pod }} memory usage > 85%"
            description: "Memory usage is {{ $value | humanizePercentage }}"

        - alert: PodHighCPUUsage
          expr: |
            sum(rate(container_cpu_usage_seconds_total{
              namespace="default",
              pod=~"my-application.*",
              container!=""
            }[5m])) by (pod)
            /
            sum(kube_pod_container_resource_limits{
              namespace="default",
              pod=~"my-application.*",
              resource="cpu"
            }) by (pod) > 0.85
          for: 10m
          labels:
            severity: warning
            team: backend
          annotations:
            summary: "Pod {{ $labels.pod }} CPU usage > 85%"
            description: "CPU usage is {{ $value | humanizePercentage }}"
```

```bash
kubectl apply -f manifests/prometheus-rules.yaml

# Verify rules are loaded
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090 &
# Open http://localhost:9090/alerts
```

---

## Step 9 — Configure Alertmanager Notifications

### 9.1 Create notification secrets

```bash
# Slack
kubectl create secret generic alertmanager-notifications \
  --namespace monitoring \
  --from-literal=slack-webhook-url='https://hooks.slack.com/services/XXX/YYY/ZZZ' \
  --from-literal=pagerduty-routing-key='YOUR_PAGERDUTY_ROUTING_KEY' \
  --from-literal=smtp-password='YOUR_SMTP_PASSWORD'
```

### 9.2 Alertmanager configuration

```yaml
# manifests/alertmanager-config.yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: my-application-alerts
  namespace: monitoring
  labels:
    alertmanagerConfig: "true"
spec:

  # ──────────────────────────────────────────
  # Route tree
  # ──────────────────────────────────────────
  route:
    groupBy: ['alertname', 'namespace', 'severity']
    groupWait:      30s     # Wait before sending first notification in a group
    groupInterval:  5m      # Wait before sending next notification for same group
    repeatInterval: 1h      # Re-notify if alert is still firing

    receiver: slack-warnings   # Default receiver

    routes:
      # Critical alerts → PagerDuty + Slack #critical
      - matchers:
          - name: severity
            value: critical
        receiver: pagerduty-critical
        continue: true         # Also send to next matching route

      - matchers:
          - name: severity
            value: critical
        receiver: slack-critical

      # Warning alerts → Slack #warnings
      - matchers:
          - name: severity
            value: warning
        receiver: slack-warnings

  # ──────────────────────────────────────────
  # Inhibition rules (suppress child alerts
  # when parent alert is firing)
  # ──────────────────────────────────────────
  inhibitRules:
    - sourceMatch:
        - name: alertname
          value: AppDown
      targetMatch:
        - name: alertname
          value: AppHighErrorRate
      equal: ['namespace', 'job']

    - sourceMatch:
        - name: alertname
          value: AppDown
      targetMatch:
        - name: alertname
          value: AppHighLatency
      equal: ['namespace', 'job']

  # ──────────────────────────────────────────
  # Receivers
  # ──────────────────────────────────────────
  receivers:

    - name: slack-critical
      slackConfigs:
        - apiURL:
            name: alertmanager-notifications
            key: slack-webhook-url
          channel: '#alerts-critical'
          sendResolved: true
          iconEmoji: ':red_circle:'
          username: 'Prometheus Alertmanager'
          title: |-
            [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}]
            {{ .CommonLabels.alertname }}
          text: |-
            {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Severity:* `{{ .Labels.severity }}`
            *Namespace:* `{{ .Labels.namespace }}`
            *Description:* {{ .Annotations.description }}
            *Runbook:* {{ .Annotations.runbook_url }}
            *Started:* {{ .StartsAt | since }}
            {{ end }}

    - name: slack-warnings
      slackConfigs:
        - apiURL:
            name: alertmanager-notifications
            key: slack-webhook-url
          channel: '#alerts-warnings'
          sendResolved: true
          iconEmoji: ':warning:'
          username: 'Prometheus Alertmanager'
          title: |-
            [{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}
          text: |-
            {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Description:* {{ .Annotations.description }}
            {{ end }}

    - name: pagerduty-critical
      pagerdutyConfigs:
        - routingKey:
            name: alertmanager-notifications
            key: pagerduty-routing-key
          sendResolved: true
          severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
          description: '{{ .CommonAnnotations.summary }}'
          details:
            firing: '{{ .Alerts.Firing | len }}'
            description: '{{ .CommonAnnotations.description }}'
            runbook: '{{ .CommonAnnotations.runbook_url }}'

    - name: email-alerts
      emailConfigs:
        - to: 'oncall-team@yourcompany.com'
          from: 'alertmanager@yourcompany.com'
          smarthost: 'smtp.yourprovider.com:587'
          authUsername: 'alertmanager@yourcompany.com'
          authPassword:
            name: alertmanager-notifications
            key: smtp-password
          sendResolved: true
          requireTLS: true
          headers:
            subject: '[{{ .Status | toUpper }}] {{ .CommonLabels.alertname }}'
          html: |-
            <h2>{{ .CommonAnnotations.summary }}</h2>
            <p><strong>Severity:</strong> {{ .CommonLabels.severity }}</p>
            <p><strong>Description:</strong> {{ .CommonAnnotations.description }}</p>
            <p><a href="{{ .CommonAnnotations.runbook_url }}">View Runbook</a></p>
```

```bash
kubectl apply -f manifests/alertmanager-config.yaml
```

### 9.3 Update Alertmanager to use AlertmanagerConfig CRDs

Add to `values/kube-prometheus-stack-values.yaml` under `alertmanager`:

```yaml
alertmanager:
  alertmanagerSpec:
    alertmanagerConfigSelector:
      matchLabels:
        alertmanagerConfig: "true"
    alertmanagerConfigNamespaceSelector: {}

  config:
    global:
      resolve_timeout: 5m
      http_config:
        follow_redirects: true
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'null'
      routes:
        - matchers:
            - alertname =~ "InfoInhibitor|Watchdog"
          receiver: 'null'
    receivers:
      - name: 'null'
    inhibit_rules:
      - source_matchers:
          - severity = critical
        target_matchers:
          - severity =~ warning|info
        equal: [alertname, namespace, pod]
```

Then upgrade:

```bash
helm upgrade kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values/kube-prometheus-stack-values.yaml \
  --atomic --timeout 10m
```

---

## Step 10 — Import Grafana Dashboards

### 10.1 Pre-built dashboards (recommended)

Access Grafana and import by dashboard ID from grafana.com:

| Dashboard | ID | Purpose |
|-----------|-----|---------|
| Kubernetes Cluster Overview | `7249` | Nodes, pods, resources |
| Kubernetes Deployment | `8588` | Per-deployment metrics |
| Node Exporter Full | `1860` | Host metrics |
| Kubernetes Namespace | `6417` | Per-namespace view |
| AlertManager Overview | `9578` | Alert history |

**Import steps:**
1. Open Grafana → **Dashboards** → **Import**
2. Enter the dashboard ID → **Load**
3. Select **Prometheus** as the data source → **Import**

### 10.2 Custom application dashboard via ConfigMap

```yaml
# Auto-loaded by Grafana sidecar
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-application-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"      # Triggers sidecar to load this
data:
  my-application.json: |
    {
      "title": "My Application",
      "uid": "my-application",
      "panels": [
        {
          "title": "Request Rate",
          "type": "stat",
          "targets": [{
            "expr": "sum(rate(http_request_duration_seconds_count[5m]))",
            "legendFormat": "req/s"
          }]
        },
        {
          "title": "Error Rate",
          "type": "timeseries",
          "targets": [{
            "expr": "sum(rate(http_request_duration_seconds_count{status_code=~\"5..\"}[5m])) / sum(rate(http_request_duration_seconds_count[5m]))",
            "legendFormat": "Error Rate"
          }]
        },
        {
          "title": "p95 Latency",
          "type": "timeseries",
          "targets": [{
            "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))",
            "legendFormat": "p95"
          }]
        }
      ]
    }
```

```bash
kubectl apply -f manifests/grafana-dashboard-configmap.yaml
```

---

## Step 11 — Expose via Ingress (HTTPS)

### 11.1 Install AWS Load Balancer Controller (if not present)

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=<CLUSTER_NAME> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 11.2 Create Ingress for Grafana

```yaml
# manifests/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing     # Use "internal" for private
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:REGION:ACCOUNT:certificate/CERT_ID"
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-2021-06
    alb.ingress.kubernetes.io/wafv2-acl-arn: "arn:aws:wafv2:..."   # Optional WAF
    # Restrict access to your corporate IP range
    alb.ingress.kubernetes.io/inbound-cidrs: "203.0.113.0/24"
spec:
  rules:
    - host: grafana.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: monitoring-grafana
                port:
                  number: 80
    - host: prometheus.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: monitoring-prometheus
                port:
                  number: 9090
```

```bash
kubectl apply -f manifests/ingress.yaml

# Get the ALB DNS name
kubectl get ingress -n monitoring monitoring-ingress
```

---

## Step 12 — Backup & Retention Policy

### 12.1 Prometheus data backup with Thanos (recommended for long-term storage)

```yaml
# In kube-prometheus-stack-values.yaml, add to prometheusSpec:
prometheus:
  prometheusSpec:
    thanos:
      image: quay.io/thanos/thanos:v0.34.0
      objectStorageConfig:
        name: thanos-objstore-secret
        key: objstore.yml
```

Create S3 bucket config:

```bash
kubectl create secret generic thanos-objstore-secret \
  --namespace monitoring \
  --from-file=objstore.yml=- <<EOF
type: S3
config:
  bucket: my-thanos-metrics-bucket
  region: us-east-1
  endpoint: s3.amazonaws.com
  sse_config:
    type: SSE-S3
EOF
```

### 12.2 Grafana backup via Velero

```bash
# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backup-bucket \
  --backup-location-config region=us-east-1 \
  --use-node-agent

# Schedule daily backup of monitoring namespace
velero schedule create monitoring-daily \
  --schedule="0 2 * * *" \
  --include-namespaces monitoring \
  --ttl 720h                           # 30 days retention
```

---

## Verification & Testing

### Verify Prometheus is scraping your app

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090

# Check targets in browser: http://localhost:9090/targets
# Run a test query: up{job="my-application"}
```

### Send a test alert

```bash
# Temporarily kill your application pods
kubectl scale deployment my-application --replicas=0 -n default

# Watch for AppDown alert firing (~1 min)
kubectl port-forward -n monitoring svc/monitoring-alertmanager 9093:9093
# Open: http://localhost:9093/#/alerts

# Restore
kubectl scale deployment my-application --replicas=2 -n default
```

### Verify Slack notifications

```bash
# Manually send a test alert via Alertmanager API
curl -XPOST http://localhost:9093/api/v1/alerts \
  -H 'Content-Type: application/json' \
  -d '[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "warning",
      "job": "my-application"
    },
    "annotations": {
      "summary": "This is a test alert",
      "description": "Testing Alertmanager notification pipeline"
    },
    "startsAt": "'"$(date -u +%Y-%m-%dT%H:%M:%SZ)"'",
    "endsAt":   "'"$(date -u -d '+10 minutes' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v+10M +%Y-%m-%dT%H:%M:%SZ)"'"
  }]'
```

---

## Troubleshooting

### Prometheus not scraping your app

```bash
# 1. Check ServiceMonitor is created
kubectl get servicemonitor -n monitoring

# 2. Check Prometheus targets
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090
# http://localhost:9090/targets — look for errors

# 3. Check labels match
kubectl get svc my-application -n default --show-labels
# Ensure "monitoring: true" label is present

# 4. Check Prometheus operator logs
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-operator
```

### Alerts not reaching Slack

```bash
# 1. Check Alertmanager config is valid
kubectl get alertmanagerconfig -n monitoring

# 2. Check Alertmanager logs
kubectl logs -n monitoring alertmanager-monitoring-alertmanager-0 -c alertmanager

# 3. Verify secret exists
kubectl get secret alertmanager-notifications -n monitoring

# 4. Check Alertmanager UI
kubectl port-forward -n monitoring svc/monitoring-alertmanager 9093:9093
# http://localhost:9093 — check Status > Config
```

### Grafana can't reach Prometheus

```bash
# Check datasource from Grafana pod
kubectl exec -n monitoring deploy/monitoring-grafana -c grafana -- \
  curl -s http://monitoring-prometheus:9090/-/ready
```

### PVC stuck in Pending state

```bash
# Check storage class
kubectl get storageclass

# Check PVC events
kubectl describe pvc -n monitoring

# Verify EBS CSI driver is running
kubectl get pods -n kube-system -l app=ebs-csi-controller
```

---

## Security Hardening

### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prometheus-network-policy
  namespace: monitoring
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: prometheus
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: grafana
      ports:
        - port: 9090
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
  egress:
    - {}   # Allow all outbound for scraping
```

### RBAC for Grafana viewers

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: grafana-viewer
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: grafana-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: grafana-viewer
subjects:
  - kind: ServiceAccount
    name: monitoring-grafana
    namespace: monitoring
```

### Grafana LDAP / OIDC (production recommended)

Add to `grafana.ini` in Helm values:

```ini
[auth.generic_oauth]
enabled = true
name = AWS SSO
client_id = YOUR_CLIENT_ID
client_secret = YOUR_CLIENT_SECRET
scopes = openid email profile
auth_url = https://your-okta-domain/oauth2/v1/authorize
token_url = https://your-okta-domain/oauth2/v1/token
api_url = https://your-okta-domain/oauth2/v1/userinfo
role_attribute_path = contains(groups[*], 'grafana-admins') && 'Admin' || 'Viewer'
```

---

## Upgrade & Maintenance

### Upgrade kube-prometheus-stack

```bash
# Always check release notes for breaking changes first
# https://github.com/prometheus-community/helm-charts/releases

helm repo update

# Diff before upgrading
helm diff upgrade kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values/kube-prometheus-stack-values.yaml

# Apply upgrade
helm upgrade kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values values/kube-prometheus-stack-values.yaml \
  --atomic --timeout 15m
```

### Check component versions

```bash
helm list -n monitoring
kubectl get pods -n monitoring -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u
```

---

## Quick Reference

```bash
# Access Grafana locally
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
# URL: http://localhost:3000  (admin / <your-password>)

# Access Prometheus locally
kubectl port-forward -n monitoring svc/monitoring-prometheus 9090:9090
# URL: http://localhost:9090

# Access Alertmanager locally
kubectl port-forward -n monitoring svc/monitoring-alertmanager 9093:9093
# URL: http://localhost:9093

# View all monitoring resources
kubectl get all -n monitoring

# View firing alerts
kubectl get prometheusrule -n monitoring
```

---

## License

MIT — See [LICENSE](LICENSE) for details.

---

> **Maintainer:** Your Team · **Last reviewed:** 2026-04 · **Cluster:** EKS 1.28+
