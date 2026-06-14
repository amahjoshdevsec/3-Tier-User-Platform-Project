# CLAUDE.md

Guidance for Claude Code (and other AI assistants) working in this repository.

## Project Overview

A 3-tier "User Management" web app (DevOps Shack demo project) used to teach
CI/CD, Docker, Kubernetes (EKS) and DevSecOps pipelines:

- **Presentation tier** (`client/`): React 17 app bundled with Webpack into
  static assets.
- **Application tier** (`server/`): Express.js REST API.
- **Data tier**: MySQL (`users` table: `id`, `name`, `email`, `role`).

In production the Express server serves the built React app as static files
and exposes `/api/*` routes — there is no separate frontend deployment.

## Repository Structure

```
client/                  React + Webpack frontend
  src/
    App.js               Main app component (inline form + user list)
    components/          UsersList.js, UserItem.js (alternate UI using api/users.js)
    api/users.js          axios client hitting http://localhost:5000/api/users
  public/                 Build output (bundle.js, index.html, style.css) — generated
  webpack.config.js       Production-mode build, outputs to client/public

server/                  Express backend
  server.js              Actual entrypoint (npm start). Inline routes + DB init + static file serving
  app.js                 Alternative app factory using routes/userRoutes.js (NOT wired to server.js)
  routes/
    userRoutes.js         Routes -> controllers/userController.js (used by app.js)
    users.js              Standalone router with inline db queries (also not mounted by server.js)
  controllers/userController.js
  models/userModel.js
  config/db.js            mysql connection (host/user/pass/db via env vars)

k8-manifests/
  qa/                     Deployment, Service, Ingress for the `qa` namespace
  prod/                   Deployment, Service, Ingress for the `prod` namespace

.github/workflows/
  qa-cicd.yml             Main CI/CD pipeline (triggers on push to `qa`)
  prod-cd.yaml            Promotion pipeline (triggers on push to `main`)

Dockerfile                Multi-stage-ish build: builds client, copies build output into server/public
sonar-project.properties  SonarQube project config
```

## Known Inconsistencies (be aware, don't "fix" silently)

- **Two backend wiring styles coexist**: `server/server.js` (the real
  entrypoint, defines all `/api/users*` routes inline and serves
  `client/public`) vs. `server/app.js` + `routes/userRoutes.js` +
  `controllers/` + `models/` (a layered MVC structure that is **not**
  currently used — `app.js` is never required anywhere). `routes/users.js`
  is a third, also-unused, inline implementation. If asked to add an API
  endpoint, add it to `server/server.js` unless told to migrate to the MVC
  structure (and if you do migrate, remove the dead alternatives rather than
  adding a fourth variant).
- **Two React UIs coexist**: `client/src/App.js` (rendered by `index.js`, has
  its own inline form/list and fetch calls) vs.
  `client/src/components/UsersList.js` + `UserItem.js` (uses
  `client/src/api/users.js`, currently unused by `App.js`). Edit/Delete
  buttons in both are partially wired (Edit is a stub).
- `client/src/api/users.js` hardcodes `http://localhost:5000/api/users` —
  won't work as-is when client/server are served from the same origin in
  Docker/K8s.
- Default DB credentials (`appuser`/`password123`, `root`/`password`) are
  hardcoded as fallback values in `server/server.js` and `server/config/db.js`.
  Real credentials come from the `mysql-secret` K8s Secret via env vars
  (`DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`). Never put real secrets in
  source files.

## Development Workflows

### Running locally
```bash
# Backend (serves API + built client on :5000)
cd server && npm install && npm start

# Frontend dev build (writes to client/public)
cd client && npm install && npm start    # webpack --mode development
# or for a production bundle:
npm run build                            # webpack --mode production
```
The server expects a MySQL instance reachable via `DB_HOST`/`DB_USER`/
`DB_PASSWORD`/`DB_NAME` env vars (defaults assume a `mysql` host).

### Docker
```bash
docker build -t nodejs-app .
```
Builds the client, copies the build output into `server/public`, installs
production server deps, and runs as a non-root `appuser` on port 5000.

## CI/CD Pipeline

Two GitHub Actions workflows implement a QA → Prod promotion flow:

1. **`qa-cicd.yml`** (push to `qa`, paths: `client/**`, `server/**`, workflow
   file itself):
   - Security/compliance scans: Gitleaks, Checkov (terraform/k8s/dockerfile),
     Trivy filesystem scans (client & server) — all non-blocking
     (`exit-code 0` / `continue-on-error`).
   - Lint and test for both `client/` and `server/` (non-blocking,
     `--if-present`).
   - SonarQube scan + quality gate (gate failure currently does not block
     via `continue-on-error: true`).
   - Build client, build & push Docker image to Docker Hub
     (`devopsshack/nodejs-app:<sha>` and `:latest`), Trivy image scan + SBOM
     generation.
   - Updates `k8-manifests/qa/app-deployment.yaml` image tag to `<sha>`,
     commits to `qa` branch, and deploys to the `qa` namespace on EKS
     (`my-cluster`, `ap-south-1`) via OIDC.

2. **`prod-cd.yaml`** (push to `main`, paths limited to
   `k8-manifests/prod/app-deployment.yaml` and the workflow file itself):
   - Pulls `devopsshack/nodejs-app:latest` (the QA-validated image), retags
     as `devopsshack/nodejs-app:prod-<sha>`, pushes to Docker Hub.
   - Updates `k8-manifests/prod/app-deployment.yaml` with the new tag,
     commits to `main`.
   - Deploys `k8-manifests/prod/` to the `prod` namespace on EKS.

**Implication for AI assistants**: editing `k8-manifests/prod/app-deployment.yaml`
on `main` will trigger a production deployment using whatever image is
currently tagged `:latest` in Docker Hub (i.e. the latest QA build) — be
careful about touching that file outside of the automated promotion commits.
Editing files under `client/` or `server/` should generally happen via a
branch that flows through `qa` first.

## Conventions

- Plain JavaScript (no TypeScript) throughout; CommonJS in `server/`, ES
  modules/JSX in `client/`.
- No test suite currently exists (`npm test --if-present` is a no-op); CI
  steps for lint/test are best-effort and won't fail the pipeline.
- Kubernetes manifests use `qa` and `prod` namespaces with an ALB ingress
  controller; DB credentials come from a `mysql-secret` Secret.
- Keep new server endpoints consistent with the structure already used by
  `server.js` (inline `db.query` calls with `?` placeholders for
  parameterization — never interpolate user input directly into SQL).
