# Cache Fusion

## Block Modes and PCM Resources

```mermaid
flowchart LR
    subgraph GCS[Global Cache Service - per master instance]
        Res[GCS Resource per DBA<br/>owner instance + modes]
    end
    subgraph Inst1[Instance 1]
        LMS1[LMS0..N]
        BC1[Buffer Cache<br/>V$BH.STATE]
    end
    subgraph Inst2[Instance 2]
        LMS2[LMS0..N]
        BC2[Buffer Cache]
    end
    subgraph Inst3[Instance 3]
        LMS3[LMS0..N]
        BC3[Buffer Cache]
    end
    IC[Private Interconnect]
    Inst1 <--> IC
    Inst2 <--> IC
    Inst3 <--> IC
    Res -.tracks.-> BC1
    Res -.tracks.-> BC2
    Res -.tracks.-> BC3
```

## 2-Way Transfer (Requester + Master/Holder)

```mermaid
sequenceDiagram
    participant R as Requester (Inst2)
    participant M as Master + Holder (Inst1)

    R->>M: gc cr request DBA=X mode=S
    M->>M: LMS locates block
    M->>M: prepare CR image if needed
    M-->>R: ship block + grant
    Note right of R: gc cr block 2-way wait
```

## 3-Way Transfer (Requester -> Master -> Holder)

```mermaid
sequenceDiagram
    participant R as Requester (Inst3)
    participant M as Master (Inst1)
    participant H as Holder (Inst2)

    R->>M: gc current request DBA=X mode=X
    M->>H: forward request
    H->>H: prepare block + PI copy
    H-->>R: ship block
    H-->>M: ACK + updated state
    Note right of R: gc current block 3-way wait
```

## Past Image (PI) Block

```mermaid
flowchart LR
    subgraph Before[Before Transfer]
        I1[Inst1 X-holder, dirty]
    end
    subgraph After[After Transfer to Inst2]
        I1P[Inst1 keeps PI - dirty pre-transfer image<br/>V$BH.STATE=8]
        I2X[Inst2 X-holder - current dirty]
    end
    Before --> After
```

## Grant vs Read from Disk

```mermaid
sequenceDiagram
    participant R as Requester
    participant M as Master
    participant Disk

    R->>M: request block
    M-->>R: no instance has it - GRANT
    Note over R: gc cr grant 2-way (short)
    R->>Disk: db file sequential read
    Disk-->>R: block
```
