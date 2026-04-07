# Advanced CQRS — Cosmos DB + Azure Service Bus

This folder contains a step-by-step guide to extending the Clean Architecture foundation (modules 01–10) with a production-grade CQRS implementation using:

- **Azure Service Bus** as the message broker (write-side event publishing)
- **Azure Cosmos DB** as the read-side optimised database (query store)
- **MediatR 13.x** for in-process CQRS dispatch
- **.NET 10** throughout

## Modules in This Folder

| File | Topic |
|------|-------|
| [11-versions-and-setup.md](./11-versions-and-setup.md) | Compatible versions, NuGet packages, local emulator setup |
| [12-cqrs-cosmos-read-store.md](./12-cqrs-cosmos-read-store.md) | Cosmos DB as CQRS read store — design + code |
| [13-service-bus-events.md](./13-service-bus-events.md) | Publishing domain events via Azure Service Bus |
| [14-end-to-end-flow.md](./14-end-to-end-flow.md) | Full end-to-end request flow with flowcharts |

## Prerequisites

- Completed modules 01–10 in this repository
- Docker Desktop installed (for local emulators)
- .NET 10 SDK installed
- Visual Studio 2022 / VS 2026 or Rider

## Domain Used in Examples

All examples use an **Order Management** domain:
- `CreateOrderCommand` — write side (SQL Server via EF Core)
- `OrderCreatedEvent` — published to Service Bus
- `OrderReadModel` — projected into Cosmos DB
- `GetOrderByIdQuery` / `GetAllOrdersQuery` — read from Cosmos DB
