# Memobase K8s Deployment — memory-benchmark cluster

Deploys the instrumented Memobase fork (`feat/otel-memory-instrumentation`) as the
single-resident memory solution on the memory-benchmark cluster, into the
**`benchmark-memobase`** namespace (matches the `benchmark-<backend>` convention
already provisioned on the cluster for every memory backend).

## Prerequisites

- K8s cluster context `memory-benchmark-admin@memory-benchmark`
  (kubeconfig: `~/.config/capmox/memory-benchmark.kubeconfig`).
- OTel collector reachable at `otel-collector.observability.svc.cluster.local:4317`
  (verified present on the cluster).
- **A pullable fork image** (see Build section) — this is currently the deploy blocker.

## Build the fork image  ⚠️ REQUIRED — not yet published

The manifest `04-api.yaml` references `ghcr.io/henrikrexed/memobase:<tag>`. No such
image is published yet:

- CI (`.github/workflows/publish.yaml`) only builds on push to `main`/`dev`, on
  `v*` tags, and on PRs — **not** on this feature branch. So no image was produced
  by CI for this branch.
- CI's tag scheme is `docker/metadata-action` `type=sha`, which emits
  `sha-<shortsha>` (e.g. `sha-eb83043`), **never** a bare `:f96ebe2`.
- The previously-pinned `f96ebe2` is a dangling pre-rebase commit; the on-branch
  instrumentation commit is `eb83043`.

To publish, either build+push manually (needs a container runtime + GHCR write creds):

```bash
cd src/server/api
docker build -t ghcr.io/henrikrexed/memobase:otel-eb83043 .
docker push ghcr.io/henrikrexed/memobase:otel-eb83043
```

…or trigger CI (`workflow_dispatch` on `publish.yaml`, or open a PR to `dev`), then
read the exact pushed tag from the run and set it in `04-api.yaml`.

## Deploy

```bash
export KUBECONFIG=~/.config/capmox/memory-benchmark.kubeconfig

# 0. Set the real image tag in 04-api.yaml (see Build section).

# 1. Create the namespace + secret with REAL credentials (do NOT commit real values):
kubectl apply -f deploy/k8s/00-namespace.yaml
kubectl create secret generic memobase-secret -n benchmark-memobase \
  --from-literal=DATABASE_USER=memobase \
  --from-literal=DATABASE_PASSWORD=<real> \
  --from-literal=DATABASE_NAME=memobase \
  --from-literal=REDIS_PASSWORD=<real> \
  --from-literal=ACCESS_TOKEN=<real> \
  --from-literal=PROJECT_ID=default \
  --from-literal=LLM_API_KEY=<real>
# (01-secret.yaml is a placeholder template only — its `changeme` values will NOT
#  produce a working campaign because the API needs a live LLM_API_KEY.)

# 2. Apply the rest:
kubectl apply -f deploy/k8s/02-postgres.yaml
kubectl apply -f deploy/k8s/03-redis.yaml
kubectl apply -f deploy/k8s/04-api.yaml

# 3. Wait for API to become ready:
kubectl rollout status deployment/memobase-server-api -n benchmark-memobase

# 4. Run campaign:
kubectl apply -f deploy/k8s/05-campaign-runner-job.yaml
kubectl logs -f job/campaign-runner-memobase -n benchmark-memobase

# 5. (Optional) Apply ServiceMonitor if using prometheus-operator:
kubectl apply -f deploy/k8s/06-prometheus-scrape.yaml
```

## Telemetry

- **Traces**: OTLP-push to `otel-collector.observability` → LGTM/Tempo
- **Metrics**: Prometheus-pull from `:9464/metrics` (pod annotations enable auto-scrape)
  - `memobase_server_memory_*` — memory-semconv v0.1.0 metrics
  - `memory.*` — span names from OTel traces

## Hand-off to ISI-1925

After campaign-runner Job completes with ≥5 reps:
1. Confirm spans `memory.*` appear in Tempo
2. Confirm metrics `memobase_server_memory_*` appear in Prometheus/LGTM
3. Hand kube-context + LGTM access to Observability Agent for [ISI-1925](ISI-1925) conformance check
