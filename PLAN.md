# Local Knowledge Engine — Build Plan

Status: plan only. Do not implement until this document is accepted.

Repo: https://github.com/veraycent/local-knowledge-engine

This repository will become the reusable **base app**. You clone or fork it, feed it a zip of source files, and query that corpus through a UI. Each product (game wiki Q&A, SOP assistant, contract helper) is a fork or a use-case config on top of the same engine — not a rewrite.

---

## 1. Goal

Ship one local, free, offline-capable app that can:

1. Create a named **collection** (a knowledge base).
2. Accept source data as a **zip** of PDFs, text/markdown, Office files, and images.
3. Store originals on disk and extract searchable text.
4. Index that text locally.
5. Answer free-form questions in a UI, with **citations** back to file and page.
6. Be forked into custom Q&A apps by changing prompts and content, not core plumbing.

First planned fork: a **game knowledge base** that answers questions from manuals, patch notes, wikis exported to files, item lists, and screenshots.

---

## 2. Non-goals for v1

- Cloud LLMs or paid APIs
- Multi-user auth / SaaS tenancy
- Live web scraping of a game wiki
- Fine-tuning a model on the corpus
- Agents, tool-calling, or multi-collection routing
- Mobile apps
- Perfect OCR on every scanned image

Those can be later forks. The base app only needs ingest + retrieve + answer + cite.

---

## 3. Constraints

| Constraint | Decision |
|---|---|
| Cost to develop | $0 — open-source stack, LLM-written code |
| Cost to run | $0 beyond the machine you already own |
| Network | Online only to install packages and download models once |
| Privacy | Documents and queries never leave the machine |
| Licence target | MIT app code; prefer Apache/MIT models and libraries |
| Hardware | Must run on a laptop CPU; GPU optional and faster |

---

## 4. How forks / use cases will work

Keep the engine generic. Specialise with config and data, not forks of business logic — unless a use case truly needs new features.

```
local-knowledge-engine          ← this repo (base)
        │
        ├── use_cases/default.yaml
        ├── use_cases/game_kb.yaml      ← first customisation
        └── data/                       ← not committed; local only
```

A use-case YAML will hold:

- Display name and description
- System prompt (tone, what to do when unsure, game-specific rules)
- Retrieval settings (top_k, temperature)
- Suggested collection name

Workflow for the game app later:

1. Clone this repo (or fork it).
2. Copy `use_cases/game_kb.yaml` and edit the prompt for that game.
3. Zip the game files (guide PDFs, item tables, lore notes, map screenshots).
4. Create a collection, upload the zip, ask questions.

Do not bake a specific game into the base engine.

---

## 5. Architecture

Three stores, always:

1. **Raw files** — unzipped originals (`data/raw/{collection_id}/`)
2. **Extracted text** — markdown/text + metadata sidecars (`data/extracted/...`)
3. **Vector index** — embeddings per collection (`data/indexes/...`)

Plus SQLite for collections, documents, jobs, and chat history (`data/app.db`).

```
Browser UI
    │
FastAPI
    ├── Collections API
    ├── Ingest job API   (zip → unpack → parse → chunk → embed)
    └── Query API        (embed question → retrieve → local LLM → answer + sources)
    │
Ollama (local)
    ├── embedding model
    └── chat model
    │
Chroma or LanceDB (local files)
SQLite
Disk
```

Ingest is a **background job**. The UI polls progress. A large zip must not block the chat.

---

## 6. Stack (locked for v1)

All free, all local.

| Layer | Choice | Fallback |
|---|---|---|
| Language | Python 3.11+ | — |
| API | FastAPI + Uvicorn | — |
| UI | Single-page app served by FastAPI (`app/static`) | no separate Node build in v1 |
| Chat + embeddings | Ollama | refuse to start query if Ollama is down, with a clear error |
| Embedder | `nomic-embed-text` | configurable in `.env` |
| Chat model | Small instruct model (Qwen3 8B / 4B or Phi-4) | configurable in `.env` |
| Parser | Built-in adapters first: text/markdown, pypdf, python-docx | optional Docling extra for hard PDFs, Office, images |
| Images in v1 | Store original; extract text if Docling/OCR is available; otherwise index filename + note | multimodal RAG later |
| App DB | SQLite | — |
| Vectors | Chroma persistent client | LanceDB if Chroma becomes a pain |
| Config | `.env` + `use_cases/*.yaml` | — |

