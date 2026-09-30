# RAGSearchJev Setup Guide

**Audience:** first-time local administrator  
**Repository:** [rahulparimal/RAGSearchJev](https://github.com/rahulparimal/RAGSearchJev)  
**Last checked:** 30 September 2026

> This guide follows the current `main` branch setup files. The workflow and configuration are verified against `README.md`, `docker-compose.yml`, `.env.example`, and `docs/ADMIN_GUIDE.md`. Real application screenshots are not embedded in this copy because a running browser session was not available when it was prepared. Screenshot capture points are called out in the relevant steps; capture them from your deployed instance so they reflect your exact build.

## 1. What you are setting up

RAGSearchJev (shown in the UI as Atlas Search Jev) is a local-first document search and grounded-answer application.

- **React web UI** for search, document administration, provider settings, and evaluation.
- **FastAPI** for authentication, document management, and search orchestration.
- **PostgreSQL** for users, document metadata, ACLs, chunks, and lexical search.
- **Qdrant** for dense vector search.
- **Redis + Celery** for asynchronous document indexing.
- **Sentence Transformers** for embeddings.
- **TypeSafe Jev** for optional typed decisions, including bounded scoring and routing.
- **Configurable OpenAI-compatible endpoint** for generated answers.

The search path combines dense and PostgreSQL full-text results with reciprocal rank fusion (RRF), optional Jev reranking, duplicate passage removal, and token-budgeted evidence packing. The advertised 20–40% context-token reduction is a target, not an established result; validate it with the benchmark steps in §9.

## 2. Prerequisites

Install:

- Docker Desktop or Docker Engine with the Compose plugin
- Git
- A terminal
- Optional: a TypeSafe API key and an OpenAI-compatible answer-generation endpoint

The initial build downloads application images and an embedding model. Allow several minutes and ensure the host has enough disk space. Keep the application local until you have configured non-default credentials and reviewed your network exposure.

## 3. Get the code and create environment settings

Clone the repository and enter it:

```bash
git clone https://github.com/rahulparimal/RAGSearchJev.git
cd RAGSearchJev
cp .env.example .env
```

Edit `.env` before first startup. At minimum, replace the database password, application secret, and administrator password.

Generate a strong application secret:

```bash
python3 -c 'import secrets; print(secrets.token_urlsafe(48))'
```

Use the output as `APP_SECRET_KEY`. Keep it safe and stable: it protects signing and encryption of saved provider settings. Losing it can make stored provider settings unreadable; rotating it invalidates existing sessions and requires a planned re-encryption approach.

Use unique values for `POSTGRES_PASSWORD` and `ADMIN_PASSWORD`. Do not commit `.env` or share it in screenshots. The values in `.env.example` are examples and are not safe for a deployed environment.

### Environment variable reference

| Variable | Required? | Purpose |
|---|---:|---|
| `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` | **Required** | PostgreSQL bootstrap database and credentials. Set a unique password. |
| `DATABASE_URL` | **Required** | SQLAlchemy connection used by API and worker. Keep credentials consistent with the PostgreSQL variables. Compose default uses service hostname `postgres`. |
| `APP_SECRET_KEY` | **Required** | JWT signing and encryption key material for saved AI provider settings. Use a unique random value. |
| `ADMIN_USERNAME`, `ADMIN_PASSWORD` | **Required on first setup** | Bootstrap administrator. Use a strong password of at least 12 characters. |
| `QDRANT_URL` | **Required** | Qdrant endpoint. Compose default: `http://qdrant:6333`. |
| `REDIS_URL` | **Required** | Celery broker. Compose default: `redis://redis:6379/0`. |
| `EMBEDDING_MODEL` | **Required** | Sentence Transformers embedding model name. Default: `sentence-transformers/all-MiniLM-L6-v2`. |
| `EMBEDDING_DIMENSION` | **Required** | Vector size matching the selected embedding model. Default: `384`. Do not change without creating a compatible collection and reindexing. |
| `CORS_ORIGINS` | **Required for hosted use** | Allowed frontend origin. Local default: `http://localhost:5173`. Restrict to the actual UI origin in a deployment. |
| `TYPESAFE_API_KEY` | Optional | Jev key. Search can run without it; Jev features are optional. |
| `JEV_MODEL`, `JEV_BASE_URL` | Optional | Jev model and API base URL. Default model: `jev-latest`; base URL: `https://api.typesafe.ai`. |
| `JEV_RERANK_ENABLED` | Optional | Enables bounded Jev Score reranking when configured and evaluated. Default: `false`. |
| `JEV_CHUNK_PROFILE_ENABLED` | Optional | Enables Jev-assisted chunk-profile choice. Default: `false`. |
| `LLM_BASE_URL`, `LLM_MODEL` | Optional | OpenAI-compatible chat endpoint and model for generated answers. |
| `LLM_API_KEY` | Optional | Answer provider key; some local or managed identity endpoints may not need one. |
| `LLM_CONTEXT_BUDGET_TOKENS` | Optional | Evidence context budget. Default: `2800`. |
| `MAX_UPLOAD_MB` | Optional | Upload limit. Default: `50`. |
| `UPLOAD_DIR` | Deployment-specific | Upload location if supported by the backend configuration. The Compose volume is mounted at `/app/uploads`. |

## 4. Start the application

From the repository root:

```bash
docker compose up --build
```

The Compose stack starts PostgreSQL, Qdrant, Redis, the FastAPI API, the Celery worker, and the web UI. Keep this terminal open to see startup messages. On a later restart, use:

```bash
docker compose up -d
docker compose ps
```

Open:

- **Web UI:** http://localhost:5173
- **API:** http://localhost:8000

Wait until the services are healthy and the worker is running. The first embedding-model download may take several minutes.

**Snapshot 1 to capture:** browser showing the sign-in page at localhost:5173. Crop out browser profile/account information.  
**Snapshot 2 to capture:** terminal output from `docker compose ps` showing the service names and running state. Do not include environment values.

## 5. Sign in and verify admin access

Sign in with the `ADMIN_USERNAME` and `ADMIN_PASSWORD` values set in `.env`. The bootstrap administrator is created only if the database does not already contain a user with that username. Changing those environment values after initial creation does not automatically rotate the existing account password.

After sign-in, confirm that the Admin console is available. If this is not a new database and the credentials do not work, review the account state using the application’s supported account administration workflow rather than deleting persistent database volumes.

**Snapshot 3 to capture:** successful sign-in and the Admin console navigation. Blur usernames or other account information before sharing.

## 6. Configure providers and retrieval

Open **Admin console → AI configuration**. Settings are grouped into tabs so secrets and model parameters are not presented as one long form.

### TypeSafe Jev tab

1. Enter the TypeSafe API key if you plan to use Jev decisions.
2. Confirm the API base URL and model.
3. Save the settings.
4. Leave optional chunk-profile selection and reranking disabled until you have approved the data handling and evaluated them.

Jev receives text samples or candidate/query text for the enabled decision. Do not enable optional Jev processing for sensitive content until external processing is approved.

### Answer LLM tab

1. Select or type the OpenAI-compatible endpoint.
2. Enter the model identifier.
3. Add an API key if your endpoint requires one.
4. Save and run a small grounded query.

A local or managed identity endpoint may not require a key. Blank key fields preserve an already saved key. Saved keys are not displayed back to the browser.

### Retrieval tab

Review the embedding model, vector dimension, and context budget. Changing the embedding model or dimension is a deployment operation: ensure the vector collection uses the matching dimension and reindex documents after changes.

**Snapshot 4 to capture:** AI configuration with the tab names visible. Use a test environment; blur key inputs and never show a real API key.  
**Snapshot 5 to capture:** Retrieval tab showing model, dimension, and context budget, with no secrets visible.

## 7. Upload and index a document

1. Open **Admin console → Add a source**.
2. Choose a supported file: text-based PDF, DOCX, TXT, Markdown, or HTML.
3. Enter a display name and set the access list.
4. Use `*` for all signed-in readers, or enter a comma-separated list of usernames.
5. Submit the upload.
6. Wait for indexing status to change from `processing` to `ready`.
7. If status becomes `failed`, inspect worker logs.

The worker extracts text, splits it into structure-aware chunks, creates embeddings, and indexes the document. Search results are limited to ready documents and the caller’s document access.

Scanned PDFs currently need OCR, which is not integrated. PPTX support, document replacement/version history, tenant management, and recovery-grade object storage are not included in this MVP.

**Snapshot 6 to capture:** Add a source form with a non-sensitive sample document selected; do not reveal confidential file names.  
**Snapshot 7 to capture:** document list showing a sample upload as `ready`.  
**Snapshot 8 to capture:** a search result with citation/source snapshot preview expanded. Use only a public or synthetic document.

## 8. Run a search and inspect evidence

1. Open the search screen.
2. Ask a question that the uploaded sample document answers directly.
3. Inspect the returned passages and citations.
4. Open the source snapshot and verify the cited content supports the answer.
5. Try a question not covered by the corpus and confirm the application does not present unsupported claims as grounded facts.

The retrieval system uses dense and lexical search, then merges rankings. Optional Jev scoring is bounded to a shortlist. Evidence is de-duplicated and packed under a context budget before answer generation.

**Snapshot 9 to capture:** a successful query with result cards and source citation.  
**Snapshot 10 to capture:** expanded source snapshot showing the evidence passage, with any sensitive material redacted.

## 9. Evaluation snapshots and token-efficiency measurement

A token-reduction estimate in the UI is not a measured quality result. Use the same corpus snapshot and labeled query set for baseline and optimized runs.

1. Copy `scripts/evaluation_template.jsonl` and create a representative gold set with human-validated relevant document IDs.
2. Sign in and obtain an administrator bearer token using the documented API flow.
3. Set the token in the shell and run:

```bash
export RAGSEARCH_TOKEN='YOUR_BEARER_TOKEN'
python3 scripts/evaluate.py --dataset my-gold-set.jsonl --api http://localhost:8000
```

4. Review the timestamped files written under `evaluation-runs/`.
5. Compare Recall@20, MRR, nDCG@20, retrieved IDs/ranks, and estimated token reduction for equivalent settings.
6. Separately measure actual provider input tokens, answer citation support, p95 latency, and spend. The current runner’s token estimate is a proxy; it does not establish real API token savings or answer quality.

**Snapshot 11 to capture:** evaluation run summary with dataset names and tokens removed.  
**Snapshot 12 to capture:** a comparison table or chart from two runs using the same corpus version and labels.

## 10. Common operational checks

View service status:

```bash
docker compose ps
```

Inspect logs:

```bash
docker compose logs --tail=200 api worker postgres qdrant redis
```

Stop the stack while preserving named volumes:

```bash
docker compose down
```

Do not use `docker compose down -v` unless you intend to remove persistent database, Qdrant, and uploaded-file data.

| Symptom | Checks |
|---|---|
| UI does not load | Check `docker compose ps`, then `docker compose logs web api`; confirm port 5173 is available. |
| API does not start | Check `APP_SECRET_KEY`, database URL/credentials, and API logs. The API rejects missing, weak, or example signing secrets. |
| Admin login fails | Confirm the first-run credentials and whether this database already has a bootstrap user. Environment password changes do not rotate an existing account. |
| Upload remains processing | Check worker and Redis logs; confirm worker can reach PostgreSQL, Redis, and Qdrant. |
| Upload fails | Check file type, text extraction, file size, and worker error. Scanned PDFs need OCR. |
| No search results | Confirm document status is `ready`, the access list includes the signed-in user, and the embedding model/dimension match the index. |
| No generated answer | Configure a compatible answer model endpoint; verify key and model ID. Retrieval can work without answer generation. |
| Saved provider settings unreadable | Restore the original `APP_SECRET_KEY`; do not rotate it casually. |

## 11. Security and deployment boundary

This repository is an MVP. Before exposing it beyond a trusted local environment:

- Use TLS and private networking for PostgreSQL, Qdrant, and Redis.
- Restrict CORS to the intended frontend origin.
- Move production secrets to an approved secret manager and rotate them with a re-encryption plan.
- Add enterprise identity/SSO, MFA, password reset/deactivation, CSRF controls, signed downloads, and complete audit retention.
- Configure encrypted, tested backups for PostgreSQL, Qdrant, and uploaded originals.
- Review ACL and data-retention behavior with representative users and documents.
- Approve any external processing by Jev or the answer model provider.
- Complete API, integration, restore, performance, and security tests before production use.

## 12. Screenshot checklist

Capture screenshots from the running build and replace this checklist with the images before circulating a fully illustrated operator guide:

1. Docker services running.
2. Sign-in page.
3. Admin console navigation.
4. TypeSafe Jev settings tab (no key visible).
5. Answer LLM settings tab (no key visible).
6. Retrieval settings tab.
7. Upload form using a synthetic/public file.
8. Document processing and ready states.
9. Search result with citation.
10. Expanded source snapshot.
11. Evaluation run summary and same-corpus comparison.

Never include API keys, passwords, bearer tokens, private document contents, or personal user data in screenshots.
