# Borod Bank — CD Repository

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.32-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-App%20of%20Apps-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Kubernetes manifests and **ArgoCD App of Apps** configuration for the Borod Bank platform. Manages **18 applications** declaratively via GitOps — Jenkins CI updates the image tag, ArgoCD auto-syncs the cluster with canary deployments.

---

## GitOps Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                         CD / GitOps Flow                           │
│                                                                    │
│  Jenkins CI ──► git push (new image tag in app/deployment.yml)     │
│       │                                                            │
│       ▼                                                            │
│  ArgoCD detects change ──► Auto-sync                               │
│       │                                                            │
│       ▼                                                            │
│  Argo Rollouts (canary deployment)                                 │
│       │  Step 1: 10% traffic ── pause 60s                          │
│       │  Step 2: 50% traffic ── AnalysisTemplate (Prometheus)      │
│       │  Step 3: 100% traffic                                      │
│       │                                                            │
│       ▼  (auto-rollback if error rate > 5%)                        │
│                                                                    │
│  EKS Cluster (Kubernetes 1.32)                                     │
│       ├── Spring Boot App (2-6 replicas, HPA)                      │
│       ├── Prometheus + Grafana (5 dashboards)                      │
│       ├── Jenkins Controller + SonarQube                           │
│       ├── Kyverno (3 policies) + Chaos Mesh                        │
│       └── AIOps (Claude API + FastAPI gateway)                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## ArgoCD — 18 Managed Applications

The `app-of-apps.yml` pattern manages all applications from a single root:

| Category | Applications |
|---|---|
| **Core** | borod-bank, ingress-nginx, ingress, cert-manager, secrets, priority-classes |
| **CI/CD** | jenkins, sonarqube, argo-rollouts |
| **Monitoring** | monitoring (Prometheus + Grafana + Pushgateway), efs |
| **Security** | kyverno, external-secrets, policy-controller, falco |
| **Advanced** | chaos-mesh, cluster-autoscaler, aws-load-balancer-controller, aiops |
| **Observability** | loki, kubecost, keda, nvidia-device-plugin |

---

## Repository Structure

