# Borod Bank — AIOps Module

[![Claude API](https://img.shields.io/badge/Claude_API-Haiku_4.5-D97757?logo=anthropic&logoColor=white)](https://docs.anthropic.com/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.115-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-CronJob-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](../../LICENSE)

AI-powered operations module providing **automated code review** on every CI build and **daily log analysis** with Slack reporting. Uses Claude API (Haiku) via a FastAPI gateway running inside the cluster.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        namespace: aiops                                  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  COMPONENT 1: AIOps Gateway (always running)                       │  │
│  │                                                                    │  │
│  │  Jenkins CI ──► POST /review ──► FastAPI ──► Claude API (Haiku)   │   │
│  │       │              │                            │                │  │
│  │       │         code diff +                 security issues,      │   │
│  │       │         build context               quality feedback,     │   │
│  │       │                                     recommendations       │   │
│  │       ◄────────────── JSON response ─────────────┘                │   │
│  │                                                                    │  │
│  │  Prometheus ◄── GET /metrics (aiops_gateway_pending_requests)     │   │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  COMPONENT 2: Log Analyzer CronJob (daily at 3:00 UTC)            │   │
│  │                                                                    │  │
│  │  Loki ──► query ERROR logs ──► deduplicate ──► Claude API (Haiku) │   │
│  │  (last 24h, up to 500)    (MD5 fingerprint,      │                │   │
│  │                            max 10 unique)         ▼                │  │
│  │                                              Slack webhook         │  │
│  │                                              (daily report)        │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  SECRETS (External Secrets Operator → AWS Secrets Manager)         │  │
│  │                                                                    │  │
│  │  borod-bank/aiops    → ANTHROPIC_API_KEY  → aiops-anthropic-secret │  │
│  │  borod-bank/jenkins  → SLACK_WEBHOOK_URL  → aiops-slack-secret     │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  GUARDS: ResourceQuota (5 pods, 2 CPU, 2Gi) + LimitRange per container   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Components

### 1. AIOps Gateway — AI Code Review Service

A FastAPI application exposed as a Kubernetes Deployment + Service. Jenkins pipelines send code diffs via HTTP, the gateway forwards them to Claude API and returns the review.

**Endpoints:**

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/review` | Send code diff for AI review |
| `GET` | `/health` | Liveness/readiness probe |
| `GET` | `/metrics` | Prometheus metrics (via `prometheus_client`) |

**Request format (`POST /review`):**

```json
{
  "diff": "git diff output or nightly summary text",
  "context": "Build #42 commit a1b2c3d4"
}
```

**Response:**

```json
{
  "review": "1) Security: No issues found. 2) Code quality: ..."
}
```

**System prompt** instructs Claude to act as a senior DevOps/Java engineer and provide: security issues, code quality issues, and top 2 actionable improvements in ≤10 lines of plain text.

**How Jenkins uses it:**

| Pipeline | Stage | What is sent |
|---|---|---|
| **CI** (on merge) | `AI Code Review` | `git diff HEAD~1 HEAD -- src/` (capped at 4KB) |
| **Nightly** (daily 2 AM) | `AI Nightly Analysis` | Build metrics summary (tests, coverage, mutations, CVEs, k6 perf) |

Both stages are **non-blocking** — wrapped in `catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE')`. If the gateway is unreachable, the pipeline continues.

---

### 2. Log Analyzer — Daily Error Report

A CronJob that runs a Python script (`aiops_analyzer.py`) every day at **3:00 UTC**. It queries Loki for application errors, deduplicates them, sends unique patterns to Claude for root cause analysis, and posts a summary to Slack.

**Pipeline:**

```
Loki (namespace=borod-bank, |= "ERROR", last 24h)
    │
    ▼ up to 500 log lines
Deduplication (MD5 of first 5 words per line)
    │
    ▼ max 10 unique error patterns
Claude API (Haiku) — root cause + 3 recommendations
    │
    ▼
Slack Block Kit message (header + stats + analysis)
```

**Schedule:** `0 3 * * *` — one hour after the nightly pipeline (2 AM), so fresh errors from nightly tests are included.

**Failure policy:** `backoffLimit: 0`, `restartPolicy: Never` — if the job fails, it does not retry. History keeps 3 successes and 3 failures for debugging.

---

## Manifest Files

| File | Resource | Purpose |
|---|---|---|
| `namespace.yaml` | Namespace | `aiops` namespace with `app.kubernetes.io/part-of: borod-bank` label |
| `gateway-deployment.yaml` | ConfigMap + Deployment + Service | FastAPI gateway: script, pod spec, ClusterIP on port 8080 |
| `analyzer-configmap.yaml` | ConfigMap | Python script for log analysis (`aiops_analyzer.py`) |
| `log-analyzer-cronjob.yaml` | CronJob | Runs log analyzer daily at 3:00 UTC |
| `externalsecret-anthropic.yaml` | ExternalSecret | Syncs `ANTHROPIC_API_KEY` from AWS Secrets Manager |
| `externalsecret-slack.yaml` | ExternalSecret | Syncs `SLACK_WEBHOOK_URL` from AWS Secrets Manager |
| `resourcequota.yaml` | ResourceQuota | Namespace limits: 5 pods, 1 CPU / 1Gi requests, 2 CPU / 2Gi limits |
| `limitrange.yaml` | LimitRange | Per-container defaults: 100m–200m CPU, 128Mi–256Mi RAM (max 1 CPU / 1Gi) |

---

## Resource Limits

### Namespace Quota

| Resource | Limit |
|---|---|
| Pods | 5 |
| CPU requests | 1 core |
| Memory requests | 1Gi |
| CPU limits | 2 cores |
| Memory limits | 2Gi |

### Per-Container Defaults (LimitRange)

| Parameter | CPU | Memory |
|---|---|---|
| Default request | 100m | 128Mi |
| Default limit | 200m | 256Mi |
| Maximum | 1 core | 1Gi |
| Minimum | 10m | 64Mi |

### Actual Resource Usage

| Component | CPU request | CPU limit | Memory request | Memory limit |
|---|---|---|---|---|
| Gateway (Deployment) | 50m | 500m | 128Mi | 512Mi |
| Log Analyzer (CronJob) | 100m | 500m | 256Mi | 512Mi |

---

## Secret Management

All secrets are injected at runtime via **External Secrets Operator** — zero hardcoded credentials in manifests.

```
AWS Secrets Manager          ExternalSecret              K8s Secret
─────────────────────        ──────────────              ──────────
borod-bank/aiops             aiops-anthropic      →      aiops-anthropic-secret
  └─ ANTHROPIC_API_KEY                                     └─ ANTHROPIC_API_KEY

borod-bank/jenkins           aiops-slack          →      aiops-slack-secret
  └─ SLACK_WEBHOOK_URL                                     └─ SLACK_WEBHOOK_URL
```

Both ExternalSecrets use `ClusterSecretStore: aws-secrets-manager` with a 1-hour refresh interval.

---

## ArgoCD Integration

The AIOps module is managed by the App of Apps pattern:

| Field | Value |
|---|---|
| ArgoCD Application | `aiops` |
| Sync wave | 4 (deployed after core infrastructure) |
| Source path | `aiops/` |
| Auto-sync | `prune: true`, `selfHeal: true` |
| Retry | 3 attempts, exponential backoff (5s → 3m) |

Definition: `borod-bank-cd/argocd/apps/aiops.yml`

---

## Monitoring

### Prometheus Metrics

The gateway exposes metrics via `prometheus_client` at `/metrics`:

| Metric | Type | Description |
|---|---|---|
| `aiops_gateway_pending_requests` | Gauge | Currently in-flight review requests |

Pod annotations enable Prometheus auto-discovery:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "8080"
prometheus.io/path: "/metrics"
```

### Grafana Dashboard

A dedicated **AIOps Code Review** dashboard is provisioned via ConfigMap (`monitoring/grafana/configmap-aiops.yml`):

| Panel | Query | Purpose |
|---|---|---|
| Gateway Status | `up{job="aiops"}` | UP/DOWN indicator |
| Pending Requests | `aiops_gateway_pending_requests` | Current in-flight requests |
| Request History | `aiops_gateway_pending_requests` over time | Request patterns and spikes |

Dashboard UID: `borod-aiops`

---

## Security Considerations

| Concern | Mitigation |
|---|---|
| API key exposure | Stored in AWS Secrets Manager, injected via ESO. Zero hardcoded credentials |
| Data sent to API | Only source code diffs and build metrics. No customer data, no secrets, no env vars |
| Data retention | Anthropic API has a [zero data retention policy](https://www.anthropic.com/policies/privacy) — inputs are not used for model training |
| Container security | `runAsNonRoot: true`, `runAsUser: 1000`, `allowPrivilegeEscalation: false` |
| Blast radius | Non-blocking in CI (`catchError`). Gateway failure does not break builds |
| Resource isolation | Dedicated namespace with ResourceQuota and LimitRange |

---

## Why Claude API, Not a Local Model?

The project originally ran **Ollama (qwen2.5-coder:7b)** on a GPU SPOT node (g4dn.xlarge). It was replaced with Claude API for three reasons:

1. **Cost** — GPU node ~$120/month (even on SPOT) vs Claude API Haiku ~$2/month for nightly-only usage
2. **Quality** — a 7B parameter local model produces generic reviews; Claude Haiku catches real logic bugs and security issues
3. **Ops overhead** — no GPU drivers, CUDA, model updates, or OOM debugging to maintain

See [ADR-006: Claude API over Local LLM](../../docs/adr/ADR-006-claude-api-over-local-llm.md) for the full decision record.

---

## Related Files

| Location | Description |
|---|---|
| `borod-bank-ci/Jenkinsfile` | CI pipeline — `AI Code Review` stage (sends diff to gateway) |
| `borod-bank-ci/Jenkinsfile.nightly` | Nightly pipeline — `AI Nightly Analysis` stage (sends metrics summary) |
| `borod-bank-cd/monitoring/grafana/configmap-aiops.yml` | Grafana dashboard for AIOps metrics |
| `borod-bank-cd/argocd/apps/aiops.yml` | ArgoCD Application manifest |
| `docs/adr/ADR-006-claude-api-over-local-llm.md` | Architecture Decision Record |

## Author

**Vlad Vesninsky** — [GitHub](https://github.com/Vlad774)
