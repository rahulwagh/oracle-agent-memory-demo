# Installation Instructions

Everything you need to run the two demos — **flat history** vs **Oracle Agent
Memory** — on a fresh machine. All local, all free: an Oracle AI Database
container, two small Ollama models, one Python environment. No cloud, no API
keys. Expect ~15 minutes, most of it download time.

Each step ends with a **Verify** check — don't move on until it passes.

---

## 0. Prerequisites

| Tool | Version | Verify with |
|---|---|---|
| Docker (Desktop or Engine) | 20.10+ | `docker --version` |
| Python | **3.11+ (3.12 recommended)** | `python3.12 --version` |
| Git | any recent | `git --version` |
| Free disk | ~12 GB | Oracle image ~9 GB + Llama ~5 GB |

> ⚠️ **Python 3.10 will not work** — a dependency (`litellm`) needs 3.11+.
> On macOS: `brew install python@3.12`, then use `python3.12` below.

macOS: make sure Docker Desktop is **running** before any `docker` command.

## 1. Get the code

```bash
git clone https://github.com/rahulwagh/oracle-agent-memory-demo.git
cd oracle-agent-memory-demo/agent-memory-use-case/live-demo
```

Everything below runs from `agent-memory-use-case/live-demo/`.

## 2. The database — where the memories will live

Create the Oracle AI Database Free container (first time only; ~3.5 GB
download — note: Oracle brands it *26ai*, but the container tag is `23.26.x`):

```bash
docker run -d --name oracle26ai \
  -p 1521:1521 \
  -e ORACLE_PWD=Welcome_123 \
  -v oracle26ai-data:/opt/oracle/oradata \
  container-registry.oracle.com/database/free:23.26.1.0
```

Then bootstrap it — starts the container if stopped, waits until the listener
**really** accepts connections, and creates the `memdemo` user the demos
connect as (idempotent, re-run any time):

```bash
bash setup_db.sh
```

**Verify:**
```
  ✓ database is up
  ✓ memdemo/Welcome_123 ready on oracle26ai (localhost:1521/FREEPDB1)
```

> A fresh container's first boot takes several minutes (it builds the seed
> database) — `setup_db.sh` polls with a real query, just let it wait.

## 3. The models — the brain and the embedder

The demo talks to two small local models through [Ollama](https://ollama.com):
`llama3.1:8b` answers questions **and** distills memories; `nomic-embed-text`
turns text into vectors so memories can be found by meaning.

```bash
brew install ollama                 # macOS (or: ollama.com/download)
brew services start ollama          # daemon on :11434

ollama pull llama3.1:8b             # ~5 GB
ollama pull nomic-embed-text        # ~270 MB
```

**Verify:**
```bash
curl -s http://127.0.0.1:11434/api/tags | grep -o '"name":"[^"]*"'
# expect both: llama3.1:8b and nomic-embed-text
```

## 4. The Python environment

```bash
python3.12 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install oracleagentmemory rich matplotlib requests
```

`oracleagentmemory` is the official Oracle Agent Memory package — the memory
layer both demos are built around.

**Verify:**
```bash
python -c "import litellm, oracleagentmemory, oracledb; import sys; print('imports OK —', sys.version.split()[0])"
# imports OK — 3.12.x
```

> Every `python` command below assumes the venv is **active**
> (`source .venv/bin/activate` in each new shell). Running the system
> `python3` instead is the #1 cause of `No module named 'litellm'`.

## 5. End-to-end check

One line proves the venv can reach the database:

```bash
python -c "
import oracledb
c = oracledb.connect(user='memdemo', password='Welcome_123', dsn='localhost:1521/FREEPDB1')
print('connected:', c.version); c.close()"
# connected: 23.26.x.x
```

And warm the model so the first demo call isn't ~30 s slow:

```bash
curl -s http://127.0.0.1:11434/api/generate -d '{"model":"llama3.1:8b","prompt":"warm","stream":false}' > /dev/null
```

## 6. Run the demos

```bash
python demo1_flat_history.py     # 🔴 the problem — tokens climb 64 → 1,433
python demo2_agent_memory.py     # 🟢 the fix — stays ≈500, prints the memories
python show_memories.py          # 🔍 plain SQL over the MEMORY table
python plot_comparison.py        # 📉 your own token chart from the real logs
```

What success looks like (demo-2 numbers vary a few tokens per run —
extraction is LLM-driven):

| | turn 1 | turn 12 | whole run |
|---|---|---|---|
| demo 1 · flat history | 64 tk | **1,433 tk** | ~9,000 tk |
| demo 2 · agent memory | ~240 tk | **≈ 500 tk** | ~6,000 tk |

---

## Troubleshooting

<details>
<summary><b><code>No module named 'litellm'</code></b></summary>

You ran the system `python3` instead of the venv — `source .venv/bin/activate`
first (or call `./.venv/bin/python …`). If it happens *inside* the venv, the
venv was created with Python ≤ 3.10 — recreate it with `python3.12 -m venv .venv`.
</details>

<details>
<summary><b><code>setup_db.sh</code> waits forever</b></summary>

First boot of a fresh container takes 5+ minutes. Watch
`docker logs -f oracle26ai` until `DATABASE IS READY TO USE!`, then re-run
`bash setup_db.sh`.
</details>

<details>
<summary><b><code>ORA-51962: vector memory area is out of space</code></b></summary>

The Free image ships with `vector_memory_size = 0`; give it a pool and restart:

```bash
docker exec -i oracle26ai sqlplus -S -L sys/Welcome_123@localhost:1521/FREE as sysdba <<'SQL'
ALTER SYSTEM SET vector_memory_size = 512M SCOPE=SPFILE;
EXIT;
SQL
docker restart oracle26ai && bash setup_db.sh
```
</details>

<details>
<summary><b>Warning about a missing purge job (CREATE JOB)</b></summary>

Benign for the demo. `setup_db.sh` grants the privilege — re-run it once if
you created the user another way.
</details>

<details>
<summary><b>First demo-2 turn is very slow</b></summary>

Ollama loading the model into memory — do the warm-up curl in step 5, or let
the first call take its ~30 s once.
</details>

<details>
<summary><b>Demo 2 fumbles a fact (e.g., the allergy)</b></summary>

Known limitation: extraction quality depends on the extractor LLM, and an 8B
model is the floor. One env var upgrades it:
`CHAT_MODEL=ollama/qwen2.5:14b python demo2_agent_memory.py`.
The memory *lifecycle* is a database problem — memory *quality* is still an
LLM problem.
</details>

---

## Clean up / start over

```bash
docker stop oracle26ai              # pause the DB (data persists in the volume)
docker start oracle26ai             # bring it back
docker rm -f oracle26ai && docker volume rm oracle26ai-data   # remove everything
```

Demo 2 wipes and recreates its own thread + memories on every run — takes are
reproducible with no manual cleanup.
