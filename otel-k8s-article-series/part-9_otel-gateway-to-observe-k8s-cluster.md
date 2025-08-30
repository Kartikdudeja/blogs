# OpenTelemetry in Action on Kubernetes: Part 9 - Cluster-Level Observability with OpenTelemetry Agent + Gateway

Welcome to the grand finale of our observability series! So far, we’ve added visibility into our **application** through **logs**, **metrics**, and **traces** — all flowing beautifully into **Grafana** via **OpenTelemetry Collector**.

But there’s still one big puzzle piece left: **the Kubernetes cluster itself**.

In this final part, we’ll:

- Collect host and node-level metrics using `hostmetrics`  
- Deploy a **centralized Collector in Deployment mode (gateway)**  
- Introduce `ServiceAccount` for permissions  
- Collect Kubernetes control plane metrics using `k8s_cluster`  
- Use the `debug` exporter to troubleshoot data pipelines  
- And finally, conclude the series with a high-level recap

![OTel-k8s](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/OTel-k8s.png)

---

## Why Cluster-Level Observability Matters

While we've focused on application telemetry so far, it's just one piece of the puzzle. For full visibility, we must also observe the **Kubernetes cluster** itself — the infrastructure running our apps.

Cluster observability helps us:

* Monitor **node health and resource usage**
* Track **control plane performance** (API server, scheduler, etc.)
* Understand **pod scheduling and evictions**
* Improve **scaling decisions**
* Troubleshoot **infrastructure-level issues**
* Strengthen **security and governance**

In short, without visibility into the cluster, you're flying blind. This part of the series ensures you're watching **not just the app, but the platform beneath it.**

## Add `hostmetrics` Receiver in the Agent

We’ll start by updating our **otel-collector-agent** (running as DaemonSet) to use the `hostmetrics` receiver. This receiver scrapes system-level metrics from each node, such as CPU, memory, disk, filesystem, and load.

**Config – `otel-collector-agent-configmap.yaml`**
```yaml
receivers:
  hostmetrics:
    collection_interval: 1m
    scrapers:
      cpu: {}
      memory: {}
      disk: {}
      load: {}
      filesystem: {}
      network: {}
      system: {}

processors:
  memory_limiter:
    check_interval: 1s
    limit_percentage: 80
    spike_limit_percentage: 15
  
  batch:
    send_batch_size: 1000
    timeout: 5s

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
    enable_open_metrics: true
    resource_to_telemetry_conversion:
      enabled: true

service:
  pipelines:
    # collect metrics from otlp and hostmetrics receiver and expose in prometheus compatible format
    metrics:
      receivers: [otlp, hostmetrics]
      processors: [memory_limiter, batch]
      exporters: [prometheus]
```

> Each `hostmetrics` receiver runs inside the agent pod on every node, giving us **node-specific insights**.

## Deploy the OpenTelemetry Gateway

### 1. Why Deployment Mode?
* **Deployment Mode** is used for centralized collection, aggregation, and export of telemetry data.
* Unlike the **DaemonSet agent**, which runs on each node, a **Deployment** collector can scrape and process cluster-wide metrics.

### 2. Create a ServiceAccount, ClusterRole, and ClusterRoleBinding

To use the `k8s_cluster` receiver, the collector must have permission to access Kubernetes objects like nodes, pods, namespaces, etc.

#### What is a ServiceAccount in Kubernetes?

A **ServiceAccount** in Kubernetes is an identity used by pods to authenticate and interact securely with the Kubernetes API. While every pod gets a default ServiceAccount, you often need to create custom ones with specific **RBAC (Role-Based Access Control)** permissions for security and least privilege.

In our case, the OpenTelemetry Collector needs to **read cluster state**—like nodes, pods, and namespaces—to collect metrics using the `k8s_cluster` receiver. So, we create a dedicated ServiceAccount and bind it to a **ClusterRole** with read-only access to those resources. This ensures our collector can operate properly without over-privileging it.

```yaml
# otel-collector-gateway-serviceaccount.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: otel-collector-gateway-sa
  namespace: observability
  labels:
    app: otel-collector-gateway  

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-gateway-role
  labels:
    app: otel-collector-gateway
rules:
- apiGroups:
  - ""
  resources:
  - events
  - namespaces
  - namespaces/status
  - nodes
  - nodes/spec
  - pods
  - pods/status
  - replicationcontrollers
  - replicationcontrollers/status
  - resourcequotas
  - services
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - daemonsets
  - deployments
  - replicasets
  - statefulsets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - replicasets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - jobs
  - cronjobs
  verbs:
  - get
  - list
  - watch
- apiGroups:
    - autoscaling
  resources:
    - horizontalpodautoscalers
  verbs:
    - get
    - list
    - watch

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-gateway-binding
  labels:
    app: otel-collector-gateway
subjects:
  - kind: ServiceAccount
    name: otel-collector-gateway-sa
    namespace: observability
roleRef:
  kind: ClusterRole
  name: otel-collector-gateway-role
  apiGroup: rbac.authorization.k8s.io

```

