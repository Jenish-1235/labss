# Lab brief: DB connectivity & IaC fix

## Incident

Since this morning, the items API's readiness checks have been failing and the database migration Job has been in `CrashLoopBackOff`. The platform team shipped a Terraform/Pulumi refactor yesterday that touched the VPC and security groups. The app pods aren't crashing, so dashboards are quiet, but every DB call times out. The service is running on the lab host reproduce it, find what's broken, and fix it with IaC.

## Access

### SSH to the lab host

```bash
ssh ubuntu@13.233.238.117
```

Private key (save as `~/.ssh/lab_key`, `chmod 600`, then use `ssh -i ~/.ssh/lab_key ubuntu@13.233.238.117`):

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDcbW50rK+turiEoKv6qqQ+VcEns6NjhVoSYBN6vNhBkgAAAIjsrLVD7Ky1
QwAAAAtzc2gtZWQyNTUxOQAAACDcbW50rK+turiEoKv6qqQ+VcEns6NjhVoSYBN6vNhBkg
AAAECiX/y+DOgE1b16fhKWYxB44UsAeS1AyFrB3NIa7BH8ftxtbnSsr626uISgq/qqpD5V
wSezo2OFWhJgE3q82EGSAAAAA2xhYgEC
-----END OPENSSH PRIVATE KEY-----
```

### AWS console (optional UI debugging)

URL: [https://084375564929.signin.aws.amazon.com/console](https://084375564929.signin.aws.amazon.com/console)

- **User:** `candidate`
- **Password:** `candidate@123`

## Where things live

- **Codebase + IaC** on the host: `~/lab`  
  (FastAPI app under `app/`, Terraform under `deploy/infra/terraform/`, Pulumi mirror under `deploy/infra/pulumi/`, Kubernetes manifests under `deploy/k8s/`.)

### Kubernetes (k3s)

The cluster is **k3s** (single-node) on the same EC2. `kubectl` is preconfigured for `ubuntu`.

All lab workloads run in the **`lab`** namespace:

| Workload      | Kind        | Role |
|---------------|-------------|------|
| `app`         | Deployment  | FastAPI API |
| `redis`       | Deployment  | Cache |
| `app-migrate` | Job         | Runs `alembic upgrade head` against Postgres |

Quick checks:

```bash
kubectl get pods -n lab
```

## Tools you can use

`terraform`, `aws` (CLI), `kubectl`, `curl` — plus normal Linux tooling on the host (`nc`, `dig`, `psql`, etc.).

## Health endpoints

The API is exposed on the host (port-forward to the `app` Deployment):

```bash
curl -sS -o /dev/null -w 'health %{http_code}\n' http://localhost:8000/health
curl -sS -o /dev/null -w 'ready  %{http_code}\n'  http://localhost:8000/ready
```

- **`/health`** — liveness (process is up).
- **`/ready`** — readiness (includes dependency checks; expect problems while the database path is broken).

## Done when

You can explain **root cause**, show the **IaC change**, and demonstrate **`/health` and `/ready` both return 200**, the **`app-migrate` Job is `Complete`**, and the database path behaves as expected.
