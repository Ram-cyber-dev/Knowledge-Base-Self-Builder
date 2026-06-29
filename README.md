# 🧠 Knowledge Base Self-Builder

A multi-agent RAG (Retrieval-Augmented Generation) system with knowledge graph extraction, built in n8n. Drop a document into a Google Drive folder — it gets read, categorized, mapped into a knowledge graph, checked for contradictions, scored for confidence, and embedded into a searchable vector brain. Then anyone can ask questions about it in plain English and get cited, hallucination-free answers.

Built as part of the Haus of Intelligence (HOI) 14-week cohort program.

## The Problem

Every business has knowledge trapped in PDFs, WhatsApp forwards, and the owner's head. Staff ask the same questions repeatedly — "who approves a refund?", "what's the hygiene checklist?", "where's the supplier list?" The answers exist, but nobody can find them fast enough. This system fixes that permanently, for any business, one document at a time.

## Architecture — Two Halves Working Together

### The Builder (runs on autopilot)
Watches a Google Drive folder. The moment a new file lands, it automatically:
1. Detects the file type (PDF, DOCX, text, image, audio)
2. Extracts the text
3. Runs a **Categorizer Agent** (GPT-4o-mini) — labels the doc with title, category, subcategory, summary, tags
4. Runs a **Relationship Mapper Agent** (GPT-4o-mini) — extracts entities and relationships into a knowledge graph
5. Checks the new content against the existing knowledge base for **contradictions** before saving
6. Scores the document's **confidence** (0-100) based on extraction quality and contradiction status
7. Saves it three ways: a structured index row, a vector embedding (semantic search), and a knowledge graph (relationship search)
8. A **Gap Finder Agent** reports what knowledge is still missing

### The Librarian (answers on demand)
A chat interface where anyone can ask a question in plain English. The agent:
- Decides whether the question needs semantic search (vector brain) or relationship search (knowledge graph)
- Searches accordingly
- Replies with a citation — never invents an answer

## The 5 Enhancements

| # | Enhancement | What it does |
|---|---|---|
| 1 | **Multi-Format Ingestion Engine** | Auto-detects file type (PDF, DOCX, text, image, audio) before processing, so the system can handle whatever a real business actually has, not just clean PDFs |
| 2 | **Contradiction Detector** | Before saving a new document, an AI checks it against the existing knowledge base for conflicting information and flags it |
| 3 | **Confidence Scoring + Citation Chain** | Every document gets a 0-100 confidence score based on extraction completeness and contradiction status, banded into high/medium/low |
| 4 | **Knowledge Decay Alert** | A daily scheduled check flags documents older than 30/60/90 days as potentially outdated, alerting the owner to review them |
| 5 | **Weekly Intelligence Digest** | Every Monday, an AI analyst reviews the entire knowledge base and reports on health score, coverage gaps, and recommended additions |

## Tech Stack

- **n8n** — workflow orchestration (self-hosted)
- **OpenAI GPT-4o-mini** — categorization, relationship mapping, contradiction detection, Q&A
- **OpenAI text-embedding-ada-002** — vector embeddings
- **Supabase** — pgvector for semantic search + relational table for structured index and knowledge graph
- **Google Drive** — file ingestion trigger
- **Telegram** — decay alerts and weekly digest delivery

## Supabase Setup

Two tables are required. Run this in your Supabase SQL Editor:

```sql
-- Enable vector extension and create the vector store
create extension if not exists vector;

create table if not exists documents (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(1536)
);

create or replace function match_documents (
  query_embedding vector(1536),
  match_count int default 5,
  filter jsonb default '{}'
) returns table (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
language plpgsql
as $$
begin
  return query
  select
    documents.id,
    documents.content,
    documents.metadata as metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  from documents
  where documents.metadata @> filter
  order by documents.embedding <=> query_embedding
  limit match_count;
end;
$$;

-- Create the structured knowledge index
create table if not exists knowledge_index (
  id bigserial primary key,
  title text,
  category text,
  subcategory text,
  summary text,
  tags text[],
  entities jsonb,
  relationships jsonb,
  source_file text,
  file_id text,
  added_at timestamptz default now(),
  last_verified_at timestamptz default now(),
  confidence_score float default 1.0,
  has_contradictions boolean default false
);
```

You'll also need a `search_graph` RPC function if you want full knowledge graph traversal support for the Librarian's graph search tool.

## Known Limitation

The Telegram delivery nodes (Decay Alert, Weekly Digest) can intermittently throw `ETIMEDOUT` errors connecting to Telegram's IPv6 endpoints, depending on network/ISP routing. This is a network-layer issue, not a workflow bug — the AI logic, contradiction detection, and confidence scoring all complete successfully regardless. Both Telegram nodes have "Continue On Fail" enabled so a delivery failure never blocks the rest of the pipeline. If you hit this, try a different network or enable retries.

## Real Use Cases

| Business | What gets stored | What gets answered |
|---|---|---|
| HVAC company | Technician manuals, pricing, warranty terms | "What's the warranty on Daikin units?" |
| Restaurant | SOPs, supplier lists, staff policies | "Who approves a refund?" |
| Real estate agency | Property listings, client history | "Which properties are available under ₹50L?" |
| D2C brand | Product specs, return policies, FAQs | "What's the return window for serums?" |
| MCR (Missed Call Recovery) client | Call scripts, pricing tiers, service areas | "Do we cover AC repair in Sector 54?" |

## Demo Walkthrough

The worked example throughout this build is **Café Bloom**, a fictional café in Bandra, Mumbai. A single SOP document covering refunds, hygiene, supplier management, and staff escalation was fed into the system. The Builder correctly extracted 7 entities (Café Bloom, Anita Roy, Vikram Patel, Razorpay, Blue Tokai, Amul, Intercom) and 9 relationships, categorized it under Operations, and scored it at full confidence with zero contradictions. The Librarian was then able to correctly answer complex synthesis questions — explaining the difference between its own Builder and Librarian halves, and correctly distinguishing when to use vector search versus knowledge graph search — entirely from the ingested document, with accurate citations and no hallucinations.

## Workflow Stats

- **40 nodes total**
- 4 independent entry points (Drive trigger, Daily schedule, Weekly schedule, Chat trigger)
- 5 AI agents (Categorizer, Relationship Mapper, Gap Finder, Librarian, Weekly Intelligence Analyst)
- 2 Supabase-backed storage layers (vector + relational)

---

Built by [Ram](https://github.com/Ram-cyber-dev) 
