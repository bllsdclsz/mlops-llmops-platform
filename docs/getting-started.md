# Get started

Everything runs on a laptop. The cloud is optional.

## Requirements

| Tool | Needed for |
| --- | --- |
| [uv](https://docs.astral.sh/uv/) | Everything: it installs Python 3.14 and every package from `uv.lock` |
| Docker engine (Docker Desktop, Rancher Desktop, …) | The full stack in Docker and the Kubernetes platform |
| [Ollama](https://ollama.com/) ≥ 0.34 with `granite4.2:8b` | Answers from the support assistant (retrieval and its gates work without it) |
| k3d, Terraform, Helm, kubectl | The Kubernetes platform (`scoop install k3d terraform helm kubectl` on Windows) |

```bash
make install          # .venv with all packages and dev tools + pre-commit hooks
```

## Churn model, without Docker

```bash
make mlflow           # MLflow at http://127.0.0.1:5000 (leave it running)
make download train   # fetch data, train, register as telco-churn@dev if it passes the gate
make promote FROM=dev TO=prod APPROVED_BY="Your Name"   # add FAIRNESS_REVIEWED=1 after reading a flagged model card
make serve            # API at http://127.0.0.1:8000/docs
```

The API's `/docs` page shows the request schema and can send a test request. See [Churn prediction](churn/index.md) for the pipeline and the model.

## Full stack in Docker

MLflow, Redis with the Feast feature server, the API, Prometheus and Grafana:

```bash
make up data train features
make promote APPROVED_BY="Your Name" FAIRNESS_REVIEWED=1   # after reading the model card in MLflow
make serve-docker
make demo-drift       # simulated drift -> automatic retrain to staging
```

| Service | URL |
| --- | --- |
| MLflow | <http://localhost:5000> |
| Churn API | <http://localhost:8000/docs> |
| Feast feature server | `localhost:6566` |
| Prometheus | <http://localhost:9090> |
| Grafana (dashboard *Churn API*) | <http://localhost:3000> |

`make ps` shows the containers and `make down` stops them. The volumes keep MLflow runs and the registry, so after a restart `make up serve-docker` is enough. `train promote` is only needed on a fresh volume.

## Support assistant

```bash
make rag-ingest                                            # index the help center (embedded Qdrant)
uv run rag search "Handy im Zug geklaut" --lang de         # retrieval only
uv run rag ask "Handy im Zug geklaut, was nun?" --lang de  # needs Ollama with the model
uv run rag serve                                           # API on http://127.0.0.1:8100/docs
make rag-eval                                              # retrieval + safety gates, as in CI
```

See [Support assistant](support-rag/index.md).

## Kubernetes platform

```bash
make cluster-up platform-apply   # k3d cluster, namespaces with budgets, Argo CD; dev/test/prod sync themselves
make argocd-ui                   # prints the admin password, then http://localhost:8081
```

Argo CD then deploys the shared model registry and the three environments from this repository. See [Platform](platform/index.md) for promoting models and images, and `make cluster-down` to delete everything.

## Local environment notes

- **Ports:** 5000 (MLflow), 6379 (Redis), 6566 (Feast), 8000 (churn API), 8100 (assistant API), 9090 (Prometheus), 3000 (Grafana); k3d: 6550 (Kubernetes API), 8080/8443 (ingress), 8081 (Argo CD UI), 5001 (registry port-forward).
- **Docker on Windows:** when the Windows Docker pipe is unavailable, `scripts/compose.sh` and `platform/scripts/k3d.sh` run the commands inside the Rancher Desktop VM with `rdctl shell`.
- **Kubernetes:** every command uses `platform/.kube/telcoai.yaml`; the current kubectl context is never changed, so other clusters on the machine are not touched.
