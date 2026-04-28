---
tags: [dotnet, advanced, devops, testing]
aliases: [CI/CD, GitHub Actions, Pipeline, Release, Environment Promotion]
level: Advanced
---

# CI-CD and DevOps

> **One-liner**: A typical .NET pipeline runs **build → test → publish** on every push, pushes a versioned container image to a registry, then promotes it through **dev → staging → prod** environments — most commonly on **GitHub Actions** or **Azure DevOps**.

---

## Quick Reference

| Stage | What it does | Tools |
|-------|--------------|-------|
| Restore | Pull NuGet packages | `dotnet restore`, NuGet cache |
| Build | Compile in Release | `dotnet build -c Release --no-restore` |
| Test | Unit + integration | `dotnet test --no-build` |
| Coverage | Collect & report | `coverlet`, `ReportGenerator` |
| Static analysis | Lint, security | Roslyn analyzers, `dotnet format`, CodeQL |
| Publish | Produce artifacts | `dotnet publish`, container image |
| Push | Image to registry | GHCR, Docker Hub, ACR, ECR |
| Deploy | Roll out | Helm, K8s, Azure App Service, ACA, ECS |
| Verify | Smoke / health | curl /health, synthetic tests |

| Convention | Detail |
|------------|--------|
| Versioning | SemVer + GitVersion / `Nerdbank.GitVersioning` |
| Tags | `v1.2.3`, image tags `:1.2.3` and `:latest` (mutable) |
| Environments | `dev` (auto), `staging` (auto), `prod` (manual approval) |
| Secrets | GitHub Environments / GitHub OIDC → cloud, Key Vault |
| Branching | trunk-based or GitHub Flow; PR builds gated by CI |

---

## Core Concept

CI/CD enforces that "can it build and pass tests?" never depends on a developer's laptop. Every push runs the pipeline; if it fails, the change doesn't merge. The artifact built by CI is the **same** artifact promoted to prod — never rebuild for a different environment.

For .NET, the canonical pipeline is six steps: cache restore, build once with `--no-restore`, test with `--no-build`, publish with `--no-restore --no-build`, build the container image, push it. Each later step skipping work that earlier ones did keeps things fast.

Modern best practice is **OIDC** (OpenID Connect) for cloud authentication: GitHub Actions issues a short-lived federated token to Azure/AWS/GCP, no long-lived secrets in CI. Combined with **GitHub Environments** for protection rules (required reviewers, wait timers, deployment branches) you get audited, gated releases.

The shape of "DevOps" goes beyond CI: monitoring, on-call, incident response, IaC (Bicep/Terraform), feature flags. Treat the pipeline as the *system that ships software safely*, not just a build script.

---

## Syntax & API

### Minimal GitHub Actions — build + test
```yaml
# .github/workflows/ci.yml
name: CI
on:
  push: { branches: [main] }
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: 9.0.x }

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build -c Release --no-restore

      - name: Test
        run: dotnet test -c Release --no-build --logger trx --collect:"XPlat Code Coverage"

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with: { name: test-results, path: '**/TestResults/**' }
```

### Cache NuGet packages
```yaml
- uses: actions/cache@v4
  with:
    path: ~/.nuget/packages
    key: nuget-${{ runner.os }}-${{ hashFiles('**/packages.lock.json', '**/*.csproj') }}
    restore-keys: nuget-${{ runner.os }}-
```

### Matrix — multi-OS / multi-version
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest]
    dotnet: [8.0.x, 9.0.x]
runs-on: ${{ matrix.os }}
steps:
  - uses: actions/setup-dotnet@v4
    with: { dotnet-version: ${{ matrix.dotnet }} }
```

### Build & push container (OIDC to Azure)
```yaml
deploy:
  needs: build
  runs-on: ubuntu-latest
  permissions:
    id-token: write
    contents: read
  steps:
    - uses: actions/checkout@v4

    - uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Login to ACR
      run: az acr login --name myacr

    - uses: docker/setup-buildx-action@v3

    - uses: docker/build-push-action@v6
      with:
        context: .
        push: true
        platforms: linux/amd64,linux/arm64
        tags: |
          myacr.azurecr.io/shop-api:${{ github.sha }}
          myacr.azurecr.io/shop-api:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

### Promote with GitHub Environments
```yaml
deploy-staging:
  environment: staging
  needs: build
  steps:
    - run: ./deploy.sh staging ${{ github.sha }}

deploy-prod:
  environment:
    name: production
    url: https://app.example.com
  needs: deploy-staging
  steps:
    - run: ./deploy.sh prod ${{ github.sha }}
```

