```mermaid
sequenceDiagram
    participant Client
    participant Order
    Client->>Order: Update order status
    critical Establish a transaction to database
        Order->Order: Process order status (this is old logic + refactor to transaction)
        Order->Order: sagaId = GenerateSagaId()
        Order->Order: db.SagaOrder.insert(sagaId, sagaStatus: "STARTED")
        Order->Order: outbox = PrepareEventPayload(order.SourceOrderId)
        Order->Order: db.Outbox.insert(outbox, outboxStatus: "STARTED")
        Order->Order: db.Commit()
    option Exception
        Order->Order: db.Rollback()
        Order->Order: logger.LogError(exception)
    end
```