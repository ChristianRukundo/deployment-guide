# Complete Production Deployment Guide

### React Frontend + Node.js Backend + Database + Docker + GitHub Actions + GHCR + Ubuntu + Docker Compose + Nginx + DNS + HTTPS

> **Purpose**
>
> This is a reusable instructor/student handout for projects that already run locally.
>
> The complete production path is:
>
> ```text
> Source Code
>     ↓
> Dockerfiles
>     ↓
> GitHub Actions CI
>     ↓
> Versioned Docker Images
>     ↓
> GitHub Container Registry (GHCR)
>     ↓
> Ubuntu Production Server
>     ↓
> Docker Compose
>     ↓
> Frontend + Backend + Database
>     ↓
> Host Nginx Reverse Proxy
>     ↓
> DNS / Subdomains
>     ↓
> HTTPS / Let's Encrypt
>     ↓
> Automated Deployment + Rollback + Backups
> ```
>
> This guide is production-oriented for a **single Ubuntu server**. It is not a high-availability architecture: if the server fails, the application remains unavailable until that server is recovered or replaced.

---

# Master Table of Contents

1. Step 1 — Dockerization & GitHub Actions CI/CD
2. Step 2 — Prepare the Ubuntu Production Server
3. Step 3 — Authenticate the Server to GHCR
4. Step 4 — Create Production Environment and Docker Compose
5. Step 5 — Pull and Run the Application
6. Step 6 — Configure DNS and Subdomains
7. Step 7 — Configure Host Nginx Reverse Proxy
8. Step 8 — Add HTTPS with Certbot
9. Step 9 — Complete Continuous Deployment
10. Step 10 — Rollback, Backups, Logs, and Operations
11. Step 11 — Hosting Multiple Student Projects on One Server
12. Step 12 — Final Production Checklist
13. Instructor Quick Reference
14. Official References

---

# STEP 1 — Dockerization & GitHub Actions CI/CD
### React Frontend + Node.js Backend + GitHub Container Registry (GHCR)

> **Purpose**
>
> This guide is designed to be reused across multiple React + Node.js projects that already run locally.
>
> The goal of this section is to:
>
> 1. Dockerize the frontend.
> 2. Dockerize the backend.
> 3. Verify both images locally.
> 4. Add a production-oriented GitHub Actions CI pipeline.
> 5. Automatically publish versioned images to GitHub Container Registry (GHCR) after successful CI.
>
> **This guide intentionally stops before server deployment.**
>
> The next deployment stage is:
>
> `GHCR → Ubuntu Server → Docker Compose → Nginx Reverse Proxy → Domain → HTTPS`

---

## Step 1 Table of Contents

