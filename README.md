<!--
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "Build a RAG System with LlamaIndex and Qdrant in Docker",
  "description": "Step-by-step tutorial for building a multi-collection RAG with LlamaIndex, Qdrant, and Docker.",
  "totalTime": "PT30M",
  "estimatedCost": {"@type": "MonetaryAmount", "currency": "USD", "value": "0.50"},
  "tool": [
    {"@type": "HowToTool", "name": "Docker"},
    {"@type": "HowToTool", "name": "Docker Compose"},
    {"@type": "HowToTool", "name": "Python 3.12"},
    {"@type": "HowToTool", "name": "Git"}
  ],
  "supply": [
    {"@type": "HowToSupply", "name": "OpenAI API key (optional)"},
    {"@type": "HowToSupply", "name": "Ollama running locally (optional)"}
  ],
  "step": [
    {"@type": "HowToStep", "name": "Clone and configure", "text": "Clone the repo, copy .env.example to .env, copy collections.yaml.example to collections.yaml."},
    {"@type": "HowToStep", "name": "Start Qdrant", "text": "Run docker compose up -d qdrant and verify it is healthy."},
    {"@type": "HowToStep", "name": "Drop in your documents", "text": "Copy PDFs into documents/<collection>/."},
    {"@type": "HowToStep", "name": "Run the first ingest", "text": "Run ingest.py to extract, chunk, embed, and store."},
    {"@type": "HowToStep", "name": "Ask your first question", "text": "Run query.py with a collection name and a question."},
    {"@type": "HowToStep", "name": "Schedule daily sync", "text": "Add a cron entry pointing at sync-rag.sh."}
  ]
}
-->

# Build a RAG System with LlamaIndex and Qdrant in Docker

## What you'll build

RAG stands for Retrieval-Augmented Generation. Retrieval-Augmented Generation lets a language model access external data during the generation process to improve accuracy and reduce hallucinations. The resulting system combines a Qdrant vector database with LlamaIndex orchestration and Docker containers to provide a scalable, multi-collection information retrieval pipeline for local or cloud deployments.

By the end of this tutorial, you will have a working stack running on your local machine. That stack includes a Qdrant vector database for persistent storage, Python scripts powered by LlamaIndex for document ingestion and querying, and a multi-collection setup that separates different document sets cleanly. You can choose between OpenAI embeddings for a fast first run or Ollama for a fully private, offline alternative.

The stack architecture follows a straightforward pattern:

```text
User → query.py → LlamaIndex → Qdrant → LlamaIndex → LLM → Answer
```

Every component is containerized using Docker to ensure consistency across different developer environments. LlamaIndex acts as the orchestration layer that connects raw files to the vector store and the language model. Qdrant provides the high-performance similarity search required to surface the right paragraph among thousands of pages. For a detailed view of the internal data flow, see the architecture diagram in `docs/architecture.png`.

## Before you start

Building a retrieval system requires specific tools and hardware to ensure consistent behavior across operating systems. Docker version 24.0 and Docker Compose version 2.20 provide the containerized environment for Qdrant and the ingest scripts. Python 3.12 serves as the runtime for LlamaIndex logic. Git 2.40 is required to clone and track the repository.

Verify each tool is installed before continuing:

