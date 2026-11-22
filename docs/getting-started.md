# Get started

Everything runs on a laptop. The cloud is optional.

## Requirements

| Tool | Needed for |
| --- | --- |
| [uv](https://docs.astral.sh/uv/), `make`, `git` | Everything: uv installs Python 3.14 and every package from `uv.lock` |
| A Docker engine | The full stack in Docker and the Kubernetes platform |
| [Ollama](https://ollama.com/) ≥ 0.34 with `granite4.2:8b` | Answers from the support assistant (retrieval and its gates work without it); `qwen2.5` too for the LLM judge |
| k3d, Terraform, Helm, kubectl | The Kubernetes platform |

=== "macOS"

    ```bash
    xcode-select --install                     # make and git (Command Line Tools)
    brew install uv k3d helm kubectl ollama
    brew install hashicorp/tap/terraform       # Terraform is no longer in Homebrew core
    ollama pull granite4.2:8b && ollama pull qwen2.5
    ```

    Docker engine: Docker Desktop, Rancher Desktop, OrbStack or Colima all provide the `docker` CLI and Compose that the Makefile uses.

    !!! warning "Not tested on macOS yet"
        The project is developed and tested on Windows 11. The scripts are POSIX `sh` and avoid GNU-only flags, so they should work on macOS, but nobody has run them there yet. On **Apple silicon**, see the note under [Kubernetes platform](#kubernetes-platform).

=== "Windows"

    Run the commands in Git Bash (`make` and the scripts are POSIX shell).

    ```bash
    scoop install uv make k3d terraform helm kubectl
    ollama pull granite4.2:8b && ollama pull qwen2.5   # after installing Ollama from ollama.com
    ```

    Docker engine: Docker Desktop or Rancher Desktop (tested with Rancher Desktop).

Then, in the repository:

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

!!! warning "Apple silicon"
    CI publishes the churn API image (`ghcr.io/bllsdclsz/telco-churn-api`) for **amd64 only**. The cluster pulls that image, so on an Apple silicon Mac the churn API pods will likely fail to start unless your Docker engine runs amd64 images through emulation. The other images (MLflow, Argo CD, k3s) are multi-arch. Docker Compose is not affected: `make serve-docker` builds the image locally for your CPU.

## Local environment notes

- **Ports:** 5000 (MLflow), 6379 (Redis), 6566 (Feast), 8000 (churn API), 8100 (assistant API), 9090 (Prometheus), 3000 (Grafana); k3d: 6550 (Kubernetes API), 8080/8443 (ingress), 8081 (Argo CD UI), 5001 (registry port-forward). On macOS, AirPlay Receiver can hold port 5000; turn it off in System Settings → General → AirDrop & Handoff, or change the MLflow port.
- **Docker:** the scripts use the `docker` CLI directly. Only on Windows, when the Docker pipe is unavailable, `scripts/compose.sh` and `platform/scripts/k3d.sh` fall back to running the commands inside the Rancher Desktop VM with `rdctl shell`.
- **Kubernetes:** every command uses `platform/.kube/telcoai.yaml`; the current kubectl context is never changed, so other clusters on the machine are not touched.
