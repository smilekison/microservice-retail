# microservice-retail

A lightweight sample microservice retail application (product catalog, cart, orders, UI). This repository contains multiple small services implemented in Java, Go and TypeScript and example deployment configurations.

Purpose: provide a reproducible app you can use to practice packaging, deployment and operational tasks (containers, orchestration, infrastructure-as-code).

---

## Required tools (minimum)

1. Git (>=2.30)
2. Docker (>=20.10) and Docker Compose (v2+) for local container runs
3. kubectl (latest stable) for Kubernetes deployments
4. Helm (v3+) if installing charts
5. Terraform (>=1.5) for infrastructure provisioning
6. A container registry (Docker Hub, GHCR, ECR) and credentials for pushing images
7. Optional: AWS CLI (for EKS/ECS/App Runner), or cloud provider CLIs for your target environment

Make sure you have credentials and network access to the target servers/clusters before starting.

---

## Quick overview of deployment techniques

This section describes, step-by-step, how to deploy the app to different server environments. Choose one technique depending on your target server or environment.

1) Docker (single-server / VM)

- Use-case: fast smoke test or single-node demo on a Linux server or VM.

Steps:

1. Install Docker on the target server and ensure your user can run docker (or use sudo).
2. Pull and run the UI image (example):

```bash
docker run -d --restart=always --name retail-ui -p 8888:8080 \
  public.ecr.aws/aws-containers/retail-store-sample-ui:1.0.0
```

3. If you need catalog/cart services too, either run their images or use local/in-memory mode depending on the service configuration.
4. Verify: open http://<server-ip>:8888 or curl http://localhost:8888
5. Stop/cleanup:

```bash
docker stop retail-ui && docker rm retail-ui
```

Notes:
- For production-like usage, run containers under a process manager (systemd) or use a container runtime with proper restart policies.
- Configure environment variables (DB endpoints, secrets) via Docker `-e` flags or files; do not bake secrets into images.


2) Docker Compose (single server / VM with multi-container)

- Use-case: run the whole stack (UI, catalog, cart, DB) on one server for staging or demo.

Steps:

1. Copy the repository or download the docker-compose.yaml provided in the repo or release.
2. Create an .env file (or export environment variables) with required secrets, e.g. DB_PASSWORD.

Example minimal env file (.env):

```env
DB_PASSWORD=testing
```

3. Start the stack:

```bash
DB_PASSWORD='testing' docker compose -f docker-compose.yaml up -d --build
```

4. Check status:

```bash
docker compose -f docker-compose.yaml ps
```

5. Tear down when finished:

```bash
docker compose -f docker-compose.yaml down --volumes
```

Notes:
- Use named volumes for persistence. Monitor container logs with `docker compose logs -f`.
- In production, prefer orchestrators (Kubernetes, ECS) or convert Compose to Kubernetes manifests via tools if needed.


3) Kubernetes (recommended for server clusters)

- Use-case: staging and production on a cluster (EKS/GKE/AKS or on-prem K8s).

Pre-reqs: a working cluster, kubectl configured with correct context, and optional Helm if charts are used.

Steps (manifest-based):

1. Apply the release manifest (example):

```bash
kubectl apply -f https://github.com/aws-containers/retail-store-sample-app/releases/latest/download/kubernetes.yaml
kubectl wait --for=condition=available deployments --all --timeout=180s
```

2. Inspect resources and get service endpoint:

```bash
kubectl get pods -n default
kubectl get svc ui -o wide
```

3. If LoadBalancer type service is used, obtain external IP or use port-forward for testing:

```bash
kubectl port-forward svc/ui 8888:8080
# then open http://localhost:8888
```

4. To remove:

```bash
kubectl delete -f https://github.com/aws-containers/retail-store-sample-app/releases/latest/download/kubernetes.yaml
```

Steps (Helm-based):

1. If Helm chart exists in the repo or a chart repo, install it with values overrides:

```bash
helm upgrade --install retail ./charts/retail -n retail --create-namespace -f values.yaml
```

2. Monitor:

```bash
kubectl -n retail get all
```

Notes:
- Ensure you set resource requests/limits, liveness/readiness probes, and configure secrets via Kubernetes Secrets or External Secrets.
- Use imagePullSecrets if private registries are used.
- Prefer immutable image tags (digests) in production.


4) Terraform → Provision cluster + deploy (infrastructure-as-code)

- Use-case: create predictable, repeatable infrastructure (VPC, cluster, managed DBs) and then deploy app manifests or Helm charts.

Steps (example for AWS EKS module located under ./terraform/eks/default):

1. Prepare terraform backend (S3 bucket + DynamoDB table) or use a remote backend supported by your organization.
2. Initialize and plan:

```bash
cd terraform/eks/default
terraform init
terraform plan -out plan.tf
```

3. Review plan and apply:

```bash
terraform apply plan.tf
```

4. After cluster is ready, configure kubectl (often outputs kubeconfig) and deploy the app via kubectl or Helm.

Notes:
- Create separate Terraform workspaces for environments (dev/stage/prod) and restrict who can apply to production.
- Use secure storage (KMS) for sensitive state data and enable state locking.


5) Managed container platforms (ECS/Fargate, App Runner)

- Use-case: run containerized services without managing Kubernetes. Common when you want simpler operational surface.

Steps (ECS + ECR - high level):

1. Build and push images to ECR (or other registry):

```bash
docker build -t retail-ui:latest ./src/ui
aws ecr create-repository --repository-name retail-ui || true
aws ecr get-login-password | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
docker tag retail-ui:latest <repo-uri>:latest
docker push <repo-uri>:latest
```

2. Create ECS task definitions and services (via CloudFormation, Terraform or console) referencing the pushed image URIs.
3. Configure load balancer target group and security groups to expose the UI.

Notes:
- Use IAM roles for tasks and least-privilege policies.
- App Runner can deploy from a container registry or GitHub; follow the provider console/terraform steps to connect and deploy.


Common configuration and secrets

- Database endpoints (MySQL/RDS) and passwords: set via environment variables or secrets in the target platform.
- For services that support multiple persistence providers, configure provider via env var (for example, set `RETAIL_CATALOG_PERSISTENCE_PROVIDER=mysql` and supply connection details).
- Observability: set OTLP endpoint as an env var if you run a collector (EX: OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317).


Cleanup commands summary

- Docker compose: `docker compose -f docker-compose.yaml down --volumes`
- Kubernetes: `kubectl delete -f <manifest>` or `helm uninstall <release>`
- Terraform: `terraform destroy` (run only in non-production or after backup)

---

## Where to find things in this repo

- Services: `src/` (ui, catalog, cart, orders, checkout)
- Docker Compose examples: root or `deploy/compose` (if present)
- Kubernetes manifests: `k8s/` or use release manifests linked from README
- Terraform examples: `terraform/`

---

If you want, I will now replace the current README on the repository with this simplified, step-by-step version. Confirm and I will push the change.