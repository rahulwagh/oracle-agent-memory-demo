# The Memory Illusion — AI Agent Memory, Live on Your Laptop

Your AI chat app feels like it remembers you. It doesn't. This repo proves it —
and then fixes it — with two runnable demos and a set of interactive
explainer decks. Everything runs **100% local**: a free Oracle AI Database in
Docker, Llama via Ollama. No cloud. No API keys.

> 📺 Companion video: coming soon on [@RahulWagh](https://www.youtube.com/@RahulWagh)

## The experiment

One scripted 12-turn conversation (planning a Tokyo trip), run twice:

| | Demo 1 — flat history | Demo 2 — Oracle Agent Memory |
|---|---|---|
| Conversation state | re-sent inside every prompt | rows in **Oracle AI Database** |
| Tokens per request | 64 → **1,433** (climbing) | ≈ **500** (flat) |
| Turn-12 recall ("which seat?") | works — by re-reading everything | works — by **remembering** |

The memory layer is the official
[`oracleagentmemory`](https://pypi.org/project/oracleagentmemory/) package,
doing three jobs: **store** the chat (`add_messages`) · **distill** the facts
(a plain-English extraction policy) · **find** them (`get_context_card`).

## Run it

```bash
cd agent-memory-use-case/live-demo
bash setup_db.sh                       # Oracle AI Database Free (Docker) + demo user
ollama pull llama3.1:8b && ollama pull nomic-embed-text
python3.12 -m venv .venv && ./.venv/bin/pip install oracleagentmemory rich matplotlib requests

./.venv/bin/python demo1_flat_history.py    # the problem  (red)
./.venv/bin/python demo2_agent_memory.py    # the fix      (green)
./.venv/bin/python show_memories.py         # "Memory updated" = SELECT-able rows
./.venv/bin/python plot_comparison.py       # your own token chart
```

Full details, config, and a known-limitations section:
[`agent-memory-use-case/live-demo/README.md`](agent-memory-use-case/live-demo/README.md)

## The explainer decks

Self-contained HTML slides (open in a browser, arrow keys to step) in
[`agent-memory-use-case/demo/`](agent-memory-use-case/demo/):

- `memory-illusion-demo.html` — the concept: 5 facts your chat app never shows you
- `demo1-step-by-step.html` — flat history, with a live payload X-ray
- `demo2-step-by-step.html` — the memory layer: store · distill · find
- `code-walkthrough-step-by-step.html` — the whole thing in three calls
- `agent-memory-step-by-step.html` — combined architecture overview

## Reference

- Oracle Developers Blog: [Agent memory is a database problem](https://blogs.oracle.com/developers/agent-memory-is-a-database-problem-oracle-research-makes-the-case)
- Technical report: *Oracle Agent Memory as an Enterprise Memory Substrate for
  Long-Horizon AI Agents* (arXiv:2607.13157)

## License

Copyright (c) 2026 Rahul Wagh. Licensed under the Universal Permissive License
v1.0 — see [LICENSE](LICENSE).
