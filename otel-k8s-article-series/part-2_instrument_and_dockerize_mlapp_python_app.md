# OpenTelemetry in Action on Kubernetes: Part 2 - Instrument And Dockerizing

What’s Happening in Part 2?

Welcome back, observability explorers! In [Part 1](./part-1_building-mlapp-in-python-with-fastapi.md), we built a basic FastAPI app to predict house prices with the help of linear regression. But let’s be honest — a naked app in production is like flying blind with no cockpit instruments.

So in this chapter of our telemetry tale, we’re going to:

* Add **OpenTelemetry instrumentation** for **traces**, **metrics**, and **logs**
* Generate **custom metrics** for traffic and latency
* Log everything in beautiful **JSON format**
* Dockerize our instrumented app
* Run and observe logs in action

By the end, your app will be **telemetry-ready**, trace-emitting, log-spraying, and metric-capturing — just like a real production-grade microservice.


![otel-k8s](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/OTel-k8s.png)

---

### Let’s Talk Observability Instrumentation

Here’s what we added to our Python FastAPI app:

#### Tracing

* We're using OpenTelemetry’s SDK to generate spans for each endpoint.
* Each request to `/` or `/predict` is wrapped in a **tracer span**.
* We tag the `/predict` span with the **input features** as metadata.

#### Metrics

* We define two custom metrics:

  * `api_requests_total`: A counter to track total API hits
  * `api_latency_seconds`: A histogram to record request duration
* Each metric includes useful **labels** like `endpoint` and `method`.

#### Logging

* Logs are now output in **structured JSON**, perfect for parsing.
* We log helpful events like:

  * Health check hits
  * Predictions made

Here’s a breakdown of the key code snippets:

### Tracing Setup

```python
trace.set_tracer_provider(TracerProvider(resource=resource))
tracer = trace.get_tracer(__name__)
span_exporter = OTLPSpanExporter()
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(span_exporter))
```

> OTLP = OpenTelemetry Protocol. We're exporting spans using OTLP over gRPC — which will be picked up by the OpenTelemetry Collector later in the series.

### Metrics Setup

```python
metrics.set_meter_provider(MeterProvider(resource=resource, metric_readers=[metric_reader]))
api_counter = meter.create_counter("api_requests_total", ...)
api_latency = meter.create_histogram("api_latency_seconds", ...)
```

> Custom metrics give you way more flexibility than auto-generated ones — and they're fun to build!

### Logging in JSON

```python
class JsonFormatter(logging.Formatter):
    def format(self, record):
        ...
```

> We format every log entry into JSON — making them machine-parseable and human-readable (if your brain speaks JSON).

### Application Routes

We wrap each route in a span, log the action, increment counters, and record latency:

```python
@app.get("/")
def read_root(request: Request):
    with tracer.start_as_current_span("GET /", kind=SpanKind.SERVER):
        logger.info("Health check hit")
        api_counter.add(1, {"endpoint": "/", "method": "GET"})
        api_latency.record(...)
```

### Full Python Code — Instrumented with Tracing, Metrics, and Logs

