# Note

- ไม่ใช้ hangfire เนื่องจาก UI hangfire ไม่มี filter สำหรับใช้ค้นหาเพื่อ manual trigger
- implement api for re-trigger

# Tasks
- Create API for clear file in blob storage and history log
    * remove file from blob storage
    * remove history log
- Job for clear file in blob storage and history log
    * create job for trigger synchonus (Hangfire)
- Implement API for upload file
    * <mark>upload file to blob storage (service interact)</mark>
    * insert HistoryLogDb
- Implement function process file
    * fetch data from blob storage
    * validate data (require unit test)
        - check require field
        - check max length
        - check duplicate in system
        - etc.
    * business process (UNKNOW รอคุยคุยกับ team)
    * generate file failure report to CSV
    * <mark>upload file to blob storage(service interact)</mark>
    * update fileInfo to HistoryLogDb
- Create service for interact with Blob Storage
    * upload file to blob storage
    * fetch file from blob storage