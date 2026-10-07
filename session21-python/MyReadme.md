# Session 21: Final DevOps Project & Troubleshooting

**Name:** Shaikh Suja Rahaman
**Enrollment Number:** 24bcs10038

---

## Assignment

> Build a complete end-to-end DevOps project using the concepts learned throughout the course.
> **Flow:** Application → Git → GitHub → CI Pipeline → Build & Test → Security Scanning → Docker Image → Container Registry → Kubernetes → Helm → Monitoring → GitOps
>
> - **Infrastructure:** Terraform provisions the required cloud infrastructure.
> - **Kubernetes:** Deployment, Service, ConfigMap, Secret, Ingress, HPA, Probes, Storage.
> - **CI/CD:** GitHub Actions – build, test, Docker build, image push, Kubernetes deployment.
> - **DevSecOps:** SAST, SCA, secret scanning, container image scanning, security gates.
> - **Monitoring & GitOps:** monitoring, logs/metrics, GitOps workflow.
> - **Final Troubleshooting Challenge:** intentionally introduce several issues, then identify, investigate (logs and resources), find the root cause, fix, verify and document each one.

The project is **TaskBoard**, a small project-management SaaS app (React + FastAPI + PostgreSQL). It is built, scanned, packaged and delivered through the whole pipeline below. Everything was done on **Mon 5 Oct 2026**.

---

## Contents

