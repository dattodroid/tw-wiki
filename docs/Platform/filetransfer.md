# File Transfer

## External Resources

* PTC Help Center - [File Transfers](https://support.ptc.com/help/thingworx/platform/r9.7/en/index.html#page/ThingWorx/Help/FileTransfer/FileTransfers.html){:target="\_blank"}
* PTC Help Center - [Edge-Controlled File Transfers](https://support.ptc.com/help/thingworx/platform/r9.7/en/index.html#page/ThingWorx/Help/FileTransfer/EdgeControlledFileTransfers.html){:target="\_blank"}
* PTC Help Center - [Always-on File Transfers](https://support.ptc.com/help/thingworx/platform/r9.7/en/index.html#page/ThingWorx/Help/FileTransfer/AlwaysonFileTransfers.html){:target="\_blank"}

## Upload File from Edge flow

!!! warning
    Created by reverse engineering logs. Not a source of trust.

### Edge-Controlled (Axeda devices)

The file transfers are managed by the edge device (typically Axeda devices via eMessage connector).

All Edge-controlled file transfer activities are initiated by the edge and executed by WSExecutionProcessor threads on the platform.

#### 9.5 with bind egress (happy path)

``` mermaid
sequenceDiagram
    box Edge
    participant Axeda as Axeda device
    end
    box Connector
    participant EMC as eMessage
    end
    box Platform
    participant Thing as AxedaThing
    participant FTSS as FileTransferSubsystem
    participant repo as Repository
    participant ftjob as TransferJob
    participant resa as Reservation
    end
    
    Axeda->>+EMC: <Ur> Upload Request
        EMC->>+FTSS: Copy() async / queuable
            FTSS--)ftjob: enqueue ft job
            Note right of ftjob: <  Max File Transfers Allowed in Offline Queue  <br/> <  Max File Transfers Allowed in Offline Queue Per Thing
            Note over ftjob: PENDING
            Note over ftjob: Offline queue
        FTSS-->>-EMC: 
    EMC-->>-Axeda: ok
    
    Axeda->>+EMC: /lwPing Ping (or other)
        EMC->>+Thing: DequeueEgress()
            Thing->>+FTSS: dequeueTransferJobs()
                FTSS--)resa: check if reservation available
                Note right of resa: < Total Max Edge-Controlled File Transfers Allowed <br/> < Total Max Edge-Controlled File Transfers Allowed Per Thing
                FTSS--)resa: create reservation
                FTSS--)ftjob: dequeue ft job and assign reservation
                Note over ftjob: Online queue
                FTSS->>+ftjob: start ft job
                    Note over ftjob: ACTIVE
                    ftjob->>EMC: StartFileTransfer()
                ftjob-->>-FTSS: 
            FTSS-->>-Thing: 
        Thing-->>-EMC: 
        EMC->>FTSS: GetActiveTransferJob()
    EMC-->>-Axeda: FT jobs detail
    
    Axeda->>+EMC: <Ps> Job QUEUED
    EMC-->>-Axeda: 
    
    Axeda->>+EMC: <Ps> Job STARTED
    EMC->>FTSS: GetActiveTransferJob()
    EMC->>FTSS: UpdateTransferStatus() > state=ACTIVE
    EMC-->>-Axeda: 
    
    loop
    Axeda->>+EMC: /upload File Upload
    EMC->>FTSS: GetConfigurationTable()
    EMC->>FTSS: IsTransferJobActive()
    EMC->>FTSS: TouchTransferJob()
    alt First chunk
        EMC->>repo: CreateBinaryFile()
    else
        EMC->>repo: WriteToBinaryFile()
    end
    EMC->>FTSS: UpdateTransferStatus() > Byte written, ...
    EMC-->>-Axeda: 
    end

    Axeda->>+EMC: <Ps> Job SUCCESS
        EMC->>FTSS: GetActiveTransferJob() 
        EMC->>+FTSS: UpdateTransferStatus() > state=VALIDATED
            FTSS->>+FTSS: completeTransferJob()
                FTSS->>ftjob: complete ft job
                Note over ftjob: VALIDATED
                FTSS--xresa: return reservation
                FTSS--xftjob: remove ft job
            FTSS->>-FTSS: compare checksum
        FTSS-->>-EMC: 
    EMC-->>-Axeda: 
```

#### 9.6.1 with bindless egress (happy path)

``` mermaid
sequenceDiagram
    box Edge
    participant Axeda as Axeda device
    end
    box Connector
    participant EMC as eMessage
    end
    box Platform
    participant Thing as AxedaThing
    participant FTSS as FileTransferSubsystem
    participant repo as Repository
    participant ftjob as TransferJob
    participant resa as Reservation
    end
    
    Axeda->>+EMC: <Ur> Upload Request
        EMC->>+FTSS: Copy() async / queuable
            FTSS--)ftjob: enqueue ft job
            Note right of ftjob: <  Max File Transfers Allowed in Offline Queue  <br/> <  Max File Transfers Allowed in Offline Queue Per Thing
            Note over ftjob: PENDING
            Note over ftjob: Offline queue
        FTSS-->>-EMC: 
    EMC-->>-Axeda: ok
    
    Axeda->>+EMC: /lwPing Ping (or other)
        EMC->>+Thing: DequeueAndGetEgress()
            Thing->>+FTSS: dequeueTransferJobs()
                FTSS--)resa: check if reservation available
                Note right of resa: < Total Max Edge-Controlled File Transfers Allowed <br/> < Total Max Edge-Controlled File Transfers Allowed Per Thing
                FTSS--)resa: create reservation
                FTSS--)ftjob: dequeue ft job and assign reservation
                Note over ftjob: Online queue
                FTSS->>+ftjob: start FileTransferTask
                Note over ftjob: ACTIVE
                ftjob-->>-FTSS: 
            FTSS-->>-Thing: 
        Thing-->>-EMC: 
        EMC->>FTSS: AcknowledgeDelivery()
    EMC-->>-Axeda: FT jobs detail
    
    Axeda->>+EMC: <Ps> Job QUEUED
    EMC-->>-Axeda: 
    
    Axeda->>+EMC: <Ps> Job STARTED
    EMC->>FTSS: GetActiveTransferJob()
    EMC->>FTSS: UpdateTransferStatus() > state=ACTIVE
    EMC-->>-Axeda: 
    
    loop
    Axeda->>+EMC: /upload File Upload
    EMC->>FTSS: GetConfigurationTable()
    EMC->>+FTSS: PerformChunkUpload()
        alt First chunk
            FTSS->>repo: CreateBinaryFile()
        else
            FTSS->>repo: WriteToBinaryFile()
        end
        FTSS->>ftjob: update ft job 
    FTSS-->>-EMC: 
    EMC->>FTSS: GetActiveTransferJob()
    EMC->>FTSS: UpdateTransferStatus() > Byte written, ...
    EMC-->>-Axeda: 
    end

    Axeda->>+EMC: <Ps> Job SUCCESS
        EMC->>FTSS: GetActiveTransferJob() 
        EMC->>+FTSS: UpdateTransferStatus() > state=VALIDATED
            FTSS->>+FTSS: completeTransferJob()
                FTSS->>ftjob: complete ft job
                Note over ftjob: VALIDATED
                FTSS--xresa: return reservation
                FTSS--xftjob: remove ft job
            FTSS->>-FTSS: compare checksum
        FTSS-->>-EMC: 
    EMC-->>-Axeda: 
```

## Platform-Controlled (AlwaysOn devices)

All steps in this file transfer are controlled by the Platform.
