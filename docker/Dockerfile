FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# System deps for PDF extraction (LlamaIndex SimpleDirectoryReader uses pypdf;
# tesseract optional for scanned PDFs)
RUN apt-get update && apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
        tesseract-ocr \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r /app/requirements.txt

# Scripts and config are bind-mounted at runtime via docker-compose
CMD ["python", "/app/scripts/ingest.py", "--help"]
