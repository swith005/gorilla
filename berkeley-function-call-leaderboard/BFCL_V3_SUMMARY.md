# BFCL v3 via RITS — Changes Summary

## Goal

Run the Berkeley Function Calling Leaderboard v3 (BFCL v3) against `ibm-granite/granite-4.0-h-micro` hosted on RITS, from a Mac with no GPU. This is a standalone test using the BFCL repo directly (not integrated into Sage).

## Supported Inference Backends

BFCL v3 supports the following runtimes:
- **OpenAI** (and OpenAI-compatible APIs)
- **Anthropic**
- **Mistral**
- **Google (Gemini)**
- **Cohere**
- **Local/OSS** (via vLLM or any OpenAI-compatible server)

We use the **Local/OSS** handler (`base_oss_handler.py`) with `--skip-server-setup`, pointing it at RITS as a remote OpenAI-compatible endpoint.

## What Was Changed

### BFCL v3 Repo (2 files + 1 config)

| File | Change | Why |
|---|---|---|
| `bfcl_eval/constants/model_config.py` | Registered `ibm-granite/granite-4.0-h-micro` in `MODEL_CONFIG_MAPPING` with `Granite4FCHandler` and `is_fc_model=True` | Model must be registered before BFCL can generate/evaluate against it |
| `bfcl_eval/model_handler/local_inference/base_oss_handler.py` | Added `import json`; modified OpenAI client creation to support `OPENAI_DEFAULT_HEADERS` env var; modified server readiness check to include auth headers | RITS requires a custom `RITS_API_KEY` header instead of standard OpenAI `Authorization: Bearer` header |
| `.env` (local config, not committed) | Set `REMOTE_OPENAI_BASE_URL`, `REMOTE_OPENAI_API_KEY`, and `OPENAI_DEFAULT_HEADERS` | Configures the OSS handler to connect to RITS |

### Detailed Changes

**Model Registration** (`model_config.py`):
```python
"ibm-granite/granite-4.0-h-micro": ModelConfig(
    model_name="ibm-granite/granite-4.0-h-micro",
    display_name="Granite-4.0-H-Micro (FC)",
    url="https://huggingface.co/ibm-granite/granite-4.0-h-micro",
    org="IBM",
    license="Apache-2.0",
    model_handler=Granite4FCHandler,
    input_price=None,
    output_price=None,
    is_fc_model=True,
    underscore_to_dot=False,
),
```

**RITS Header Support** (`base_oss_handler.py` — client creation):
```python
client_kwargs = dict(base_url=self.base_url, api_key=self.api_key)
if headers_env := os.getenv("OPENAI_DEFAULT_HEADERS"):
    client_kwargs["default_headers"] = json.loads(headers_env)
self.client = OpenAI(**client_kwargs)
```

**RITS Header Support** (`base_oss_handler.py` — server readiness check):
```python
readiness_headers = {}
if self.api_key and self.api_key != "EMPTY":
    readiness_headers["Authorization"] = f"Bearer {self.api_key}"
if headers_env := os.getenv("OPENAI_DEFAULT_HEADERS"):
    readiness_headers.update(json.loads(headers_env))
response = requests.get(f"{self.base_url}/models", headers=readiness_headers)
```

**Environment Config** (`.env`):
```
REMOTE_OPENAI_BASE_URL=https://inference-3scale-apicast-production.apps.rits.fmaas.res.ibm.com/granite-4-h-micro/v1
REMOTE_OPENAI_API_KEY=<your-rits-api-key>
OPENAI_DEFAULT_HEADERS={"RITS_API_KEY": "<your-rits-api-key>"}
```

## Available Test Categories

### Non-Live (Synthetic)

| Category | Description |
|---|---|
| `simple` | Simple function calls (Python, Java, JavaScript) |
| `simple_python` | Python simple function calls (400 entries) |
| `simple_java` | Java simple function calls |
| `simple_javascript` | JavaScript simple function calls |
| `multiple` | Multiple function calls |
| `parallel` | Parallel function calls |
| `parallel_multiple` | Parallel + multiple function calls |
| `irrelevance` | Irrelevance detection |

### Live (Real-world APIs)

| Category | Description |
|---|---|
| `live_simple` | Real-world simple calls |
| `live_multiple` | Real-world multiple calls |
| `live_parallel` | Real-world parallel calls |
| `live_parallel_multiple` | Real-world parallel + multiple calls |
| `live_irrelevance` | Real-world irrelevance detection |
| `live_relevance` | Real-world relevance detection |

### Multi-Turn

| Category | Description |
|---|---|
| `multi_turn_base` | Base multi-turn conversations |
| `multi_turn_miss_func` | Missing function scenarios |
| `multi_turn_miss_param` | Missing parameter scenarios |
| `multi_turn_long_context` | Long context handling |

### Agentic

| Category | Description |
|---|---|
| `web_search_summary` | Web search with summary |
| `web_search_base` | Web search base |
| `web_search_no_snippet` | Web search without snippets |
| `memory_kv` | Key-value memory |
| `memory_vector` | Vector memory |
| `memory_recursive_summarization` | Recursive summarization memory |

## Test Commands

### Install
```bash
git clone https://github.com/swith005/gorilla.git
cd gorilla/berkeley-function-call-leaderboard
pip install -e .
pip install soundfile  # required dependency not in setup.py
```

### Generate Responses
```bash
bfcl generate --model ibm-granite/granite-4.0-h-micro --test-category simple_java --skip-server-setup --temperature 0.001
```

### Evaluate Results
```bash
bfcl evaluate --model ibm-granite/granite-4.0-h-micro --test-category simple_java
```

### View Scores
```bash
bfcl scores --model ibm-granite/granite-4.0-h-micro --test-category simple_java
```

## Results

### Non-Live: Java Simple AST

| Category | Accuracy |
|---|---|
| Java Simple AST | 96.25% |

Other categories (Python, JavaScript, Multiple, Parallel, etc.) have not been run yet. Scores CSV files are generated in `score/data_*.csv`.

## Issues Encountered and Resolved

| Issue | Root Cause | Fix |
|---|---|---|
| `ModuleNotFoundError: No module named 'soundfile'` | Missing dependency not listed in setup.py | `pip install soundfile` |
| `zsh: command not found: --model` | Multi-line command split incorrectly in terminal | Run as single-line command |
| RITS auth failure on server readiness check | `GET /v1/models` requires RITS_API_KEY header | Added auth headers to readiness check in `base_oss_handler.py` |
| Model not found in BFCL | `ibm-granite/granite-4.0-h-micro` not registered | Added to `MODEL_CONFIG_MAPPING` in `model_config.py` |

## Notes

- **Not integrated into Sage** — BFCL v3 is run standalone via `bfcl` CLI, not through `sage` commands
- **Tokenizer**: The OSS handler loads the tokenizer from HuggingFace. If the model is gated, `HF_TOKEN` must be set or use `REMOTE_OPENAI_TOKENIZER_PATH` for a local tokenizer
- **Function calling support**: `granite-4.0-h-micro` supports function calling as confirmed on its HuggingFace model card (capabilities include "Function-calling tasks")
- **Granite4FCHandler**: Uses `<tool_call>` XML format for function call parsing, which is the native format for Granite 4.0 models
