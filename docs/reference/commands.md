# Commands

Every workflow starts from `make` at the repository root. The targets call the `churn` and `rag` CLIs, Docker Compose, k3d, Terraform and kubectl.

## make targets

### Setup and checks

| Target | What it does |
| --- | --- |
| `make install` | Create `.venv` with all packages and dev tools, install the pre-commit hooks |
| `make lint` | ruff check, ruff format check, mypy on both packages |
| `make test` | pytest |
| `make check` | `lint` + `test`: everything CI runs for the Python code |
| `make docs-serve` | This site with live reload on <http://127.0.0.1:8200> |
| `make docs-build` | Build the site into `site/`, as the docs workflow does |

### Churn

| Target | What it does |
| --- | --- |
| `make mlflow` | Local MLflow server without Docker on <http://127.0.0.1:5000> |
| `make download` | Download the raw dataset |
| `make data` | Pull the versioned dataset with DVC (falls back to `download`) |
| `make train` | Train, evaluate, register as `@dev` if the model passes the gate |
| `make pipeline [TUNE=30]` | Prefect flow: fetch → [tune] → train + gate + model card → `@staging` |
| `make promote FROM=… TO=… APPROVED_BY="…" [FAIRNESS_REVIEWED=1]` | Move an alias (defaults: `FROM=dev TO=prod`) |
| `make features` | Publish features to Feast (needs Redis: `make up`) |
| `make serve` | The churn API on <http://127.0.0.1:8000/docs> |
| `make simulate [N=300] [DRIFT=1]` | Send customer traffic to the API |
| `make monitor` | Prefect drift check on recent predictions; retrains on drift |
| `make demo-drift` | Normal traffic → drifted traffic → drift check → retrain to staging |

### Support assistant

| Target | What it does |
| --- | --- |
| `make rag-ingest` | Index the help-center corpus |
| `make rag-eval` | Retrieval and safety gates, as in CI |
| `make rag-eval-gen` | Judge calibration, answer-quality and action gates with local LLMs |

### Docker

| Target | What it does |
| --- | --- |
| `make up` | MLflow + Redis |
| `make serve-docker` | API + feature server + Prometheus (:9090) + Grafana (:3000); needs a prod model |
| `make ps` | Container status |
| `make down` | Stop everything |

### Kubernetes platform

| Target | What it does |
| --- | --- |
| `make cluster-up` | k3d cluster `telcoai` (API on :6550, ingress on :8080) + kubeconfig |
| `make cluster-down` | Delete the cluster |
| `make kubeconfig` | Write `platform/.kube/telcoai.yaml` |
| `make platform-apply` | Terraform: namespaces, quotas, Argo CD, root app |
| `make argocd-ui` | Argo CD on <http://localhost:8081> (user `admin`, password printed first) |
| `make registry-ui` | Shared MLflow registry on <http://localhost:5001> |
| `make grafana-ui` | Cluster Grafana on <http://localhost:3001> (anonymous read-only), dashboard *Platform SLOs* |
| `make prometheus-ui` | Cluster Prometheus on <http://localhost:9090>; stop the Docker stack's Prometheus first (same port) |
| `make slo-report [WINDOW=7d]` | SLO report per environment and service (needs `make prometheus-ui`) |
| `make promote-image [SERVICE=churn-api] ENV=test [SHA=<commit>]` | Open the PR that deploys an image (then `ENV=prod`) |
| `make new-service NAME=<name>` | Scaffold a model service wired into the platform |

## churn CLI

`uv run churn <command> --help` lists each command's options.

| Command | What it does |
| --- | --- |
| `download` | Download the raw Telco churn dataset |
| `train` | Train, evaluate and register a model |
| `promote` | Move a model version between aliases (`--from`, `--to`, `--approved-by`, `--fairness-reviewed`) |
| `pipeline` | Run the Prefect training flow (`--tune-trials N`) |
| `features` | Publish features to Feast (offline + online store) |
| `monitor` | Run the Prefect drift monitoring flow (`--no-retrain` to only report) |
| `simulate` | Send customer traffic to the API (`--n`, `--drift`, `--seed`) |
| `serve` | Run the scoring API |

## rag CLI

`uv run rag <command> --help` lists each command's options.

| Command | What it does |
| --- | --- |
| `ingest` | Chunk, embed and index the help-center corpus |
| `search` | Search the index |
| `ask` | Answer a question with the configured LLM |
| `serve` | Run the HTTP API |
| `eval-retrieval` | Score retrieval on the golden questions (`--gate`, `--dense`, `--no-sparse`) |
| `eval-safety` | Score the input guardrails on `eval/safety.yaml` (`--gate`, `--rules-only`) |
| `train-injection` | Train the prompt-injection classifier artifact |
| `eval-actions` | Check that the model proposes credits exactly when asked (`--gate`) |
| `audit` | Query the request traces (`--reason`, `--request-id`) |
| `calibrate-judge` | Score the LLM judge on known answers |
| `eval-generation` | Answer quality: facts, language, LLM judge (`--gate`, `--model`, `--prompt-version`) |