1. [Project overview](#1-project-overview)
2. [Deliverable layout (where each required folder lives)](#2-deliverable-layout)
3. [Architecture diagram](#3-architecture-diagram)
4. [Technologies used](#4-technologies-used)
5. [Application setup](#5-application-setup)
6. [Docker setup](#6-docker-setup)
7. [Terraform infrastructure](#7-terraform-infrastructure)
8. [CI/CD pipeline](#8-cicd-pipeline)
9. [DevSecOps implementation](#9-devsecops-implementation)
10. [Kubernetes deployment](#10-kubernetes-deployment)
11. [Helm deployment](#11-helm-deployment)
12. [Autoscaling (HPA) under load](#12-autoscaling-hpa-under-load)
13. [Monitoring (metrics and logs)](#13-monitoring-metrics-and-logs)
14. [GitOps with Argo CD](#14-gitops-with-argo-cd)
15. [Troubleshooting challenge](#15-troubleshooting-challenge)
16. [Terraform destroy](#16-terraform-destroy)
17. [Screenshots](#17-screenshots)
18. [Lessons learned](#18-lessons-learned)

---

## 1. Project overview

| Item | Value |
|---|---|
| Application | **TaskBoard**: dashboard with KPI cards, task table, filters, create-task modal |
| Frontend | React + Vite, served by Nginx (multi-stage image), proxies `/api` to the backend |
| Backend | FastAPI + SQLAlchemy + Alembic, endpoints `/`, `/health`, `/ready`, `/metrics`, `/api/tasks` (GET/POST/PUT/DELETE), `/api/tasks/stats` |
| Database | PostgreSQL 16 (Docker Compose volume locally, Deployment + 5Gi PVC in Kubernetes) |
| Repository | `github.com/Shaikh-Suja-Rahaman/devops-heros` → folder `session21-python/` |
| Images | `ghcr.io/shaikh-suja-rahaman/taskboard-backend` and `ghcr.io/shaikh-suja-rahaman/taskboard-frontend`, tagged with the **full commit SHA** |
| Cloud infrastructure | AWS `ap-south-1`: VPC `taskboard-vpc` + EKS `taskboard-eks` (Terraform, 63 resources) |
| Runtime cluster | minikube (`192.168.49.2`), addons `ingress` + `metrics-server`, namespace `taskboard` |
| Delivery | Helm chart `taskboard` deployed through Argo CD (GitOps) |
| Observability | kube-prometheus-stack (Prometheus + Grafana) with a ServiceMonitor and a `TaskBoard API` dashboard |

**Day plan (IST):** 09:35 Terraform → 10:11 pipeline run #11 (blocked by the SCA gate) → 10:23 dependency fix + local scans → 10:28 Docker Compose → 10:32 push, pipeline #12 green → 10:58 monitoring stack → 11:08 Helm install → 11:24 HPA load test → 12:24 Argo CD → 12:41 GitOps change → 14:02–14:58 troubleshooting challenge → 16:10 `terraform destroy`.

---

## 2. Deliverable layout

The assignment asks for a `final-devops-project/` folder. This project already lives in `session21-python/`, so instead of moving files the required folders map like this:

| Required folder | Location in this repo | Contents |
|---|---|---|
| `application/` | `backend/`, `frontend/` | FastAPI app, Alembic migration, pytest tests; React/Vite UI |
| `docker/` | `backend/Dockerfile`, `frontend/Dockerfile`, `frontend/nginx.conf`, `docker-compose.yml` | Images and local full stack |
| `kubernetes/` | `k8s/namespace.yaml`, `helm/taskboard/templates/*`, `troubleshooting/` | Namespace, rendered workload manifests, broken and fixed scenarios |
| `helm/` | `helm/taskboard/` | Chart, `values.yaml`, `values-dev.yaml`, `values-prod.yaml` |
| `terraform/` | `terraform/` | VPC + EKS modules, variables, outputs, `terraform.tfvars.example` |
| `.github/workflows/` | `.github/workflows/ci-cd.yml` | DevSecOps + build + GitOps promotion pipeline |
| `security/` | `security/` | `.gitleaks.toml`, `bandit.yaml`, `trivy.yaml`, `.trivyignore` |
| `monitoring/` | `monitoring/` | `prometheus-values.yaml`, `taskboard-dashboard.yaml` (Grafana dashboard ConfigMap) |
| `gitops/` | `gitops/` | `argocd-application.yaml`, `values-minikube.yaml` (environment state owned by Git) |
| `README.md` | `README.md` (course guide) + **this `MyReadme.md`** | Documentation |

**What I added or changed to complete the project:**

| Area | Change |
|---|---|
| Helm | `configmap.yaml` (ConfigMap `taskboard-config`). Backend reads `DATABASE_URL` from the Secret (`database-url` key) and `APP_NAME` from the ConfigMap. Standard `fullname` helper (release `taskboard` gives `taskboard-backend`). Ingress now targets the real backend Service and port **8000** (it pointed at a non-existent `taskboard-backend:8080`). Added a `/health` path. |
| Frontend | `nginx.conf` became a template with `${BACKEND_URL}`, so the same image works in Compose (`http://backend:8000`) and Kubernetes (`http://taskboard-backend:8000` from the ConfigMap). |
| Backend | `tests/conftest.py` with a SQLite test DB and a client fixture that runs the startup event. Tests grew from 3 to 9. Dependency pins were bumped after the SCA gate failed (see §9). |
| Compose | Postgres healthcheck plus `depends_on: condition: service_healthy`, so Alembic no longer races the database. |
| Terraform | Rewrote the single-line blocks as valid multi-line HCL. Added default tags, subnet role tags, more outputs (`configure_kubectl`, subnets, version) and `terraform.tfvars.example`. |
| CI/CD | Workflow made monorepo-aware (`session21-python/**` paths, ignores `gitops/**`). Added Gitleaks, Bandit, pip-audit + Trivy fs jobs as gates, a lowercase GHCR owner and a GitOps promotion job. |
| New folders | `security/`, `gitops/`, `monitoring/taskboard-dashboard.yaml`, `troubleshooting/broken-db-secret.yaml`, `broken-readiness.yaml`, `broken-ingress.yaml`, `troubleshooting/fixes/*` |
| Scripts | `scripts/load-test.sh` now targets `/api/tasks` (the old `/api/health` path does not exist) and runs requests concurrently. |

---

## 3. Architecture diagram

![Architecture](./screenshots/architecture.png)

```mermaid
flowchart LR
    dev[Developer] -->|git push| gh[(GitHub<br/>devops-heros)]
    gh --> ci

    subgraph ci[GitHub Actions - ci-cd.yml]
        direction LR
        gl[Secret scan<br/>Gitleaks] --> bsp
        sast[SAST<br/>Bandit] --> bsp
        sca[SCA<br/>pip-audit + Trivy fs] --> bsp
        test[Test & build<br/>pytest + vite] --> bsp
        bsp[Build, Trivy scan<br/>& push images] --> promote[Promote image tag<br/>GitOps commit]
    end

    bsp -->|docker push :SHA| ghcr[(GHCR)]
    promote -->|commit values-minikube.yaml| gh

    tf[Terraform] --> aws[AWS ap-south-1<br/>VPC + EKS taskboard-eks]

    subgraph mk[minikube cluster]
        argo[Argo CD<br/>app: taskboard] -->|helm render + sync| tbns
        subgraph tbns[namespace taskboard]
            ing[Ingress taskboard.local] --> fe[frontend x2<br/>nginx + React]
            ing --> be[backend 2-6<br/>FastAPI]
            fe --> be
            be --> pg[(postgres + PVC 5Gi)]
            hpa[HPA cpu 60%] --> be
            cm[ConfigMap / Secret] -.-> be
        end
        subgraph mon[namespace monitoring]
            prom[Prometheus] --> graf[Grafana]
        end
        be -.->|/metrics via ServiceMonitor| prom
    end

    gh -->|pull desired state| argo
    ghcr -->|image pull| tbns
    user[Browser / curl] --> ing
```

---

## 4. Technologies used

| Layer | Tools (versions used) |
|---|---|
| Application | Python 3.12, FastAPI 0.142.2 (Starlette 1.7), SQLAlchemy 2.0, Alembic 1.14, psycopg 3, React + Vite, Nginx 1.27 |
| Testing | pytest 9.1.1, FastAPI TestClient, SQLite test database |
| Containers | Docker 29.3.1, Docker Compose v2, multi-stage frontend build, non-root backend user (uid 10001) |
| Source control | Git, GitHub, GitHub CLI |
| CI/CD | GitHub Actions (`ci-cd.yml`), GHCR |
| DevSecOps | Gitleaks (secrets), Bandit (SAST), pip-audit + Trivy fs (SCA), Trivy image (container scanning) |
| IaC | Terraform v1.16.4, `terraform-aws-modules/vpc` 5.8.1, `terraform-aws-modules/eks` 20.37.1, AWS provider 5.100 |
| Kubernetes | minikube v1.39.0, kubectl v1.34.1, ingress-nginx, metrics-server, EKS 1.31 |
| Packaging | Helm v4.3.0 (chart `taskboard` 1.0.0) |
| GitOps | Argo CD v3.5.2 |
| Monitoring | kube-prometheus-stack (Prometheus 3, Grafana), prometheus-fastapi-instrumentator 8.1.0 |

---

## 5. Application setup

**Backend endpoints** (`backend/app/main.py`):

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Liveness: `{"status":"UP"}` |
| GET | `/ready` | Readiness: runs a DB query, `{"status":"READY"}` |
| GET | `/metrics` | Prometheus metrics (`http_requests_total`, `http_request_duration_seconds`) |
| GET/POST | `/api/tasks` | List / create tasks |
| GET/PUT/DELETE | `/api/tasks/{id}` | Read / update / delete one task |
| GET | `/api/tasks/stats` | Counters for the KPI cards |

**Local run and tests:**

```bash
cd session21-python/backend
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest -v --disable-warnings
```

```text
collecting ... collected 9 items

tests/test_api.py::test_health PASSED                                    [ 11%]
tests/test_api.py::test_root PASSED                                      [ 22%]
tests/test_api.py::test_ready PASSED                                     [ 33%]
tests/test_api.py::test_create_task_validation PASSED                    [ 44%]
tests/test_api.py::test_create_task_rejects_empty_title PASSED           [ 55%]
tests/test_api.py::test_list_and_get_task PASSED                         [ 66%]
tests/test_api.py::test_update_task_and_stats PASSED                     [ 77%]
tests/test_api.py::test_delete_task PASSED                               [ 88%]
tests/test_api.py::test_metrics_endpoint PASSED                          [100%]

======================== 9 passed, 3 warnings in 0.61s =========================
```

![Dependency fix, local security checks and pytest](./screenshots/04-deps-fix-local-checks.png)

* `tests/conftest.py` sets `DATABASE_URL=sqlite:///./test.db` **before** the app is imported, so tests never touch PostgreSQL.
* The `client` fixture uses `with TestClient(app)`, which runs the startup event (`create_all`). Without it, a fresh CI runner has no `tasks` table.
* The 3 warnings are FastAPI deprecation notices (`on_event` → lifespan). They are not failures.

The UI in the browser, served through the Kubernetes Ingress:

![TaskBoard UI](./screenshots/16-taskboard-ui.png)

---

## 6. Docker setup

* `backend/Dockerfile`: `python:3.12-slim` base. Installs requirements, copies Alembic and the app, runs as **uid 10001**, and on start runs `alembic upgrade head && uvicorn`.
* `frontend/Dockerfile`: multi-stage build. `node:22-alpine` runs `npm run build`, then `nginx:1.27-alpine` serves `dist/`. `nginx.conf` is copied to `/etc/nginx/templates/` so the entrypoint fills in `${BACKEND_URL}`.
* `docker-compose.yml`: three services. Postgres has a healthcheck and the backend waits for `service_healthy`.

```bash
cd session21-python
docker compose up --build -d
docker compose ps
docker compose logs backend
curl -s localhost:8000/health; curl -s localhost:8000/ready
curl -s -X POST localhost:3000/api/tasks -H 'Content-Type: application/json' \
     -d '{"title":"Run stack with Docker Compose","priority":"HIGH","assignee":"Shaikh Suja Rahaman"}'
docker compose exec postgres psql -U taskboard -c 'select id, title, status from tasks;'
```

```text
 ✔ Container session21-python-postgres-1    Healthy
 ✔ Container session21-python-backend-1     Started
 ✔ Container session21-python-frontend-1    Started
backend-1  | INFO  [alembic.runtime.migration] Running upgrade  -> 0001_create_tasks,
backend-1  | INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
{"status":"UP"}
{"status":"READY"}
{"title":"Run stack with Docker Compose",...,"id":1,"created_at":"2026-10-05T04:58:41.312907Z"}
  1 | Run stack with Docker Compose | TODO
```

![docker compose up](./screenshots/05-docker-compose-up.png)

The POST goes to port **3000**, the Nginx frontend container, which proxies `/api/` to `http://backend:8000`. That proves the frontend-to-backend path works. The images the pipeline later pushed to GHCR were pulled back and checked:

```bash
set SHA (git rev-parse 4b7e9d2)
docker pull ghcr.io/shaikh-suja-rahaman/taskboard-backend:$SHA
docker run --rm --entrypoint id ghcr.io/shaikh-suja-rahaman/taskboard-backend:$SHA
```

```text
Digest: sha256:8e3b1f0c6d2a49f7b5c1e8d03a6f92b4c7e15d80f3a2b96c4d7e1f08a5b3c6d9
uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)
```

![GHCR pull and verification](./screenshots/10-ghcr-pull-verify.png)

---

## 7. Terraform infrastructure

`terraform/` builds the AWS side with two registry modules:

* **VPC** `taskboard-vpc`: `10.20.0.0/16`, AZs `ap-south-1a/b`, private subnets `10.20.1.0/24` and `10.20.2.0/24`, public subnets `10.20.101.0/24` and `10.20.102.0/24`, a single NAT gateway, and ELB role tags on the subnets.
* **EKS** `taskboard-eks`: Kubernetes 1.31 on the private subnets with a public endpoint and cluster-creator admin access. Managed node group `main` runs 2 × `t3.medium` (min 2, max 4).

```bash
cd session21-python/terraform
cp terraform.tfvars.example terraform.tfvars
terraform init
terraform fmt -check && terraform validate
terraform plan -out=tfplan
terraform apply tfplan
aws eks update-kubeconfig --region ap-south-1 --name taskboard-eks
kubectl get nodes -o wide
```

```text
Plan: 63 to add, 0 to change, 0 to destroy.
...
Apply complete! Resources: 63 added, 0 changed, 0 destroyed.

cluster_endpoint = "https://7C3E9A1F4B2D8E6A0F5C1B3D9E7A2F4C.gr7.ap-south-1.eks.amazonaws.com"
cluster_name = "taskboard-eks"
cluster_version = "1.31"
configure_kubectl = "aws eks update-kubeconfig --region ap-south-1 --name taskboard-eks"
vpc_id = "vpc-0d3a8f61c5b7e2940"

NAME                                         STATUS   ROLES    AGE     VERSION
ip-10-20-1-87.ap-south-1.compute.internal    Ready    <none>   2m11s   v1.31.7-eks-473151a
ip-10-20-2-143.ap-south-1.compute.internal   Ready    <none>   2m9s    v1.31.7-eks-473151a
```

![terraform init / validate / plan](./screenshots/01-terraform-init-plan.png)

![terraform apply + EKS nodes](./screenshots/02-terraform-apply-eks.png)

* The EKS control plane took about 9.5 minutes and the node group about 2 minutes. Both nodes sit in the private subnets (`10.20.1.x` and `10.20.2.x`) and reach the internet through the NAT gateway.
* An EKS cluster plus a NAT gateway costs money every hour. The infrastructure was verified and then destroyed at the end of the day (§16). The application itself ran on minikube, which Argo CD can reach without any cloud cost.

---

## 8. CI/CD pipeline

`.github/workflows/ci-cd.yml` runs on every push or PR to `main` that touches `session21-python/**`, except changes that only touch `gitops/**`:

| Job | Needs | What it does |
|---|---|---|
| Secret scan (Gitleaks) | – | Full-history scan with `security/.gitleaks.toml` |
| SAST (Bandit) | – | `bandit -c security/bandit.yaml -r backend/app`, fails on medium/high |
| SCA (pip-audit + Trivy fs) | – | `pip-audit --strict` on `requirements.txt` + Trivy filesystem scan (HIGH/CRITICAL, exit 1) |
| Test & build | – | `pytest -v`, then `npm install && npm run build` |
| Build, scan & push images | all four above | Builds both images tagged with `github.sha`, runs a Trivy image scan on each (gate), then pushes to GHCR |
| Promote image tag (GitOps) | build | `yq` writes the SHA into `gitops/values-minikube.yaml`, then commits `chore(gitops): promote … [skip ci]` |

There is no `kubectl` in the pipeline. The runner never needs cluster credentials, because Argo CD pulls the change (§14).

```bash
git add backend/requirements.txt
git commit -m "fix(deps): bump fastapi and instrumentator to clear pip-audit findings"
git push origin main
gh run watch 18240517733 --exit-status
git pull --quiet && git log --oneline -4
```

```text
   d61a0f3..4b7e9d2  main -> main
✓ Secret scan (Gitleaks) in 14s
✓ SAST (Bandit) in 21s
✓ SCA (pip-audit + Trivy fs) in 58s
✓ Test & build in 49s
✓ Build, scan & push images in 4m31s
✓ Promote image tag (GitOps) in 9s
9c1f5ab (HEAD -> main, origin/main) chore(gitops): promote taskboard images to 4b7e9d2 [skip ci]
4b7e9d2 fix(deps): bump fastapi and instrumentator to clear pip-audit findings
```

![git push and gh run watch](./screenshots/06-git-commit-push.png)

![Pipeline run #12 – all jobs green](./screenshots/07-actions-run-success.png)

![Push images step – digests in GHCR](./screenshots/09-actions-image-push.png)

* The four gate jobs run in parallel, so a slow scanner does not hold up the tests.
* Image tag = full commit SHA (`4b7e9d2c81f3a6e05d9b72c4e1a8f3065d2c7b19`). Every running Pod can be traced back to one commit.
* The owner is lowercased (`${GITHUB_REPOSITORY_OWNER,,}`) because GHCR rejects `Shaikh-Suja-Rahaman` in an image reference.

---

## 9. DevSecOps implementation

| Control | Tool | Config | Gate |
|---|---|---|---|
| Secret scanning | Gitleaks | `security/.gitleaks.toml`: default rules plus a custom `postgres-url-with-password` rule; demo credentials allow-listed by path | Job fails on any leak |
| SAST | Bandit | `security/bandit.yaml`: tests and migrations excluded | `--severity-level medium` |
| SCA | pip-audit, Trivy fs | `security/trivy.yaml`: vuln + secret scanners, HIGH/CRITICAL, `ignore-unfixed`, exit code 1 | Job fails |
| Container image scanning | Trivy image | same `security/trivy.yaml` | Build job fails before **Push images** |
| Accepted risks | `.trivyignore` | empty, so nothing is waived | – |

**The security gate in action.** Run #11 (`d61a0f3`, the commit that added these jobs) failed in the **SCA** job. The build and promote jobs were skipped, so no vulnerable image was ever pushed:

```text
Found 8 known vulnerabilities in 2 packages
Name      Version ID              Fix Versions
pytest    8.3.4   PYSEC-2026-1845 9.0.3
starlette 0.41.3  PYSEC-2026-1941 0.47.2
starlette 0.41.3  PYSEC-2026-1942 0.49.1
...
starlette 0.41.3  PYSEC-2026-249  1.3.1
Error: Process completed with exit code 1.
```

![Run #11 blocked by the SCA gate](./screenshots/03-actions-sca-gate-failed.png)

* **Root cause:** `fastapi==0.115.6` pins Starlette `<0.42`, and `prometheus-fastapi-instrumentator==7.0.2` pins Starlette `<1.0`. The fixed Starlette versions could not be installed.
* **Fix:** `fastapi==0.142.2`, `prometheus-fastapi-instrumentator==8.1.0` and `pytest==9.1.1`. Starlette resolves to 1.7.0. Before pushing, I confirmed the fix locally: pip-audit clean, Bandit 0 issues, Gitleaks no leaks, pytest 9/9 (screenshot in §5).

Run #12 then scanned the built images. Both came back **clean** (0 HIGH/CRITICAL fixable vulnerabilities, no secrets):

![Trivy image scan in the pipeline](./screenshots/08-actions-trivy-scan.png)

**What the Trivy result means:** Trivy unpacked the image layers. It checked the Debian 13.1 OS packages (88 packages) and the Python site-packages against vulnerability databases, and looked for secrets baked into the layers. A clean report under `ignore-unfixed` means no HIGH or CRITICAL CVE with an available fix exists in either layer type. Unfixed CVEs are still worth reviewing, but they cannot be resolved by upgrading, so they do not block the build.

---

## 10. Kubernetes deployment

All resources come from the Helm chart (rendered with `values-dev.yaml` + `gitops/values-minikube.yaml`):

| Requirement | Resource |
|---|---|
| Deployment | `taskboard-backend` (2 to 6 replicas), `taskboard-frontend` (2), `taskboard-postgres` (1) |
| Service | `taskboard-backend` :8000, `taskboard-frontend` :80, `taskboard-postgres` :5432 (ClusterIP) |
| ConfigMap | `taskboard-config`: `APP_NAME`, `BACKEND_URL`, `DB_HOST`, `DB_PORT`, `DB_NAME` |
| Secret | `taskboard-postgres`: `username`, `password`, `database-url` |
| Ingress | `taskboard` (class nginx, host `taskboard.local`): `/api` and `/health` go to the backend, `/` goes to the frontend |
| HPA | `taskboard-backend`: CPU 60%, min 2 (later 3), max 6 |
| Probes | backend liveness `/health`, readiness `/ready` (checks the DB); frontend `GET /`; postgres `pg_isready` |
| Storage | PVC `taskboard-postgres-data` 5Gi RWO (`standard` StorageClass) mounted at `/var/lib/postgresql/data` |
| Monitoring hook | ServiceMonitor `taskboard-backend` (port `http`, `/metrics`, 15s) |

```bash
kubectl get all -n taskboard
kubectl get ingress,pvc,configmap,secret,servicemonitor -n taskboard
kubectl get endpointslices -n taskboard
kubectl describe deploy/taskboard-backend -n taskboard | sed -n '/Containers:/,/Mounts:/p'
```

```text
pod/taskboard-backend-7d9f6c8b5d-4xk2p    1/1     Running   0               2m31s
pod/taskboard-backend-7d9f6c8b5d-r8m7w    1/1     Running   1 (2m20s ago)   2m31s
pod/taskboard-frontend-5c8d7b9f4d-hq2zn   1/1     Running   0               2m31s
pod/taskboard-frontend-5c8d7b9f4d-w6tcx   1/1     Running   0               2m31s
pod/taskboard-postgres-6b4f9d7c88-m2v9s   1/1     Running   0               2m31s
horizontalpodautoscaler.autoscaling/taskboard-backend   Deployment/taskboard-backend   cpu: 3%/60%   2   6   2
ingress.networking.k8s.io/taskboard   nginx   taskboard.local   192.168.49.2   80
persistentvolumeclaim/taskboard-postgres-data   Bound   pvc-3f6a2c1e-8b4d-4e57-9a0c-2d7e5b1f9c43   5Gi   RWO   standard
    Liveness:   http-get http://:8000/health delay=20s timeout=1s period=15s #success=1 #failure=3
    Readiness:  http-get http://:8000/ready delay=10s timeout=1s period=10s #success=1 #failure=3
      DATABASE_URL:  <set to the key 'database-url' in secret 'taskboard-postgres'>  Optional: false
```

![kubectl get all](./screenshots/13-kubectl-get-all.png)

![Ingress, PVC, ConfigMap, Secret, probes](./screenshots/14-kubectl-config-storage.png)

* The single restart on `r8m7w` came from the first-start race: the backend ran Alembic before Postgres accepted connections (`Connection refused` in `--previous` logs). The restart policy plus the readiness probe handled it, and no traffic reached the Pod until `/ready` passed.

**Access through the Ingress.** `minikube tunnel` was running and `/etc/hosts` maps `taskboard.local` to `127.0.0.1`.

```bash
curl -si http://taskboard.local/health
curl -s -X POST http://taskboard.local/api/tasks -H 'Content-Type: application/json' -d '{"title":"Move Postgres to Amazon RDS",...}'
curl -s http://taskboard.local/api/tasks | jq -r '.[] | [.id, .priority, .status, .assignee, .title] | @tsv'
curl -s http://taskboard.local/api/tasks/stats
kubectl port-forward -n taskboard svc/taskboard-backend 8000:8000 &
curl -s localhost:8000/metrics | grep '^http_requests_total'
```

```text
HTTP/1.1 200 OK
{"status":"UP"}
{"total":6,"todo":2,"inProgress":2,"done":2}
http_requests_total{handler="/health",method="GET",status="2xx"} 24.0
http_requests_total{handler="/ready",method="GET",status="2xx"} 37.0
http_requests_total{handler="/api/tasks",method="POST",status="2xx"} 3.0
```

![Ingress, API and /metrics](./screenshots/15-ingress-api-metrics.png)

---

## 11. Helm deployment

The monitoring stack is installed first because the chart contains a `ServiceMonitor`, and that CRD comes from kube-prometheus-stack.

```bash
kubectl config use-context minikube
minikube addons enable ingress
minikube addons enable metrics-server
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace -f monitoring/prometheus-values.yaml
kubectl apply -f monitoring/taskboard-dashboard.yaml

kubectl apply -f k8s/namespace.yaml
helm lint helm/taskboard -f helm/taskboard/values-dev.yaml -f gitops/values-minikube.yaml
helm upgrade --install taskboard ./helm/taskboard -n taskboard \
  -f helm/taskboard/values-dev.yaml -f gitops/values-minikube.yaml
helm list -n taskboard
```

```text
1 chart(s) linted, 0 chart(s) failed
Release "taskboard" does not exist. Installing it now.
NAME: taskboard
LAST DEPLOYED: Mon Oct  5 11:08:42 2026
STATUS: deployed
REVISION: 1
taskboard   taskboard   1   2026-10-05 11:08:42.581734912 +0530 IST   deployed   taskboard-1.0.0   1.0.0
```

![minikube addons + kube-prometheus-stack](./screenshots/11-minikube-monitoring-setup.png)

![helm lint / template / install](./screenshots/12-helm-install-taskboard.png)

* Values are layered: `values.yaml` (defaults) → `values-dev.yaml` (ingress on) → `gitops/values-minikube.yaml` (real image names, SHA tags, replicas, HPA). The later file wins.
* `helm template … | grep '^kind:'` confirmed the chart renders all the required objects before anything touched the cluster.

---

## 12. Autoscaling (HPA) under load

```bash
env REQUESTS=30000 CONCURRENCY=50 ./scripts/load-test.sh &
kubectl get hpa taskboard-backend -n taskboard -w
kubectl top pods -n taskboard -l app=taskboard-backend
kubectl describe hpa taskboard-backend -n taskboard | tail -n 4
```

```text
taskboard-backend   Deployment/taskboard-backend   cpu: 3%/60%      2   6   2   16m
taskboard-backend   Deployment/taskboard-backend   cpu: 97%/60%     2   6   2   17m
taskboard-backend   Deployment/taskboard-backend   cpu: 168%/60%    2   6   4   17m
taskboard-backend   Deployment/taskboard-backend   cpu: 89%/60%     2   6   6   18m
taskboard-backend   Deployment/taskboard-backend   cpu: 57%/60%     2   6   6   19m
  Normal  SuccessfulRescale  2m41s  horizontal-pod-autoscaler  New size: 4; reason: cpu resource utilization (percentage of request) above target
  Normal  SuccessfulRescale  2m11s  horizontal-pod-autoscaler  New size: 6; reason: cpu resource utilization (percentage of request) above target
  HTTP 200: 30000 requests
```

![HPA scaling 2 → 4 → 6 → 2](./screenshots/17-hpa-load-test.png)

* Utilisation is measured against the **request** (100m). 168% means the pods averaged about 168m each, so the HPA asked for `ceil(2 × 168/60) = 6` replicas, capped by its scale-up rate (4 first, then 6).
* Every one of the 30,000 requests returned 200. After the load stopped, the 5-minute scale-down stabilisation window passed and the HPA returned to 2 replicas (`cpu: 4%/60%`).

---

## 13. Monitoring (metrics and logs)

* **Metrics:** prometheus-fastapi-instrumentator exposes `/metrics`. The chart's `ServiceMonitor` (port `http`, 15s) is discovered because `prometheus-values.yaml` sets `serviceMonitorSelectorNilUsesHelmValues: false`.
* **Dashboard:** `monitoring/taskboard-dashboard.yaml` is a ConfigMap labelled `grafana_dashboard: "1"`. The Grafana sidecar loads it as **TaskBoard API**. Panels: request rate, p95 latency, 5xx error %, backend pods, requests per handler, CPU per pod, p50/p95 latency, HPA replicas.
* **Logs:** Uvicorn access logs and Alembic logs through `kubectl logs`, plus Postgres server logs. These were the main evidence in the troubleshooting challenge.

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

![Prometheus targets – taskboard backend 2/2 up](./screenshots/18-prometheus-targets.png)

![Grafana – TaskBoard API dashboard during the load test](./screenshots/19-grafana-dashboard.png)

The dashboard shows the load test from §12 clearly. `/api/tasks` peaked at about 142 req/s, p95 latency rose from about 12 ms to about 170 ms, CPU spread from 2 pods to 6, and the HPA panel steps 2 → 4 → 6 → 2. The 5xx rate stayed at 0.00%.

---

## 14. GitOps with Argo CD

`gitops/argocd-application.yaml` points Argo CD at `https://github.com/Shaikh-Suja-Rahaman/devops-heros.git`, path `session21-python/helm/taskboard`, value files `values-dev.yaml` + `../../gitops/values-minikube.yaml`, release name `taskboard`, with automated sync, `prune` and `selfHeal`. Argo CD (already running in the cluster since Session 20) adopted the Helm-installed resources without restarting any Pod, because the rendered manifests were identical.

```bash
kubectl apply -f gitops/argocd-application.yaml
argocd app wait taskboard --sync --health --timeout 180 > /dev/null; and argocd app get taskboard
argocd app resources taskboard
```

```text
Sync Policy:        Automated (Prune)
Sync Status:        Synced to main (9c1f5ab)
Health Status:      Healthy
```

![Argo CD application synced](./screenshots/20-argocd-application.png)

**GitOps change flow.** No `kubectl` or `helm` was used for this change. Git is the only interface:

```bash
sed -i '' 's/minReplicas: 2/minReplicas: 3/' gitops/values-minikube.yaml
git commit -qam "gitops: raise taskboard backend minReplicas to 3"; and git push -q origin main
gh run list --workflow ci-cd.yml --limit 1      # no new run: gitops/** is ignored by CI
argocd app history taskboard
kubectl get hpa -n taskboard; kubectl get pods -n taskboard -l app=taskboard-backend
```

```text
ID  DATE                           REVISION
0   2026-10-05 12:24:51 +0530 IST  main (9c1f5ab)
1   2026-10-05 12:41:37 +0530 IST  main (a71d3e4)
taskboard-backend   Deployment/taskboard-backend   cpu: 3%/60%   3   6   3   94m
taskboard-backend-7d9f6c8b5d-z7n4c   1/1   Running   0   41s
```

![GitOps change: commit → Argo CD sync → 3rd replica](./screenshots/21-gitops-change.png)

![Argo CD resource tree at a71d3e4](./screenshots/22-argocd-app-tree.png)

There are two GitOps loops in this project:

1. **CI → Git:** the pipeline's `Promote image tag` job commits the new SHA (`9c1f5ab`).
2. **Human → Git:** I edit environment config (`a71d3e4`).

Either way Argo CD notices the new commit and reconciles the cluster, and `argocd app history` gives a full audit trail with one-click rollback.

---

## 15. Troubleshooting challenge

Six faults were introduced on purpose. Manifests are in `troubleshooting/` and the corrected versions in `troubleshooting/fixes/`. Each broken workload used its own name and labels (not managed by Argo CD), so the live app kept serving traffic during the exercise. Issue 6 was a cluster-level fault.

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | `ImagePullBackOff` | Image tag `does-not-exist` under the wrong registry owner | Use the CI-pushed SHA tag |
| 2 | Service reachable by name but connections refused, no endpoints | Selector matches no Pod, then `targetPort` 8080 ≠ container 8000 | Correct selector and `targetPort: 8000` |
| 3 | `CrashLoopBackOff` | Secret `DATABASE_URL` had the password `taskb0ard` | Correct the Secret, then `rollout restart` |
| 4 | Pod `Running` but `0/1` Ready | Readiness probe path `/readyz` returns 404 | Probe `/ready` |
| 5 | Ingress returns **503** | Ingress backend port 8080, Service only exposes 8000 | Backend port 8000 (same bug existed in the original chart, fixed there too) |
| 6 | HPA `cpu: <unknown>/60%` | metrics-server removed, so the Metrics API is unavailable | Re-enable metrics-server |

### Issue 1: ImagePullBackOff

**Problem:** `kubectl apply -f troubleshooting/broken-image.yaml` leaves the Pod in `ImagePullBackOff`.

**Investigation:**

```bash
kubectl get pods -n taskboard -l app=broken-image
kubectl describe pod -n taskboard -l app=broken-image | sed -n '/^Events:/,$p'
kubectl get events -n taskboard --field-selector involvedObject.kind=Pod,type=Warning --sort-by=.lastTimestamp
kubectl get deploy taskboard-broken-image -n taskboard -o jsonpath='{.spec.template.spec.containers[0].image}'
docker manifest inspect ghcr.io/example/taskboard-backend:does-not-exist
```

```text
taskboard-broken-image-6f9c8d7b5c-q8x2v   0/1     ImagePullBackOff   0          41s
Warning  Failed  16s (x3 over 43s)  kubelet  Failed to pull image "ghcr.io/example/taskboard-backend:does-not-exist": Error response from daemon: manifest unknown
manifest unknown
```

![Issue 1 – broken](./screenshots/23-t1-imagepull-broken.png)

**Root cause:** the kubelet cannot pull `ghcr.io/example/taskboard-backend:does-not-exist`, because that tag (and owner) does not exist in the registry. The scheduler and node were fine. The failure is purely at the image-pull stage, which is why the container never started and there are no container logs.

**Fix and verification:**

```bash
diff -I '^#' troubleshooting/broken-image.yaml troubleshooting/fixes/image-fixed.yaml
kubectl apply -f troubleshooting/fixes/image-fixed.yaml
kubectl rollout status deploy/taskboard-broken-image -n taskboard
kubectl logs -n taskboard deploy/taskboard-broken-image --tail 3
```

```text
<         image: ghcr.io/example/taskboard-backend:does-not-exist
>         image: ghcr.io/shaikh-suja-rahaman/taskboard-backend:4b7e9d2c81f3a6e05d9b72c4e1a8f3065d2c7b19
deployment "taskboard-broken-image" successfully rolled out
taskboard-broken-image-79d4b6c8f5-tj6mw   1/1     Running   0          23s
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

![Issue 1 – fixed](./screenshots/24-t1-imagepull-fixed.png)

### Issue 2: Service with no endpoints (and a wrong targetPort)

**Problem:** `broken-service` exists and resolves, but connections to it fail.

**Investigation:**

```bash
kubectl get svc broken-service -n taskboard -o wide
kubectl get endpointslices -n taskboard -l kubernetes.io/service-name=broken-service
kubectl run curl-test -n taskboard --rm -i --restart=Never --image=curlimages/curl:8.16.0 -- curl -sS -m 5 http://broken-service:8080/health
kubectl get pods -n taskboard -l app=taskboard-backend --show-labels
```

```text
broken-service   ClusterIP   10.101.88.164   <none>   8080/TCP   14s   app=label-that-does-not-exist
broken-service-4hq8n   IPv4   <unset>   <unset>   15s
curl: (7) Failed to connect to broken-service port 8080 after 2 ms: Could not connect to server
```

![Issue 2 – broken](./screenshots/25-t2-service-broken.png)

**Root cause:** the selector `app=label-that-does-not-exist` matches no Pod, so the EndpointSlice is empty and kube-proxy rejects the connection. After patching the selector, the endpoints appeared but still on port **8080**, and curl still failed. That revealed a second fault: `targetPort` must be the container port **8000**.

**Fix and verification:**

```bash
kubectl patch svc broken-service -n taskboard -p '{"spec":{"selector":{"app":"taskboard-backend"}}}'
kubectl apply -f troubleshooting/fixes/service-fixed.yaml
kubectl get endpointslices -n taskboard -l kubernetes.io/service-name=broken-service
kubectl run curl-test ... -- curl -sS -m 5 http://broken-service:8080/health
```

```text
broken-service-4hq8n   IPv4   8000   10.244.0.21,10.244.0.22,10.244.0.31   3m5s
{"status":"UP"}
```

![Issue 2 – fixed](./screenshots/26-t2-service-fixed.png)

### Issue 3: CrashLoopBackOff from a bad DB Secret

**Problem:** `taskboard-backend-v2` restarts again and again (`CrashLoopBackOff`, exit code 1).

**Investigation:**

```bash
kubectl describe pod -n taskboard -l app=taskboard-backend-v2 | grep -A4 'Last State'
kubectl logs -n taskboard deploy/taskboard-backend-v2 --previous --tail 4
kubectl logs -n taskboard deploy/taskboard-postgres --tail 2
kubectl get secret taskboard-backend-v2-db -n taskboard -o jsonpath='{.data.database-url}' | base64 -d
```

```text
taskboard-backend-v2-5b8f7c9d6d-h4k9t   0/1     CrashLoopBackOff   3 (24s ago)   88s
sqlalchemy.exc.OperationalError: (psycopg.OperationalError) connection failed: ... FATAL:  password authentication failed for user "taskboard"
2026-10-05 08:51:04.118 UTC [1873] FATAL:  password authentication failed for user "taskboard"
postgresql+psycopg://taskboard:taskb0ard@taskboard-postgres:5432/taskboard
```

![Issue 3 – broken](./screenshots/27-t3-db-secret-broken.png)

**Root cause:** the container starts with `alembic upgrade head`. Alembic connects with the `DATABASE_URL` from Secret `taskboard-backend-v2-db`, whose password is `taskb0ard` (zero instead of "o"). Postgres rejects it, the process exits with 1, and the kubelet backs off. `--previous` was essential here, because the current container had not logged anything yet.

**Fix and verification:** correct the Secret, then restart. Pods do not reload env vars from a changed Secret on their own.

```bash
kubectl apply -f troubleshooting/fixes/db-secret-fixed.yaml
kubectl rollout restart deploy/taskboard-backend-v2 -n taskboard
kubectl rollout status deploy/taskboard-backend-v2 -n taskboard
kubectl exec -n taskboard deploy/taskboard-backend-v2 -- python -c "import urllib.request as u; print(u.urlopen('http://localhost:8000/ready').read().decode())"
```

```text
secret/taskboard-backend-v2-db configured
deployment "taskboard-backend-v2" successfully rolled out
taskboard-backend-v2-6c4d9b7f8-x2rnl   1/1     Running   0          38s
{"status":"READY"}
```

![Issue 3 – fixed](./screenshots/28-t3-db-secret-fixed.png)

### Issue 4: Pod Running but never Ready (wrong readiness probe)

**Problem:** `taskboard-backend-probe` stays `0/1 Running`, and the Deployment shows `0/1` available. No crash, no restart.

**Investigation:**

```bash
kubectl describe pod -n taskboard -l app=taskboard-backend-probe | grep -E 'Readiness|Ready:|Unhealthy'
kubectl logs -n taskboard deploy/taskboard-backend-probe --tail 3
kubectl exec -n taskboard deploy/taskboard-backend-probe -- python -c "...urlopen('http://localhost:8000/ready')..."
```

```text
    Readiness:      http-get http://:8000/readyz delay=5s timeout=1s period=5s #success=1 #failure=3
    Ready:          False
  Warning  Unhealthy  2s (x15 over 70s)  kubelet  Readiness probe failed: HTTP probe failed with statuscode: 404
INFO:     10.244.0.1:49816 - "GET /readyz HTTP/1.1" 404 Not Found
{"status":"READY"}
```

![Issue 4 – broken](./screenshots/29-t4-readiness-broken.png)

**Root cause:** the application is healthy (`/ready` answers `READY` from inside the Pod), but the probe asks for `/readyz`, which FastAPI does not serve. A failing readiness probe never restarts the container. It only keeps the Pod out of Service endpoints, so the symptom is "running but no traffic".

**Fix and verification:**

```bash
kubectl apply -f troubleshooting/fixes/readiness-fixed.yaml
kubectl rollout status deploy/taskboard-backend-probe -n taskboard
```

```text
<           httpGet: {path: /readyz, port: 8000}
>           httpGet: {path: /ready, port: 8000}
deployment.apps/taskboard-backend-probe   1/1     1            1           3m41s
    Ready:          True
INFO:     10.244.0.1:52274 - "GET /ready HTTP/1.1" 200 OK
```

![Issue 4 – fixed](./screenshots/30-t4-readiness-fixed.png)

### Issue 5: Ingress returns 503 (wrong Service port)

**Problem:** `http://broken.taskboard.local/api/tasks/stats` returns `503 Service Temporarily Unavailable` from nginx.

**Investigation:**

```bash
kubectl describe ingress taskboard-broken -n taskboard | sed -n '/^Rules:/,/^Annotations:/p'
kubectl get svc taskboard-backend -n taskboard -o jsonpath='{.spec.ports}'
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --since=5m | grep -E '8080|503'
```

```text
                          /api   taskboard-backend:8080 ()
[{"name":"http","port":8000,"protocol":"TCP","targetPort":8000}]
W1005 09:12:31.204718  7 controller.go:1232] Service "taskboard/taskboard-backend" does not have any active Endpoint for TCP port 8080
... "GET /api/tasks/stats HTTP/1.1" 503 190 ... [taskboard-taskboard-backend-8080] [] - - - -
```

![Issue 5 – broken](./screenshots/31-t5-ingress-503-broken.png)

**Root cause:** the Service is fine, but the Ingress asks for port **8080** and the Service only exposes **8000**. The `()` after the backend in `describe ingress` (no endpoint list) and the controller warning show the controller has no upstream, so it answers 503 itself. The original chart's `ingress.yaml` had exactly this bug (`taskboard-backend:8080`). It is fixed in the chart, and the `helm template` check in the screenshot confirms it.

**Fix and verification:**

```bash
kubectl apply -f troubleshooting/fixes/ingress-fixed.yaml
curl -si http://broken.taskboard.local/api/tasks/stats
```

```text
                          /api   taskboard-backend:8000 (10.244.0.21:8000,10.244.0.22:8000,10.244.0.31:8000)
HTTP/1.1 200 OK
{"total":6,"todo":2,"inProgress":2,"done":2}
```

![Issue 5 – fixed](./screenshots/32-t5-ingress-503-fixed.png)

### Issue 6: HPA shows `<unknown>` targets (no metrics-server)

**Problem:** after `minikube addons disable metrics-server`, the HPA shows `cpu: <unknown>/60%` and would never scale.

**Investigation:**

```bash
kubectl get hpa -n taskboard
kubectl describe hpa taskboard-backend -n taskboard | sed -n '/^Conditions:/,$p'
kubectl top pods -n taskboard
kubectl get apiservice v1beta1.metrics.k8s.io
```

```text
taskboard-backend   Deployment/taskboard-backend   cpu: <unknown>/60%   3   6   3   3h48m
ScalingActive  False   FailedGetResourceMetric  ... unable to fetch metrics from resource metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)
error: Metrics API not available
Error from server (NotFound): apiservices.apiregistration.k8s.io "v1beta1.metrics.k8s.io" not found
```

![Issue 6 – broken](./screenshots/33-t6-hpa-unknown-broken.png)

**Root cause:** the HPA reads CPU from the `metrics.k8s.io` aggregated API. metrics-server provides that API, and it was removed. The HPA itself, its target and the resource requests were all correct (`AbleToScale True`). Only `ScalingActive` was false.

**Fix and verification:**

```bash
minikube addons enable metrics-server
kubectl rollout status deploy/metrics-server -n kube-system
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top pods -n taskboard
kubectl get hpa -n taskboard
```

```text
v1beta1.metrics.k8s.io   kube-system/metrics-server   True   52s
taskboard-backend   Deployment/taskboard-backend   cpu: 3%/60%   3   6   3   3h53m
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate a replica count from cpu resource utilization
```

![Issue 6 – fixed](./screenshots/34-t6-hpa-unknown-fixed.png)

**Investigation pattern used every time:** `get` (what state?) → `describe` / events (what does the controller say?) → `logs` / `--previous` (what does the app say?) → compare the spec with reality (selectors, ports, paths, secrets, APIs) → change one thing → verify with the same commands that showed the failure.

---

## 16. Terraform destroy

```bash
cd session21-python/terraform
terraform plan -destroy | tail -n 3
terraform destroy -auto-approve
aws eks list-clusters --region ap-south-1
aws ec2 describe-vpcs --region ap-south-1 --filters Name=tag:Name,Values=taskboard-vpc --query 'Vpcs[].VpcId'
```

```text
Plan: 0 to add, 0 to change, 63 to destroy.
Destroy complete! Resources: 63 destroyed.
{
    "clusters": []
}
[]
```

![terraform destroy](./screenshots/35-terraform-destroy.png)

---

## 17. Screenshots

| # | Screenshot | Shows |
|---|---|---|
| – | [architecture.png](./screenshots/architecture.png) | End-to-end architecture |
| 01 | [01-terraform-init-plan.png](./screenshots/01-terraform-init-plan.png) | init, fmt, validate, plan (63 to add) |
| 02 | [02-terraform-apply-eks.png](./screenshots/02-terraform-apply-eks.png) | apply, outputs, kubeconfig, EKS nodes |
| 03 | [03-actions-sca-gate-failed.png](./screenshots/03-actions-sca-gate-failed.png) | Run #11 blocked by pip-audit |
| 04 | [04-deps-fix-local-checks.png](./screenshots/04-deps-fix-local-checks.png) | Pin bump, pip-audit, Bandit, Gitleaks, pytest |
| 05 | [05-docker-compose-up.png](./screenshots/05-docker-compose-up.png) | Compose stack, logs, API, DB |
| 06 | [06-git-commit-push.png](./screenshots/06-git-commit-push.png) | Commit, push, `gh run watch`, bot commit |
| 07 | [07-actions-run-success.png](./screenshots/07-actions-run-success.png) | Run #12 job graph |
| 08 | [08-actions-trivy-scan.png](./screenshots/08-actions-trivy-scan.png) | Trivy image scan (clean) |
| 09 | [09-actions-image-push.png](./screenshots/09-actions-image-push.png) | Push to GHCR with digests |
| 10 | [10-ghcr-pull-verify.png](./screenshots/10-ghcr-pull-verify.png) | Pull from GHCR, non-root check |
| 11 | [11-minikube-monitoring-setup.png](./screenshots/11-minikube-monitoring-setup.png) | Addons + kube-prometheus-stack |
| 12 | [12-helm-install-taskboard.png](./screenshots/12-helm-install-taskboard.png) | helm lint/template/install, pods coming up |
| 13 | [13-kubectl-get-all.png](./screenshots/13-kubectl-get-all.png) | Deployments, Services, ReplicaSets, HPA |
| 14 | [14-kubectl-config-storage.png](./screenshots/14-kubectl-config-storage.png) | Ingress, PVC, ConfigMap, Secret, probes |
| 15 | [15-ingress-api-metrics.png](./screenshots/15-ingress-api-metrics.png) | curl via Ingress, `/metrics`, logs |
| 16 | [16-taskboard-ui.png](./screenshots/16-taskboard-ui.png) | React UI at `http://taskboard.local` |
| 17 | [17-hpa-load-test.png](./screenshots/17-hpa-load-test.png) | HPA 2 → 4 → 6 under load |
| 18 | [18-prometheus-targets.png](./screenshots/18-prometheus-targets.png) | Prometheus targets UP |
| 19 | [19-grafana-dashboard.png](./screenshots/19-grafana-dashboard.png) | Grafana TaskBoard API dashboard |
| 20 | [20-argocd-application.png](./screenshots/20-argocd-application.png) | Argo CD app Synced/Healthy |
| 21 | [21-gitops-change.png](./screenshots/21-gitops-change.png) | Git-only change and its sync |
| 22 | [22-argocd-app-tree.png](./screenshots/22-argocd-app-tree.png) | Argo CD resource tree |
| 23–34 | `23-t1-…` to `34-t6-…` | Troubleshooting issues 1–6 (broken / fixed) |
| 35 | [35-terraform-destroy.png](./screenshots/35-terraform-destroy.png) | Infrastructure torn down |

---

## 18. Lessons learned

1. **Security gates only matter if they can say no.** The SCA job blocked run #11 on Starlette CVEs pulled in by an old FastAPI pin. Because Build/Push depends on the gate jobs, no vulnerable image ever reached GHCR. Fixing a transitive dependency meant upgrading the packages that pin it (FastAPI and the instrumentator), not the CVE package directly.
2. **Tag images with the commit SHA, never `latest`.** Every Pod, Argo CD revision and Trivy report pointed to `4b7e9d2…`, which made tracing and rollback trivial.
3. **GitOps removes cluster credentials from CI.** The pipeline only writes to Git, and Argo CD pulls. Excluding `gitops/**` from CI triggers stopped config-only commits from rebuilding images, and `[skip ci]` stopped the bot's commit from looping.
4. **Readiness and liveness answer different questions.** `/ready` checks the database, so a backend that starts before Postgres takes no traffic. A wrong readiness path (Issue 4) never restarts anything, so it looks "fine" until you check endpoints.
5. **Names and ports have to line up across objects.** Ingress → Service port → targetPort → containerPort, and Service selector → Pod labels. Issues 2 and 5 (and the original chart's ingress bug) were all mismatches, and `describe ingress` / EndpointSlices showed them right away.
6. **`kubectl logs --previous` is the key to CrashLoopBackOff.** The current container is usually too new to have logs. The Postgres server log confirmed the client-side error from the other end.
7. **Secrets are read at start-up.** Fixing a Secret needs a `rollout restart`. That is also why Argo CD `selfHeal` would immediately undo a manual `kubectl edit` on managed objects, so the broken scenarios used separate, unmanaged names.
8. **The HPA depends on the Metrics API.** `<unknown>` targets point to metrics-server or a missing APIService, not to the HPA spec. Resource requests are the 100% baseline for the percentage.
9. **Make the same artifact work everywhere.** Templating `BACKEND_URL` into the Nginx config let one frontend image run in Compose and Kubernetes. Hard-coding `backend:8000` would have crashed Nginx in the cluster.
10. **Infrastructure costs money while you look at it.** Terraform made the 63-resource VPC + EKS reproducible in about 12 minutes, and `terraform destroy` made it just as easy to remove. The application itself ran on minikube.
