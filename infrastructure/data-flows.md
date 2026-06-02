# External Data Flows

> Documented 2026-06-02 — what data leaves this system, where it goes, and why.

Memory OS sends data to two external services: **OpenRouter** (dense embeddings + LLM extraction) and **Together AI** (fine-tuning training data + replacement model inference). No telemetry, analytics, or tracking data is sent to any third party.

---

## Flow 1: OpenRouter — Dense Embeddings

**Purpose:** Generate vector embeddings for semantic search in Qdrant.

**Source files:**
- `docker/worker/services/embedding.py` — ARQ worker embedding calls
- `scripts/context_enhancer.py` — `embed_query()` used by Icarus pre_llm_call hook

**Endpoint:** `POST https://openrouter.ai/api/v1/embeddings`

**Model:** `qwen/qwen3-embedding-8b`

**Data sent:**
| Field | Content | Truncation |
|---|---|---|
| `input` | User's query/message text | 8000 chars |
| `model` | `qwen/qwen3-embedding-8b` | — |
| `dimensions` | `EMBEDDING_DIMS` (default: 4096) | — |

**When it fires:**
- ARQ worker: during wiki ingestion pipeline (every new wiki file at hourly cron)
- Icarus hook: on every user message that passes the social-closer gate, as part of automatic context injection (pre_llm_call → `_search_qdrant()`)

**Auth:** `OPENROUTER_API_KEY` via `Authorization: Bearer` header.

**Persistence:** The returned embedding vector is stored in Qdrant's `knowledge_base` collection. The plaintext is **not** persisted by OpenRouter (standard embedding API — no conversation logging).

**Fallback:** If embedding fails, the system returns empty results (fail-open). No data is cached or retried.

---

## Flow 2: OpenRouter — Session Extraction (LLM)

**Purpose:** Extract structured knowledge entries (decisions, resolutions, notes) from session transcripts.

**Source file:** `icarus/hooks.py` → `_llm_extract_entries()`

**Endpoint:** `POST https://openrouter.ai/api/v1/chat/completions`

**Model:** `deepseek/deepseek-v4-flash` (configurable via `ICARUS_EXTRACTION_MODEL`)

**Data sent:**
| Field | Content | Limits |
|---|---|---|
| `messages[0]` (system) | Extraction prompt instructing the LLM to identify significant entries | — |
| `messages[1]` (user) | Formatted session transcript: `[Turn N — User/Agent]` entries | 8000 chars total |
| `max_tokens` | Configurable via `ICARUS_EXTRACTION_MAX_TOKENS` (default 1024, recommended 4096) | — |
| `temperature` | 0.2 (low randomness — deterministic extraction) | — |

**The session transcript contains:**
- Each exchange's user message (truncated to 500 chars)
- Each exchange's assistant response (truncated to 800 chars)
- Timestamps, tool outputs (included in the assistant response field)

**When it fires:** `on_session_end` hook — after every session that has accumulated 2+ exchanges.

**Auth:** `OPENROUTER_FULL_API_KEY` → `OPENROUTER_DS_API_KEY` → `OPENROUTER_API_KEY` (fallback chain). Resolved at `icarus/hooks.py` line 15-19.

**Persistence:** The LLM result (`extracted_entries` JSON) is written to `$FABRIC_DIR/` as markdown files with YAML frontmatter. The raw transcript passed to the API is not persisted — only the extracted entries survive.

**Fallback:** If the LLM call fails or returns garbage, the system falls back to legacy truncation (`_legacy_session_write`) which simply copies the first user message + last decision response.

---

## Flow 3: Together AI — Training Data Upload

**Purpose:** Upload exported training pairs to Together AI for fine-tuning.

**Source file:** `icarus/state.py` → `start_training()` (lines 714-738)

**Endpoint:** `POST https://api.together.xyz/v1/files/upload`

**Format:** Multipart form-data (`multipart/form-data`)

**Data sent:**
| Part | Content |
|---|---|
| `purpose` | `fine-tune` |
| `file` | `training.jsonl` — JSONL in Together AI format (messages with system prompt) |

**Each training pair is a JSON object:**
```json
{
  "messages": [
    {"role": "system", "content": "You are a helpful AI agent with shared memory across platforms."},
    {"role": "user", "content": "[decision] What did you decide?"},
    {"role": "assistant", "content": "..."}
  ]
}
```

