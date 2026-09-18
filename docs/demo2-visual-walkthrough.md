# `demo2_agent_memory.py` — the visual walkthrough

The whole file, explained in **7 pictures**. No jargon required.
(Prefer the deep, line-by-line version? →
[demo2-agent-memory-breakdown.md](demo2-agent-memory-breakdown.md))

---

## 1 · The whole idea

The chat goes to the **database**. The model only ever gets a small
cheat-sheet plus your newest question.

![The chat is stored as rows in Oracle AI Database; the model receives only a ~350-token cheat-sheet plus the new question](img/d2v-1.png)

## 2 · One object, three ingredients

`build_memory()` wires up the memory layer — a database, an embedder, and an
LLM. That's the entire setup.

![build_memory: a database connection, an embedder for meaning-search, and an LLM as the distiller](img/d2v-2.png)

## 3 · The distiller follows a plain-English policy

You don't program what to remember — you *describe* it.

![The extraction policy in plain English, with examples of what is kept and dropped](img/d2v-3.png)

## 4 · Job 1 — STORE

Every message becomes a labeled row. Nothing is lost — and nothing is re-sent.

![A chat message flows through add_messages into a labeled MESSAGE row](img/d2v-4.png)

## 5 · Job 2 — DISTILL

A second AI reads each exchange and keeps only the real facts — typed and
labeled with whose they are.

![An exchange about Ankita's peanut allergy flows through the extraction LLM into typed FACT and PREFERENCE rows](img/d2v-5.png)

## 6 · Job 3 — FIND

Before every answer, the database builds a fresh cheat-sheet — capped at 350
tokens — and that plus the question is ALL the model sees.

![get_context_card builds a capped cheat-sheet; the model sees only card plus question, staying about 500 tokens flat](img/d2v-6.png)

## 7 · Why the token count can't grow

Flat history drags the whole transcript along — and it gets longer every turn.
The memory request is capped by construction.

![Turn-12 requests drawn to scale: flat history 1,433 tokens vs agent memory about 500](img/d2v-7.png)

---

**That's the file.** Run it yourself:

```bash
cd agent-memory-use-case/live-demo
./.venv/bin/python demo2_agent_memory.py
```
