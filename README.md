
# Baby Trackr — Local Development Setup

Baby Trackr is a multi-service application with a Kotlin/Spring Boot API, a Kotlin/Spring Boot analytics consumer, a React/Vite frontend, PostgreSQL databases, and Apache Kafka.

This guide supports **two workflows**:

1. **Docker-only:** Run the UI, both Spring Boot applications, both databases, and Kafka with Docker Compose.
2. **Hybrid development:** Run PostgreSQL and Kafka in Docker; run the Spring Boot apps in IntelliJ and React with Vite.

> **Project status:** The main API and infrastructure containers have been started successfully. The analytics and frontend container startup, complete end-to-end Kafka flow, and fresh-clone setup still need verification. Instructions below describe the intended configuration; adjust the noted project-specific settings before publishing this guide as fully tested.

## 1. Prerequisites

- Git
- Docker Desktop with the `docker` and `docker compose` commands available (or a compatible Docker Engine/Compose installation)
- For **hybrid** development: JDK 21, IntelliJ IDEA (optional, but recommended for Kotlin), and Node.js/npm compatible with `baby-trackr-ui`
- Free local ports: **5432**, **5433**, **9094**, **8080**, **8081**, and (Docker UI) **3000**. Vite typically uses **5173**, subject to the UI's configuration.

Verify Docker:

```bash
docker --version
docker compose version
docker info
```

Make sure Docker Desktop is running before using Compose.

## 2. Clone all four repositories

The repositories must be **siblings in one parent directory**, because the infrastructure Compose file builds the applications using relative paths (`../baby-trackr-service`, etc.).

```text
baby-trackr/                  # Parent directory (any name)
├── baby-trackr-service/      # Main Spring Boot API
├── baby-trackr-analytics/    # Spring Boot analytics consumer
├── baby-trackr-ui/           # React/Vite frontend
└── baby-trackr-infrastructure/
    ├── compose.yaml
    ├── .env                  # Local only; never commit
    └── README.md
```

```bash
mkdir baby-trackr
cd baby-trackr

git clone <MAIN_SERVICE_REPO_URL> baby-trackr-service
git clone <ANALYTICS_REPO_URL> baby-trackr-analytics
git clone <UI_REPO_URL> baby-trackr-ui
git clone <INFRASTRUCTURE_REPO_URL> baby-trackr-infrastructure

cd baby-trackr-infrastructure
```

Replace the four placeholders with the real GitHub clone URLs before publishing the README.

## 3. Configure local environment variables

Create a local `.env` in `baby-trackr-infrastructure/`. Prefer copying a committed `.env.example` if the project includes one:

```bash
cp .env.example .env
```

If no `.env.example` has been committed yet, create `.env` manually from this **example**, using your own local development passwords:

```dotenv
# Main PostgreSQL container initialization
DB_MAIN_NAME=baby_trackr
DB_MAIN_USERNAME=postgres
DB_MAIN_PASSWORD=CHANGE_ME_MAIN

# Analytics PostgreSQL container initialization
ANALYTICS_DB_NAME=baby_trackr_analytics
ANALYTICS_DB_USER=analytics_user
ANALYTICS_DB_PASSWORD=CHANGE_ME_ANALYTICS

# Docker-internal JDBC addresses (not localhost)
MAIN_DB_URL=jdbc:postgresql://trackr-db:5432/baby_trackr
ANALYTICS_DB_URL=jdbc:postgresql://analytics-db:5432/baby_trackr_analytics
```

**Important:** The database names and credentials in this sample are examples. They must match the actual names passed to `POSTGRES_DB`, `POSTGRES_USER`, and `POSTGRES_PASSWORD` for each database in `compose.yaml`. Check that `analytics-db` uses `ANALYTICS_DB_*` values rather than accidentally reusing `DB_MAIN_*` values.

**Security:** Never commit `.env` or credentials. Add `.env` to the infrastructure repository's `.gitignore`. Commit a credential-free `.env.example` instead. Do not paste unredacted `docker compose config` output into issues or chats—it expands secrets.

### How the configuration reaches applications

- The infrastructure `.env` provides values for `${...}` substitutions in `compose.yaml`.
- Only values explicitly supplied under a service's `environment:` (or `env_file:`) become container environment variables.
- Spring Boot uses those environment variables to configure its datasource and Kafka connections.
- **A Compose `.env` does not automatically become an IntelliJ run configuration.** Hybrid mode needs the local settings described below.