```
borod-bank-cd/
├── namespace.yml                          # borod-bank namespace
│
├── app/                                   # Spring Boot application
│   ├── rollout.yml                        #   Argo Rollout (canary 10→50→100%)
│   ├── analysis-template.yml              #   Prometheus error-rate check
│   ├── deployment.yml                     #   Fallback deployment spec
│   ├── service.yml                        #   ClusterIP service
│   ├── hpa.yaml                           #   HPA (min 2, max 6)
│   ├── pdb.yaml                           #   Pod Disruption Budget
│   ├── network-policy.yml                 #   Network isolation
│   ├── externalsecret-mysql.yml           #   ESO → AWS Secrets Manager
│   ├── feature-flags-configmap.yml        #   Runtime feature toggles
│   ├── limitrange.yaml                    #   Container resource defaults
│   ├── resourcequota.yaml                 #   Namespace resource limits
│   └── kustomization.yaml                 #   Kustomize overlay
│
├── argocd/                                # ArgoCD configuration
│   ├── app-of-apps.yml                    #   Root application (manages all)
│   ├── apps/                              #   23 ArgoCD Application manifests
│   ├── notifications-configmap.yaml       #   Slack + drift detection
│   └── externalsecret-argocd-notifications.yml
│
├── monitoring/                            # Observability stack
│   ├── prometheus/                        #   Prometheus server + config
│   ├── grafana/                           #   Grafana + 5 dashboards (ConfigMap)
│   ├── pushgateway/                       #   Metrics sink for nightly pipeline
│   ├── node-exporter/                     #   Host-level metrics (DaemonSet)
│   ├── kube-state-metrics/                #   Kubernetes object metrics
│   ├── namespace.yml
│   ├── network-policy.yml
│   ├── limitrange.yaml
│   └── resourcequota.yaml
│
├── jenkins/                               # Jenkins controller
│   ├── deployment.yml                     #   Jenkins pod (custom image)
│   ├── pvc.yml                            #   20Gi persistent volume
│   ├── service.yml                        #   Web (8080) + Agent (50000)
│   ├── rbac.yml                           #   Pod creation for agents
│   ├── pdb.yaml
│   ├── namespace.yml
│   ├── limitrange.yaml
│   └── resourcequota.yaml
│
├── sonarqube/                             # SonarQube code quality
│   ├── deployment.yml
│   ├── persistentvolumeclaim.yml          #   Persistent data
│   ├── service.yml
│   ├── setup-job.yml                      #   Auto-configure webhooks/tokens
│   ├── secret.yml                         #   Local config secret
│   ├── rbac.yml
│   ├── pdb.yaml
│   ├── namespace.yml
│   ├── limitrange.yaml
│   └── resourcequota.yaml
│
├── ingress/                               # Ingress & TLS
│   ├── ingress-borod-bank.yml             #   app.vladinnovations.info
│   ├── ingress-jenkins.yml                #   jenkins.vladinnovations.info
│   ├── ingress-argocd.yml                 #   argocd.vladinnovations.info
│   ├── ingress-grafana.yml                #   grafana.vladinnovations.info
│   └── cluster-issuer.yml                 #   Let's Encrypt (cert-manager)
│
├── kyverno/                               # Policy enforcement
│   └── policies/
│       ├── require-resource-limits.yaml
│       ├── disallow-latest-tag.yaml
│       └── require-readonly-root.yaml
│
├── chaos/                                 # Chaos engineering
│   ├── pod-kill.yaml
│   ├── network-delay.yaml
│   ├── stress-cpu.yaml
│   └── namespace.yaml
│
├── aiops/                                 # AI-powered operations
│   ├── gateway-deployment.yaml            #   FastAPI + Claude API
│   ├── log-analyzer-cronjob.yaml          #   Nightly AI analysis
│   ├── analyzer-configmap.yaml
│   ├── externalsecret-anthropic.yaml      #   API key from Secrets Manager
│   ├── externalsecret-slack.yaml
│   ├── namespace.yaml
│   ├── limitrange.yaml
│   └── resourcequota.yaml
│
├── cosign/                                # Supply chain security
│   └── clusterimagepolicy.yaml            #   Image signature verification
│
├── secrets/                               # Cluster-wide secret config
│   ├── cluster-secret-store.yml           #   ESO → AWS Secrets Manager
│   ├── externalsecret-jenkins.yml
│   ├── externalsecret-sonarqube.yml
│   ├── externalsecret-cosign.yml
│   ├── externalsecret-falco.yml
│   └── storageclass-gp2-csi.yml
│
├── rbac/                                  # RBAC for Jenkins
│   ├── service-account.yml
│   ├── role.yml
│   ├── role-binding.yml
│   └── cluster-role.yml
│
├── cluster-autoscaler/                    # Cluster Autoscaler
│   ├── deployment.yml
│   ├── rbac.yml
│   └── sa.yml
│
├── efs/                                   # EFS storage (Maven cache)
│   ├── storageclass.yml
│   └── pvc-maven-cache.yml
│
├── priority-classes/                      # Pod scheduling priorities
│   ├── jenkins-build.yml
│   └── low-priority.yml
│
└── LICENSE
```

---

## Canary Deployments

Deployments use **Argo Rollouts** with a Prometheus-based `AnalysisTemplate`:

1. **10% traffic** → pause 60 seconds
2. **50% traffic** → run AnalysisTemplate (query Prometheus for HTTP 5xx rate)
3. **100% traffic** → promotion complete

If the error rate exceeds **5%** during analysis, the rollout **automatically rolls back** to the previous stable version.

---

## Secret Management

All secrets are managed through **External Secrets Operator (ESO)**:

```
AWS Secrets Manager  ──►  ClusterSecretStore  ──►  ExternalSecret  ──►  K8s Secret
```

No hardcoded secrets in manifests. ESO syncs credentials for: MySQL, Jenkins, SonarQube, ArgoCD, Cosign, Anthropic API, Slack, and Falco.

---

## Related Repositories

| Repository | Description |
|---|---|
| [borod-bank-ci](https://github.com/Synaptix-Innovations/borod-bank-ci) | Spring Boot app + Jenkins CI pipeline |
| [borod-bank-infra](https://github.com/Synaptix-Innovations/borod-bank-infra) | Terraform IaC for AWS EKS |
| [jenkins-setup](https://github.com/Synaptix-Innovations/jenkins-setup) | Custom Jenkins controller image |
| [borod-bank-docs](https://github.com/Synaptix-Innovations/borod-bank-docs) | Project documentation and screenshots |

## Author

**Vlad Vesninsky** — [GitHub](https://github.com/Vlad774)
