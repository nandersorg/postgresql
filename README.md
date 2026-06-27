# PostgreSQL Data Store

Central PostgreSQL database for storing data from multiple projects:
- **airflow**: News sentiment analysis results
- **ml_platform**: Model metadata, training runs, predictions
- **portfolio**: Projects, skills, etc.

## Quick Start

```bash
# Deploy to MicroK8s
microk8s kubectl apply -f k8s/namespace.yaml
microk8s kubectl apply -f k8s/postgres.yaml

# Check status
microk8s kubectl get pods -n postgresql
microk8s kubectl get svc -n postgresql

# Port-forward to connect locally
microk8s kubectl port-forward -n postgresql svc/postgres 5432:5432
```

Then connect with:
```bash
psql -h localhost -U postgres -d postgres
# Password is set in k8s/postgres.yaml (change it!)
```

## Schemas

Each project has its own schema:

### `airflow` schema
```sql
CREATE SCHEMA airflow;

CREATE TABLE airflow.articles (
  id SERIAL PRIMARY KEY,
  title TEXT NOT NULL,
  description TEXT,
  url TEXT,
  sentiment VARCHAR(10) NOT NULL,  -- POSITIVE or NEGATIVE
  scraped_at TIMESTAMP NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX airflow_articles_scraped_at 
  ON airflow.articles(scraped_at DESC);
```

### `ml_platform` schema
```sql
CREATE SCHEMA ml_platform;

CREATE TABLE ml_platform.models (
  id SERIAL PRIMARY KEY,
  name TEXT NOT NULL UNIQUE,
  version TEXT,
  framework TEXT,
  path TEXT,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE ml_platform.training_runs (
  id SERIAL PRIMARY KEY,
  model_id INTEGER REFERENCES ml_platform.models(id),
  epoch INTEGER,
  loss FLOAT,
  accuracy FLOAT,
  timestamp TIMESTAMP DEFAULT NOW()
);
```

## Connection Details

**Service:** `postgres.postgresql.svc.cluster.local:5432`
**Database:** `postgres`
**User:** `postgres`
**Password:** See k8s/postgres.yaml (or set via secret)

## Backups

PostgreSQL data is stored in a PVC. For backups:

```bash
# Dump database
microk8s kubectl exec -n postgresql postgres-0 -- \
  pg_dump -U postgres postgres > backup.sql

# Restore from dump
microk8s kubectl exec -i -n postgresql postgres-0 -- \
  psql -U postgres < backup.sql
```

## Pre-commit hooks

```bash
pre-commit install
pre-commit run --all-files
```
