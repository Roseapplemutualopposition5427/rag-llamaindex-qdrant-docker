---
agent_runnable: true
estimated_duration_minutes: 30
requires_internet: true
prerequisites: prerequisites.json
---

# Agent runbook: rag-llamaindex-qdrant-docker

This runbook is for AI agents (Claude Code, Gemini CLI, Cursor, Cline) automating the setup and operation of a multi-collection RAG pipeline built on LlamaIndex, Qdrant, and Docker. Before starting, verify all required tools and versions defined in `prerequisites.json` are present on the host — specifically Docker >= 24.0, Docker Compose >= 2.20, Python 3.12, and Git >= 2.40. Execute tasks in order from top to bottom and stop immediately at any failed Verify line — do not proceed to the next task if the current Verify does not pass.

## Task 0: Pick provider

Inputs: one of `OPENAI_API_KEY` or `OLLAMA_HOST` set in the environment. The choice determines embedding dimensions for the Qdrant collection — OpenAI uses 3072-dimensional vectors, Ollama `bge-m3` uses 1024. This cannot be changed after the first ingest without deleting `qdrant-data/` and re-indexing.

Action:
```bash
if [ -n "$OPENAI_API_KEY" ]; then echo "Provider: OpenAI"; elif [ -n "$OLLAMA_HOST" ]; then echo "Provider: Ollama ($OLLAMA_HOST)"; else echo "ERROR: no provider set"; exit 1; fi
```

Verify: [ -n "$OPENAI_API_KEY" ] || [ -n "$OLLAMA_HOST" ]

## Task 1: Clone and configure

Clone the repository and populate the local configuration files from the provided templates. The `.env` file holds provider credentials and the Qdrant URL. The `collections.yaml` file declares each collection name and its chunking parameters.

Action:
```bash
git clone https://github.com/<your-handle>/rag-llamaindex-qdrant-docker.git && cd rag-llamaindex-qdrant-docker && cp .env.example .env && cp collections.yaml.example collections.yaml
```

Edit `.env`: uncomment the provider block that matches Task 0. If using OpenAI, set `OPENAI_API_KEY`, `OPENAI_EMBED_MODEL`, and `OPENAI_LLM_MODEL`. If using Ollama, set `OLLAMA_HOST`, `OLLAMA_EMBED_MODEL`, and `OLLAMA_LLM_MODEL`. Leave the other block commented. The `QDRANT_URL` line is always-on and should remain as `http://qdrant:6333` when running inside Docker. Edit `collections.yaml` to declare the required collections — each `name:` value must exactly match a folder name under `documents/`.

Verify: grep -E '^(OPENAI_API_KEY|OLLAMA_HOST)=' .env

Expected: exactly one uncommented line returned.

## Task 2: Start Qdrant

Launch the Qdrant vector database container in detached mode. The `qdrant-data/` directory at the repo root is bind-mounted as the persistence layer — all vectors survive container restarts.

Action:
```bash
docker compose -f docker/docker-compose.yml up -d qdrant
```

Poll until the container reports healthy, up to 30 seconds:

```bash
for i in $(seq 1 30); do status=$(docker inspect --format='{{.State.Health.Status}}' rag-qdrant 2>/dev/null); [ "$status" = "healthy" ] && echo "healthy" && break; sleep 1; done
```

Verify: docker inspect --format='{{.State.Health.Status}}' rag-qdrant

Expected value: `healthy`

## Task 3: Drop in documents

Inputs: at least one `.pdf`, `.txt`, or `.md` file for each collection declared in `collections.yaml`. The folder name under `documents/` must exactly match the `name:` field — a mismatch causes the ingest script to skip the collection silently.

Action:
```bash
mkdir -p documents/<collection_name>
cp /path/to/source/files/* documents/<collection_name>/
```

Replace `<collection_name>` with the value from the `name:` field in `collections.yaml`.

Verify: ls documents/<collection_name>/*.{pdf,txt,md} 2>/dev/null | wc -l

Expected value: integer >= 1

## Task 4: Ingest

Run the ingest script as a one-shot container using the `manual` Docker Compose profile. The script reads `collections.yaml`, loads each document folder, chunks and embeds all files, and upserts the resulting vectors into the corresponding Qdrant collection. The `ingest` service declares `depends_on: qdrant: condition: service_healthy`, so this step requires Task 2 to have passed.

Action:
```bash
docker compose -f docker/docker-compose.yml --profile manual run --rm ingest python /app/scripts/ingest.py 2>&1 | tee /tmp/ingest.log
```

Verify: `grep -q '\[ingest\] Done\.' /tmp/ingest.log`
Verify: `curl -s http://127.0.0.1:6333/collections | grep -q '"result"'`

## Task 5: Query

Run a test query against the ingested collection to confirm end-to-end retrieval and generation. Replace `<collection_name>` with the target collection and supply a question that should be answerable from the ingested documents.

Action:
```bash
docker compose -f docker/docker-compose.yml --profile manual run --rm ingest python /app/scripts/query.py <collection_name> "<test question>"
```

Verify: stdout contains `Sources:` and at least one source line shows a score >= 0.5.

## Task 6: Schedule daily sync (optional)

Configure a cron job to run `sync-rag.sh` daily. The script invokes `ingest.py --sync` in incremental mode: new files are embedded and added, deleted files are removed from Qdrant, unchanged documents are skipped. Resolve the repo path at runtime so the cron entry is portable.

Action:
```bash
REPO=$(pwd) && (crontab -l 2>/dev/null; echo "0 3 * * * $REPO/scripts/sync-rag.sh >> /var/log/rag-sync.log 2>&1") | crontab -
```

Verify: `crontab -l | grep -q sync-rag.sh`

## Failure modes

- If Task 4 fails with `UnicodeEncodeError`: see "Surrogate character UnicodeEncodeError" in `docs/troubleshooting.md`.
- If Task 5 hangs with no output: see "Async LlamaIndex query hangs forever" in `docs/troubleshooting.md`.
- If Task 2 healthcheck never reaches `healthy`: see "Qdrant healthcheck fails on Apple Silicon" in `docs/troubleshooting.md`.

For all other failures, read `docs/troubleshooting.md` end-to-end before retrying.
