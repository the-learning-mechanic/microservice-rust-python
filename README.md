# AWS App Runner Deployment Guide

This guide demonstrates how to deploy both Python FastAPI and Rust Rocket applications to AWS App Runner using ECR and CodeBuild.

## Python FastAPI Version

### Project Structure
```
.
├── main.py
├── requirements.txt
├── Dockerfile
├── buildspec.yml
└── test_main.py
```

### Code
```python
# main.py
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}
```

```dockerfile
# Dockerfile
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

## Rust Rocket Version

### Project Structure
```
.
├── Cargo.toml
├── src/
│   └── main.rs
├── Dockerfile
├── buildspec.yml
└── tests/
    └── test_main.rs
```

### Code
```rust
// src/main.rs
use rocket::serde::json::Json;
use serde::Serialize;

#[derive(Serialize)]
struct Response {
    message: String,
}

#[rocket::get("/")]
fn index() -> Json<Response> {
    Json(Response {
        message: "Hello, World!".to_string(),
    })
}

#[rocket::main]
async fn main() {
    rocket::build()
        .mount("/", rocket::routes![index])
        .launch()
        .await
        .unwrap();
}
```

```dockerfile
# Dockerfile
FROM rust:1.75 as builder
WORKDIR /usr/src/app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
COPY --from=builder /usr/src/app/target/release/rust-aws-app /usr/local/bin/
EXPOSE 8000
CMD ["rust-aws-app"]
```

## Deployment Steps

1. Create ECR Repository:
```bash
aws ecr create-repository --repository-name your-app-name
```

2. Configure CodeBuild:
    - Create a new build project
    - Set environment variables:
        - AWS_ACCOUNT_ID
        - AWS_DEFAULT_REGION
        - IMAGE_REPO_NAME
    - Use buildspec.yml from repository
    - Enable GitHub webhook for automatic builds

3. Set up App Runner:
    - Create new service
    - Choose ECR source
    - Select repository and tag
    - Configure auto-deployment
    - Set port to match Dockerfile (8080 for Python, 8000 for Rust)

4. Test Deployment:
    - Push changes to repository
    - Monitor CodeBuild execution
    - Check App Runner service status
    - Verify endpoint using provided URL

## Performance Considerations

- Python FastAPI:
    - Lighter initial container build
    - Faster cold starts
    - Good for rapid prototyping

- Rust Rocket:
    - Better runtime performance
    - Lower memory usage
    - Longer build times
    - Ideal for production workloads

## Cost Optimization

- Use ECR lifecycle policies to clean old images
- Configure App Runner auto-scaling
- Consider Rust for lower compute costs in high-traffic scenarios

## Monitoring and Logs

Access logs and metrics:
```bash
aws logs describe-log-groups --log-group-prefix /aws/apprunner/
aws cloudwatch get-metric-statistics --namespace AWS/AppRunner
```