---
name: k8s-workflow
description: >-
  Kubernetes core workflow rules, manifest conventions, security best practices,
  and version considerations. Use for general Kubernetes operations and manifest authoring.
---

# Kubernetes Workflow Rules

## 1. Resource Naming

### Conventions

| Element | Convention | Example |
| --- | --- | --- |
| Namespace | lowercase kebab | `myapp-prod`, `myapp-dev` |
| Deployment | `{app}-{component}` | `myapp-api`, `myapp-worker` |
| Service | Same as Deployment | `myapp-api` |
| ConfigMap | `{app}-{purpose}-config` | `myapp-api-app-config` |
| Secret | `{app}-{purpose}-secret` | `myapp-api-db-secret` |
| Ingress | `{app}-ingress` | `myapp-api-ingress` |
| CronJob | `{app}-{action}` | `myapp-cleanup-logs` |

---

## 2. Labels and Annotations

### Required Labels

```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp-api
    app.kubernetes.io/component: backend
    app.kubernetes.io/part-of: myapp
    app.kubernetes.io/managed-by: kubectl
    app.kubernetes.io/version: "1.2.0"
    environment: production
```

### Label Usage

| Label | Purpose |
| --- | --- |
| `app.kubernetes.io/name` | Application name (used in selectors) |
| `app.kubernetes.io/component` | Component role (backend, frontend, worker, db) |
| `app.kubernetes.io/part-of` | Parent application group |
| `environment` | Deployment environment (dev, staging, production) |

---

## 3. Resource Limits

### Always Set requests and limits

```yaml
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### Sizing Guidelines

| Workload Type | CPU Request | CPU Limit | Memory Request | Memory Limit |
| --- | --- | --- | --- | --- |
| API server | 100m–500m | 500m–2000m | 256Mi–512Mi | 512Mi–1Gi |
| Background worker | 50m–200m | 200m–1000m | 128Mi–256Mi | 256Mi–512Mi |
| CronJob | 50m–100m | 200m–500m | 128Mi–256Mi | 256Mi–512Mi |

- Set `requests` based on steady-state usage
- Set `limits` at 2–4x requests for burst capacity
- Always set memory limits to prevent OOM kills affecting other pods

---

## 4. Environment-Specific Management

### Directory Structure (Kustomize)

```text
k8s/
├── base/                  # Common manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   └── prod/
│       ├── kustomization.yaml
│       └── patches/
```

### Environment Differences

| Config | Dev | Staging | Prod |
| --- | --- | --- | --- |
| Replicas | 1 | 2 | 3+ |
| Resource limits | Low | Medium | Full |
| Log level | DEBUG | INFO | INFO |
| Ingress TLS | Optional | Yes | Yes |

---

## 5. Security Best Practices

### Pod Security

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

### Secrets Management

- Never store plaintext secrets in manifests
- Use `SealedSecret`, `ExternalSecret`, or cloud provider secret manager
- Reference secrets via `secretKeyRef`, not environment variable literals
- Rotate secrets regularly without pod restart (use volume mounts for auto-reload)

---

## 6. Health Checks

### Always Define Probes

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30
```

### Probe Guidelines

| Probe | Purpose | Failure Action |
| --- | --- | --- |
| Startup | App initialization complete | Kill and restart pod |
| Liveness | App still running | Kill and restart pod |
| Readiness | App ready for traffic | Remove from Service endpoints |

---

## 7. Version Support (2026-03-14)

### Kubernetes Release Cadence

Kubernetes releases a new minor version approximately every 4 months.

### Current Versions

| Version | Status | Latest Patch | End of Support |
| --- | --- | --- | --- |
| 1.35 | Supported | 1.35.2 | Dec 2026 |
| 1.34 | Supported | 1.34.5 | Feb 2027 |
| 1.33 | Supported | 1.33.9 | Jun 2026 |
| 1.32 | EOL | 1.32.13 | Feb 2026 |
| 1.31 | EOL | 1.31.14 | Dec 2025 |

### Key Features in Recent Versions

#### Kubernetes 1.35 (Dec 2025)

- **In-Place Pod Resize (GA)**: CPU/memory adjustments without pod restart
- **PreferSameNode Traffic Distribution**: Local endpoint priority for reduced latency
- **Image Volumes**: Deliver data artifacts using OCI container images

#### Kubernetes 1.34 (Aug 2025)

- **Finer-grained Authorization**: Field/label selector-based decisions
- **Anonymous Request Restrictions**: Limit unauthenticated access to specific endpoints
- **Ordered Namespace Deletion**: Structured deletion order for security

#### Kubernetes 1.33 (Apr 2025)

- **Sidecar Containers (GA)**: Native sidecar lifecycle management
- **Recursive Read-Only Mounts (GA)**: Full read-only volume mounts
- **Bound ServiceAccount Token Improvements**: Unique token IDs, node binding

### Upgrade Planning

1. **Review deprecation notices** before each minor version upgrade
2. **Test in non-production** environments first
3. **Check API compatibility** using `kubectl deprecations` or `pluto`
4. **Update CRDs and webhooks** before control plane upgrade
5. **Node pools**: Drain and upgrade incrementally

---

## 8. Managed Kubernetes Services

For provider-specific managed Kubernetes services, see dedicated skills:

- **AWS EKS**: `k8s-providers` 규칙의 EKS 섹션
- **Azure AKS**: `k8s-providers` 규칙의 AKS 섹션
- **GCP GKE**: `k8s-providers` 규칙의 GKE 섹션

### Quick Comparison

| Feature | EKS | AKS | GKE |
| --- | --- | --- | --- |
| Latest K8s | 1.35 | 1.34 | 1.35 |
| Control Plane | Free | Free | Free |
| Node Autoscaling | Karpenter/MNG | Cluster Autoscaler | GKE Autopilot |
| Service Mesh | App Mesh (deprecated) | Istio (add-on) | Anthos Service Mesh |
| Secrets | AWS Secrets Manager | Azure Key Vault | Secret Manager |

---

## 9. Anti-Patterns

- Using `latest` tag for container images — always pin to specific version
- Defining resources without `requests`/`limits`
- Running containers as root
- Hardcoding config values in manifests — use ConfigMap/Secret
- Single replica for production workloads
- Missing health check probes
- Using `NodePort` in production — use `ClusterIP` + Ingress
- Ignoring deprecation warnings before upgrades
- Not testing upgrades in non-production first
