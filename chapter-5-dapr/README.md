# Chapter 5: Dapr Introduction

In this chapter, you'll learn how to use Dapr (Distributed Application Runtime) to build resilient, microservice-based applications. Dapr provides building blocks that make it easier to build portable, distributed systems.

## 🎯 Learning Objectives

- Understand Dapr architecture and the sidecar pattern
- Install Dapr on Kubernetes
- Implement Service Invocation between microservices
- Use State Management for persistent storage
- Work with Pub/Sub messaging
- Explore Input/Output Bindings

## 📚 Prerequisites

- Completed [Chapter 0: Setup and Prerequisites](../chapter-0-setup/README.md)
- AKS cluster running
- Basic understanding of microservices concepts

## 🔄 Load Your Configuration

```bash
# Load your lab configuration
source ./lab-config.sh

# Verify cluster access
kubectl get nodes
```

## 📖 Understanding Dapr

Dapr (Distributed Application Runtime) is a portable, event-driven runtime that makes it easy for developers to build resilient microservices. It provides building blocks as APIs that abstract away common distributed system challenges.

Key concepts:
- **Sidecar Architecture**: Dapr runs as a sidecar container alongside your application
- **Building Blocks**: APIs for common capabilities (state, pub/sub, service invocation)
- **Components**: Pluggable implementations (Redis, Azure Service Bus, etc.)
- **Language Agnostic**: Works with any language via HTTP or gRPC

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Dapr Architecture                               │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                           Pod                                         │  │
│  │  ┌─────────────────────┐        ┌─────────────────────────────────┐  │  │
│  │  │                     │        │         Dapr Sidecar            │  │  │
│  │  │   Your Application  │◄──────►│                                 │  │  │
│  │  │                     │  HTTP/ │  ┌────────┐ ┌────────────────┐  │  │  │
│  │  │   - Business Logic  │  gRPC  │  │Service │ │ State          │  │  │  │
│  │  │   - No Dapr SDK     │        │  │Invoke  │ │ Management     │  │  │  │
│  │  │     required        │        │  └────────┘ └────────────────┘  │  │  │
│  │  │                     │        │  ┌────────┐ ┌────────────────┐  │  │  │
│  │  └─────────────────────┘        │  │Pub/Sub │ │ Bindings       │  │  │  │
│  │                                 │  └────────┘ └────────────────┘  │  │  │
│  │                                 └─────────────┬───────────────────┘  │  │
│  └───────────────────────────────────────────────┼──────────────────────┘  │
│                                                  │                          │
│  ┌───────────────────────────────────────────────┼──────────────────────┐  │
│  │                           Components          │                       │  │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────────┐  │   ┌──────────────┐   │  │
│  │  │  Redis  │  │  Azure   │  │   Azure     │◄─┘   │  External    │   │  │
│  │  │ (State) │  │  Service │  │   Storage   │      │  Services    │   │  │
│  │  │         │  │  Bus     │  │   (Binding) │      │              │   │  │
│  │  └─────────┘  └──────────┘  └─────────────┘      └──────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

> **⚠️ Important Distinction**: Dapr is **not** a policy engine or infrastructure manager.
> It complements Kyverno, Gatekeeper, ASO, and Crossplane by addressing **application-level distributed system concerns**.
>
> | Tool | Layer | Purpose |
> |------|-------|--------|
> | Kyverno / Gatekeeper | Platform | Policy & guardrails |
> | ASO / Crossplane | Infrastructure | Cloud resource lifecycle |
> | Dapr | Application | Distributed system building blocks |

## 📦 Step 1: Install Dapr on Kubernetes

```bash
# Install Dapr using Helm
helm upgrade --install dapr dapr/dapr \
  --namespace dapr-system \
  --create-namespace \
  --set global.mtls.enabled=true \
  --set global.logAsJson=true

echo "⏳ Waiting for Dapr to be ready..."
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=dapr -n dapr-system --timeout=300s

echo "✅ Dapr installed successfully"
```

## 🔍 Step 2: Verify Dapr Installation

```bash
# Check Dapr pods
kubectl get pods -n dapr-system

# Check Dapr components
kubectl get components.dapr.io -A

# List Dapr CRDs
kubectl get crds | grep dapr
```

Expected pods:
```
dapr-operator
dapr-sidecar-injector
dapr-placement-server
dapr-scheduler-server (multiple replicas)
dapr-sentry
```

> **Note**: The Dapr dashboard is not installed by default. Components will show "No resources found" initially - we'll create them in the following steps.

