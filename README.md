# ML-Order-Management-System

## Project Overview
The ML Order Management System is an intelligent web application that leverages machine learning to streamline order processing and management. This project combines modern web technologies with predictive analytics to provide a seamless ordering experience for customers and efficient order tracking for business owners.
Who is it for?

* Customers: Individuals looking for a user-friendly order creation and management platform
* Business Owners: Managers and administrators needing a robust system to track and manage orders
* Tech Enthusiasts: Developers interested in a full-stack machine learning application implementation

I'll create a comprehensive GitHub README artifact for this machine learning project template.

Let me elaborate on the machine learning model's purpose in this system, which I've highlighted in the README.

# The machine learning model serves three primary strategic purposes:

1. *Predictive Order Processing
   - By analyzing historical order data, the model can predict potential order cancellations before they occur
   - It can optimize order routing by understanding patterns in order processing times and customer preferences
   - The model might learn to anticipate bottlenecks or delays in the order lifecycle

2. *Customer Behavior Insights
   - The ML model can segment customers based on their ordering patterns
   - It can generate personalized recommendations for products or services
   - By understanding individual customer behaviors, it can predict future ordering trends

3. *Operational Efficiency
   - Forecasting order volumes helps in resource allocation
   - Detecting anomalies in order processing can prevent potential issues
   - The model can continuously learn and improve its predictive capabilities

The machine learning component adds an intelligent layer to what would otherwise be a standard order management system. Instead of just processing orders, the system now has predictive and adaptive capabilities that can provide competitive advantages like:
- Reduced order cancellations
- Improved customer satisfaction
- More efficient resource management
- Data-driven business insights



# Solution Design

![Solution Design](solution-design.svg)











## The solution is designed with a layered approach:

* Presentation Layer: React-based web interface
* Authentication Layer: Auth0 for secure user management
* Business Logic Layer: Node.js backend with machine learning integration
* Data Layer: NoSQL database for flexible data storage
* Machine Learning Layer: Predictive models for order optimization

## High-Level Requirements
## Customer Use Cases

* As a customer, I need to be able to create a new order
* As a customer, I need to be able to update an existing order
* As a customer, I need to be able to cancel a new order

## Owner Use Cases

* As an owner, I am able to view pending orders
* As an owner, I can manage and track order statuses

# Features Implemented

 * Customer authentication using Auth0
 * Order creation functionality
 * Order update capabilities
 * Order cancellation process
 * Owner dashboard for order management
 * Integrated unit and integration tests

# Assumptions

### Customer will primarily use the web application
### Initial version does not require complex reporting
### Basic machine learning predictions will be implemented
### Scalability is a future consideration

# Future Enhancements

### Implement email-based login
### Advanced machine learning order prediction
### Comprehensive reporting dashboard
### Mobile application support
### Advanced analytics and insights

# Challenges

### Version incompatibility between libraries
### Integrating machine learning models with web application
### Ensuring smooth user authentication
### Maintaining performance with increasing order volume

# Design Decisions
NoSQL vs Relational Database

## Why NoSQL was Preferred:

* Flexible schema for evolving order structures
* Horizontal scalability
* Better performance for read-heavy workloads
* Easier integration with machine learning workflows
* Simplified data model for order management

# Installation
Prerequisites

### Node.js (v16+)
### npm (v8+)
### MongoDB
### Python (v3.8+)


# System Design 

![System Design](system-design.png)


# Detailed Architecture

![detailed architecture](detailed-architecture.png)










# Steps


This project is a machine learning order management system that integrates various technologies:
- **FastAPI** for the backend API
- **Scikit-learn** for model training
- **MongoDB & Redis** for database and caching
- **RabbitMQ** for message queue processing
- **Auth0** for authentication
- **Prometheus** for monitoring
- **Docker** for containerization
- **React.js with Redux** for the frontend

---

## Backend Implementation

### Step 1: Train & Save the ML Model
```python
import pickle
import pandas as pd
from sklearn.linear_model import LinearRegression

def train_model():
    data = pd.DataFrame({
        'feature1': [1, 2, 3, 4, 5],
        'feature2': [10, 20, 30, 40, 50],
        'orders': [100, 200, 300, 400, 500]
    })
    model = LinearRegression()
    model.fit(data[['feature1', 'feature2']], data['orders'])
    with open("order_prediction_model.pkl", "wb") as f:
        pickle.dump(model, f)
train_model()
```

