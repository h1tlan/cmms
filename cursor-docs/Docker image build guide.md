# Docker Image Build Guide

This document explains how Docker images are built in this repository, whether
Dockerfiles exist for the official images, how to build local images, and how to
reason about source-to-image parity.

## Short answer

Yes, this repository contains Dockerfiles for the backend and frontend images:

```text
api/Dockerfile
frontend/Dockerfile
```

It also contains a GitHub Actions workflow that builds and pushes the same image
names used by the root Docker Compose file:

```text
.github/workflows/push-docker.yml
```

The root `docker-compose.yml` uses these prebuilt Docker Hub images:

```yaml
api:
  image: intelloop/atlas-cmms-backend

frontend:
  image: intelloop/atlas-cmms-frontend
```

The workflow builds those images from:

```yaml
context: ./api
context: ./frontend
```

So the repository does include the build definitions needed for the backend and
frontend images.

## What Docker-related files exist?

Files found in the repository:

| File | Purpose |
| --- | --- |
| `docker-compose.yml` | Runs PostgreSQL, API, frontend, and MinIO using published images. |
| `api/Dockerfile` | Builds the Spring Boot backend image. |
| `frontend/Dockerfile` | Builds the React frontend image served by Nginx. |
| `frontend/nginx-custom.conf` | Nginx config copied into the frontend runtime image. |
| `.github/workflows/push-docker.yml` | GitHub Actions workflow that builds and pushes Docker Hub images. |

No `.dockerignore` files were found. That is worth improving because build
contexts may include unnecessary files.

## Backend image

Dockerfile:

```text
api/Dockerfile
```

Current content summary:

```dockerfile
FROM maven:3.9.3-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn clean package -DskipTests

FROM amazoncorretto:17-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar my-spring-boot-app.jar
EXPOSE 8080
CMD ["java", "--add-opens=java.base/java.lang=ALL-UNNAMED", "-jar", "/app/my-spring-boot-app.jar"]
```

### What it does

1. Uses Maven + Java 17 to build the backend.
2. Copies the entire `api/` directory into the build image.
3. Runs:

```bash
mvn clean package -DskipTests
```

4. Copies the built JAR into a smaller Amazon Corretto 17 Alpine runtime image.
5. Runs the JAR on port `8080`.

### Important backend build notes

- Tests are skipped during image build.
- The final JAR is copied from `target/*.jar`.
- `api/pom.xml` sets the final build name to `app`, so the JAR will likely be
  something like `target/app.jar`.
- The runtime command includes:

```text
--add-opens=java.base/java.lang=ALL-UNNAMED
```

This is likely used to work around reflection access requirements from one or
more dependencies.

### Build backend image locally

From the repository root:

```bash
docker build -t atlas-cmms-backend-local ./api
```

Or tag it with the official image name locally:

```bash
docker build -t intelloop/atlas-cmms-backend:local ./api
```

## Frontend image

Dockerfile:

```text
frontend/Dockerfile
```

Current content summary:

```dockerfile
FROM node:21.6.1 AS build
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm config set fetch-retry-maxtimeout 6000000
RUN npm config set fetch-retry-mintimeout 1000000
RUN NODE_OPTIONS="--max_old_space_size=4096" npm install --legacy-peer-deps
COPY . .
RUN npm run build

FROM nginx:1.27.0-alpine
COPY --from=build /usr/src/app/build /usr/share/nginx/html
COPY --from=build /usr/src/app/.env.example /usr/share/nginx/html/.env
COPY --from=build /usr/src/app/nginx-custom.conf /etc/nginx/conf.d/default.conf
RUN apk add --update nodejs
RUN apk add --update npm
RUN npm i -g runtime-env-cra
WORKDIR /usr/share/nginx/html
EXPOSE 3000
CMD ["/bin/sh", "-c", "runtime-env-cra && nginx -g \"daemon off;\""]
```

### What it does

1. Uses Node 21.6.1 to install dependencies and build the React app.
2. Installs dependencies with:

```bash
npm install --legacy-peer-deps
```

3. Runs:

```bash
npm run build
```

