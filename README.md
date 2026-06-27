# PostgreSQL Data Store

Central PostgreSQL database for storing data from multiple projects:
- **airflow**: News sentiment analysis results
- **ml_platform**: Model metadata, training runs, predictions
- **portfolio**: Projects, skills, etc.

## ⚠️ Security Setup (REQUIRED for Public Repo)

Before deploying, you **must** add these secrets to your GitHub repository:

### GitHub Repository Secrets

Navigate to **Settings → Secrets and variables → Actions** and add:

| Secret Name | Description | Example |
|---|---|---|
| `POSTGRES_PASSWORD` | Password for postgres superuser | Generate with `openssl rand -base64 24` |
| `AIRFLOW_DB_PASSWORD` | Password for airflow schema user | Generate with `openssl rand -base64 24` |
| `ML_PLATFORM_DB_PASSWORD` | Password for ml_platform schema user | Generate with `openssl rand -base64 24` |

**Generate secure passwords:**
```bash
openssl rand -base64 24
# Output: example: xK9mPqL2wN8vR4jH7bT3sF5uC6dE
```

These secrets are **only accessible** to GitHub Actions and are **never** stored in the repository.

## Quick Start

### 1. Add secrets to GitHub
See "Security Setup" section above.

### 2. Deploy to MicroK8s
Push to main branch or trigger workflow manually:

```bash
git push origin main
# or manually trigger:
# GitHub UI → Actions → Deploy PostgreSQL → Run workflow
```

### 3. Verify deployment
```bash
microk8s kubectl get pods -n postgresql
microk8s kubectl get svc -n postgresql
```

### 4. Connect locally
```bash
microk8s kubectl port-forward -n postgresql svc/postgres 5432:5432

# In another terminal:
psql -h localhost -U postgres
# Password: from POSTGRES_PASSWORD secret
```

## Schemas

Each project has its own schema with dedicated user (least privilege):

### `airflow` schema
```sql
CREATE SCHEMA airflow;

CREATE TABLE airflow.articles (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  description TEXT,
  url TEXT,
  sentiment VARCHAR(10) NOT NULL CHECK (sentiment IN ('POSITIVE', 'NEGATIVE')),
  scraped_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_airflow_articles_scraped_at
  ON airflow.articles(scraped_at DESC);
```

**User:** `airflow` (can only access airflow schema)

### `ml_platform` schema
```sql
CREATE SCHEMA ml_platform;

CREATE TABLE ml_platform.models (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  version TEXT,
  framework TEXT,
  path TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ml_platform.training_runs (
  id SERIAL PRIMARY KEY,
  model_id INTEGER REFERENCES ml_platform.models(id),
  epoch INTEGER,
  loss FLOAT,
  accuracy FLOAT,
  val_loss FLOAT,
  val_accuracy FLOAT,
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_ml_platform_training_runs_model_id
  ON ml_platform.training_runs(model_id);
```

**User:** `ml_platform` (can only access ml_platform schema)

## Connection Strings

**Inside Kubernetes cluster** (use actual password from secret):
```
# pragma: allowlist secret
postgresql://airflow:<AIRFLOW_PASSWORD>@postgres.postgresql.svc.cluster.local:5432/postgres
postgresql://ml_platform:<ML_PLATFORM_PASSWORD>@postgres.postgresql.svc.cluster.local:5432/postgres
```

**From outside** (after port-forward, use actual password from secret):
```
# pragma: allowlist secret
postgresql://postgres:<POSTGRES_PASSWORD>@localhost:5432/postgres
```

## Backups

Backup the database:
```bash
microk8s kubectl exec -n postgresql deployment/postgres -- \
  pg_dump -U postgres postgres > backup_$(date +%Y%m%d_%H%M%S).sql
```

Restore from backup:
```bash
microk8s kubectl exec -i -n postgresql deployment/postgres -- \
  psql -U postgres < backup.sql
```

## File Structure

```
k8s/
├── namespace.yaml          # PostgreSQL namespace
├── postgres.yaml           # Deployment, PVC, Service (passwords from secrets)
├── postgres-init.yaml      # ConfigMap with schema initialization SQL
└── README.md

.github/workflows/
└── deploy.yaml            # GitHub Actions workflow (manages secrets)
```

## Pre-commit Hooks

```bash
pre-commit install
pre-commit run --all-files
```

This validates YAML syntax and prevents secret commits.