Do **not** use OpenAI, Anthropic, Pinecone, LlamaParse cloud, or any “cloud fallback.”

---

## 7. Data model

**collections**  
id, name, description, use_case, embed_model, chat_model, created_at

**documents**  
id, collection_id, original_filename, rel_path, mime, sha256, byte_size, status, error, page_count

**chunks** (optional in SQLite; Chroma is source for vectors)  
id, document_id, collection_id, text, page, heading, chunk_index

**jobs**  
id, collection_id, type (`ingest` / `reindex`), status, progress_pct, message, error, created_at, finished_at

**messages**  
id, collection_id, role, content, sources_json, created_at

Vector metadata on every chunk: `collection_id`, `document_id`, `filename`, `page`, `chunk_index`.

---

## 8. API contract (freeze this)

```
POST   /api/collections
GET    /api/collections
GET    /api/collections/{id}
DELETE /api/collections/{id}

POST   /api/collections/{id}/ingest          # multipart zip
GET    /api/jobs/{id}

GET    /api/collections/{id}/documents
GET    /api/documents/{id}/file              # original download

POST   /api/collections/{id}/query
GET    /api/collections/{id}/messages

GET    /api/health                           # disk + Ollama + models
GET    /api/use-cases                        # list yaml configs
```

Query request:

```json
{
  "question": "How does parry timing work?",
  "top_k": 8
}
```

Query response:

```json
{
  "answer": "...",
  "sources": [
    { "document_id": "...", "filename": "combat.pdf", "page": 12, "snippet": "..." }
  ]
}
```

---

## 9. Ingest pipeline

1. Save zip under `data/uploads/`.
2. Unzip into `data/raw/{collection_id}/`, skip junk (`.DS_Store`, `__MACOSX`, executables).
3. Classify by extension.
4. Parse:
   - `.txt` `.md` `.csv` `.json` — read as text
   - `.pdf` — pypdf (text layer); Docling if enabled / scanned
   - `.docx` — python-docx
   - `.png` `.jpg` `.webp` `.tif` — optional OCR; always keep the file
5. Write extracted text + JSON sidecar (parser used, page count, hash).
6. Skip unchanged files via sha256.
7. Chunk ~400–512 tokens, ~10–15% overlap, split on headings then paragraphs.
8. Embed with the collection’s embedder. Never mix embedders in one collection.
9. Upsert into that collection’s vector space.
10. Mark job complete; list per-file failures without aborting the whole zip.

---

## 10. Query pipeline

1. Load the collection’s use-case prompt.
2. Embed the question with the **same** embedder used at ingest.
3. Retrieve top_k chunks from that collection only.
4. Call Ollama with low temperature (0.1–0.2).
5. System rules: answer only from passages; if missing, say so; cite filename and page; do not invent game mechanics.
6. Persist the turn and return sources to the UI.

V1 retrieval is dense vectors only. Hybrid search (BM25 + vectors) and a reranker are v1.1.

---

## 11. UI (v1)

One screen:

- Left: collection list, create collection, pick use-case YAML
- Top of main pane: upload zip, ingest progress, document list
- Centre: chat
- Under each assistant message: source chips (file + page). Clicking a source downloads or previews the original when practical

No login. Bind to localhost by default.

---

## 12. Proposed repo layout