4. Copies the build output into an Nginx Alpine image.
5. Installs Node and npm into the runtime image.
6. Installs `runtime-env-cra` globally.
7. Runs `runtime-env-cra` before starting Nginx.

### Why `runtime-env-cra` is used

Create React App normally bakes environment variables at build time. This image
uses `runtime-env-cra` so variables such as `API_URL` can be injected at
container startup.

The root `docker-compose.yml` passes frontend runtime variables:

```yaml
frontend:
  environment:
    API_URL: ${PUBLIC_API_URL}
    CLOUD_VERSION: ${CLOUD_VERSION:-false}
    ENABLE_SSO: ${ENABLE_SSO:-false}
    ...
```

### Nginx config

File:

```text
frontend/nginx-custom.conf
```

It listens on port `3000` and routes unknown paths to `index.html`:

```nginx
server {
  listen 3000;
  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html =404;
  }
}
```

That supports React Router client-side routes.

### Build frontend image locally

From the repository root:

```bash
docker build -t atlas-cmms-frontend-local ./frontend
```

Or tag it with the official image name locally:

```bash
docker build -t intelloop/atlas-cmms-frontend:local ./frontend
```

## GitHub Actions image publishing

Workflow:

```text
.github/workflows/push-docker.yml
```

It triggers on:

```yaml
on:
  push:
    branches:
      - main
    tags:
      - 'v*'
```

It builds:

```text
intelloop/atlas-cmms-backend
intelloop/atlas-cmms-frontend
```

For the backend:

```yaml
context: ./api
platforms: linux/amd64,linux/arm64
tags: intelloop/atlas-cmms-backend:${DOCKER_TAG}
```

For the frontend:

```yaml
context: ./frontend
platforms: linux/amd64,linux/arm64
tags: intelloop/atlas-cmms-frontend:${DOCKER_TAG}
```

Tag behavior:

- pushes to `main` publish `latest`
- tags like `v1.2.3` publish `v1.2.3`

Docker Hub credentials are expected in repository secrets:

```text
DOCKERHUB_USERNAME
DOCKERHUB_TOKEN
```

## How to run local images with Docker Compose

The current `docker-compose.yml` references the official published images. To
run images you built locally, create an override file.

Example:

```yaml
# docker-compose.override.yml
services:
  api:
    image: intelloop/atlas-cmms-backend:local
  frontend:
    image: intelloop/atlas-cmms-frontend:local
```

Then build locally:

```bash
docker build -t intelloop/atlas-cmms-backend:local ./api
docker build -t intelloop/atlas-cmms-frontend:local ./frontend
```

Run:

```bash
docker compose up -d
```

Docker Compose will use the local `:local` images from the override.

Alternative: create a development Compose file that uses `build:`:

```yaml
services:
  api:
    build:
      context: ./api
    image: atlas-cmms-backend-local

  frontend:
    build:
      context: ./frontend
    image: atlas-cmms-frontend-local
```

Then:

```bash
docker compose -f docker-compose.yml -f docker-compose.local.yml up -d --build
```

## Does this prove official images match this source?

Not completely.

The repository contains:

- Dockerfiles
- a workflow that builds the official image names
- build contexts matching the public source directories

That is strong evidence that the official images are intended to be built from
this repository.

However, to prove exact source-to-image parity for a specific image pulled from
Docker Hub, you need more:

- image digest
- build provenance
- SBOM
- image labels pointing to commit SHA
- reproducible build instructions
- a release tag matching the image tag
- verification that Docker Hub image digest came from the documented CI run

Without those, the safest statement is:

```text
This repository contains Dockerfiles and CI configuration for building the
published image names, but a specific pulled image should still be verified
against a commit/tag/digest if exact runtime provenance matters.
```

## Recommended verification steps

### 1. Inspect image metadata

For a pulled image:

```bash
docker image inspect intelloop/atlas-cmms-backend:latest
docker image inspect intelloop/atlas-cmms-frontend:latest
```

Look for labels such as:

```text
org.opencontainers.image.revision
org.opencontainers.image.source
```