Before proceeding:

```bash
docker compose config --quiet
```

No output usually means Compose configuration validation succeeded. It does not guarantee the services will start.

## 4. Workflow A — Everything in Docker

Run these commands from `baby-trackr-infrastructure/`:

```bash
docker compose up -d --build
```

This starts **all services defined by `compose.yaml`**, including:

| Service | Host access | Docker-network access |
| --- | --- | --- |
| Main API (`baby-trackr-service`) | `http://localhost:8080/v1/...` | `http://baby-trackr-service:8080/v1/...` |
| Analytics (`baby-trackr-analytics`) | `localhost:8081` | `baby-trackr-analytics:8080` |
| Main PostgreSQL (`trackr-db`) | `localhost:5432` | `trackr-db:5432` |
| Analytics PostgreSQL (`analytics-db`) | `localhost:5433` | `analytics-db:5432` |
| Kafka (`kafka`) | `localhost:9094` | `kafka:9092` |
| Frontend (`baby-trackr-ui`) | `http://localhost:3000` | Depends on web-server configuration |

> The UI port assumes the existing Compose mapping `3000:80` and a frontend image serving on container port 80. Browser-origin API calls need a browser-accessible API URL such as `http://localhost:8080`, not the Docker-only hostname `baby-trackr-service`. Confirm Vite build-time environment configuration and backend CORS settings when bringing the UI online.

Inspect status and logs:

```bash
docker compose ps
docker compose logs --tail=60 baby-trackr-service
docker compose logs --tail=60 baby-trackr-analytics
```

Rebuild just one application after changes:

```bash
docker compose up -d --build baby-trackr-service
# Or:
docker compose up -d --build baby-trackr-analytics
```

With the current Dockerfiles, **editing Kotlin code does not automatically update a running container**. Rebuild the relevant image and recreate its container after changing the application.

## 5. Workflow B — Infrastructure in Docker, apps locally

First stop any application containers that would compete with local ports. It's fine if some of them have never been created:

```bash
docker compose stop baby-trackr-service baby-trackr-analytics baby-trackr-ui
```

Then start only the infrastructure:

```bash
docker compose up -d trackr-db analytics-db kafka
```

Run the Spring Boot applications from their own repositories, typically using **Run/Debug** in IntelliJ. Run React/Vite from `baby-trackr-ui`:

```bash
npm install
npm run dev
```

Use **host-accessible** connection settings for locally running Spring Boot apps:

| Dependency | Main/analytics app running on Mac | App running in Docker |
| --- | --- | --- |
| Main DB | `jdbc:postgresql://localhost:5432/baby_trackr` | `jdbc:postgresql://trackr-db:5432/baby_trackr` |
| Analytics DB | `jdbc:postgresql://localhost:5433/baby_trackr_analytics` | `jdbc:postgresql://analytics-db:5432/baby_trackr_analytics` |
| Kafka | `localhost:9094` | `kafka:9092` |

Database names here must match your actual `.env` values. Example IntelliJ environment variables for **main API**:

```dotenv
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/baby_trackr
SPRING_DATASOURCE_USERNAME=postgres
SPRING_DATASOURCE_PASSWORD=YOUR_LOCAL_MAIN_PASSWORD
KAFKA_BOOTSTRAP_SERVERS=localhost:9094
```

Example IntelliJ environment variables for **analytics** (if it reads the same Kafka variable):

```dotenv
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5433/baby_trackr_analytics
SPRING_DATASOURCE_USERNAME=analytics_user
SPRING_DATASOURCE_PASSWORD=YOUR_LOCAL_ANALYTICS_PASSWORD
KAFKA_BOOTSTRAP_SERVERS=localhost:9094
SERVER_PORT=8081
```

> **Confirm the analytics service's actual configuration property names before copying this verbatim.** It may still reference `BABYTRACKR_KAFKA_BOOTSTRAP_SERVERS` or use a different local server port. It should not publish the same host port as the main API when both run locally.

For the main API, the YAML should provide the same Kafka address to Spring Kafka and custom producer configuration:

```yaml
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9094}

babytrackr:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9094}
    topics:
      baby-events: baby-events
```

