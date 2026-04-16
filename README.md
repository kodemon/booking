# Booking System

An event-driven, highly decoupled booking engine built with .NET/C#. 

The system leverages **Event Sourcing** and **CQRS** to ensure immutable audit trails and high availability, utilizing a choreographed saga pattern for cross-domain workflows.

## Dependencies

* Event Sourcing [WolverineFX](https://wolverinefx.net)

## System Architecture

### Core Modules (Domains)

Each module is self-contained, possessing its own command handlers, aggregates, and event streams.

* **Booking**: The primary domain. It encapsulates the following sub-domains:
    * **Catalog**: Manages bookable resources (rooms, equipment, services) and metadata.
    * **Reservation**: Orchestrates reservation lifecycles and state transitions.
    * **Scheduling**: Computates availability, time-slots, and conflict detection.
    * **Review**: Handles post-service ratings and user feedback.
* **Principal**: Manmanages identity, authentication, and fine-grained authorization (ABAC + RBAC) via **Cerbos**.
* **Payment**: Manages transaction processing, invoicing, and third-party gateway integration.
* **Notification**: Dispatches asynchronous alerts (Email/SMS/Push) triggered by system events.

### System Services

* **API Gateway**: The entry point for all external traffic. It validates HTTP requests, enforces authentication, and dispatches initial `Commands` to the domain modules.
* **Saga Orchestrator (Choreographer)**: A reactive layer that listens to domain-wide integration events. It is responsible for triggering subsequent commands in a multi-step workflow (e.g., reacting to `PaymentCompleted` by issuing `ConfirmBooking`).
* **Projection Service**: A dedicated service that consumes all system events to build highly optimized, denormalized read models for complex queries and real-time dashboards.

## Communication Pattern

The system follows an **Event-Driven Choreography** pattern:

1. **Action**: API to `Command` to **Domain Module**.
2. **Reaction**: **Domain Module** to `Domain Event` to **Saga/Projections**.
3. **Continuation**: **Saga** to `New Command` to **Next Domain Module**.

## Engineering Principles

### 1. Concurrency Control (Optimistic Locking & Retry)

To prevent "double-booking" in a highly decoupled environment, the system employs **Optimistic Concurrency Control (OCC)** via monotonic versioning.

* **The Mechanism**: Every event in a stream is assigned a strictly increasing `version` number. 
* **Conflict Detection**: When a command attempts to append a new entry, the system verifies that the `expected_version` matches the current state of the stream.
* **Retry Strategy**: If a collision occurs (i.e., the version has changed), the system automatically intercepts the failure and retries the command. The retry process re-fetches the latest stream state, applies the command to the new aggregate version, and attempts a fresh append.
* **Resolution**: In scenarios like "double booking," the second attempt will fail the business logic validation (e.g., `if (seat.is_taken) throw Conflict;`) because the "race-winning" entry has already updated the aggregate state, ensuring strict consistency even under high contention.

### 2. Cryptographic Integrity (Verifiable Ledger)

While primarily an event store, this system implements a **verifiable audit trail** using cryptographic chaining.
* **The Mechanism**: Each `Event` record entry contains a `hash` derived from its own payload and the `prev_hash` of the preceding entry in the stream.
* **Tamper Detection**: This creates a linked chain of events. Any unauthorized modification to historical data (e.g., changing a reservation price or date) invalidly breaks the hash chain, making the tampering immediately detectable during validation.
* **Trust**: This provides a high-assurance audit log suitable for regulated environments.

### 3. Distributed Observability (Traceability)

In an event-driven architecture, tracking a single user action across multiple decoupled modules is a significant challenge.
* **The Mechanism**: The system utilizes **Distributed Tracing** by injecting a `trace_id` into the `meta` payload of every event.
* **End-to-End Visibility**: This `trace_id` propagates from the initial API Gateway request, through the Command handlers, across the Saga orchestrator, and finally into the Projection services.
* **Impact**: This enables developers to reconstruct the entire lifecycle of a complex, multi-step workflow (e.g., Booking to Payment to Notification) within a single unified view, drastically reducing Mean Time to Resolution (MTTR) for distributed bugs.