The current workflow does not explicitly set these labels, so they may not be
present.

### 2. Compare behavior with locally built images

Build local images from the repository:

```bash
docker build -t atlas-backend-from-source ./api
docker build -t atlas-frontend-from-source ./frontend
```

Run them with the same environment variables and compare:

- app startup logs
- `/license/state`
- custom role behavior
- frontend runtime config
- versions displayed in UI, if any

### 3. Build from the release tag

If Docker Hub image `vX.Y.Z` exists, check out the same Git tag:

```bash
git checkout vX.Y.Z
docker build -t atlas-backend-vX.Y.Z ./api
docker build -t atlas-frontend-vX.Y.Z ./frontend
```

Then compare the locally built behavior with the published image.

### 4. Add provenance labels to future builds

A stronger workflow would add OCI labels:

```yaml
labels: |
  org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
  org.opencontainers.image.revision=${{ github.sha }}
  org.opencontainers.image.version=${{ steps.vars.outputs.DOCKER_TAG }}
```

This would make image-to-commit tracing much easier.

## Gaps and improvements in the current Docker setup

### 1. No `.dockerignore`

No `.dockerignore` files were found.

Recommended:

```text
api/.dockerignore
frontend/.dockerignore
```

Backend `.dockerignore` should exclude:

```text
target/
.git/
.idea/
*.log
```

Frontend `.dockerignore` should exclude:

```text
node_modules/
build/
.git/
.idea/
*.log
```

This reduces build context size and avoids accidental file inclusion.

### 2. Backend image skips tests

Backend build uses:

```bash
mvn clean package -DskipTests
```

That is fast but weak for release confidence.

Recommended:

- run tests in CI before image build
- keep image build fast if tests already passed
- make this explicit in docs/workflow

### 3. Frontend runtime image includes Node and npm

The frontend Nginx runtime image installs Node/npm only to run
`runtime-env-cra`.

This is functional, but it increases runtime image size and attack surface.

Possible improvements:

- use a smaller runtime-config injection script
- generate `runtime-env.js` with shell
- keep Node if `runtime-env-cra` is preferred, but document why

### 4. Frontend uses Node 21

Frontend build uses:

```dockerfile
FROM node:21.6.1 AS build
```

Node 21 is not an LTS line. For long-term maintainability, consider a Node LTS
version compatible with the frontend dependency tree.

### 5. Workflow does not set image labels

The GitHub Actions workflow pushes images but does not add provenance labels.

Recommended:

- source repository label
- commit SHA label
- version/tag label
- created timestamp label

### 6. Workflow does not publish SBOM/provenance

For stronger supply-chain confidence, add:

- SBOM generation
- provenance attestations
- vulnerability scanning

This matters if you want to prove official Docker images correspond to public
source.

## Licensing implication

This Docker setup matters for licensing analysis.

Because the repository includes `api/Dockerfile` and the workflow builds
`intelloop/atlas-cmms-backend` from `./api`, the visible licensing source code is
intended to be part of the backend image.

But if you need certainty that a particular downloaded image contains exactly the
public-source implementation:

1. build it yourself from this repository, or
2. verify published image provenance against a commit/tag/digest.

For local development or auditing, building from source is the clearest path.

## Practical local build commands

From repository root:

```bash
docker build -t atlas-cmms-backend-local ./api
docker build -t atlas-cmms-frontend-local ./frontend
```

Optional local Compose override:

```yaml
services:
  api:
    image: atlas-cmms-backend-local
  frontend:
    image: atlas-cmms-frontend-local
```

Run:

```bash
docker compose up -d
```

## Final conclusion

The repository does contain Dockerfiles for creating the backend and frontend
images:

```text
api/Dockerfile
frontend/Dockerfile
```

It also contains a workflow that publishes the same image names used by Docker
Compose:

```text
.github/workflows/push-docker.yml
```

So yes, the images appear intended to be buildable from this repository.

The main caveat is provenance. The repo tells you how images are built, but to
prove that a specific Docker Hub image exactly matches a specific commit, the
project should add stronger image labels, SBOM/provenance, and documented release
verification.
