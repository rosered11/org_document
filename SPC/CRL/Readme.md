# Tasks
- Create API for clear file in blob storage and history log
    * remove file from blob storage
    * remove history log
- Job for clear file in blob storage and history log
    * create job for trigger synchonus (Hangfire)
- Implement API for upload file
    * <mark>upload file to blob storage (FileManagement)</mark>
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
    * <mark>upload file to blob storage(FileManagement)</mark>
    * update fileInfo to HistoryLogDb
- Implement API for call function process file
- Create service for interact with Blob Storage (FileManagement)
    * upload file to blob storage
    * fetch file from blob storage
    * initial new project (prepare structure project + authen)
- Function for gzip file (wait to discuss for FE or BE implement)