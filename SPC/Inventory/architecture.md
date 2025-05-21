# Archetecture

งาน:
- ปรับ performance cronjob adjust อาจจะไป check full เผือด้วย
- ทำ table เพิ่มสำหรับ sync data จาก stock main

## Overview

### Data flow cycle
```mermaid
graph LR
FCSnapFull(FC<br>Table: Snap Full Sync<br>re-initial product start <br>00:00 - 02:00) -- cronjob trigger at 04:00 --> CronFull(Cronjob Full)
CronFull --> SPCStock(Table: Stock Main)
FCSnapAdjust(FC<br>Table: Snap Adjust Sync<br>adjust product start <br>02:00 - 03:59) -- cronjob trigger at 05:00 - 23:00 every 15 mins --> CronAdjust(Cronjob Adjust)
CronAdjust --> SPCAdjust(Table: Adjustment Stock)
SPCAdjust --> SPCStock

FCSnapAdjust -- internal adjust FC --> FCSnapFull

classDef client fill:#f96,stroke:#333,stroke-width:4px;
class CronFull,CronAdjust client
```

### Solutions

```mermaid
graph LR
StockSvc(Stock Service) --> ControllerMain(API Service Main)
ControllerMain -- 1.re-init data to main stock --> StockMainDb(Table: Stock Main)
ControllerMain -- 2.create/publish event for product update --> Kafka(Kafka)
StockSvc(Stock Service) --> ControllerAdjust(API Service Adjust)
ControllerAdjust -- 1.adjust data to adjust stock --> StockAdjustDb(Table: Stock Adjustment)
ControllerAdjust -- 2.build event in adjust stock to adjust data in Stock Main --> StockMainDb
ControllerAdjust -- 3.create/publish event for product update --> Kafka

CronFull(Cronjob full<br>trigger 04:00) -- call api service main--> StockSvc
CronAdjust(Cronjob Adjust<br>trigger 05:00 - 23:00) -- call api service adjust --> StockSvc
Kafka -- consume --> Worker(Work consume Snapshot)
Worker -- adjust data --> DatabaseSnap(Database Warehouse Snapshot)
```

**Detail**
- Cronjob Full: ทำหน้าที่ในการ re-initial data ใหม่ มาแทนที่ โดยจะทำงานช่วงเวลา 04:00 ของทุกวัน เนื่องจากทางฝั่ง FC เขาจะมีการ re-adjust เพื่อให้ data up to date ซึงฝั่ง FC เขาจะ re-cale data ในแต่ละวัน เสร็จก่อนตี 4 เสมอ
- Cronjob Adjust: ทำหน้าที่ในการ ดึงข้อมูล adjust จาก table adjust ของ FC มา update ที่ฝั่ง SPC และ re-build adjust data to Stock Main ของ SPC โดยเงื่อนไขการทำงานหลักๆ มีดังนี้
    
    * ดึงข้อมูลตาม flag version ที่ใหม่กว่า ที่กำหนดไว้ มาเก็บไว้ที่ Stock Adjust ของ FC
    * build adjust data ที่เข้ามาใหม่ทั้งหมดใน table Stock Adjust ของ SPC เพื่อ update/insert ไปที่ Stock Main ของ SPC แต่มีเงื่อนไข adjust data ตามนี้
        
        - event ที่ได้รับ ถ้ามีการส่ง qty มาแต่ สถานะ เป็นค่าติดลบ และเมื่อมีการหักลบออกจาก Product ใน stock main แล้วได้ค่าติดลบ จะ Ignore event นั้น
        - event ที่ได้รับ มีการส่ง qty มา และสถานะเป็นค่าบวก แต่ไม่มี product ใน stock main ให้ทำการสร้าง record ใหม่ใน stock main ได้เลย (หมายเหตุ ข้อมูลใน event ต้องเพียงพอต่อการสร้าง order)
        - event ที่ได้รับนอกเหนือจากนั้น ให้ +/- qty ปกติ

### Background
```mermaid
graph LR
subgraph SPC
    SPCAdjust(Adjustment Stock) --> SPCStock(Stock Main) 
    SPCAdjust <--> Cron((Cronjob))
end
subgraph FC
    Cron <--> FCSnapFull(Snap Full Sync)
    Cron <--> FCSnapAdjust(Snap Adjust Sync)
    StockFC(Stock Main) --> FCSnapFull
    StockFC --> FCSnapAdjust
    StockFC <-- adjust --> InboundFC(Inbound)
    StockFC <-- adjust --> OutboundFC(Outbound)
end
```

**Ubiquitous Language**
- Inbound: สั่งซื้อสินค้า, สินค้านำเข้า เป็นสินค้าที่สั่งซื้อไว้เพื่อที่จะเก็บไว้ใน stock สินค้า
- Outbound: สินค้าที่ถูกขายออก คือสินค้าที่ถูกขายออกไป เมื่อถูกขายจะนำออกจาก stock สินค้า
- Stock Main: เป็น core management stock สินค้า ว่ามีเพิ่มเข้ามา หรือลดลง
- Cronjob: เป็น cronjob ที่ทำหน้าที่ในการ sync data ระหว่าง FC กับ SPC

## Event Sourcing

### Event Store
```mermaid
graph LR
    Cronjob(Cronjob<br>Starter) <-- trigger every 04:00 --> FCSnapFull(Snap Full Sync)
    Cronjob <-- unknow --> FCSnapAdjust(Snap Adjust Sync)
    Cronjob --> SPCAdjust(Adjustment Stock)
    SPCAdjust --> SPCStock(Stock Main) 
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
