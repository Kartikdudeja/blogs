# 🔭 OpenTelemetry in Action on Kubernetes: Part 8 - Visualize Everything, Building a Unified Observability Dashboard with Grafana

## 🔍 Why Visualization Matters

Telemetry data — logs, metrics, and traces — gives you deep insights into your system’s behavior. But let’s be honest: staring at JSON traces or YAML logs isn’t exactly thrilling.

**That’s where visualization comes in.**

A good dashboard:

* Gives instant visibility into system health
* Helps correlate metrics, logs, and traces
* Makes debugging, alerting, and capacity planning effortless

![OTel-k8s](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/OTel-k8s.png)

---

## 🧭 Meet Grafana: The Observatory for Observability

**Grafana** is an open-source analytics and visualization platform designed to work with various telemetry backends — including:

* **Prometheus** (for metrics)
* **Loki** (for logs)
* **Jaeger** (for traces)

Grafana is:

* Pluggable
* Real-time
* Customizable

It turns raw observability data into **actionable dashboards**.

## 🚀 Deploying Grafana in Kubernetes

We’ll deploy Grafana with a basic Deployment and Service. You can customize it with a persistent volume or admin credentials if needed.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:10.3.1
          resources:
            requests:
              cpu: "10m"
              memory: "56Mi"
            limits:
              cpu: "20m"
              memory: "128Mi"
          ports:
            - containerPort: 3000
          volumeMounts:
            - name: grafana-storage
              mountPath: /var/lib/grafana
          env:
            - name: GF_SECURITY_ADMIN_USER
              value: "admin"
            - name: GF_SECURITY_ADMIN_PASSWORD
              value: "admin"
      volumes:
        - name: grafana-storage
          emptyDir: {}  # Replace with PersistentVolumeClaim for persistence

---

apiVersion: v1
kind: Service
metadata:
  name: grafana
  labels:
    app: grafana
spec:
  selector:
    app: grafana
  ports:
    - protocol: TCP
      port: 3000
      targetPort: 3000
  type: ClusterIP
```

### 🧾 Deploy Grafana

```bash
# Apply deployment and service files
kubectl -n observability apply -f grafana.yaml

# Check Grafana pod logs
kubectl logs -l app=grafana -n observability

# Port-forward Grafana service to access UI locally
kubectl -n observability port-forward svc/grafana 3000:3000
```

Now visit [http://localhost:3000](http://localhost:3000) in your browser.
**Default credentials**:

* Username: `admin`
* Password: `admin`

## 🔌 Configure Datasources in Grafana

Once inside the Grafana UI, follow these steps to add your observability backends:

### 1️⃣ Add **Prometheus** as a Datasource:

* Go to **Home** →  **Connections** →  **Data sources**
* Click **Add new data source**
* Choose **Prometheus**
* Set URL to:

  ``` text
  http://prometheus.observability.svc.cluster.local:9090
  ```
* Click **Save & Test**

### 2️⃣ Add **Loki** as a Datasource:

* Repeat above steps, choose **Loki**
* Set URL to:

  ``` text
  http://loki.observability.svc.cluster.local:3100
  ```
* Save & Test

### 3️⃣ Add **Jaeger** as a Datasource:

* Choose **Jaeger** from the list
* Set URL to:

  ``` text
  http://jaeger.observability.svc.cluster.local:16686
  ```
* Save & Test

![grafana-datasources-list](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/grafana-datasources-list.png)

## 🔍 Explore Logs, Metrics, and Traces

Head over to the **Explore** tab in Grafana:

* Select **Loki** → Run a log query like `{exporter="OTLP"} |= `house-price-service``

![grafana-explore-loki](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/grafana-explore-loki.png)

* Select **Jaeger** → Search traces for your app, filtered by service name
* Select **Prometheus** → Query custom app metrics

> 🔥 This is your real-time debugging playground.

## 🧱 Build a Unified Dashboard

Now let’s pull it all together.

### 🔧 Steps to Create a Dashboard:

1. Go to the **Dashboards** section → Click **New Dashboard**

2. Add a **Panel**:

   * **For Metrics**: Use Prometheus queries (e.g., request rate, latency)
   * **For Logs**: Use Loki query (e.g., by app label)
   * **For Traces**: Use Jaeger panel or link to trace visualizer

3. Organize the panels side-by-side:

   * App throughput (metric)
   * App logs (filtered view)
   * Recent traces

4. Save the dashboard and give it a name like `House Price App Observability`

![grafana-dashboard](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/grafana-dashboard.png)

## 🎯 Conclusion

You now have a complete, **three-pillar observability stack** running on Kubernetes:

* 📈 Metrics via **Prometheus**
* 🪵 Logs via **Loki**
* 🌐 Traces via **Jaeger**
* 🧭 Visualized in **Grafana**

All powered by OpenTelemetry — the glue connecting them.

---

## ⏭️ What’s Next?

You now have full visibility into your application — but what about the **Kubernetes cluster itself**?

In the **final part** of the series, we’ll expand our observability beyond the app and dive into **cluster-level insights**. This includes monitoring:

* Node and pod CPU/memory usage
* Kubernetes control plane metrics
* Scheduler performance, kubelet stats, and more

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
