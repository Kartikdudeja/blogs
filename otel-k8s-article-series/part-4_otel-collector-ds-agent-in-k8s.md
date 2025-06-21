# 🔭 OpenTelemetry in Action on Kubernetes: Part 4 - Deploying OpenTelemetry Collector (Agent Mode) in Kubernetes

Welcome back, observability artisans! So far in our series:

* We trained a simple ML model and wrapped it in a FastAPI app (Part 1).
* We instrumented it with OpenTelemetry to emit traces, metrics, and logs (Part 2).
* We dockerized and deployed the app in Kubernetes (Part 3).

Now it’s time to **build the telemetry pipeline** by deploying the **OpenTelemetry Collector** in **agent mode**. Think of it as your app’s personal observability sidekick — sitting beside your pod, collecting and forwarding telemetry like a seasoned ops ninja.

![OTel-k8s](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/OTel-k8s.png)

---

## 🔧 What Is the OpenTelemetry Collector?

The **OpenTelemetry Collector** is a vendor-agnostic service that can receive, process, and export telemetry data (metrics, logs, and traces). It acts like a modular observability router.

In **agent mode**, it's typically deployed as a **DaemonSet**, meaning one collector pod runs on each node — perfect for scraping local app telemetry.

## 🔍 The Collector Pipeline – A Three-Stage Flow

The pipeline is made up of:

### 1. **Receivers**

These are the collectors' "ears." They listen for telemetry data from your app.
**Example:** OTLP receiver listens on port `4317` for gRPC telemetry.

📦 *Analogy:* Like a parcel dropbox at the post office — it accepts incoming packages (telemetry).

### 2. **Processors**

Processors act like post-office sorters — they batch, sample, or modify telemetry before export.
**Example:** Batching to reduce load or adding attributes to spans.

📦 *Analogy:* Sorting parcels by zip code before shipping.

### 3. **Exporters**

Exporters are your delivery trucks. They ship telemetry off to destinations like Prometheus, Jaeger, or Loki.

📦 *Analogy:* The final delivery van that takes your parcel to your house.

## 🧾 Configuration in Kubernetes: The ConfigMap

We store our OpenTelemetry pipeline config in a Kubernetes **ConfigMap** — a way to inject config data into pods as files or environment variables.

## 🔩 Step-by-Step: Deploying Otel Collector (Agent)

We'll deploy three components:

### 🔸 1. ConfigMap (Collector Pipeline)

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

    processors:      
      batch:

    exporters:
      debug:
        verbosity: detailed

    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [debug]
```

> 🧠 *This simple pipeline receives traces using OTLP (gRPC), batches them, and prints them to stdout using a debug exporter. We'll replace this with Jaeger later in Part 5.*


#### 🧩 Deep Dive: Key Components of the OpenTelemetry Pipeline


![otel-collector-agent-pipeline-demo](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/otel-collector-agent-pipeline-demo.png)

##### 🎧 Receiver: `otlp` with `grpc` Protocol
```yaml
receivers:
  otlp:
    protocols:
      grpc:
```

🔍 What it does:

The **receiver** is the entry point into the Collector. In this case, we’re telling the collector to accept data over the **OTLP (OpenTelemetry Protocol)** using the **gRPC** transport.

* **OTLP** is the default protocol for OpenTelemetry.
* **gRPC** is a high-performance, open-source RPC framework — it’s fast, efficient, and used widely in modern telemetry systems.

#### ⚙️ Processor: `batch`

```yaml
processors:
  batch:
```

🔍 What it does:

Processors manipulate or enhance telemetry **after it's received but before it's exported**.

The `batch` processor is highly recommended in most pipelines. It collects telemetry data in small batches and sends them together instead of one at a time. This improves performance and reduces resource usage.

🚀 Benefits:

* Reduces the number of outgoing requests.
* Improves throughput by sending larger payloads.
* Helps smooth out traffic spikes.

#### 🪵 Exporter: `debug`
```yaml
exporters:
  debug:
    verbosity: detailed
