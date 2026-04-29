---
tags: [dotnet, advanced, devops]
aliases: [Docker, Dockerfile, Multi-stage Build, docker-compose]
level: Advanced
---

# Docker and Containers

> **One-liner**: Package an ASP.NET Core app as an OCI image using a **multi-stage Dockerfile** (SDK builds, runtime serves), or let the SDK do it with **`dotnet publish /t:PublishContainer`** — and orchestrate locally with **docker-compose**.

---

## Quick Reference

| Image | Purpose | Approx size |
|-------|---------|-------------|
| `mcr.microsoft.com/dotnet/sdk:9.0` | Build (SDK) | ~700 MB |
| `mcr.microsoft.com/dotnet/aspnet:9.0` | Runtime for ASP.NET | ~210 MB |
| `mcr.microsoft.com/dotnet/runtime:9.0` | Runtime for console worker | ~190 MB |
| `mcr.microsoft.com/dotnet/aspnet:9.0-alpine` | Smaller, musl libc | ~100 MB |
| `mcr.microsoft.com/dotnet/runtime-deps:9.0-noble-chiseled` | For self-contained / AOT | ~25 MB |

| Concept | Detail |
|---------|--------|
| Multi-stage build | One stage compiles, another copies output — final image has no SDK |
| Layer cache | Order COPYs from least- to most-changed for fast rebuilds |
| .dockerignore | Exclude `bin/`, `obj/`, `node_modules/`, secrets |
| Non-root user | `USER app` — built into chiseled images by default |
| Healthcheck | `HEALTHCHECK --interval=30s CMD curl ... /health` |
| Image signing / SBOM | `docker scout`, `cosign`, attestations |
| Container ports | `EXPOSE 8080`; ASP.NET defaults to `http://+:8080` in containers |

---

## Core Concept

A container is a process running with isolated filesystem, network, and PID namespaces — packaged as a tarball ("image") with a tiny manifest. .NET integrates well: every official image is the same base across cloud providers, the runtime detects container limits via cgroups (CPU/memory), and the GC adjusts heap sizing accordingly.

A **multi-stage Dockerfile** is the standard pattern: stage 1 uses the SDK image to restore + publish, stage 2 copies just the publish output into a slim runtime image. The result is small, has no compiler, and ships only what the app needs.

The **SDK can build images directly** (.NET 7+) — `dotnet publish /t:PublishContainer` produces an OCI image without a Dockerfile. Saves ceremony for typical apps. For complex stacks (custom OS packages, multi-app images), keep the Dockerfile.

For local multi-service development, **docker-compose** wires up your app + its dependencies (Postgres, Redis, RabbitMQ) into one `docker compose up`. For production, deploy to Kubernetes / Azure Container Apps / ECS — compose is for dev only.

---

## Syntax & API

### Multi-stage Dockerfile
```dockerfile
# syntax=docker/dockerfile:1.7
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Restore in its own layer for caching
COPY *.sln ./
COPY src/Shop.Api/*.csproj src/Shop.Api/
COPY src/Shop.Domain/*.csproj src/Shop.Domain/
RUN dotnet restore src/Shop.Api/Shop.Api.csproj

# Copy and build
COPY . .
RUN dotnet publish src/Shop.Api/Shop.Api.csproj -c Release -o /app /p:UseAppHost=false

# --- Runtime ---
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app
COPY --from=build /app .

USER app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_RUNNING_IN_CONTAINER=true

ENTRYPOINT ["dotnet", "Shop.Api.dll"]
```

### .dockerignore
```
bin/
obj/
**/.git
**/node_modules
**/.vs
**/*.user
**/.env
TestResults/
```

### Build, run
```bash
docker build -t shop-api:dev .
docker run --rm -p 8080:8080 -e ConnectionStrings__Default="..." shop-api:dev

# .NET 7+ — no Dockerfile
dotnet publish -c Release /t:PublishContainer
docker run --rm -p 8080:8080 shop-api:1.0.0
```

### docker-compose.yml
```yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      ConnectionStrings__Default: "Host=db;Username=app;Password=app;Database=shop"
      Redis__Connection: "redis:6379"
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
      POSTGRES_DB: shop
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d shop"]
      interval: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

### Healthcheck (in Dockerfile)
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD wget -q --spider http://localhost:8080/health || exit 1
```