- **Docker >= 24.0**: `docker --version` — install at [docs.docker.com/get-docker](https://docs.docker.com/get-docker/)
- **Docker Compose >= 2.20**: `docker compose version` — install at [docs.docker.com/compose/install](https://docs.docker.com/compose/install/)
- **Python 3.12**: `python3 --version` — install at [python.org/downloads](https://www.python.org/downloads/)
- **Git >= 2.40**: `git --version`

You also need one AI provider. Get an OpenAI API key from [platform.openai.com](https://platform.openai.com) for the cloud path. Or install [Ollama](https://ollama.com) locally for the free, private path. Only one provider needs to be active at a time.

Resources required: approximately 2 GB of free RAM and 2 GB of free disk space for Docker images and the Qdrant vector index.

Windows users should perform all steps inside a WSL2 instance to avoid path and permission issues that appear with native Windows shells.

## Pick your embeddings provider

Choosing an embedding provider determines the vector dimensions, cost structure, and privacy profile of the entire Qdrant collection. OpenAI offers high-speed API access using the text-embedding-3-large model with 3072-dimensional vectors for strong semantic precision. Ollama provides the bge-m3 model for local CPU or GPU execution, generating 1024-dimensional vectors at zero cost.

The following table compares both options directly:

| Metric | OpenAI text-embedding-3-large | Ollama bge-m3 |
| :--- | :--- | :--- |
| **Cost** | Paid per token | Free (local) |
| **Speed** | Fast (external API) | Slower (local CPU/GPU) |
| **RAM Footprint** | ~0 local RAM | ~600 MB local RAM |
| **Internet Required** | Yes | No |
| **Vector Dimensions** | 3072 | 1024 |

OpenAI is the recommended choice for the quickest first run — no local model download needed. Ollama is better for free and private workloads where documents must not leave your machine.

Switching providers after the first ingest is a destructive operation. Qdrant pins vector dimensions at the moment a collection is first created. If you start with Ollama (1024 dims) and want to switch to OpenAI (3072 dims), you must delete your `qdrant-data/` directory and re-ingest all documents from scratch. Decide on a provider before running the ingest script.

## Step 1 — Clone and configure

The configuration phase involves cloning the repository and populating the environment file for the chosen embedding provider. Copying the example `.env` and `collections.yaml` templates separates local credentials from the tracked source files. The folder name declared under `documents/` must match the `name:` field in `collections.yaml` for the ingest script to locate the correct source files.

Clone the repository and enter the project directory:

```bash
git clone https://github.com/<your-handle>/rag-llamaindex-qdrant-docker.git
cd rag-llamaindex-qdrant-docker
```

Copy the template files to create your local configuration:

```bash
cp .env.example .env
cp collections.yaml.example collections.yaml
```

Open `.env` in a text editor and fill in your provider credentials. Both blocks are pre-written in the example — uncomment one and leave the other commented out:

```env
# --- OpenAI (comment out the Ollama block below if using this) ---
OPENAI_API_KEY=sk-your-real-key-here
OPENAI_EMBED_MODEL=text-embedding-3-large
OPENAI_LLM_MODEL=gpt-4o-mini

# --- Ollama (comment out the OpenAI block above if using this) ---
# OLLAMA_HOST=http://host.docker.internal:11434
# OLLAMA_EMBED_MODEL=bge-m3
# OLLAMA_LLM_MODEL=llama3.2:3b

# --- Qdrant (always required) ---
QDRANT_URL=http://qdrant:6333
```

`QDRANT_URL` defaults to the Docker service name `qdrant`, which resolves automatically inside the Compose network. If you ever run the scripts on the host (outside Docker), change this to `http://localhost:6333`.

Now open `collections.yaml`. Each entry creates one Qdrant collection and maps to one folder under `documents/`. The `chunk_size` and `chunk_overlap` values are measured in tokens:

```yaml
collections:
  - name: research_papers
    description: arXiv PDFs and conference proceedings
    chunk_size: 512
    chunk_overlap: 50
  - name: notes
    description: Personal markdown notes
    chunk_size: 256
    chunk_overlap: 25
```

Larger chunks preserve more context per result but increase embedding cost slightly. Smaller chunks give more precise citations. The default values above are a solid starting point for most use cases.

**Verify:** `grep -E '^(OPENAI_API_KEY|OLLAMA_HOST)=' .env` returns one line — the active provider.

## Step 2 — Start Qdrant

Starting the Qdrant service initializes a persistent vector database inside a Docker container designed for high-performance semantic search operations. The `qdrant/qdrant:v1.13.4` image binds to `127.0.0.1:6333` so the database is only accessible from the local machine. A volume mount at `../qdrant-data` persists all indexed vectors across container restarts and system reboots.

Launch Qdrant in the background with Docker Compose:

```bash
docker compose -f docker/docker-compose.yml up -d qdrant
```

A vector database is fundamentally different from a standard SQL or key-value store. Traditional databases find exact matches — they cannot answer "find me chunks that mean the same thing as my question." Qdrant stores each document chunk as a mathematical vector and uses cosine similarity to measure the angle between two vectors. A small angle means high semantic similarity. This approach works at scale: Qdrant can search millions of vectors in milliseconds because it uses approximate nearest-neighbour (ANN) indexing rather than scanning every row.

The `qdrant-data/` directory at the repo root is the persistence layer. All your collections, vectors, and metadata live there. Docker writes to it via the bind mount defined in `docker-compose.yml`. You can stop the container, update the image, or reboot without losing your indexed documents.

The healthcheck in `docker-compose.yml` opens a TCP connection to port 6333 every five seconds (the Qdrant image does not ship `curl`, so a bash `/dev/tcp/` probe is used instead of an HTTP GET). The `ingest` service declares `depends_on: qdrant: condition: service_healthy`, so the ingest container will not start until Qdrant confirms it is ready.

**Verify:** `curl http://127.0.0.1:6333/` — expected response is JSON containing `{"title":"qdrant","version":"..."}`.

## Step 3 — Drop in your documents

Organizing source files into the `documents/` directory allows LlamaIndex to process each collection independently using the `SimpleDirectoryReader` class. Supported file types include `.pdf`, `.txt`, and `.md` placed within named subdirectories that correspond exactly to the `name:` fields declared in `collections.yaml`. A strict folder-to-collection mapping prevents cross-contamination of retrieval contexts during query time.

Create a subfolder for each collection you declared in `collections.yaml`, then copy your files in:

```text
documents/
├── research_papers/
│   ├── attention_is_all_you_need.pdf
│   └── bert_paper.pdf
└── notes/
    ├── project_ideas.md
    └── daily_log.txt
```

`SimpleDirectoryReader` scans each folder recursively. Drop any number of supported files in and the ingest script will pick them all up on the next run.

The folder name is the binding key between your files and Qdrant. If the folder is named `research_papers` but `collections.yaml` lists `name: papers`, the ingest script will not find the directory and will skip that collection silently. Always keep both names in sync.

Stick to alphanumeric characters, underscores, and hyphens for both folder names and file names. Special characters and spaces can break shell commands on some systems.

**Verify:** `ls documents/` shows at least one folder whose name matches a `name:` entry in `collections.yaml`.

## Step 4 — Run the first ingest

The ingestion process converts raw files from the `documents/` directory into high-dimensional floating-point vectors for storage in the Qdrant database. LlamaIndex orchestrates the full pipeline: text extraction via `SimpleDirectoryReader`, surrogate-character stripping, overlapping-window chunking, embedding via the chosen provider, and batch upsert into the target Qdrant collection. Each chunk is stored alongside source metadata for citation at query time.

Run the ingest script using Docker Compose:

```bash
docker compose -f docker/docker-compose.yml --profile manual run --rm ingest python /app/scripts/ingest.py
```

The `ingest` service is declared with `profiles: ["manual"]` in `docker-compose.yml`, which keeps it out of the auto-start path of `docker compose up`. The `--profile manual` flag activates it for one-shot runs.

The script reads every collection from `collections.yaml`, loads the matching folder under `documents/`, and works through the following pipeline steps for each file: extract text, strip surrogate characters that can cause encoding failures, split the text into overlapping chunks based on `chunk_size` and `chunk_overlap`, send each chunk to your embedding provider, and store the resulting vectors in Qdrant.

Expected terminal output:

```text
[ingest] Loading collection: research_papers
[ingest] Found 3 file(s) in documents/research_papers
[ingest] Chunking and embedding... (this may take a moment)
[ingest] Stored 147 vectors in collection 'research_papers'
[ingest] Loading collection: notes
[ingest] Found 8 file(s) in documents/notes
[ingest] Chunking and embedding...
[ingest] Stored 312 vectors in collection 'notes'
[ingest] Done.
```

> **Why this matters: `aclient=` is required.**
> `QdrantVectorStore` in LlamaIndex requires both a synchronous `QdrantClient` and an `AsyncQdrantClient` passed as the `aclient=` argument. Without the async client, async LlamaIndex query methods silently block — the script appears to hang with no error message. Always pass both.

If you are using OpenAI, monitor your usage dashboard. Ingesting 100 to 500 pages with `text-embedding-3-large` typically costs between $0.05 and $0.50 depending on average document density.

**Verify:** `curl http://127.0.0.1:6333/collections` — the JSON response lists your collection name(s).

## Step 5 — Ask your first question

Querying the RAG system sends a natural language question through LlamaIndex to retrieve semantically relevant chunks from Qdrant and synthesizes a grounded answer via the configured language model. The `query.py` script accepts a collection name and a quoted question string as positional arguments. Each response includes source citations with cosine similarity scores to verify the relevance of the generated content.

Pass the collection name and your question as arguments:

```bash
docker compose -f docker/docker-compose.yml --profile manual run --rm ingest python /app/scripts/query.py research_papers "What are the main findings on transformer attention mechanisms?"
```

Sample output:

```text
Query: What are the main findings on transformer attention mechanisms?
Collection: research_papers

Answer:
Transformer attention mechanisms enable models to weigh the relevance of different
input tokens relative to each output token. Self-attention computes query, key, and
value projections and aggregates context via scaled dot-product attention, allowing
parallelism absent in recurrent architectures.

Sources:
  [1] attention_is_all_you_need.pdf, page 4  (score: 0.87)
  [2] attention_is_all_you_need.pdf, page 6  (score: 0.81)
  [3] bert_paper.pdf, page 2                 (score: 0.74)
```

The scores represent cosine similarity on a 0-to-1 scale. A score of 1.0 is a mathematically perfect match. Scores above 0.5 are typically relevant to the question. Scores below 0.3 often represent noise — context that happened to share vocabulary without sharing meaning.

To query a different collection, replace the first argument. To query your notes instead of research papers:

```bash
docker compose -f docker/docker-compose.yml --profile manual run --rm ingest python /app/scripts/query.py notes "What was the main idea from last week's planning session?"
```

**Verify:** the output includes at least one source citation with a score >= 0.5.

## Step 6 — Schedule daily sync

Automating daily synchronization keeps the Qdrant vector collections current without requiring manual intervention after the first ingest. The `scripts/sync-rag.sh` wrapper script invokes `ingest.py` with the `--sync` flag to process only new files and remove vectors for deleted files. A cron entry running at a fixed time ensures fresh embeddings are available before the first query of each day.

Two update modes are available. A full reindex drops the Qdrant collection entirely and re-embeds all files — use this mode if you change `chunk_size` or `chunk_overlap` in `collections.yaml`, because vector dimensions and chunk boundaries are incompatible across different settings. The `--sync` mode is incremental: it detects new files, ingests them, identifies deleted files, and removes their vectors from Qdrant. Existing, unchanged documents are left untouched. Use `--sync` for routine daily updates to minimize API cost and processing time.

To set up the automated daily sync, open your crontab:

```bash
crontab -e
```

Add this line, replacing `/absolute/path/to/` with the actual absolute path to the repo on your machine:

```bash
0 3 * * * /absolute/path/to/rag-llamaindex-qdrant-docker/scripts/sync-rag.sh >> /var/log/rag-sync.log 2>&1
```

The wrapper script `sync-rag.sh` invokes `docker compose --profile manual run --rm ingest python /app/scripts/ingest.py --sync` internally, activates the manual profile so the one-shot ingest service runs, and resolves the path to the `docker` binary at runtime. Cron does not inherit your shell `PATH`, so the wrapper handles that detail in one place. The `>> /var/log/rag-sync.log 2>&1` suffix appends both standard output and errors to a log file for later inspection. Find the absolute path with `pwd` from inside the repo directory.

After saving, verify the entry was accepted:

**Verify:** `crontab -l` shows the new entry.

## Troubleshooting

Resolving common errors in a LlamaIndex and Qdrant pipeline requires checking connection strings, API credentials, and Python library configuration. Systematic diagnosis of the five most frequent failure modes covers Qdrant collection state, OpenAI API limits, async hangs, character encoding, and local-only setups. Each fix below resolves the root cause rather than suppressing the symptom.

### Qdrant collection not found error fix

The ingest script has not run yet, or the folder name in `documents/` does not match the `name:` field in `collections.yaml`. Run the ingest command from Step 4 to create the collection and populate it with vectors.

### OpenAI rate limit during ingest

The embedding API is receiving requests faster than your account tier allows. Add a short delay in `scripts/ingest.py` by inserting `import time` at the top and `time.sleep(0.5)` inside the file-processing loop to stay under the rate limit.

### Async LlamaIndex query hangs forever

The `aclient` argument is missing from the `QdrantVectorStore` constructor call inside `scripts/query.py`. Pass an `AsyncQdrantClient` instance as `aclient=AsyncQdrantClient(url="http://qdrant:6333")` alongside the standard synchronous `client=` argument.

### Surrogate character UnicodeEncodeError

Some PDFs embed non-standard characters that Python cannot encode into UTF-8 metadata. Sanitize the extracted text string before chunking with `text = text.encode('utf-8', 'ignore').decode('utf-8')`.

### Don't want to run Docker?

The Python scripts run natively without Docker if the dependencies are installed in a local virtual environment. Run `pip install -r docker/requirements.txt` inside a Python 3.12 virtual environment, then change `QDRANT_URL=http://qdrant:6333` to `QDRANT_URL=http://localhost:6333` in your `.env` (the `qdrant` hostname only resolves inside the Compose network), and call the scripts directly with `python scripts/ingest.py` and `python scripts/query.py`. You will also need a Qdrant instance running locally on port 6333 — start one with `docker compose -f docker/docker-compose.yml up -d qdrant` even when running the scripts on the host.

For the full list and supported file formats, see [`docs/troubleshooting.md`](docs/troubleshooting.md).

## Where to go next

Extending this tutorial involves three concrete improvements: replacing the plain-text API key with a secrets manager, exposing the RAG collections through an MCP server so AI agents can query them, and contributing fixes back to the canonical repository. Each improvement builds directly on the Docker and LlamaIndex foundation from this post.

The first next step is secret management. The `OPENAI_API_KEY` in your `.env` file is stored in plain text. Anyone who can read that file can run up charges on your account. Infisical is a self-hosted secrets platform that injects environment variables at runtime without ever storing them on disk. The companion post walks through adding it to this exact Docker stack in under 15 minutes.

The second next step is the Model Context Protocol. MCP is an open standard that lets AI agents — Claude Code, Gemini CLI, Cursor, and others — call your RAG system as a tool call rather than a raw HTTP request. Adding an MCP server in front of `query.py` means your agents can retrieve grounded answers from your private documents without writing any retrieval logic themselves.

The third next step is the repo itself. If a step in this tutorial did not work as described, open an issue. Reproducible bug reports and clean pull requests help every future reader.

- **Your API key is in plain text. Let's fix that → [Add Infisical for Secret Management](POST_2_URL)**
- **Want your AI agents to query this RAG directly? → [Add an MCP Server to Your RAG](POST_3_URL)**
- **Spotted an error or want to improve the tutorial? → [GitHub repo](https://github.com/<your-handle>/rag-llamaindex-qdrant-docker)**

## Licence and credits

Licence: MIT for code (see `LICENSE-CODE`), CC-BY-4.0 for prose (see `LICENSE-PROSE`). By Alec Silva Couto, 2026.

Contributions are welcome. Open an issue or submit a pull request via the GitHub repo above. Clear bug reports and well-scoped PRs make the tutorial better for every reader who follows after you.
