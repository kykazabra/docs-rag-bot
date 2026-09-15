# Docs RAG Bot

A Telegram bot that answers questions about **your own documents**. Upload a PDF or DOCX,
switch between uploaded files, ask questions about the active one.

Each document becomes its own vector collection, so answers never mix sources: when you
ask about a contract you get the contract, not a paragraph from last year's report that
happened to be semantically close.

```
docker run --env-file .env kykazabra/file_rag_bot:latest
```

## How it works

```
upload ──► text extraction (PyPDF2 / python-docx)
             │
             └─► RecursiveCharacterTextSplitter (1000 / 200)
                      │
                      └─► OpenAI embeddings ──► Chroma collection (one per document)

question ──► retrieve top-3 chunks from the active collection ──► LLM ──► answer
```

Chat state keeps which collection is active per user, so several documents can live side
by side and the user switches between them with a keyboard.

## Design decisions

**One collection per document, not one shared index.** A shared index is cheaper and scales
better, but for a personal document assistant it produces the worst possible failure: a
confident answer sourced from the wrong file. Per-document collections make the scope of
every answer explicit and let a document be deleted by dropping its collection.

**Chunks of 1000 with 200 overlap.** Large enough that a clause or a paragraph survives
intact, overlapping enough that a fact split across a boundary is still retrievable from at
least one chunk. Smaller chunks retrieved cleaner but lost the context needed to answer
"why" questions.

**Top-3 retrieval.** With per-document collections the candidate pool is already narrow, so
pulling more chunks mostly adds noise and tokens rather than recall.

**"Answer only from the context, otherwise say you don't know."** The prompt refuses rather
than improvises. For a document assistant a missing answer is recoverable; an invented one
is not — the user has no way to tell it apart from a real quote.

## Quickstart

```bash
pip install -r requirements.txt
cp .env.example .env        # OPENAI_API_KEY, TELEGRAM_TOKEN
python bot.py
```

Or use the published image: `kykazabra/file_rag_bot:latest`.

## Limitations

- PDF and DOCX only; scanned documents need OCR first (none is wired in).
- Collections live on local disk under `collections/` — no shared storage, one instance.
- Model, embedding model, chunk size and retrieval depth are currently constants at the top
  of `bot.py` rather than configuration.
