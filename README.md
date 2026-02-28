📊 Kubernetes Monitoring Enhancement – Alertmanager + Webhook Integration
📌 Overview
This task extends the default kube-prometheus-stack Helm installation by:
✅ Enabling Alertmanager
✅ Configuring a Webhook receiver
✅ Routing all cluster alerts to the webhook
✅ Deploying a test webhook service to validate alert delivery
This demonstrates understanding of:
Prometheus alerting
Alertmanager routing
Kubernetes CRDs (AlertmanagerConfig)
Helm upgrade workflows
In-cluster service communication
�� Architecture
Prometheus → Alertmanager → Webhook Receiver → Logs
Prometheus detects alert conditions.
Alertmanager receives alerts.
Alertmanager routes alerts to configured webhook.
Webhook service receives JSON payload.
🔧 Changes Made
1️⃣ Modified monitoring-values.yaml
Enabled Alertmanager
alertmanager:enabled: truealertmanagerSpec:replicas: 1resources:requests:memory: 64Micpu: 50mlimits:memory: 128Micpu: 100m
Why?
By default, Alertmanager was disabled for lightweight local development.
Enabling it allows:
Alert routing
Notification handling
Integration with external systems
2️⃣ Created alertmanager-webhook.yaml
Purpose
Defines routing configuration using the AlertmanagerConfig CRD.
apiVersion: monitoring.coreos.com/v1alpha1kind: AlertmanagerConfigmetadata:name: webhook-receivernamespace: monitoringlabels:release: monitoringspec:route:receiver: "webhook"groupBy: ["alertname", "namespace"]
    groupWait: 10sgroupInterval: 1mrepeatInterval: 1hreceivers:- name: "webhook"webhookConfigs:- url: "http://alert-receiver.monitoring.svc.cluster.local:8080/alerts"sendResolved: true
What It Does
Routes all alerts to a receiver named webhook
Sends HTTP POST requests containing alert data
Sends both firing and resolved alerts
3️⃣ Created alert-receiver.yaml (Test Webhook Service)
Purpose
Deploys a simple HTTP echo server to verify alert delivery.
apiVersion: apps/v1kind: Deploymentmetadata:name: alert-receivernamespace: monitoringspec:replicas: 1selector:matchLabels:app: alert-receivertemplate:metadata:labels:app: alert-receiverspec:containers:- name: receiverimage: mendhak/http-https-echo:31ports:- containerPort: 8080env:- name: HTTP_PORTvalue: "8080"---apiVersion: v1kind: Servicemetadata:name: alert-receivernamespace: monitoringspec:selector:app: alert-receiverports:- port: 8080targetPort: 8080
What It Does
Exposes a service inside the cluster
Receives webhook POST requests
Logs alert JSON payloads
🚀 Deployment Steps
Upgrade Helm Release
helm upgrade monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f monitoring-values.yaml
Apply Alertmanager Configuration
kubectl apply -f alertmanager-webhook.yaml
Deploy Webhook Receiver
kubectl apply -f alert-receiver.yaml
🧪 Testing the Setup
Create a Failing Deployment
kubectl create deployment broken-app --image=nginx:doesnotexist
This causes:
ImagePullBackOff
Prometheus alert firing
Check Prometheus Alerts
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-prometheus 9090
Visit:
http://localhost:9090/alerts
Check Alertmanager UI
kubectl port-forward -n monitoring svc/monitoring-kube-prometheus-alertmanager 9093
Visit:
http://localhost:9093
Check Webhook Logs
kubectl logs -n monitoring deploy/alert-receiver -f
You should see JSON alert payloads when alerts fire.
📦 Example Alert Payload
{"status": "firing","alerts": [{"labels": {"alertname": "KubePodCrashLooping","namespace": "default","pod": "broken-app-xxxxx"}}]}
🏢 Real-World Usage
In production, the webhook receiver would typically forward alerts to:
Slack
Microsoft Teams
PagerDuty
ServiceNow
Email (SMTP)
Custom incident management systems
🎯 What This Demonstrates
This enhancement shows:
Understanding of Prometheus alert lifecycle
Alertmanager routing configuration
Kubernetes CRDs usage
Helm upgrade strategy
Service-to-service networking in Kubernetes
Monitoring observability best practices
