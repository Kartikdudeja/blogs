# OpenTelemetry in Action on Kubernetes: Part 6 - Tracking Metrics with Prometheus and OpenTelemetry

Observability isn’t complete without **metrics**—the vital signs of your applications and services. In this part, we integrate **Prometheus** into our Kubernetes-based observability stack. You’ll learn how Prometheus works with OpenTelemetry, deploy it into your cluster, and finally visualize custom application metrics generated in [Part 2](./part-2_instrument_and_dockerize_mlapp_python_app.md).

![OTel-k8s](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/OTel-k8s.png)

---

## What is Prometheus?

**Prometheus** is an open-source monitoring system that scrapes metrics from configured targets, stores them in a time-series database, and allows you to query them using PromQL. It’s widely adopted in the Kubernetes ecosystem for infrastructure and application monitoring.

Prometheus doesn’t “pull” metrics directly from applications. Instead, apps **expose metrics at an endpoint**, and Prometheus regularly scrapes these endpoints to collect data.

When integrated with OpenTelemetry, the **OpenTelemetry Collector** acts as a bridge—it collects metrics from instrumented applications and exposes them in a Prometheus-compatible format.

## What are Metrics?

**Metrics** are numerical data points that capture the health, performance, and resource usage of your system. For example:

* API requests per second
* Response latency
* CPU and memory usage

In our app, we’ve already defined two custom metrics:

* `api_requests_total`: total number of requests per endpoint
* `api_latency_seconds`: histogram for API latency

Now, let’s expose them to Prometheus.

## Prometheus Deployment

Let’s deploy Prometheus into our Kubernetes cluster. You’ll need the following YAML file:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus:latest
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus"
            - "--log.level=debug"
          resources:
            requests:
              cpu: "10m"
              memory: "56Mi"
            limits:
              cpu: "20m"
              memory: "128Mi"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: config-volume
              mountPath: /etc/prometheus/
            - name: storage-volume
              mountPath: /prometheus
      volumes:
        - name: config-volume
          configMap:
            name: prometheus-config
        - name: storage-volume
          emptyDir: {}

---

apiVersion: v1
kind: Service
metadata:
  name: prometheus
  labels:
    app: prometheus
spec:
  selector:
    app: prometheus
  ports:
    - protocol: TCP
      port: 9090
      targetPort: 9090
  type: ClusterIP

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s

    scrape_configs:
      - job_name: 'prometheus'
        static_configs:
          - targets: ['localhost:9090']

      - job_name: 'otel-collector-agent'
        static_configs:
          - targets: ['otel-collector-agent.observability.svc.cluster.local:8889']
```

This YAML file deploys **Prometheus** into the Kubernetes cluster with three key components:

* A **Deployment** that runs the Prometheus server using the official image and mounts a configuration volume,
* A **Service** that exposes Prometheus on port `9090`, enabling access to its UI and scrape endpoint, and
* A **ConfigMap** that provides the Prometheus scrape configuration, telling it to scrape metrics from itself and from the OpenTelemetry Collector agent on port `8889`.

Together, these resources allow Prometheus to run continuously, collect metrics from OTEL, and expose them for querying and visualization.

## Prometheus Config Explained

Here’s a minimal Prometheus configuration we’ll use:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'otel-collector-agent'
    static_configs:
      - targets: ['otel-collector-agent.observability.svc.cluster.local:8889']
```

This configuration tells Prometheus to scrape metrics every **15 seconds**. It monitors itself (`localhost:9090`) and also scrapes the **OpenTelemetry Collector agent** at its service endpoint (`otel-collector-agent.observability.svc.cluster.local:8889`). This is where our app metrics are exposed.

## Updating OpenTelemetry Collector Config

We need to update our OTEL Collector configuration to export metrics to Prometheus.

Here’s the relevant config:

```yaml
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317  # receive traces and metrics from instrumented application

    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 80
        spike_limit_percentage: 15
      
      batch:
        send_batch_size: 1000
        timeout: 5s

    exporters:
      otlp/jaeger:
        endpoint: "http://jaeger.observability.svc.cluster.local:4317"  # export traces to jaeger
        tls:
          insecure: true

      prometheus:
        endpoint: "0.0.0.0:8889"
        enable_open_metrics: true
        resource_to_telemetry_conversion:
          enabled: true

    service:
      pipelines:
        # collect trace data using otlp receiver and send it to jaeger
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp/jaeger]

        # collect metrics from otlp receiver and expose in prometheus compatible format
        metrics:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
```

## Deploying Prometheus to Kubernetes

```bash
# Apply Prometheus config and deployment
kubectl -n observability apply -f prometheus.yaml
```

## Visualizing Custom Metrics

1. Make some API calls to the application:

   ```bash
   API_ENDPOINT_IP=$(kubectl -n mlapp get svc -l app=house-price-service -o json | jq -r '.items[].spec.clusterIP')

   curl -X POST "http://${API_ENDPOINT_IP}:80/predict/" \
    -H "Content-Type: application/json" \
    -d '{"features": [1200]}'
   ```

2. Open Prometheus UI:

   ```
   kubectl port-forward svc/prometheus -n observability 9090:9090
   ```

3. In the Prometheus UI (`http://localhost:9090`), search for:

   * `api_requests_total`
   * `api_latency_seconds`

You should see data flowing in!

![prometheus-ui](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/prometheus-ui.png)

---

## What’s Next?

Now that we’ve captured and visualized **metrics**, the observability story is coming together. But there’s still one pillar left—**logs**.

In [Part 7](./part-7_logs_with_logi_with_opentelemetry.md), we’ll deploy **Loki**, the log aggregation system, and configure the OpenTelemetry Collector to ship structured logs from our app to Loki. Stay tuned!

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