```

🔍 What it does:

Exporters are responsible for sending telemetry to an external backend (e.g., Jaeger, Prometheus, Datadog).

In this case, we’re using the **debug exporter** — which doesn’t send data to an external system but **prints it to stdout**.

* `verbosity: detailed` means it will output detailed telemetry, including span names, attributes, and events.
* This is great for **local testing or debugging**, but not suitable for production.

### 🔸 2. Deployment (DaemonSet - Agent Mode)

```yaml
# otel-collector-agent-daemonset.yaml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector-agent
  namespace: observability
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # One pod at a time will be unavailable during update
  selector:
    matchLabels:
      app: otel-collector-agent
  template:
    metadata:
      labels:
        app: otel-collector-agent
    spec:
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:latest
          args: ["--config=/conf/otel-collector-config.yaml"]
          resources:
            requests:
              cpu: 10m
              memory: 32Mi
            limits:
              cpu: 50m
              memory: 128Mi          
          volumeMounts:
            - name: config-volume
              mountPath: /conf
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: config-volume
          configMap:
            name: otel-collector-agent-config
        - name: varlog
          hostPath:
            path: /var/log
```

### 🔸 3. Service (Internal Communication)

```yaml
# otel-collector-agent-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: otel-collector-agent
  labels:
    app: otel-collector-agent
spec:
  selector:
    app: otel-collector-agent
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
      protocol: TCP
    - name: otlp-http
      port: 4318
      targetPort: 4318
      protocol: TCP
    - name: prometheus
      port: 8889
      targetPort: 8889
      protocol: TCP    
  type: ClusterIP
```

## 🚀 Deploying with `kubectl`

```bash
# create a new namespace
kubectl create namespace observability
```

Deploy the Collector:
```bash
kubectl -n observability apply -f otel-collector-agent-configmap.yaml -f otel-collector-agent-service.yaml -f otel-collector-agent-daemonset.yaml
```

![otel-collector-agent-apply](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/otel-collector-agent-apply.png)

To check the status:
```bash
kubectl -n observability get all -l app=otel-collector-agent
```

![otel-collector-agent-get-all](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/otel-collector-agent-get-all.png)

## 📡 Updating the App Deployment

We now need to **add the OTLP endpoint to the app** as an environment variable. 

When you create a Kubernetes **Service**, it gets a DNS name like this:

```
<service-name>.<namespace>.svc.cluster.local
```

So our service name `otel-collector-agent` in namespace `observability` is reachable at:

```
otel-collector-agent.observability.svc.cluster.local:4317
```

Magic ✨, courtesy of Kubernetes DNS.

Adding the `OTEL_EXPORTER_OTLP_ENDPOINT` environment variable in your application deployment tells the OpenTelemetry SDK where to send telemetry data (traces, metrics, and logs). This line effectively connects your instrumented app to the OpenTelemetry Collector, acting as the central receiver and router for all observability signals within the Kubernetes environment.

```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector-agent.observability.svc.cluster.local:4317"
  - name: OTEL_EXPORTER_OTLP_INSECURE
    value: "true"
```

You’ll insert this under the container spec in your **house-price-app.yaml**.

After adding the above config, we will have to apply new changes to Application Deployment:

```bash
kubectl -n mlapp apply -f house-price-app.yaml
```

You can check the deployment rollout status with the following command:
```bash
kubectl -n mlapp rollout status deployment house-price-service
```

![app-deployment-rollout](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/app-deployment-rollout.png)

---

## 👀 What’s Next?

In **Part 5**, we’ll deploy **Jaeger**, a UI-based distributed tracing tool, and rewire our OTEL pipeline to send trace data there. You’ll get to *see* spans, visualize your API behavior, and debug latency like a real tracing wizard. 🧙‍♂️

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
