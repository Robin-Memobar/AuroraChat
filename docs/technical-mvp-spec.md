# Technical MVP Specification: Codex Data + Chat Workflows

## Overview
This document describes the minimum viable product (MVP) workflows required to ingest website content into AuroraChat and power question-answering experiences for tenant chatbots. The solution combines n8n for orchestration, Supabase for persistence, OpenAI for embeddings and language model completions, and Scrappey for website scraping.

---

## System Components

| Component | Responsibilities |
| --- | --- |
| **n8n (Core Orchestrator)** | Hosts inbound webhooks, orchestrates workflow execution, performs data shaping, and coordinates integrations with external APIs and Supabase. |
| **Supabase (Postgres + pgvector)** | Stores tenants, raw document chunks, and embedding vectors. Exposes SQL functions (e.g., `search_docs`) for semantic retrieval. |
| **OpenAI API** | Generates embeddings for both documents and user questions. Serves chat completions that produce natural-language answers grounded in retrieved context. |
| **Scrappey API** | Crawls tenant websites, returning structured page content for downstream normalization and chunking. |

---

## Workflow 1: Onboarding (Website → Docs + Embeddings)

**Trigger**
- Webhook Node receives `POST /onboard` with payload `{ "tenant_id": "uuid", "domain": "https://example.com" }`.

**Flow**
1. **Prep (Function Node)**
   - Normalize the domain URL.
   - Persist `tenant_id` in `$flow` variables for use by subsequent nodes.

2. **HTTP Request (Scrappey API)**
   - Endpoint: `POST https://api.scrappey.com/scrape`
   - Body: `{ "url": <domain>, "depth": 2, "extractText": true }`
   - Response: structured scrape result with page metadata and text.

3. **Normalize Pages (Function Node)**
   - Input: Scrappey payload.
   - Transform into array of `{ tenant_id, source_url, title, text }` objects.

4. **Chunk Content (Function Node)**
   - Segment long page text into ~1,200-character chunks with 200-character overlap to preserve continuity.
   - Output: `{ tenant_id, source_url, title, content, chunk_index }` per chunk.

5. **Insert Docs (Postgres Node)**
   - Upsert or insert each chunk into `docs` table with fields `tenant_id`, `source_url`, `title`, `content`, `chunk_index`.

6. **Select Doc (Postgres Node)**
   - Retrieve `doc_id` of the newly inserted chunk for relational linking.

7. **OpenAI Embedding (HTTP Request Node)**
   - Endpoint: `POST https://api.openai.com/v1/embeddings`
   - Body: `{ "model": "text-embedding-3-small", "input": <chunk content> }`
   - Response: vector embedding for the chunk.

8. **Insert Embedding (Postgres Node)**
   - Insert into `doc_embeddings` table with fields `tenant_id`, `doc_id`, `chunk_index`, `embedding` (pgvector column).

**Outputs**
- Populated `docs` and `doc_embeddings` tables keyed by tenant.
- Chunks ready for semantic retrieval during chat interactions.

---

## Workflow 2: Chatbot (Question → Context → Answer)

**Trigger**
- Webhook Node receives `POST /chatbot` with payload `{ "tenant_id": "uuid", "question": "<user question text>" }`.

**Flow**
1. **OpenAI Embedding (HTTP Request Node)**
   - Generate embedding vector for user question.

2. **Search Docs (Postgres Node)**
   - Execute SQL function:
     ```sql
     select d.id, d.content, d.source_url, d.title, d.chunk_index
     from search_docs($1, $2, 5) as d
     ```
   - Parameters: tenant_id, question embedding, `k = 5`.
   - Output: Top 5 semantically related document chunks.

3. **Build Prompt (Function Node)**
   - Concatenate retrieved chunks into context string including source metadata.
   - Prepare structured prompt: `Question: <user question>\nContext: <top-k docs>`.

4. **OpenAI Chat Completion (HTTP Request Node)**
   - Endpoint: `POST https://api.openai.com/v1/chat/completions`
   - Body:
     ```json
     {
       "model": "gpt-4o-mini",
       "messages": [
         {"role": "system", "content": "You are a helpful assistant answering questions based on company docs."},
         {"role": "user", "content": "Question: <user question> \nContext: <top-k docs>"}
       ]
     }
     ```
   - Response: GPT-formatted answer grounded in provided context.

5. **Respond Node**
   - Return JSON response `{ "answer": <GPT reply>, "sources": [<source_url>...] }`.

**Outputs**
- Chat response that cites supporting document URLs.
- Logging/analytics hooks can be added via additional n8n nodes if desired.

---

## Data Model & Persistence

### Tables
- **`docs`**: Stores normalized content chunks.
  - Columns: `id`, `tenant_id`, `source_url`, `title`, `content`, `chunk_index`, timestamps.
- **`doc_embeddings`**: Stores embedding vectors aligned with `docs` rows.
  - Columns: `id`, `tenant_id`, `doc_id`, `chunk_index`, `embedding` (`vector`), timestamps.

### Search Function
- **`search_docs(tenant_id uuid, embedding vector, k integer)`**
  - Returns top-`k` relevant document chunks by cosine similarity.
  - Implementation uses pgvector indexing (e.g., `ivfflat`) scoped per tenant.

---

## Operational Considerations

- **Authentication & Secrets**: Store API keys (OpenAI, Scrappey, Supabase) as n8n credentials. Use Supabase Row Level Security (RLS) to isolate tenant data.
- **Error Handling**: Capture failures in scraping, embeddings, or database operations using n8n error workflows or retry logic.
- **Scalability**: Batch inserts for chunk loading, leverage Supabase connection pooling, and consider queueing for large sites.
- **Monitoring**: Instrument n8n executions and Supabase logs to track onboarding success and chatbot usage.

---

## End-to-End Flow Summary
1. Tenant initiates onboarding; website pages are scraped, normalized, chunked, and stored with embeddings.
2. Chatbot receives a question, embeds it, retrieves semantically similar chunks, and prompts GPT with contextual information.
3. GPT response is returned alongside source URLs, enabling grounded question answering per tenant.

