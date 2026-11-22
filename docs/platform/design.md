# Design decisions

- **Terraform installs the platform, Argo CD deploys the workloads.** Terraform owns things that change rarely and need a plan/apply review (namespaces, budgets, Argo CD itself). Services change daily and belong to GitOps, where every deployment is a commit and a rollback is `git revert`.
- **One registry, one alias per environment.** A model is never copied between environments; it is promoted. The version prod serves is the exact artifact that was tested, and its approval is recorded on it.
- **Immutable tags outside dev.** `main` moves; `sha-<commit>` never does. What runs in test and prod is readable from git, and prod can only run an image that was deployed to test first (CI reads the history of the test values).
- **Budgets per namespace.** Every namespace has a `ResourceQuota` and a `LimitRange`, so a container without resource settings still gets defaults, and one environment cannot starve another. The quota also counts the headroom a rolling update needs; the bootstrap job's CPU limit was lowered for this.
- **The bootstrap job only trains, and only once.** It is a sync hook (a plain Job's spec is immutable, so a new image would fail the sync) that exits at once if a `@dev` model exists. Training sets `@dev` only if the model passes the quality gate; the job never promotes.
- **Least privilege.** The API runs as a non-root user with a read-only root filesystem and no Linux capabilities; writable paths are `emptyDir` volumes.

## Found while testing on the cluster

- **MLflow was OOM-killed at 1.5 GiB.** MLflow 3.x starts a GenAI job runner next to the server: 8 more Python processes, 1.8 GiB in total instead of 0.6 GiB (measured with `docker stats`). The kernel log showed the kills; the container status only said `Error`, exit 137. A model registry does not need the runner, so `MLFLOW_SERVER_ENABLE_JOB_EXECUTION=false`.
- **Default probes killed a healthy MLflow.** The 1 s default timeout is too short for a Python server starting under a CPU limit. A `startupProbe` covers the slow start, and the liveness probe only starts after it passes.
- **The API used to exit when its alias had no model**, so a fresh environment crash-looped, and after a promotion it could wait out the 5-minute restart back-off. Now it starts unready (`/health` 503, `/livez` 200 for the liveness probe) and follows its alias: on the cluster, moving an alias from version 1 to 2 switched the running pod in about 10 s, with no restart. The same path applies a rollback.
- **A registry outage would have blocked API startup for minutes.** MLflow's client retries 7 times with exponential backoff and a 120 s timeout by default. The API retries in its own loop, so it now sets 1 retry and a 10 s timeout: with the registry unreachable it starts unready in seconds (found while testing the service template's image).