## 📋 Step 3: Create a Namespace for Dapr Applications

```bash
# Create namespace for Dapr demo
kubectl create namespace dapr-demo

echo "✅ Namespace 'dapr-demo' created"
```

## 🗄️ Step 4: Deploy Redis for State Store

```bash
# Deploy Redis for state management and pub/sub
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: dapr-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: dapr-demo
spec:
  ports:
  - port: 6379
    targetPort: 6379
  selector:
    app: redis
EOF

echo "⏳ Waiting for Redis to be ready..."
kubectl wait --for=condition=ready pod -l app=redis -n dapr-demo --timeout=120s

echo "✅ Redis deployed"
```

> **🏢 Production Note**: In production, Redis is often replaced with **Azure Cache for Redis**, **Cosmos DB**, or other managed state stores. The Dapr component model allows swapping backends without changing application code.

## ⚙️ Step 5: Configure Dapr State Store Component

```bash
# Create Dapr State Store component
cat <<EOF | kubectl apply -f -
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
  namespace: dapr-demo
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
  - name: redisPassword
    value: ""
  - name: actorStateStore
    value: "true"
EOF

echo "✅ State Store component created"
```

## 📬 Step 6: Configure Dapr Pub/Sub Component

```bash
# Create Dapr Pub/Sub component
cat <<EOF | kubectl apply -f -
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
  namespace: dapr-demo
spec:
  type: pubsub.redis
  version: v1
  metadata:
  - name: redisHost
    value: redis:6379
  - name: redisPassword
    value: ""
EOF

echo "✅ Pub/Sub component created"
```

## 🔍 Step 7: Verify Components

```bash
# Check Dapr components
kubectl get components.dapr.io -n dapr-demo

# Describe components
kubectl describe component statestore -n dapr-demo
kubectl describe component pubsub -n dapr-demo
```

## 🚀 Step 8: Deploy a Backend Service (Order Service)

```bash
# Deploy Order Service with Dapr sidecar
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: dapr-demo
  labels:
    app: order-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "order-service"
        dapr.io/app-port: "5000"
        dapr.io/enable-api-logging: "true"
    spec:
      containers:
      - name: order-service
        image: python:3.11-slim
        ports:
        - containerPort: 5000
        command: ["python", "-c"]
        args:
        - |
          from http.server import HTTPServer, BaseHTTPRequestHandler
          import json
          import os

          class OrderHandler(BaseHTTPRequestHandler):
              orders = {}
              order_counter = 0

              def do_GET(self):
                  if self.path == '/health':
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps({"status": "healthy"}).encode())
                  elif self.path == '/orders':
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps(list(OrderHandler.orders.values())).encode())
                  else:
                      self.send_response(404)
                      self.end_headers()

              def do_POST(self):
                  if self.path == '/orders':
                      content_length = int(self.headers.get('Content-Length', 0))
                      body = self.rfile.read(content_length).decode()
                      order_data = json.loads(body) if body else {}
                      
                      OrderHandler.order_counter += 1
                      order_id = f"ORD-{OrderHandler.order_counter:04d}"
                      order = {"id": order_id, "status": "created", **order_data}
                      OrderHandler.orders[order_id] = order
                      
                      self.send_response(201)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps(order).encode())
                      print(f"Order created: {order}")
                  else:
                      self.send_response(404)
                      self.end_headers()

          print("Order Service starting on port 5000...")
          HTTPServer(('0.0.0.0', 5000), OrderHandler).serve_forever()
---
apiVersion: v1
kind: Service
metadata:
  name: order-service
  namespace: dapr-demo
spec:
  ports:
  - port: 5000
    targetPort: 5000
  selector:
    app: order-service
EOF

echo "⏳ Waiting for Order Service to be ready..."
kubectl wait --for=condition=ready pod -l app=order-service -n dapr-demo --timeout=120s

echo "✅ Order Service deployed with Dapr sidecar"
```

## 🔍 Step 9: Verify Dapr Sidecar Injection

```bash
# Check that the Dapr sidecar was injected
kubectl get pods -n dapr-demo -l app=order-service -o jsonpath='{.items[0].spec.containers[*].name}'
echo ""

# You should see: order-service daprd
echo "Expected containers: order-service daprd"

# Check sidecar logs
kubectl logs -n dapr-demo -l app=order-service -c daprd | head -20
```

## 🌐 Step 10: Deploy a Frontend Service (API Gateway)

