---
layout: single
title: "Retrieval Augmented Generation (RAG): How It Works Step-by-Step"
date: 2026-03-04
category: "concepts"
tags: ["concepts", "atlas-signal", "deep-research"]
description: "RAG combines vector search with LLMs to ground AI responses in your own data. By embedding documents, storing them in a vector database, and retrieving relevant"
canonical_url: "https://atlassignal.in/posts/retrieval-augmented-generation-rag-how-it-works-step-by-step/"
og_title: "Retrieval Augmented Generation (RAG): How It Works Step-by-Step"
og_description: "RAG combines vector search with LLMs to ground AI responses in your own data. By embedding documents, storing them in a vector database, and retrieving relevant"
og_url: "https://atlassignal.in/posts/retrieval-augmented-generation-rag-how-it-works-step-by-step/"
og_image: "https://images.pexels.com/photos/6289063/pexels-photo-6289063.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940"
header:
  overlay_image: https://images.pexels.com/photos/6289063/pexels-photo-6289063.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940
  overlay_filter: 0.75
twitter_card: summary_large_image
toc: true
toc_sticky: true
---

![Retrieval Augmented Generation (RAG): How It Works Step-by-Step](https://images.pexels.com/photos/6289063/pexels-photo-6289063.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=650&w=940)


**Difficulty:** Intermediate | **Category:** Concepts

# Retrieval Augmented Generation (RAG): How It Works Step-by-Step


<ins class="adsbygoogle"
     style="display:block"
     data-ad-client=""
     data-ad-slot="AUTO"
     data-ad-format="auto"
     data-full-width-responsive="true"></ins>
<script>(adsbygoogle = window.adsbygoogle || []).push({});</script>


By March 2026, over 60% of production LLM applications use RAG architecture rather than fine-tuning—and for good reason. When Shopify reduced hallucinations in their customer support AI from 23% to under 3%, they didn't retrain their model; they implemented RAG to ground responses in actual support documentation.

RAG (Retrieval Augmented Generation) solves the fundamental problem that LLMs can't access information they weren't trained on. Instead of hoping GPT-4 magically knows about your Q3 product updates, RAG retrieves the relevant documents first, then feeds them to the LLM as context. Think of it as giving the AI an open-book exam instead of testing its memory.

## Prerequisites

Before diving in, you should have:
- Basic understanding of how LLMs work (prompts in, tokens out)
- Python 3.9+ installed locally
- An OpenAI API key (gpt-3.5-turbo costs ~$0.002/1K tokens as of March 2026)
- Familiarity with pip and virtual environments

## Step-by-Step Guide

### Step 1: Understanding the RAG Pipeline

RAG operates in two distinct phases: **indexing** (preparing your knowledge base) and **querying** (retrieving + generating answers).

The indexing phase:
1. Split documents into chunks (typically 200-1000 tokens each)
2. Convert chunks into vector embeddings using an embedding model
3. Store embeddings in a vector database with metadata

The querying phase:
1. Convert user question into an embedding
2. Find the most similar document chunks (cosine similarity)
3. Inject retrieved chunks into the LLM prompt
4. Generate response grounded in the retrieved context

**Gotcha:** Many developers skip chunk overlap, leading to context breaks mid-sentence. Always overlap chunks by 10-20% to preserve semantic continuity.

### Step 2: Install the Essential Libraries

Set up your environment with the core RAG stack:

```bash
pip install langchain==0.1.12 \
            openai==1.14.0 \
            chromadb==0.4.24 \
            tiktoken==0.6.0 \
            pypdf==4.1.0
```

These versions are tested as of March 2026. LangChain handles orchestration, ChromaDB provides vector storage, and tiktoken counts tokens accurately for OpenAI models.

**Pro tip:** Use `chromadb` for local development and testing. For production with >1M documents, migrate to Pinecone, Weaviate, or Qdrant which offer managed scaling.

### Step 3: Chunk Your Documents

Document splitting is make-or-break for RAG quality. Here's how to do it right:

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.document_loaders import PyPDFLoader

# Load a document
loader = PyPDFLoader("company_handbook.pdf")
documents = loader.load()

# Split with overlap
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,  # characters, not tokens
    chunk_overlap=100,  # 20% overlap
    length_function=len,
    separators=["\n\n", "\n", ". ", " ", ""]
)

chunks = text_splitter.split_documents(documents)
print(f"Split {len(documents)} pages into {len(chunks)} chunks")
# Output: Split 47 pages into 312 chunks
```

The `RecursiveCharacterTextSplitter` tries to split on paragraph breaks first, then sentences, then words—preserving semantic units.

**Gotcha:** Chunk size dramatically affects results. Too small ( 2000 chars) exceeds embedding model limits and dilutes relevance scores. Start with 500 characters and A/B test.

### Step 4: Generate and Store Embeddings

Convert text chunks into vectors that capture semantic meaning:

```python
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma
import os

os.environ["OPENAI_API_KEY"] = "sk-proj-your-key-here"

# Initialize embedding model
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small"  # $0.02 per 1M tokens
)

# Create vector store
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db",
    collection_name="company_docs"
)

vectorstore.persist()
print(f"Stored {vectorstore._collection.count()} embeddings")
```

The `text-embedding-3-small` model produces 1536-dimensional vectors. Each chunk's embedding is stored alongside its text content and metadata (page number, source file, etc.).

**Pro tip:** Batch your embedding calls to stay under rate limits. OpenAI allows 3,000 requests/minute on text-embedding-3-small for tier 3 accounts.

### Step 5: Implement Semantic Search

When a user asks a question, retrieve the most relevant chunks:

```python
from langchain.chains import RetrievalQA
from langchain.chat_models import ChatOpenAI

