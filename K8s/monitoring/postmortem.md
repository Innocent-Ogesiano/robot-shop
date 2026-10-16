# Postmortem: Catalogue outage caused by a MongoDB host that did not resolve

**Date:** 2026-10-16 | **Severity:** Drill (planned failure) | **Author:** Innocent Ekweogu | **Status:** Draft

## Summary
At about 11:31 UTC a change set the catalogue service's `MONGO_URL` to `mongodb://wronghost:27017/catalogue`. The host does not resolve, so catalogue could not connect to MongoDB and its product endpoints failed. Probes to `/api/catalogue/products` and `/api/catalogue/categories` failed for about 7 minutes. The change was rolled back with `kubectl rollout undo` at about 11:39 UTC, and service recovered.

## Impact
- **Affected:** the product list and category endpoints, so users could not browse products. The other four probed endpoints (`/`, `user/uniqueid`, `cart/metrics`, `payment/metrics`) were not affected.
- **Duration:** probes failed from 11:32 to 11:38 UTC, 7 consecutive minutes.
- **Failure rate during the incident:** 2 of 6 probe targets, which is 33% and above the 7.2% fast-burn alert threshold.
- **Error budget used:** about 2.3 failed minutes out of the 216 minutes allowed per 30 days (0.5%), which is about 1%. Because the probe history is only hours long, treat this as an estimate.

## Timeline (UTC, 2026-10-16)
| Time | Event |
|---|---|
| 11:30:59 | The bad catalogue ReplicaSet `catalogue-6d49c8894f` is created, with `MONGO_URL` pointing to `wronghost`. The rollout begins. |
| 11:32 | The first catalogue probe fails. |
| <time> | `RobotShopAvailabilityBurn` fires and is sent to Slack. <fill in from Slack> |
| <time> | The operator sees the alert and starts the response. |
| ~11:39 | `kubectl rollout undo deploy/catalogue` brings back `catalogue-658b585c4d` and removes the bad ReplicaSet (scaled to 0). |
| 11:38 to 11:39 | Probes recover. Errors return to 0% on the dashboard. |

## Detection
The failure showed up on the Robot Shop dashboard as errors on the catalogue endpoints. The alert is meant to catch this, since 33% of probes failing is above the 7.2% threshold, but it needs a failure for 2 minutes plus Alertmanager's `group_wait`, so notification is delayed by a few minutes. <note the actual delay and whether Slack delivered it>

## Mitigation
The fastest safe action was a rollback, not a fix forward: `kubectl -n robot-shop-namespace rollout undo deploy/catalogue`. It restored the previous ReplicaSet, which still had the correct configuration. Time from first failure to recovery was about 7 minutes.

## Root cause
Catalogue reads its database address from the `MONGO_URL` environment variable. A deployment change set that value to `mongodb://wronghost:27017/catalogue`. The host `wronghost` is not a Kubernetes service, so DNS lookup failed. The catalogue code reports `mongo: false` on its health check and returns no products when it is not connected to Mongo.

**Why it was not caught before reaching users:**
1. The `catalogue` Deployment has no readiness probe that checks the database connection, so Kubernetes treated the new pod as healthy and moved traffic to it.
2. A configuration change has no validation step before rollout.

Evidence:
- `kubectl get rs catalogue-6d49c8894f -o yaml` shows `MONGO_URL=mongodb://wronghost:27017/catalogue`, and the rollback restored the previous ReplicaSet.
- Loki has error lines from the bad pod `catalogue-6d49c8894f-g472n`, for example at 11:39:02 UTC: `{"level":"error","msg":"ERROR {\"name\":\"MongoNetworkError\"}"}`. Query: `{kubernetes_labels_service="catalogue"} |= "MongoNetworkError"`.

## What went well
- The rollback was quick and safe, and it needed no knowledge of the cause.
- Metrics, logs (`MongoNetworkError` in Loki) and the rollout history together pointed to a database connection problem caused by a configuration change.
- The probe and dashboard showed the failure and the recovery clearly.

## What went badly
- The Slack alert needed fixing before it could deliver (the webhook channel was wrong). <confirm whether the alert reached Slack during this incident>
- The service got traffic while it could not reach its database.
- The catalogue error message is generic. It logs only `ERROR {"name":"MongoNetworkError"}`, with no host name or reason, so the logs show that Mongo was unreachable but not which address was wrong. The wrong `MONGO_URL` had to be found from the Deployment spec.

## Action items
| Action | Owner | Priority |
|---|---|---|
| Add a readiness probe to catalogue that fails when `mongo` is false (the `/health` endpoint already returns this) | <owner> | High |
| Add a rollout check to the deploy process so a failing readiness probe stops the rollout and rolls back automatically | <owner> | High |
| Add a Prometheus alert for pods that are not ready or are restarting, as an earlier signal than the SLO alert | <owner> | Medium |
| Keep the MongoDB address in the Helm chart values, not set by hand with `kubectl set env` | <owner> | Medium |
| Make the catalogue connection error log the target host and the underlying reason, not only the error name | <owner> | Low |
