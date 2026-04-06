---
name: azure-container-apps-deploy
description: "Scaffold, extend, or diagnose Azure Container Apps deployments using azd and Bicep. Covers azure.yaml setup, Bicep IaC modules, Dockerfiles, GitHub Actions CI/CD, and troubleshooting common deployment failures. WHEN: deploy to Container Apps, set up azd deployment, create azure.yaml, scaffold Bicep infra, add container app service, diagnose azd provision, fix deployment pipeline, container app not starting, azd deploy fails, GitHub Actions deploy workflow, containerize .NET or React app for Azure."
license: MIT
metadata:
  author: twoday
  version: "1.0.0"
---

# Azure Container Apps Deploy

> Scaffold, extend, or diagnose Azure Container Apps deployments orchestrated by Azure Developer CLI (`azd`) with Bicep IaC and GitHub Actions CI/CD.

## Rules

1. Follow the selected workflow (scaffold / add-service / diagnose) — do not mix.
2. Always read existing project structure before generating files.
3. Never hardcode secrets — use environment variables, Key Vault, or managed identity.
4. Bicep MUST be subscription-scoped (`targetScope = 'subscription'`).
5. Dockerfiles MUST run as non-root and include health checks.
6. Destructive actions (deleting resources, overwriting infra files) require user confirmation.
7. Validate generated files against the conventions in the reference docs.

## Workflows

