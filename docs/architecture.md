# Architecture

The repository is one uv workspace with two ML products and the platform that runs them. The products are what a product team would own; the platform and the service template are what a central ML platform team would provide.

```mermaid
flowchart TB
    platform["<b>Platform</b><br/>k3d · Terraform · Argo CD<br/>dev / test / prod"]
    churn["<b>Churn prediction</b><br/>training + gate · model card<br/>API · drift → retrain"]
    rag["<b>Support assistant</b><br/>retrieval · guardrails<br/>cited answers · approvals"]
    registry[("<b>MLflow</b><br/>registry @dev @staging @prod<br/>runs · traces")]
    feast[("Feast<br/>parquet + Redis")]
    qdrant[("Qdrant<br/>e5 embeddings")]
    llm["LLM via LiteLLM<br/>local Ollama"]
    obs["Prometheus + Grafana<br/>SLO alerts"]

    platform -- "deploys" --> churn
    platform -- "hosts" --> registry
    churn -- "registers, serves alias" --> registry
    feast -- "features" --> churn
    churn -- "/metrics" --> obs
    rag -- "traces, eval runs" --> registry
    qdrant -- "sources" --> rag
    rag -- "prompt" --> llm
```

Each component page has the detailed flow: [churn pipeline](churn/index.md#pipeline), [assistant retrieval](support-rag/index.md#retrieval-pipeline), [GitOps](platform/index.md).

## Shared building blocks

| Concern | Choice | Used by |
| --- | --- | --- |
| Experiment tracking, model registry, traces | MLflow 3 with **aliases** (`@dev`, `@staging`, `@prod`); stages are deprecated | churn, assistant (traces and evaluation runs), platform |
| Configuration | pydantic-settings with `CHURN_*` and `RAG_*` environment variables | both products ([Configuration](reference/configuration.md)) |
| Serving | FastAPI with a validated request contract | churn API, assistant API, generated services |
| Packaging | One uv workspace, one lock file, non-root container images | everything |
| Deployment | Helm charts deployed by Argo CD from `main` | platform |

## How a model reaches production

Code and models move on separate tracks, and each move leaves a record.

```mermaid
flowchart LR
    t["Training<br/>passes the gate"] --> dev["@dev<br/>served in dev"]
    dev -- "make promote<br/>FROM=dev TO=staging" --> stg["@staging<br/>served in test"]
    stg -- "make promote<br/>FROM=staging TO=prod<br/>APPROVED_BY=… + model card<br/>+ fairness review" --> prod["@prod<br/>served in prod"]
    prod -. "rollback:<br/>FROM=prod-previous" .-> prod
```

- **Models** are never copied between environments. All environments read one registry and each serves one alias, so the version in prod is the exact artifact that was tested. The approver and timestamp are tags on the version.
- **Images** move by pull request: `make promote-image ENV=test`, then `ENV=prod`, each editing one immutable `sha-` tag. CI refuses a prod image that hasn't run in test.
- **APIs follow their alias:** a promotion or rollback takes effect within 30 seconds, without a restart.

See [Promotion](platform/promotion.md) for both tracks in detail.

## Repository layout

| Path | What |
| --- | --- |
| `churn/` | Churn package: data, training, registry, model cards, API, Feast definitions, drift monitoring |
| `support-rag/` | Assistant package, plus its corpus, prompts, evaluation sets and injection classifier |
| `platform/` | k3d cluster, Terraform, Helm charts, Argo CD apps and per-environment values, promotion scripts |
| `templates/ml-service/` | Copier template for new model services |
| `monitoring/` | Prometheus rules and Grafana dashboards for the Docker stack |
| `docs/` | This site (Zensical) |
| `scripts/` | Docker Compose wrapper |
| `data/` | DVC pointers to the raw data; local outputs (gitignored) |
| `.github/workflows/` | One CI workflow per component, plus the docs site |
