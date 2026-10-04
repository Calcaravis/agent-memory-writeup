# Giving a local AI agent a memory it can trust

*A write-up of rebuilding the memory of a self-hosted [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research) in a Proxmox homelab: a curated knowledge tree, hybrid search over old chats with a reranker, a nightly consolidation job with human approval, and everything that went wrong on the way.*

> Written with the help of an AI assistant (Claude), based on the actual logs, measurements and scripts of the project. All numbers are from my setup.

---

## TL;DR

- The agent's built-in "auto memory" quietly wrote unverified facts into a file that was loaded into every prompt. I switched it off and replaced it with **four layers**: a short always-loaded persona file, a **git-versioned knowledge tree** with dated and sourced facts, an **inbox** for new facts, and a **vector + keyword search** over all old chats.
- Nothing enters the knowledge tree without my **ok**: a nightly job proposes changes on a git branch, the agent sends me a summary at 08:00, I answer *ok* or *not*.
- A cross-encoder **reranker** moved the right chat to rank 1 for 19 of 24 test questions (was 11). On CPU it cost 3.6 s per search; as a small GPU service it costs 73 ms.
- Most of the agent's **tool-call loops** (1,098 in 9 days) happened in sessions longer than 150 messages, right after context compression. Automatic session resets fixed most of it.
- The biggest time sinks were not AI problems: **time zones, cgroups, cache paths and secrets in chat logs**.

---

## The setup

| Part | What runs there |
|---|---|
| Proxmox host | 2 GPUs: RTX 5090 (32 GB) and RTX PRO 4000 Blackwell SFF (24 GB) |
| LXC "inference" | OpenAI-compatible inference server with **qwen3.8-27b NVFP4** on the 5090; speech (STT/TTS) on the PRO 4000; later also the reranker service |
| LXC "agent" (8 GB RAM) | Hermes Agent v0.21 (Telegram gateway, cron, 7 profiles: orchestrator, worker bots, a mail reader), Qdrant 1.19, the vault (`/root/vault`), the memory tools |
| n8n | Mail fetching, update reports, other workflows |
| Proxmox Backup Server | Nightly snapshots of all service containers to a separate SSD |

Everything is local. The only cloud fallback the agent had was removed on purpose.

---

## Architecture: four layers instead of one memory file

```
always loaded      SOUL.md (persona, rules, lookup order)            ~50 lines
read on demand     knowledge tree  /root/vault/wissen/**  (git)     facts, one per line
inbox              /root/vault/eingang.md                           new, unconfirmed facts
search             Qdrant: old chats, exports, wiki, daily summaries   dense + sparse + reranker
procedures         skills (one per workflow)
```

**Fact line format** in the tree and the inbox:

```
- NInfer listens on port 8000 · as of 2026-09-30 · origin: measured · source: curl
```

`origin` is `stated` (I said it), `measured` (checked live) or `derived` (concluded from old chats). Facts that are no longer true move under `## Earlier` instead of being deleted. Each folder has a short `AGENTS.md` index (max. 30 lines) that the agent reads first.

**Lookup order** (one line in the persona file): tree index → topic file → inbox → vault search (max. three calls) → session search. Hits from old chats are always reported as *unconfirmed, with file and date*.

**What goes where** turned out to be the most important design rule:

- *How my system is* (hosts, ports, decisions) → knowledge tree
- *How the agent should do a task* (e.g. "never review the same video twice") → the skill for that task
- Persona and values only → SOUL.md. Paths and procedures there just bloat every prompt.

---

## Vault search (phase 3)

- **Sources:** exported chats from Claude, Gemini, ChatGPT and the agent's own Telegram sessions; wiki notes; daily summaries. About 41,000 chunks.
- **Embeddings:** `paraphrase-multilingual-MiniLM-L12-v2` (384 d, German and English) plus sparse `bm42` with IDF, both via fastembed on the CPU.
- **Chunking:** windows of 120 tokens with 20 overlap. The dense model silently truncates at 128 tokens, and 37 % of the old chunks were cut off.
- **Fusion and grouping:** Qdrant query API with prefetch 50 per vector type, then one hit per file (`query/groups`).
- **Reranker:** `jina-reranker-v2-base-multilingual` (ONNX) re-sorts the best 30 files.
- **Logging:** every query goes to a JSONL log (query, filters, hit files, no snippets). The nightly job uses it later.

### Measurements (24 test questions from real old chats)

| Mode | in top 5 | rank 1 | top 3 | MRR | s/query |
|---|---|---|---|---|---|
| dense + sparse, DBSF fusion | 20 | 11 | 18 | 0.604 | 0.03 |
| DBSF + reranker | 22 | 18 | 19 | 0.793 | 3.6 (CPU) |
| **RRF (k=60) + reranker** | **22** | **19** | **21** | **0.835** | 3.6 (CPU) |

