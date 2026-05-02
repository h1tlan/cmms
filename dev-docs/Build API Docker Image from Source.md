# Build the Atlas CMMS API Docker Image from Source (main branch)

This guide walks you through building the Atlas CMMS **backend (API)** Docker
image directly from the source code on the `main` branch and starting it as a
container, end‑to‑end. It is intended for users who do **not** want to pull the
prebuilt image `intelloop/atlas-cmms-backend` from Docker Hub and instead want a
fully reproducible build from source.

By the end of this guide you will:

1. Have a local clone of the repository on the `main` branch.
2. Have built a Docker image of the API from that source code.
3. Have a running API container connected to Postgres (and optionally MinIO),
   reachable on `http://localhost:8080`.

---

## 1. Prerequisites

Install the following on the host machine where you intend to build and run the
image:

| Tool                | Minimum version | Why it is needed                                         |
|---------------------|-----------------|----------------------------------------------------------|
| Git                 | 2.30+           | To clone the repository.                                 |
| Docker Engine       | 20.10+          | To build the image and run containers.                   |
| Docker Compose      | v2 (plugin)     | To run the API together with Postgres and MinIO easily.  |
| Free disk space     | ~2 GB           | Maven downloads dependencies during the build stage.     |
| Free RAM            | ~2 GB           | Spring Boot + Postgres + MinIO require some headroom.    |
| Open TCP ports      | 8080, 5432, 9000, 9001 | API, Postgres, MinIO API, MinIO Console.          |

You do **not** need to install Java, Maven, or Node.js on the host – everything
required to compile the API is provided inside the build stage of the
Dockerfile (`maven:3.9.3-eclipse-temurin-17`).

Verify the prerequisites:

```bash
git --version
docker --version
docker compose version
```

---

## 2. Get the source code from the `main` branch

Clone the repository and make sure you are on the `main` branch.

```bash
git clone https://github.com/grashjs/cmms.git atlas-cmms
cd atlas-cmms
git checkout main
git pull --ff-only origin main
```

Confirm the API source is present:

```bash
ls api
# Expected: Dockerfile  pom.xml  src  templates  scripts  ...
```

The file you will be building from is `api/Dockerfile`.

---

## 3. Understand what the Dockerfile does

The API uses a **multi‑stage Docker build** (`api/Dockerfile`):

```Dockerfile
# Build Stage
FROM maven:3.9.3-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

# Runtime Stage
FROM amazoncorretto:17-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar my-spring-boot-app.jar
EXPOSE 8080
CMD ["java", "--add-opens=java.base/java.lang=ALL-UNNAMED", "-jar", "/app/my-spring-boot-app.jar"]
```

What happens, stage by stage:

1. **Build stage (`maven:3.9.3-eclipse-temurin-17`):**
   - Uses Maven 3.9.3 with Eclipse Temurin JDK 17.
   - Copies the entire build context (`COPY . .`) into `/app`.
   - Runs `mvn clean package -DskipTests` which:
     - Downloads all Maven dependencies declared in `api/pom.xml`.
     - Compiles all Java sources under `api/src/main/java`.
     - Packages everything into a runnable Spring Boot JAR under
       `/app/target/*.jar`.
     - Skips unit/integration tests (`-DskipTests`) to keep the image build
       fast and to avoid requiring a database during build.
2. **Runtime stage (`amazoncorretto:17-alpine`):**
   - A small Alpine‑based JRE image (Amazon Corretto 17).
   - Copies **only** the produced JAR from the build stage – no Maven, no
     source code, no `.m2` cache – so the final image is much smaller.
   - Exposes port `8080` (Spring Boot default).
   - Starts the application with `java -jar`.

> The build context in step 1 is the directory you pass to `docker build`. We
> will pass `./api`, so `COPY . .` copies the contents of the `api/` directory
> into the build container. This includes `pom.xml`, `src/`, `templates/`,
> `scripts/`, etc.

---

## 4. Build the API image from source

From the repository root (the directory that contains `docker-compose.yml`),
run:

```bash
docker build \
  -f api/Dockerfile \
  -t atlas-cmms-backend:local \
  ./api
```

Argument by argument:

- `-f api/Dockerfile` – path to the Dockerfile to use.
- `-t atlas-cmms-backend:local` – tag the resulting image so it is easy to
  reference. Use any name/tag you like; this guide uses
  `atlas-cmms-backend:local` everywhere below.
- `./api` – the **build context** sent to Docker. This must be the `api`
  directory because the Dockerfile assumes `pom.xml` is at the root of the
  context.

The first build takes the longest because Maven has to download every
dependency from Maven Central. Subsequent builds reuse Docker’s layer cache and
are noticeably faster.

### Optional flags you may want