```bash
# Deploy API Gateway that calls Order Service via Dapr
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  namespace: dapr-demo
  labels:
    app: api-gateway
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "api-gateway"
        dapr.io/app-port: "8080"
        dapr.io/enable-api-logging: "true"
    spec:
      containers:
      - name: api-gateway
        image: python:3.11-slim
        ports:
        - containerPort: 8080
        env:
        - name: DAPR_HTTP_PORT
          value: "3500"
        command: ["python", "-c"]
        args:
        - |
          from http.server import HTTPServer, BaseHTTPRequestHandler
          import urllib.request
          import json
          import os

          DAPR_PORT = os.getenv('DAPR_HTTP_PORT', '3500')

          class GatewayHandler(BaseHTTPRequestHandler):
              def do_GET(self):
                  if self.path == '/health':
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps({"status": "healthy", "service": "api-gateway"}).encode())
                  elif self.path == '/api/orders':
                      # Call order-service via Dapr service invocation
                      try:
                          dapr_url = f"http://localhost:{DAPR_PORT}/v1.0/invoke/order-service/method/orders"
                          print(f"Calling Dapr: {dapr_url}")
                          req = urllib.request.Request(dapr_url)
                          with urllib.request.urlopen(req, timeout=10) as response:
                              data = response.read()
                              self.send_response(200)
                              self.send_header('Content-type', 'application/json')
                              self.end_headers()
                              self.wfile.write(data)
                      except Exception as e:
                          print(f"Error: {e}")
                          self.send_response(500)
                          self.send_header('Content-type', 'application/json')
                          self.end_headers()
                          self.wfile.write(json.dumps({"error": str(e)}).encode())
                  else:
                      self.send_response(404)
                      self.end_headers()

              def do_POST(self):
                  if self.path == '/api/orders':
                      # Forward to order-service via Dapr
                      try:
                          content_length = int(self.headers.get('Content-Length', 0))
                          body = self.rfile.read(content_length)
                          
                          dapr_url = f"http://localhost:{DAPR_PORT}/v1.0/invoke/order-service/method/orders"
                          print(f"Calling Dapr: {dapr_url}")
                          req = urllib.request.Request(dapr_url, data=body, method='POST')
                          req.add_header('Content-Type', 'application/json')
                          with urllib.request.urlopen(req, timeout=10) as response:
                              data = response.read()
                              self.send_response(201)
                              self.send_header('Content-type', 'application/json')
                              self.end_headers()
                              self.wfile.write(data)
                      except Exception as e:
                          print(f"Error: {e}")
                          self.send_response(500)
                          self.send_header('Content-type', 'application/json')
                          self.end_headers()
                          self.wfile.write(json.dumps({"error": str(e)}).encode())
                  else:
                      self.send_response(404)
                      self.end_headers()

          print("API Gateway starting on port 8080...")
          HTTPServer(('0.0.0.0', 8080), GatewayHandler).serve_forever()
---
apiVersion: v1
kind: Service
metadata:
  name: api-gateway
  namespace: dapr-demo
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: api-gateway
EOF

echo "⏳ Waiting for API Gateway to be ready..."
kubectl wait --for=condition=ready pod -l app=api-gateway -n dapr-demo --timeout=120s

echo "✅ API Gateway deployed with Dapr sidecar"
```

## 🧪 Step 11: Test Service Invocation

```bash
# Get the API Gateway external IP
export GATEWAY_IP=$(kubectl get svc api-gateway -n dapr-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Wait for IP assignment
while [ -z "$GATEWAY_IP" ]; do
  echo "⏳ Waiting for LoadBalancer IP..."
  sleep 10
  export GATEWAY_IP=$(kubectl get svc api-gateway -n dapr-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
done

echo "API Gateway IP: $GATEWAY_IP"

# Test health endpoint
echo ""
echo "Testing health endpoint..."
curl -s http://$GATEWAY_IP/health | jq

# Create an order via the gateway (which calls order-service via Dapr)
echo ""
echo "Creating an order via Dapr service invocation..."
curl -s -X POST http://$GATEWAY_IP/api/orders \
  -H "Content-Type: application/json" \
  -d '{"product": "Kubernetes Book", "quantity": 2}' | jq

# List orders
echo ""
echo "Listing orders..."
curl -s http://$GATEWAY_IP/api/orders | jq
```

## 🗄️ Step 12: Deploy a State Management Example

