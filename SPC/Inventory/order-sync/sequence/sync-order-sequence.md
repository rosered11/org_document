# Sequence diagram adjustment

### Sync adjustment data from WMS to SPC

```mermaid
sequenceDiagram
    participant Cronjob(Adjust)
    participant DatabaseSPC(SyncProgress)
    participant DatabaseWMS(Adjust)
    critical Fetch SyncProcess from SPC
        Cronjob(Adjust)->>DatabaseSPC(SyncProgress): Fetch lastId from SyncName('WMSStockAdjustmentSync')
        DatabaseSPC(SyncProgress)-->>Cronjob(Adjust): receive SyncProcess
        alt SyncProcess is null
            Cronjob(Adjust)->>Cronjob(Adjust): SyncProcess = create inital()
            Cronjob(Adjust)->>DatabaseSPC(SyncProgress): Insert SyncProcess
        end
    option Exception
        Cronjob(Adjust)->>Cronjob(Adjust): Send notify to MS Team(Fetch SyncProcess is failed)
    end
    critical Open transaction database
        loop Batch all data WMS follow condition 
            critical Fetch data from WMS Batch only
                Cronjob(Adjust)->>DatabaseWMS(Adjust): Fetch data by condition SyncProcess.LastId
                DatabaseWMS(Adjust)-->>Cronjob(Adjust): WMSBatchData = Receive data from WMS by condition
            option Database Exception
                Cronjob(Adjust)->>Cronjob(Adjust): Send notify to MS Team(Fetch data from WMS is failed)
            end
            Cronjob(Adjust)->>Cronjob(Adjust): BulkInsert(WMSBatchData)
        end
        Cronjob(Adjust)->>Cronjob(Adjust): Database commit
    option Exception
        Cronjob(Adjust)->>Cronjob(Adjust): Database rollback
    end
```

### TODO: [Not Done] Compute data stockadjustment to stock in SPC

```mermaid
sequenceDiagram
    participant Cronjob(Adjust)
    participant DatabaseSPC(SyncProgress)
    participant DatabaseWMS(Adjust)
    critical Fetch data stockadjustment from SPC condition is_process = false
        loop stockBatch data stockadjustment
            Cronjob(Adjust)->>DatabaseSPC(SyncProgress): Fetch data from stockadjument per batch
            DatabaseSPC(SyncProgress)-->>Cronjob(Adjust): Receive data stockajuetment per batch
            alt stockBatch don't have in StockMemory
                Cronjob(Adjust)->>Cronjob(Adjust): Keep stock per key in StockMemory
                Cronjob(Adjust)->>Cronjob(Adjust): Insert/Update stock follow logic
            end
        end
        Cronjob(Adjust)->>Cronjob(Adjust): Database commit
    option Exception
        Cronjob(Adjust)->>Cronjob(Adjust): Send notify to MS Team(Fetch SyncProcess is failed)
    end
```

### Very Shot Term

### Need TODO
- ทำเรื่องขอขึ้น service ใหม่
- สอบถาม ว่าปัจจุบันเราเคลีย stockadjustment ของ SPC ยังไง เพราะอาจจะต้องคิดเผื่อตอน ส่งไปเคลียที่ฝั่ง consumer ด้วย -> รอทำ phase ถัดไป

**Target Database new table**
- Inventory
    - stock main : spc_crl_main
    - stock adjust : spc_crl_adjustment
- Data mart
    - Front end-connect inteface : spc_temp_stock_crl

*Detail*

    sku(FC) = product_id(SPC)
    product_id(FC) = barocde(SPC)
    sku = ไม่ส่ง
    product_id = barcode

**Concern:** 
- maybe impact if it must to purge data for sync every 15 mins