1. [Target Architecture](#1-target-architecture)
2. [What Changes From One Project to Another](#2-what-changes-from-one-project-to-another)
3. [Expected Repository Structure](#3-expected-repository-structure)
4. [Five-Minute Preflight Check](#4-five-minute-preflight-check)
5. [Environment Variable Strategy](#5-environment-variable-strategy)
6. [Frontend Dockerization](#6-frontend-dockerization)
7. [Frontend Nginx Configuration](#7-frontend-nginx-configuration)
8. [Backend Dockerization](#8-backend-dockerization)
9. [Backend Runtime Requirements](#9-backend-runtime-requirements)
10. [Build and Test Images Locally](#10-build-and-test-images-locally)
11. [Docker Debugging Commands](#11-docker-debugging-commands)
12. [GitHub Repository Configuration](#12-github-repository-configuration)
13. [Production-Oriented GitHub Actions Workflow](#13-production-oriented-github-actions-workflow)
14. [How the Workflow Works](#14-how-the-workflow-works)
15. [GHCR Image Naming and Versioning](#15-ghcr-image-naming-and-versioning)
16. [Security and Production Notes](#16-security-and-production-notes)
17. [Common Problems](#17-common-problems)
18. [One-Hour Teaching Plan](#18-one-hour-teaching-plan)
19. [Final Checklist](#19-final-checklist)
20. [Next Deployment Stage](#20-next-deployment-stage)

---

# 1. Target Architecture

At the end of this lesson:

```text
Developer
   │
   │ git push / pull request
   ▼
GitHub Repository
   │
   ▼
GitHub Actions
   │
   ├── Frontend CI
   │     ├── npm ci
   │     ├── lint (if available)
   │     └── npm run build
   │
   ├── Backend CI
   │     ├── npm ci
   │     ├── lint (if available)
   │     ├── test (if available)
   │     └── build (if TypeScript)
   │
   └── If main branch CI succeeds
          │
          ├── Build frontend Docker image
          ├── Build backend Docker image
          ├── Tag images
          └── Push images
                    │
                    ▼
                   GHCR
```

The production server is **not building source code**.

The intended future production flow is:

```text
GitHub
   ↓
GitHub Actions
   ↓
GHCR
   ↓
Production Server
   ↓
Docker Compose
   ↓
Running Containers
```

---

# 2. What Changes From One Project to Another

Before copying anything, inspect these values.

| Item | Typical Example | Why It Matters |
|---|---|---|
| Frontend directory | `frontend`, `client`, `web` | Docker build context and CI working directory |
| Backend directory | `backend`, `server`, `api` | Docker build context and CI working directory |
| Node.js version | `20`, `22`, `24` | Must match a version supported by the project |
| Package manager | npm | This guide assumes npm |
| Frontend framework | React + Vite | Determines build output and environment variable naming |
| Frontend build command | `npm run build` | Used by Docker and CI |
| Frontend build output | `dist` | Vite normally outputs `dist` |
| Frontend API variable | `VITE_API_URL` | Passed while building the frontend |
| Backend language | JavaScript / TypeScript | Determines which backend Dockerfile to use |
| Backend start command | `npm start` | Used as the container startup command |
| Backend build command | `npm run build` | Required for TypeScript or transpiled projects |
| Backend application port | `5000` | Must match the application and Dockerfile |
| Backend health endpoint | `/health`, `/api/health` | Useful for container/server health checks |
| CI scripts available | `lint`, `test`, `build` | Never call package scripts that do not exist |
| Default Git branch | `main` | Workflow trigger depends on it |
| Production API URL | `https://api.example.com` | Needed by the frontend build |
| CORS frontend URL | `https://app.example.com` | Backend production runtime configuration |

## Important

Do **not** blindly edit random values throughout the files.

For each new project, first inspect:

```bash
cat frontend/package.json
cat backend/package.json
```

or the equivalent project directories.

The main files that normally require project-specific adjustment are:

```text
frontend/Dockerfile
backend/Dockerfile
.github/workflows/ci-cd.yml
```

The frontend `nginx.conf` can usually be reused unchanged for a normal React SPA.

---

# 3. Expected Repository Structure

A clean repository should eventually look similar to:

```text
repository/
│
├── frontend/
│   ├── src/
│   ├── public/
│   ├── package.json
│   ├── package-lock.json
│   ├── vite.config.*
│   ├── Dockerfile
│   ├── .dockerignore
│   └── nginx.conf
│
├── backend/
│   ├── src/
│   ├── package.json
│   ├── package-lock.json
│   ├── Dockerfile
│   └── .dockerignore
│
└── .github/
    └── workflows/
        └── ci-cd.yml
```

If the existing repository uses:

```text
client/
server/
```

instead of:

```text
frontend/
backend/
```

keep the existing names.

Do not rename project directories just to match this guide.

---

# 4. Five-Minute Preflight Check

Because the projects already run locally, this should be quick.

## 4.1 Check Node.js

```bash
node --version
npm --version
```

Check whether the project declares its preferred Node version:

```bash
cat .nvmrc
```

or inspect:

```json
{
  "engines": {
    "node": ">=22"
  }
}
```

inside `package.json`.

Use a compatible Node major version in Docker and GitHub Actions.

---

## 4.2 Check frontend scripts

```bash
cd frontend
cat package.json
```

Look for something similar to:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "lint": "eslint ."
  }
}
```

Verify:

```bash
npm ci
npm run build
```

If `package-lock.json` does not exist, run:

```bash
npm install
```

once and commit the generated lock file.

---

## 4.3 Check backend scripts

```bash
cd backend
cat package.json
```

Example JavaScript backend:

```json
{
  "scripts": {
    "dev": "nodemon src/server.js",
    "start": "node src/server.js",
    "test": "jest"
  }
}
```

Example TypeScript backend:

```json
{
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "test": "jest"
  }
}
```

The **production start command** matters more than the development command.

Do not use:

```bash
npm run dev
```

as the normal production container command.

---

## 4.4 Verify lock files

You should have:

```text
frontend/package-lock.json
backend/package-lock.json
```

Why?

Because CI and Docker will use:

```bash
npm ci
```

`npm ci` installs dependencies from the committed lock file without rewriting it.

---

# 5. Environment Variable Strategy

Environment variables are handled differently for the frontend and backend.

## 5.1 Environment Variable Locations

| Type | Example | Where It Should Live |
|---|---|---|
| Frontend public build configuration | `VITE_API_URL` | GitHub Actions Repository Variable |
| Frontend secret | **Should not exist** | Never expose secrets in browser builds |
| Backend production secret | `JWT_SECRET` | Production server `.env` |
| Database credentials | `DATABASE_URL` | Production server `.env` |
| SMTP password | `SMTP_PASS` | Production server `.env` |
| Backend CORS URL | `CORS_ORIGIN` | Production server `.env` |
| CI-only secret | Test API key | GitHub Actions Secret |
| Local developer config | Local DB URL | Local `.env` file |

---

## 5.2 Frontend variables are public

For Vite:

```env
VITE_API_URL=https://api.example.com
```

is compiled into the browser application.

Therefore this is acceptable:

```env
VITE_API_URL=https://api.example.com
```

This is **not** acceptable:

```env
VITE_DATABASE_PASSWORD=super-secret-password
```

Anything exposed through Vite client environment variables must be treated as public.

---

## 5.3 Backend variables are runtime configuration

Example backend `.env`:

```env
NODE_ENV=production
PORT=5000

DATABASE_URL=postgresql://user:password@database:5432/app

JWT_SECRET=replace-with-a-long-random-secret

CORS_ORIGIN=https://app.example.com

SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
```

This file will later live on the server, for example:

```text
/opt/application/.env
```

It is **not** copied into the backend Docker image.

---

## 5.4 Git ignore

Ensure the repository does not commit environment secrets:

```gitignore
.env
.env.*
!.env.example
```

A safe `.env.example` can be committed:

```env
DATABASE_URL=
JWT_SECRET=
CORS_ORIGIN=
SMTP_HOST=
SMTP_PORT=
SMTP_USER=
SMTP_PASS=
```

---

# 6. Frontend Dockerization

This example assumes **React + Vite**.

Create:

```text
frontend/.dockerignore
```

with:

```dockerignore
node_modules
dist
build
coverage
.git
.github
.env
.env.*
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.DS_Store
```

## Why `.dockerignore` matters

It prevents Docker from sending unnecessary files into the build context.

Especially important:

```text
node_modules
.env
.git
dist
```

Do not copy local `node_modules` into a Linux container.

Docker should install its own dependencies.

---

## 6.1 Frontend Dockerfile

Create:

```text
frontend/Dockerfile
```

```dockerfile
# syntax=docker/dockerfile:1

# -------------------------------------------------------
# Stage 1: Build the React application
# -------------------------------------------------------
FROM node:22-alpine AS build

WORKDIR /app

# Copy dependency manifests first so Docker can cache npm ci
COPY package.json package-lock.json ./

RUN npm ci

# Copy application source
COPY . .

# Public frontend configuration supplied during docker build
ARG VITE_API_URL
ENV VITE_API_URL=${VITE_API_URL}

# Produce optimized static production files
RUN npm run build


# -------------------------------------------------------
# Stage 2: Serve the production build using Nginx
# -------------------------------------------------------
FROM nginx:stable-alpine AS runtime

# Replace the default Nginx site configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Vite normally produces /app/dist
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget -qO- http://127.0.0.1/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
```

---

## 6.2 Why the frontend uses two stages

Stage 1 needs:

```text
Node.js
npm
source code
development/build dependencies
Vite
```

because it must run:

```bash
npm run build
```

After the build, React becomes static files:

```text
index.html
assets/*.js
assets/*.css
```

The production container does **not** need Node.js.

Therefore:

```text
Node build image
      ↓
npm run build
      ↓
dist/
      ↓
copy only dist/
      ↓
Nginx runtime image
```

This produces a cleaner runtime image.

---

## 6.3 If the frontend is not Vite

If the build directory is:

```text
build/
```

instead of:

```text
dist/
```

change:

```dockerfile
COPY --from=build /app/dist /usr/share/nginx/html
```

to:

```dockerfile
COPY --from=build /app/build /usr/share/nginx/html
```

If the project uses another frontend environment variable convention, update:

```dockerfile
ARG VITE_API_URL
ENV VITE_API_URL=${VITE_API_URL}
```

accordingly.

---

# 7. Frontend Nginx Configuration

Yes, the frontend container should have an Nginx configuration when Nginx is being used to serve the built React SPA.

Create:

```text
frontend/nginx.conf
```

Use:

```nginx
server {
    listen 80;
    server_name _;

    server_tokens off;

    root /usr/share/nginx/html;
    index index.html;
    charset utf-8;

    # ---------------------------------------------------
    # Container health endpoint
    # ---------------------------------------------------
    location = /health {
        access_log off;
        default_type text/plain;
        return 200 "ok\n";
    }

    # ---------------------------------------------------
    # Vite fingerprinted static assets
    # These filenames normally include hashes and can be
    # cached for a long time.
    # ---------------------------------------------------
    location /assets/ {
        try_files $uri =404;

        expires 1y;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }

    # ---------------------------------------------------
    # Do not aggressively cache the SPA entry document.
    # A new deployment may reference new asset filenames.
    # ---------------------------------------------------
    location = /index.html {
        add_header Cache-Control "no-cache, no-store, must-revalidate";
    }

    # ---------------------------------------------------
    # React Router / SPA fallback
    # ---------------------------------------------------
    location / {
        try_files $uri $uri/ /index.html;
    }

    # ---------------------------------------------------
    # Basic compression
    # ---------------------------------------------------
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 5;

    gzip_types
        text/plain
        text/css
        application/json
        application/javascript
        application/xml
        image/svg+xml;
}
```

---

## 7.1 Why `nginx.conf` is needed

Without custom configuration, Nginx can serve:

```text
/
```

but React Router refreshes may fail.

For example, React handles:

```text
/dashboard
```

in the browser.

If the user refreshes the page, the browser requests:

```text
GET /dashboard
```

Nginx may try to find a real file:

```text
/usr/share/nginx/html/dashboard
```

which does not exist.

This line fixes SPA routing:

```nginx
try_files $uri $uri/ /index.html;
```

If the requested physical file does not exist, Nginx serves:

```text
index.html
```

and React Router takes over.

---

## 7.2 Why there are eventually two Nginx instances

This is intentional.

### Nginx inside the frontend container

```text
Frontend Container
      │
      └── Nginx
            │
            └── serves React HTML/CSS/JS
```

Its responsibility is:

```text
serve frontend static files
SPA routing
frontend caching
compression
```

### Nginx installed on the production server

Later:

```text
Internet
   │
   ▼
Host Nginx
   │
   ├── app.example.com
   │       ↓
   │   frontend container
   │
   └── api.example.com
           ↓
       backend container
```

Host Nginx handles:

```text
domain routing
HTTPS certificates
reverse proxying
```

The two Nginx instances perform different jobs.

---

# 8. Backend Dockerization

First create:

```text
backend/.dockerignore
```

```dockerignore
node_modules
dist
build
coverage
.git
.github
.env
.env.*
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.DS_Store
```

Choose **one** backend Dockerfile depending on the project.

---

## 8.1 JavaScript backend Dockerfile

Use this when production runs JavaScript directly.

```dockerfile
# syntax=docker/dockerfile:1

FROM node:22-alpine AS dependencies

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --omit=dev \
    && npm cache clean --force


FROM node:22-alpine AS runtime

WORKDIR /app

ENV NODE_ENV=production

COPY --from=dependencies /app/node_modules ./node_modules

COPY --chown=node:node package.json package-lock.json ./
COPY --chown=node:node . .

USER node

EXPOSE 5000

CMD ["npm", "start"]
```

---

## 8.2 TypeScript backend Dockerfile

Use this when source code must be compiled before production.

```dockerfile
# syntax=docker/dockerfile:1

# -------------------------------------------------------
# Stage 1: Compile application
# -------------------------------------------------------
FROM node:22-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci

COPY . .

RUN npm run build


# -------------------------------------------------------
# Stage 2: Install production dependencies
# -------------------------------------------------------
FROM node:22-alpine AS dependencies

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --omit=dev \
    && npm cache clean --force


# -------------------------------------------------------
# Stage 3: Production runtime
# -------------------------------------------------------
FROM node:22-alpine AS runtime

WORKDIR /app

ENV NODE_ENV=production

COPY --from=dependencies /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist
COPY --chown=node:node package.json package-lock.json ./

USER node

EXPOSE 5000

CMD ["node", "dist/server.js"]
```

Confirm the actual compiled entry point.

It may be:

```text
dist/server.js
dist/index.js
dist/main.js
dist/app.js
```

Do not guess.

Inspect the project.

---

# 9. Backend Runtime Requirements

## 9.1 Listen on `0.0.0.0`

A backend running inside Docker must accept connections through the container network.

Example:

```javascript
const PORT = process.env.PORT || 5000;

app.listen(PORT, "0.0.0.0", () => {
  console.log(`API listening on port ${PORT}`);
});
```

Avoid intentionally binding the container application only to:

```text
127.0.0.1
```

inside the container.

---

## 9.2 Add a health endpoint

If the project does not already have one, a simple Express endpoint is useful:

```javascript
app.get("/health", (req, res) => {
  res.status(200).json({
    status: "ok"
  });
});
```

This gives infrastructure a simple endpoint that does not depend on frontend behavior.

A more advanced application can additionally expose a readiness endpoint that verifies required dependencies.

---

## 9.3 Do not bake secrets into the image

Never do this:

```dockerfile
COPY .env .
```

Never do this:

```dockerfile
ENV JWT_SECRET=my-secret
```

The production image should contain:

```text
application code
runtime
production dependencies
```

The production server should provide:

```text
database credentials
JWT signing secrets
SMTP passwords
CORS configuration
other environment-specific settings
```

at runtime.

---

# 10. Build and Test Images Locally

Do not push to GitHub until both Docker images work locally.

---

## 10.1 Build backend

From repository root:

```bash
docker build \
  -t backend:local \
  ./backend
```

Run it:

```bash
docker run \
  --rm \
  --name backend-local \
  --env-file ./backend/.env \
  -p 5000:5000 \
  backend:local
```

Test:

```bash
curl http://localhost:5000/health
```

Expected example:

```json
{"status":"ok"}
```

---

## 10.2 Build frontend

The frontend needs to know where the API is.

For local testing:

```bash
docker build \
  --build-arg VITE_API_URL=http://localhost:5000 \
  -t frontend:local \
  ./frontend
```

Run:

```bash
docker run \
  --rm \
  --name frontend-local \
  -p 3000:80 \
  frontend:local
```

Open:

```text
http://localhost:3000
```

---

## 10.3 Important browser networking detail

When browser JavaScript calls:

```text
http://localhost:5000
```

`localhost` refers to the user's machine because the request is made by the browser.

Later, when containers communicate directly through Docker Compose, service names will be used instead of localhost.

That is covered in the server/Compose section.

---

# 11. Docker Debugging Commands

## Running containers

```bash
docker ps
```

## All containers

```bash
docker ps -a
```

## Local images

```bash
docker images
```

## View backend logs

```bash
docker logs backend-local
```

## Follow logs

```bash
docker logs -f backend-local
```

## Enter the backend container

```bash
docker exec -it backend-local sh
```

Useful commands inside:

```bash
pwd
ls -la
node --version
npm --version
printenv
```

## Inspect container metadata

```bash
docker inspect backend-local
```

## Stop a container

```bash
docker stop backend-local
```

## Delete unused build cache

Use carefully:

```bash
docker builder prune
```

---

# 12. GitHub Repository Configuration

The workflow will derive container image names from the GitHub repository automatically.

For a repository:

```text
github.com/acme/library-system
```

the workflow produces:

```text
ghcr.io/acme/library-system-frontend
ghcr.io/acme/library-system-backend
```

You do **not** need to manually invent image names for every project.

---

## 12.1 Add the frontend API URL as a GitHub Repository Variable

Go to:

```text
GitHub Repository
→ Settings
→ Secrets and variables
→ Actions
→ Variables
```

Create:

```text
Name:
VITE_API_URL

Value:
https://api.example.com
```

This value is not secret.

It becomes part of browser JavaScript.

---

## 12.2 Secrets

For this build/publish stage, GHCR authentication uses:

```text
GITHUB_TOKEN
```

which GitHub generates automatically.

Do **not** create a personal access token just for the workflow to push repository packages.

Backend production secrets are not needed to build the image.

They will be configured later on the production server.

---

# 13. Production-Oriented GitHub Actions Workflow

Create:

```text
.github/workflows/ci-cd.yml
```

```yaml
name: CI and Publish Containers

on:
  pull_request:
    branches:
      - main

  push:
    branches:
      - main

  workflow_dispatch:

# If multiple commits are pushed quickly to the same branch,
# cancel obsolete CI runs and keep the latest run.
concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

# Default workflow permission is read-only.
permissions:
  contents: read

jobs:

  # =====================================================
  # FRONTEND CI
  # =====================================================
  frontend-ci:
    name: Frontend CI
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: frontend

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7

      - name: Setup Node.js
        uses: actions/setup-node@v7
        with:
          node-version: "22"
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      # Keep this step only if "lint" exists in package.json.
      - name: Lint frontend
        run: npm run lint

      - name: Build frontend
        env:
          VITE_API_URL: ${{ vars.VITE_API_URL }}
        run: npm run build


  # =====================================================
  # BACKEND CI
  # =====================================================
  backend-ci:
    name: Backend CI
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: backend

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7

      - name: Setup Node.js
        uses: actions/setup-node@v7
        with:
          node-version: "22"
          cache: npm
          cache-dependency-path: backend/package-lock.json

      - name: Install dependencies
        run: npm ci

      # Keep this step only if "lint" exists in package.json.
      - name: Lint backend
        run: npm run lint

      # Keep this step only if real tests exist.
      - name: Test backend
        run: npm test

      # If this is a TypeScript backend, keep this.
      # If JavaScript requires no compilation, remove it.
      - name: Build backend
        run: npm run build


  # =====================================================
  # PUBLISH FRONTEND IMAGE
  # =====================================================
  publish-frontend:
    name: Publish Frontend Image

    needs:
      - frontend-ci
      - backend-ci

    # Pull requests run CI only.
    # Images are published only after main is updated.
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7

      - name: Normalize repository name
        id: repo
        shell: bash
        run: |
          echo "name=${GITHUB_REPOSITORY,,}" >> "$GITHUB_OUTPUT"

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Login to GHCR
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Generate frontend image metadata
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ steps.repo.outputs.name }}-frontend
          tags: |
            type=raw,value=latest
            type=sha,format=long,prefix=sha-

      - name: Build and push frontend image
        uses: docker/build-push-action@v7
        with:
          context: ./frontend
          push: true

          build-args: |
            VITE_API_URL=${{ vars.VITE_API_URL }}

          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

          cache-from: type=gha,scope=frontend
          cache-to: type=gha,mode=max,scope=frontend


  # =====================================================
  # PUBLISH BACKEND IMAGE
  # =====================================================
  publish-backend:
    name: Publish Backend Image

    needs:
      - frontend-ci
      - backend-ci

    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7

      - name: Normalize repository name
        id: repo
        shell: bash
        run: |
          echo "name=${GITHUB_REPOSITORY,,}" >> "$GITHUB_OUTPUT"

      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Login to GHCR
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Generate backend image metadata
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ steps.repo.outputs.name }}-backend
          tags: |
            type=raw,value=latest
            type=sha,format=long,prefix=sha-

      - name: Build and push backend image
        uses: docker/build-push-action@v7
        with:
          context: ./backend
          push: true

          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

          cache-from: type=gha,scope=backend
          cache-to: type=gha,mode=max,scope=backend
```

---

# 14. How the Workflow Works

## 14.1 Trigger behavior

| Event | Frontend CI | Backend CI | Push Images |
|---|---:|---:|---:|
| Pull request to `main` | ✅ | ✅ | ❌ |
| Push/merge to `main` | ✅ | ✅ | ✅ |
| Manual workflow dispatch | ✅ | ✅ | ❌ by default |

Pull requests validate code.

Only successfully integrated code on `main` is published as deployable images.

---

## 14.2 CI jobs

Frontend:

```text
checkout
   ↓
setup Node
   ↓
npm ci
   ↓
lint
   ↓
build
```

Backend:

```text
checkout
   ↓
setup Node
   ↓
npm ci
   ↓
lint
   ↓
test
   ↓
build (if required)
```

If either CI job fails:

```text
NO IMAGE IS PUBLISHED
```

The publish jobs have:

```yaml
needs:
  - frontend-ci
  - backend-ci
```

This creates a quality gate.

---

## 14.3 Why CI runs `npm ci` even though Docker does too

The two installs happen in different environments.

CI:

```text
GitHub runner
   ↓
npm ci
   ↓
verify source code
```

Docker:

```text
Docker build filesystem
   ↓
npm ci
   ↓
create self-contained image
```

The GitHub runner's `node_modules` directory is not automatically part of the Docker image.

---

## 14.4 Why Docker Buildx is used

Buildx is Docker's modern builder.

It supports features such as:

```text
BuildKit
build caching
multi-platform builds
advanced build outputs
```

This workflow also uses GitHub Actions cache storage:

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

which can make repeated Docker builds faster.

---

## 14.5 Why metadata-action is used

Instead of manually assembling Docker tags, the workflow uses:

```text
docker/metadata-action
```

to generate standardized image metadata.

Each successful `main` build receives:

```text
latest
sha-<full-git-commit>
```

Example:

```text
ghcr.io/acme/library-system-backend:latest

ghcr.io/acme/library-system-backend:sha-22c79918...
```

---

# 15. GHCR Image Naming and Versioning

The workflow derives image names from:

```text
github.repository
```

Example repository:

```text
ACME/Library-System
```

The workflow lowercases it:

```text
acme/library-system
```

and publishes:

```text
ghcr.io/acme/library-system-frontend
ghcr.io/acme/library-system-backend
```

No manually invented project-image variable is required.

---

## 15.1 Why use two tags

### `latest`

```text
ghcr.io/acme/library-system-backend:latest
```

Convenient for pulling the newest main-branch image.

### Git SHA

```text
ghcr.io/acme/library-system-backend:sha-22c79918...
```

Identifies the exact Git commit that created the image.

This gives traceability:

```text
Git commit
    ↓
CI run
    ↓
Docker image
    ↓
production version
```

It also enables rollback.

---

# 16. Security and Production Notes

## 16.1 Do not commit secrets

Never commit:

```text
.env
production.env
database passwords
JWT signing secrets
SMTP passwords
GHCR tokens
SSH private keys
```

---

## 16.2 `GITHUB_TOKEN` permissions are intentionally limited

Default workflow permission:

```yaml
permissions:
  contents: read
```

Only the image-publishing jobs receive:

```yaml
permissions:
  contents: read
  packages: write
```

This follows least privilege.

---

## 16.3 The server will not need GHCR write access

Later, the production server only needs to:

```text
pull images
```

not publish them.

Its registry credential should therefore be read-only.

---

## 16.4 Run backend as a non-root user

The Dockerfiles end with:

```dockerfile
USER node
```

which prevents the application process from unnecessarily running as root inside the container.

---

## 16.5 Pin versions more strictly for high-assurance production

For a classroom or normal application, these are convenient:

```text
node:22-alpine
nginx:stable-alpine
actions/checkout@v7
docker/build-push-action@v7
```

For stricter reproducibility and supply-chain controls, pin:

```text
container image digests
GitHub Actions full commit SHAs
```

instead of relying only on moving major tags.

---

## 16.6 Never put secrets into frontend build arguments

Build argument:

```text
VITE_API_URL
```

is okay because the browser must know the API URL anyway.

Do not use build args for:

```text
database password
JWT signing secret
SMTP password
private third-party API secret
```

---

# 17. Common Problems

## Problem 1 — `npm ci` fails

Typical causes:

```text
package-lock.json missing
package-lock.json out of sync with package.json
wrong Node version
```

Fix locally:

```bash
npm install
```

then commit the updated lock file after verifying it is correct.

---

## Problem 2 — Frontend Docker build succeeds but routes return 404 after refresh

Check:

```text
frontend/nginx.conf
```

and ensure:

```nginx
location / {
    try_files $uri $uri/ /index.html;
}
```

exists.

---

## Problem 3 — React uses the wrong API URL

For Vite, confirm:

```text
VITE_API_URL
```

exists in:

```text
GitHub
→ Settings
→ Secrets and variables
→ Actions
→ Variables
```

and confirm the Docker build receives:

```yaml
build-args: |
  VITE_API_URL=${{ vars.VITE_API_URL }}
```

Remember:

```text
frontend API URL is fixed at build time
```

for a normal Vite static build.

---

## Problem 4 — Backend container starts but cannot be reached

Verify the application listens on:

```text
0.0.0.0
```

not only:

```text
127.0.0.1
```

inside the container.

---

## Problem 5 — Backend Docker image crashes immediately

Check:

```bash
docker ps -a
docker logs backend-local
```

Common reasons:

```text
missing environment variable
wrong start command
wrong compiled entry point
wrong application port
missing production dependency
```

---

## Problem 6 — CI says `Missing script: lint`

The project does not have:

```json
"lint": "..."
```

in `package.json`.

Remove that CI step.

Do not create fake scripts just to satisfy copied workflow YAML.

---

## Problem 7 — CI says `Missing script: build` for backend

A JavaScript backend may not require compilation.

Remove:

```yaml
- name: Build backend
  run: npm run build
```

if the project genuinely has no build script.

---

## Problem 8 — GHCR push receives permission errors

Confirm the publish job contains:

```yaml
permissions:
  contents: read
  packages: write
```

and authentication uses:

```yaml
username: ${{ github.actor }}
password: ${{ secrets.GITHUB_TOKEN }}
```

---

# 18. One-Hour Teaching Plan

The instructor should understand more than the students need during the practical session.

Use the hour primarily to show the flow.

| Time | Activity | Main Concept |
|---:|---|---|
| 0–5 min | Inspect package files | Adapt deployment to the real application |
| 5–10 min | Explain image vs container | Docker mental model |
| 10–20 min | Add frontend Dockerfile + Nginx | Multi-stage build and SPA hosting |
| 20–30 min | Add backend Dockerfile | Runtime image and environment variables |
| 30–38 min | Build/run both locally | Verify before automation |
| 38–50 min | Add GitHub Actions | CI quality gates |
| 50–57 min | Push and observe workflow | Automation in practice |
| 57–60 min | Inspect GHCR images | Deployable artifacts |

---

## Suggested explanation during the frontend section

```text
React source
   ↓
Node build container
   ↓
npm ci
   ↓
npm run build
   ↓
dist/
   ↓
Nginx runtime container
   ↓
static production website
```

---

## Suggested explanation during the backend section

```text
Node source
   ↓
Docker image
   ↓
production dependencies
   ↓
non-root runtime user
   ↓
npm start
```

---

## Suggested explanation during CI/CD

```text
git push
   ↓
GitHub Actions
   ↓
test/build source
   ↓
Docker build
   ↓
GHCR
```

Emphasize:

> GitHub stores source code. GHCR stores built container images.

---

# 19. Final Checklist

## Local project

- [ ] Frontend already runs locally.
- [ ] Backend already runs locally.
- [ ] Frontend has `package-lock.json`.
- [ ] Backend has `package-lock.json`.
- [ ] Frontend production build succeeds.
- [ ] Backend production command is known.
- [ ] Node version is known.
- [ ] Backend port is known.
- [ ] Frontend API environment variable is known.

## Docker

- [ ] `frontend/.dockerignore` added.
- [ ] `frontend/Dockerfile` added.
- [ ] `frontend/nginx.conf` added.
- [ ] `backend/.dockerignore` added.
- [ ] Correct backend Dockerfile selected.
- [ ] Frontend Docker image builds locally.
- [ ] Backend Docker image builds locally.
- [ ] Frontend container starts.
- [ ] Backend container starts.
- [ ] Backend health endpoint works.
- [ ] React route refresh works.

## Environment configuration

- [ ] `.env` files are ignored by Git.
- [ ] No backend secrets are copied into Docker images.
- [ ] No frontend secret is exposed through `VITE_*`.
- [ ] GitHub Repository Variable `VITE_API_URL` is configured.

## GitHub Actions

- [ ] `.github/workflows/ci-cd.yml` added.
- [ ] Workflow branch name matches the repository.
- [ ] CI only calls scripts that actually exist.
- [ ] Pull request CI succeeds.
- [ ] `main` branch CI succeeds.
- [ ] Frontend image appears in GHCR.
- [ ] Backend image appears in GHCR.
- [ ] Both `latest` and `sha-*` tags appear.

---

# 20. Next Deployment Stage

After this guide is complete, the repository should produce deployable images automatically.

The next stage is:

```text
GHCR
  ↓
Fresh Ubuntu Server
  ↓
Install Docker
  ↓
Create /opt/application/
  ↓
Create production .env
  ↓
Create compose.yml
  ↓
Authenticate server to GHCR
  ↓
docker compose pull
  ↓
docker compose up -d
  ↓
DNS records
  ↓
Host Nginx reverse proxy
  ↓
Let's Encrypt / Certbot
  ↓
HTTPS
```

At that stage, the production server should **pull images**, not compile React or install Node dependencies manually.

---

# Quick Instructor Reference

Before starting a different project, check only these questions:

1. What are the real frontend and backend directory names?
2. What Node version does this project use?
3. Is the frontend Vite?
4. Does it output `dist/`?
5. What is the frontend API environment variable called?
6. Is the backend JavaScript or TypeScript?
7. What is the backend production start command?
8. What port does the backend listen on?
9. Which `lint`, `test`, and `build` scripts actually exist?
10. Is the default branch really `main`?
11. What is the production API URL?
12. Does the backend listen on `0.0.0.0`?
13. Does the backend have a health endpoint?

If those answers are known, the rest of this guide should be largely reusable.

---

# STEP 2 — Prepare the Ubuntu Production Server

The application images now exist in GHCR.

The production server should **not** clone the source code and rebuild the frontend/backend during every deployment.

Its main responsibility is to run already-built images.

The server should mainly contain:

```text
Docker
Docker Compose
production environment variables
compose.yml
host Nginx configuration
TLS certificates
persistent application data
```

---

## 2.1 Server Assumptions

The examples below use:

| Item | Example |
|---|---|
| Operating system | Ubuntu 24.04 LTS |
| Initial VPS SSH user | `ubuntu` |
| Deployment user | `deploy` |
| Server public IP | `203.0.113.10` |
| Frontend domain | `app.example.com` |
| Backend domain | `api.example.com` |
| Frontend host port | `3000` |
| Backend host port | `5000` |
| Registry | `ghcr.io` |

Change only the values that differ for the real project.

---

## 2.2 Connect to the Server

From your computer:

```bash
ssh ubuntu@203.0.113.10
```

Use the actual username supplied by the VPS provider.

---

## 2.3 Update the Server

```bash
sudo apt update
sudo apt upgrade -y
```

Install basic tools:

```bash
sudo apt install -y \
  ca-certificates \
  curl \
  git \
  nano \
  ufw
```

---

## 2.4 Create a Dedicated Deployment User

```bash
sudo adduser deploy
```

Allow administrative commands when necessary:

```bash
sudo usermod -aG sudo deploy
```

Using a dedicated deployment account is cleaner than performing every deployment as `root`.

---

## 2.5 Configure SSH Key Access

On your computer, create a key if needed:

```bash
ssh-keygen -t ed25519 -C "production-admin"
```

Copy it:

```bash
ssh-copy-id deploy@203.0.113.10
```

If `ssh-copy-id` is not available, copy the contents of:

```bash
cat ~/.ssh/id_ed25519.pub
```

to:

```text
/home/deploy/.ssh/authorized_keys
```

Then on the server:

```bash
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

Test from a **new terminal**:

```bash
ssh deploy@203.0.113.10
```

Do not disable existing SSH access until the new key has been confirmed to work.

---

## 2.6 Install Docker from Docker's Official Repository

Remove conflicting packages if installed:

```bash
sudo apt remove -y \
  docker.io \
  docker-compose \
  docker-compose-v2 \
  docker-doc \
  docker-buildx \
  podman-docker \
  containerd \
  runc
```

It is fine if some packages are reported as not installed.

Add Docker's official signing key:

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL \
  https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker's repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

Install Docker Engine, Buildx, and Compose:

```bash
sudo apt update

sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Verify:

```bash
sudo docker run --rm hello-world
```

Check Compose:

```bash
docker compose version
```

Use:

```text
docker compose
```

rather than the legacy:

```text
docker-compose
```

---

## 2.7 Allow the Deployment User to Run Docker

```bash
sudo usermod -aG docker deploy
```

Log out and reconnect:

```bash
exit
```

```bash
ssh deploy@203.0.113.10
```

Test:

```bash
docker ps
```

> **Security note:** membership in the `docker` group is effectively root-equivalent. Add only trusted deployment/admin users.

---

## 2.8 Create the Application Directory

A useful production layout is:

```text
/opt/apps/
```

Create it:

```bash
sudo mkdir -p /opt/apps
sudo chown deploy:deploy /opt/apps
```

For an example repository named `student-portal`:

```bash
mkdir -p /opt/apps/student-portal
cd /opt/apps/student-portal
```

The production folder will eventually contain:

```text
/opt/apps/student-portal/
├── compose.yml
├── .env
└── backups/
```

The source Dockerfiles remain in GitHub.

---

## 2.9 Configure the Firewall

Allow SSH first:

```bash
sudo ufw allow OpenSSH
```

Later Nginx will use HTTP/HTTPS:

```bash
sudo ufw allow 'Nginx Full'
```

Enable:

```bash
sudo ufw enable
```

Check:

```bash
sudo ufw status
```

Typical exposure:

| Port | Purpose | Public? |
|---:|---|---|
| 22 | SSH | Yes, ideally restricted |
| 80 | HTTP | Yes |
| 443 | HTTPS | Yes |
| 3000 | Frontend container | No |
| 5000 | Backend container | No |
| 5432 | PostgreSQL | No |

Docker manages iptables rules itself, so published ports must be treated carefully. In this guide, application ports bind to `127.0.0.1` so they are intended to be reachable only from the server itself.

Also check any VPS/cloud firewall or security group.

---

# STEP 3 — Authenticate the Server to GHCR

If GHCR images are private, the production server needs permission to pull them.

The production server should **not** receive write access.

---

## 3.1 Create a Read-Only Token

GitHub Container Registry currently uses a **personal access token (classic)** for external registry authentication.

Create one with:

```text
read:packages
```

only.

If the organization requires SSO, authorize the token for that organization.

For organizational production systems, a dedicated service/machine account is preferable to permanently tying production to one employee account.

---

## 3.2 Login to GHCR

On the server:

```bash
read -s CR_PAT
```

Paste the token.

Then:

```bash
echo "$CR_PAT" | docker login ghcr.io \
  -u YOUR_GITHUB_USERNAME \
  --password-stdin
```

Expected:

```text
Login Succeeded
```

Remove the shell variable:

```bash
unset CR_PAT
```

Test a pull:

```bash
docker pull ghcr.io/acme/student-portal-backend:latest
```

Use the real image created in Step 1.

---

## 3.3 Credential Responsibilities

| Location | Credential | Minimum Permission |
|---|---|---|
| GitHub Actions | `GITHUB_TOKEN` | `packages: write` in publish jobs |
| Production server | PAT classic / service-account token | `read:packages` |
| Developer computer | Optional | Only what developer needs |

Principle:

```text
CI publishes.
Production pulls.
```

---

# STEP 4 — Create Production Environment and Docker Compose

This stage connects:

```text
frontend image
+
backend image
+
database
+
runtime secrets
+
persistent storage
```

The example below uses PostgreSQL.

If the project uses MySQL, MariaDB, MongoDB, or an external managed database, change the database service accordingly.

---

## 4.1 Create Runtime Files

```bash
cd /opt/apps/student-portal
mkdir -p backups
```

Final structure:

```text
/opt/apps/student-portal/
├── compose.yml
├── .env
└── backups/
```

---

## 4.2 Create the Production `.env`

```bash
nano .env
```

Example:

```env
# Exact Docker image version.
# The first manual deployment may use latest.
IMAGE_TAG=latest

NODE_ENV=production
PORT=5000

JWT_SECRET=CHANGE_TO_A_LONG_RANDOM_SECRET

CORS_ORIGIN=https://app.example.com

POSTGRES_DB=student_portal
POSTGRES_USER=student_portal
POSTGRES_PASSWORD=CHANGE_TO_A_LONG_RANDOM_DATABASE_PASSWORD

DATABASE_URL=postgresql://student_portal:CHANGE_TO_A_LONG_RANDOM_DATABASE_PASSWORD@database:5432/student_portal

SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASS=
```

Restrict access:

```bash
chmod 600 .env
```

Generate strong secrets with:

```bash
openssl rand -hex 32
```

or:

```bash
openssl rand -base64 48
```

Use different secrets for unrelated purposes.

---

## 4.3 Production `compose.yml`

```bash
nano compose.yml
```

Example:

```yaml
services:

  database:
    image: postgres:17-alpine

    restart: unless-stopped

    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"
        ]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 10s

    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

    networks:
      - application


  backend:
    image: ghcr.io/acme/student-portal-backend:${IMAGE_TAG:-latest}

    restart: unless-stopped

    env_file:
      - .env

    depends_on:
      database:
        condition: service_healthy

    ports:
      - "127.0.0.1:5000:5000"

    healthcheck:
      test:
        [
          "CMD",
          "node",
          "-e",
          "fetch('http://127.0.0.1:5000/health').then(r=>{if(!r.ok)process.exit(1)}).catch(()=>process.exit(1))"
        ]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 15s

    init: true

    security_opt:
      - no-new-privileges:true

    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

    networks:
      - application


  frontend:
    image: ghcr.io/acme/student-portal-frontend:${IMAGE_TAG:-latest}

    restart: unless-stopped

    depends_on:
      backend:
        condition: service_healthy

    ports:
      - "127.0.0.1:3000:80"

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "wget -qO- http://127.0.0.1/health >/dev/null 2>&1 || exit 1"
        ]
      interval: 30s
      timeout: 5s
      retries: 5
      start_period: 10s

    security_opt:
      - no-new-privileges:true

    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

    networks:
      - application


volumes:
  postgres_data:


networks:
  application:
    driver: bridge
```

Replace only the actual GHCR image paths with the ones produced by the current repository.

---

## 4.4 Why Use `IMAGE_TAG`

Initial deployment:

```env
IMAGE_TAG=latest
```

Later, automatic deployment changes it to:

```env
IMAGE_TAG=sha-<full-git-commit>
```

Example:

```env
IMAGE_TAG=sha-9c1a8ff21dd29d9e7d0...
```

This means production runs an exact build rather than an ambiguous moving tag.

---

## 4.5 Why Bind to `127.0.0.1`

Frontend:

```yaml
ports:
  - "127.0.0.1:3000:80"
```

Backend:

```yaml
ports:
  - "127.0.0.1:5000:5000"
```

Flow:

```text
Host localhost:3000 → frontend container:80
Host localhost:5000 → backend container:5000
```

The public internet should go through host Nginx on ports 80/443.

---

## 4.6 Why the Database Has No Public Port

The database service deliberately has no:

```yaml
ports:
```

entry.

The backend connects internally:

```text
backend
   ↓
database:5432
```

PostgreSQL does not need to be exposed publicly for this architecture.

---

## 4.7 Why the Database Host Is `database`

Inside a container:

```text
localhost
```

means the current container itself.

Docker Compose provides service-name DNS.

Because the service is:

```yaml
database:
```

the backend reaches it using:

```text
database:5432
```

not:

```text
localhost:5432
```

---

## 4.8 Persistent Database Volume

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

means database files survive container replacement.

Be careful:

```bash
docker compose down
```

normally leaves named volumes.

But:

```bash
docker compose down -v
```

removes Compose volumes and can delete the database data.

Do not casually run `down -v` in production.

---

## 4.9 Validate Compose

```bash
docker compose config
```

This catches YAML and interpolation errors.

Do not publish the rendered output because it may include environment values.

---

# STEP 5 — Pull and Run the Application

---

## 5.1 Pull Images

```bash
cd /opt/apps/student-portal
docker compose pull
```

---

## 5.2 Start Containers

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

You want to see the database, backend, and frontend running, with health checks becoming healthy.

---

## 5.3 View Logs

All services:

```bash
docker compose logs
```

Backend:

```bash
docker compose logs backend
```

Follow:

```bash
docker compose logs -f backend
```

Database:

```bash
docker compose logs database
```

Frontend:

```bash
docker compose logs frontend
```

---

## 5.4 Test Before DNS

Frontend:

```bash
curl -I http://127.0.0.1:3000
```

Frontend health:

```bash
curl http://127.0.0.1:3000/health
```

Backend:

```bash
curl http://127.0.0.1:5000/health
```

Troubleshoot in this order:

```text
1. Container
2. localhost port
3. Nginx
4. DNS
5. HTTPS
```

---

## 5.5 Run Production Database Migrations

Use the actual project's migration mechanism.

Examples only:

### Prisma

```bash
docker compose exec backend npx prisma migrate deploy
```

### Project-defined migration script

```bash
docker compose exec backend npm run migrate
```

Do not invent a migration command if the project has none.

Do not run destructive development reset commands against production.

---

# STEP 6 — Configure DNS and Subdomains

DNS maps names to the server IP.

```text
app.example.com
      ↓
203.0.113.10
```

Nginx later decides which local container receives the request.

---

## 6.1 Recommended Naming

One project:

```text
app.example.com
api.example.com
```

Several projects:

```text
project1.example.com
api.project1.example.com

project2.example.com
api.project2.example.com
```

They may all point to the same server IP.

---

## 6.2 Add DNS A Records

Frontend:

| Field | Value |
|---|---|
| Type | `A` |
| Name | `app` |
| Value | `203.0.113.10` |
| TTL | Default/Auto |

Backend:

| Field | Value |
|---|---|
| Type | `A` |
| Name | `api` |
| Value | `203.0.113.10` |
| TTL | Default/Auto |

---

## 6.3 IPv6 Warning

Do not create an `AAAA` record unless IPv6 is actually configured on the server.

An incorrect AAAA record can cause some clients to fail even when IPv4 is correct.

---

## 6.4 Verify DNS

```bash
nslookup app.example.com
```

or:

```bash
dig +short app.example.com
```

Backend:

```bash
dig +short api.example.com
```

Both should return the server's public IP before you proceed to TLS certificate issuance.

---

# STEP 7 — Configure Host Nginx Reverse Proxy

There are now two Nginx roles:

```text
Internet
   ↓
HOST NGINX
   ↓
frontend container
   ↓
CONTAINER NGINX
   ↓
React files
```

Host Nginx handles:

```text
domain routing
reverse proxying
public ports
HTTPS termination
```

Frontend-container Nginx handles:

```text
React static files
SPA fallback
asset caching
compression
```

---

## 7.1 Install Nginx

```bash
sudo apt update
sudo apt install -y nginx
```

Enable:

```bash
sudo systemctl enable --now nginx
```

Check:

```bash
sudo systemctl status nginx
```

---

## 7.2 Create a Project Site

```bash
sudo nano /etc/nginx/sites-available/student-portal
```

Add:

```nginx
# Frontend
server {
    listen 80;
    listen [::]:80;

    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}


# Backend API
server {
    listen 80;
    listen [::]:80;

    server_name api.example.com;

    client_max_body_size 10m;

    location / {
        proxy_pass http://127.0.0.1:5000;

        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
    }
}
```

If the project accepts uploads larger than 10 MB, choose a size appropriate to that application rather than blindly increasing it.

---

## 7.3 WebSockets

If the backend uses WebSockets/Socket.IO, add to the relevant proxy location:

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

Only add this when the project needs it.

---

## 7.4 Enable the Site

```bash
sudo ln -s \
  /etc/nginx/sites-available/student-portal \
  /etc/nginx/sites-enabled/student-portal
```

Optionally disable the default site:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

---

## 7.5 Test Before Reloading

Always:

```bash
sudo nginx -t
```

Then:

```bash
sudo systemctl reload nginx
```

Test:

```bash
curl -I http://app.example.com
```

```bash
curl http://api.example.com/health
```

Do not add HTTPS until normal HTTP routing works.

---

# STEP 8 — Add HTTPS with Certbot

Let's Encrypt provides free TLS certificates.

Certbot automates certificate issuance and renewal.

---

## 8.1 Install Certbot

```bash
sudo apt install -y snapd
```

```bash
sudo snap install --classic certbot
```

Make the command available:

```bash
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
```

If that symlink already exists, leave it as-is.

Verify:

```bash
certbot --version
```

---

## 8.2 Issue Certificates

Once DNS is correct and HTTP works:

```bash
sudo certbot --nginx \
  -d app.example.com \
  -d api.example.com
```

Certbot will obtain the certificate and update Nginx.

Choose HTTP→HTTPS redirect when appropriate.

---

## 8.3 Verify HTTPS

```bash
curl -I https://app.example.com
```

```bash
curl https://api.example.com/health
```

Also test in a browser.

---

## 8.4 Test Renewal

```bash
sudo certbot renew --dry-run
```

Certificate renewal should be automated rather than performed manually every few months.

---

## 8.5 Certificate Troubleshooting

Check:

```text
DNS points to the right IP
port 80 is reachable
port 443 is allowed
Nginx is running
nginx -t succeeds
no wrong AAAA record exists
Nginx server_name exactly matches the domain
```

---

# STEP 9 — Complete Continuous Deployment

CI currently ends at:

```text
push main
   ↓
test/build
   ↓
publish images to GHCR
```

Now add:

```text
GHCR
   ↓
SSH production
   ↓
deploy exact SHA tag
   ↓
docker compose pull
   ↓
docker compose up -d
   ↓
health checks
```

---

## 9.1 Create a Dedicated GitHub Actions Deployment Key

On a trusted administrator computer:

```bash
ssh-keygen \
  -t ed25519 \
  -f github-actions-production \
  -C "github-actions-production"
```

Files:

```text
github-actions-production
github-actions-production.pub
```

The private key must never be committed.

---

## 9.2 Add the Public Key to Production

Display it:

```bash
cat github-actions-production.pub
```

On the server append it to:

```text
/home/deploy/.ssh/authorized_keys
```

Then:

```bash
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

---

## 9.3 Create a GitHub `production` Environment

In the repository:

```text
Settings
→ Environments
→ New environment
→ production
```

This keeps production credentials separated from normal CI configuration.

---

## 9.4 Add Production Secrets

Add:

| Secret | Example |
|---|---|
| `PRODUCTION_HOST` | `203.0.113.10` |
| `PRODUCTION_USER` | `deploy` |
| `PRODUCTION_SSH_KEY` | Private deployment key |
| `PRODUCTION_KNOWN_HOSTS` | Verified SSH host key entry |

Do not turn off SSH host-key checking.

Avoid production shortcuts like:

```text
StrictHostKeyChecking=no
```

---

## 9.5 Obtain the Host-Key Entry

From a trusted network:

```bash
ssh-keyscan -H 203.0.113.10
```

Verify the host fingerprint before trusting it, then save the verified output in:

```text
PRODUCTION_KNOWN_HOSTS
```

---

## 9.6 Add the Deployment Job

Add this job after the frontend/backend publishing jobs created in Step 1:

```yaml
  deploy-production:
    name: Deploy Production

    needs:
      - publish-frontend
      - publish-backend

    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    runs-on: ubuntu-latest

    environment:
      name: production

    permissions:
      contents: read

    steps:
      - name: Configure SSH
        shell: bash
        run: |
          mkdir -p ~/.ssh
          chmod 700 ~/.ssh

          printf '%s\n' "${{ secrets.PRODUCTION_SSH_KEY }}" \
            > ~/.ssh/id_ed25519

          chmod 600 ~/.ssh/id_ed25519

          printf '%s\n' "${{ secrets.PRODUCTION_KNOWN_HOSTS }}" \
            > ~/.ssh/known_hosts

          chmod 600 ~/.ssh/known_hosts

      - name: Deploy exact Git commit
        shell: bash
        env:
          DEPLOY_TAG: sha-${{ github.sha }}
        run: |
          ssh \
            "${{ secrets.PRODUCTION_USER }}@${{ secrets.PRODUCTION_HOST }}" \
            "DEPLOY_TAG='${DEPLOY_TAG}' bash -s" <<'REMOTE'
              set -euo pipefail

              cd /opt/apps/student-portal

              echo "Deploying: ${DEPLOY_TAG}"

              if grep -q '^IMAGE_TAG=' .env; then
                sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=${DEPLOY_TAG}/" .env
              else
                printf '\nIMAGE_TAG=%s\n' "${DEPLOY_TAG}" >> .env
              fi

              docker compose config >/dev/null

              docker compose pull

              docker compose up -d --remove-orphans

              echo "Waiting for services..."
              sleep 10

              curl --fail --silent --show-error \
                http://127.0.0.1:5000/health >/dev/null

              curl --fail --silent --show-error \
                http://127.0.0.1:3000/health >/dev/null

              docker compose ps
          REMOTE
```

The main project-specific value here is:

```text
/opt/apps/student-portal
```

Change it to the real deployment directory.

---

## 9.7 Full CD Flow

```text
Developer merges to main
        ↓
Frontend CI
Backend CI
        ↓
Both pass
        ↓
Build frontend SHA image
Build backend SHA image
        ↓
Push to GHCR
        ↓
SSH production
        ↓
Set IMAGE_TAG=sha-<same commit>
        ↓
Pull
        ↓
Recreate changed containers
        ↓
Health checks
        ↓
Production now runs that exact commit
```

---

## 9.8 Why SHA Tags Are Better Than Only `latest`

Suppose:

```text
sha-aaa111
```

worked yesterday, but:

```text
sha-bbb222
```

is broken.

Rollback is explicit:

```env
IMAGE_TAG=sha-aaa111
```

then:

```bash
docker compose pull
docker compose up -d --remove-orphans
```

This is easier to reason about than guessing what `latest` used to contain.

---

## 9.9 Database Migrations During CD

If migrations are required, include the project's actual production migration command.

Examples:

```bash
docker compose exec -T backend npm run migrate
```

or:

```bash
docker compose exec -T backend npx prisma migrate deploy
```

Schema changes need deployment planning.

For mature systems, prefer backward-compatible migrations:

```text
expand schema
deploy compatible code
migrate data
remove old schema later
```

---

# STEP 10 — Rollback, Backups, Logs, and Operations

A deployable system also needs a recovery path.

---

## 10.1 Roll Back the Application

Find the previous successful Git SHA.

```bash
cd /opt/apps/student-portal
nano .env
```

Change:

```env
IMAGE_TAG=sha-BROKEN
```

to:

```env
IMAGE_TAG=sha-PREVIOUS_WORKING
```

Then:

```bash
docker compose pull
docker compose up -d --remove-orphans
```

Verify:

```bash
curl --fail http://127.0.0.1:5000/health
curl --fail http://127.0.0.1:3000/health
```

---

## 10.2 Application Rollback Is Not Database Rollback

A Docker image can be switched back quickly.

A destructive database migration may not be reversible in the same way.

Therefore production needs:

```text
safe migrations
+
tested backups
```

---

## 10.3 PostgreSQL Backup

```bash
cd /opt/apps/student-portal
mkdir -p backups
```

Create a compressed dump:

```bash
docker compose exec -T database \
  sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' \
  | gzip \
  > "backups/postgres-$(date +%Y-%m-%d_%H-%M-%S).sql.gz"
```

List:

```bash
ls -lh backups/
```

A persistent Docker volume is **not** the same as a backup.

---

## 10.4 Restore Example

Use a tested backup and follow the application's maintenance procedure.

Example:

```bash
gunzip -c backups/FILE.sql.gz \
  | docker compose exec -T database \
      sh -c 'psql -U "$POSTGRES_USER" "$POSTGRES_DB"'
```

Always test restore procedures outside production before depending on them.

---

## 10.5 Backup Script

Create:

```bash
nano /opt/apps/student-portal/backup.sh
```

```bash
#!/usr/bin/env bash

set -euo pipefail

cd /opt/apps/student-portal

mkdir -p backups

BACKUP_FILE="backups/postgres-$(date +%Y-%m-%d_%H-%M-%S).sql.gz"

docker compose exec -T database \
  sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' \
  | gzip \
  > "$BACKUP_FILE"

find backups \
  -type f \
  -name 'postgres-*.sql.gz' \
  -mtime +7 \
  -delete

echo "Backup created: $BACKUP_FILE"
```

Protect and test:

```bash
chmod 700 /opt/apps/student-portal/backup.sh
```

```bash
/opt/apps/student-portal/backup.sh
```

At least one backup copy should eventually be moved off the application server.

---

## 10.6 Useful Docker Operations

Status:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs
```

Follow:

```bash
docker compose logs -f
```

Backend only:

```bash
docker compose logs --tail=100 backend
```

Restart backend:

```bash
docker compose restart backend
```

Pull:

```bash
docker compose pull
```

Apply current config:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Avoid:

```bash
docker compose down -v
```

unless volume deletion is intentional.

---

## 10.7 Disk and Resource Monitoring

Docker disk usage:

```bash
docker system df
```

Server disk:

```bash
df -h
```

Memory:

```bash
free -h
```

Processes:

```bash
top
```

Container resources:

```bash
docker stats
```

Be conservative with automatic image cleanup because older images may be needed for rollback.

---

## 10.8 Nginx Operations

Validate:

```bash
sudo nginx -t
```

Reload:

```bash
sudo systemctl reload nginx
```

Status:

```bash
sudo systemctl status nginx
```

Access log:

```bash
sudo tail -f /var/log/nginx/access.log
```

Error log:

```bash
sudo tail -f /var/log/nginx/error.log
```

---

## 10.9 Docker Daemon Logs

```bash
sudo journalctl -u docker
```

Recent events:

```bash
sudo journalctl -u docker --since "30 minutes ago"
```

---

## 10.10 Minimum Monitoring Concepts

At minimum monitor:

```text
server uptime
disk usage
memory
container status
backend health
frontend health
HTTP/HTTPS availability
certificate renewal
database backups
```

More advanced systems may later add:

```text
Prometheus
Grafana
Loki
Uptime Kuma
Sentry
OpenTelemetry
centralized alerting
```

---

# STEP 11 — Hosting Multiple Student Projects on One Server

For three student projects on one VPS, each project should have:

```text
its own /opt directory
its own Compose stack
its own database volume
its own localhost ports
its own domain names
its own Nginx site
```

---

## 11.1 Directory Layout

```text
/opt/apps/
├── project-one/
│   ├── compose.yml
│   ├── .env
│   └── backups/
│
├── project-two/
│   ├── compose.yml
│   ├── .env
│   └── backups/
│
└── project-three/
    ├── compose.yml
    ├── .env
    └── backups/
```

---

## 11.2 Unique Host Ports

| Project | Frontend Host Port | Backend Host Port |
|---|---:|---:|
| Project One | 3101 | 5101 |
| Project Two | 3102 | 5102 |
| Project Three | 3103 | 5103 |

Project One:

```yaml
frontend:
  ports:
    - "127.0.0.1:3101:80"

backend:
  ports:
    - "127.0.0.1:5101:5000"
```

Project Two:

```yaml
frontend:
  ports:
    - "127.0.0.1:3102:80"

backend:
  ports:
    - "127.0.0.1:5102:5000"
```

The backend container can still listen on `5000`; only the host port must be unique.

---

## 11.3 Unique Domains

Example:

```text
project1.example.com
api.project1.example.com

project2.example.com
api.project2.example.com

project3.example.com
api.project3.example.com
```

All records may point to the same public server IP.

Nginx routes based on hostname.

---

## 11.4 What Changes Per Project

| Area | Change? | Example |
|---|---:|---|
| GitHub repository | Yes | Different repo |
| GHCR image paths | Yes | Derived from repo |
| `/opt/apps/...` folder | Yes | Unique folder |
| Production `.env` | Yes | Unique secrets |
| Database name/user/password | Yes | Unique |
| Frontend host port | Yes | 3101/3102/3103 |
| Backend host port | Yes | 5101/5102/5103 |
| Frontend container port | Usually no | 80 |
| Backend container port | Usually no | e.g. 5000 |
| Domains | Yes | Unique |
| Nginx site | Yes | One per project |
| TLS certificate domains | Yes | Unique domains |
| Docker installation | No | Once per server |
| Host Nginx installation | No | Once per server |
| Certbot installation | No | Once per server |

---

# STEP 12 — Final Production Checklist

## Source / Docker

- [ ] Frontend runs locally.
- [ ] Backend runs locally.
- [ ] Lock files are committed.
- [ ] Frontend Dockerfile builds.
- [ ] Backend Dockerfile builds.
- [ ] Frontend `nginx.conf` exists.
- [ ] `.dockerignore` files exist.
- [ ] `.env` files are excluded from Git.
- [ ] Backend has a health endpoint.

## CI / Registry

- [ ] PR CI succeeds.
- [ ] Main CI succeeds.
- [ ] Workflow calls only scripts that really exist.
- [ ] GHCR images are published after CI passes.
- [ ] Images receive `latest`.
- [ ] Images receive `sha-<commit>` tags.

## Server

- [ ] Ubuntu is updated.
- [ ] Dedicated deploy user exists.
- [ ] SSH key login works.
- [ ] Docker is installed from official repository.
- [ ] Docker Compose plugin works.
- [ ] `/opt/apps/<app>` exists.
- [ ] Firewall is configured.
- [ ] App ports bind only to `127.0.0.1`.

## Runtime

- [ ] Server can pull GHCR images.
- [ ] Registry token is read-only.
- [ ] Production `.env` exists.
- [ ] `.env` permissions are restricted.
- [ ] Strong unique secrets are used.
- [ ] `docker compose config` passes.
- [ ] Database is not publicly exposed.
- [ ] Database uses persistent storage.
- [ ] Containers are healthy.
- [ ] Migrations are complete.

## DNS / Nginx / HTTPS

- [ ] DNS A records point to the VPS.
- [ ] No incorrect AAAA record exists.
- [ ] Host Nginx is configured.
- [ ] `nginx -t` passes.
- [ ] HTTP works before Certbot.
- [ ] HTTPS certificate succeeds.
- [ ] HTTPS works.
- [ ] `certbot renew --dry-run` succeeds.

## CD / Recovery

- [ ] Dedicated GitHub Actions SSH key exists.
- [ ] GitHub `production` environment exists.
- [ ] Known-host verification is enabled.
- [ ] CD deploys exact SHA tags.
- [ ] Health checks run after deployment.
- [ ] Previous SHA can be identified.
- [ ] Rollback procedure is understood.
- [ ] Database backup works.
- [ ] Restore procedure has been tested.
- [ ] Off-server backup strategy exists.

---

# Instructor Quick Reference

Before each project, answer:

1. What are the frontend/backend folder names?
2. What Node version is supported?
3. Is frontend Vite?
4. What is the frontend output folder?
5. What is the frontend API variable called?
6. Is backend JavaScript or TypeScript?
7. What is the backend production start command?
8. What backend port is used?
9. Which lint/test/build scripts actually exist?
10. What health endpoint exists?
11. What GHCR image paths were generated?
12. Which database does the project use?
13. Which backend environment variables are required?
14. Which frontend/backend domains will be used?
15. Which localhost ports are free?
16. Does the project require migrations?
17. Does it use WebSockets?
18. Does it accept large uploads?
19. What backup command matches its database?
20. What previous SHA will be used for rollback?

---

# Complete End-to-End Flow

```text
1. Existing app runs locally
                ↓
2. Frontend Dockerfile
                ↓
3. Frontend nginx.conf
                ↓
4. Backend Dockerfile
                ↓
5. Test images locally
                ↓
6. GitHub Actions CI
                ↓
7. Publish SHA-tagged images to GHCR
                ↓
8. Prepare Ubuntu VPS
                ↓
9. Install Docker + Compose
                ↓
10. Create /opt/apps/<app>
                ↓
11. GHCR read authentication
                ↓
12. Production .env
                ↓
13. compose.yml
                ↓
14. docker compose pull
                ↓
15. docker compose up -d
                ↓
16. Database migrations
                ↓
17. localhost health checks
                ↓
18. DNS A records
                ↓
19. Host Nginx reverse proxy
                ↓
20. Verify HTTP
                ↓
21. Certbot / Let's Encrypt
                ↓
22. Verify HTTPS + renewal
                ↓
23. GitHub Actions deployment job
                ↓
24. Deploy exact sha-<commit>
                ↓
25. Post-deployment health checks
                ↓
26. Rollback + backups + logs
```

---

# Official References

- Docker Engine on Ubuntu: `https://docs.docker.com/engine/install/ubuntu/`
- Docker Compose: `https://docs.docker.com/compose/`
- Dockerfile reference: `https://docs.docker.com/reference/dockerfile/`
- GitHub Container Registry: `https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry`
- GitHub Actions workflow syntax: `https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax`
- Nginx proxy module: `https://nginx.org/en/docs/http/ngx_http_proxy_module.html`
- Certbot Nginx instructions: `https://certbot.eff.org/instructions?ws=nginx&os=snap`
