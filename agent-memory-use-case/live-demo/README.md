# Live Demo — Flat History vs. Oracle Agent Memory

Two runnable demos for the "Memory Illusion" video. **Identical scripted
12-turn conversation** ([conversation.py](conversation.py)), two ways of carrying state:

| | Demo 1 — flat history | Demo 2 — Oracle Agent Memory |
|---|---|---|
| What's sent per turn | the **entire transcript**, re-uploaded | a bounded **context card** + the new question |
| Token bar on screen | grows every turn (red) | stays flat & low (green) |
| Where history lives | inside the prompt | in **Oracle AI Database** (searchable rows) |

Everything runs 100% local: Ollama (`llama3.1:8b` + `nomic-embed-text`),
Oracle AI Database Free container (`oracle26ai`, reused from the RAG course),
and the official [`oracleagentmemory`](https://pypi.org/project/oracleagentmemory/) PyPI package.

## One-time setup

```bash
# 1. Oracle DB container + memdemo user (idempotent)
bash setup_db.sh

# 2. Ollama models
ollama pull llama3.1:8b
ollama pull nomic-embed-text

# 3. Python env (needs 3.11+; litellm requirement)
python3.12 -m venv .venv
./.venv/bin/pip install oracleagentmemory rich matplotlib requests
```

## Recording flow

```bash
# Act 1 — the problem: watch the red bar run away
./.venv/bin/python demo1_flat_history.py

# Act 2 — the fix: green bar stays flat; ends by printing the
# extracted memories ("Memory updated", made visible)
./.venv/bin/python demo2_agent_memory.py

# Act 3 — proof it's a database: raw SQL over the memory tables
./.venv/bin/python show_memories.py

# Act 4 — the money chart (your own local Figure 5)
./.venv/bin/python plot_comparison.py && open token_comparison.png
```

## Talking points wired into the code

- Both demos use the **same model, same script, same max_tokens** — the only
  variable is how conversation state reaches the model.
- Token counts use the **chars ÷ 4** estimate — the same convention as the
  Oracle Agent Memory technical report (arXiv:2607.13157, Fig. 5), so your
  local curve is directly comparable to the paper's 13,900 vs 1,300 result.
- Demo 2's turns 11–12 ("what are our dietary restrictions?", "which seat?")
  are answered from **retrieved memory**, not from a replayed transcript —
  that's the recall proof.
- `show_memories.py` runs plain `SELECT`s — agent memory is literally rows,
  scoped by `user_id` / `agent_id` / `thread_id`.

## Known limitation (worth saying on camera)

The **token story is deterministic and stable** — every run reproduces the
flat red curve vs. the flat green plateau (~2.7× on the final turn, and the
gap keeps widening with conversation length).

**Extraction quality is not** — `llama3.1:8b` is a weak extractor. It reliably
captures the vegetarian diet and the window-seat preference, but details like
"Ankita is allergic to peanuts" are sometimes mangled or missed, and you may
see an occasional `invalid structured output` warning. This is not a demo bug;
it is an honest property of the system: memory extraction is only as good as
the extraction LLM (Oracle's own published evaluation used far stronger
models). If you have a bigger local model, one env var upgrades the extractor:

```bash
CHAT_MODEL=ollama/qwen2.5:14b ./.venv/bin/python demo2_agent_memory.py
```

Great narrative beat for the video: "the memory lifecycle is a database
problem — but memory *quality* is still an LLM problem."

## Config

Edit [config.py](config.py) or set env vars (`ORACLE_DSN`, `CHAT_MODEL`, …).
Demo 2 wipes and recreates its thread (`tokyo_trip_demo`) each run, so takes
are reproducible on camera.
