# 🔭 OpenTelemetry in Action on Kubernetes: Part 1 - Building a Simple ML App with FastAPI

🚀 **Introduction to the Series**

Welcome to _**“The Observability Blueprint: Instrument, Deploy, Observe — OpenTelemetry in Action on Kubernetes”**_ – where we take a humble machine learning application and give it superpowers: observability across metrics, logs, and traces using open-source tools like OpenTelemetry, Prometheus, Jaeger, Loki, and Grafana.

In this blog series, we’ll:
- 🤖 Build a simple ML app using FastAPI
- 📦 Instrument it using OpenTelemetry
- 🐳 Containerize it with Docker
- ☸️ Deploy it to Kubernetes
- 🔭 Collect and visualize observability data like a pro

🎯 Goal: show how to fully observe a production-ish ML app from every angle using only open-source tools.

![OTel-k8s](https://github.com/Kartikdudeja/blogs/blob/main/images/OTel-k8s.png)

---
## Part 1: Building a Simple ML App with FastAPI

🧰 _Prerequisites:_
Before we start, make sure you have:
- Python 3.8+
- `pip`
- Basic familiarity with Python and REST APIs (we promise to keep it friendly)

🔧 **Step 0: Set Up a Virtual Environment**
Let’s avoid polluting your global Python setup. Run the following:
```bash
python3 -m venv venv
source venv/bin/activate    # On Windows: venv\Scripts\activate
pip3 install --upgrade pip
```

Create a `requirements.txt`:
```text
fastapi==0.110.0
pydantic==1.10.14
uvicorn==0.29.0
scikit-learn==1.4.2
numpy==1.26.4
```

Then Install:
```bash
pip3 install -r requirements.txt
```

🧠 **Step 1: What’s Machine Learning? And Why Linear Regression?**
**Machine Learning** is the art (and science) of teaching computers to learn patterns from data instead of writing rule-based code.

In this part, we’ll use **Linear Regression**, the "hello world" of ML models. It assumes there’s a straight-line relationship between input and output — in our case, the bigger the house, the higher the price. (Groundbreaking stuff, we know.)

![linear-regression](https://github.com/Kartikdudeja/blogs/blob/main/images/linear-regression.png)

Here’s the model training code:
```python
# train_model.py

import numpy as np
import pickle
from sklearn.linear_model import LinearRegression

# Sample dataset: house sizes in sq ft and corresponding prices in dollars
# Data: [size (sq ft)], [price ($)]
X = np.array([[500], [1000], [1500], [2000], [2500]])
y = np.array([100000, 150000, 200000, 250000, 300000])

# Train the model
model = LinearRegression()
model.fit(X, y)

# Save the model to a file
with open("house_price_model.pkl", "wb") as f:
    pickle.dump(model, f)

print("Model trained and saved as 'house_price_model.pkl'")
```
This generates a `.pkl` file — a serialized version of our model we’ll later load inside an Application.

⚡ **Step 2: Serve It via FastAPI**
Let’s make it accessible to the outside world.

📦 **Why FastAPI?**

FastAPI is a modern Python web framework built for speed (thanks to Starlette & Pydantic), great dev experience, and automatic documentation. It’s a joy to use — kind of like Flask, but with type hints and Swagger included.

Let’s write our app:
```python
# app.py

from fastapi import FastAPI, Request
from pydantic import BaseModel
import pickle
import numpy as np

# --------------------------
# FastAPI App Setup
# --------------------------
app = FastAPI()

# Load the model
with open("house_price_model.pkl", "rb") as f:
    model = pickle.load(f)

# --------------------------
# Healthcheck endpoint
# --------------------------
@app.get("/")
def read_root(request: Request):
        return {"message": "House Price Prediction API is live!"}

# --------------------------
# Define the request schema
# --------------------------
class HouseFeatures(BaseModel):
    features: list[float]

# --------------------------
# Prediction endpoint
# --------------------------
@app.post("/predict/")
def predict(data: HouseFeatures, request: Request):
        prediction = model.predict(np.array(data.features).reshape(1, -1))
        logger.info(f"Prediction made: {prediction[0]}")
        return {"predicted_price": prediction[0]}
```

▶️ **Run it with:**
```bash
uvicorn app:app --reload
```

🧪 **Try It Out**
```bash
curl -i 'http://127.0.0.1:8000/'
```

POST to `/predict/` with:
```bash
curl -i -X POST 'http://127.0.0.1:8000/predict/' -H "Content-Type: application/json" -d '{"features": [1500]}'
```
💡 Output:
```json
{
  "predicted_price": 200000.0
}
```
_Congratulations, you just served your first ML model! 🎉_

---

✅ What’s Next?

In Part 2, we’ll dockerize the app so it's ready for deployment in the wild. We’ll also start thinking about instrumenting the app — because what’s an API without observability metrics to brag about?

---

```json
{
    "author"   :  "Kartik Dudeja",
    "email"    :  "kartikdudeja21@gmail.com",
    "linkedin" :  "https://linkedin.com/in/kartik-dudeja",
    "github"   :  "https://github.com/Kartikdudeja"
}
```