```bash
# Deploy a service that uses Dapr state management
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: counter-service
  namespace: dapr-demo
  labels:
    app: counter-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: counter-service
  template:
    metadata:
      labels:
        app: counter-service
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "counter-service"
        dapr.io/app-port: "5001"
    spec:
      containers:
      - name: counter-service
        image: python:3.11-slim
        ports:
        - containerPort: 5001
        env:
        - name: DAPR_HTTP_PORT
          value: "3500"
        command: ["python", "-c"]
        args:
        - |
          from http.server import HTTPServer, BaseHTTPRequestHandler
          import urllib.request
          import json
          import os

          DAPR_PORT = os.getenv('DAPR_HTTP_PORT', '3500')
          STATE_STORE = 'statestore'

          class CounterHandler(BaseHTTPRequestHandler):
              def get_counter(self):
                  try:
                      url = f"http://localhost:{DAPR_PORT}/v1.0/state/{STATE_STORE}/counter"
                      req = urllib.request.Request(url)
                      with urllib.request.urlopen(req, timeout=5) as response:
                          return int(response.read().decode() or '0')
                  except:
                      return 0

              def save_counter(self, value):
                  url = f"http://localhost:{DAPR_PORT}/v1.0/state/{STATE_STORE}"
                  data = json.dumps([{"key": "counter", "value": value}]).encode()
                  req = urllib.request.Request(url, data=data, method='POST')
                  req.add_header('Content-Type', 'application/json')
                  urllib.request.urlopen(req, timeout=5)

              def do_GET(self):
                  if self.path == '/counter':
                      counter = self.get_counter()
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps({"counter": counter}).encode())
                  else:
                      self.send_response(404)
                      self.end_headers()

              def do_POST(self):
                  if self.path == '/counter/increment':
                      counter = self.get_counter() + 1
                      self.save_counter(counter)
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps({"counter": counter}).encode())
                      print(f"Counter incremented to: {counter}")
                  else:
                      self.send_response(404)
                      self.end_headers()

          print("Counter Service starting on port 5001...")
          HTTPServer(('0.0.0.0', 5001), CounterHandler).serve_forever()
---
apiVersion: v1
kind: Service
metadata:
  name: counter-service
  namespace: dapr-demo
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5001
  selector:
    app: counter-service
EOF

echo "⏳ Waiting for Counter Service to be ready..."
kubectl wait --for=condition=ready pod -l app=counter-service -n dapr-demo --timeout=120s

echo "✅ Counter Service deployed with state management"
```

## 🧪 Step 13: Test State Management

```bash
# Get the Counter Service external IP
export COUNTER_IP=$(kubectl get svc counter-service -n dapr-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

while [ -z "$COUNTER_IP" ]; do
  echo "⏳ Waiting for LoadBalancer IP..."
  sleep 10
  export COUNTER_IP=$(kubectl get svc counter-service -n dapr-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
done

echo "Counter Service IP: $COUNTER_IP"

# Get current counter (should be 0 initially)
echo ""
echo "Current counter value:"
curl -s http://$COUNTER_IP/counter | jq

# Increment the counter multiple times
echo ""
echo "Incrementing counter..."
for i in {1..5}; do
  curl -s -X POST http://$COUNTER_IP/counter/increment | jq
  sleep 1
done

# Check final value
echo ""
echo "Final counter value (persisted in Redis):"
curl -s http://$COUNTER_IP/counter | jq

# Verify in Redis directly (Dapr uses a hash structure)
echo ""
echo "Verifying in Redis (listing Dapr state keys):"
kubectl exec -n dapr-demo deploy/redis -- redis-cli KEYS "*counter*"
```

> **Note**: Dapr stores state in Redis using a specific key format and data structure. The counter value is persisted and will survive pod restarts.

## 📬 Step 14: Deploy Pub/Sub Example

