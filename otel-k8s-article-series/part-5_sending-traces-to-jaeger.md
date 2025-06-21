# 🔭 OpenTelemetry in Action on Kubernetes: Part 5 - Tracing the Lines, Sending Spans from App to Jaeger

In the last part, we set up the OpenTelemetry Collector in **agent mode** to receive telemetry data from our ML app. But telemetry isn't useful if it's just sitting in logs, right? We want **end-to-end traces** that we can **visualize, search, and troubleshoot**.

And that’s exactly where **Jaeger** enters the scene.


![OTel-k8s](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bg7a2b1jejddbdfjb32s.png)

---

## 🕵️‍♂️ What is Jaeger?

Jaeger is an **open-source distributed tracing system**, originally built by Uber, and now part of the CNCF. It helps you:

* Monitor distributed transactions
* Understand application latency
* Perform root cause analysis
* Visualize request flow across services

In short, if your app is a mystery novel, Jaeger is Sherlock Holmes. 🕵️

## 🔗 What are Traces and Spans?

* A **trace** is a complete journey of a request through your app — from start to finish.
* A **span** is a single step in that journey, like one function call or one external API hit.

Think of a **trace** as the delivery of a pizza. Every **span** is a milestone in that process — order placed, pizza prepared, baked, out for delivery, delivered. Jaeger shows you the whole pizza journey.

## 📦 Jaeger Deployment in Kubernetes

Let’s deploy Jaeger in our Kubernetes cluster.

This YAML configuration sets up a single-instance Jaeger deployment in all-in-one mode within a Kubernetes cluster, suitable for development environments. The deployment uses the `jaegertracing/all-in-one` image and exposes key ports for telemetry (OTLP gRPC on 4317) and visualization (UI on 16686).

The associated ClusterIP service allows internal communication within the cluster, enabling the OpenTelemetry Collector to send trace data to Jaeger and providing access to the Jaeger UI via port forwarding for trace analysis and visualization.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  labels:
    app: jaeger
  
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:latest
        resources:
          requests:
            cpu: "10m"
            memory: "128Mi"
          limits:
            cpu: "20m"
            memory: "256Mi"        
        ports:
        - containerPort: 4317
        - containerPort: 6831
        - containerPort: 16686
        - containerPort: 14250

---

apiVersion: v1
kind: Service
metadata:
  name: jaeger
spec:
  selector:
    app: jaeger
  type: ClusterIP    
  ports:
  - name: ui
    port: 16686
    targetPort: 16686
  - name: grpc
    port: 4317
    targetPort: 4317
```

Save this configuration in a yaml file `jaeger.yaml` and deploy the jaeger using the following command:

```bash
kubectl -n observability apply -f jaeger.yaml
```

You can verify it's up using:

```bash
kubectl -n observability get all -l app=jaeger
```

![jaeger-get-all](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/jaeger-get-all.png)

The Jaeger UI will be available at the service's `ClusterIP`. Use `kubectl port-forward` to access the Jaeger UI locally:

```bash
kubectl -n observability port-forward svc/jaeger 16686:16686
```

Now open [http://localhost:16686](http://localhost:16686) in your browser.

![jaeger-ui](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/jaeger-ui.png)

## 🔁 Update the OpenTelemetry Collector Pipeline

Now that Jaeger is live, we need to **update the OpenTelemetry Collector config** to **export spans to Jaeger**.

```yaml
# otel-collector-agent-configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-agent-config
  namespace: observability
data:
  otel-collector-config.yaml: |

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

    service:
      pipelines:
        # collect trace data using otlp receiver and send it to jaeger
        traces:
          receivers: [otlp]
          processors: [memory_limiter, batch]
          exporters: [otlp/jaeger]

```

Apply the updated ConfigMap:

```bash
kubectl -n observability apply -f otel-collector-agent-configmap.yaml
```

Rollout the collector to pick up the new config:

```bash
kubectl -n observability rollout restart deployment otel-collector-agent
```

## 🧪 Test the Setup

Get the Endpoint IP from the K8s service:
```bash
API_ENDPOINT_IP=$(kubectl -n mlapp get svc -l app=house-price-service -o json | jq -r '.items[].spec.clusterIP')
```

Test it locally using `curl` or Postman:

```bash
curl -X POST "http://${API_ENDPOINT_IP}:80/predict/" \
  -H "Content-Type: application/json" \
  -d '{"features": [1200]}'
```

## 🎨 View Traces in Jaeger UI

Open Jaeger UI in your browser.

* Select the `house-price-service`
* Hit **Find Traces**
* Voilà! You can now **trace requests**, **view span timings**, and **debug latency** in style.

![jaeger-ui-traces](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/jaeger-ui-traces.png)

---

### 🔜 Coming Up Next: From Spans to Stats — Let’s Talk Metrics 📊

Now that Jaeger is live and humming—collecting traces and giving us deep insights into our application’s behavior—it's time to turn our attention to the second pillar of observability: **metrics**.

👉 Stay tuned as we wire up Prometheus and bring **metrics into the mix**, completing another piece of our observability blueprint.

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