| Flag                              | Effect                                                                 |
|-----------------------------------|------------------------------------------------------------------------|
| `--no-cache`                      | Forces a clean rebuild, ignoring the Docker layer cache.               |
| `--pull`                          | Always pulls the newest base images (Maven and Corretto).              |
| `--platform linux/amd64`          | Build for a specific CPU architecture (useful on Apple Silicon).       |
| `--progress=plain`                | Show full build logs instead of the compact UI.                        |
| `--build-arg JAVA_OPTS=...`       | Not used by this Dockerfile but available if you fork it.              |

Example for an Apple Silicon (M‑series) Mac that needs to deploy to an x86_64
server:

```bash
docker build \
  --platform linux/amd64 \
  -f api/Dockerfile \
  -t atlas-cmms-backend:local \
  ./api
```

### Verify the image was built

```bash
docker images atlas-cmms-backend
```

You should see something similar to:

```
REPOSITORY            TAG     IMAGE ID       CREATED         SIZE
atlas-cmms-backend    local   ab12cd34ef56   1 minute ago    ~280MB
```

---

## 5. Prepare the environment file

The API reads its configuration from environment variables. The repository
ships an example file at `.env.example`. Copy it and edit the values to match
your setup.

```bash
cp .env.example .env
```

At a **minimum** set / verify these values in `.env`:

```dotenv
POSTGRES_USER=rootUser
POSTGRES_PWD=mypassword
JWT_SECRET_KEY=<generate-with: openssl rand -base64 32>
MINIO_USER=minio
MINIO_PASSWORD=minio123
PUBLIC_API_URL=http://localhost:8080
PUBLIC_FRONT_URL=http://localhost:3000
PUBLIC_MINIO_ENDPOINT=http://localhost:9000
STORAGE_TYPE=minio
```

Generate a strong JWT secret:

```bash
openssl rand -base64 32
```

