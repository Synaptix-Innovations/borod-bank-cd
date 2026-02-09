# Borod Bank - CD Repository

[![Kubernetes](https://img.shields.io/badge/kubernetes-1.28+-blue)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-managed-green)](https://argo-cd.readthedocs.io/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Kubernetes manifests and ArgoCD configuration for the Borod Bank application. Managed via GitOps — Jenkins CI updates the image tag, ArgoCD auto-syncs the cluster.

```
┌─────────────────────────────────────────────────────────┐
│                   CD / GitOps FLOW                      │
│                                                         │
│  Jenkins CI ──► Updates image tag in app/deployment.yml │
│       │                                                 │
│       ▼                                                 │
│  Git Push ──► ArgoCD detects change                     │
│                    │                                    │
│                    ▼                                    │
│              Auto-sync to EKS cluster                   │
│                    │                                    │
│          ┌─────────┴─────────┐                          │
│          ▼                   ▼                          │
│    borod-bank app       MySQL database                  │
│    (2 replicas)         (1 replica)                     │
│    port: 8080           port: 3306                      │
│          │                   │                          │
│          └─────────┬─────────┘                          │
│                    ▼                                    │
│           LoadBalancer :80                              │
└─────────────────────────────────────────────────────────┘
```

## Repository Structure

```
borod-bank-cd/
├── namespace.yml              # borod-bank namespace
├── app/
│   ├── deployment.yml         # Spring Boot app (2 replicas)
│   └── service.yml            # LoadBalancer on port 80
├── database/
│   ├── secret.yml             # MySQL credentials (base64)
│   ├── deployment.yml         # MySQL 8 (1 replica)
│   └── service.yml            # ClusterIP on port 3306
├── rbac/
│   ├── service-account.yml    # Jenkins service account
│   ├── role.yml               # Namespace-scoped permissions
│   ├── role-binding.yml       # Bind role to SA
│   └── cluster-role.yml       # Cluster-wide PV access
└── argocd/
    └── application.yml        # ArgoCD Application resource
```

## Quick Start

```bash
# Apply namespace first
kubectl apply -f namespace.yml

# Apply secrets, then database, then app
kubectl apply -f database/secret.yml
kubectl apply -f database/
kubectl apply -f app/

# (Optional) Apply RBAC for Jenkins
kubectl apply -f rbac/

# (Optional) Apply ArgoCD application
kubectl apply -f argocd/application.yml
```

## Security

- Database credentials stored in Kubernetes Secrets (base64-encoded)
- Application reads credentials via `secretKeyRef`
- RBAC configured with least-privilege principle

## Related Repositories

| Repository | Description |
|------------|-------------|
| [borod-bank-ci](https://github.com/Vlad774/borod-bank-ci) | Spring Boot app + Jenkins pipeline |
| [borod-bank-infra](https://github.com/Vlad774/borod-bank-infra) | Terraform for AWS EKS |

## Author

**Vlad Vesninsky** - [GitHub](https://github.com/Vlad774)