Here's the Complete Code After Instrumentation:
```python
from fastapi import FastAPI, Request
from pydantic import BaseModel
import pickle
import numpy as np
import logging
import json
import time

from opentelemetry import trace, metrics
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.logging import LoggingInstrumentor
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.trace import SpanKind


# --------------------------
# JSON Logging Setup
# --------------------------
class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            "timestamp": self.formatTime(record),
            "level": record.levelname,
            "name": record.name,
            "message": record.getMessage(),
        }
        return json.dumps(log_entry)

logger = logging.getLogger("house-price-service")
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# --------------------------
# OpenTelemetry Tracing
# --------------------------
resource = Resource(attributes={"service.name": "house-price-service"})

# Tracer setup
trace.set_tracer_provider(TracerProvider(resource=resource))
tracer = trace.get_tracer(__name__)

span_exporter = OTLPSpanExporter()
trace.get_tracer_provider().add_span_processor(BatchSpanProcessor(span_exporter))

# Metrics setup
metric_reader = PeriodicExportingMetricReader(OTLPMetricExporter())
metrics.set_meter_provider(MeterProvider(resource=resource, metric_readers=[metric_reader]))
meter = metrics.get_meter(__name__)

# Metrics
api_counter = meter.create_counter(
    name="api_requests_total",
    unit="1",
    description="Total number of API requests",
)

api_latency = meter.create_histogram(
    name="api_latency_seconds",
    unit="s",
    description="API response latency in seconds",
)

# --------------------------
# FastAPI App Setup
# --------------------------
app = FastAPI()
FastAPIInstrumentor().instrument_app(app)
LoggingInstrumentor().instrument(set_logging_format=True)

# Load the model
with open("house_price_model.pkl", "rb") as f:
    model = pickle.load(f)

@app.get("/")
def read_root(request: Request):
    start_time = time.time()
    with tracer.start_as_current_span("GET /", kind=SpanKind.SERVER):
        logger.info("Health check hit")
        api_counter.add(1, {"endpoint": "/", "method": "GET"})
        api_latency.record(time.time() - start_time, {"endpoint": "/", "method": "GET"})
        return {"message": "House Price Prediction API is live!"}

class HouseFeatures(BaseModel):
    features: list[float]

@app.post("/predict/")
def predict(data: HouseFeatures, request: Request):
    start_time = time.time()
    with tracer.start_as_current_span("POST /predict", kind=SpanKind.SERVER) as span:
        span.set_attribute("input.features", str(data.features))
        api_counter.add(1, {"endpoint": "/predict", "method": "POST"})

        prediction = model.predict(np.array(data.features).reshape(1, -1))
        logger.info(f"Prediction made: {prediction[0]}")

        api_latency.record(time.time() - start_time, {"endpoint": "/predict", "method": "POST"})
        return {"predicted_price": prediction[0]}
```

Updated `requirements.txt`:
```text
annotated-types==0.7.0
anyio==4.9.0
asgiref==3.8.1
certifi==2025.4.26
charset-normalizer==3.4.2
click==8.2.1
Deprecated==1.2.18
exceptiongroup==1.3.0
fastapi==0.110.0
googleapis-common-protos==1.70.0
grpcio==1.71.0
h11==0.16.0
httptools==0.6.4
idna==3.10
importlib_metadata==7.1.0
joblib==1.5.1
numpy==1.26.4
opentelemetry-api==1.25.0
opentelemetry-exporter-otlp==1.25.0
opentelemetry-exporter-otlp-proto-common==1.25.0
opentelemetry-exporter-otlp-proto-grpc==1.25.0
opentelemetry-exporter-otlp-proto-http==1.25.0
opentelemetry-instrumentation==0.46b0
opentelemetry-instrumentation-asgi==0.46b0
opentelemetry-instrumentation-fastapi==0.46b0
opentelemetry-instrumentation-logging==0.46b0
opentelemetry-proto==1.25.0
opentelemetry-sdk==1.25.0
opentelemetry-semantic-conventions==0.46b0
opentelemetry-util-http==0.46b0
protobuf==4.25.7
pydantic==1.10.14
pydantic_core==2.33.2
python-dotenv==1.1.0
PyYAML==6.0.2
requests==2.32.3
scikit-learn==1.4.2
scipy==1.15.3
sniffio==1.3.1
starlette==0.36.3
threadpoolctl==3.6.0
typing-inspection==0.4.1
typing_extensions==4.13.2
urllib3==2.4.0
uvicorn==0.29.0
uvloop==0.21.0
watchfiles==1.0.5
websockets==15.0.1
wrapt==1.17.2
zipp==3.21.0
```

## Dockerizing the App (Now with Telemetry Magic!)

Now that the app can observe itself, it’s time to containerize it.

### Dockerfile Summary

