# Quarterly Checklist

Once a quarter. Deeper reviews.

## 1. DR / BCP Exercise

- [ ] Full-scale switchover to standby, real workload run.
- [ ] Switchback and verify no data loss.
- [ ] Measured RTO / RPO vs SLA.
- [ ] Retro on any surprises.

## 2. RU Application

- [ ] Apply current-quarter Release Update on production (or approved tier).
- [ ] Datapatch run and verified.
- [ ] Post-patch smoke tests pass.
- [ ] AWR compare — no regression.

## 3. Security Audit

- [ ] Full privileges audit.
- [ ] STIG / CIS compliance scan.
- [ ] Certificate rotations (OEM, Wallet, TDE keys).
- [ ] TDE keystore rekey (if policy requires).

## 4. Performance Review

- [ ] Top SQL by DB time — action list.
- [ ] Storage IO trends.
- [ ] Buffer cache hit ratio trend.
- [ ] Deadlocks/latch waits trend.

## 5. Capacity Review

- [ ] CPU utilization 95th percentile per DB.
- [ ] Storage growth actual vs forecast.
- [ ] Memory utilization.
- [ ] Order growth-driven purchases.

## 6. Health Checks

- [ ] Full [Database Health Check](../35-production-health-checks/database-health-check.md).
- [ ] Data Guard health check.
- [ ] RAC health check.
- [ ] Security health check.
- [ ] Storage health check.

## 7. Runbook & Documentation

- [ ] Update every runbook touched by an incident this quarter.
- [ ] Version-control changes.
- [ ] Publish updated architecture diagram.

## 8. Team

- [ ] Quarterly training goals.
- [ ] Certification renewals.
- [ ] Cross-training coverage (no single-point-of-failure DBA).

## 9. Governance

- [ ] Change advisory board — any Oracle-related standing items.
- [ ] Vendor: Oracle licensing utilization vs entitlement.
- [ ] Third-party: OEM plugins current, RMAN MML supported.

## Signoff

Executive summary of the quarter's Oracle posture.

## Related

- [Yearly](yearly.md).
- [Production Health Checks](../35-production-health-checks/index.md).
