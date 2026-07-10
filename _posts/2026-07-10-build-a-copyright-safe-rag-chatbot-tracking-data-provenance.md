---
layout: single
title: "Build a Copyright-Safe RAG Chatbot: Tracking Data Provenance with OpenAI Embeddings"
date: 2026-07-10
author: "AtlasSignal Desk"
category: "ai_tools"
tags: ["ai_tools", "atlas-signal", "deep-research", "OpenAI", "AITools", "Productivity"]
description: "In light of OpenAI's ongoing copyright litigation, this tutorial shows you how to build a RAG chatbot that tracks and attributes source documents at every query"
canonical_url: "https://atlassignal.in/posts/build-a-copyright-safe-rag-chatbot-tracking-data-provenance/"
og_title: "Build a Copyright-Safe RAG Chatbot: Tracking Data Provenance with OpenAI Embeddings"
og_description: "In light of OpenAI's ongoing copyright litigation, this tutorial shows you how to build a RAG chatbot that tracks and attributes source documents at every query"
og_url: "https://atlassignal.in/posts/build-a-copyright-safe-rag-chatbot-tracking-data-provenance/"
og_image: "https://images.pexels.com/photos/4976710/pexels-photo-4976710.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940"
header:
  overlay_image: https://images.pexels.com/photos/4976710/pexels-photo-4976710.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940
  overlay_filter: 0.75
twitter_card: summary_large_image
toc: true
toc_sticky: true
---

![Build a Copyright-Safe RAG Chatbot: Tracking Data Provenance with OpenAI Embeddings](https://images.pexels.com/photos/4976710/pexels-photo-4976710.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940)


**Difficulty:** Intermediate | **Category:** Ai Tools

# Build a Copyright-Safe RAG Chatbot: Tracking Data Provenance with OpenAI Embeddings

The New York Times' latest accusations against OpenAI—claiming hidden evidence in their copyright trial—underscore a critical vulnerability for anyone building RAG (Retrieval-Augmented Generation) applications today. By the end of this tutorial, you'll deploy a production-grade RAG chatbot that tracks every document source, maintains a queryable audit log, and attributes every AI-generated response to its original content—protecting you from the legal landmines that caught OpenAI off-guard.

## Prerequisites

Before you begin, ensure you have:

- **Python >=3.11** installed locally
- **OpenAI API key** with GPT-4o access (2026 pricing: ~$2.50/M input tokens, $10/M output)
- **PostgreSQL >=15** or access to a hosted instance (we'll use pgvector extension)
- **Basic familiarity** with vector embeddings and SQL
- **git** and a code editor

Install required packages:
```bash
pip install openai==1.35.0 psycopg2-binary==2.9.9 pgvector==0.2.5 python-dotenv==1.0.1
```

## Step-by-Step Guide

### Step 1: Initialize Your Database with Provenance Tracking

Create a PostgreSQL database with the pgvector extension and a custom schema that logs document lineage:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    source_url TEXT NOT NULL,
    author TEXT,
    publication_date TIMESTAMP,
    license_type TEXT NOT NULL,
    ingestion_timestamp TIMESTAMP DEFAULT NOW(),
    content_hash TEXT UNIQUE NOT NULL
);

CREATE TABLE embeddings (
    id SERIAL PRIMARY KEY,
    document_id INTEGER REFERENCES documents(id) ON DELETE CASCADE,
    embedding vector(1536),
    chunk_index INTEGER,
    chunk_text TEXT NOT NULL
);

CREATE TABLE query_audit_log (
    id SERIAL PRIMARY KEY,
    query_text TEXT NOT NULL,
    query_timestamp TIMESTAMP DEFAULT NOW(),
    retrieved_doc_ids INTEGER[],
    response_text TEXT,
    model_used TEXT NOT NULL
);

CREATE INDEX ON embeddings USING ivfflat (embedding vector_cosine_ops);
```

⚠️ **WARNING:** The `content_hash` field prevents accidental duplicate ingestion. Always compute SHA-256 hashes of source documents before insertion to avoid poisoning your vector store with redundant data.

### Step 2: Build a Document Ingestion Pipeline with Attribution

Create `ingest.py` to chunk documents and store embeddings alongside full metadata:

```python
import hashlib
import os
from openai import OpenAI
import psycopg2
from datetime import datetime

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def chunk_text(text, chunk_size=500, overlap=50):
    """Split text into overlapping chunks for better context preservation."""
    words = text.split()
    chunks = []
    for i in range(0, len(words), chunk_size - overlap):
        chunk = " ".join(words[i:i + chunk_size])
        chunks.append(chunk)
    return chunks

def ingest_document(content, source_url, author, pub_date, license_type, conn):
    """Ingest a document with full provenance tracking."""
    content_hash = hashlib.sha256(content.encode()).hexdigest()
    
    cursor = conn.cursor()
    
    # Check for duplicates
    cursor.execute("SELECT id FROM documents WHERE content_hash = %s", (content_hash,))
    if cursor.fetchone():
        print(f"Document already ingested: {source_url}")
        return
    
    # Insert document metadata
    cursor.execute("""
        INSERT INTO documents (content, source_url, author, publication_date, license_type, content_hash)
        VALUES (%s, %s, %s, %s, %s, %s) RETURNING id
    """, (content, source_url, author, pub_date, license_type, content_hash))
    
    doc_id = cursor.fetchone()[0]
    
    # Chunk and embed
    chunks = chunk_text(content)
    for idx, chunk in enumerate(chunks):
        response = client.embeddings.create(
            model="text-embedding-3-small",  # $0.02/M tokens in 2026
            input=chunk
        )
        embedding = response.data[0].embedding
        
        cursor.execute("""
            INSERT INTO embeddings (document_id, embedding, chunk_index, chunk_text)
            VALUES (%s, %s, %s, %s)
        """, (doc_id, embedding, idx, chunk))
    
    conn.commit()
    print(f"✓ Ingested {len(chunks)} chunks from {source_url}")
```

**Pro Tip:** For production use, batch embed up to 100 chunks per API call to reduce latency. OpenAI's `text-embedding-3-small` supports batch inputs and costs 10x less than the older Ada-002 model.

### Step 3: Implement Attribution-First Retrieval

Build `retrieval.py` that returns not just relevant chunks but full source attribution:

```python
from openai import OpenAI
import psycopg2
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def retrieve_with_attribution(query, conn, top_k=3):
    """Retrieve relevant chunks with full source metadata."""
    # Generate query embedding
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=query
    )
    query_embedding = response.data[0].embedding
    
    cursor = conn.cursor()
    cursor.execute("""
        SELECT 
            e.chunk_text,
            d.source_url,
            d.author,
            d.publication_date,
            d.license_type,
            d.id,
            1 - (e.embedding  %s::vector) AS similarity
        FROM embeddings e
        JOIN documents d ON e.document_id = d.id
        ORDER BY e.embedding  %s::vector
        LIMIT %s
    """, (query_embedding, query_embedding, top_k))
    
    results = cursor.fetchall()
    
    return [{
        "content": r[0],
        "source_url": r[1],
        "author": r[2],
        "publication_date": r[3],
        "license_type": r[4],
        "document_id": r[5],
        "similarity": r[6]
    } for r in results]
