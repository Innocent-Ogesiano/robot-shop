# Robot Shop Availability SLO and Error Budget Policy

**Service:** Robot Shop storefront and API (reached through the `web` service)
**Owner:** Innocent Ekweogu | **Reviewed:** 2026-10-16 | **Next review:** one month later

## 1. SLO

**99.5% of availability probes succeed over a rolling 30 days.**

- **SLI:** the share of successful HTTP probes. The blackbox exporter probes six URLs through `web` every 15 seconds. A probe succeeds on an HTTP 2xx response within the timeout.
- **Query:** `avg(avg_over_time(probe_success{job="probe/monitoring/robot-shop"}[30d]))`
- **Limit of this SLI:** the probes are synthetic and run at a fixed rate. They show whether the shop is reachable, not how many real users are affected.

## 2. Error budget

The budget is 0.5% of probes, which is about **3.6 hours of full outage per 30 days**.

Budget remaining (1 = untouched, 0 = spent, negative = SLO broken):
`1 - ((1 - avg(avg_over_time(probe_success{job="probe/monitoring/robot-shop"}[30d]))) / 0.005)`

It is shown on the Robot Shop Grafana dashboard.

## 3. Alerting

| Alert | Fires when | Action |
|---|---|---|
| `RobotShopAvailabilityBurn` | More than 7.2% of probes fail over 5 minutes, for 2 minutes. This is 14.4 times the allowed error rate. | Posts to Slack `#<channel>`. The owner investigates right away. |

## 4. What we do, by budget remaining

| Budget remaining | Status | Policy |
|---|---|---|
| More than 50% | Healthy | Release as normal. |
| 25% to 50% | Watch | Review risky changes before release. Put reliability fixes at the top of the backlog. |
| Below 25% | At risk | Slow down releases. Only ship changes that reduce failures or are low risk. Find the main cause of failed probes. |
| 0% or less | Exhausted | Freeze feature releases until the budget recovers above 25%. Write a short postmortem. Only reliability, security and rollback changes may ship. |

## 5. Rules and exceptions

- Security fixes and rollbacks may ship at any budget level.
- The owner decides when a freeze starts and ends, and records the reason.
- Planned maintenance is announced in advance and still counts against the budget, because users are affected either way.
- After every incident that uses more than 10% of the budget, write down the cause, the fix and a follow-up.

## 6. Review

Check the SLO target and the probe list each month. If the budget is always unused, tighten the target. If it is always spent, fix the service before loosening the target.
