# MovieMonster Infrastructure

Infrastructure and deployment configuration for the MovieMonster application stack.

This repository defines the Docker Compose runtime for the MovieMonster backend,
frontend, and PostgreSQL database, plus GitHub Actions workflows for staging and
production deployments.

## Stack

| Service | Description | Default Port |
| --- | --- | --- |
| `postgres` | PostgreSQL 17 database with a persistent Docker volume | `5432` |
| `backend` | Spring backend image published as `jbaldwin215/moviemonster-backend:latest` | `8080` |
| `frontend` | Frontend image built from `../MovieMonsterFrontend` and tagged as `jbaldwin215/moviemonster-frontend:latest` | `5000` |

All services run on the shared `moviemonster-net` Docker network. Database data
is stored in the `moviemonster_postgres_data` volume - soon going to migrate to a PostgreSQL VM, with backups.

## Repository Layout

```text
.
├── compose.yaml                         # Docker Compose stack definition
├── DEPLOYMENT.md                        # Deployment setup notes
├── backend/backend.env.example          # Backend environment example
├── frontend/frontend.env.example        # Frontend environment example
├── postgres/db.env.example              # PostgreSQL environment example
└── .github/workflows/
    ├── deploy.yml                       # Reusable deployment workflow
    ├── deploy-staging.yml               # Staging deployment trigger
    └── deploy-production.yml            # Production deployment trigger
```

## Prerequisites

- Docker Engine with the Docker Compose plugin
- Access to the backend image: `jbaldwin215/moviemonster-backend:latest`
- The frontend source repository checked out beside this repository as
  `../MovieMonsterFrontend` when building the frontend image locally
- Environment values for database credentials, JWT signing, TMDB access, S3, and
  CORS configuration

## Local Usage

Create a `.env` file at the repository root before starting the stack. The
example files under `backend/`, `frontend/`, and `postgres/` document the values
used by `compose.yaml`.

Example:

```bash
cp postgres/db.env.example .env
cat backend/backend.env.example >> .env
cat frontend/frontend.env.example >> .env
```

Review `.env` and replace placeholder secrets before running the stack.

Start the services:

```bash
docker compose up -d --build
```

Check service status:

```bash
docker compose ps
```

Stop the stack:

```bash
docker compose down
```

To remove the database volume as well:

```bash
docker compose down -v
```

## Configuration

The Compose file reads configuration from environment variables and provides
development-oriented defaults where possible.

| Variable | Purpose |
| --- | --- |
| `POSTGRES_USER` | PostgreSQL username |
| `POSTGRES_PASSWORD` | PostgreSQL password |
| `POSTGRES_DB` | PostgreSQL database name |
| `SPRING_PROFILES_ACTIVE` | Active Spring profile for the backend |
| `LOCAL_JDBC_DATABASE_URL` | Backend JDBC connection URL |
| `LOCAL_JDBC_DATABASE_USERNAME` | Backend database username |
| `LOCAL_JDBC_DATABASE_PASSWORD` | Backend database password |
| `LOCAL_JWT_SECRET` | JWT signing secret |
| `LOCAL_TMDB_KEY` | TMDB API key |
| `LOCAL_S3_ACCESS_KEY` | S3 access key |
| `LOCAL_S3_SECRET_KEY` | S3 secret key |
| `LOCAL_S3_BUCKET_NAME` | S3 bucket name |
| `LOCAL_APP_CORS_ALLOWED_ORIGINS` | Allowed frontend origins for backend CORS |
| `FRONTEND_API_ORIGIN` | API origin baked into the frontend build |

Do not commit real environment files or production secrets. The repository
ignores `.env`, `backend/backend.env`, `frontend/frontend.env`, and
`postgres/db.env`.

## Deployment

Deployments run through GitHub Actions using self-hosted runners:

- Staging deploys on pushes to the `staging` branch and can also be run manually.
- Production deploys manually through the `Deploy Production` workflow.
- The reusable deploy workflow runs on the environment-specific self-hosted
  runner label, writes a `.env` file in `DEPLOY_PATH`, validates the Compose
  configuration, pulls images, starts the stack, waits for health checks, and
  prunes unused images.

Each GitHub environment should provide the variables and secrets consumed by
`.github/workflows/deploy.yml`.

Environment variables:

| Variable | Description |
| --- | --- |
| `DEPLOY_PATH` | Absolute path to the server directory containing `compose.yaml` |
| `FRONTEND_API_ORIGIN` | Public API origin used by the frontend |
| `LOCAL_APP_CORS_ALLOWED_ORIGINS` | Origins allowed by the backend CORS config |
| `LOCAL_JDBC_DATABASE_URL` | JDBC URL used by the backend |
| `LOCAL_JDBC_DATABASE_USERNAME` | Backend database username |
| `LOCAL_S3_BUCKET_NAME` | S3 bucket used by the backend |
| `POSTGRES_DB` | PostgreSQL database name |
| `POSTGRES_USER` | PostgreSQL username |
| `SPRING_PROFILES_ACTIVE` | Spring profile for the deployment |

Environment secrets:

| Secret | Description |
| --- | --- |
| `LOCAL_JDBC_DATABASE_PASSWORD` | Backend database password |
| `LOCAL_JWT_SECRET` | JWT signing secret |
| `LOCAL_S3_ACCESS_KEY` | S3 access key |
| `LOCAL_S3_SECRET_KEY` | S3 secret key |
| `LOCAL_TMDB_KEY` | TMDB API key |
| `POSTGRES_PASSWORD` | PostgreSQL password |

Production should be protected with required reviewers in the GitHub environment
settings.

## Operational Notes

- PostgreSQL readiness is checked with `pg_isready` before the backend starts.
- The backend uses the Compose service name `postgres` for local database
  connectivity.
- The frontend service has both an image tag and a build context. Deployment
  hosts should either have access to a published frontend image or have the
  sibling `MovieMonsterFrontend` build context available.