```

⚠️ **WARNING:** The `` operator computes cosine distance. Lower values = higher similarity. Always use `ORDER BY embedding  query` not `ORDER BY similarity DESC` to leverage the IVFFlat index.

### Step 4: Generate Attributed Responses with Audit Logging

Create `chatbot.py` that logs every query and forces citation in responses:

```python
from openai import OpenAI
import psycopg2
import json
import os

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

def generate_response(query, conn):
    """Generate response with mandatory source attribution."""
    # Retrieve context
    results = retrieve_with_attribution(query, conn, top_k=3)
    
    if not results:
        return "I don't have enough information to answer that question."
    
    # Build context with citation markers
    context_parts = []
    for idx, r in enumerate(results, 1):
        context_parts.append(
            f"[Source {idx}: {r['source_url']}, Author: {r['author']}, "
            f"License: {r['license_type']}]\n{r['content']}\n"
        )
    
    context = "\n".join(context_parts)
    
    # System prompt enforcing attribution
    messages = [
        {"role": "system", "content": """You are a helpful assistant that ALWAYS cites sources.
        For every fact you state, reference the source number in brackets like [Source 1].
        If information comes from multiple sources, cite all: [Source 1, 2].
        Never make claims without citing the provided sources."""},
        {"role": "user", "content": f"Context:\n{context}\n\nQuestion: {query}"}
    ]
    
    response = client.chat.completions.create(
        model="gpt-4o-2024-11-20",  # Latest stable GPT-4o in mid-2026
        messages=messages,
        temperature=0.3
    )
    
    answer = response.choices[0].message.content
    
    # Log to audit trail
    doc_ids = [r['document_id'] for r in results]
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO query_audit_log (query_text, retrieved_doc_ids, response_text, model_used)
        VALUES (%s, %s, %s, %s)
    """, (query, doc_ids, answer, "gpt-4o-2024-11-20"))
    conn.commit()
    
    # Append source details to response
    source_list = "\n\n**Sources:**\n"
    for idx, r in enumerate(results, 1):
        source_list += f"{idx}. {r['source_url']} ({r['author']}, {r['license_type']})\n"
    
    return answer + source_list
```

**Pro Tip:** Set `temperature=0.3` for factual queries to minimize hallucination. For creative tasks, increase to 0.7-0.9, but always keep attribution enforcement in the system prompt.

### Step 5: Test with Real-World Data

Create `test_pipeline.py` to validate end-to-end:

```python
import psycopg2
import os
from dotenv import load_dotenv
from ingest import ingest_document
from chatbot import generate_response

load_dotenv()

conn = psycopg2.connect(os.getenv("DATABASE_URL"))

# Ingest sample document
sample_content = """
Large language models can reproduce copyrighted material from their training data.
This poses legal risks for developers deploying RAG systems without proper attribution.
Always track document provenance and implement citation mechanisms.
"""

ingest_document(
    content=sample_content,
    source_url="https://example.com/ai-copyright-guide",
    author="Jane Smith",
    pub_date="2026-07-01",
    license_type="CC-BY-4.0",
    conn=conn
)

# Query with attribution
response = generate_response("What are the copyright risks of LLMs?", conn)
print(response)

conn.close()
```

Run it:
```bash
export DATABASE_URL="postgresql://user:pass@localhost/ragdb"
export OPENAI_API_KEY="sk-proj-..."
python test_pipeline.py
```

Expected output includes the answer with inline citations like `[Source 1]` and a full source list at the bottom.

## Complete Working Example

Here's a minimal `main.py` combining all components:

```python
import psycopg2
import os
from dotenv import load_dotenv
from ingest import ingest_document
from chatbot import generate_response

load_dotenv()

def main():
    conn = psycopg2.connect(os.getenv("DATABASE_URL"))
    
    # Ingest documents (run once)
    documents = [
        {
            "content": "OpenAI's GPT-4 was trained on a mixture of licensed data, publicly available data, and data created by human trainers.",
            "source_url": "https://openai.com/research/gpt4-system-card",
            "author": "OpenAI",
            "pub_date": "2023-03-14",
            "license_type": "Proprietary"
        },
        {
            "content": "Fair use doctrine in the US allows limited use of copyrighted material without permission for purposes like commentary, criticism, and education.",
            "source_url": "https://copyright.gov/fair-use/",
            "author": "US Copyright Office",
            "pub_date": "2024-01-10",
            "license_type": "Public Domain"
        }
    ]
    
    for doc in documents:
        ingest_document(conn=conn, **doc)
    
    # Interactive chatbot
    print("RAG Chatbot with Provenance Tracking (type 'exit' to quit)\n")
    while True:
        query = input("You: ")
        if query.lower() == 'exit':
            break
        
        response = generate_response(query, conn)
        print(f"\nBot:\n{response}\n")
    
    conn.close()

if __name__ == "__main__":
    main()
```

Run with: `python main.py`

## Debugging Common Issues

**Error:** `psycopg2.errors.UndefinedObject: type "vector" does not exist`  
**Cause:** pgvector extension not installed  
**Fix:** Connect to your database and run `CREATE EXTENSION vector;` as a superuser

**Error:** `openai.error.InvalidRequestError: The model 'text-embedding-ada-002' has been deprecated`  
**Cause:** Using outdated embedding model  
**Fix:** Update all `model=` parameters to `"text-embedding-3-small"` or `"text-embedding-3-large"`

**Error:** Query returns no results despite having ingested documents  
**Cause:** Index not created or using wrong distance operator  
**Fix:** Verify index exists with `\d embeddings` in psql. Ensure queries use `` not `` for cosine distance

**Error:** Response lacks citations even with enforcement prompt  
**Cause:** Context too long, causing truncation  
**Fix:** Reduce `top_k` to 2-3 chunks or increase `max_tokens` parameter in completion call

## Key Takeaways

- **Provenance-first design** protects you legally: every response includes source URLs, authors, and license types that can be audited
- **Content hashing** prevents duplicate ingestion and helps track when documents were last updated
- **Audit logging** creates a defensible record showing you retrieved and attributed specific sources for each query—critical if you face copyright claims
- **Modern embeddings** (`text-embedding-3-small`) cost 90% less than legacy models while delivering better retrieval quality

## What's Next

Extend this system by implementing automated license compatibility checks before ingestion, blocking documents with restrictive copyrights unless you have explicit permission—a feature that would have saved OpenAI significant legal headaches.

---

**Key Takeaway:** In light of OpenAI's ongoing copyright litigation, this tutorial shows you how to build a RAG chatbot that tracks and attributes source documents at every query, creating an audit trail that demonstrates responsible AI usage and protects against copyright claims.

---

*New AI tutorials published daily on [AtlasSignal](https://atlassignal.in). Follow [@AtlasSignalDesk](https://twitter.com/AtlasSignalDesk) for more.*

---

*This report was produced with AI-assisted research and drafting, curated and reviewed under AtlasSignal's [editorial standards](/editorial-standards/). For corrections or feedback, contact [atlassignal.ai@gmail.com](mailto:atlassignal.ai@gmail.com).*


