# LAB03 — CI/CD with GitHub Actions

## 1. Overview

This lab implements a CI/CD pipeline for the **DevOps Info Service** using **GitHub Actions**.

- **Testing framework:** `pytest`
- **Linting:** `ruff`
- **Container build & publish:** Docker Buildx → Docker Hub
- **Security scanning:** Snyk CLI (dependency scan from `requirements.txt`)

**What is tested:**
- `GET /` returns a valid JSON structure with required sections (`service`, `system`, `runtime`, `request`, `endpoints`)
- `GET /health` returns `"status": "healthy"` and includes `timestamp` and `uptime_seconds`
- unknown endpoints return a JSON 404 response

**Workflow triggers:**
- Runs on `push` and `pull_request` for branches `master` and `lab03`
- Runs only when files in `app_python/**` or the workflow file change (path filter)

**Versioning strategy (Docker tags):**
- **CalVer** tag: `YYYY.MM.DD-<shortSHA>` (traceable build version)
- Branch tag: `lab03` (easy to pull the latest build from this branch)

---

## 2. Workflow Evidence

### 2.1 GitHub Actions run
- Workflow file: `.github/workflows/python-ci.yaml`
- Successful run link: **<PASTE LINK TO ACTION RUN HERE>**

### 2.2 Test + lint output (CI)
Paste minimal evidence (or screenshots):
- Ruff:
```text
<PASTE ruff output here>
``` 
- Pytest:
```text
<PASTE pytest output here>
```

### 2.3 Docker image publish
Docker Hub repository:
- `wkwtfigo/devops-info-service`
- URL: 
```text
https://hub.docker.com/r/wkwtfigo/devops-info-service
```

Published tags from CI:
- `lab03`
- `YYYY.MM.DD-<shortSHA>`

## 3. Best Practices Implemented
### 3.1 Separation of concerns (jobs)

- `test` job runs lint + unit tests
- `docker` job runs only after tests succeed (`needs: test`)
- Result: images are published only from verified code.

### Matrix testing

Tests run on Python 3.12 and 3.13, which increases confidence that the service works across supported versions.

### 3.3 Dependency caching

`actions/setup-python` is configured with pip caching using:
- `app_python/requirements.txt`
- `app_python/requirements-dev.txt`

This reduces CI runtime by reusing cached wheels between runs.

### 3.4 Path filtering

Workflow runs only when app_python/** or the workflow file changes.
This avoids running CI for unrelated edits (docs/lectures/etc.).

### 3.5 Concurrency control

The workflow uses:

```yaml
concurrency:
  group: python-ci-${{ github.ref }}
  cancel-in-progress: true
```

This prevents wasting minutes on outdated runs when multiple commits are pushed quickly.

## 4. Security (Snyk)

Snyk is executed using the Snyk CLI and scans the runtime dependency file:

- Target file: `app_python/requirements.txt`
- Command:

```bash
snyk test --file=app_python/requirements.txt --severity-threshold=high --skip-unresolved
```

**Result:**

- Snyk findings: <PASTE: 0 issues OR N issues>
- Action taken:
    - If findings exist: describe whether you upgraded dependencies or left as known risk
    - The step is configured with continue-on-error: true to keep CI green while still reporting risks (appropriate for a training project).

<PASTE snyk output excerpt here>

## 5. Key Decisions

- Ruff instead of flake8: faster and simpler configuration; good for CI.
- CalVer tags for Docker images: easy to see when an image was produced and link it to a commit (shortSHA).
- Tests validate JSON structure, not exact host values: values like `hostname` and platform differ between local machine and CI runner.

## 6. Challenges & Fixes
### 6.1 Workflow not triggering

**Cause:** incorrect path filter (`.yml` vs `.yaml`) prevented workflow changes from triggering runs.

**Fix:** updated paths to match the real filename: `.github/workflows/python-ci.yaml`.

### 6.2 Snyk action failing due to missing packages

**Cause:** `snyk/actions/python@master` runs in a container and didn’t see installed packages.

**Fix:** switched to Snyk CLI installed via `npm`, scanning `requirements.txt` directly.

### 6.3 Dependency separation

**Cause:** dev tools (pytest/ruff/httpx) were mixed into runtime requirements, increasing image size and scan noise.

**Fix:** split dependencies into:
- `requirements.txt` (runtime)
- `requirements-dev.txt` (dev/test tools)