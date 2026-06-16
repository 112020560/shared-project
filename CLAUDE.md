# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# Build the solution
dotnet build SharedProjects.sln

# Build a specific project
dotnet build Src/SharedKernel/SharedKernel.csproj
dotnet build Src/SmartCore.Telemetry/SmartCore.Telemetry.csproj

# Pack NuGet packages (both projects have GeneratePackageOnBuild=true, so build does this automatically)
dotnet pack Src/SharedKernel/SharedKernel.csproj -c Release
dotnet pack Src/SmartCore.Telemetry/SmartCore.Telemetry.csproj -c Release
```

## Architecture Overview

This solution contains two NuGet packages targeting **net9.0**, designed to be shared across microservices in the SmartCore platform.

### SharedKernel (`SmartCore.SharedKernel`)

Domain-driven design primitives:

- **`Result` / `Result<TValue>`** — Railway-oriented programming pattern. Use `Result.Success()` / `Result.Failure(error)` instead of throwing exceptions. `Result<TValue>` has an implicit conversion from `TValue` (returns `NullValue` error if null).
- **`Error`** — Immutable record with `Code`, `Description`, and `ErrorType`. Factory methods: `Error.Failure()`, `Error.NotFound()`, `Error.Problem()`, `Error.Conflict()`.
- **`ValidationError`** — Extends `Error` with `ErrorType.Validation`; aggregates multiple `Error[]` from failed results via `ValidationError.FromResults()`.
- **`AggregateRoot`** — Base class for DDD aggregates; holds a private `List<IDomainEvent>` and exposes `RaiseDomainEvent()` / `ClearDomainEvents()`.
- **`IDomainEvent`** — Marker interface for domain events.
- **`ApiResponse`** — HTTP response wrapper with `IsSuccess`, `StatusCode`, `Message`, `Data`, `TotalRecords`, and `TraceId`. Use static factory methods (`ApiResponse.Success()`, `ApiResponse.Failure()`, `ApiResponse.NotFound()`, etc.).

**Contracts** (message bus integration types under `SharedKernel.Contracts`):
- `Crm/Customers/` — `CustomerCreated`, `CustomerUpdated`
- `Payments/` — `ProcessPaymentCommand`, `ProcessRevolvingPaymentCommand`, `PaymentProcessed`, `RevolvingPaymentProcessed`, `PaymentFailed`, `PaymentRejected`
- `Catalogs/` — Product, Category, and Brand event interfaces
- `Inventory/` — Stock movement, reservation, and physical inventory event interfaces
- `ElectronicInvoice/` — Costa Rican e-invoice contracts (`FacturaElectronicaContract`, `NotaCreditoElectronicaContract`, `ResultadoFacturaElectronica`, `ElectronicDocumentProcessedEvent`)
- `Sales/` — Sale quote and invoice events

### SmartCore.Telemetry (`SmartCore.Telemetry`)

OpenTelemetry setup for .NET 9 microservices. Register via:

```csharp
services.AddSmartCoreTelemetry(options =>
{
    options.ServiceName = "MyService";
    options.Version = "1.0.0";
    options.Environment = "production";
    options.OtlpEndpoint = "http://collector:4317"; // defaults to localhost:4317
    options.EnableRedis = true;       // optional
    options.SamplerRatio = 0.5;       // 0.0–1.0, defaults to 1.0 (100%)
});
```

Instruments: ASP.NET Core, HttpClient, EF Core, MassTransit (traces + metrics), Redis (optional), and .NET runtime metrics. Exports via OTLP/gRPC.
