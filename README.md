# ServiX

Multi-tenant SaaS business operations platform (ERP/Ops) for small and medium businesses.

> Status: early development (Phase 1 — foundation).

## Tech stack

- ASP.NET Core Web API (.NET 10) + Entity Framework Core
- PostgreSQL (Row-Level Security for tenant isolation)
- Redis
- xUnit + Testcontainers

## Project structure

```
backend/
  ServiX.sln
  src/
    ServiX.Api/             # HTTP layer: controllers, middleware, DI
    ServiX.Application/     # use cases, DTOs, interfaces
    ServiX.Domain/          # entities, enums, domain events
    ServiX.Infrastructure/  # EF Core, Redis, auth
  tests/
    ServiX.UnitTests/
    ServiX.IntegrationTests/
```

## Getting started

Requirements: .NET SDK 10.

```powershell
dotnet build backend/ServiX.sln
dotnet test backend/ServiX.sln
dotnet run --project backend/src/ServiX.Api
```

## License

MIT — see [LICENSE](LICENSE).