# Initialize LLM
llm = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    temperature=0.2  # Lower = more deterministic
)

# Create retrieval chain
qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",  # Stuff all retrieved docs into prompt
    retriever=vectorstore.as_retriever(
        search_type="similarity",
        search_kwargs={"k": 4}  # Retrieve top 4 chunks
    ),
    return_source_documents=True
)

# Query
result = qa_chain({
    "query": "What is the remote work policy for engineering teams?"
})

print(result["result"])
print(f"\nSources: {[doc.metadata for doc in result['source_documents']]}")
```

The `k=4` parameter retrieves the four most similar chunks. These get inserted into the prompt template before generation.

**Gotcha:** The "stuff" chain type concatenates all retrieved docs into the prompt. This fails if total context exceeds the model's limit (16K tokens for gpt-3.5-turbo). For larger retrievals, use `chain_type="map_reduce"` or `"refine"`.

### Step 6: Customize the Prompt Template

Default RAG prompts are generic. Here's how to add source citation:

```python
from langchain.prompts import PromptTemplate

template = """You are a helpful assistant answering questions about company policies.
Use the following context to answer the question. If you don't know, say so.
Always cite the page number when referencing information.

Context: {context}

Question: {question}

Answer with citations:"""

PROMPT = PromptTemplate(
    template=template,
    input_variables=["context", "question"]
)

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",
    retriever=vectorstore.as_retriever(search_kwargs={"k": 4}),
    chain_type_kwargs={"prompt": PROMPT},
    return_source_documents=True
)
```

This prompt engineering drastically reduces hallucinations by explicitly instructing the model to acknowledge knowledge gaps.

### Step 7: Add Metadata Filtering

Filter retrieved chunks by document type, date, or department:

```python
# When creating chunks, add metadata
chunks_with_metadata = []
for i, chunk in enumerate(chunks):
    chunk.metadata["department"] = "Engineering"
    chunk.metadata["last_updated"] = "2026-03-01"
    chunks_with_metadata.append(chunk)

# Query with filters
retriever = vectorstore.as_retriever(
    search_kwargs={
        "k": 4,
        "filter": {"department": "Engineering"}
    }
)
```

This ensures the LLM only sees Engineering-specific policies, not HR or Legal documents.

**Pro tip:** Add a `last_updated` timestamp to all chunks. During retrieval, boost recent documents' scores or filter out stale content entirely.

## Practical Example

Here's a complete RAG system for a product documentation chatbot:

```python
import os
from langchain.document_loaders import DirectoryLoader, TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import Chroma
from langchain.chains import RetrievalQA
from langchain.chat_models import ChatOpenAI

os.environ["OPENAI_API_KEY"] = "sk-proj-your-actual-key"

# 1. Load all markdown docs from directory
loader = DirectoryLoader(
    "./docs", 
    glob="**/*.md",
    loader_cls=TextLoader
)
documents = loader.load()

# 2. Chunk documents
splitter = RecursiveCharacterTextSplitter(
    chunk_size=600,
    chunk_overlap=120
)
chunks = splitter.split_documents(documents)

# 3. Create vector store
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = Chroma.from_documents(
    chunks,
    embeddings,
    persist_directory="./doc_db"
)

# 4. Build QA chain
llm = ChatOpenAI(model="gpt-3.5-turbo-0125", temperature=0.1)
qa = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(search_kwargs={"k": 3}),
    return_source_documents=True
)

# 5. Query
response = qa({
    "query": "How do I authenticate API requests?"
})

print(f"Answer: {response['result']}\n")
print("Sources:")
for doc in response['source_documents']:
    print(f"  - {doc.metadata['source']}")
```

Run this script, and you'll have a functional documentation chatbot grounded in your actual docs. Total cost: ~$0.05 for 500 chunks embedded + ~$0.002 per query.

## Key Takeaways

- **RAG = Retrieval + Generation**: First find relevant documents via vector similarity search, then inject them as context for the LLM to reference
- **Chunk size matters**: Start with 500 characters and 20% overlap. A/B test based on your use case—technical docs benefit from larger chunks (800-1000), while FAQ-style content works better with smaller chunks (200-400)
- **Metadata filtering is powerful**: Add department, date, or document type filters to ensure users only retrieve relevant content subsets
- **Cost efficiency**: RAG is 10-50x cheaper than fine-tuning for knowledge-intensive tasks, with embedding costs around $0.02 per million tokens and no training required

## What's Next

Now that you understand RAG fundamentals, explore **hybrid search** (combining keyword and vector search) to handle queries with specific terminology or proper nouns that pure semantic search might miss.

---

**Key Takeaway:** RAG combines vector search with LLMs to ground AI responses in your own data. By embedding documents, storing them in a vector database, and retrieving relevant chunks before generation, you can build AI systems that reference specific sources instead of hallucinating facts.

---

*New AI tutorials published daily on [AtlasSignal](https://atlassignal.in). Follow [@AtlasSignalDesk](https://twitter.com/AtlasSignalDesk) for more.*

---

## 📧 Get Daily AI & Macro Intelligence

Stay ahead of market-moving news, emerging tech, and global shifts.

<div class="email-capture">
  <form action="" method="post" target="_blank">
    <input type="email" name="EMAIL" placeholder="Your email address" required />
    <button type="submit">Subscribe Free →</button>
  </form>
</div>

