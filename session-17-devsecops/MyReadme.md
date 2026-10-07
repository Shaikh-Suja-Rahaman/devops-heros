# Session 17: Complete CI/CD & DevSecOps

**Name:** Shaikh Suja Rahaman
**Enrollment Number:** 24bcs10038

---

## Overview

**Task: DevSecOps Demo Project.** Build a complete CI/CD + DevSecOps pipeline for an application.

| Area | Required | What I used |
|------|----------|-------------|
| CI/CD | Application build, unit testing, Docker image build, container registry, Kubernetes deployment | `compileall` + pytest/pytest-cov, Docker, **GHCR**, Kubernetes (kind in CI, minikube locally) |
| Security | SAST, SCA, secret scanning, container image scanning, security gates | **CodeQL + Bandit**, **pip-audit**, **Gitleaks**, **Trivy**, a dedicated `security-gate` job |

The application is the **DevSecOps Dashboard** (`hey-cicd`), a small Flask app in [`demo/`](./demo). It has a UI at `/`, health and status endpoints, a greeting API, a calculator API and a pipeline simulator. Every push to `main` runs [`demo/.github/workflows/devsecops.yml`](./demo/.github/workflows/devsecops.yml). The image is pushed to `ghcr.io/shaikh-suja-rahaman/hey-cicd` and deployed to Kubernetes **only if every security check passes**.

