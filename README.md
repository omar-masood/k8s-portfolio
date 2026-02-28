# 📊 Kubernetes Monitoring Enhancement – Alertmanager + Webhook Integration

## 📌 Overview

This task extends the default `kube-prometheus-stack` Helm installation by:

* ✅ Enabling **Alertmanager**
* ✅ Configuring a **Webhook receiver**
* ✅ Routing all cluster alerts to the webhook
* ✅ Deploying a test webhook service to validate alert delivery

This demonstrates understanding of:

* Prometheus alerting
* Alertmanager routing
* Kubernetes CRDs (`AlertmanagerConfig`)
* Helm upgrade workflows
* In-cluster service communication

---

# 🧠 Architecture

```
Prometheus → Alertmanager → Webhook Receiver → Logs
```

1. Prometheus detects alert conditions.
2. Alertmanager receives alerts.
3. Alertmanager routes alerts to configured webhook.
4. Webhook service receives JSON payload.

---

# 🔧 Changes Made

## 1️⃣ Modified `monitoring-values.yaml`

### Enabled Alertmanager

```yaml
alertmanager:
  enabled: true

  alertmanagerSpec:
    replicas: 1
    resources:
      requests:
        memory: 64Mi
        cpu: 50m
      limits:
        memory: 128Mi
        cpu: 100m
```

### Why?

By default, Alertmanager was disabled for lightweight local development.

Enabling it allows:

* Alert routing
* Notification handling
* Integration with external systems

---

## 2️⃣ Created `alertmanager-webhook.yaml`

### Purpose

Defines routing configuration using the `AlertmanagerConfig` CRD.

```yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: webhook-receiver
  namespace: monitoring
  labels:
    release: monitoring
spec:
  route:
    receiver: "webhook"
    groupBy: ["alertname", "namespace"]
    groupWait: 10s
    groupInterval: 1m
    repeatInterval: 1h

  receivers:
    - name: "webhook"
      webhookConfigs:
        - url: "http://alert-receiver.monitoring.svc.cluster.local:8080/alerts"
          sendResolved: true
```

### What It Does

* Routes all alerts to a receiver named `webhook`
* Sends HTTP POST requests containing alert data
* Sends both firing and resolved alerts

---

## 3️⃣ Created `alert-receiver.yaml` (Test Webhook Service)

### Purpose

Deploys a simple HTTP echo server to verify alert delivery.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alert-receiver
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alert-receiver
  template:
    metadata:
      labels:
        app: alert-receiver
    spec:
      containers:
        - name: receiver
          image: mendhak/http-https-echo:31
          ports:
            - containerPort: 8080
          env:
            - name: HTTP_PORT
              value: "8080"
---
apiVersion: v1
kind: Service
metadata:
  name: alert-receiver
  namespace: monitoring
spec:
  selector:
    app: alert-receiver
  ports:
    - port: 8080
      targetPort: 8080
```

### What It Does

* Exposes a service inside the cluster
* Receives webhook POST requests
* Logs alert JSON payloads

---

# 🚀 Deployment Steps

## Upgrade Helm Release

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f monitoring-values.yaml
```

## Apply Alertmanager Configuration

```bash
kubectl apply -f alertmanager-webhook.yaml
```

## Deploy Webhook Receiver

```bash
kubectl apply -f alert-receiver.yaml
```

---

# 🧪 Testing the Setup

## Create a Failing Deployment

```bash
kubectl create deployment broken-app --image=nginx:doesnotexist
```

This causes:

* ImagePullBackOff
* Prometheus alert firing

---

## Check Prometheus Alerts

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090
```

Visit:

```
http://localhost:9090/alerts
```

---

## Check Alertmanager UI

```bash
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093
```

Visit:

```
http://localhost:9093
```

---

## Check Webhook Logs

```bash
kubectl logs -n monitoring deploy/alert-receiver -f
```

You should see JSON alert payloads when alerts fire.

---

# 📦 Example Alert Payload

```json
{
  "status": "firing",
  "alerts": [
    {
      "labels": {
        "alertname": "KubePodCrashLooping",
        "namespace": "default",
        "pod": "broken-app-xxxxx"
      }
    }
  ]
}
```

---

# 🏢 Real-World Usage

In production, the webhook receiver would typically forward alerts to:

* Slack
* Microsoft Teams
* PagerDuty
* ServiceNow
* Email (SMTP)
* Custom incident management systems

---

# 🎯 What This Demonstrates

This enhancement shows:

* Understanding of Prometheus alert lifecycle
* Alertmanager routing configuration
* Kubernetes CRDs usage
* Helm upgrade strategy
* Service-to-service networking in Kubernetes
* Monitoring observability best practices
 