```bash
# Deploy a publisher service
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: publisher
  namespace: dapr-demo
  labels:
    app: publisher
spec:
  replicas: 1
  selector:
    matchLabels:
      app: publisher
  template:
    metadata:
      labels:
        app: publisher
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "publisher"
    spec:
      containers:
      - name: publisher
        image: python:3.11-slim
        env:
        - name: DAPR_HTTP_PORT
          value: "3500"
        command: ["python", "-c"]
        args:
        - |
          import urllib.request
          import json
          import time
          import os

          DAPR_PORT = os.getenv('DAPR_HTTP_PORT', '3500')

          while True:
              time.sleep(10)
              message = {
                  "orderId": f"order-{int(time.time())}",
                  "timestamp": time.strftime("%Y-%m-%d %H:%M:%S")
              }
              
              url = f"http://localhost:{DAPR_PORT}/v1.0/publish/pubsub/orders"
              data = json.dumps(message).encode()
              req = urllib.request.Request(url, data=data, method='POST')
              req.add_header('Content-Type', 'application/json')
              
              try:
                  urllib.request.urlopen(req, timeout=5)
                  print(f"Published: {message}")
              except Exception as e:
                  print(f"Error publishing: {e}")
EOF

# Deploy a subscriber service
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: subscriber
  namespace: dapr-demo
  labels:
    app: subscriber
spec:
  replicas: 1
  selector:
    matchLabels:
      app: subscriber
  template:
    metadata:
      labels:
        app: subscriber
      annotations:
        dapr.io/enabled: "true"
        dapr.io/app-id: "subscriber"
        dapr.io/app-port: "5002"
    spec:
      containers:
      - name: subscriber
        image: python:3.11-slim
        ports:
        - containerPort: 5002
        command: ["python", "-c"]
        args:
        - |
          from http.server import HTTPServer, BaseHTTPRequestHandler
          import json

          class SubscriberHandler(BaseHTTPRequestHandler):
              received_messages = []

              def do_GET(self):
                  if self.path == '/dapr/subscribe':
                      # Tell Dapr what topics to subscribe to
                      subscriptions = [
                          {
                              "pubsubname": "pubsub",
                              "topic": "orders",
                              "route": "/orders"
                          }
                      ]
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps(subscriptions).encode())
                  elif self.path == '/messages':
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps(SubscriberHandler.received_messages[-10:]).encode())
                  else:
                      self.send_response(404)
                      self.end_headers()

              def do_POST(self):
                  if self.path == '/orders':
                      content_length = int(self.headers.get('Content-Length', 0))
                      body = self.rfile.read(content_length).decode()
                      message = json.loads(body)
                      
                      print(f"Received order: {message}")
                      SubscriberHandler.received_messages.append(message)
                      
                      self.send_response(200)
                      self.send_header('Content-type', 'application/json')
                      self.end_headers()
                      self.wfile.write(json.dumps({"status": "SUCCESS"}).encode())
                  else:
                      self.send_response(404)
                      self.end_headers()

          print("Subscriber starting on port 5002...")
          HTTPServer(('0.0.0.0', 5002), SubscriberHandler).serve_forever()
---
apiVersion: v1
kind: Service
metadata:
  name: subscriber
  namespace: dapr-demo
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5002
  selector:
    app: subscriber
EOF

echo "⏳ Waiting for Pub/Sub services to be ready..."
kubectl wait --for=condition=ready pod -l app=publisher -n dapr-demo --timeout=120s
kubectl wait --for=condition=ready pod -l app=subscriber -n dapr-demo --timeout=120s

echo "✅ Pub/Sub services deployed"
```

## 🧪 Step 15: Test Pub/Sub Messaging

```bash
# Check publisher logs
echo "Publisher logs (publishing messages every 10 seconds):"
kubectl logs -n dapr-demo -l app=publisher -c publisher --tail=5

# Check subscriber logs
echo ""
echo "Subscriber logs (receiving messages):"
kubectl logs -n dapr-demo -l app=subscriber -c subscriber --tail=10

# Get subscriber service IP
export SUBSCRIBER_IP=$(kubectl get svc subscriber -n dapr-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

while [ -z "$SUBSCRIBER_IP" ]; do
  echo "⏳ Waiting for LoadBalancer IP..."
  sleep 10
  export SUBSCRIBER_IP=$(kubectl get svc subscriber -n dapr-demo -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
done

# Check received messages
echo ""
echo "Messages received by subscriber:"
curl -s http://$SUBSCRIBER_IP/messages | jq
```

## 📊 Step 16: View All Dapr Resources

```bash
echo "========================================"
echo "  DAPR RESOURCES"
echo "========================================"

echo ""
echo "Dapr Components:"
kubectl get components.dapr.io -n dapr-demo

echo ""
echo "Dapr-enabled Pods:"
kubectl get pods -n dapr-demo -o custom-columns=NAME:.metadata.name,READY:.status.containerStatuses[*].ready,DAPR:.metadata.annotations.dapr\\.io/app-id

echo ""
echo "Services:"
kubectl get svc -n dapr-demo
```

## 📋 Step 17: Dapr Building Blocks Summary

