# Semantic Caching

A small toolkit for experimenting with **semantic cache** strategies for FAQ-style question answering.  
The repo includes:

- a Redis-backed semantic cache wrapper
- a lightweight fuzzy string baseline
- optional rerankers using a cross-encoder or an LLM
- evaluation helpers for hit rate, precision, recall, F1, and threshold sweeps
- sample datasets and a notebook for running experiments

## What This Repo Does

The main idea is to store question/answer pairs in a cache and retrieve the closest answer for a new query.
Instead of exact string matching, the semantic cache uses embeddings to find similar questions.

You can also:

- hydrate the cache from a `pandas.DataFrame` or a list of pairs
- register a reranker to refine cache hits
- compare cache behavior across thresholds
- evaluate cache quality against labeled test data

## Project Layout

- `utils/cache/wrapper.py` — main `SemanticCacheWrapper` abstraction
- `utils/cache/fuzzy_cache.py` — simple fuzzy matching baseline
- `utils/cache/cross_encoder.py` — cross-encoder scoring and reranker
- `utils/cache/llm_evaluator.py` — LLM-based similarity checker and reranker
- `utils/cache/evals.py` — evaluation and reporting helpers
- `utils/cache/vis.py` — plots and confusion-matrix styling
- `utils/cache/config.py` — environment-driven cache configuration
- `utils/data/faq_seed.csv` — seed FAQ data
- `utils/data/test_dataset.csv` — test/evaluation dataset
- `02_evaluate_cache.ipynb` — notebook for end-to-end evaluation

## Requirements

Install dependencies with:

```bash
pip install -r requirements.txt
```

## Setup

1. Start Redis locally.
2. Create a `.env` file.
3. Add the variables you need.

Example `.env`:

```env
REDIS_URL=redis://localhost:6377
CACHE_NAME=semantic-cache
CACHE_DISTANCE_THRESHOLD=0.3
CACHE_TTL_SECONDS=3600
OPENAI_API_KEY=your_key_here
```

Notes:

- `REDIS_URL` defaults to `redis://localhost:6377` in `utils/cache/config.py`.
- `OPENAI_API_KEY` is needed for GPT-based evaluation/reranking.
- If you use Gemini-based evaluation, configure the corresponding Google credentials as well.

## Quick Start

### 1) Create a cache

```python
from utils.cache.wrapper import SemanticCacheWrapper

cache = SemanticCacheWrapper()
```

You can also build it from the repo config:

```python
from utils.cache.config import config
from utils.cache.wrapper import SemanticCacheWrapper

cache = SemanticCacheWrapper.from_config(config)
```

### 2) Load FAQ pairs

```python
import pandas as pd

faq_df = pd.read_csv("utils/data/faq_seed.csv")
cache.hydrate_from_df(faq_df, q_col="question", a_col="answer")
```

You can also hydrate from pairs:

```python
pairs = [
    ("How do I reset my password?", "Use the reset password link."),
    ("Where is my order?", "Check your order history page."),
]
cache.hydrate_from_pairs(pairs)
```

### 3) Query the cache

```python
result = cache.check("How can I change my password?")

if result.matches:
    top_match = result.matches[0]
    print(top_match.prompt)
    print(top_match.response)
```

Batch queries are supported too:

```python
queries = [
    "How do I reset my password?",
    "Where can I see my order status?",
]
results = cache.check_many(queries)
```

## Rerankers

The wrapper supports optional reranking after the initial semantic retrieval.

### Cross-encoder reranking

```python
from utils.cache.cross_encoder import CrossEncoder

cross_encoder = CrossEncoder()
cache.register_reranker(cross_encoder.create_reranker())
```

### LLM reranking

```python
from utils.cache.llm_evaluator import LLMEvaluator

evaluator = LLMEvaluator.construct_with_gpt()
cache.register_reranker(evaluator.create_reranker())
```

To disable reranking:

```python
cache.clear_reranker()
```

## Fuzzy Baseline

If you want a simpler string-similarity baseline:

```python
from utils.cache.fuzzy_cache import FuzzyCache

fuzzy = FuzzyCache()
fuzzy.hydrate_from_df(faq_df)
results = fuzzy.check_many(["How do I reset my password?"])
```

## Evaluation

`utils/cache/evals.py` provides helpers to:

- compute cache hit rate, precision, recall, F1, and accuracy
- sweep distance thresholds
- visualize the results with `panel` and `matplotlib`

The notebook `02_evaluate_cache.ipynb` shows a full workflow:

1. load FAQ data
2. hydrate the cache
3. run test queries
4. evaluate threshold behavior
5. compare retrieval strategies

## Configuration

Environment-driven settings live in `utils/cache/config.py`:

- `REDIS_URL`
- `CACHE_NAME`
- `CACHE_DISTANCE_THRESHOLD`
- `CACHE_TTL_SECONDS`

## Data Files

The repo includes sample data under `utils/data/`:

- `faq_seed.csv` — seed FAQ entries for the cache
- `test_dataset.csv` — evaluation queries and labels
- PDF documents used for experimentation and policy reference

## Troubleshooting

- If Redis is unavailable, `SemanticCacheWrapper` will fail fast when connecting.
- If embeddings or rerankers are slow, start with the fuzzy baseline or reduce batch sizes.
- If you use an LLM reranker, make sure the required API key is present in the environment.

## License

No license file is included in this repo. Add one if you plan to share or publish the project.
