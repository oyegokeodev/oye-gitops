# AI Copilot Instructions for OyeGitOps

## Project Overview
OyeGitOps is a **Kubernetes GitOps deployment repository** that uses ArgoCD to manage two primary services:
- **Grafana** (monitoring & dashboards) - PostgreSQL-backed with HA setup
- **n8n** (automation engine) - workflow orchestration

The architecture follows GitOps principles: YAML configurations in this repo define the desired state, and ArgoCD automatically synchronizes them to the cluster.

## Core Architecture

### Two-Layer Application Structure
1. **Root Application** (`internal-psi-monitoring/root-app.yaml`): ArgoCD Application that watches the `applications/` directory
2. **Child Applications** (`internal-psi-monitoring/applications/{grafana,n8n}/`): Individual service deployments

**Key Pattern**: Each application uses `sources` with dual repositories:
- External Helm chart repo (grafana/helm-charts, 8gears/n8n-helm-chart)
- Local GitOps repo (`https://github.com/oyegokeodev/oye-gitops.git`) for value file overrides

### Critical Configuration Files by Service

**Grafana** (`internal-psi-monitoring/applications/grafana/`):
- `application.yaml`: ArgoCD configuration pulling from Grafana's Helm repo
- `values.yaml`: Helm overrides including database connection, HA setup (2 replicas), headless service enablement
- **HA-Critical Settings**: `headlessService: true` (required for DNS peer discovery), `max_open_conn: 3` per replica (stays under 20-conn DB limit), `unified_alerting: enabled` (prevents duplicate alerts across replicas)

**n8n** (`internal-psi-monitoring/applications/n8n/`):
- `n8n-application.yaml`: ArgoCD config pulling from 8gears' Helm repo
- `values-n8n.yaml`: Minimal setup with basic auth, ClusterIP service, disabled persistence

## Developer Workflows

### Common Tasks
1. **Update Service Configuration**: Edit `applications/{service}/values-{service}.yaml` → commit → ArgoCD auto-syncs
2. **Scale Replicas**: Modify `replicas:` in values.yaml (note: Grafana HA has specific connection pool limits)
3. **Change Database/External Secrets**: Update corresponding environment variables in `grafana.ini` (use `$__env{VAR}` syntax)
4. **Deploy New Service**: Create `applications/{service}/` directory with `{service}-application.yaml` and `values-{service}.yaml`

### Key Commands
```bash
# View sync status (run on cluster)
kubectl get applications -n argocd

# Force ArgoCD sync
argocd app sync internal-psi-monitoring

# Check Grafana replication status
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -f
```

## Project-Specific Patterns & Conventions

### Secret Management
- Store passwords in Kubernetes Secrets (e.g., `my-grafana-admin-secret`, `grafana-db`)
- Reference via `envValueFrom` (env var) or `existingSecret` fields in values.yaml
- Example: Grafana maps `GF_DATABASE_PASSWORD` env var from Secret key
- **ArgoCD Secret Sync**: Add `ignoreDifferences` to skip Secret fields that have different encodings between desired and actual state (e.g., `admin-password`). This prevents false sync conflicts when password hashing differs

### HA Configuration (Grafana)
- Always enable `headlessService: true` for DNS-based peer discovery
- Set `max_open_conn` based on replica count and database limits (current: 3 conns × 2 replicas = 15/20 DB limit)
- Use `session.provider: database` to ensure session consistency across replicas
- Enable `unified_alerting` to prevent duplicate notifications

### Value File Overrides
- Use `$myrepo/` reference in `sources[].helm.valueFiles` to access local values
- Split values into minimal defaults vs. environment-specific overrides

### Namespace Management
- Each service gets its own namespace (`monitoring`, `n8n`)
- Root app in `argocd` namespace
- Auto-create namespaces via `syncOptions: [CreateNamespace=true]`

## Integration Points & External Dependencies

- **Grafana**: PostgreSQL DB (Aiven Cloud) at `pg-1e155c21-oyegokeodev-f11d.f.aivencloud.com:20109`
- **ArgoCD**: Manages all deployments, watches this repo for changes
- **Helm Chart Repositories**: 
  - `https://github.com/grafana/helm-charts.git` (main branch)
  - `https://github.com/8gears/n8n-helm-chart.git` (master branch)
- **GitHub Repository**: `https://github.com/oyegokeodev/oye-gitops.git` (source of truth)

## Common Pitfalls & Solutions

| Issue | Root Cause | Solution |
|-------|-----------|----------|
| Grafana DNS "server misbehaving" errors | `headlessService: false` | Enable `headlessService: true` in values.yaml |
| "remaining connection slots reserved" DB error | Too many concurrent connections | Reduce `max_open_conn` per replica (check formula: `max_conn * replicas < DB_limit`) |
| Duplicate Grafana alerts in HA mode | Unified alerting disabled | Set `unified_alerting.enabled: true` |
| ArgoCD not syncing changes | Repo URL mismatch or branch wrong | Verify `repoURL` and `targetRevision` (HEAD or specific branch) |

## When Adding New Applications

1. Create `applications/{app-name}/` directory
2. Create `{app-name}-application.yaml` with proper `sources` structure (external Helm repo + local overrides)
3. Create `values-{app-name}.yaml` with minimal, production-safe defaults
4. Add `CreateNamespace=true` to syncPolicy to auto-create namespace
5. Test: `kubectl apply -f root-app.yaml` and verify ArgoCD picks up the new app
