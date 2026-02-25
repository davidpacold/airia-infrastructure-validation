# Airia Infrastructure Validation

## What This Is
A Kubernetes-native infrastructure validation tool. Customers deploy this via Helm chart (OCI registry) before installing the Airia platform. It tests connectivity to databases, storage, AI services, and Kubernetes resources, providing a web dashboard with pass/fail results and remediation guidance.

## Tech Stack
- Python 3.11, FastAPI, Uvicorn, Jinja2 templates
- Pydantic Settings for configuration
- passlib/bcrypt + python-jose for JWT auth
- Docker multi-stage build, Helm 3 chart
- Published to ghcr.io/davidpacold/airia-infrastructure-validation (OCI)

## Key Directories
- `app/` - FastAPI application
  - `app/main.py` - Routes and endpoints
  - `app/config.py` - Pydantic Settings (env var config, cached)
  - `app/auth.py` - JWT authentication with bcrypt
  - `app/tests/` - Infrastructure test classes (BaseTest -> individual tests)
  - `app/tests/base_test.py` - BaseTest ABC + TestSuite + TestResult
  - `app/tests/test_runner.py` - TestRunner singleton (thread-safe)
- `helm/airia-infrastructure-validation/` - Helm chart (Chart.yaml, values.yaml, templates/)
- `templates/` - Jinja2 HTML templates (dashboard, login)
- `static/` - CSS/JS assets

## Running Locally
```bash
pip install -r requirements.txt
SECURE_COOKIES=false uvicorn app.main:app --host 0.0.0.0 --port 8080 --reload
# Open http://localhost:8080 (login: admin/changeme)
```

## How Tests Work
Each test inherits from `BaseTest` (app/tests/base_test.py). Tests are registered in `TestRunner._register_tests()`. The `execute()` method handles timeout enforcement (ThreadPoolExecutor), retry logic, and error handling. Tests run concurrently when using "Run All". Results are stored in the TestRunner singleton with thread-safe locking.

## Release Workflow
- App version is static (2.0.0) in config.py; runtime version from APP_VERSION env var
- Chart version is managed by release workflow in Chart.yaml
- `.github/workflows/release.yml` builds Docker image + Helm chart
- Published to OCI registry: `oci://ghcr.io/davidpacold/airia-infrastructure-validation/charts`
- Triggered by workflow_dispatch or tag push (`git tag v1.0.X && git push origin v1.0.X`)

## Conventions
- Test classes: one per service, inherit BaseTest, implement run_test()
- Config: use Pydantic Settings native env loading (no manual os.getenv in config.py)
- Secrets: never log, always from env vars / K8s secrets
- Endpoints: auth required via Depends(require_auth)
- Thread safety: use self._lock for shared state in TestRunner

## Relationship to airia-test-pod
This repo was created from `davidpacold/airia-test-pod` (Feb 2025) as a clean rebrand with no git history. The old repo remains deployed for existing customers. Key differences:
- Repo/chart/image name: `airia-infrastructure-validation` (was `airia-test-pod`)
- OCI registry: `ghcr.io/davidpacold/airia-infrastructure-validation`
- Helm install: `helm upgrade --install airia-infrastructure-validation oci://ghcr.io/davidpacold/airia-infrastructure-validation/charts/airia-infrastructure-validation -n airia`
- Internal code identifiers (`TestPodException`, `test_pod_exception_handler`) were intentionally kept as-is

## Deploying to Kubernetes
```bash
# Install/upgrade
helm upgrade --install airia-infrastructure-validation \
  oci://ghcr.io/davidpacold/airia-infrastructure-validation/charts/airia-infrastructure-validation \
  -n airia

# Access dashboard
kubectl port-forward -n airia svc/airia-infrastructure-validation 8080:80

# Get password
kubectl get secret airia-infrastructure-validation-auth -n airia -o jsonpath='{.data.password}' | base64 -d
```
