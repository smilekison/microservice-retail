![Banner](./docs/images/banner.png)

<div align="center">

[![License](https://img.shields.io/github/license/smilekison/microservice-retail?color=green)](https://github.com/smilekison/microservice-retail/blob/main/LICENSE)
[![Repo Size](https://img.shields.io/github/repo-size/smilekison/microservice-retail)](https://github.com/smilekison/microservice-retail)
[![Languages](https://img.shields.io/github/languages/count/smilekison/microservice-retail)](https://github.com/smilekison/microservice-retail)


## microservice-retail — DevOps-focused README

A trimmed and DevOps-oriented README for the microservice-retail sample application. This README focuses on deployment, observability, CI/CD, and operational best practices to help platform and SRE teams deploy and run the application reliably.


---

### Quick links

- Architecture (animated): /docs/images/animated-architecture.svg
- Docs: ./docs
- Terraform modules: ./terraform
- Helm charts: included in each service directory (where present)

---

## Goals for DevOps / SRE

- Reproducible, idempotent infrastructure provisioning (Terraform)
- Container images built for multi-arch and scanned for vulnerabilities
- Automated CI/CD pipeline (build, test, image push, deploy) with promotion gates
- Production-grade observability: metrics (Prometheus), traces (OpenTelemetry), logs (structured JSON)
- Secure secrets management (HashiCorp Vault / AWS Secrets Manager / SSM)
- Horizontal autoscaling, resource requests/limits, and readiness/liveness probes


## Animated Architecture

![Animated architecture](/docs/images/animated-architecture.svg)


## Recommended deployment flows

1) Local dev: Docker/Docker Compose — fast iteration
2) Kubernetes (EKS/GKE/AKS) — staging and production
3) Managed containers (ECS/Fargate, App Runner) — simple managed deployments
4) Terraform for infra lifecycle management


## Quickstart (Dev)

Prereqs: Docker 24+, Docker Compose v2+, git

Run a UI-only container (quick test):

```
docker run -it --rm -p 8888:8080 public.ecr.aws/aws-containers/retail-store-sample-ui:1.0.0
```

Or run the docker compose stack (uses included compose files):

```
DB_PASSWORD='testing' docker compose -f docker-compose.yaml up --build
```

Stop and remove resources:

```
docker compose -f docker-compose.yaml down --volumes
```


## Kubernetes (recommended staging flow)

Prereqs: kubectl, a K8s cluster (EKS/GKE/AKS), helm

Apply manifests (example using release artifacts):

```
kubectl apply -f https://github.com/aws-containers/retail-store-sample-app/releases/latest/download/kubernetes.yaml
kubectl wait --for=condition=available deployments --all --timeout=180s
```

Get the UI service (LoadBalancer / NodePort):

```
kubectl get svc ui -o wide
```

Helm (if using charts in repo):

```
helm repo add local-retail https://example.com/charts
helm upgrade --install retail ./charts/retail -n retail --create-namespace
```


## Terraform (infrastructure-as-code)

This repository contains Terraform examples under ./terraform. Use these as starting points and adapt to your org standards (backend state, workspaces, modules, tagging, policies).

Example apply (ensure backend configured):

```
cd terraform/eks/default
terraform init
terraform plan -out plan.tf
terraform apply plan.tf
```

Tip: Use separate state per environment and enable remote state locking.


## CI/CD (example GitHub Actions workflow)

Save this as .github/workflows/ci-cd.yaml. It shows a minimal pipeline for build, test, container scan, and deploy to a Kubernetes cluster.

```yaml
name: CI/CD
on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
      - name: Build (maven)
        run: ./mvnw -B -DskipTests package
      - name: Build and push images
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/microservice-retail:latest
      - name: Scan image
        uses: aquasecurity/trivy-action@v1
        with:
          image-ref: ghcr.io/${{ github.repository_owner }}/microservice-retail:latest
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to cluster
        uses: azure/k8s-deploy@v4
        with:
          manifests: | 
            k8s/*.yaml
```

Notes:
- Use image tags by digest for immutable deployments
- Gate promotions with integration tests and manual approvals for prod
- Use OIDC or dedicated deploy service accounts for auth


## Observability and Monitoring

- Metrics: Prometheus scraping endpoints (all services expose Prometheus metrics)
- Tracing: OpenTelemetry OTLP exporters are built into services — configure OTLP collector
- Logs: Output structured JSON (enables ingestion by Loki/Elasticsearch/CloudWatch)
- Dashboards: Create Grafana dashboards for request latency, error rate, throughput, and resource usage

Example: run prometheus and grafana via Helm in-cluster and configure scraping for the services.


## Security & Secrets

- Do NOT store secrets in the repo. Use external secret stores (Vault, AWS Secrets Manager, SSM)
- Use KMS encryption for state files and secrets at rest
- Enforce least-privilege IAM roles for services and CI runners
- Enable dependency scanning in CI (Trivy, Snyk)


## Best practices for production readiness

- Add resource requests & limits for all pods
- Configure readiness and liveness probes
- Implement circuit-breakers and retries in client calls
- Enforce network policies and use service mesh where needed
- Run chaos testing in staging (some services include chaos endpoints for testing)


## Troubleshooting

- Pod not starting: kubectl describe pod / kubectl logs
- High latency: check tracing spans, CPU/Memory throttling, and DB slow queries
- Data persistence: verify backing services (RDS, DynamoDB, Redis) are reachable


## Contributing / Security

See CONTRIBUTING.md for security reporting guidelines.


## License

This project is licensed under the MIT-0 License. See LICENSE for details.