### Code coverage report
```yaml
- name: Generate coverage report
  run: |
    dotnet tool install -g dotnet-reportgenerator-globaltool
    reportgenerator -reports:**/coverage.cobertura.xml -targetdir:coverage -reporttypes:Html

- uses: actions/upload-artifact@v4
  with: { name: coverage, path: coverage }
```

### CodeQL security scan
```yaml
analyze:
  runs-on: ubuntu-latest
  permissions: { security-events: write, contents: read }
  steps:
    - uses: actions/checkout@v4
    - uses: github/codeql-action/init@v3
      with: { languages: csharp }
    - run: dotnet build -c Release
    - uses: github/codeql-action/analyze@v3
```

### Release versioning with Nerdbank.GitVersioning
```json
// version.json
{
  "version": "1.2",
  "publicReleaseRefSpec": [ "^refs/heads/main$", "^refs/tags/v\\d+\\.\\d+" ],
  "cloudBuild": { "buildNumber": { "enabled": true } }
}
```

### Azure DevOps pipeline (alternative to Actions)
```yaml
trigger: [ main ]
pool: { vmImage: ubuntu-latest }

steps:
  - task: UseDotNet@2
    inputs: { version: 9.0.x }
  - script: dotnet restore
  - script: dotnet build -c Release --no-restore
  - script: dotnet test -c Release --no-build --logger trx --collect:"XPlat Code Coverage"
  - task: PublishTestResults@2
    inputs: { testResultsFormat: VSTest, testResultsFiles: '**/*.trx' }
```

### Trivy / vulnerability scan
```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: myacr.azurecr.io/shop-api:${{ github.sha }}
    severity: HIGH,CRITICAL
    exit-code: 1
```

---

## Common Patterns

```yaml
# Pattern: PR title-based concurrency to cancel stale builds
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

```yaml
# Pattern: deploy only on tag
on:
  push:
    tags: ['v*.*.*']
```

```yaml
# Pattern: dependabot auto-merge for minor bumps
# .github/workflows/dependabot.yml
on: pull_request
permissions: { contents: write, pull-requests: write }
jobs:
  auto-merge:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - uses: dependabot/fetch-metadata@v2
        id: meta
      - if: steps.meta.outputs.update-type == 'version-update:semver-patch'
        run: gh pr merge --auto --squash "$PR_URL"
        env: { PR_URL: ${{ github.event.pull_request.html_url }}, GH_TOKEN: ${{ secrets.GITHUB_TOKEN }} }
```

```bash
# Pattern: produce SBOM for supply-chain transparency
dotnet sbom-tool generate -b artifacts -bc . -pn shop-api -ps Org -nsb https://sbom.example
```

---

## Gotchas & Tips

- **Build once, deploy many times** — the artifact promoted to prod must be the same one tested in staging. Rebuilds drift.
- **Lock NuGet versions** — use `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>` and commit `packages.lock.json` to make CI reproducible.
- **Tests as gates** — failed tests must block merge. PR-only branch protection on `main`.
- **OIDC > long-lived secrets** — GitHub → cloud federation uses short-lived tokens. Long-lived `AZURE_CREDENTIALS` JSON is a leak in waiting.
- **Cache wisely** — NuGet cache by lockfile hash, not blanket `*`. A blanket key serves stale packages.
- **Use `--no-restore` and `--no-build`** in later steps — saves seconds per command and prevents accidental rebuild on different inputs.
- **Pin action versions** — `actions/checkout@v4`, not `@main`. Major version pin is the sweet spot between safety and getting fixes.
- **Run integration tests against a real DB** in CI using Testcontainers or service containers — InMemory hides real bugs.
- **`continue-on-error`** is rarely the right answer — fix the flake or quarantine the test.
- **Image tags should be immutable identifiers** — `:1.2.3` or `:<sha>`. `:latest` is convenient but ambiguous; never deploy "latest" to prod.
- **Required environment approvers** for prod — GitHub Environments lets you enforce manual review on the deploy job. Audit log is built in.
- **Keep PR pipeline under ~10 minutes** — slower than that and developers context-switch. Parallelize, cache, prune.
- **Monitor pipeline health** — flaky tests + slow CI compound. Treat CI as a product.

---

## See Also

- [[20 - Testing]]
- [[17 - Docker and Containers]]
- [[05 - Security and Auth]]
- [[06 - Performance Optimization]]
