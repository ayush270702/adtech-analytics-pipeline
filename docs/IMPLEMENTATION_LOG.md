# Implementation Log — Media Ads Intelligence Platform

**Document purpose:** A practical, command-by-command record of the project setup and implementation. This is the working runbook to use alongside `docs/PROJECT_DOCUMENTATION_V1.md`.

- **Repository:** `ayush270702/adtech-analytics-pipeline`
- **Working directory:** `D:\\adtech-analytics-pipeline`
- **Default branch:** `main`
- **Approach:** Build incrementally. Create folders and files only when a milestone needs them; verify each milestone before proceeding.
- **Last updated:** 2026-11-05
- **Current progress:** Phase 1 — Project Foundation; PostgreSQL is running, project schemas and the first two source tables are created and seeded.

---

## How to maintain this document

After each verified milestone:

1. Add the step number, purpose, and completion date.
2. Record the exact commands or SQL that were used.
3. Explain what the commands do and what successful output looks like.
4. Record any errors and the fix that resolved them.
5. Update the progress checklist and the next planned step.
6. Commit and push the documentation update to GitHub.

Do not mark a step complete until its verification succeeds.

## Progress tracker

| Step | Milestone | Status |
|---|---|---|
| 1.1 | Verify local development tools | Complete |
| 1.2 | Clone and verify Git repository | Complete |
| 1.3 | Create PostgreSQL Docker Compose configuration | Complete |
| 1.4 | Start PostgreSQL container | Complete |
| 1.5 | Connect to PostgreSQL and verify database | Complete |
| 1.6 | Create project schemas | Complete |
| 1.7 | Create source business tables (`clients`, `platforms`) | Complete |
| 1.8 | Create remaining source tables | Not started |
| 1.9 | Create operational metadata tables | Not started |
| 1.10 | Verify database structure | Not started |


---

# Phase 1 — Project Foundation

## Step 1.1 — Verify the development environment

**Purpose:** Confirm that the local machine has the tools needed for development and containerized services.

Commands run in Windows PowerShell:

```powershell
docker --version
docker compose version
wsl --status
git --version
python --version
java -version
```

Versions observed during setup:

| Tool | Observed version / configuration |
|---|---|
| Docker CLI | 29.0.1 |
| Docker Compose | v2.40.3-desktop.1 |
| WSL | Default version 2 |
| Default WSL distribution | FedoraLinux-43 |
| Git | 2.35.1.windows.2 |
| Python | 3.12.7 |
| Java | 24.0.1 |

**Notes:**

- Docker and Compose are used to run project services consistently in containers.
- Python is used for synthetic data generation, API simulation, and extraction.
- Git tracks source code and documentation.
- Java is required by the Spark runtime. Recheck Spark's supported Java versions when setting up PySpark; the machine currently reports Java 24. Do not assume this version will be used for Spark.
- WSL 2 is available. The WSL distributions observed were FedoraLinux-43, docker-desktop, Ubuntu, and podman-astro-machine.

### Docker engine verification

The first `docker info` attempt showed the Docker client but could not connect to the server:

```text
failed to connect to the docker API ... dockerDesktopLinuxEngine
```

**Resolution:** Open Docker Desktop from the Windows Start menu and wait for the engine to finish starting.

Then verify:

```powershell
docker info
docker run hello-world
wsl -l -v
```

Successful verification means:

- `docker info` displays both Client and Server sections.
- `docker run hello-world` prints `Hello from Docker!`.
- `wsl -l -v` shows Docker's WSL distribution running with version 2.

**Result:** Docker Engine and WSL were confirmed working.

## Step 1.2 — Clone and verify the Git repository

**Purpose:** Work from the project's GitHub repository so implementation and documentation can be version-controlled.

Repository URL:

`https://github.com/ayush270702/adtech-analytics-pipeline`

Clone command (run only if the repository is not already present locally):

```powershell
cd D:\
git clone https://github.com/ayush270702/adtech-analytics-pipeline.git
cd D:\adtech-analytics-pipeline
```

If the repository is already cloned, only navigate to it:

```powershell
cd D:\adtech-analytics-pipeline
```

Verify the repository:

```powershell
git status
```

Observed result:

```text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

**Result:** Local repository is on `main`, synchronized with `origin/main`, and had a clean working tree at verification time.

**Working convention:** We will create project directories only when a component requires them, rather than generating the entire folder tree at the beginning.

## Step 1.3 — Create the PostgreSQL Docker Compose configuration

**Purpose:** Define PostgreSQL as a containerized service. Docker Compose keeps the service configuration in a file that can be reproduced on another machine.

Create the file in the repository root:

```powershell
New-Item -ItemType File -Name "docker-compose.yml"
```

The current `docker-compose.yml` content:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: media_ads_postgres
    restart: unless-stopped

    environment:
      POSTGRES_DB: media_ads
      POSTGRES_USER: ads_admin
      POSTGRES_PASSWORD: ads_password

    ports:
      - "5432:5432"

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ads_admin -d media_ads"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

### Configuration explanation

| Configuration | Meaning |
|---|---|
| `services.postgres` | Defines the PostgreSQL service |
| `image: postgres:16` | Uses the official PostgreSQL 16 image |
| `container_name` | Gives the container a predictable name |
| `POSTGRES_DB` | Initializes the `media_ads` database on first startup |
| `POSTGRES_USER` | Initializes the `ads_admin` user |
| `POSTGRES_PASSWORD` | Sets the local development password |
| `5432:5432` | Maps Windows host port 5432 to container port 5432 |
| `postgres_data` | Persists database files outside the container lifecycle |
| `healthcheck` | Uses `pg_isready` to check PostgreSQL readiness |

**Important security note:** `ads_password` is only a temporary local-development password. Before treating this configuration as production-ready or sharing it for deployment, move credentials into an environment file or secret manager and ensure the real environment file is excluded from Git. Never commit production credentials.

### Validate the Compose file

Run from the repository root:

```powershell
docker compose config
```

**Successful result:** Compose prints the normalized configuration without a YAML or configuration error. Compose may add generated project network and volume names to the output; this is expected.

**Result:** Configuration validation succeeded.

## Step 1.4 — Start PostgreSQL

**Purpose:** Create and start the database container in the background.

```powershell
docker compose up -d
```

Verify the running container:

```powershell
docker ps
```

Expected container name:

`media_ads_postgres`

Check the health status:

```powershell
docker inspect --format='{{.State.Health.Status}}' media_ads_postgres
```

Expected output:

```text
healthy
```

If the result is initially `starting`, wait a short time and run the health-check command again.

**Result:** PostgreSQL container reported `healthy`.

## Step 1.5 — Connect to PostgreSQL and verify the database

**Purpose:** Confirm that the database accepts SQL connections and that the expected database and user exist.

Open an interactive PostgreSQL terminal inside the container:

```powershell
docker exec -it media_ads_postgres psql -U ads_admin -d media_ads
```

A successful connection displays a prompt similar to:

```text
media_ads=#
```

Run these commands inside `psql`:

### Check the database

```sql
SELECT current_database();
```

Expected value: `media_ads`.

### Check the connected user

```sql
SELECT current_user;
```

Expected value: `ads_admin`.

### Check the PostgreSQL version

```sql
SELECT version();
```

### List databases

```text
\l
```

### List schemas

```text
\dn
```

The initial schema list includes PostgreSQL's default `public` schema.

To leave the PostgreSQL terminal:

```text
\q
```

**Result:** Connection and database verification succeeded.

## Step 1.6 — Create the project schemas

**Purpose:** Organize database objects by pipeline responsibility. A schema is a namespace inside a PostgreSQL database. Tables in different schemas can share a table name without conflicting; for example, `source.campaigns` and `analytics.campaigns`.

Reconnect if needed:

```powershell
docker exec -it media_ads_postgres psql -U ads_admin -d media_ads
```

Run the following SQL inside `psql`:

```sql
CREATE SCHEMA source;
CREATE SCHEMA ops;
CREATE SCHEMA silver;
CREATE SCHEMA analytics;
```

Verify:

```text
\dn
```

The project schemas are:

| Schema | Responsibility |
|---|---|
| `source` | Structured source/business data such as clients, campaigns, ads, and performance |
| `ops` | Ingestion watermarks, batch records, pipeline logs, quality results, and rejected records |
| `silver` | Cleaned and standardized datasets published for downstream analytics |
| `analytics` | dbt dimensions, facts, reporting marts, and analytical outputs |
| `public` | PostgreSQL's default schema |

**Architecture note:** The raw Bronze layer will be stored as JSON files in the local data lake, not as a PostgreSQL schema. Silver will have Parquet files in the data lake and corresponding PostgreSQL tables for dbt consumption in this local V1 architecture.

**Result:** The four project schemas were created and verified.

---

## Step 1.7 — Create the first source business tables

**Completion date:** 2026-11-05

**Purpose:** Create the first two business reference tables in the `source` schema and insert small seed records. These tables establish the clients and advertising platforms that campaigns will reference.

### Create `source.clients`

```sql
CREATE TABLE source.clients (
    client_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    client_name VARCHAR(150) NOT NULL,
    industry VARCHAR(100),
    country VARCHAR(100),
    currency_code CHAR(3) NOT NULL DEFAULT 'INR',
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Create `source.platforms`

```sql
CREATE TABLE source.platforms (
    platform_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    platform_name VARCHAR(50) NOT NULL UNIQUE,
    platform_type VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### Column and constraint notes

- `BIGINT GENERATED ALWAYS AS IDENTITY` generates numeric IDs automatically. `PRIMARY KEY` guarantees each row has a unique, non-null identifier.
- `NOT NULL` marks required fields.
- `UNIQUE` on `platform_name` prevents duplicate platform names.
- `CHAR(3)` stores a three-character currency code such as INR.
- `TIMESTAMPTZ` stores timestamps with time-zone awareness.
- The `created_at` default records the insert time. `updated_at` currently defaults on insert too; it will not automatically change on updates unless we add application logic or a trigger later.

### Seed the initial records

```sql
INSERT INTO source.clients
    (client_name, industry, country, currency_code)
VALUES
    ('FreshMart India', 'Retail', 'India', 'INR'),
    ('TechNova Solutions', 'Technology', 'India', 'INR');

INSERT INTO source.platforms
    (platform_name, platform_type)
VALUES
    ('Meta', 'Social'),
    ('Google', 'Search'),
    ('YouTube', 'Video');
```

### Verification commands

Run in `psql`:

```text
\\dt source.*
\\d source.clients
\\d source.platforms
```

Then verify the seed records:

```sql
SELECT * FROM source.clients;
SELECT * FROM source.platforms;
```

**Verified result:** The user confirmed both tables and their seed records appeared correctly: 2 client rows and 3 platform rows (Meta, Google, YouTube).

**Modeling note:** YouTube is represented as its own reporting platform/channel in this portfolio model, although it belongs to Google's ecosystem. In a real cross-platform reporting implementation, clarify source/account ownership and attribution to avoid double-counting Google/YouTube performance.

# Useful commands reference

Run these in Windows PowerShell from the repository root unless noted otherwise.

| Task | Command |
|---|---|
| Show Docker CLI version | `docker --version` |
| Show Compose version | `docker compose version` |
| Show running containers | `docker ps` |
| Show all containers | `docker ps -a` |
| Start Compose services | `docker compose up -d` |
| Stop Compose services | `docker compose down` |
| View service logs | `docker compose logs postgres` |
| Follow service logs | `docker compose logs -f postgres` |
| Check PostgreSQL health | `docker inspect --format='{{.State.Health.Status}}' media_ads_postgres` |
| Open PostgreSQL shell | `docker exec -it media_ads_postgres psql -U ads_admin -d media_ads` |
| Check Git working tree | `git status` |
| View recent commits | `git log --oneline -5` |

**Data safety:** `docker compose down` stops and removes the Compose container and network but preserves the named volume. Do not use `docker compose down -v` unless you intentionally want to delete the PostgreSQL volume and its data.

---

# Next planned milestone

## Step 1.7 — Create source business tables

We will create the source tables incrementally, starting with the business entities and their relationships:

1. `source.clients`
2. `source.platforms`
3. `source.campaigns`
4. `source.ads`
5. `source.audiences`
6. `source.ad_daily_performance`
7. `source.ad_audience_performance`
8. `source.campaign_daily_budget`

For each table, document its purpose, grain, columns, data types, primary key, foreign keys, uniqueness rules, and verification queries before moving to the next table.

After the source tables, we will create the operational metadata tables in `ops`.

---

## Change history

| Date | Change |
|---|---|
| 2026-10-31 | Created the implementation runbook and documented verified environment, Git, Docker Compose, PostgreSQL, and schema setup steps. |
| 2026-11-05 | Documented and verified the first two source tables and seed records (`source.clients`, `source.platforms`). |