**Source of training pairs:** Fabric entries (`$FABRIC_DIR/*.md`) parsed by `icarus/export-training.py`. The export script:
1. Scans all fabric entries (hot + cold)
2. Filters by quality mode (high-precision / normal / high-volume)
3. Extracts pairs: outcomes, structured sessions, decisions, reviews, cross-platform entries
4. Weights high-value pairs (verified, cross-agent, structured sessions get higher duplication)

**When it fires:** When the agent calls `fabric_train()` tool, triggered by user request.

**Auth:** `TOGETHER_API_KEY` via `Authorization: Bearer` header (resolved from env or `~/.hermes/.env`).

**Persistence:** The uploaded file becomes a Together AI file ID, which is used in the subsequent fine-tuning job. The training JSONL is ephemeral (written to temp directory `export_training()` → `tempfile.TemporaryDirectory()`, discarded after upload). The raw pairs are not persisted locally beyond the temp directory.

**Minimum pair threshold:** Default 10 pairs. The system tries `high-precision` → `normal` → `high-volume` in order, picking the highest-quality mode that meets the threshold.

---

## Flow 4: Together AI — Fine-Tuning Job

**Purpose:** Kick off a fine-tuning job on Together AI after the training file is uploaded.

**Source file:** `icarus/state.py` → `start_training()` (continuation, lines ~740-780)

**Endpoint:** `POST https://api.together.xyz/v1/fine-tunes`

**Data sent:**
```json
{
  "training_file": "<uploaded_file_id>",
  "model": "Qwen/Qwen2-7B-Instruct",
  "wandb_project": "memory-os",
  "n_epochs": 3,
  "n_checkpoints": 1,
  "batch_size": 8,
  "learning_rate": 1e-5,
  "suffix": "<agent-name-or-custom-suffix>"
}
```

**When it fires:** Immediately after a successful file upload (same `fabric_train()` call).

**Auth:** `TOGETHER_API_KEY`

**Persistence:** The job ID is saved to `$HERMES_HOME/.icarus-training-job.txt`. The resulting fine-tuned model ID is registered in `$HERMES_HOME/.icarus-models.json` once `fabric_train_status()` detects completion.

**Monitoring:** `fabric_train_status()` polls `GET https://api.together.xyz/v1/fine-tunes/{job_id}` to check progress.

---

## Flow 5: Together AI — Model Inference (Eval)

**Purpose:** Compare a candidate replacement model against the current model using eval prompts derived from fabric entries.

**Source file:** `icarus/scripts/eval-replacement.py`

**Endpoint:** Together AI's OpenAI-compatible chat endpoint  
`POST https://api.together.xyz/v1/chat/completions`

**Data sent:** Eval prompts extracted from high-value fabric entries. Each eval turn sends the prompt to both the candidate and base model, and compares responses.

**When it fires:** When the agent calls `fabric_eval(candidate_model="...")`, typically before switching to a replacement model.

**Auth:** `TOGETHER_API_KEY`

**Persistence:** Eval results (scores per prompt) are saved to the model registry (`$HERMES_HOME/.icarus-models.json`), which enables the eval gate on `fabric_switch_model()`.

---

## Data Excluded

The following are **not** sent to any external service:

| Data | Why |
|---|---|
| Raw user credentials / API keys | Only used locally for authenticating the external calls themselves |
| File contents from `$VAULT_PATH/wiki/` | Only embeddings (vectors) are sent to OpenRouter; the plain text stays local |
| System configuration | Environment variables with secrets are used for auth headers only, not sent as payload |
| Qdrant documents / payloads | Vector search is local; Qdrant never sends data externally |
| Tool outputs / intermediate results | Only the assistant response text (as visible to the user) is included in session extraction |

## API Keys Summary

| Key | Used by | Scope |
|---|---|---|
| `OPENROUTER_API_KEY` | Embedding pipeline, Docker worker | Embedding + full API access |
| `OPENROUTER_FULL_API_KEY` | Icarus hooks (preferred) | LLM extraction (fallback chain) |
| `OPENROUTER_DS_API_KEY` | Icarus hooks (fallback) | LLM extraction (fallback chain) |
| `TOGETHER_API_KEY` | Icarus training/eval | File upload, fine-tuning, model eval |

All keys are read from environment (`.env` file) and never written to disk except via the user's own `.env` management.