The reranker was supposed to stay only if it put two more questions into the top 5. With a baseline of 20 to 22 out of 24 that rule was useless, so it now also counts rank 1 and MRR.

On the CPU (4 threads) a search took 6.3 s in total: 2 s to load the model in every new process and 3.6 s to sort. As a small HTTP service on the second GPU (about 2.6 GB VRAM next to the speech server) the sorting takes **73 to 93 ms** and a whole search **1.5 s**. The client falls back to the CPU model, and then to no reranking, if the service is down, and says so in its output.

---

## Nightly consolidation (phase 4)

A systemd timer runs one Python script at 03:30, after the backup at 02:00. Each step is independent: a failing step is reported and the others still run.

1. **profile**: every agent profile must have the built-in memory and "background review" switched off. New profiles start with both *on*.
2. **export**: new or grown Telegram sessions go to Markdown (only user and assistant text, no tool output). Then a scan for literal secret values runs over all exports.
3. **summary**: the local LLM writes a daily note into the wiki. Long days are summarised in parts of about 40k characters, then merged.
4. **promote**: chat passages found by **three different questions** in 14 days become `derived` fact proposals in the inbox.
5. **cleanup**: inbox lines that are already in the tree are removed, after a backup.
6. **propose**: inbox lines are proposed for the tree on a **git branch `nacht/<date>`**, via a git worktree, so master is never touched. The LLM only picks the target file and the line it replaces. The script checks that the file exists, the format is valid and the replaced line exists word for word.
7. **index**: incremental re-index.
8. **measure** (Sundays): runs the test set and warns below 20/24.

At 08:00 a cron job of the agent sends me the report. A small script merges the branch on **ok** (`git merge --no-ff`, lines leave the inbox) or deletes it on **nein** (lines go to `abgelehnt.md`). The agent may only run it after my explicit answer.

**Day one:** on the first night qwen proposed to file *"don't review already-checked videos again"* under… my camera NVR. I said *nein*, nothing changed. That rule now lives in the video-review skill. On the second night all four proposals were right.

Since the latest change, a gateway hook (`session:end` / `session:reset`) also starts a quick export and index **without LLM** after every `/new`, so a conversation is searchable a minute after it ends.

---

## Problems, pitfalls and fixes

### Memory and data

1. **Auto memory writes unchecked facts.** The built-in memory and the post-session "background review" wrote whatever the model concluded into files that were loaded into every prompt. *Fix:* both off; facts only via inbox and approval. *Pitfall:* the review setting is **fail-open** (on by default), and every newly created profile gets the defaults again. The nightly job now checks all profiles.
2. **Secrets in chat logs.** API keys had been pasted into chats over the months. *Fix:* masking in two ways, by pattern (sk-, ghp_, JWT, `token=…`) **and** by the literal values from the secrets files and `.env`, plus a scan that runs after every export and counts only names, never values. *Bug found on the way:* titles were truncated **before** masking, which can leave half a key. Redact first, then truncate.
3. **Over-masking.** Treating every `.env` value as secret also masked URLs, paths and ports. Values that are URLs without credentials, paths or numbers, and names ending in `_URL`, `_PORT`, `_PATH`, are now excluded.
4. **Test and admin sessions polluting the memory.** An exclude list (session id or its last 8 characters) plus "first message starts with *Testlauf*" keeps them out. Previously exported files of excluded sessions are deleted and removed from the index.
5. **Wrong source labels.** Old Telegram logs had been exported as "ChatGPT". *Fix:* a marker check and a separate source value, so search results say who was talking.

### Infrastructure

6. **Long jobs died with the agent.** Anything the agent started with `nohup` in its terminal lived in the gateway's cgroup and died on every gateway restart, including a 40-minute index build. *Fix:* long jobs only as systemd units (`systemd-run` or real services), started from the host shell.
7. **"Could not load model" only under systemd.** fastembed caches models under `$TMPDIR`, and `$TMPDIR` differs between the agent's terminal and systemd. *Fix:* fixed `FASTEMBED_CACHE_PATH`.
8. **OOM at 4 GB.** Loading four embedding models at once for a comparison killed the container. *Fix:* load models one after another with small batches; container raised to 8 GB.
9. **Hybrid search worse than its parts.** Qdrant's default RRF uses a small *k*, which caused many ties. *Fix:* explicit `{"rrf": {"k": 60}}` or DBSF, and the sparse vector with `modifier: idf`.
10. **Time zones.** The inference container logs in UTC, the agent thinks in Berlin time. One morning the agent "found" a mysterious one-hour load spike at night, which was its own analysis session two hours later. *Fix:* a timeline tool that reads the journal as Unix seconds and converts everything itself.
11. **Watchdog false alarm.** A latency watchdog alarmed on "cache hit rate 0 %" right after the nightly job, whose single questions can never hit the KV cache. *Fix:* the cache check only counts multi-turn requests (4+ messages), and every alarm now names the caller ("nightly job, 14 requests").
12. **Manual runs hid the real report.** A test run wrote the newest report file, and the morning cron would have sent that instead of the night's run. Manual runs now write to a separate folder.