### Chiseled (Ubuntu) base for size
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0-noble-chiseled AS runtime
WORKDIR /app
COPY --from=build /app .
USER $APP_UID            # built-in non-root
ENTRYPOINT ["dotnet", "Shop.Api.dll"]
```

### Alpine variant (musl)
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0-alpine AS runtime
RUN apk add --no-cache icu-libs   # if you need globalization
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false
```

### Self-contained / AOT image
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0-noble AS build
WORKDIR /src
COPY . .
RUN dotnet publish src/Shop.Api/Shop.Api.csproj -c Release -r linux-x64 \
    --self-contained -p:PublishAot=true -o /app

FROM mcr.microsoft.com/dotnet/runtime-deps:9.0-noble-chiseled
WORKDIR /app
COPY --from=build /app/Shop.Api ./Shop.Api
USER $APP_UID
ENTRYPOINT ["./Shop.Api"]
```

### MSBuild container properties
```xml
<PropertyGroup>
  <ContainerRepository>shop-api</ContainerRepository>
  <ContainerImageTag>1.2.3</ContainerImageTag>
  <ContainerBaseImage>mcr.microsoft.com/dotnet/aspnet:9.0-noble-chiseled</ContainerBaseImage>
  <ContainerUser>$(ContainerUser)</ContainerUser>
</PropertyGroup>
```

---

## Common Patterns

```dockerfile
# Pattern: BuildKit cache mount for restore
# syntax=docker/dockerfile:1.7
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY *.sln ./
COPY src/ src/
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore Shop.sln
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet publish src/Shop.Api/Shop.Api.csproj -c Release -o /app
```

```yaml
# Pattern: separate compose files for dev vs prod
# docker-compose.override.yml — applied automatically
services:
  api:
    build:
      target: build              # stop at SDK stage for hot-reload
    volumes:
      - ./src:/src
    command: dotnet watch run --project src/Shop.Api
```

```bash
# Pattern: container memory limits + GC tuning
docker run -m 512m \
  -e DOTNET_GCHeapHardLimit=400000000 \
  -e DOTNET_GCServer=1 \
  -e DOTNET_GCConcurrent=1 \
  shop-api:1.0
```

```yaml
# Pattern: secrets via env file, never baked in
services:
  api:
    env_file: .env.local       # in .gitignore
```

---

## Gotchas & Tips

- **Layer order matters for cache** — copy `csproj` files first, restore, then copy source. A code change shouldn't bust the restore layer.
- **Always `.dockerignore`** — without it, `node_modules/` and `bin/` blow up the build context and slow every build.
- **Don't run as root** — recent images include a non-root `app` user; chiseled enforces it. Bind to `8080+` (non-privileged port) accordingly.
- **`ASPNETCORE_URLS=http://+:8080`** is the container default. Don't try `https://` inside the container — terminate TLS at the ingress.
- **`mcr.microsoft.com/dotnet/aspnet`** ≠ `runtime`. ASP.NET image adds the framework reference. A console worker should use `runtime`.
- **Chiseled images** are small but tools-free — no shell, no `apt`. You can't `docker exec` and poke around. Great for prod.
- **Alpine + globalization** — base image is musl + minimal ICU. Either add `icu-libs` or set `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=true`.
- **GC respects cgroup memory limits** — but only if `<ServerGarbageCollection>true</ServerGarbageCollection>` and the container has a real limit (`-m`). Otherwise GC sees the host RAM.
- **Use multi-platform builds** for arm64 + amd64: `docker buildx build --platform linux/amd64,linux/arm64`.
- **Don't bake secrets** — use environment variables, BuildKit secrets (`--secret`), or your platform's secret store.
- **Healthcheck endpoints** must be cheap — orchestrators poll often. `/health/live` returns "up", `/health/ready` checks dependencies.
- **`dotnet publish /t:PublishContainer` is strict about credentials** — `docker login` first or set `ContainerRegistry`/`ContainerImageName` precisely.

---

## See Also

- [[18 - CI-CD and DevOps]]
- [[04 - Microservices]]
- [[16 - Native Interop and AOT]]
- [[17 - Configuration]]
