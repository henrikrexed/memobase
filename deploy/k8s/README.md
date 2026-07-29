# Memobase K8s Deployment — memory-benchmark cluster

Deploys the instrumented Memobase fork (`feat/otel-memory-instrumentation`, commit `f96ebe2`) as the single-resident memory solution on the memory-benchmark cluster.

## Prerequisites

- K8s cluster with the `memory-benchmark` context
- OTel collector reachable at `otel-collector.observability.svc.cluster.local:4317`
- Image built from fork and pushed (see Build section)

## Build the fork image

```bash
cd src/server/api
docker build -t ghcr.io/henrikrexed/memobase:f96ebe2 .
docker push ghcr.io/henrikrexed/memobase:f96ebe2
```

## Deploy

```bash
# 1. Edit 01-secret.yaml with real credentials, then:
kubectl apply -f deploy/k8s/00-namespace.yaml
kubectl apply -f deploy/k8s/01-secret.yaml
kubectl apply -f deploy/k8s/02-postgres.yaml
kubectl apply -f deploy/k8s/03-redis.yaml
kubectl apply -f deploy/k8s/04-api.yaml

# 2. Wait for API to become ready:
kubectl rollout status deployment/memobase-server-api -n memobase

# 3. Run campaign:
kubectl apply -f deploy/k8s/05-campaign-runner-job.yaml
kubectl logs -f job/campaign-runner-memobase -n memobase

# 4. (Optional) Apply ServiceMonitor if using prometheus-operator:
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
