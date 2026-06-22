# envbert-mcp

MCP server exposing **raw envbert** (DistilBERT EDD classification) to
Claude — standalone.

---

This server calls `envbert.due_diligence.envbert_predict()` directly.

If you need the LLM-fallback behaviour (e.g. building a non-LLM pipeline),
use [`envbert-agent`](https://pypi.org/project/envbert-agent/).

---

## Architecture

```
Claude / Claude Code
        │ MCP (stdio)
        ▼
envbert_mcp/server.py     ←── this file, single process
        │ in-process call
        ▼
envbert.due_diligence.envbert_predict()
        │
        ▼
DistilBERT (d4data/environmental-due-diligence-model)
```

Everything runs in one process. The model loads once at startup
(warmup, ~10-40s on a cold HuggingFace cache) and stays in memory for
the life of the server.

---

## Tools

| Tool | Purpose |
|---|---|
| `check_envbert_status` | Confirm the model is loaded before relying on fast responses — the very first call in a session may be slow while warmup is still in progress |
| `classify_environmental_text` | Classify one sentence/paragraph — label + confidence, no LLM step |
| `classify_environmental_document` | Classify all paragraphs of a document concurrently, with category distribution and low-confidence flagging |

> **Note on `confidence`:** this is a cosine similarity between the input
> text and a reference embedding for the predicted category — not a
> calibrated probability. It's a useful relative signal (higher = closer
> match) but shouldn't be read as "X% likely correct." Scores at or below
> 0.3 are already relabeled `Not Relevant` internally before they reach
> you. Every response also includes a `score_basis` field as a reminder.

### Why low-confidence flagging matters here

Without an LLM fallback, a low envbert confidence score (e.g. 0.42) is the
final answer — there's no second opinion baked in. `classify_environmental_document`
surfaces a `low_confidence_items` list explicitly so Claude can apply its
own judgement to exactly those paragraphs, rather than treating every
result as equally reliable.

Note that `confidence` here is a cosine similarity to a reference
embedding, not a calibrated probability — see the caveat under
[Tools](#tools) above.

```json
{
  "category_distribution": {"Geology": 4, "Contaminants": 2},
  "score_basis": "cosine_similarity_not_probability",
  "low_confidence_count": 1,
  "low_confidence_items": [
    {"index": 3, "label": "Remediation Standards", "confidence": 0.42}
  ],
  "results": [ ... ]
}
```

---

## Quickstart

```bash
pip install envbert envbert-mcp
```

Register the server with Claude Code:

```bash
claude mcp add envbert -- envbert-mcp
```

(Add `--scope user` instead of the default local scope if you want it
available in every project, not just the one you ran this from.)

Restart Claude Code. The model warms up in the background on first
launch — `check_envbert_status` will report `"loading": true` until ready.

**Example prompts:**
- *"Is envbert ready?"*
- *"Classify this: 'weathered shale was encountered below the surface with fluvial deposits'"*
- *"Here's a 12-paragraph site report — classify each section and flag anything you're not confident about."*

---

## envbert-mcp vs envbert-agent which to use

| | This package (raw envbert) | envbert-agent  |
|---|---|---|
| Used by | Claude / MCP clients | CLI, pipelines, non-LLM consumers |
| LLM fallback | None | Yes — Ollama (local) or Azure |
| Typical latency | <1s always | <1s confident, ~10s on fallback |
| External dependencies | None beyond envbert | Ollama or Azure OpenAI |
| Confidence on ambiguous text | Raw model score only | LLM-resolved final label |
| Why this shape | Claude can reason over raw scores itself — a second hidden LLM call is redundant | No reasoning layer downstream; the agent must resolve ambiguity itself |

Both are legitimate — they're solving for different consumers of the
classification, not competing implementations of the same thing.

---

## Configuration

| Variable | Default | Description |
|---|---|---|
| `LOG_LEVEL` | `INFO` | Logging verbosity |

No other configuration needed — there's no backend URL, no LLM provider,
no API keys. That's the point.

---

## License

MIT