Add these sections to the **existing** `spring:` and `babytrackr:` blocks rather than creating duplicate top-level YAML keys. Keep the authoritative application configuration at `src/main/resources/application.yaml` in each Spring Boot repository.

For the React app, confirm that its Vite environment settings target `http://localhost:8080` for browser API requests and that any `/v1/` paths remain unchanged. Follow any UI-specific README instructions for its exact environment-variable name.

## 6. Day-to-day Docker commands

Run from `baby-trackr-infrastructure/`:

| Command | Purpose |
| --- | --- |
| `docker compose ps` | See container status and published ports |
| `docker compose up -d` | Start or recreate services as needed |
| `docker compose up -d trackr-db analytics-db kafka` | Start infrastructure dependencies only |
| `docker compose logs -f baby-trackr-service` | Follow main API logs |
| `docker compose stop` | Stop existing containers; keep them for `start` |
| `docker compose start` | Start previously created containers |
| `docker compose down` | Remove Compose containers and networks; retain named volumes by default |
| `docker compose build --no-cache baby-trackr-service` | Rebuild one image without Docker layer cache (troubleshooting) |

**Do not run `docker compose down -v` casually:** it removes Compose-managed named volumes and can delete your local PostgreSQL data.

## 7. Verify the system

1. Run `docker compose ps`. Both databases should report **healthy**; Kafka and the applications should be running.
2. Check both Spring Boot logs for `Started ...ApplicationKt`. This indicates application startup, **not** proof of end-to-end Kafka processing.
3. Call a **known, implemented** endpoint under `http://localhost:8080/v1/` using your API client's documented path. Avoid assuming `/api/` exists.
4. Create a valid baby event through the API and verify that the event appears in the analytics database, using the application's actual schema/query or logs.
5. Verify that the UI can call the API from the browser; inspect browser developer tools for CORS or incorrect API URL errors.

## 8. Troubleshooting

**`docker: command not found`** — Confirm Docker Desktop is installed and that Docker's CLI tools are installed in a directory on your shell `PATH`. In Docker Desktop, review CLI tools installation settings.

**`docker-credential-desktop: executable file not found in $PATH`** — Docker's credential helper isn't discoverable by the shell; check the Docker Desktop CLI tools installation.

**`Failed to bind properties under 'babytrackr.kafka'`** — Check the actual packaged `src/main/resources/application.yaml`, the `@ConfigurationProperties` prefix, and the required `bootstrap-servers` / `topics` settings. Files in the repository root aren't automatically the same as the resources packaged into the JAR.

**Kafka client tries `localhost:9092` from Docker** — The application is using a local-only broker address. Inside Docker use `kafka:9092`; on the Mac use `localhost:9094`. Check both `spring.kafka.bootstrap-servers` (including KafkaAdmin) and custom `babytrackr.kafka.bootstrap-servers`.

**`Connection refused` to PostgreSQL** — Verify the container's health status, its hostname and container port, and that main and analytics credentials are not mixed up.

**Port already allocated / address in use** — A Docker app and a locally running app may both be trying to use port 8080 or 8081. Stop the duplicate process or container.

**Changes to `.env` don't change existing database users** — PostgreSQL image initialization variables apply when a fresh data directory is created. Changing them does not automatically update users/passwords in an already initialized named volume. Avoid deleting data to solve a credential mismatch unless the data is disposable.

**Docker appears to use an old `application.yaml`** — Confirm the source path is `src/main/resources/application.yaml`, not just `./application.yaml`; rebuild the affected image and inspect its logs. Only use `--no-cache` if needed after verifying the correct file is in the build context.

## 9. Repository maintenance checklist

Before sharing setup publicly:

- [ ] Replace all four GitHub URL placeholders with actual clone URLs.
- [ ] Commit a matching `.env.example` (never secrets) and ensure `.env` is gitignored.
- [ ] Make sure `analytics-db` uses its own `ANALYTICS_DB_*` initialization variables.
- [ ] Confirm analytics config and IntelliJ port `8081` work.
- [ ] Confirm the frontend Dockerfile, Vite API environment variable, and CORS setup.
- [ ] Test a fresh clone using both workflows on a clean machine.
- [ ] Test event publication, Kafka consumption, and analytics database updates.

---

**Architecture note:** The infrastructure repository owns container orchestration. Each application repository still owns its source code, Dockerfile, and application-level configuration. These responsibilities are separate by design.