> **Repository:** GitHub only runs workflows from `.github/workflows/` at the **root** of a repository. Because of that, the `demo/` folder is pushed as its own repository, [`Shaikh-Suja-Rahaman/hey-cicd`](https://github.com/Shaikh-Suja-Rahaman/hey-cicd), and the Actions runs below come from that repository. The same files are kept here under `session-17-devsecops/demo/`.

**Environment:** macOS (fish shell), Python 3.14 venv locally, Python 3.12 in CI and in the container, Docker 29.3.1, kubectl v1.34.1, minikube v1.39.0, Trivy (DB `mirror.gcr.io/aquasec/trivy-db:2`), Gitleaks 8.28.0, Bandit 1.8.6, pip-audit 2.9.0. Date: **27 Sep 2026 (IST)**.

---

## Pipeline Flow

```text
 git push (main)
      │
      ▼
┌──────────────────────┐ ┌───────────────────────┐ ┌──────────────────────┐ ┌───────────────────────┐
│ Build & Unit Tests   │ │ SAST - CodeQL&Bandit  │ │ SCA - Dependency Scan│ │ Secret Scan - Gitleaks│
│ compileall + pytest  │ │ code-scanning + B201… │ │ pip-audit            │ │ full git history      │
└──────────┬───────────┘ └──────────┬────────────┘ └──────────┬───────────┘ └──────────┬────────────┘
           └────────────────────────┴───────────┬─────────────┴────────────────────────┘
                                                 ▼  (all four must pass)
                                        ┌──────────────────┐
                                        │   Docker Build   │  docker build + docker save → artifact
                                        └────────┬─────────┘
                                                 ▼
                                        ┌──────────────────┐
                                        │ Image Scan-Trivy │  HIGH/CRITICAL (fixable) → exit 1
                                        └────────┬─────────┘
                                                 ▼
                                        ┌──────────────────┐
                                        │  Security Gate   │  if: always() – every check must be "success"
                                        └───┬──────────┬───┘
                                       FAIL │          │ PASS
                                            ▼          ▼
                                          STOP   ┌──────────────────┐
                                                 │ Push Image→GHCR  │  :<git-sha> and :latest
                                                 └────────┬─────────┘
                                                          ▼
                                                 ┌──────────────────┐
                                                 │ Deploy to K8s    │  apply → rollout status → curl
                                                 └──────────────────┘
```

This is the expected flow **Code → Build → Unit Test → SAST → SCA → Secret Scan → Docker Build → Container Image Scan → Security Gate → Push Image → Deploy to Kubernetes**. The four "code" checks are independent, so they run in parallel to save time. Docker Build `needs` all four.

### What I added to the starter project

The starter `demo/` had tests, CodeQL, pip-audit and Trivy. Some parts of the assignment were missing, so I added them:

| Gap in the starter | Change |
|---|---|
| No secret scanning stage | New `secret-scan` job (Gitleaks) + [`.gitleaks.toml`](./demo/.gitleaks.toml) |
| Trivy only reported findings (no `--exit-code`), so it was not a gate | [`trivy.yaml`](./demo/trivy.yaml) with `exit-code: 1`, `HIGH,CRITICAL`, `ignore-unfixed` + [`.trivyignore`](./demo/.trivyignore) |
| No explicit security gate | New `security-gate` job that checks the result of every security job |
| Image pushed to someone else's Docker Hub account | Push to **GHCR** with `GITHUB_TOKEN` (`packages: write`) |
| Image rebuilt in 3 different jobs | Built **once**, saved as the `docker-image` artifact, then scanned and pushed from that same artifact |
| No local SAST tool, Flask ran with `debug=True` | Bandit step + [`bandit.yaml`](./demo/bandit.yaml); debug now off by default |
| Container ran as root, `.dockerignore` was empty | Non-root `appuser` (uid 10001), real [`.dockerignore`](./demo/.dockerignore) |
| Deployment had no probes/limits | readiness/liveness probes on `/health`, resource limits, `runAsNonRoot` |
| `SECURITY.md` was the GitHub template | Rewritten with the actual controls and reporting process |

---

## Project Structure

```text
session-17-devsecops/
├── 02-…08-*/README.md            # topic notes (registry, k8s, SAST, SCA, secrets, image scan, gates)
├── demo/                         # the DevSecOps project (pushed as Shaikh-Suja-Rahaman/hey-cicd)
│   ├── app/app.py                # Flask application
│   ├── app/templates, app/static # dashboard UI
│   ├── tests/test_app.py         # 8 unit tests
│   ├── Dockerfile                # python:3.12-slim, non-root
│   ├── .dockerignore
│   ├── requirements.txt / requirements-dev.txt / pytest.ini
│   ├── bandit.yaml               # SAST config
│   ├── .gitleaks.toml            # secret-scanning config
│   ├── trivy.yaml / .trivyignore # image-scanning + gate config
│   ├── k8s/deployment.yaml       # Deployment (2 replicas, probes, limits)
│   ├── k8s/service.yaml          # NodePort 30001 → 5001
│   ├── SECURITY.md
│   └── .github/workflows/devsecops.yml
├── screenshots/
└── MyReadme.md
```

---

## Stage 1: Build & Unit Tests

**Goal:** Install the dependencies, compile the application and run the unit tests with coverage. The pipeline stops if a test fails or coverage drops below 60%.

**Workflow (`test` job):**
```yaml
  test:
    name: Build & Unit Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - name: Install dependencies
        run: pip install -r requirements-dev.txt
      - name: Build application (byte-compile)
        run: python -m compileall -q app
      - name: Run tests
        run: pytest --cov=app --cov-report=term-missing --cov-fail-under=60
```

**Commands (local):**
```bash
cd session-17-devsecops/demo
python3 -m venv .venv && source .venv/bin/activate.fish
pip install -q -r requirements-dev.txt
pytest -v --cov=app --cov-report=term-missing --cov-fail-under=60
```

**Output:**
```text
collected 8 items

tests/test_app.py::test_home PASSED                                       [ 12%]
tests/test_app.py::test_health PASSED                                     [ 25%]
...
tests/test_app.py::test_status PASSED                                     [100%]

Name              Stmts   Miss  Cover   Missing
-----------------------------------------------
app/__init__.py       0      0   100%
app/app.py           96     32    67%   100, 110-111, 127, 134, 138-139, 151, 185-215, 231, 236, 242
-----------------------------------------------
TOTAL                96     32    67%
Required test coverage of 60% reached. Total coverage: 66.67%
=============================== 8 passed in 0.41s ===============================
```

![pytest with coverage](./screenshots/01-pytest-coverage.png)

* All 8 tests pass: home page, `/health`, greeting, add (valid and missing field), calculator multiply and divide by zero, and `/api/status`.
* The uncovered lines are mostly the `/api/pipeline/run` simulator (185–215) and the error branches. Coverage is 66.67%, which passes the `--cov-fail-under=60` quality gate.
* While doing this I also replaced `datetime.utcnow()` with a small `_utcnow()` helper, because `utcnow()` is deprecated from Python 3.12 and was creating warnings.

---

## Stage 2: SAST – CodeQL + Bandit

**Goal:** Check our own source code for security weaknesses without running it.

**Workflow (`sast` job):**
```yaml
  sast:
    name: SAST - CodeQL & Bandit
    permissions:
      contents: read
      security-events: write        # CodeQL uploads results to the Security tab
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: python
      - uses: github/codeql-action/analyze@v3
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Run Bandit (fail on MEDIUM or higher)
        run: |
          pip install bandit
          bandit -r app -c bandit.yaml --severity-level medium
```

**Tool config: [`demo/bandit.yaml`](./demo/bandit.yaml)**
```yaml
exclude_dirs:
  - tests
  - .venv
skips:
  # B104: the app runs inside a container and must listen on 0.0.0.0
  - B104
```

CodeQL gives deep data-flow analysis and shows its alerts under **Security → Code scanning**. Bandit is fast, I can run it on my laptop, and it gives a hard pass/fail. Bandit is the part of this job that blocks the pipeline.

### 2.1 Gate fails: Flask debug mode

**Command:**
```bash
bandit -r app -c bandit.yaml --severity-level medium
```

**Output:**
```text
Test results:
>> Issue: [B201:flask_debug_true] A Flask app appears to be run with debug=True, which exposes the Werkzeug debugger and allows the execution of arbitrary code.
   Severity: High   Confidence: Medium
   CWE: CWE-94 (https://cwe.mitre.org/data/definitions/94.html)
   Location: app/app.py:240:4
239     if __name__ == "__main__":
240         app.run(host="0.0.0.0", port=5001, debug=True)
...
        Total issues (by severity):
                Low: 5
                Medium: 0
                High: 1
```

![Bandit finds B201](./screenshots/02-bandit-sast-fail.png)

`debug=True` turns on the Werkzeug interactive debugger. Anyone who can reach the pod could use it to run Python code (CWE-94). Bandit exits with code `1`, so the SAST job would fail and Docker Build would never start.

### 2.2 Fix and pass

The fix in `app/app.py` makes debug mode opt-in:
```python
    # Debug mode is OFF by default (Bandit B201 / CodeQL py/flask-debug).
    # Set FLASK_DEBUG=1 only for local development.
    app.run(host="0.0.0.0", port=5001, debug=os.getenv("FLASK_DEBUG", "0") == "1")
```

**Output:**
```text
Test results:
        No issues identified.
...
        Total issues (by severity):
                Low: 5
                Medium: 0
                High: 0
```

![Bandit passes](./screenshots/03-bandit-sast-pass.png)

* The 5 **LOW** findings are `B311` (`random.choice/random/uniform/randint`). They are used for the greeting text and the pipeline simulator, not for security, so they are only reported and do not fail the gate (`--severity-level medium`).
* `B104` (bind to `0.0.0.0`) is skipped on purpose in `bandit.yaml`, and the reason is written in the config. A container has to listen on all interfaces so the Service can reach it.

---

## Stage 3: SCA – pip-audit

**Goal:** Check third-party dependencies (Flask and the packages it pulls in) against the PyPI/OSV advisory database.

**Workflow (`sca` job):**
```yaml
      - name: Install pip-audit
        run: pip install pip-audit
      - name: Run dependency scan
        run: |
          pip-audit -r requirements.txt
          pip-audit -r requirements-dev.txt
```

`pip-audit -r` resolves the requirement files in a temporary environment. It exits with a non-zero code if **any** known vulnerability is found, so this job is a gate on its own.

**Commands:**
```bash
pip-audit -r requirements.txt
pip-audit -r requirements-dev.txt
pip-audit -r requirements.txt -f json 2>/dev/null | jq -c '.dependencies[]'
```

**Output:**
```text
No known vulnerabilities found
No known vulnerabilities found
{"name":"flask","version":"3.1.3","vulns":[]}
{"name":"blinker","version":"1.9.0","vulns":[]}
{"name":"click","version":"8.3.0","vulns":[]}
{"name":"itsdangerous","version":"2.2.0","vulns":[]}
{"name":"jinja2","version":"3.1.6","vulns":[]}
{"name":"markupsafe","version":"3.0.3","vulns":[]}
{"name":"werkzeug","version":"3.1.5","vulns":[]}
```

![pip-audit](./screenshots/04-pip-audit-sca.png)

The JSON output shows the full resolved tree: `Flask==3.1.3` and the 6 packages it depends on. None of them has a known advisory, so `.fixes` is empty.

---

## Stage 4: Secret Scanning – Gitleaks

**Goal:** Stop credentials from reaching the repository.

**Workflow (`secret-scan` job):**
```yaml
  secret-scan:
    name: Secret Scan - Gitleaks
    steps:
      - name: Checkout code (full history)
        uses: actions/checkout@v4
        with:
          fetch-depth: 0              # scan every commit, not only the last one
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_CONFIG: .gitleaks.toml
```

**Tool config: [`demo/.gitleaks.toml`](./demo/.gitleaks.toml)**
```toml
[extend]
useDefault = true          # built-in rules: AWS keys, GitHub tokens, private keys, generic API keys…

[allowlist]
description = "Generated files that can never contain real secrets"
paths = [ '''(^|/)\.coverage$''', '''(^|/)__pycache__/''', '''(^|/)\.pytest_cache/''', '''(^|/)\.venv/''' ]
```

### 4.1 Demo: a planted (fake) AWS key is caught

For the demo I created `app/settings.py` with a **fake** AWS key pair, just like a developer "testing quickly" might do. I scanned the working tree **before committing**.

**Command:**
```bash
gitleaks dir . --config .gitleaks.toml -v
```

**Output:**
```text
Finding:     AWS_ACCESS_KEY_ID = "AKIAXXXXXXXXXXXXXXXX"
Secret:      AKIAXXXXXXXXXXXXXXXX
RuleID:      aws-access-token
File:        app/settings.py
Line:        3

Finding:     AWS_SECRET_ACCESS_KEY = "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX"
RuleID:      generic-api-key
File:        app/settings.py
Line:        4

10:15AM INF scanned ~63541 bytes (63.54 KB) in 34.8ms
10:15AM WRN leaks found: 2
```

![Gitleaks finds the planted key](./screenshots/05-gitleaks-leak-found.png)

Both values are caught: the access key ID by the `aws-access-token` rule (`AKIA…` pattern), and the secret key by `generic-api-key` (a `…KEY = "<high-entropy string>"` assignment, entropy 5.17). Gitleaks exits with `1`, so the same commit would fail the `secret-scan` job and the security gate in CI.

### 4.2 Fix and pass

I deleted the file. Real credentials belong in environment variables or **GitHub Actions secrets** (`${{ secrets.NAME }}`), never in the code.

```bash
rm app/settings.py
gitleaks dir . --config .gitleaks.toml -v
```

**Output:**
```text
10:17AM INF scanned ~63338 bytes (63.34 KB) in 29.1ms
10:17AM INF no leaks found
```

![Gitleaks clean](./screenshots/06-gitleaks-clean.png)

> The key was never committed, so nothing had to be rotated. If a **real** key is ever pushed, deleting the line is not enough. The key has to be revoked/rotated first, and then removed from the history (see `06-secret-scanning/README.md`).

---

## Stage 5: Docker Build

**Goal:** Package the application as a small, non-root image. Build it **once** and pass that exact image to the scan and push jobs.

**[`demo/Dockerfile`](./demo/Dockerfile):**
```dockerfile
FROM python:3.12-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
RUN useradd --create-home --uid 10001 appuser   # run as non-root
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app ./app
USER appuser
EXPOSE 5001
CMD ["python", "app/app.py"]
```

**Workflow (`docker-build` job):**
```yaml
  docker-build:
    needs: [test, sast, sca, secret-scan]
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker image
        run: docker build -t $LOCAL_IMAGE:${{ github.sha }} .
      - name: Save image as artifact
        run: docker save $LOCAL_IMAGE:${{ github.sha }} | gzip > image.tar.gz
      - uses: actions/upload-artifact@v4
        with:
          name: docker-image
          path: image.tar.gz
          retention-days: 1
```

**Commands (local check):**
```bash
docker build -t session17-python:1.0 .
docker run -d --name s17 -p 5001:5001 session17-python:1.0
docker exec s17 id
curl -s localhost:5001/health
docker rm -f s17
```

**Output:**
```text
[+] Building 18.6s (12/12) FINISHED                         docker:desktop-linux
 => [1/6] FROM docker.io/library/python:3.12-slim@sha256:5b1f9c3e…
 => [2/6] RUN useradd --create-home --uid 10001 appuser
 ...
 => => naming to docker.io/library/session17-python:1.0
uid=10001(appuser) gid=10001(appuser) groups=10001(appuser)
{"status":"healthy","timestamp":"2026-09-27T05:14:52.318406Z","uptime_seconds":6.42}
```

![docker build](./screenshots/07-docker-build.png)

The `.dockerignore` keeps `tests/`, `k8s/`, `.git`, `.venv`, `.coverage` and the scanner configs out of the build context (only 41 kB is sent). `id` confirms the process runs as `appuser`, not root.

---

## Stage 6: Container Image Scanning – Trivy

**Goal:** Scan the **final image** (OS packages + Python packages). The source code can be clean while the base image still has vulnerable libraries.

**Tool config: [`demo/trivy.yaml`](./demo/trivy.yaml)**
```yaml
scan:
  scanners:
    - vuln
severity:
  - HIGH
  - CRITICAL
exit-code: 1                 # <- this is what makes the scan a gate
vulnerability:
  ignore-unfixed: true       # only block on CVEs that actually have a fix
format: table
```

**Workflow (`image-scan` job):**
```yaml
  image-scan:
    needs: [docker-build]
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: docker-image
      - name: Load image
        run: gunzip -c image.tar.gz | docker load
      - name: Install Trivy
        run: …            # official aquasecurity apt repository
      - name: Scan image
        run: |
          # severity, exit-code, ignore-unfixed and scanners come from trivy.yaml
          trivy image --config trivy.yaml $LOCAL_IMAGE:${{ github.sha }}
```

`.trivyignore` is there for accepted risks (one CVE ID per line, with a reason). It is empty on purpose because nothing is accepted.

---

## Stage 7: Security Gate (fail → fix → pass)

The `security-gate` job is the **single decision point**. It runs with `if: always()`, so it still runs (and reports) when an earlier job failed. It fails unless **every** security job finished with `success`. `push` only `needs` this job, so a failed gate means nothing is published or deployed.

**Workflow (`security-gate` job):**
```yaml
  security-gate:
    name: Security Gate
    needs: [sast, sca, secret-scan, image-scan]
    if: always()
    steps:
      - name: Evaluate security checks
        run: |
          echo "SAST (CodeQL + Bandit) : ${{ needs.sast.result }}"
          echo "SCA (pip-audit)        : ${{ needs.sca.result }}"
          echo "Secret scan (Gitleaks) : ${{ needs.secret-scan.result }}"
          echo "Image scan (Trivy)     : ${{ needs.image-scan.result }}"
          if [ "${{ needs.sast.result }}" != "success" ] || \
             [ "${{ needs.sca.result }}" != "success" ] || \
             [ "${{ needs.secret-scan.result }}" != "success" ] || \
             [ "${{ needs.image-scan.result }}" != "success" ]; then
            echo "::error title=Security Gate::Security gate FAILED - image will not be pushed or deployed"
            exit 1
          fi
          echo "Security gate PASSED - image can be published"
```

### 7.1 Run #6 is blocked by the gate

My first push with the new pipeline was commit `4e9b1c7` ("ci: add secret scan, bandit and security gate"). At that time the Dockerfile still used the old pinned base image `FROM python:3.12.3-slim` (Debian 12.5). Tests, SAST, SCA, secret scan and Docker build all passed, but **Trivy found 12 fixable HIGH/CRITICAL CVEs**. It exited with `1`, the gate failed, and **Push and Deploy were skipped**.

![Run #6 blocked at the security gate](./screenshots/10-actions-run-blocked.png)

![Run #6 – Trivy job log](./screenshots/11-actions-job-image-scan-failed.png)

I reproduced it locally with the same config:

**Command:**
```bash
grep FROM Dockerfile          # FROM python:3.12.3-slim
trivy image --config trivy.yaml session17-python:0.9
```

**Output (partial):**
```text
session17-python:0.9 (debian 12.5)
==================================
Total: 12 (HIGH: 6, CRITICAL: 6)

│ Library          │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version    │
│ libc-bin / libc6 │ CVE-2024-2961  │   HIGH   │ fixed  │ 2.36-9+deb12u4    │ 2.36-9+deb12u7   │
│                  │ CVE-2024-33599 │   HIGH   │ fixed  │ 2.36-9+deb12u4    │ 2.36-9+deb12u7   │
│ libexpat1        │ CVE-2024-45491 │ CRITICAL │ fixed  │ 2.5.0-1           │ 2.5.0-1+deb12u1  │
│                  │ CVE-2024-45492 │ CRITICAL │ fixed  │ 2.5.0-1           │ 2.5.0-1+deb12u1  │
│ libgssapi-krb5-2, libk5crypto3, libkrb5-3, libkrb5support0                                    │
│                  │ CVE-2024-37371 │ CRITICAL │ fixed  │ 1.20.1-2+deb12u1  │ 1.20.1-2+deb12u2 │
│ libssl3          │ CVE-2024-6119  │   HIGH   │ fixed  │ 3.0.11-1~deb12u2  │ 3.0.14-1~deb12u2 │
│ perl-base        │ CVE-2023-31484 │   HIGH   │ fixed  │ 5.36.0-7          │ 5.36.0-7+deb12u1 │
```

![Trivy gate fails locally](./screenshots/08-trivy-gate-fail.png)

**Why:** pinning an exact old patch tag (`3.12.3-slim`) freezes the OS layer from April 2024. Our code and our Python dependencies were fine (`Python … python-pkg … 0`). Every finding is in a Debian package (glibc, expat, krb5, OpenSSL, perl), and each one already has a fixed version.

### 7.2 Fix: move to the maintained base image

I changed the base image to `FROM python:3.12-slim`. This tag is rebuilt regularly and is now based on Debian 13 "trixie". I rebuilt the image and scanned it again:

**Command:**
```bash
grep FROM Dockerfile          # FROM python:3.12-slim
trivy image --config trivy.yaml session17-python:1.0
echo $status
```

**Output:**
```text
┌────────────────────────────────────┬────────────┬─────────────────┐
│               Target               │    Type    │ Vulnerabilities │
├────────────────────────────────────┼────────────┼─────────────────┤
│ session17-python:1.0 (debian 13.5) │   debian   │        0        │
│ Python                             │ python-pkg │        0        │
└────────────────────────────────────┴────────────┴─────────────────┘
Total: 0 (HIGH: 0, CRITICAL: 0)
0
```

![Trivy gate passes](./screenshots/09-trivy-gate-pass.png)

The exit status is `0`. There are still 23 LOW/MEDIUM findings with fixes available. They are below the gate threshold, so they are tracked but do not block delivery.

### 7.3 Run #7 – full pipeline green

I pushed the fix as commit `a71f3d2` ("fix(docker): use python:3.12-slim base image to pass Trivy gate"). **All 9 jobs passed** in 4m 52s, and the image was pushed and deployed.

![Run #7 successful](./screenshots/12-actions-run-success.png)

The Trivy step in run #7 scans the exact image that was built in Docker Build (loaded from the `docker-image` artifact):

![Run #7 – Trivy job log](./screenshots/13-actions-job-image-scan-passed.png)

| Run | Commit | Tests | SAST | SCA | Secrets | Build | Trivy | Gate | Push | Deploy |
|-----|--------|-------|------|-----|---------|-------|-------|------|------|--------|
| #6 | `4e9b1c7` | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ 12 HIGH/CRIT | ❌ | ⏭ skipped | ⏭ skipped |
| #7 | `a71f3d2` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ 0 | ✅ | ✅ | ✅ |

---

## Stage 8: Push Image to GHCR

**Workflow (`push` job):**
```yaml
  push:
    name: Push Image to GHCR
    needs: [security-gate]
    if: github.event_name == 'push'      # pull requests never publish images
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: docker-image
      - run: gunzip -c image.tar.gz | docker load
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Tag image
        run: |
          docker tag $LOCAL_IMAGE:${{ github.sha }} $IMAGE_NAME:${{ github.sha }}
          docker tag $LOCAL_IMAGE:${{ github.sha }} $IMAGE_NAME:latest
      - name: Push image
        run: |
          docker push $IMAGE_NAME:${{ github.sha }}
          docker push $IMAGE_NAME:latest
```

`IMAGE_NAME` is `ghcr.io/shaikh-suja-rahaman/hey-cicd`, all lowercase because GHCR requires that. No personal token is needed, because the built-in `GITHUB_TOKEN` gets `packages: write` only in this job.

**Output (Push image step):**
```text
The push refers to repository [ghcr.io/shaikh-suja-rahaman/hey-cicd]
9f2c4e7a1b3d: Pushed
...
a71f3d2c9e84b6015fd2a3c7e9b48d16f0c2e5a7: digest: sha256:4c8e2f7a1d9b3605e8c4f2a7d1b9e3c605f8a2d4c7e1b9f3a6d0c8e5b2f7a419 size: 2201
latest: digest: sha256:4c8e2f7a1d9b3605e8c4f2a7d1b9e3c605f8a2d4c7e1b9f3a6d0c8e5b2f7a419 size: 2201
```

![Push to GHCR](./screenshots/14-actions-job-push-ghcr.png)

Both tags point to the same digest. The immutable `:<git-sha>` tag is the one that gets deployed, so every running pod can be traced back to one commit and one pipeline run.

---

## Stage 9: Deploy to Kubernetes

**[`demo/k8s/deployment.yaml`](./demo/k8s/deployment.yaml)** (key parts):
```yaml
spec:
  replicas: 2
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
      containers:
        - name: session17-python
          image: ghcr.io/shaikh-suja-rahaman/hey-cicd:__IMAGE_TAG__   # replaced with the git SHA
          ports:
            - containerPort: 5001
          securityContext:
            allowPrivilegeEscalation: false
          resources:
            limits: { cpu: 250m, memory: 128Mi }
          readinessProbe:
            httpGet: { path: /health, port: 5001 }
          livenessProbe:
            httpGet: { path: /health, port: 5001 }
```

**[`demo/k8s/service.yaml`](./demo/k8s/service.yaml):** `NodePort` service `session17-python`, port `80` → targetPort `5001`, nodePort `30001`.

### 9.1 In the pipeline (kind cluster on the runner)

A GitHub-hosted runner cannot reach the minikube cluster on my laptop. So the `deploy` job creates a throw-away **kind** cluster, deploys the exact SHA that was just pushed, waits for the rollout and then calls the app:

```yaml
      - uses: helm/kind-action@v1.10.0
      - run: sed -i "s|__IMAGE_TAG__|${{ github.sha }}|g" k8s/deployment.yaml
      - run: |
          kubectl apply -f k8s/deployment.yaml
          kubectl apply -f k8s/service.yaml
      - run: |
          kubectl rollout status deployment/session17-python --timeout=120s
          kubectl get pods -l app=session17-python -o wide
      - run: |
          kubectl port-forward service/session17-python 5001:80 &
          sleep 3
          curl -s http://localhost:5001/health
          curl -s http://localhost:5001/api/status
```

![Deploy job](./screenshots/15-actions-job-deploy.png)

### 9.2 On my minikube cluster

**Commands:**
```bash
sed "s|__IMAGE_TAG__|a71f3d2c9e84b6015fd2a3c7e9b48d16f0c2e5a7|" k8s/deployment.yaml | kubectl apply -f -
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/session17-python
kubectl get deploy,svc session17-python
kubectl get pods -l app=session17-python -o wide
```

**Output:**
```text
deployment.apps/session17-python created
service/session17-python created
Waiting for deployment "session17-python" rollout to finish: 0 of 2 updated replicas are available...
Waiting for deployment "session17-python" rollout to finish: 1 of 2 updated replicas are available...
deployment "session17-python" successfully rolled out

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/session17-python   2/2     2            2           19s

NAME                       TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/session17-python   NodePort   10.108.42.173   <none>        80:30001/TCP   17s

NAME                                READY   STATUS    RESTARTS   AGE   IP             NODE
session17-python-7f5c9d8b46-m4qzt   1/1     Running   0          22s   10.244.0.131   minikube
session17-python-7f5c9d8b46-w8x2j   1/1     Running   0          22s   10.244.0.132   minikube
```

![kubectl deploy on minikube](./screenshots/16-kubectl-deploy-minikube.png)

The events show the kubelet pulling `ghcr.io/shaikh-suja-rahaman/hey-cicd:a71f3d2…` straight from GHCR. The pod-template hash `7f5c9d8b46` is the same as in the kind cluster in CI, because both clusters got exactly the same pod spec and image.

### 9.3 Verify the application

**Commands:**
```bash
kubectl port-forward svc/session17-python 8080:80 > /dev/null &
curl -s localhost:8080/health | jq
curl -s localhost:8080/api/status | jq
curl -s -X POST localhost:8080/api/calculate -H "Content-Type: application/json" \
     -d '{"a": 6, "b": 7, "operation": "multiply"}' | jq -c
```

**Output:**
```text
{
  "status": "healthy",
  "timestamp": "2026-09-27T05:41:18.206733Z",
  "uptime_seconds": 223.87
}
{
  "app": "DevSecOps Dashboard",
  "platform": "Linux",
  "python_version": "3.12.12",
  "status": "running",
  ...
  "version": "2.0.0"
}
{"a":6.0,"b":7.0,"expression":"6.0 × 7.0 = 42.0","operation":"multiply","result":42.0,"symbol":"×"}
```

![curl the app](./screenshots/17-app-access.png)

![/api/status in the browser](./screenshots/18-app-api-status.png)

`total_requests` keeps going up even when I don't call the app. This is because the readiness (every 10s) and liveness (every 20s) probes also call `/health`, and the counter is per pod.

---

## Required Secrets / Settings

| Item | Where | Why |
|------|-------|-----|
| `GITHUB_TOKEN` | automatic | GHCR login (`packages: write`), Gitleaks action, CodeQL upload (`security-events: write`) |
| GHCR package visibility → **Public** | GitHub → Packages → hey-cicd → Settings | so the kind/minikube nodes can pull without an `imagePullSecret` |
| Workflow-level `permissions: contents: read` | `devsecops.yml` | least privilege; jobs only ask for more where they need it |

---

## Deliverables Checklist

| Deliverable | Location |
|-------------|----------|
| ✅ Application | [`demo/app/app.py`](./demo/app/app.py), [`demo/app/templates/index.html`](./demo/app/templates/index.html) |
| ✅ Unit tests | [`demo/tests/test_app.py`](./demo/tests/test_app.py), [`demo/pytest.ini`](./demo/pytest.ini) |
| ✅ Dockerfile | [`demo/Dockerfile`](./demo/Dockerfile), [`demo/.dockerignore`](./demo/.dockerignore) |
| ✅ GitHub Actions workflow | [`demo/.github/workflows/devsecops.yml`](./demo/.github/workflows/devsecops.yml) |
| ✅ Security tools configuration | [`bandit.yaml`](./demo/bandit.yaml), [`.gitleaks.toml`](./demo/.gitleaks.toml), [`trivy.yaml`](./demo/trivy.yaml), [`.trivyignore`](./demo/.trivyignore), [`SECURITY.md`](./demo/SECURITY.md) |
| ✅ Kubernetes manifests | [`demo/k8s/deployment.yaml`](./demo/k8s/deployment.yaml), [`demo/k8s/service.yaml`](./demo/k8s/service.yaml) |
| ✅ Successful pipeline output | Run #7: [summary](./screenshots/12-actions-run-success.png), [image scan](./screenshots/13-actions-job-image-scan-passed.png), [push](./screenshots/14-actions-job-push-ghcr.png), [deploy](./screenshots/15-actions-job-deploy.png) |
| ✅ Security gate demonstration | Run #6 [blocked](./screenshots/10-actions-run-blocked.png); Bandit [fail](./screenshots/02-bandit-sast-fail.png)/[pass](./screenshots/03-bandit-sast-pass.png); Gitleaks [fail](./screenshots/05-gitleaks-leak-found.png)/[pass](./screenshots/06-gitleaks-clean.png); Trivy [fail](./screenshots/08-trivy-gate-fail.png)/[pass](./screenshots/09-trivy-gate-pass.png) |
| ✅ Screenshots | [`screenshots/`](./screenshots) (18 images) |
| ✅ Complete README | this file + [`demo/README.md`](./demo/README.md) |

---

## Notes / Learnings

* **A scan is not a gate.** The original Trivy step printed CVEs and still passed. Only `exit-code: 1` (or a failing job checked in `needs`) actually stops delivery.
* **`if: always()` on the gate job** makes the decision visible. Without it, a failed scan would just make the downstream jobs "skipped", and nobody would see a clear "Security gate FAILED" message.
* **Build once, promote the same artifact.** The image that Trivy scanned is byte-for-byte the image that was pushed (same `docker-image` artifact). Rebuilding in every job would scan a different image from the one that is shipped.
* **Pinning a patch tag freezes the OS too.** `python:3.12.3-slim` looked "safe and reproducible", but it carried 6 CRITICAL CVEs that already had fixes. Using the maintained `3.12-slim` tag (or pinning by digest and updating it regularly, e.g. with Dependabot) keeps the base image patched.
* **Tool overlap is useful.** CodeQL (deep analysis, Security tab) and Bandit (fast, local, hard gate) both flag Flask debug mode. pip-audit covers our declared dependencies, while Trivy also covers OS packages and anything installed in the image.
* **Thresholds are policy.** `HIGH,CRITICAL` + `ignore-unfixed` for images, `MEDIUM+` for Bandit and "any" for pip-audit and Gitleaks are good for a class project. A real organisation should set these per application risk and use `.trivyignore` (with reasons) for exceptions it has accepted.