Apply it:

```bash
kubectl -n observability apply -f otel-collector-gateway-serviceaccount.yaml
```

### 3. OpenTelemetry Collector Config with `k8s_cluster` Receiver

Create the config file as a ConfigMap.

```yaml
# otel-collector-gateway-configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-gateway-config
  namespace: observability
  labels:
    app: otel-collector-gateway
data:
  otel-collector-config.yaml: |

    receivers:
      k8s_cluster:
        auth_type: "serviceAccount"
        collection_interval: 30s

    processors:
      memory_limiter:
        check_interval: 1s
        limit_percentage: 80
        spike_limit_percentage: 15
      
      batch:
        send_batch_size: 1000
        timeout: 5s

    exporters:
      debug:
        verbosity: detailed

      prometheus:
        endpoint: "0.0.0.0:8889"
        enable_open_metrics: true
        resource_to_telemetry_conversion:
          enabled: true

    service:
      pipelines:
        metrics:
          receivers: [k8s_cluster]
          processors: [memory_limiter, batch]
          exporters: [prometheus]
```

Apply it:

```bash
kubectl -n observability apply -f otel-collector-gateway-configmap.yaml
```

### 4. Deploy the OpenTelemetry Collector

```yaml
# otel-collector-gateway-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector-gateway
  namespace: observability
  labels:
    app: otel-collector-gateway  
spec:
  replicas: 1

  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%           # Allow 25% more pods than desired during update
      maxUnavailable: 25%     # Allow 25% of desired pods to be unavailable during update

  selector:
    matchLabels:
      app: otel-collector-gateway
  template:
    metadata:
      labels:
        app: otel-collector-gateway
    spec:
      serviceAccountName: otel-collector-gateway-sa
      containers:
        - name: otel-collector
          image: otel/opentelemetry-collector-contrib:latest
          args: ["--config=/conf/otel-collector-config.yaml"]
          volumeMounts:
            - name: config-volume
              mountPath: /conf
          resources:
            requests:
              cpu: 10m
              memory: 32Mi
            limits:
              cpu: 50m
              memory: 128Mi
      volumes:
        - name: config-volume
          configMap:
            name: otel-collector-gateway-config

```

Apply it:

```bash
kubectl -n observability apply -f otel-collector-gateway-deployment.yaml
```

### 5. Expose Collector to Prometheus

```yaml
# otel-collector-gateway-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: otel-collector-gateway
  namespace: observability
  labels:
    app: otel-collector-gateway
spec:
  selector:
    app: otel-collector-gateway
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

Apply:

```bash
kubectl -n observability apply -f otel-collector-gateway-service.yaml
```

Then add this to your Prometheus `scrape_configs`:

```yaml
- job_name: 'otel-collector-gateway'
  static_configs:
    - targets: ['otel-collector-gateway.observability.svc.cluster.local:8889']
```

### Test and Verify

Check deployment status:

```bash
kubectl -n observability get all -l app=otel-collector-gateway
```

![otel-collector-deployment-get-all](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/otel-collector-deployment-get-all.png)

## Special Mention: Debug Exporter - Your Observability Wingman

The **`debug` exporter** in OpenTelemetry Collector is a lightweight and incredibly helpful tool for developers and DevOps engineers when building or troubleshooting telemetry pipelines.

Instead of exporting telemetry data (like logs, metrics, and traces) to a backend system like Prometheus or Jaeger, the debug exporter simply **prints the data to the Collector's stdout**. This means:

- You can **see exactly what telemetry data is being received and processed**—live in the logs.
- It helps **validate instrumentation quickly**, without setting up full observability backends.
- It's especially useful when you're **testing new receivers, processors, or pipelines**, and want a quick look at the output.

### When to Use

- **Local testing or dev environments**.
- **Debugging broken data flow**—if Prometheus or Jaeger isn’t showing what you expect.
- **Learning how OpenTelemetry transforms and routes telemetry data.**

### Example Configuration Snippet

```yaml
exporters:
  debug:
    verbosity: detailed  # outputs full content of each signal
```

Then, reference it in your pipeline like this:

```yaml
service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger, debug]
```

This ensures traces are sent to Jaeger **and also printed** to the console—great for double-checking what's going in.

---

## Conclusion: You Now Have Full Observability!

Over the past 9 parts, you’ve:

* Containerized a real ML application  
* Instrumented it with OpenTelemetry  
* Collected traces, logs, and metrics  
* Deployed observability tools in Kubernetes  
* Visualized everything in Grafana  
* Monitored the entire Kubernetes cluster with Agent + Gateway mode

You’ve essentially built a **production-grade observability platform** from scratch — **without cloud vendor lock-in**.

---

### Missed the previous article?

Check out **[Part 8: Visualize Everything, Building a Unified Observability Dashboard with Grafana](./part-8_visualize_with_grafana_unfied_dashboard.md)** to see how we got here.

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
