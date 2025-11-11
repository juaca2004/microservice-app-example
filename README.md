

# Microservice App - PRFT DevOps Training

This is the application you will use throughout the entire training. It aims to teach you the fundamental concepts required in a real-world project. You will find a basic TODO application designed with a [microservice architecture](https://microservices.io).

Although it is a TODO app, it is interesting because each microservice is built using different programming languages and frameworks (Go, Python, Vue, Java, and NodeJS). This setup allows you to experiment with multiple build tools and environments.

---

## Components

Each folder contains a detailed explanation of its respective component:

1. **[Users API](/users-api)** — A Spring Boot application providing user profile management. Currently, it supports fetching a single user and all users (no full CRUD yet).
2. **[Auth API](/auth-api)** — A Go-based service that handles authentication and generates [JWT](https://jwt.io/) tokens for secure communication with other APIs.
3. **[TODOs API](/todos-api)** — A NodeJS service implementing CRUD operations for user TODOs. It also logs “create” and “delete” actions to a [Redis](https://redis.io/) queue.
4. **[Log Message Processor](/log-message-processor)** — A Python-based queue processor that reads messages from Redis and outputs them to the console.
5. **[Frontend](/frontend)** — A Vue.js application that provides the user interface.

---

## Architecture Overview

Below is the architecture diagram showing how all components interact:

![microservice-app-example](/arch-img/Microservices.png)

---

# Microservice Project with Docker and Kubernetes

In this project, I designed and implemented a **microservice-based architecture**, using **Docker** for containerization and **Kubernetes** for deployment.
Each service performs a specific role and communicates with others via HTTP requests or internal messaging.

---

## Project Structure

The system consists of five main microservices, each with its own `Dockerfile` and Kubernetes deployment configuration:

| Service                   | Language / Framework   | Port | Description                                           |
| ------------------------- | ---------------------- | ---- | ----------------------------------------------------- |
| **frontend**              | Node.js (Vue or React) | 8080 | User interface that communicates with the APIs.       |
| **auth-api**              | Go                     | 8000 | Handles user authentication and JWT token generation. |
| **users-api**             | Java (Spring Boot)     | 8083 | Manages user data and profiles.                       |
| **todos-api**             | Node.js                | 8082 | Manages tasks and communicates with Redis and Zipkin. |
| **log-message-processor** | Python                 | 8084 | Processes log and queue messages from Redis.          |

---

## 1. Building Docker Images

To build the Docker images, navigate to the project root and run the following commands.
Each image is then pushed to Docker Hub for later use in Kubernetes deployments.

```bash
# Build frontend image
docker build -t juaca2004/frontendmicroservice -f ./frontend/Dockerfile .

# Build Auth API image
docker build -t juaca2004/authapi -f ./auth-api/Dockerfile .

# Build Users API image
docker build -t juaca2004/userapi -f ./users-api/Dockerfile .

# Build Todos API image
docker build -t juaca2004/todosapi -f ./todos-api/Dockerfile .

# Build Log Processor image
docker build -t juaca2004/logmessageprocessor -f ./log-message-processor/Dockerfile .
```

---

## 2. Local Container Testing

Before deploying to Kubernetes, each container was tested locally to verify that all services worked properly.

```bash
docker run -p 8080:8080 juaca2004/frontendmicroservice
docker run -p 8000:8000 juaca2004/authapi
docker run -p 8083:8083 juaca2004/userapi
```

During this stage, required environment variables were configured for inter-service communication (e.g., frontend’s connection to `auth-api`).

---

## Kubernetes Manifest Structure

The `deployment.yaml` file includes the following Kubernetes resources:

### 1. **Namespaces**

Separate namespaces were created for each component:

* `frontend-ns`
* `auth-ns`
* `users-ns`
* `todos-ns`
* `redis-ns`
* `log-ns`

### 2. **Deployments and Services**

Each microservice includes a dedicated **Deployment** and **ClusterIP Service** to ensure stable internal DNS-based communication.

Example — Frontend:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend-ns
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
```

Service for Frontend:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: frontend-ns
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
```

---
Secrets

JWT secrets are stored securely in each API’s namespace using Kubernetes Secrets of type Opaque.
These secrets store the base64-encoded JWT key shared across the auth-api, users-api, and todos-api services.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: jwt-secret
  namespace: auth-ns
type: Opaque
data:
  JWT_SECRET: bXlmYW5jeXNlY3JldA== 
---
apiVersion: v1
kind: Secret
metadata:
  name: jwt-secret
  namespace: users-ns
type: Opaque
data:
  JWT_SECRET: bXlmYW5jeXNlY3JldA==
---
apiVersion: v1
kind: Secret
metadata:
  name: jwt-secret
  namespace: todos-ns
type: Opaque
data:
  JWT_SECRET: bXlmYW5jeXNlY3JldA==

```

### 3. **Horizontal Pod Autoscaler (HPA)**

The `frontend` service includes an HPA configuration to automatically scale between 2 and 5 replicas based on CPU utilization.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
spec:
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

---

### 4. **Deployment Strategies**

Two deployment strategies were used:

* **RollingUpdate (Frontend):** Gradual deployment ensuring zero downtime.
* **Recreate (Auth API):** Stops old pods before creating new ones (causes temporary downtime but ensures version consistency).

---

### 5. **Network Policies**

Security policies were applied using **NetworkPolicy** resources:

* `default-deny-all` — Blocks all traffic by default in each namespace.
* Specific rules were added to allow:

  * External → `frontend`
  * `frontend` → `auth-api` and `todos-api`
  * `auth-api` → `users-api`
  * `todos-api` → `redis`
  * `log-message-processor` → `redis`

---

## Network Security Model

The system follows a **Zero Trust** approach:

* All traffic is denied by default.
* Only explicitly defined paths are allowed.
* Each microservice communicates only with necessary components.

---

## Environment Variables

| Variable                   | Service                        | Description                        |
| -------------------------- | ------------------------------ | ---------------------------------- |
| `AUTH_API_ADDRESS`         | frontend                       | Internal URL of the Auth API.      |
| `TODOS_API_ADDRESS`        | frontend                       | Internal URL of the Todos API.     |
| `JWT_SECRET`               | auth-api, todos-api, users-api | Secret key for signing JWT tokens. |
| `REDIS_HOST`, `REDIS_PORT` | todos-api, log-processor       | Redis service connection details.  |
| `REDIS_CHANNEL`            | todos-api, log-processor       | Message exchange channel.          |

---

## Deployment Commands

Make sure you are in the same directory as `deployment.yaml`.

### Create all resources

```bash
kubectl apply -f deployment.yaml
```

### Check created namespaces

```bash
kubectl get namespaces
```

### Verify deployments and pods

```bash
kubectl get deployments --all-namespaces
kubectl get pods --all-namespaces
```

### Check services

```bash
kubectl get svc --all-namespaces
```

### Check network policies

```bash
kubectl get networkpolicy --all-namespaces
```

### View logs

```bash
kubectl logs -l app=log-message-processor -n log-ns
```

---

## Monitoring with Prometheus and Grafana

To enhance observability, **Prometheus** and **Grafana** were installed for detailed cluster monitoring.

### Install Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus -n prometheus --create-namespace
kubectl get pods -n prometheus
```

Enable the ingress controller (for Minikube):

```bash
minikube addons enable ingress
```

Create an ingress rule (`ingress.yaml`) for NGINX to route external traffic to Prometheus.

---

### Install Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana --namespace monitoreo --create-namespace

# Get the secret name
SECRET_NAME=$(kubectl get secret --namespace monitoreo -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")

# Decode and show the password
kubectl get secret --namespace monitoreo $SECRET_NAME -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Access Grafana via port-forwarding:

```bash
kubectl port-forward svc/grafana 3000:80 -n monitoreo
```

After adding Prometheus as a data source and importing a dashboard, Grafana will display real-time metrics from the cluster.

---

## Evidence of Successful Deployment

![Cluster Running Example 1](https://github.com/user-attachments/assets/2b95c7cf-3ede-4e25-a6d3-2380c9a1671f)
![Cluster Running Example 2](https://github.com/user-attachments/assets/abe0a070-6d9d-4de3-b8ec-97a508d2a77c)

<img width="1413" height="925" alt="image" src="https://github.com/user-attachments/assets/06223fdd-6f96-4b26-9d01-dd538597ce47" />

<img width="1406" height="634" alt="image" src="https://github.com/user-attachments/assets/b41ae6ed-df5f-491f-a4a2-6b57079d7581" />










