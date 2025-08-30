# OpenTelemetry in Action on Kubernetes: Part 3 - Deploying the Application on Kubernetes

## Deploying Our Instrumented ML App to Kubernetes

Welcome to Part 3! If you’ve followed along so far, by the end of [Part 2](./part-2_instrument_and_dockerize_mlapp_python_app.md) you had:

* A FastAPI-based machine learning app
* Instrumented with OpenTelemetry for full-stack observability
* Dockerized and ready to ship

Now, it's time to **bring in the big orchestration guns — Kubernetes**.

![OTel-k8s](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/OTel-k8s.png)

---

## Understanding Kubernetes Deployment & Service

Before we throw YAML at a cluster, let’s understand what these two crucial building blocks do:

### **Deployment**

A **Deployment** in Kubernetes manages a set of replicas (identical Pods running our app). It provides:

* **Declarative updates**: You describe *what* you want, K8s makes it so.
* **Rolling updates**: Smooth upgrades without downtime.
* **Self-healing**: If a Pod dies, K8s spins up a new one.

Think of it as a smart manager for your app's pods.

### **Service**

A **Service** exposes your app inside the cluster (or externally, if needed). It:

* Provides a stable DNS name.
* Load balances traffic between pods.
* In our case, exposes:

  * Port `80` → App port `8000` (FastAPI HTTP)
  * Port `4317` → OTLP gRPC (Telemetry)

---

## Kubernetes Manifest Breakdown

Let’s break down the configuration:

### Deployment: `house-price-service`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: house-price-service
```

We declare a Deployment that manages our app.

```yaml
spec:
  replicas: 2
```

We want **2 replicas** of our app running — high availability for the win.

```yaml
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
```

Kubernetes will update pods *gracefully*. It allows some extra pods during rollout and ensures some stay alive.

```yaml
      containers:
        - name: app
          image: house-price-predictor:v2
```

We use the Docker image built in Part 2, deployed as a container.

```yaml
          ports:
            - containerPort: 8000   # App port
            - containerPort: 4317   # OTLP telemetry port
```

Complete Deployment Manifest:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: house-price-service
  labels:
    app: house-price-service

spec:
  replicas: 2

  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%           # Allow 25% more pods than desired during update
      maxUnavailable: 25%     # Allow 25% of desired pods to be unavailable during update

  selector:
    matchLabels:
      app: house-price-service
  template:
    metadata:
      labels:
        app: house-price-service

    spec:
      containers:
        - name: app
          image: house-price-predictor:v2
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              cpu: "10m"
              memory: "128Mi"
            limits:
              cpu: "20m"
              memory: "256Mi"
          ports:
            - containerPort: 8000   # Application Port
            - containerPort: 4317   # OTLP gRPC Port
```

### Service: `house-price-service`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: house-price-service
  labels:
    app: house-price-service  
```

This **ClusterIP** Service lets other K8s workloads communicate with our app.

```yaml
  ports:
    - port: 80
      targetPort: 8000
    - port: 4317
      targetPort: 4317
```

The Service maps:

* Port `80` → App HTTP server
* Port `4317` → For OTLP spans, metrics, logs

Complete Service Manifest File:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: house-price-service
  labels:
    app: house-price-service  
spec:
  selector:
    app: house-price-service
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8000
    - name: otlp-grpc
      protocol: TCP
      port: 4317
      targetPort: 4317
  type: ClusterIP
```

Add both in the one file: `house-price-app.yaml`

## Deploying with `kubectl`

Before deploying the app, let's create a Kubernetes namespace. This helps group related resources together.

```bash
kubectl create namespace mlapp
```

Run the following to deploy your app:
```bash
kubectl -n mlapp apply -f house-price-app.yaml
```

To check the deployment status:

```bash
kubectl -n mlapp get deployments
kubectl -n mlapp get pods
```

![mlapp-k8s-get-deploy-pods](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/mlapp-k8s-get-deploy-pods.png)

To see pod logs (structured JSON + OpenTelemetry info):
```bash
kubectl -n mlapp logs -f -l app=house-price-service
```

To view the exposed service:
```bash
kubectl -n mlapp get svc -l app=house-price-service
```

## Testing the App in Kubernetes

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

You should get a prediction response like:

```json
{"predicted_price": 170000.0}
```

And voilà — telemetry data is flowing.

## What’s Next: Meet the OpenTelemetry Collector

In [Part 4](./part-4_otel-collector-ds-agent-in-k8s.md), we’ll introduce the **OpenTelemetry Collector Agent**:

* Deploy it as a DaemonSet alongside your app
* Configure it to collect traces, metrics, and logs
* Route the data to a gateway, and onward to backends like Prometheus, Jaeger, and Loki

> TL;DR: It’s where the real observability magic begins.

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```

