# Archetecture

## Event Sourcing

### Event Store
```mermaid
graph LR
    Client(Client) -- 1.create order --> Order(Order Service)
    Order -- 2.create CreateOrderEvent --> Event(Event Store)
    Client -- 3.update qty --> Order
    Order -- 4.create UpdateOrderEvent --> Event
    Cron(Cronjob) -- 5.build event to snapshot --> Order
    Order -- 6.query event isn't sync<br>and adjust data snapshot --> Event
    Client -- 7.query data snapshot --> Order
```

### Cronjob
```mermaid
graph LR

    Cron(Cronjob) -- 1.build event to snapshot --> Order
    Order(Order Service) -- 2.query event isn't sync --> Event(Event Store)
    Event -- 3.receive and build adjust snapshot --> Order
```

### View
```mermaid
graph LR

    Client(Client) -- 1.query order --> Order(Order Query Service)
    Order -- 2.return data view from snapshot --> Client
```

### Intro

- เราจะต้อง list ทางเข้าทั้งหมดที่ทำให้เกิดการเปลี่ยนแปลงข้อมูล
- เราต้อง design event ให้ cover ในจุดต่างๆ ที่เราสนใจ
- เราต้องทำ function สร้าง event ในจุดเหล่านั้น
- เราจะทำ api สำหรับ ดึงข้อมูลจาก table snapshot
- เราจะทำ api สำหรับ ดึงข้อมูล history ด้วย identity จาก table event

### API Recommend

- /fetch/fullhistory/{order-id} -- query from event, snapshot and order-detail
- /fetch/order-list (need-filter-by-default) -- query from snapshot

### Component

- **Event Store:** ที่เก็บ event สำหรับการทำ snapshot ของ inventory

```mermaid
erDiagram
    InventoryEvent {
        long Id PK
        string(50) AggregateId "Identity สำหรับอ้างอิงกับ transaction เช่น Order-1234, Product-1234"
        string(50) AggregateType "Domain object หรือประเภทของข้อมูล เช่น Order, Product"
        string(50) EventType "Event ที่เกิดขึ้น เช่น ProductCreated, OrderCreated"
        string(max) EventPayload "Data payload ของ event"
        boolean IsSnapshotSynced "flag สำหรับบอกว่าทำการ sync ข้อมูลหรือยัง"
        dateTime2 CreatedAt "เวลาที่สร้าง"
    }
```
- **Inventory Product:** ใช้ในการเก็บ current data ของ product (snapshot) โดย data ใน table นี้ต้องเกิดจากการ build event ใน event store เท่านั้น
```mermaid
erDiagram
    InventoryProduct {
        long Id PK
        string(50) ProductId "Identity ของ product เช่น Product-1234"
        string(50) ProductName "ชื่อ product"
        int Quantity "จำนวนสินค้า"
        boolean IsDeleted "ใช้ flag สำหรับ product ที่ไม่ต้องการแล้ว จะถูกใช้กรณีมี case Event: ProductDeleted"
        string(max) EventPayload "Data payload ของ event"
        dateTime2 UpdatedAt "เวลาที่ update"
    }
```