```
local-knowledge-engine/
  PLAN.md
  README.md
  LICENSE
  .env.example
  .gitignore
  requirements.txt
  requirements-docling.txt          # optional heavy extra
  app/
    main.py
    settings.py
    db.py
    api/
      collections.py
      ingest.py
      query.py
      health.py
    services/
      ingest.py
      parse.py
      chunk.py
      embed.py
      retrieve.py
      generate.py
      ollama.py
    static/
      index.html
      app.js
      styles.css
  use_cases/
    default.yaml
    game_kb.yaml
  examples/
    game-kb/
      README.md                     # what files to zip, sample questions
  tests/
    test_chunk.py
    test_parse_text.py
  data/                             # gitignored
```

---

## 13. First use case: game knowledge base

File: `use_cases/game_kb.yaml`

Responsibilities of the YAML, not the engine:

- Speak as a game guide for *whatever game the collection contains*
- Prefer official / uploaded sources over general training knowledge
- Separate mechanics, lore, and spoilers when the user asks
- If patch notes and a guide disagree, say so and cite both
- Never invent drop rates, stat numbers, or quest steps

Example source zip the user will supply later (not in the base repo):

- Digital artbook / manual PDF
- Exported wiki pages as markdown
- Item / recipe tables as CSV
- Patch notes as `.txt`
- Screenshots of skill trees or maps (best-effort until OCR extra is on)

Example questions the game fork should handle:

- “How do I craft X?”
- “What resists does boss Y have?”
- “What’s the difference between build A and build B according to these notes?”
- “Where is item Z mentioned?”

The base README will tell people: put those files in a zip, create a collection using `game_kb`, upload, ask.

---

## 14. Build phases

### Phase 0 — this document
Plan only. Repo exists with README + PLAN.md.

### Phase 1 — skeleton
- FastAPI app boots
- SQLite schema
- Health endpoint checks Ollama
- Empty UI shell
- `.env.example`, gitignore, MIT licence

### Phase 2 — ingest text + PDF
- Zip upload job
- Parse txt/md/pdf
- Chunk + Chroma + Ollama embeddings
- File list + hashes

### Phase 3 — query
- `/query` with citations
- Chat UI wired up
- `default.yaml` prompt

### Phase 4 — mixed files + use cases
- docx
- optional Docling extra
- image originals stored
- `game_kb.yaml` + `examples/game-kb/README.md`

### Phase 5 — harden
- Per-file ingest errors
- Re-ingest / delete document
- Basic tests
- README: install Ollama, pull models, run, upload zip, fork for a new product

Stop after Phase 5 before adding agents, hybrid search, or a second frontend.

---

## 15. Local setup the README will document

```bash
# once, while online
ollama pull nomic-embed-text
ollama pull qwen3:8b          # or a smaller model if RAM is tight

python -m venv .venv
source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Open `http://127.0.0.1:8000`. Create a collection, upload a zip, ask a question.

---

## 16. Success criteria for the base app

The base is done when all of these work on a laptop with Ollama installed:

- [ ] Create two collections without them mixing answers
- [ ] Upload a zip of at least one PDF and two markdown files
- [ ] Job shows progress and finishes
- [ ] A question whose answer is in the PDF returns that fact plus filename and page
- [ ] A question with no support in the files is refused rather than invented
- [ ] Switching the collection to `game_kb` changes the assistant behaviour
- [ ] No cloud API key is required at any point
- [ ] `data/` is gitignored so game files never get committed by accident

---

## 17. Risks

| Risk | Mitigation |
|---|---|
| Scanned PDFs / screenshots yield empty text | Phase 4 optional Docling; v1 still stores the file |
| Weak laptop + 8B model is slow | Document 4B/Phi alternatives in `.env.example` |
| Changing embedder silently breaks search | Store embedder name on the collection; block mixed use |
| Game community wikis are messy HTML | v1 accepts exported md/pdf only; no crawler |
| Hallucinated game mechanics | Strict grounded prompt + citations in the UI |

---

## 18. Decision log

- Base repo name: `local-knowledge-engine`
- Owner: `veraycent`
- One engine, many use-case YAMLs
- First use-case file: `game_kb.yaml`
- Implementation starts only after this plan is approved
