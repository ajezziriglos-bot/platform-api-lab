# Platform API Lab

Proof of concept of an internal platform API for governed Kubernetes capabilities. The API exposes a small golden path for creating namespaces with a predictable name, ownership metadata and a resource quota selected from a supported size profile.

The project is intended to be portable across Kubernetes distributions such as OpenShift and EKS, as long as the Kubernetes API and the required RBAC permissions are available.

## Current capabilities

- Public health check at `GET /health`.
- Authenticated namespace listing at `GET /api/v1/namespaces`.
- Developer-only namespace creation at `POST /api/v1/namespaces`.
- Request validation for application, team, environment and size.
- Platform-generated namespace names in the form `<application>-<environment>`.
- Automatic namespace labels for application, team, environment, size and manager.
- Automatic `ResourceQuota` creation for each managed namespace.
- Kubernetes access through the `platform-api` ServiceAccount and the RBAC manifest in `manifests/rbac.yaml`.

## Architecture

The consumer authenticates to the Platform API with a laboratory Bearer token. The API validates the request and applies its naming and quota policies. Kubernetes then authorizes the operation using the identity configured in `platform-api.kubeconfig`.

```text
Consumer
	 |
	 | Bearer token
	 v
Platform API (FastAPI)
	 |
	 | Kubernetes client + platform-api.kubeconfig
	 v
Kubernetes API
	 |
	 +-- Namespace
	 +-- ResourceQuota
```

## Quota profiles

| Profile | CPU | Memory | Pods |
| --- | ---: | ---: | ---: |
| `small` | `2` | `4Gi` | `10` |
| `medium` | `8` | `16Gi` | `30` |
| `large` | `16` | `32Gi` | `60` |

Supported environments are `dev`, `test` and `prod`.

## Requirements

- Python 3.10 or newer.
- Access to a Kubernetes cluster.
- A kubeconfig for the `platform-system/platform-api` ServiceAccount saved as `platform-api.kubeconfig`.
- The RBAC resources applied to the cluster.

## Local setup

Create and activate a virtual environment, then install the libraries used by the application:

```bash
python -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn kubernetes pydantic
```

Apply the Kubernetes permissions:

```bash
kubectl apply -f manifests/rbac.yaml
```

Place the runtime kubeconfig at the project root with the exact name `platform-api.kubeconfig`. This file is intentionally ignored by Git because it contains cluster credentials.

Start the API:

```bash
uvicorn main:app --reload
```

The interactive API documentation is available at http://127.0.0.1:8000/docs.

## API examples

Health check:

```bash
curl http://127.0.0.1:8000/health
```

List namespaces with either laboratory token:

```bash
curl \
	-H 'Authorization: Bearer viewer-token' \
	http://127.0.0.1:8000/api/v1/namespaces
```

Create a namespace with the developer token:

```bash
curl -X POST \
	-H 'Authorization: Bearer developer-token' \
	-H 'Content-Type: application/json' \
	http://127.0.0.1:8000/api/v1/namespaces \
	-d '{
		"application": "payments",
		"team": "platform",
		"environment": "dev",
		"size": "small"
	}'
```

That request creates the namespace `payments-dev` and a `platform-quota` ResourceQuota in it.

## Laboratory authentication

The current implementation contains two fixed tokens in `main.py`:

- `viewer-token`: can list namespaces.
- `developer-token`: can list and create namespaces.

These tokens are only for the proof of concept. A production implementation should replace them with OIDC/OAuth2 and an external identity provider, and should load configuration and secrets from a secure configuration system.

## RBAC scope

The `platform-api` ClusterRole can `get`, `list` and `create`:

- Kubernetes `namespaces`.
- `resourcequotas`.

The `ClusterRoleBinding` grants those permissions to the `platform-system/platform-api` ServiceAccount. The application does not grant update or delete operations.

## Repository safety

The `.gitignore` excludes `.venv`, Python bytecode, local environment files and `*.kubeconfig`. Do not commit a kubeconfig, ServiceAccount token or other cluster credentials.

## Project status

This is a laboratory proof of concept. It currently focuses on the namespace golden path and does not yet include automated tests, persistent application data, production authentication, transactional rollback if quota creation fails, or deployment manifests for the API itself.
