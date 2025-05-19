# Inventory

## Description

Step:

- New orders → store and track normally.

- Existing orders:

    + If quantity changes → track delta (increase or decrease).
    + Maintain a history of adjustments with timestamps or versions.
    + High performance is the top priority.

##  Proposed Architecture Overview

✅ Key Principles:
Event sourcing for change tracking.

Command Query Responsibility Segregation (CQRS) for performance and separation.

Use in-memory or cache layers to accelerate reads.

## Component

1. Write Layer (Command Side)
API Gateway → Receives order create/update requests.

    Order Service:

    Determines if the order is new or existing.

    If existing and qty has changed, calculate delta.

    Emit OrderQuantityAdjusted event if quantity changes.

    Persist changes via the Event Store and Write DB.

2. Read Layer (Query Side)
Read-optimized DB (e.g., PostgreSQL read replica, Redis, or Elasticsearch).

    Keeps materialized views of current order quantities and history.

    Updated asynchronously via event consumers.

3. Event Store / Message Bus
Kafka, RabbitMQ, or Redis Streams.

    Stores all domain events like OrderCreated, OrderQuantityAdjusted.

    Allows rebuilding the current state or history anytime.

## Performance Optimizations
| Area | Optimization |
| - | - |
| Write | Use asynchronous processing for updates(non-blocking). |
| Read | Cache current order state in Redis for instant access. |
| Events | Use compact, high-throughput event formats (e.g., Protobuf). |
| Storage | Partition large tables by date or customer ID. |
| Indexing | Index OrderID, UpdatedAt, and Qty in both write/read DBs. |
| Scalability | Scale with microservices (e.g., Order Service, History Service).

## Workflow

### Sample Workflow

New Order

1. API receives CreateOrder(OrderID: X, Qty: 100)

2. Order Service stores in Orders table.

3. Emit OrderCreated event.

Update Existing Order

1. API receives UpdateOrder(OrderID: X, Qty: 120)

2. Load current order → Qty = 100.

3. Calculate delta = +20.

4. Update Orders table.

5. Emit OrderQuantityAdjusted with delta = +20.

6. Event listener updates read DB / cache and logs history.

##  Tech Stack (Suggested)
| Layer | Tools |
| - | - |
| API Layer | FastAPI (Python), Gin (Go), ASP.NET Core |
| Event Bus | Kafka, NATS, RabbitMQ |
| DB | PostgreSQL, Redis (cache), Elasticsearch (history/search) |
| Message Storage | Kafka (event source), or PostgreSQL event table |
| Deployment | Kubernetes / Docker (optional) |


## Event Sourcing vs Outbox Pattern
| Concept | Description | Primary Purpose |
|-|-|-|
| Event Sourcing | Stores the state of an entity as a series of events (e.g., OrderCreated, OrderQtyAdjusted) instead of just storing the current state. | Source of truth and ability to rebuild state or audit. |
| Outbox Pattern | Stores events/messages in a DB table as part of the same transaction with business data changes. A message relay later reads from this table and publishes to the message bus. | Ensures reliable event delivery in distributed systems (avoids dual-write issues).

## Can They Use the Same Payload?
Yes, they can share the same event structure, especially if you:

- Store events in an Outbox table for reliable messaging, AND

- Use the same Outbox table as the Event Store, if you're also doing event sourcing.

But that depends on whether you're using event sourcing as your source of truth.

## When to Combine Them
You can combine event sourcing + outbox in these scenarios:

You control all writes via your own application/service (no legacy DB updates).

You need both auditability and reliable event dispatch.

You're okay with replaying events to rebuild state if needed.

## When NOT to Combine
- If you do not use event sourcing for your business logic (i.e., use CRUD + outbox), then keep the outbox decoupled.

- If events for sourcing and integration differ in format or semantics.

## Best Practice
If combining:

- Design an Events table with proper metadata, such as published, retry_count, etc.

- Use transactional outbox (i.e., insert event in same DB transaction as order update).

- Have a reliable dispatcher (poller or Debezium CDC) to forward events to the bus.

## Goals of Schema
Store full history of changes (e.g., quantity changes).

Support rebuilding state from events.

Enable querying for order change history (timeline).

Be extendable to support more event types later (e.g., address change).

## 🚀 Advantages of This Design
✅ Fully auditable change log.

✅ JSON payload gives flexibility per event type.

✅ Can be used with the Outbox pattern directly.

✅ Enables future support for soft delete, replay, snapshots.

## Bonus: Read Model Table (Optional)
To serve dashboards or APIs faster:

```sql
CREATE TABLE OrderSnapshots (
    OrderID UUID PRIMARY KEY,
    CurrentQty INT,
    LastUpdated TIMESTAMPTZ
);
```
Updated asynchronously from events.

Provides fast lookup without event replay.

## 🎯 Purpose of OrderSnapshots
OrderSnapshots is a denormalized, read-optimized table that reflects the latest state of an order. It allows you to:

- Avoid replaying all events during every read.

- Serve fast queries (e.g., “What is the current quantity of order X?”).

- Integrate with caching layers like Redis for even faster reads.

### Table Schema Design

```sql
CREATE TABLE OrderSnapshots (
    OrderID UUID PRIMARY KEY,
    ProductID UUID,
    CustomerID UUID,
    CurrentQty INT NOT NULL,
    TotalAdjustments INT DEFAULT 0,
    LastAdjustmentDelta INT,
    LastEventID UUID,
    LastUpdated TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 🔍 Fields Explained
| Field | Purpose |
|-|-|
| OrderID | Uniquely identifies the order. |
| ProductID, CustomerID | Optional, useful for filters/search. |
| CurrentQty | The current quantity — updated via event projection. |
| TotalAdjustments | Count of all qty changes for fast stats. |
| LastAdjustmentDelta | The latest delta (+10, -5, etc). |
| LastEventID | Last event applied (idempotency & traceability). |
| LastUpdated | Timestamp of last qty adjustment. |

## ⚙️ How It Gets Updated
Option 1: Synchronous Projection
When an OrderQuantityAdjusted event is stored, also immediately:

Apply the delta to CurrentQty

Update snapshot table in the same DB transaction or a safe follow-up

✅ Faster but tightly coupled.