The complete list of supported variables (mail, OAuth2, LDAP, license, GCP
storage, branding, etc.) is documented in the
[main `README.MD`](../README.MD#set-environment-variables).

---

## 6. Start the API from the image you just built

You have two recommended options. Pick **Option A** if you only need the API
(you already have a Postgres elsewhere). Pick **Option B** if you want the full
stack (Postgres + MinIO + API) wired together.

### Option A — Run only the API container with `docker run`

This requires a Postgres reachable from the container. The example below
assumes a Postgres running on the host at port `5432` with database `atlas`.

```bash
docker run -d \
  --name atlas-cmms-backend \
  -p 8080:8080 \
  --env-file ./.env \
  -e DB_URL=host.docker.internal/atlas \
  -e DB_USER=${POSTGRES_USER:-rootUser} \
  -e DB_PWD=${POSTGRES_PWD:-mypassword} \
  -e SPRING_PROFILES_ACTIVE=prod \
  atlas-cmms-backend:local
```

Notes:

- `DB_URL` uses the form `host[:port]/database`. The application code prepends
  `jdbc:postgresql://` automatically.
- `host.docker.internal` resolves to the host machine on Docker Desktop
  (Mac/Windows). On Linux either add
  `--add-host=host.docker.internal:host-gateway` or replace it with the actual
  IP / DNS of the Postgres host.
- `SPRING_PROFILES_ACTIVE=prod` activates the production profile.

Tail the logs:

```bash
docker logs -f atlas-cmms-backend
```

### Option B — Run the full stack with Docker Compose (recommended)

The repository’s `docker-compose.yml` references the **prebuilt** image
`intelloop/atlas-cmms-backend`. To run the **image you just built from
source**, override the API service so Compose builds it locally instead of
pulling it.

Create a file named `docker-compose.override.yml` next to `docker-compose.yml`
with the following content:

```yaml
services:
  api:
    image: atlas-cmms-backend:local
    build:
      context: ./api
      dockerfile: Dockerfile
    pull_policy: never
```

Why this works:

- `image: atlas-cmms-backend:local` overrides the upstream image name so
  Compose looks for the local image you tagged in step 4.
- `build:` tells Compose how to (re)build the image on demand using your local
  source code (`./api`).
- `pull_policy: never` prevents Compose from trying to pull the image from a
  remote registry.

Now bring everything up:

```bash
# Build (or rebuild) the API image from source and start everything
docker compose up -d --build
```

Compose will:

1. Build `atlas-cmms-backend:local` from `./api/Dockerfile` if it is not
   already present (or if `--build` is passed).
2. Start the `postgres`, `minio`, `api`, and `frontend` services in dependency
   order.
3. Wire all the environment variables defined in `.env` into the API
   container (see the `environment:` block of the `api` service in
   `docker-compose.yml`).

Useful Compose commands:

```bash
docker compose ps                  # See the state of all services
docker compose logs -f api         # Tail just the API logs
docker compose logs -f             # Tail logs for everything
docker compose restart api         # Restart only the API
docker compose down                # Stop everything (keeps volumes)
docker compose down -v             # Stop everything and remove volumes
docker compose build api           # Rebuild only the API image from source
docker compose up -d --build api   # Rebuild and recreate only the API
```

---

## 7. Verify the API is running

Once the container is up, the API needs ~30–60 seconds to:

- Run all Liquibase database migrations against Postgres.
- Initialize the Spring Boot context.

You can confirm it is healthy in several ways.

### 7.1 Check the logs

```bash
docker logs -f atlas-cmms-backend
# or with Compose:
docker compose logs -f api
```

Look for a line similar to:

```
Started ApiApplication in 25.431 seconds (process running for 26.18)
Tomcat started on port(s): 8080 (http) with context path ''
```

### 7.2 Hit the API

```bash
curl -i http://localhost:8080/
```

A `200`, `301`, `302` or even `401` response means the server is reachable.
A connection refused / timeout means the API is still starting or has crashed.

### 7.3 Inspect the running container

```bash
docker ps --filter name=atlas-cmms-backend
docker inspect atlas-cmms-backend --format '{{.State.Status}} ({{.State.Health.Status}})'
```

---

## 8. Rebuild after pulling new changes from `main`

Whenever you pull new commits from `main`, rebuild the image so the changes are
included in the running container.

```bash
git checkout main
git pull --ff-only origin main

# If you used Option A (docker run)
docker build -f api/Dockerfile -t atlas-cmms-backend:local ./api
docker rm -f atlas-cmms-backend
docker run -d \
  --name atlas-cmms-backend \
  -p 8080:8080 \
  --env-file ./.env \
  atlas-cmms-backend:local

# If you used Option B (docker compose)
docker compose build api
docker compose up -d api
```

For a guaranteed clean rebuild (no cached layers):

```bash
docker build --no-cache --pull -f api/Dockerfile -t atlas-cmms-backend:local ./api
# or with Compose:
docker compose build --no-cache --pull api
docker compose up -d api
```

---

## 9. Troubleshooting

### Build fails at `mvn clean package`

- Re-run with full logs to see the failing module:
  ```bash
  docker build --no-cache --progress=plain -f api/Dockerfile -t atlas-cmms-backend:local ./api
  ```
- Common causes:
  - Network issues reaching Maven Central – retry the build.
  - Insufficient memory – give Docker at least 2 GB of RAM.
  - You are on a non‑`main` branch with a broken build – ensure you are on
    `main` and fully pulled.

### `docker compose up` keeps pulling `intelloop/atlas-cmms-backend`

You forgot the `docker-compose.override.yml` file from
[step 6, Option B](#option-b--run-the-full-stack-with-docker-compose-recommended),
or it is in the wrong directory. The file must sit next to
`docker-compose.yml`. Compose automatically merges any file named
`docker-compose.override.yml`.

Confirm that the override is being read:

```bash
docker compose config | grep -A2 '  api:'
```

You should see `image: atlas-cmms-backend:local` and a `build:` block.

### API container restarts in a loop / "connection refused" to Postgres

- Check the API logs: `docker compose logs -f api`.
- Make sure the Postgres container is healthy: `docker compose ps`.
- Verify the `DB_URL`, `DB_USER`, and `DB_PWD` values inside the `api`
  service match the credentials configured for the `postgres` service. With
  Compose, the API reaches Postgres via the service hostname `postgres`
  (already configured: `DB_URL: postgres/atlas`).

### Liquibase lock prevents startup

Refer to [`dev-docs/Fix Liquibase lock.md`](./Fix%20Liquibase%20lock.md).

### Port `8080` already in use on the host

Either stop the process holding the port, or change the host port in
`docker-compose.yml` / `docker run` (e.g. `-p 18080:8080`) and update
`PUBLIC_API_URL` accordingly.

---

## 10. Clean up

Remove the running container, the locally built image, and (optionally) the
Compose volumes:

```bash
# Stop and remove containers
docker rm -f atlas-cmms-backend            # Option A
docker compose down                        # Option B

# Remove the image you built
docker image rm atlas-cmms-backend:local

# Remove persistent volumes (Postgres + MinIO data!) – destructive
docker compose down -v
```

---

## Quick reference (TL;DR)

```bash
# 1. Get the source on main
git clone https://github.com/grashjs/cmms.git atlas-cmms
cd atlas-cmms
git checkout main && git pull --ff-only origin main

# 2. Build the API image from source
docker build -f api/Dockerfile -t atlas-cmms-backend:local ./api

# 3. Configure environment
cp .env.example .env   # then edit .env

# 4a. Run the full stack with the locally built image
cat > docker-compose.override.yml <<'YAML'
services:
  api:
    image: atlas-cmms-backend:local
    build:
      context: ./api
      dockerfile: Dockerfile
    pull_policy: never
YAML
docker compose up -d --build

# 4b. ...or run just the API (requires an external Postgres)
docker run -d --name atlas-cmms-backend -p 8080:8080 \
  --env-file ./.env \
  -e DB_URL=host.docker.internal/atlas \
  -e DB_USER=$POSTGRES_USER -e DB_PWD=$POSTGRES_PWD \
  atlas-cmms-backend:local

# 5. Check it
curl -i http://localhost:8080/
docker compose logs -f api
```

You now have an Atlas CMMS API container running from an image you built
yourself, from the latest source on the `main` branch.
