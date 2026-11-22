# Evaluation

Three gates protect the assistant. Retrieval and safety run in CI on every change; answer quality needs local LLMs, so it runs before merging a prompt or model change.

| Gate | Command | Runs in |
| --- | --- | --- |
| Retrieval | `rag eval-retrieval --gate` | CI (`support-rag-ci`) |
| Safety | `rag eval-safety --gate` | CI ([Guardrails](guardrails.md#ci-gate)) |
| Answer quality and actions | `make rag-eval-gen` | Locally, before prompt or model changes |

## Retrieval

`eval/retrieval_golden.yaml` holds **94 questions** (about 23 per language) phrased the way customers ask, including 10 deliberately confusable ones. Examples: slow mobile data at home versus while roaming; cancelling versus porting a number; the cost of calling the US from Switzerland versus calling from the US.

A question counts as a hit at rank *r* when the *r*-th retrieved chunk belongs to the expected topic. The harness reports:

- hit@1/3/5 and MRR@10 overall and per language, with the UI language known (the production path);
- the same metrics without the language filter, to show cross-lingual robustness;
- latency;
- every miss.

### Embedding model benchmark (2026-09-29, CPU)

| Dense model | Search | hit@1 | hit@3 | MRR | DE / FR / IT / EN hit@3 | No lang filter hit@3 | ms/query |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **multilingual-e5-large** | dense | **0.872** | **0.979** | **0.927** | 0.96 / 1.00 / 1.00 / 0.96 | **0.979** | 53 |
| multilingual-e5-large | hybrid (+BM25) | 0.830 | 0.957 | 0.901 | 0.96 / 0.96 / 0.96 / 0.96 | 0.968 | 59 |
| jina-embeddings-v3 | dense | 0.840 | 0.968 | 0.900 | 0.96 / 1.00 / 0.96 / 0.96 | 0.872 | 1090 |
| paraphrase-multilingual-MiniLM-L12-v2 | hybrid (+BM25) | 0.777 | 0.904 | 0.848 | 0.92 / 0.87 / 0.96 / 0.88 | 0.883 | 11 |
| paraphrase-multilingual-MiniLM-L12-v2 | dense | 0.702 | 0.883 | 0.802 | 0.83 / 0.87 / 0.91 / 0.92 | 0.809 | 6 |
| potion-multilingual-128M (static) | dense | 0.596 | 0.862 | 0.725 | 0.83 / 0.83 / 0.91 / 0.88 | 0.723 | 1 |

**Decision:** multilingual-e5-large, dense-only.

- **Best quality:** best on every metric, and the most robust when the language is unknown.
- **Fast enough:** about 50 ms per query on CPU.
- **Why BM25 is off:** hybrid search *helps* the weak model but *hurts* e5. fastembed's BM25 uses English stemming, which adds noise on DE/FR/IT text. A language-specific keyword index would be the next thing to try.

The two remaining misses are genuinely ambiguous: "blocked internet in Thailand" returns the outage article before roaming, and "line blocked although I pay by eBill" returns third-party blocking before late payment. They stay in the golden set rather than being tuned away.

### CI gate

The `support-rag-ci` workflow runs the unit tests (with a hashing encoder, so no model download is needed), then **`rag eval-retrieval --gate`** with the real model. The model is cached between runs. The gate fails the PR when a change (model, chunking, corpus edit) drops below `eval/thresholds.yaml`:

- hit@1 ≥ 0.80, hit@3 ≥ 0.95 and MRR ≥ 0.88 overall;
- hit@3 ≥ 0.90 in every language.

The JSON report is uploaded as a build artifact.

## Answer quality

`rag eval-generation` runs the full assistant on `eval/generation_golden.yaml`: 24 answerable questions (6 per language), each with the fact the answer must contain, plus 8 the help center doesn't cover.

**Deterministic checks:**

- **Answered:** the question got an answer rather than a fallback.
- **Facts:** the expected fact is present. Accepted spellings are allowed, e.g. `3.50|3,50`.
- **Style:** right language, no repeated sentences, no echoed instructions or meta-talk (DE/FR/IT/EN).
- **Declines:** questions outside the help center are declined.

**LLM judge** (`prompts/judge/v2.yaml`): is every claim supported by the cited sources (*faithful*), and does the answer address the question (*relevant*)? The judge is a different model family from the generator, so no model grades its own answers.

Every run is logged to MLflow (`RAG_EVAL_MLFLOW_URI`), with the model, prompt version and hash, judge, metrics and a per-question table. That makes model and prompt comparisons (A/B) a side-by-side view of runs.

### Evaluating the evaluator

Before its verdicts count, the judge is scored on `eval/judge_calibration.yaml`: 14 answers with known labels, including correct answers, invented prices, an invented extra claim, echoed instructions, a duplicated answer, meta-talk, wrong language and answers to the wrong question.

| Criterion | qwen2.5 judge (`judge@v1`) | After the split (`judge@v2` + style check) |
| --- | --- | --- |
| faithful | 93% | 93% (judge) |
| relevant | 93% | 93% (judge) |
| clean (style) | **71%**: it called an English answer "German" and missed a duplicated answer | **100%** (deterministic check) |

A 7B judge is reliable for faithfulness and relevance but not for style. So style moved to deterministic checks, and the judge only rates the two criteria it handles well. Because the style check's 100% is on the labeled set it was written against, it was validated on unseen output too: it flagged the real repetitions and meta-talk Granite 3B produced on questions it had never seen (below).

### Results (2026-09-29, RTX 3060 Laptop GPU with 6 GB)

| Generator · prompt (judge) | Answered | Facts | Style | Declines | Faithful | Relevant | **Quality pass** | Median latency |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **Granite 4.2 8B** · `answer@v3` (qwen2.5), default | 100% | 100% | 100% | 88% | 96% | 100% | **96%** | 4.2 s |
| Granite 4.2 8B · `answer@v2` (qwen2.5) | 100% | 100% | 100% | 88% | 96% | 96% | **92%** | 4.3 s |
| Granite 4.2 8B · `answer@v1` (qwen2.5) | 100% | 100% | 100% | 88% | 96% | 96% | **92%** | 4.9 s |
| qwen2.5 7B · `answer@v2` (Granite 8B) | 100% | 100% | 100% | 100% | 96% | 96% | **96%** | 1.6 s |
| Granite 4.2 3B · `answer@v1` (qwen2.5) | 83% | 83% | 80% | 100% | 95% | 95% | **62.5%** | 0.8 s |
| Granite 4.2 3B · `answer@v2` (qwen2.5) | 67% | 62.5% | 81% | 100% | 100% | 94% | **46%** | 0.8 s |

*Quality pass* is the share of answerable questions that got a fully good answer: right facts, clean style, and judged faithful and relevant.

- **Scale matters for the 16/16 result.** The 3B model had scored 16/16 on the earlier 16-case fact check. On the larger set it wrongly says "not found" for a third of answerable questions, and its answers repeat themselves or talk about their sources.
- **Prompt A/B.** The canary line added in `answer@v2` (leak detection) costs Granite 8B nothing (92% on both versions) but costs the 3B model 17 points of quality. `answer@v2` stays, and switching to a small model requires re-running this comparison.
- **Model choice.** qwen2.5 is as good as Granite 8B here and 2.7× faster. The difference is one decline out of 8, too small a sample to call. Granite 8B remains the default (newest model, above every threshold); qwen2.5 is a config switch away.
- **Judge-only failures** (e.g. "Wi-Fi extender costs CHF 99 [1]" judged *not relevant*) match the judge's calibrated ~93% accuracy. That's why the thresholds keep a margin.

### Gate

`make rag-eval-gen` (calibration, then `rag eval-generation --gate` and `rag eval-actions --gate`) fails below `eval/thresholds.yaml`, which requires quality pass ≥ 80%, faithful and relevant ≥ 85%, and facts, style and answered ≥ 90%. It needs the local LLMs, so it runs before merging a prompt or model change, not on GitHub's CPU runners. The unit tests cover its logic with fake models in CI.
