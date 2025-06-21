# 🔭 OpenTelemetry in Action on Kubernetes: Part 7 - Let There Be Logs, Observability’s Final Pillar with Loki

## 🪵 Logs: The Footprints of Your System

**Logs** are timestamped records of events that happen in your system — like breadcrumbs left behind by your application as it performs operations. They help you understand what happened, when it happened, and often why it happened.

In observability, logs play a key role when:

* Metrics show a spike but don’t tell you why.
* Traces reveal latency but not the root cause.
* You want to debug something that happened at 3 AM… last Thursday.

![OTel-k8s](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/OTel-k8s.png)

---

## 🧰 Meet Loki — Prometheus for Logs

**Loki**, built by the folks at Grafana Labs, is a log aggregation system designed to be:

* **Lightweight**: It indexes only labels, not the full log content.
* **Kubernetes-native**: Integrates beautifully with pod logs.
* **Prometheus-like**: Designed to feel familiar if you’ve used Prometheus.

Instead of shipping logs to a bulky ELK stack, Loki works smoothly with Promtail, FluentBit, or OpenTelemetry Collector to aggregate logs from across your cluster.

## 🚀 Deploying Loki in Kubernetes

Let’s deploy Loki using a simple YAML manifest that includes:

1. A **Deployment** to run the Loki service
2. A **Service** to expose Loki inside the cluster
3. A **ConfigMap** to configure how Loki receives and stores logs

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
  labels:
    app: loki
data:
  loki.yaml: |
    auth_enabled: false

    server:
      http_listen_port: 3100

    common:
      path_prefix: /loki
      ring:
        instance_addr: 127.0.0.1
        kvstore:
          store: inmemory

    ingester_client:
      grpc_client_config:
        max_send_msg_size: 104857600
        max_recv_msg_size: 104857600
      remote_timeout: 5s

    ingester:
      lifecycler:
        ring:
          kvstore:
            store: inmemory
          replication_factor: 1

    schema_config:
      configs:
        - from: 2020-10-27
          store: boltdb-shipper
          object_store: filesystem
          schema: v11
          index:
            prefix: index_
            period: 24h

    storage_config:
      boltdb_shipper:
        active_index_directory: /loki/index
        cache_location: /loki/cache
        shared_store: filesystem
      filesystem:
        directory: /loki/chunks

    limits_config:
      enforce_metric_name: false
      max_streams_per_user: 0
      max_chunks_per_query: 1000000
      max_query_series: 50000
      max_query_lookback: 720h

    ruler:
      storage:
        type: local
        local:
          directory: /loki/rules
      ring:
        kvstore:
          store: inmemory

    analytics:
      reporting_enabled: false

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: loki
  labels:
    app: loki
spec:
  replicas: 1
  selector:
    matchLabels:
      app: loki
  template:
    metadata:
      labels:
        app: loki
    spec:
      containers:
        - name: loki
          image: grafana/loki:2.9.2
          args:
            - "-config.file=/etc/loki/loki.yaml"
          ports:
            - name: http
              containerPort: 3100
          volumeMounts:
            - name: config
              mountPath: /etc/loki
              readOnly: true
            - name: storage
              mountPath: /loki
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: config
          configMap:
            name: loki-config
        - name: storage
          emptyDir: {}

---

apiVersion: v1
kind: Service
metadata:
  name: loki
  labels:
    app: loki
spec:
  selector:
    app: loki
  ports:
    - name: http-metrics
      port: 3100
      targetPort: 3100
```

The Loki manifest sets up a log aggregator inside your Kubernetes cluster that listens for incoming logs on a defined port. The service makes Loki accessible to other components, such as the OTEL Collector, while the ConfigMap gives Loki its brain — deciding how logs flow and where they go.

## ⚙️ Updating OpenTelemetry Collector to Send Logs to Loki

We now need to tell the **OTEL Collector Agent** to collect logs using the `filelog` receiver and ship them off to Loki. Here's the flow:

* `filelog`: Reads logs from Kubernetes pod files.
* `loki exporter`: Pushes these logs to the Loki service using HTTP.

```yaml
receivers:
  filelog:
    include: [ /var/log/pods/*/*/*.log ]
    start_at: beginning
    include_file_path: true
    include_file_name: true

exporters:
  loki:
    endpoint: "http://loki.observability.svc.cluster.local:3100/loki/api/v1/push"
    tls:
      insecure: true
    sending_queue:
      enabled: true

service:
  pipelines:
    # collect logs using 'filelog' receiver and ship them to loki
    logs:
      receivers: [filelog]
      processors: [memory_limiter, batch]
      exporters: [loki]
```

## 🚀 Deploying Loki to Kubernetes

```bash
# Apply the Loki manifests
kubectl -n observability apply -f loki.yaml

# Verify Loki is running
kubectl -n observability get pods -l app=loki

# check readiness of loki
curl -X GET "http://$(kubectl -n observability get svc -l app=loki -o json | jq -r '.items[].spec.clusterIP'):3100/ready"
```

![loki-curl-ready](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/loki-curl-ready.png)

---

## 👀 What’s Next?

In **Part 8**, we’ll bring everything together with **Grafana** — the ultimate observability dashboard:

* Visualizing traces from **Jaeger**
* Querying metrics from **Prometheus**
* Searching logs from **Loki**
* All in a single unified interface.

The observability trifecta — complete, powerful, and open source. Stay tuned. 🎯

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