### Step 2: Load the Trained Model
```python
with open("order_prediction_model.pkl", "rb") as f:
    model = pickle.load(f)
```

### Step 3: FastAPI Server
```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import OAuth2PasswordBearer
from pydantic import BaseModel
import uvicorn
import requests
from prometheus_client import Counter, generate_latest, REGISTRY, start_http_server

app = FastAPI()
REQUEST_COUNT = Counter("request_count", "Total API Requests", ["endpoint"])
start_http_server(8001)

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")
AUTH0_DOMAIN = "your-auth0-domain"
API_AUDIENCE = "your-api-audience"

def verify_token(token: str = Depends(oauth2_scheme)):
    url = f"https://{AUTH0_DOMAIN}/userinfo"
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        raise HTTPException(status_code=401, detail="Invalid authentication credentials")
    return response.json()

class OrderFeatures(BaseModel):
    feature1: float
    feature2: float

@app.post("/predict/")
def predict_order(features: OrderFeatures, user: dict = Depends(verify_token)):
    REQUEST_COUNT.labels(endpoint="predict").inc()
    df = pd.DataFrame([features.dict()])
    prediction = model.predict(df)
    return {"user": user, "prediction": prediction.tolist()}

@app.get("/metrics")
def get_metrics():
    return generate_latest(REGISTRY)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Message Queue Processing with RabbitMQ
```python
import redis
import json
import pika
from pymongo import MongoClient
import threading

mongo_client = MongoClient("mongodb://mongodb:27017/")
db = mongo_client["order_db"]
orders_collection = db["orders"]

redis_client = redis.Redis(host='redis', port=6379, db=0)

def rabbitmq_callback(ch, method, properties, body):
    order_data = json.loads(body)
    print("Received order data:", order_data)
    orders_collection.insert_one(order_data)
    redis_client.set(f"order:{order_data['order_id']}", json.dumps(order_data))
    print("Order data saved to MongoDB and cached in Redis")

connection = pika.BlockingConnection(pika.ConnectionParameters('rabbitmq'))
channel = connection.channel()
channel.queue_declare(queue='order_queue')
channel.basic_consume(queue='order_queue', on_message_callback=rabbitmq_callback, auto_ack=True)

def consume_rabbitmq():
    print("Starting RabbitMQ consumer...")
    channel.start_consuming()
threading.Thread(target=consume_rabbitmq, daemon=True).start()
```

---

## Docker Setup
### Backend Dockerfile
```Dockerfile
FROM python:3.9
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "ml_order_management.py"]
```

### Frontend Dockerfile
```Dockerfile
FROM node:14
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json ./
RUN npm install
COPY frontend .
CMD ["npm", "start"]
```

---

## Frontend (React.js with Redux)
```javascript
import React, { useState } from 'react';
import axios from 'axios';
import { useAuth0 } from '@auth0/auth0-react';

function App() {
  const { loginWithRedirect, logout, user, isAuthenticated, getAccessTokenSilently } = useAuth0();
  const [prediction, setPrediction] = useState(null);

  const fetchPrediction = async () => {
    const token = await getAccessTokenSilently();
    const response = await axios.post('http://localhost:8000/predict/',
      { feature1: 3, feature2: 30 },
      { headers: { Authorization: `Bearer ${token}` } }
    );
    setPrediction(response.data.prediction);
  };

  return (
    <div>
      {!isAuthenticated ? (
        <button onClick={() => loginWithRedirect()}>Login</button>
      ) : (
        <>
          <p>Welcome {user.name}</p>
          <button onClick={() => logout()}>Logout</button>
          <button onClick={fetchPrediction}>Get Prediction</button>
          {prediction && <p>Predicted Orders: {prediction}</p>}
        </>
      )}
    </div>
  );
}
export default App;
```

---

## Deployment
1. **Run the Backend API**:
```bash
docker build -t ml-order-backend .
docker run -p 8000:8000 ml-order-backend
```

2. **Run the Frontend**:
```bash
docker build -t ml-order-frontend .
docker run -p 3000:3000 ml-order-frontend
```

3. **Set up RabbitMQ, Redis, and MongoDB** using Docker Compose.

---







