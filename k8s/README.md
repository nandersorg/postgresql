## Kubernetes Deployment Files

### File Overview

- **namespace.yaml**: Creates the `postgresql` namespace for isolation
- **postgres.yaml**: Main deployment with:
  - PersistentVolumeClaim (10Gi storage)
  - Deployment with PostgreSQL container
  - Service for accessing the database
  - References secrets for passwords (created by GitHub Actions)
- **postgres-init.yaml**: ConfigMap with initialization shell script that:
  - Creates schemas (airflow, ml_platform)
  - Creates tables with proper indexes
  - Creates database users
  - **Sets passwords from environment variables** (secure!)
  - Grants schema-specific permissions

### Security Architecture

**Why this is safe for public repos:**

1. **Passwords never in git**: The `postgres-secret` is created by GitHub Actions from GitHub Secrets
2. **GitHub Secrets are masked**: Logs don't expose password values
3. **Environment variable isolation**: Init script reads passwords from pod environment (set by Kubernetes Secret)
4. **No SQL password strings**: Passwords only appear in GitHub Actions logs (redacted by GitHub)

**Data flow:**
```
GitHub Secrets (POSTGRES_PASSWORD, etc.)
  ↓
GitHub Actions Workflow (deploy.yaml)
  ↓
kubectl create secret (--from-literal creates K8s Secret)
  ↓
Kubernetes Secret: postgres-secret
  ↓
Pod Environment Variables (set by Deployment)
  ↓
Init Script (01-init-schemas.sh uses $AIRFLOW_DB_PASSWORD, etc.)
```

### Deployment Steps

```bash
# 1. Ensure namespace exists
microk8s kubectl apply -f namespace.yaml

# 2. GitHub Actions creates Secret (automatic)
# Or manually for testing:
microk8s kubectl create secret generic postgres-secret \
  --from-literal=POSTGRES_PASSWORD='<POSTGRES_PASSWORD_FROM_GITHUB_SECRET>' \
  --from-literal=AIRFLOW_DB_PASSWORD='<AIRFLOW_DB_PASSWORD_FROM_GITHUB_SECRET>' \
  --from-literal=ML_PLATFORM_DB_PASSWORD='<ML_PLATFORM_DB_PASSWORD_FROM_GITHUB_SECRET>' \
  --namespace=postgresql \
  --dry-run=client -o yaml | microk8s kubectl apply -f -

# 3. Apply init ConfigMap
microk8s kubectl apply -f postgres-init.yaml

# 4. Apply Deployment
microk8s kubectl apply -f postgres.yaml

# 5. Wait for readiness
microk8s kubectl wait --for=condition=available --timeout=5m \
  deployment/postgres -n postgresql
```

### Troubleshooting

**Check if Secret exists:**
```bash
microk8s kubectl get secret -n postgresql
```

**View init script logs:**
```bash
microk8s kubectl logs -n postgresql deployment/postgres
```

**Test connection:**
```bash
microk8s kubectl port-forward -n postgresql svc/postgres 5432:5432
psql -h localhost -U postgres
# Enter password from POSTGRES_PASSWORD secret
```

**Verify schemas and users:**
```bash
psql -U postgres
\dn                                    # List schemas
SELECT * FROM pg_user;                 # List users
GRANT USAGE ON SCHEMA airflow TO airflow;  # Verify grants
```
