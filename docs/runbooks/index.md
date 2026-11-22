# Runbooks

A runbook answers one question for the person on call: an alert fired, what now? Every Prometheus alert carries a `runbook_url` that links straight to its section here.

| Service | Alerts | Runbook |
| --- | --- | --- |
| Every model-serving API on the cluster | `SLOAvailabilityFastBurn`, `SLOAvailabilitySlowBurn`, `SLOLatencyFastBurn`, `SLOLatencySlowBurn`, `ModelNotServed` | [SLO alerts](slo.md) |
| Churn API in the Docker stack | `ChurnApiDown`, `ChurnApiHighLatency`, `ChurnApiHighErrorRate`, `ChurnPredictionShift` | [Churn API](churn-api.md) |

The objectives, burn-rate windows and error budget policy behind the SLO alerts are in [SLOs](../slos.md).

## Common moves

| Situation | Command |
| --- | --- |
| Roll a model back | `uv run churn promote --from prod-previous --to prod --approved-by "<you>"` (on the cluster, set `CHURN_MLFLOW_TRACKING_URI=http://localhost:5001` after `make registry-ui`) |
| Roll an image back on the cluster | Revert the promotion PR; Argo CD syncs the previous tag |
| Check drift without retraining | `uv run churn monitor --no-retrain` |
| Containers and logs (Docker) | `make ps`, `sh scripts/compose.sh logs --tail 100 churn-api` |
