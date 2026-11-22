# MLOps & LLMOps Platform

An end-to-end MLOps and LLMOps platform: it takes classic ML and GenAI models from experimentation to governed production, with the engineering standards a central ML platform team provides to product teams. The example use cases come from a telecom operator: customer churn prediction and a multilingual support assistant.

!!! note "Fictional operator"
    The operator **"Nordalp Mobile"** and all its help articles, prices and URLs (`*.nordalp.example`) are fictional, written for this project. The churn data is IBM's public Telco Customer Churn dataset.

## Components

| Component | What it shows | Status |
| --- | --- | --- |
| [Churn prediction](churn/index.md) (`churn/`) | Validated data, reproducible training, MLflow registry with dev → staging → prod aliases, quality gate, model cards, FastAPI serving, Feast, drift-triggered retraining, CI | Done |
| [Support assistant](support-rag/index.md) (`support-rag/`) | Multilingual (DE/FR/IT/EN) customer-support RAG assistant: cited answers from a local LLM, guardrails, evaluation gates, tracing and audit log, human approval for actions | Done |
| [Platform](platform/index.md) (`platform/`) | Local Kubernetes (k3d), Terraform, Argo CD GitOps: dev/test/prod with budgets, one shared model registry, image promotion by PR with CI-enforced rules, APIs that follow their model alias | In progress |
| [Service template](platform/service-template.md) (`templates/ml-service/`) | `make new-service NAME=…`: a model service already wired into the platform (API, tests, image, CI, Helm chart, Argo CD app, model card), tested in CI | Done |

## Engineering standards

- **Reproducible:** Python and every dependency pinned in `uv.lock`; seeded training; data and parameters logged per run.
- **Quality gates:** schema validation (pandera) before training; a model below the metric threshold is never registered. The assistant has retrieval, safety and answer-quality gates.
- **Promotion by alias:** training only ever writes `dev`. Promotion is an explicit step, and the previous version is kept as `<env>-previous` for one-command rollback.
- **Governance:** every registered version ships with a generated model card (intended use, data lineage, per-group performance, fairness gaps, blind spots). Promotion to prod needs a named approver and an explicit fairness acknowledgement when the card flags one; both are recorded on the version.
- **GitOps:** nothing is deployed by hand. Argo CD deploys what `main` says, and a production deployment is a reviewed pull request.
- **CI on every change:** ruff, mypy, pytest, evaluation gates, chart and manifest validation, then a container build.

## Where to start

<div class="grid cards" markdown>

- **[Get started](getting-started.md)**: install, train a model and serve it in five commands.
- **[Architecture](architecture.md)**: how the components fit together.
- **[Commands](reference/commands.md)**: every `make` target and CLI command.
- **[Roadmap](roadmap.md)**: what is done and what comes next.

</div>
