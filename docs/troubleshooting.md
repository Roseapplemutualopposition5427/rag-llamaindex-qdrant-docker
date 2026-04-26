# Troubleshooting

This reference manual provides solutions for common issues encountered when deploying the RAG pipeline using LlamaIndex, Qdrant, and Docker. Each section corresponds to an error pattern from the README troubleshooting section and goes deeper into diagnosis and resolution.

## Qdrant collection not found error fix

A `collection not found` error occurs when the application attempts to query a vector store that has not been initialized, or when the collection identifier does not match what exists in Qdrant. The two root causes are a mismatch between the `name` field in `collections.yaml` and the subdirectory name under `documents/`, or the ingestion script was never run for that collection.

To diagnose the state of the database, list all active collections:

```bash
curl http://127.0.0.1:6333/collections
```

The response is a JSON object containing the names of all existing collections. If the expected collection name is absent, check the `name` field in `collections.yaml` and confirm a matching folder exists under `documents/`. For example, `name: research_papers` requires a directory at `documents/research_papers/`.

Qdrant does not support in-place renaming of collections. If a collection was created with the wrong name, the correct procedure is: update the `name` field in `collections.yaml`, rename the corresponding folder under `documents/`, delete the stale collection via `curl -X DELETE http://127.0.0.1:6333/collections/<old-name>`, and re-run the ingest script to rebuild the index from scratch.

## OpenAI rate limit during ingest

When running `text-embedding-3-large` at high volume, the API returns a `RateLimitError` once the number of tokens per minute or requests per minute exceeds the quota assigned to the account tier. This is especially common during the first ingest of a large document set.

LlamaIndex includes built-in retry logic with exponential backoff. For transient spikes, the script will recover on its own — leave it running. For sustained overages, two adjustments help: lower the embedding batch size in `scripts/ingest.py` to reduce the burst size per request window, or process one collection at a time rather than all collections in a single run. To check the per-minute embedding cap on the account, visit `https://platform.openai.com/account/limits`. If limits cannot be raised, switch to the Ollama path (`bge-m3`) to avoid external API constraints entirely.

## Qdrant healthcheck fails on Apple Silicon

The default healthcheck in `docker-compose.yml` uses a bash `/dev/tcp` probe. On M-series Macs, Docker Desktop's networking layer can introduce cold-start timing variance that causes the probe to fail before Qdrant finishes initializing its storage engine and network listeners.

The most reliable fix is to add `start_period: 30s` to the healthcheck. This tells Docker to wait 30 seconds before counting failures, absorbing the startup delay without changing the probe frequency or retry count:

```yaml
healthcheck:
  test: ["CMD-SHELL", "bash -c ':> /dev/tcp/127.0.0.1/6333' || exit 1"]
  interval: 5s
  timeout: 3s
  retries: 5
  start_period: 30s
```

If the probe continues to fail after adding `start_period`, try replacing the probe with `wget` (note that `wget` is not guaranteed to be present in the `qdrant/qdrant` image, so treat this as a fallback):

```yaml
test: ["CMD-SHELL", "wget -qO- http://127.0.0.1:6333/ || exit 1"]
```

For most Apple Silicon setups, adding `start_period: 30s` alone resolves the issue without changing the probe command.

## Surrogate character UnicodeEncodeError

Processing PDFs that contain MathML or complex mathematical symbols can produce `UnicodeEncodeError: 'utf-8' codec can't encode character '\ud835'`. This error is caused by lone surrogate code points in the extracted text — values in the range `U+D800` to `U+DFFF` that are invalid in UTF-8.

The ingest script applies a sanitisation pass before chunking. The canonical clean operation strips surrogates by round-tripping through UTF-8 with the `ignore` error handler:

```python
clean_text = original_text.encode('utf-8', errors='ignore').decode('utf-8')
```

In LlamaIndex 0.14+, `Document.text` is managed by Pydantic and direct assignment fails. Use `model_copy` to produce a new document object with the sanitized text:

```python
sanitized = doc.text.encode('utf-8', errors='ignore').decode('utf-8')
clean_doc = doc.model_copy(update={'text': sanitized})
```

Pass `clean_doc` to the chunking and embedding steps instead of the original `doc`. This pattern is compatible with LlamaIndex's internal Pydantic registry and does not require subclassing.

## Async LlamaIndex query hangs forever

A query that runs indefinitely without producing output or an error message is almost always caused by a missing `aclient=` argument in the `QdrantVectorStore` constructor. LlamaIndex's async query path requires both a synchronous `QdrantClient` and an `AsyncQdrantClient` — if only the synchronous client is provided, the async event loop blocks waiting for a client that was never initialized.

Broken constructor — causes a silent hang on async queries:

```python
from qdrant_client import QdrantClient
from llama_index.vector_stores.qdrant import QdrantVectorStore

vector_store = QdrantVectorStore(
    client=QdrantClient(url="http://qdrant:6333"),
    collection_name="my_collection",
)
```

Working constructor — both clients passed:

```python
from qdrant_client import QdrantClient, AsyncQdrantClient
from llama_index.vector_stores.qdrant import QdrantVectorStore

vector_store = QdrantVectorStore(
    client=QdrantClient(url="http://qdrant:6333"),
    aclient=AsyncQdrantClient(url="http://qdrant:6333"),
    collection_name="my_collection",
)
```

Both `qdrant_client.QdrantClient` and `qdrant_client.AsyncQdrantClient` must be passed. Omitting `aclient=` produces no exception — the process simply hangs.

## Don't want to run Docker?

Use Qdrant's embedded mode to run the full pipeline without any Docker dependency. In embedded mode, `QdrantClient(path=...)` stores the vector index in a local directory instead of connecting to a networked service.

Set up a local Python environment and install dependencies:

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r docker/requirements.txt
```

Update the `make_clients()` function in `scripts/ingest.py` to use the embedded path:

```python
client = QdrantClient(path="./qdrant-data")
aclient = None  # embedded mode handles async operations synchronously
```

Embedded mode performs well for collections under 10,000 vectors. For larger datasets, run Qdrant in Docker (even if the scripts run on the host) and set `QDRANT_URL=http://localhost:6333` in `.env` — the hostname `qdrant` only resolves inside the Compose network.

## Supported file formats

`SimpleDirectoryReader` auto-routes files to the appropriate parser based on file extension.

In-scope formats (default configuration):

- `.pdf`
- `.txt`
- `.md`

Out of scope but straightforward to add: `.docx`, `.html`, `.json`, `.csv`. To add a format, install the matching `llama-index-readers-*` package from PyPI by adding it to `docker/requirements.txt`, then rebuild the container image with `docker compose -f docker/docker-compose.yml build ingest`. `SimpleDirectoryReader` will automatically detect the new capability and route those file types without additional code changes.

---

Still stuck? Open an issue at https://github.com/<your-handle>/rag-llamaindex-qdrant-docker/issues
