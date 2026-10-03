# Checklists

Recurring DBA health/maintenance checks organized by cadence. Every DBA team should have a subset (or all) of these running as automated jobs with escalation.

## Contents

| Page                      | Purpose                    |
| ------------------------- | -------------------------- |
| [Daily](daily.md)         | Every morning verification |
| [Weekly](weekly.md)       | Once-a-week rollups        |
| [Monthly](monthly.md)     | Trend + maintenance        |
| [Quarterly](quarterly.md) | Deeper reviews, DR test    |
| [Yearly](yearly.md)       | Strategy, license, audit   |

## Automating

Most items should be scripted (see [Scripts](../30-scripts/index.md)) and shipped through:

- Cron / DBMS_SCHEDULER for the check itself.
- Email or ITSM ticket for follow-up.
- Dashboard (Grafana / OEM) for trend visibility.

## Related

- [Monitoring](../24-monitoring/index.md).
- [Scripts](../30-scripts/index.md).
- [Production Health Checks](../35-production-health-checks/index.md) — comprehensive one-shots.
