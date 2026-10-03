# Yearly Checklist

Once per year — strategic + compliance.

## 1. Licensing & Support

- [ ] Reconcile Oracle license entitlements vs actual deployment.
- [ ] EE feature usage audit (`DBA_FEATURE_USAGE_STATISTICS`).
- [ ] MOS support renewal on time.
- [ ] Third-party contract renewals (RMAN MML, backup vendor).

```sql
SELECT   name, detected_usages, first_usage_date, last_usage_date, currently_used
FROM     dba_feature_usage_statistics
WHERE    currently_used = 'TRUE'
ORDER BY last_sample_date DESC;
```

## 2. Version Strategy

- [ ] Current DB version vs Premier / Market-Driven end dates.
- [ ] Plan upgrades ahead of extended support gaps.
- [ ] Confirm 19c → next-LTR migration on the roadmap.

## 3. DR / BCP

- [ ] Annual full DR exercise: primary declared down, standby serves for a full workday.
- [ ] Update BCP documentation.

## 4. Backup Strategy

- [ ] Restore-from-tape test if using tape.
- [ ] Off-site copies retention confirmed.
- [ ] Encryption keys backed up separately, tested.

## 5. Compliance

- [ ] SOX / HIPAA / PCI / GDPR audit prep.
- [ ] Retention policies match legal requirements.
- [ ] Data classification reviewed.
- [ ] TDE/wallet key rotation confirmed.

## 6. Access Review

- [ ] Full access review — every privileged account.
- [ ] Deprovision terminated users.
- [ ] Confirm SoD (Segregation of Duties).

## 7. Architecture

- [ ] Review architecture vs current best practices.
- [ ] Modernization candidates (Bigfile, ASM, RAC, Data Guard, cloud).
- [ ] EoL hardware/OS/DB.

## 8. Vendor Support

- [ ] Meet with Oracle Account team; review roadmap.
- [ ] Renew premium support / ACS if applicable.

## 9. Team Development

- [ ] Individual development plans.
- [ ] Backfills for succession.
- [ ] Cross-training completeness.

## 10. Strategic Planning

- [ ] Present state-of-databases to leadership.
- [ ] 3-year technology roadmap.

## Signoff

Executive report; board of directors' Q4 packet if relevant.

## Related

- [Quarterly](quarterly.md).