```Dockerfile
# lightweight base image
FROM python:3.10.12-slim

# metadata
LABEL app="ml-prediction-model"
LABEL env="dev"
LABEL lab="observability"

# create a non-root user and group
RUN adduser --disabled-password --gecos '' appuser

# set working directory for application
WORKDIR /app

# copy list of required dependencies
COPY requirements.txt .

# install dependencies as root
RUN pip install --no-cache-dir -r requirements.txt

# cleanup dockerfile
RUN rm -rf /tmp/*

# add application code and ml model
COPY app.py .
COPY house_price_model.pkl .

# Change ownership to the non-root user
RUN chown -R appuser:appuser /app

# Switch to the non-root user
USER appuser

# export port for application listener and otlp
EXPOSE 8000
EXPOSE 4317

# start application
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

Let’s take a look under the hood of our Dockerfile. This container is built with observability and security in mind — no root shenanigans here.

#### Base Image

```dockerfile
FROM python:3.10.12-slim
```

We start with a **slim and clean Python 3.10** base image — small footprint, fewer vulnerabilities, and fast builds. It’s perfect for production-ready containers.

#### Metadata Labels

```dockerfile
LABEL app="ml-prediction-model"
LABEL env="dev"
LABEL lab="observability"
```

These labels help organize and identify the image in registries or orchestrators like Kubernetes. Think of it as giving your container a business card.

#### Security First: Create a Non-Root User

```dockerfile
RUN adduser --disabled-password --gecos '' appuser
```

Instead of running your app as root (a big no-no in production), we create a minimal non-root user called `appuser`. This is a best practice for container security.

#### Set the Working Directory

```dockerfile
WORKDIR /app
```

This sets `/app` as the working directory for all subsequent commands — keeping things tidy and predictable.

#### Install Python Dependencies

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

We copy the dependency list into the image and install everything in one go using `pip`. The `--no-cache-dir` keeps the image size lean by avoiding pip's cache bloat.

#### Clean Up Temporary Files

```dockerfile
RUN rm -rf /tmp/*
```

Another space-saving move — we wipe `/tmp` to keep the image leaner and meaner.

#### Add the App and Model

```dockerfile
COPY app.py .
COPY house_price_model.pkl .
```

We copy our **instrumented FastAPI app** and the **trained ML model** into the image.

#### Set Ownership and Use Non-Root User

```dockerfile
RUN chown -R appuser:appuser /app
USER appuser
```

We change ownership of the `/app` directory to `appuser` and switch the active user. This enforces that your app runs without elevated privileges. Good security hygiene!

#### Expose Ports

```dockerfile
EXPOSE 8000
EXPOSE 4317
```

* `8000` is the app port served by **Uvicorn**.
* `4317` is the default **OTLP gRPC port** used by OpenTelemetry exporters to communicate with the **OpenTelemetry Collector**.

#### Start the FastAPI App

```dockerfile
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

The default command starts your FastAPI app using **Uvicorn**, binding it to all interfaces so it’s accessible from outside the container.

### Summary

This Dockerfile is:

* **Minimal** (slim base image)
* **Secure** (non-root user)
* **Telemetry-ready** (OTLP and app ports exposed)
* **Production-friendly** (clean and explicit)

---

## Build & Run the Docker Container

Run these commands:

```bash
docker build -t house-price-predictor:v2 .
docker run -d -p 8000:8000 --name house-price-predictor house-price-predictor:v2
```
![ml-app-start-logs](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/ml-app-start-logs.png)

### **Try It Out**
```bash
curl -i -X GET 'http://127.0.0.1:8000/'
```

![ml-app-health-endpoint-curl](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/ml-app-predict-endpoint-curl.png)

![ml-app-health-endpoint-logs](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/ml-app-health-endpoint-logs.png)

POST to `/predict/` with:
```bash
curl -i -X POST 'http://127.0.0.1:8000/predict/' -H "Content-Type: application/json" -d '{"features": [1500]}'
```

![ml-app-predict-endpoint-curl](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/ml-app-predict-endpoint-curl.png)

![ml-app-predict-endpoint-logs](https://github.com/Kartikdudeja/blogs/blob/main/images/otel-k8s/ml-app-predict-endpoint-logs.png)

---

## What We've Achieved

**In this part**, we’ve:

* Instrumented the app with OpenTelemetry SDK
* Added spans, logs, and custom metrics
* Dockerized and tested the telemetry-powered service

---
### What’s Next: Let’s Get This Thing on Kubernetes

Now that our app is fully instrumented — logging in structured JSON, generating traces and spans, emitting custom metrics — and Dockerized for portability, it’s time to take the next big leap:

> **Deploying the app to a Kubernetes cluster.**

In [Part 3](./part-3_deploying-app-in-k8s.md), we’ll:

* Create Kubernetes manifest files for deploying our FastAPI + ML app.
* Expose the app so we can access it and start generating telemetry from a real-world setup.

Oh, and yes — our **OTLP port (4317)** is coming along for the ride, ready to chat with the OpenTelemetry Collector once it’s up.

> Grab your `kubectl`, fire up your cluster, and get ready — it’s time to go full cloud-native.

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
