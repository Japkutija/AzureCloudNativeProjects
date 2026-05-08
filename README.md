# Azure Cloud-Native Journey

Deliberate practice on the reliability and architecture patterns that hold up real distributed systems, in .NET 9 and Azure.

## Scope

The repository is a working record of focused engineering work on the concerns that recur in every serious distributed delivery system: at-least-once semantics, idempotency, retry with backoff and jitter, circuit breaking, dead-letter handling, partition-aware persistence, identity-based authentication, distributed tracing, and infrastructure-as-code.

The work is structured around drilling these patterns under realistic constraints, not surveying them. The repo currently contains a single project that evolves through iterations; nothing here is a tutorial.

## Layout

- `WebhookDelivery/` contains the .NET 9 webhook delivery service.
- `WebhookDelivery/Documentation/decisions/` holds Architecture Decision Records for the project.
- `WebhookDelivery/Notes/` collects concise, first-hand notes on Azure services and .NET libraries after they have been used in the project. Failure modes and "when not to use this" sections are mandatory; copy-paste from official docs is not.
- `WebhookDelivery/Journal/` is the dated working log. Short entries, written daily during active work, capturing what was built, what broke, and what was understood as a result.
- `WebhookDelivery/Infrastructure/` holds Bicep templates for the cloud-deployed iterations.

## Current work

`WebhookDelivery/` is a webhook delivery service. Iteration 1 is a deliberately monolithic .NET 9 implementation that isolates messaging and reliability patterns from cloud-platform concerns; the rationale is in `WebhookDelivery/Documentation/decisions/0001-monolith-first.md`. Iteration 2 swaps infrastructure components onto Azure one at a time — Service Bus, Cosmos DB, Azure Functions, Managed Identity, OpenTelemetry to Azure Monitor — with Bicep and GitHub Actions handling the deployment surface. Each substitution lands as its own ADR.

## Certification track, in parallel

AZ-900 by end of June 2026. AZ-204 by end of November 2026. The exam objectives are exercised in the active project or the Azure sandbox rather than memorised; topics that do not surface naturally are drilled in short, isolated sandbox experiments and written up in `WebhookDelivery/Notes/`.

## Conventions applied across the repository

Patterns are implemented before primitives. A cloud service is introduced only after the pattern it implements has been understood and exercised without it. Every non-trivial decision is recorded as an ADR before the code is written, not after, with explicit alternatives considered and rejected. Every Azure service used is summarised in `WebhookDelivery/Notes/` after first-hand use, including failure modes. Production defaults are applied from the first commit where they cost nothing: structured logging, health checks, cancellation tokens flowing through, configuration via `IOptions<T>`, no secrets in source, no `Task.Result` or `.Wait()` in async paths.

## Status

See `WebhookDelivery/Journal/` for the latest entries. Issues and discussions in this repository are open.