```bash
echo "========================================"
echo "  DAPR BUILDING BLOCKS DEMONSTRATED"
echo "========================================"
echo ""
echo "1. SERVICE INVOCATION"
echo "   - api-gateway calls order-service via Dapr"
echo "   - URL: http://localhost:3500/v1.0/invoke/order-service/method/orders"
echo ""
echo "2. STATE MANAGEMENT"
echo "   - counter-service stores state in Redis"
echo "   - URL: http://localhost:3500/v1.0/state/statestore"
echo ""
echo "3. PUB/SUB MESSAGING"
echo "   - publisher publishes to 'orders' topic"
echo "   - subscriber receives messages automatically"
echo "   - URL: http://localhost:3500/v1.0/publish/pubsub/orders"
echo ""
echo "4. OTHER BUILDING BLOCKS (not demonstrated):"
echo "   - Bindings: Connect to external systems"
echo "   - Actors: Virtual actor pattern"
echo "   - Secrets: Secret management"
echo "   - Configuration: Dynamic configuration"
echo "   - Distributed Lock: Mutual exclusion"
```

## 🧹 Step 18: Cleanup (Optional)

```bash
# To cleanup Dapr demo resources:
# kubectl delete namespace dapr-demo

echo "To cleanup, run: kubectl delete namespace dapr-demo"
```

## 📝 Step 19: Save Dapr Configuration

```bash
# Update lab config
cat >> ./lab-config.sh <<EOF

# Dapr Configuration
export GATEWAY_IP="$GATEWAY_IP"
export COUNTER_IP="$COUNTER_IP"
export SUBSCRIBER_IP="$SUBSCRIBER_IP"
EOF

echo "✅ Dapr configuration saved"
```

## 🎓 Summary

You have successfully:
- ✅ Installed Dapr on Kubernetes
- ✅ Configured state store and pub/sub components
- ✅ Implemented Service Invocation between microservices
- ✅ Used State Management for persistent storage
- ✅ Set up Pub/Sub messaging between services
- ✅ Understood the sidecar pattern

## 📝 Important Notes

- **Sidecar Injection**: Dapr sidecars are automatically injected via annotations
- **App ID**: Each service needs a unique `dapr.io/app-id`
- **Components**: Components can be scoped to specific applications
- **mTLS**: Dapr provides automatic mTLS between sidecars. Note: mTLS secures communication **between sidecars**, not directly between application containers.
- **Actors**: Dapr supports virtual actors for stateful, single-threaded programming models. Actors are best introduced after mastering service invocation and state management, once lifecycle semantics are understood.

## 🔍 Troubleshooting

```bash
# Check Dapr sidecar injector logs
kubectl logs -n dapr-system -l app=dapr-sidecar-injector

# Check sidecar logs for a specific app
kubectl logs -n dapr-demo -l app=order-service -c daprd

# Verify component configuration
kubectl describe component statestore -n dapr-demo

# Test Dapr sidecar health
kubectl exec -n dapr-demo deploy/order-service -c daprd -- wget -qO- http://localhost:3500/v1.0/healthz

# Check subscriptions
kubectl exec -n dapr-demo deploy/subscriber -c daprd -- wget -qO- http://localhost:3500/v1.0/metadata | jq
```

## 🎉 Lab Complete!

Congratulations! You have completed the Extending Kubernetes Lab!

### What You've Learned:

| Chapter | Topic                  | Key Concepts                                          |
| ------- | ---------------------- | ----------------------------------------------------- |
| 1       | Kyverno                | YAML-based policies, validation, mutation             |
| 2       | OPA Gatekeeper         | Rego language, ConstraintTemplates                    |
| 3       | Azure Service Operator | Kubernetes operators, Azure resource management       |
| 4       | Crossplane             | Compositions, XRDs, multi-cloud control plane         |
| 5       | Dapr                   | Sidecar pattern, building blocks, distributed systems |

### 🧹 Final Cleanup

To remove all resources created during this lab:

```bash
# Load configuration
source ./lab-config.sh

# Delete the entire resource group (removes everything)
az group delete \
  --name $RESOURCE_GROUP \
  --yes \
  --no-wait

echo "🗑️ Resource group deletion initiated"
echo "✅ Lab cleanup complete!"
```

---

**Thank you for completing this lab!**

For questions or feedback, please contact your instructor.

---

## ⚠️ Disclaimer

Educational/lab purposes only. Calculations and/or statements may contain errors.

This documentation is provided "as is" without warranty of any kind. The author takes no responsibility for any errors, omissions, or inaccuracies contained herein. This material may contain incorrect or outdated information. Always verify configurations against official Microsoft Azure and Kubernetes documentation before use in production environments. Use at your own risk.
