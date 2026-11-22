# Configuration

Both packages read their settings with pydantic-settings: every field can be set as an environment variable with the package prefix, or in a `.env` file at the repository root. Defaults are tuned for local runs.

```bash
CHURN_MLFLOW_TRACKING_URI=http://localhost:5001 make promote FROM=dev TO=staging
RAG_LLM_MODEL=ollama_chat/qwen2.5 uv run rag serve
```

## Churn

Prefix `CHURN_`, defined in `churn/src/churn/config.py`.

| Variable | Default | Meaning |
| --- | --- | --- |
| `CHURN_DATA_DIR` | `data` | Root for raw data, Feast files and predictions |
| `CHURN_RAW_DATA_URL` | IBM dataset on GitHub | Where `churn download` fetches the CSV |
| `CHURN_MLFLOW_TRACKING_URI` | `http://127.0.0.1:5000` | MLflow tracking server and registry |
| `CHURN_EXPERIMENT_NAME` | `telco-churn` | MLflow experiment for training runs |
| `CHURN_REGISTERED_MODEL_NAME` | `telco-churn` | Registered model name |
| `CHURN_REGISTER_ALIAS` | `dev` | Alias a freshly trained model gets; promotion is a separate step |
| `CHURN_SERVING_ALIAS` | `prod` | Alias the API serves |
| `CHURN_MODEL_REFRESH_SECONDS` | `30` | How often the API checks whether its alias moved; `0` disables the check |
| `CHURN_RANDOM_SEED` | `42` | Seed for the split and training |
| `CHURN_TEST_SIZE` | `0.2` | Held-out test share |
| `CHURN_MIN_ROC_AUC` | `0.80` | Quality gate: below this, a model is never registered |
| `CHURN_FEATURE_REPO` | `churn/feature_repo` | Feast repository |
| `CHURN_FEATURE_SERVER_URL` | unset | Feast feature server for `/predict/by-id` |
| `CHURN_PREDICTION_LOG_DIR` | `data/predictions` | Where scored requests are logged for drift monitoring |
| `CHURN_DRIFT_THRESHOLD` | `0.1` | Distance above which a feature drifts |
| `CHURN_DRIFT_SHARE` | `0.25` | Share of drifted features that makes the dataset drift |
| `CHURN_DRIFT_MIN_ROWS` | `200` | Fewer predictions than this give no verdict |
| `CHURN_DRIFT_WINDOW_DAYS` | `7` | How far back the drift check reads predictions |
| `CHURN_MONITORING_EXPERIMENT_NAME` | `telco-churn-monitoring` | MLflow experiment for drift reports |

## Support assistant

Prefix `RAG_`, defined in `support-rag/src/support_rag/config.py`.

| Variable | Default | Meaning |
| --- | --- | --- |
| `RAG_QDRANT_URL` | unset | Qdrant server; unset means an embedded store at `RAG_QDRANT_PATH` |
| `RAG_QDRANT_PATH` | `data/qdrant` | Embedded Qdrant store |
| `RAG_COLLECTION` | `support_articles` | Qdrant collection |
| `RAG_DENSE_MODEL` | `intfloat/multilingual-e5-large` | Embedding model, chosen by [benchmark](../support-rag/evaluation.md#embedding-model-benchmark-2026-09-29-cpu) |
| `RAG_SPARSE_MODEL` | unset | Set to `Qdrant/bm25` for hybrid search |
| `RAG_CHUNK_MAX_WORDS` | `120` | Chunk size |
| `RAG_TOP_K` | `5` | Chunks returned by search |
| `RAG_PROMPT_VERSION` | latest | Pin an answer prompt version |
| `RAG_LLM_MODEL` | `ollama_chat/granite4.2:8b` | Any LiteLLM model string, e.g. `azure/<deployment>` |
| `RAG_LLM_API_BASE` | `http://127.0.0.1:11434` | Provider endpoint (Ollama by default) |
| `RAG_TEMPERATURE` | `0.0` | Sampling temperature |
| `RAG_REASONING_EFFORT` | `none` | Thinking off |
| `RAG_MAX_TOKENS` | `400` | Answer length limit |
| `RAG_ANSWER_TOP_K` | `4` | Sources given to the model |
| `RAG_MIN_RETRIEVAL_SCORE` | `0.78` | Scope filter: below this, the fallback is returned without an LLM call |
| `RAG_ACTIONS_DB` | `data/actions.db` | SQLite store for proposed actions and their event log |
| `RAG_AGENT_TOKEN` | unset | Shared secret for the agent API; unset disables it |
| `RAG_TRACE_MLFLOW_URI` | unset | MLflow server for request traces; unset disables tracing |
| `RAG_TRACE_EXPERIMENT` | `support-assistant-traces` | MLflow experiment for traces |
| `RAG_JUDGE_MODEL` | `ollama_chat/qwen2.5` | LLM judge, a different family from the generator |
| `RAG_EVAL_MLFLOW_URI` | unset | MLflow server for evaluation runs |
| `RAG_EVAL_EXPERIMENT` | `support-assistant-eval` | MLflow experiment for evaluation runs |
| `RAG_INJECTION_CLASSIFIER` | `support-rag/models/injection_classifier.json` | Learned injection classifier artifact (`rag train-injection`) |
