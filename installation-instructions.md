# Installation Instructions — The Memory Illusion (Oracle Agent Memory Demo)

> Step-by-step setup so a fresh machine can run both demos.
> Everything is local and free — no cloud account, no API keys.

---

## 0. Prerequisites

| Tool | Minimum version | Verify with |
|---|---|---|
| Docker (Desktop or Engine) | 20.10+ | `docker --version` |
| Python | **3.11+ (3.12 recommended)** | `python3.12 --version` |
| Git | any recent | `git --version` |
| Disk space | ~12 GB free | Oracle image ~9 GB + Llama ~5 GB |

> ⚠️ **Python 3.10 will not work.** A dependency (`litellm`) uses
> `typing.NotRequired`, which needs Python **3.11+**. If `python3` on your
> machine is 3.10, install 3.12 (`brew install python@3.12`) and use
> `python3.12` everywhere below.

macOS users: Docker Desktop must be **running** before any `docker` command works.

---

## 1. Clone the repo

```bash
git clone https://github.com/rahulwagh/oracle-agent-memory-demo.git
cd oracle-agent-memory-demo/agent-memory-use-case/live-demo
```

All commands below run from `agent-memory-use-case/live-demo/`.

---

## 2. Boot Oracle AI Database Free (Docker)

> **Naming note:** Oracle markets the product as "Oracle AI Database 26ai",
> but the container tag is `23.26.x` — pull from the `database/free` repo.

First time only — create the container (~3.5 GB download, ~9 GB on disk):

```bash
docker run -d --name oracle26ai \
  -p 1521:1521 \
  -e ORACLE_PWD=Welcome_123 \
  -v oracle26ai-data:/opt/oracle/oradata \
  container-registry.oracle.com/database/free:23.26.1.0
```

Then bootstrap it (starts the container if stopped, **waits until the
listener really accepts connections**, and creates the `memdemo` demo user —
idempotent, safe to re-run any time):

```bash
bash setup_db.sh
```

**Verify:**
```
  ✓ database is up
  ✓ memdemo/Welcome_123 ready on oracle26ai (localhost:1521/FREEPDB1)
```

> First boot of a fresh container takes **several minutes** (it builds the
> seed database). `setup_db.sh` polls with a real `SELECT 1 FROM DUAL` probe,
> so just let it wait.

---

## 3. Recommended: enable vector memory (one-time)

The Free image ships with `vector_memory_size = 0`. If the memory layer
creates an in-memory HNSW vector index you'd hit
`ORA-51962: The vector memory area is out of space`. Set it once, persist,
restart:

```bash
docker exec -i oracle26ai sqlplus -S -L sys/Welcome_123@localhost:1521/FREE as sysdba <<'SQL'
ALTER SYSTEM SET vector_memory_size = 512M SCOPE=SPFILE;
EXIT;
SQL

docker restart oracle26ai && bash setup_db.sh   # wait for it to come back
```

**Verify:**
```bash
docker exec -i oracle26ai sqlplus -S -L sys/Welcome_123@localhost:1521/FREE as sysdba <<< "show parameter vector_memory_size;"
# Expect:  vector_memory_size ... 512M
```

---

## 4. Install Ollama + the two local models

```bash
brew install ollama                 # macOS (or: https://ollama.com/download)
brew services start ollama          # daemon on :11434, auto-restarts on login

ollama pull llama3.1:8b             # the chat + extraction LLM (~5 GB)
ollama pull nomic-embed-text        # the embedder (768-dim, ~270 MB)
```

**Verify:**
```bash
curl -s http://127.0.0.1:11434/api/tags | grep -o '"name":"[^"]*"'
# expect both: llama3.1:8b and nomic-embed-text
```

> 💡 **Before recording/demoing:** warm the model once so the first call
> isn't ~30 s slow:
> ```bash
> curl -s http://127.0.0.1:11434/api/generate -d '{"model":"llama3.1:8b","prompt":"warm","stream":false}' > /dev/null
> ```

---

## 5. Python environment

```bash
python3.12 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install oracleagentmemory rich matplotlib requests
```

**Verify:**
```bash
python -c "import litellm, oracleagentmemory, oracledb, rich, matplotlib; import sys; print('all imports OK —', sys.version.split()[0])"
# all imports OK — 3.12.x
```

> Every `python` command below assumes the venv is **active**. New shell?
> `source .venv/bin/activate` again. (Running the system `python3` instead is
> the #1 cause of `ModuleNotFoundError: No module named 'litellm'`.)

---

## 6. Smoke-test the venv → DB connection

```bash
python -c "
import oracledb
c = oracledb.connect(user='memdemo', password='Welcome_123', dsn='localhost:1521/FREEPDB1')
print('connected:', c.version)
c.close()"
# connected: 23.26.x.x
```

---

## 7. Run the demos

```bash
python demo1_flat_history.py     # 🔴 the problem — tokens climb 64 → 1,433
python demo2_agent_memory.py     # 🟢 the fix — stays ≈500, then prints the memories
python show_memories.py          # 🔍 plain SQL over the MEMORY table
python plot_comparison.py        # 📉 renders token_comparison.png from the real logs
```

Expected shape of the results (demo-2 numbers vary a few tokens per run —
extraction is LLM-driven):

| | turn 1 | turn 12 | total |
|---|---|---|---|
| demo 1 · flat history | 64 tk | **1,433 tk** | ~9,000 tk |
| demo 2 · agent memory | ~240 tk | **≈500 tk** | ~6,000 tk |

---

## 8. Troubleshooting

<details>
<summary><b><code>ModuleNotFoundError: No module named 'litellm'</code></b></summary>

You ran the system `python3` instead of the venv. Either
`source .venv/bin/activate` first, or call `./.venv/bin/python …` explicitly.
If it happens **inside** the venv, your venv was created with Python ≤3.10 —
recreate it with `python3.12 -m venv .venv`.
</details>

<details>
<summary><b><code>ORA-51962</code> when the memory layer creates a vector index</b></summary>

`vector_memory_size` is still 0 — do step 3 (set to 512M, restart the container).
</details>

<details>
<summary><b><code>setup_db.sh</code> says the database never came up</b></summary>

First boot of a fresh container can take 5+ minutes. Check progress with
`docker logs -f oracle26ai` — wait for `DATABASE IS READY TO USE!`, then
re-run `bash setup_db.sh`.
</details>

<details>
<summary><b>Warning about a missing purge job / CREATE JOB privilege</b></summary>

Benign for the demo. `setup_db.sh` grants `CREATE JOB` — re-run it once if
you created the user some other way.
</details>

<details>
<summary><b>First demo-2 turn is very slow</b></summary>

Ollama is loading the model into memory. Do the warm-up curl from step 4, or
just let the first call take its ~30 s once.
</details>

<details>
<summary><b>Demo-2 extraction fumbles a fact (e.g., an allergy)</b></summary>

Known limitation — the extraction quality depends on the extractor LLM, and
`llama3.1:8b` is the floor. A bigger local model helps:
`CHAT_MODEL=ollama/qwen2.5:14b python demo2_agent_memory.py`.
The memory *lifecycle* is a database problem; memory *quality* is still an
LLM problem.
</details>

---

## Clean up / start over

```bash
docker stop oracle26ai                      # stop the DB (data persists in the volume)
docker start oracle26ai                     # bring it back
docker rm -f oracle26ai && docker volume rm oracle26ai-data   # nuke everything
```

Demo 2 wipes and recreates its own thread + memories on every run, so takes
are reproducible without any manual cleanup.
