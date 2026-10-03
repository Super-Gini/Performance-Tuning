# Production Health Checks

Comprehensive point-in-time assessments of production Oracle environments. Different from [Checklists](../28-checklists/index.md) which are cadence-based — health checks are one-shot deep audits typically run:

- Quarterly (planned).
- Before major change windows.
- After a serious incident.
- As part of an audit or handover.

## Contents

| Page                                                  | Focus                             |
| ----------------------------------------------------- | --------------------------------- |
| [Database Health Check](database-health-check.md)     | Full DB audit — 50+ items         |
| [RAC Health Check](rac-health-check.md)               | Cluster-specific                  |
| [Data Guard Health Check](data-guard-health-check.md) | DR readiness                      |
| [Storage Health Check](storage-health-check.md)       | ASM, IO, capacity                 |
| [Security Health Check](security-health-check.md)     | Users, roles, network, encryption |

## Deliverable Format

Each health check produces:

- **Executive Summary** — 1-page red/yellow/green.
- **Detailed Findings** — per-topic, each with a severity, evidence, recommendation.
- **Action List** — prioritized, with owners.
- **Metrics Baseline** — captured at the time of the check.

## Related

- [Checklists](../28-checklists/index.md) — cadence-based.
- [Monitoring](../24-monitoring/index.md).
- [Scripts](../30-scripts/index.md).
