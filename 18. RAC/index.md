# Real Application Clusters (RAC)

**Oracle Real Application Clusters** is a shared-disk clustered database: multiple Oracle **instances** on separate hosts mount the **same** physical database. RAC provides horizontal scalability (add instances to add throughput) and high availability (an instance failure doesn't take down the database).

RAC is layered on **Oracle Grid Infrastructure (GI)** — which includes **Oracle Clusterware** (heartbeat, membership, resource management) and **ASM** (shared storage).

Licensed EE + Real Application Clusters option.

## Contents

| Page                              | Purpose                                     |
| --------------------------------- | ------------------------------------------- |
| [Clusterware](clusterware.md)     | Grid Infrastructure, CRS resources, OLR/OCR |
| [CRSD](crsd.md)                   | Cluster Ready Services daemon               |
| [CSSD](cssd.md)                   | Cluster Synchronization Services daemon     |
| [OCR](ocr.md)                     | Oracle Cluster Registry                     |
| [Voting Disk](voting-disk.md)     | Cluster membership consensus                |
| [Cache Fusion](cache-fusion.md)   | Block shipping between instances            |
| [GCS](gcs.md)                     | Global Cache Services                       |
| [GES](ges.md)                     | Global Enqueue Services                     |
| [SCAN Listener](scan-listener.md) | Single Client Access Name                   |
| [Services](services.md)           | RAC service management                      |
| [Evictions](evictions.md)         | Node eviction — causes, prevention          |

## Related

- [ASM](../19-asm/index.md) — shared storage layer.
- [Data Guard](../17-data-guard/index.md) — DR beyond the cluster.
- [Runbook / RAC Node Eviction](../27-runbooks/rac-node-eviction.md).
