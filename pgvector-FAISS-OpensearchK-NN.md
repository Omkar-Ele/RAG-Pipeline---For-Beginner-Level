

# Basic overview: 

| Tool                | What it is                        | Best fit                                                                   | Main tradeoff                                                                    |
| ------------------- | --------------------------------- | -------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| **pgvector**        | PostgreSQL extension              | Apps already using Postgres; vectors + relational data + SQL               | Simpler architecture, but not ideal for very large/high-throughput vector search |
| **FAISS**           | Vector search library             | Maximum control/performance inside your own Python/C++ service             | You must build persistence, filtering, replication, APIs, etc. yourself          |
| **OpenSearch k-NN** | Distributed search engine feature | Large-scale vector + keyword/hybrid search, filters, distributed workloads | Heavier infrastructure and operational complexity                                |



## pgvector:

pgvector stores embeddings directly in PostgreSQL,


```sql 
SELECT id, content
FROM documents
ORDER BY embedding <=> query_embedding
LIMIT 10; 
```

Its biggest advantage is everything stays in Postgres. You can combine vector similarity with normal SQL:

```sql 
SELECT *
FROM products
WHERE category = 'shoes'
  AND price < 100
ORDER BY embedding <=> query_embedding
LIMIT 10;
```

Use pgvector when your system already runs on Postgres and you want something operationally simple.

## FAISS:

FAISS, originally developed by Meta, is fundamentally a high-performance vector indexing/search library.

It has many indexing strategies, including:

```text
Flat/exact search
IVF
HNSW
Product quantization
GPU acceleration
```

FAISS can be extremely fast and memory-efficient, particularly when you tune the index carefully.

But FAISS is not really a database. Out of the box, it doesn't give you the full surrounding system you'd expect from one:

```text
authentication
distributed replication
SQL queries
rich metadata filtering
backup infrastructure
REST APIs
cluster management
```

You usually build those pieces around it.

It's excellent when you're building a specialized vector-search service and need fine-grained control.

## OpenSearch k-NN:

OpenSearch is a distributed search engine, similar architecturally to Elasticsearch. Its k-NN functionality adds vector search alongside conventional search.

This means one query can combine:

```text
vector similarity
+
BM25 keyword search
+
metadata filters
+
facets
+
permissions
```

For example, a search application might do:

```text
semantic similarity: "comfortable running shoes"
AND
brand = Nike
AND
price < $150
AND
keyword relevance
```

OpenSearch is especially attractive for hybrid search where keyword relevance and embeddings both matter.

It also handles things like:

```text
sharding
replication
distributed querying
high availability
index lifecycle
REST APIs
```

## Architecture difference

The easiest way to remember them is:


*pgvector*

```text
Postgres
 ├── relational data
 ├── SQL
 └── vector search
 ```

*FAISS*

```text
Your Application
 └── FAISS
      └── vector index
 ```
*OpenSearch*

```text
Distributed Search Cluster
 ├── keyword search
 ├── vector search
 ├── filtering
 ├── aggregations
 └── distributed indexing
```

```text
PostgreSQL + pgvector → cheapest/simplest place to start.
```

### For General Starups:
```text
Day 1 startup
        ↓
Postgres + pgvector
        ↓
Get users + product-market fit
        ↓
Millions / tens of millions of embeddings
        ↓
Measure actual bottlenecks
        ↓
┌────────────────┬──────────────────┐
│ Search-heavy   │ Vector-heavy     │
│ requirements   │ ML requirements  │
│       ↓        │        ↓         │
│ OpenSearch     │ FAISS/custom     │
└────────────────┴──────────────────┘
```

## For an AI/RAG startup specifically

My default stack would probably be:

### Backend
```text
Python / FastAPI
        ↓
PostgreSQL
        +
pgvector
        ↓
HNSW index
        ↓
Embedding API/model
        ↓
LLM
```

## High-Level AI Orchestration Frameworks
Frameworks that sit above vector databases to simplify ingestion, chunking, embedding generation, and retrieval pipelines (RAG):

* **LlamaIndex:** Framework specialized for connecting custom data sources to LLMs via vector stores, hierarchical indices, and advanced retrieval strategies.

* **LangChain:** Popular framework for building LLM applications, offering integrations across nearly all vector stores and embedding models.

* **Haystack (by Deepset):** Enterprise-grade NLP framework for building production search and modular RAG pipelines.

* **Semantic Kernel:** Microsoft's SDK for orchestrating AI models, memory connectors, and vector plugins.