### Agent behaviour

13. **Parallel search storms.** The agent once fired 15 identical searches in parallel. *Fix* in the skill: at most three calls per question, one at a time, never the same call twice. A `--datei` mode reads more of one file without anyone opening raw files, and the output ends with an explicit "end of results" marker.
14. **Tool-call loops.** A read-only analysis of the agent's own database (1,098 loops of 3+ identical calls in 9 days) showed:
    - 86 % of the loops happened after more than 150 messages in the session
    - 81 % right after context compression
    - 91 % with an identical tool result each time and no text in between
    - 45 % with an error in the result, mostly a failing `ssh` command retried verbatim

    The memory rebuild was *not* the cause: the loop rate was highest in the days before it (up to 8.9 % of tool calls). *Fix:* automatic session reset after 120 minutes idle and daily at 04:00, plus one persona rule: *"same result or same error twice → change the approach or ask"*. Habit: `/new` per topic, never "continue" after a loop.
15. **The LLM files facts in the wrong place.** That is exactly why the approval step exists. Cheap guards in code catch the rest: target file must exist, format regex, replaced line must exist verbatim, invented lines are ignored.
16. **Thinking tokens.** The nightly prompts switch thinking off via `chat_template_kwargs: {"enable_thinking": false}`. If the server rejects the field, the script retries without it and with a larger token budget.
17. **Counting popularity wrong.** "Found 3 times" first counted repeated identical queries. It now counts *distinct* questions and only the top 2 hits, and ignores searches limited to one file.

### Security: prompt injection by email

Mail arrives through n8n. One mail tried a prompt injection; the agent ignored it, but that is not a defence. Now:

- **Quarantine model:** a script sends the raw mail to the LLM once and gets back only JSON (`subject`, `from`, `summary`, `suspicious`). If the LLM fails, only subject and sender go on, never the body.
- **Fixed checks** on the *complete* raw mail, not only the first 3,000 characters the model sees. They look for "ignore previous instructions" (German and English), "system prompt", chat control tokens, tool names, "send me the API keys", invisible Unicode and long base64 blocks. A hit marks the mail suspicious whatever the model says.
- **Subject and sender** come from the raw mail, never from the model's answer, so a mail cannot make the model claim it came from someone else. Links are defanged (`hxxps://`).
- **Least privilege:** a separate agent profile with only `clarify` and `todo`, no terminal, files, web, memory or agent-to-agent tools, a few turns, memory and background review off.
- **Still open:** *replying* to the mail summary in Telegram passes the quoted text to the main agent, which has full tools. The gateway log showed 15 quoted replies. Until that is filtered: don't reply to mail summaries.

---

## What I would do the same way again

- **Measure first.** A 24-question test set from real old chats decided every search change. Without it I would have kept a "hybrid" that was worse than its parts.
- **Keep humans in the write path, not the read path.** The agent may search everything, but it may change its own long-term knowledge only through a reviewed git branch.
- **Facts with date, origin and source** make contradictions easy to resolve: newer measured beats older derived.
- **Deterministic code around the LLM:** validation, budgets, locks, fallbacks. The model makes suggestions, the script decides what is valid.
- **Read-only diagnostic scripts** that print counts instead of contents. They answered "why does it loop?" and "who used the GPU at 4 am?" in one step each, without leaking anything.

## What I decided against

- **Moving both GPUs into the agent's container.** The agent runs as root with a terminal. Keeping the model server in a separate container limits what a loop or a bad command can break, and the network hop costs about 1 ms.
- **A small fallback LLM for the agent.** A much weaker model with full tool access, used exactly when something is already wrong, felt like the bigger risk. First I'll measure how often the main model is actually down.

## Next

- Contextual sentences per chunk (Anthropic's "contextual retrieval") for new files, measured against the test set.
- A routing setup for more mail providers (web.de, Gmail) through the same quarantine path.
- Filtering quoted mail text before it reaches the main agent.

---

### Versions

Hermes Agent v0.21.5 · Qdrant 1.19.1 · fastembed 0.8.0 · onnxruntime-gpu 1.30 (CUDA 13) · qwen3.8-27b NVFP4 · Python 3.12/3.13 · Proxmox VE 9

*Note on licences:* `jina-reranker-v2-base-multilingual` is CC-BY-NC-4.0, which is fine for a private homelab but not for commercial use.

Inspired by: Anthropic's "Contextual Retrieval" write-up, OpenClaw's daily notes and "dreaming" consolidation, and the Hermes Agent docs on SOUL.md and hooks.
