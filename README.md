# MythAPI

A REST API, built with .NET 8 and Entity Framework Core, for managing mythological gods, their aliases, and the mythologies they belong to.

## Overview

MythAPI is a minimal-API (`dotnet new web`) application organized by domain rather than by technical layer. It persists data in PostgreSQL via Entity Framework Core, exposes a versioned HTTP API (`/api/v1`), and documents itself via Swagger/OpenAPI. Configuration secrets (database credentials, etc.) are retrieved from an Azure Key Vault at startup.

## Project structure

```
src/
├── Program.cs                     # Application entry point, DI and middleware wiring
├── Common/
│   └── Database/
│       ├── AppDbContext.cs        # EF Core DbContext and entity/table mapping
│       └── Models/                # Persistence models (God, Mythology, Alias)
├── Gods/
│   ├── Interfaces/                # IGodRepository
│   ├── DBRepositories/            # EF Core-backed implementation of IGodRepository
│   ├── Mocks/                     # In-memory implementation used for local/testing scenarios
│   └── Models/                    # DTOs/parameters used by the Gods domain (GodInput, GodParameter, ...)
├── Mythologies/
│   ├── Interfaces/                # IMythologyRepository
│   └── DBRepository/              # EF Core-backed implementation of IMythologyRepository
├── Endpoints/
│   └── v1/                        # Minimal API endpoint definitions (Gods.cs, Mythologies.cs)
├── Migrations/                    # EF Core migrations
└── Dockerfile, compose.yaml       # Container build/run definition

tests/
└── UnitTests/                     # xUnit test project covering endpoints and repositories

.bicep/                            # Bicep modules for provisioning Azure resources (Key Vault, database, secrets)
```

Each domain folder (`Gods`, `Mythologies`) contains its own interface, repository implementation, and models, which are registered with dependency injection in `Program.cs` and consumed by the corresponding endpoint definitions in `Endpoints/v1`.

## Requirements

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- A PostgreSQL instance (local, containerized, or Azure Database for PostgreSQL)
- An Azure Key Vault (or equivalent configuration override, see [Configuration](#configuration)) containing the application secrets
- Docker (optional, for containerized builds)

## Getting started

Clone the repository and restore dependencies:

```bash
dotnet restore
```

Build the solution:

```bash
dotnet build
```

Run the API from the `src` directory:

```bash
cd src
dotnet run
```

By default, ASP.NET Core serves the API on the ports configured in `src/Properties/launchSettings.json`. Once running, Swagger UI is available at `/swagger` for interactive exploration of the API.

## Configuration

The application reads the following configuration keys, which can be supplied via `appsettings.json`, environment variables, or Azure Key Vault:

| Key                  | Description                                              |
|----------------------|------------------------------------------------------------|
| `MYTH_KeyVaultName`  | Name of the Azure Key Vault to load secrets from           |
| `dbHost`             | PostgreSQL host name                                        |
| `dbName`             | PostgreSQL database name                                    |
| `adminUsername`      | PostgreSQL user name                                         |
| `adminPassword`      | PostgreSQL password                                           |

At startup, `Program.cs` builds a Key Vault URI from `MYTH_KeyVaultName` and authenticates using `DefaultAzureCredential`, which supports (in order of precedence) environment variables, managed identity, and locally cached `az login` credentials. The identity used must be granted the `Key Vault Secrets User` role on the target Key Vault.

For local/containerized runs, the following environment variables configure the credential used to authenticate to Azure:

```powershell
$Env:AZURE_TENANT_ID="<tenant id>"
$Env:AZURE_CLIENT_ID="<service principal client id>"
$Env:AZURE_CLIENT_SECRET="<service principal client secret>"
```

These are forwarded into the container by `src/compose.yaml`.

## Database

The database schema is managed with EF Core migrations, defined in `src/Migrations`.

To apply the latest migrations against the configured database:

```bash
dotnet tool install --global dotnet-ef   # once per machine
cd src
dotnet ef database update
```

To create a new migration after changing an entity model under `Common/Database/Models`:

```bash
dotnet ef migrations add <MigrationName>
```

### Entities

- **Mythology** — `Id`, `Name`, and a one-to-many collection of `Gods`.
- **God** — `Id`, `Name`, `Description`, `MythologyId` (foreign key), and a collection of `Aliases`.
- **Alias** — `Id`, `GodId` (foreign key), `Name`.

## API

All endpoints are versioned under `/api/v1`.

### Mythologies

| Method | Route                  | Description                 |
|--------|------------------------|------------------------------|
| GET    | `/api/v1/mythologies`  | Returns all mythologies.      |

### Gods

| Method | Route                             | Description                                                      |
|--------|------------------------------------|--------------------------------------------------------------------|
| GET    | `/api/v1/gods`                     | Returns all gods, including their aliases.                         |
| GET    | `/api/v1/gods/{id}`                | Returns a single god by id.                                          |
| GET    | `/api/v1/gods/search/{name}`       | Searches gods whose name matches `name`. Accepts an `includeAliases` query parameter (default `false`). |
| POST   | `/api/v1/gods`                     | Creates or updates one or more gods. Accepts a JSON array of `GodInput` objects (`id`, `name`, `description`, `mythologyId`); existing gods are matched by `id`. |

## Running with Docker

The repository includes a `Dockerfile` and `compose.yaml` to build and run the API in a container.

```bash
cd src
docker compose up --build
```

This builds the image from the multi-stage `Dockerfile` (SDK image for build/publish, ASP.NET runtime image for execution) and starts the `server` service, exposing port `8080`. Required environment variables (`MYTH_KeyVaultName`, `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`) must be set in the shell before running `docker compose up`, since `compose.yaml` forwards them into the container.

To stop the running container, press <kbd>Ctrl</kbd> + <kbd>C</kbd>, and to remove the created containers:

```bash
docker compose down
```

See [`README.Docker.md`](./README.Docker.md) for further Docker-specific notes.

## Infrastructure

Azure infrastructure (Key Vault, secrets, PostgreSQL database, and access policies) is defined as Bicep modules under `.bicep/Modules`.

## Testing

Unit tests live in `tests/UnitTests` and use xUnit. Run them with:

```bash
dotnet test
```

CI runs `dotnet restore`, `dotnet build`, and `dotnet test` on every pull request (see `.github/workflows/test.yml`).

## Continuous integration & deployment

- `.github/workflows/test.yml` — builds and runs the test suite on pull requests.
- `.github/workflows/docker-image.yml` — builds (and pushes) the Docker image.
- `.github/workflows/create-uat-branch.yml` — automation for creating a UAT branch.

## Project history

For a narrative, step-by-step account of how this project was originally built (scaffolding, package installation, Docker setup, etc.), see [`StepsTaken.md`](./StepsTaken.md).

## Roadmap

- Improve Bicep provisioning scripts
- Add Kubernetes manifests / image repository support
- Add API authentication & authorization
