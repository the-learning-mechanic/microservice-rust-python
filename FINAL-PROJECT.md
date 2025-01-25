# Rust Kubernetes Final Project

Build and deploy a production-grade Rust web service on Kubernetes with monitoring and CI/CD.

## Structure
```
.
├── src/
│   ├── main.rs
│   ├── handlers/
│   │   └── health.rs
│   └── models/
│       └── status.rs
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── monitoring/
│       ├── prometheus.yaml
│       └── grafana.yaml
├── .github/
│   └── workflows/
│       └── ci.yaml
├── Dockerfile
├── Cargo.toml
└── README.md
```

## Implementation

### 1. Base Web Service

```toml
# Cargo.toml
[package]
name = "rust-k8s-service"
version = "0.1.0"
edition = "2021"

[dependencies]
rocket = "0.5"
serde = { version = "1.0", features = ["derive"] }
prometheus = "0.13"
tokio = { version = "1.0", features = ["full"] }
```

```rust
// src/main.rs
use rocket::serde::json::Json;
use prometheus::{Registry, Counter, Encoder, TextEncoder};

#[derive(Debug)]
struct Metrics {
    requests: Counter,
}

#[rocket::get("/")]
fn index() -> Json<Response> {
    Json(Response {
        status: "healthy".to_string(),
        version: env!("CARGO_PKG_VERSION").to_string(),
    })
}

#[rocket::get("/metrics")]
fn metrics() -> String {
    let encoder = TextEncoder::new();
    let metric_families = REGISTRY.gather();
    let mut buffer = vec![];
    encoder.encode(&metric_families, &mut buffer).unwrap();
    String::from_utf8(buffer).unwrap()
}

#[rocket::main]
async fn main() {
    rocket::build()
        .mount("/", routes![index, metrics])
        .launch()
        .await
        .unwrap();
}
```

### 2. Containerization

```dockerfile
FROM rust:1.75 as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates
COPY --from=builder /app/target/release/rust-k8s-service /usr/local/bin/
EXPOSE 8000
CMD ["rust-k8s-service"]
```

### 3. Kubernetes Configuration

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rust-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rust-service
  template:
    metadata:
      labels:
        app: rust-service
    spec:
      containers:
      - name: rust-service
        image: your-registry/rust-k8s-service:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: rust-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: rust-service
```

### 4. Monitoring

```yaml
# k8s/monitoring/prometheus.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: rust-service
spec:
  selector:
    matchLabels:
      app: rust-service
  endpoints:
  - port: http
    path: /metrics
```

### 5. CI/CD Setup

```yaml
# .github/workflows/ci.yaml
name: CI/CD
on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Build and Test
      run: |
        cargo build --release
        cargo test

    - name: Build Docker image
      run: docker build -t your-registry/rust-k8s-service:${{ github.sha }} .

    - name: Push to registry
      run: |
        echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login -u "${{ secrets.REGISTRY_USERNAME }}" --password-stdin
        docker push your-registry/rust-k8s-service:${{ github.sha }}

    - name: Deploy to Kubernetes
      run: |
        kubectl set image deployment/rust-service rust-service=your-registry/rust-k8s-service:${{ github.sha }}
```

## Deployment

1. Create Cluster:
```bash
# AWS EKS
eksctl create cluster --name rust-cluster --region us-west-2 --nodes 3

# OR Google GKE
gcloud container clusters create rust-cluster --num-nodes=3
```

2. Install Monitoring:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```

3. Deploy:
```bash
kubectl apply -f k8s/
```

4. Verify:
```bash
kubectl get pods
kubectl get services
```

## Requirements

1. Core:
    - Rust web service with health checks
    - Prometheus metrics
    - Optimized containers
    - Resource management

2. Kubernetes:
    - High availability
    - Rolling updates
    - Load balancing
    - Monitoring

3. CI/CD:
    - Automated tests
    - Security scanning
    - Automated deployment
    - Version control

## Extensions

1. GraphQL API (Juniper)
2. Database integration
3. Custom metrics
4. Pod autoscaling
5. Service mesh

## Cleanup

```bash
# AWS
eksctl delete cluster --name rust-cluster

# GCP
gcloud container clusters delete rust-cluster
```