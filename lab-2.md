# Lab brief: S3 uploads & IAM (IaC fix)

## Incident

Since this morning, **`POST /api/v1/files`** has been returning **`500`** while **`GET /api/v1/files/...`** (downloads) still works for objects that already exist. The platform team shipped a Terraform refactor yesterday that touched **IAM policies** attached to the EC2 instance profile. The API pods are not crashing, so dashboards look fine, but every upload fails. The service is running on the lab host—reproduce it, find what is broken, and fix it with IaC.

## Access

### SSH to the lab host

```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@13.126.188.177
```

Private key (save as `~/.ssh/id_ed25519`, `chmod 600`, then use the command above):

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACCouSbdcewYhCF8w5CKSI+kIrRDi9MHN8nYFMoBqm5EBgAAAJhrNnjcazZ4
3AAAAAtzc2gtZWQyNTUxOQAAACCouSbdcewYhCF8w5CKSI+kIrRDi9MHN8nYFMoBqm5EBg
AAAEDJp8VXHLeJ9YA1jHtPOSLt0zeZ5Fnat1dkzy2lg6zajqi5Jt1x7BiEIXzDkIpIj6Qi
tEOL0wc3ydgUygGqbkQGAAAAEWhhcmlueXBhdGVsQGhhcmlpAQIDBA==
-----END OPENSSH PRIVATE KEY-----
```

### AWS console (optional UI debugging)

URL: [https://084375564929.signin.aws.amazon.com/console](https://084375564929.signin.aws.amazon.com/console)

- **User:** `candidate`
- **Password:** `candidate@123`

## Where things live

- **Codebase + IaC** on the host: `~/lab`  
  (FastAPI app under `app/`, Terraform under `deploy/terraform/`, Kubernetes manifests under `deploy/k8s/`.)

### Kubernetes (k3s)

The cluster is **k3s** (single-node) on the same EC2. `kubectl` is preconfigured for `ec2-user`.

All lab workloads run in the **`lab-python-s3-iam-misconfigured-k3s`** namespace:

| Workload   | Kind        | Role |
|------------|-------------|------|
| `api`      | Deployment  | FastAPI API (includes `POST /api/v1/files` uploads) |
| `postgres` | Deployment  | Database |
| `redis`    | Deployment  | Cache |
| `load-gen` | Deployment  | Scripted transfers + S3 traffic; **`kubectl logs`** here lines up with manual **`GET` / `POST`** below |

Quick checks:

```bash
kubectl get pods -n lab-python-s3-iam-misconfigured-k3s
```

## Tools you can use

`terraform`, `aws` (CLI), `kubectl`, `curl` — plus normal Linux tooling on the host (`nc`, `dig`, `psql`, etc.).

## Endpoints

The API is on port **30800** on the lab host. All examples use `localhost:30800`; you can also `kubectl port-forward` the `api` Deployment to `localhost:8000` if you prefer.



```bash
# health check for postgres and redis
curl -sS -o /dev/null -w 'health  %{http_code}\n' http://localhost:30800/health
```
```bash
# get files from s3
curl -sS -o /dev/null -w 's3-get  %{http_code}\n' http://localhost:30800/api/v1/files/seed/lab-readme.txt
```
```bash
# post files to s3
printf 'probe' | curl -sS -o /dev/null -w 's3-post %{http_code}\n' \
  -X POST "http://localhost:30800/api/v1/files" \
  -F "file=@-;filename=probe.txt;type=text/plain"
```

- **`/health`** - liveness check; returns `200` with `"status": "ok"` when PostgreSQL and Redis are up.
- **`GET /api/v1/files/...`** - S3 read path, works fine.
- **`POST /api/v1/files`** - S3 write path, broken due to the IAM regression. A `201` here means the fix is complete.



## Done when

You can explain the **root cause**, show the **IaC change**, and demonstrate **`/health` returns `200`**, **`GET /api/v1/files/{key}`** returns `200` and **`POST /api/v1/files` returns `201`**.