| Trigger | Workflow | Description |
|---------|----------|-------------|
| "scaffold deployment" / "set up azd" | [scaffold](#scaffold) | Create azure.yaml, Bicep modules, Dockerfiles, and GitHub Actions from scratch |
| "add service" / "add container app" | [add-service](#add-service) | Add a new deployable service to an existing setup |
| "fix deployment" / "deploy failing" | [diagnose](#diagnose) | Troubleshoot deployment failures using known pitfall patterns |

If the user's intent is ambiguous, ask which workflow they need.

---

## Scaffold

Set up a complete Azure Container Apps deployment from scratch.

### Input

Gather from the user or infer from the codebase:
- App name (used for resource naming: `rg-{app}-{env}`)
- Azure region (must match any existing resource group)
- Services to deploy (project paths, languages, ports)
- Database needs (PostgreSQL, Redis, etc.)
- Auth requirements (Entra ID, JWT)

### Steps

1. **Scan the codebase**
   - Identify deployable projects (look for `.csproj`, `package.json`, `Dockerfile`)
   - Determine languages and frameworks
   - Note port configurations

2. **Create `azure.yaml`**
   - One entry per deployable service
   - Follow conventions from [azure-yaml-reference](#azure-yaml-conventions)
   - Validate: every service has `project`, `language`, `host`, `docker` fields

3. **Create Bicep infrastructure**
   - `infra/main.bicep` — subscription-scoped orchestrator
   - `infra/main.parameters.json` — parameter mapping with `${ENV_VAR}` syntax
   - Module files in `infra/modules/`:
     - `managed-identity.bicep` — user-assigned identity for ACR pull
     - `container-registry.bicep` — ACR + AcrPull role assignment
     - `container-apps-env.bicep` — Container Apps environment (no Log Analytics by default)
     - One container app module per service
     - Additional modules for databases, caches, etc.
   - Apply conventions from [bicep-reference](#bicep-conventions)

4. **Create Dockerfiles**
   - One per service, placed in the service project directory
   - Apply conventions from [dockerfile-reference](#dockerfile-conventions)
   - Create `.dockerignore` at repo root if missing

5. **Create GitHub Actions workflow**
   - `.github/workflows/deploy.yml`
   - Dual auth: `azure/login` + `azd auth login`
   - Apply conventions from [github-actions-reference](#github-actions-conventions)

6. **Create deployment documentation**
   - `docs/deployment.md` with setup steps, required secrets, verification checklist

7. **Verify**
   - Confirm all files are syntactically valid
   - List required GitHub Actions secrets for the user to configure
   - Suggest running `azd provision --preview` to validate

---

## Add Service

Add a new deployable service to an existing Azure Container Apps setup.

### Input

- Service name
- Project path
- Language / framework
- Exposed port
- Any new infrastructure dependencies

### Steps

1. **Read existing configuration**
   - Parse `azure.yaml` for current services
   - Read `infra/main.bicep` for existing modules and outputs

2. **Add to `azure.yaml`**
   - Append new service entry following existing patterns

3. **Create Bicep module**
   - New `infra/modules/{service}-container-app.bicep`
   - Wire into `main.bicep` (module reference, outputs, tag)
   - Include placeholder image for first deploy

4. **Create Dockerfile** (if not existing)
   - Apply [dockerfile-reference](#dockerfile-conventions)

5. **Update workflow** (if needed)
   - Add any new environment variables or build steps

6. **Verify**
   - Confirm `azure.yaml` is valid
   - Confirm Bicep compiles (`az bicep build`)

---

## Diagnose

Troubleshoot a failing Azure Container Apps deployment.

### Input

- Error message or failed step from the user
- Access to workflow logs, Bicep files, azure.yaml

### Steps

1. **Classify the failure**
   - Match against [known pitfalls](#known-pitfalls)
   - If no match, read the full error context

2. **Identify root cause**
   - Read the relevant files (workflow, Bicep, Dockerfile, azure.yaml)
   - Cross-reference with conventions

3. **Apply fix**
   - Make the minimal targeted fix
   - Explain what went wrong and why the fix works

4. **Verify**
   - Check for cascading issues
   - Suggest re-running the pipeline

---

## Reference: azure.yaml Conventions

- Each service MUST have `project`, `language`, `host`, and `docker` fields.
- `language` MUST be a valid azd value: `dotnet`, `js`, `ts`, `py`, `java`. **NOT** `docker`.
- `docker.path` is relative to the `project` directory, not the repo root.
- `docker.context` is also relative to `project`. Use `../..` when Dockerfile needs repo root.
- Do NOT use `image:` when `docker:` block is present — azd builds from Dockerfile.
- Do NOT use `host: aspire` for deployment when Aspire uses `AddNpmApp` or other `executable.v0` resources.
- Container apps are matched to Bicep resources via the `azd-service-name` tag.

### Template

```yaml
name: <app-name>
services:
  <service-name>:
    project: ./<path-to-project>
    language: <dotnet|js|ts|py|java>
    host: containerapp
    docker:
      path: ./Dockerfile          # relative to project dir
      context: ../..              # relative to project dir
```

---

## Reference: Bicep Conventions

- `main.bicep` MUST use `targetScope = 'subscription'`.
- Resource group naming: `rg-{app-name}-{env}`.
- All resources MUST include `azd-env-name` and `application` tags.
- `azd deploy` requires this output: `output AZURE_CONTAINER_REGISTRY_ENDPOINT string = ...`
  - Do NOT name it `AZURE_CONTAINER_REGISTRY_LOGIN_SERVER`.
- Container app modules MUST use a placeholder image for first deploy:
  ```bicep
  var placeholderImage = 'mcr.microsoft.com/k8se/quickstart:latest'
  ```
- Auto-generate secrets in Bicep with `uniqueString()` instead of requiring manual GitHub secrets:
  ```bicep
  var dbPassword = 'P${uniqueString(subscription().subscriptionId, rgName, 'db')}x!1A'
  ```
- Do NOT create Log Analytics workspaces — org policies often block it. Use `azure-monitor` destination or accept existing workspace ID as optional param.
- Parameter flow: `main.parameters.json` uses `${ENV_VAR}` → azd resolves from env → populated by Bicep outputs + workflow env.

---

## Reference: Dockerfile Conventions

### .NET API
```dockerfile
# Build
FROM mcr.microsoft.com/dotnet/sdk:<version> AS build
WORKDIR /src
COPY **/*.csproj ./    # restore layer cache
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish

# Runtime
FROM mcr.microsoft.com/dotnet/aspnet:<version>
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
USER $APP_UID
HEALTHCHECK CMD wget --spider -q http://localhost:8080/health || exit 1
ENTRYPOINT ["dotnet", "<ProjectName>.dll"]   # MUST include ENTRYPOINT
```

### React SPA (Node → Nginx)
```dockerfile
# Build
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
COPY . .
RUN npm run build

# Runtime
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf   # SPA fallback
EXPOSE 80
USER nginx
CMD ["nginx", "-g", "daemon off;"]
```

### .dockerignore
Always include at repo root: `.git`, `node_modules`, `bin`, `obj`, `dist`, `.env*`, `docs/`, `infra/`, `openspec/`.

---

## Reference: GitHub Actions Conventions

### Dual authentication (required)

`azure/login@v2` authenticates `az` CLI only. `azd` needs its own login:

```yaml
# 1. Azure CLI
- uses: azure/login@v2
  with:
    creds: '{"clientId":"${{ secrets.AZURE_CLIENT_ID }}","clientSecret":"${{ secrets.AZURE_CLIENT_SECRET }}","tenantId":"${{ secrets.AZURE_TENANT_ID }}","subscriptionId":"${{ secrets.AZURE_SUBSCRIPTION_ID }}"}'

# 2. azd CLI
- run: azd auth login --client-id "${{ secrets.AZURE_CLIENT_ID }}" --client-secret "${{ secrets.AZURE_CLIENT_SECRET }}" --tenant-id "${{ secrets.AZURE_TENANT_ID }}"
```

### Critical rules

- Do NOT add `id-token: write` to `permissions` — it forces OIDC flow even when using SP + secret.
- `creds` MUST be a single-line JSON string (multiline YAML literals don't interpolate secrets).
- Both `azd provision` and `azd deploy` steps MUST have `AZURE_SUBSCRIPTION_ID` in their `env` block.
- `AZURE_LOCATION` must match the existing resource group's region.

### Required secrets

| Secret | Purpose |
|--------|---------|
| `AZURE_TENANT_ID` | SP tenant |
| `AZURE_SUBSCRIPTION_ID` | Target subscription |
| `AZURE_CLIENT_ID` | SP client ID (deployment) |
| `AZURE_CLIENT_SECRET` | SP client secret |

Additional app-specific secrets (Entra ID client ID, etc.) depend on the application.

### Environment variable flow

```
GitHub Secrets → Workflow env → azd env vars → main.parameters.json ${VAR} → Bicep params → Azure resources
```

---

## Known Pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| OIDC token request when using SP+secret | `id-token: write` in permissions | Remove `id-token: write` |
| `creds` secrets not interpolated | Multiline YAML literal | Single-line quoted JSON |
| `azd` commands fail with auth error | Only `az` was logged in | Add `azd auth login` step |
| Container image not found on first deploy | ACR empty | Placeholder image in Bicep |
| `AZURE_CONTAINER_REGISTRY_ENDPOINT` missing | Wrong Bicep output name | Name output exactly `AZURE_CONTAINER_REGISTRY_ENDPOINT` |
| `executable.v0` unsupported | `host: aspire` with non-container resources | Use explicit service definitions |
| "must specify language or image" | Missing `language` field in azure.yaml | Add valid `language` value |
| Resource group location conflict | Hardcoded location ≠ existing RG | Match `AZURE_LOCATION` to existing RG region |
| Log Analytics creation denied | Org policy blocks workspace creation | Use `azure-monitor` destination or skip |
| Missing ENTRYPOINT | Dockerfile has no entrypoint | Add `ENTRYPOINT` directive |
| TypeScript build errors in Docker | Strict type checks pass locally but fail in clean build | Fix the type errors (Docker builds are clean, no cache) |
| `docker.path` file not found | Path relative to repo root instead of project dir | Make path relative to `project` directory |
