# Platform: Kubernetes, Terraform, GitOps

Local Kubernetes (k3d), Terraform and Argo CD: dev/test/prod with budgets, one shared model registry, image promotion by PR with CI-enforced rules.

**Documentation:** [https://bllsdclsz.github.io/mlops-llmops-platform/platform/](https://bllsdclsz.github.io/mlops-llmops-platform/platform/) (source: [`docs/platform/`](../docs/platform/))

```bash
make cluster-up platform-apply argocd-ui
```
