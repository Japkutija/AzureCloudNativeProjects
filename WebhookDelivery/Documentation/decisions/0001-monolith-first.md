# ADR 0001: Webhook Delivery Service starts as a local monolith

- **Date:** 2026-05-03
- **Status:** Accepted
- **Author:** Jasmin Dervišević

## Context

The Webhook Delivery Service exists to drill the reliability patterns that every serious distributed delivery system converges on: at-least-once delivery, idempotency keys on the receiver contract, exponential backoff with jitter, circuit breaking against unhealthy subscribers, dead-letter handling with replay, and end-to-end correlation through asynchronous boundaries. These concerns are platform-independent — they have to be implemented correctly regardless of whether the underlying transport is RabbitMQ, Kafka, Azure Service Bus, AWS SQS, or an in-process channel.

Starting the implementation directly on Azure conflates two unrelated classes of concern:

1. **Pattern-level concerns**, listed above, which are conceptually well-defined and where mistakes are caught by tests written against the behaviour the patterns specify.
2. **Platform-level concerns** of Azure: SAS-versus-Managed-Identity authentication, Service Bus tier features such as sessions and transactions, Cosmos partition design, RBAC propagation delays, Bicep idioms, deployment scopes, regional availability of preview features.

When the two are tangled, three things go wrong. Pattern iteration slows to the speed of cloud deployments. Behaviour bugs become indistinguishable from permission and configuration bugs, which destroys the signal that tests are supposed to provide. And the platform begins to do work that the application code should be doing — for example, leaning on Service Bus's built-in retry policy without ever writing the application-level retry that would be required if the queue were ever swapped, leaving the design quietly coupled to a single vendor.

## Decision

Iteration 1 of the Webhook Delivery Service is a single .NET 9 process running locally. Its components and the contracts between them are designed so that the platform substitution in iteration 2 is a code change in one place per component, not a redesign.

The shape of iteration 1:

- **HTTP surface.** ASP.NET Core Minimal API exposing `POST /events` for publication and `POST /subscribers` for subscriber registration. Authentication is deferred to iteration 2; the surface is bound to localhost.
- **Queue.** `System.Threading.Channels.Channel<DeliveryAttempt>` behind an `IDeliveryQueue` interface whose contract assumes the weakest guarantees Azure Service Bus offers — at-least-once, no global ordering without explicit sessions, transient failures must be retried by the caller. The in-memory channel is wrapped in a fault-injection decorator for tests, which randomly drops, delays, and duplicates messages.
- **Consumer.** A `BackgroundService` reading from `IDeliveryQueue` and dispatching through `IDeliveryClient` (HttpClient-backed). The consumer holds no state beyond the message currently in flight; all durable state lives in the store.
- **Persistence.** SQLite via EF Core, behind repository interfaces (`ISubscriberStore`, `IDeliveryAttemptStore`, `IDeadLetterStore`). The schema is designed against the constraints Cosmos will impose later: partition-friendly access patterns, no cross-entity transactions, write-heavy paths kept narrow.
- **Reliability policies.** Polly resilience pipelines for retry (exponential backoff with jitter), per-attempt timeout, and per-subscriber circuit breaker. Policy parameters are read from `IOptions<DeliveryOptions>`; no inline magic numbers.
- **Idempotency.** Each delivery attempt carries a stable, message-derived `X-Webhook-Event-Id` header from day one. The semantics are documented for receivers even though the in-memory channel does not strictly require them, because the contract has to be honest about what iteration 2 will exhibit.
- **Observability.** Structured logging with Serilog. `LogContext` carries `EventId`, `SubscriberId`, and `AttemptNumber` through every delivery path. OpenTelemetry instrumentation is held back to iteration 2 to keep iteration 1 focused; the logging structure is shaped so that distributed tracing slots in without a rewrite.
- **Configuration.** Strongly typed via `IOptions<T>`, validated at startup with `ValidateDataAnnotations` and `ValidateOnStart`. No secrets, no environment-specific endpoints.

No Azure resource is provisioned for iteration 1. No `appsettings.{env}.json` references any cloud endpoint.

## Alternatives considered

**Start directly on Azure with a thin scaffold.** Rejected for the reasons in the Context section: the cost of every iteration becomes a deployment, and the signal from tests is degraded by platform noise.

**Use a local Docker-hosted Service Bus emulator or RabbitMQ.** Rejected for iteration 1. A real broker is closer to production behaviour but introduces a runtime dependency and an operational surface that is itself a source of friction unrelated to the patterns being drilled. Worth revisiting between iteration 1 and iteration 2 if the in-memory abstraction proves too forgiving.

**Skip the abstractions and write directly against `Channel<T>` and EF Core, refactoring later.** Rejected because the cost of writing the interfaces up front is small and the cost of retrofitting them under a working codebase is large. The interfaces are also the most informative artefacts of iteration 1 — they encode what the design assumes about its dependencies.

## Consequences

**What this buys.** Pattern bugs surface in seconds against unit and integration tests, not after a Bicep redeploy. The interfaces force the design's seams to be confronted now, which is exactly the surface iteration 2 will exploit. Low-value experiments — different retry curves, alternative idempotency-key derivations, dead-letter triage flows — are tractable because they cost nothing to run.

**What it risks, and how it is contained.**

The most dangerous risk is encoding assumptions that do not hold in the cloud: that the queue is always available, that delivery to the consumer is in-order or exactly-once, that the persistence store is local and immediately consistent. The mitigation is to write `IDeliveryQueue` and `IDeliveryAttemptStore` against the weakest contract their iteration-2 implementations will offer, and to enforce those weakest contracts in tests via the fault-injection decorator and a deliberately concurrent test harness.

A secondary risk is that platform-level concerns (identity, network egress, secret rotation, deployment) silently disappear from view because they have no equivalent locally. The mitigation is that they are listed explicitly in the iteration 2 plan, each one tied to a future ADR, rather than left implicit.

A third risk is that iteration 1, once working, becomes a sunk cost that pulls toward shipping the local shape as the final shape. The mitigation is that iteration 2 is scheduled work in this same repository, not a separate project, and that the substitutions in iteration 2 are designed to be incremental rather than a rewrite.

## Iteration 2 plan

The migration to Azure proceeds one substitution at a time. Each substitution lands as its own ADR documenting the platform-level decisions that the swap surfaces — none of these decisions are pre-empted here.

1. `Channel<T>` → Azure Service Bus, queue first; sessions only if message ordering becomes a hard requirement justified in its ADR.
2. SQLite → Cosmos DB, with the partition key chosen and justified in a dedicated ADR against the access patterns from iteration 1.
3. `BackgroundService` → Azure Function with Service Bus trigger. The `IDeliveryClient` and Polly stack remain untouched; this is the test of whether the seam was drawn correctly.
4. Serilog console sink → Serilog with OpenTelemetry exporter to Azure Monitor; verify end-to-end correlation across HTTP → Service Bus → Function → outbound HTTP.
5. Local configuration → Key Vault references; application identity → Managed Identity; no connection strings or shared keys in source or in pipeline secrets.
6. Bicep templates for the full topology, GitHub Actions for CI and gated CD into staging and production environments, with environment-specific approvals.

Each step is independently reversible. By construction, none of them require revisiting the application's pattern-level code.