# RAG pipeline
> two big phases: ingestion and question answering. The first prepares your knowledge base; the second retrieves the right context and gives it to the LLM.

---
## 1. Collect the source data

> Typical sources are PDFs, Word files, webpages, Notion pages, support tickets, internal docs, database rows, Slack messages, product manuals, legal documents, or customer-specific files.

Conceptually:
```
PDFs
Docs
Web pages
Database records
     ↓
Raw knowledge
```
For a startup, you usually store metadata alongside every source, such as:
```
document_id
organization_id
user_id
source
filename
created_at
permissions
tags
```
> This becomes very important later for access control.
## 2. Extract text from the documents
> You convert each source into clean text.

For a PDF, you may extract page-by-page text. For HTML, you remove navigation, headers, footers, ads, and boilerplate. For scanned PDFs, you may need OCR.
* **OCR:** Optical Character Recognition for pdf scans, image text
> The output should ideally preserve useful structure:
```
title
heading
section
paragraph
page number
table information
source URL
```
## 3. Clean and normalize the text
> Raw extracted text is often messy.

* You may remove duplicate whitespace, broken encoding, repeated page headers, repeated footers, navigation text, malformed characters, and obvious extraction errors.

* Don't aggressively rewrite the document because you want the retrieved text to remain faithful to the original source.

## 4. Split the document into chunks
> You normally don't create one embedding for an entire 100-page PDF. You split it into smaller pieces.
* A chunk might look like:
```
Chunk 142:

"Customers can request a refund within 30 days of
purchase. Refund requests must include the original
order number..."
```

> **Typical chunk sizes are roughly a few hundred tokens, although the correct size depends on your data.**
* You might use something like:
```
500 tokens per chunk
50-100 token overlap
```

> there is no single universal chunk size, but standard industry benchmarks **balance vector embedding context limits, semantic coherence, and retrieval precision**.
```text
Common Chunk Size Ranges:
 ├── Small Chunks (128 – 256 Tokens)
 │    ├── Best For: Precise fact retrieval, sentence-level matching, short Q&A, customer support
 │    └── Tradeoffs: High precision, lower noise | Risks losing broader narrative context
 ├── Medium Chunks (512 – 1024 Tokens) [Industry Standard]
 │    ├── Best For: General RAG pipelines, enterprise docs, product manuals, technical articles
 │    └── Tradeoffs: Balanced context for LLM generation | Specific enough for semantic vector capture
 └── Large Chunks (1024 – 2048+ Tokens)
      ├── Best For: Complex analytical documents, legal briefs, dense research papers, long summaries
      └── Tradeoffs: Preserves rich context and complex logic | Increases noise, consumes large context windows
```
> **Key Chunking Best Practices:**
* **Chunk Overlap (10% – 20%):** A standard overlap of 50 to 100 tokens (or ~10–20% of the total chunk size) prevents loss of semantic context or split concepts at chunk boundaries.

* **Semantic & Structural Chunking:** Rather than splitting strictly at fixed token counts, modern pipelines split text along natural structural boundaries (such as paragraphs, Markdown headings, HTML sections, or code blocks).

* **Parent-Child (Hierarchical) Retrieval:** Store smaller chunks (e.g., 200 tokens) in the vector database for fine-grained retrieval, but map them back to parent chunks (e.g., 1000 tokens) or surrounding context windows when constructing the final LLM prompt.

## 5. Attach metadata to each chunk

> Every chunk should carry enough information to understand where it came from.

* For example:
```
{
  "chunk_id": 142,
  "document_id": 18,
  "organization_id": 42,
  "filename": "refund_policy.pdf",
  "page": 7,
  "section": "Refund Eligibility",
  "text": "Customers can request...",
  "created_at": "..."
}
```
> This is critical for filtering, citations, security, deletion, and updates.
## 6. Create an embedding for every chunk
> An embedding model converts the text into a vector.

* Conceptually:
```
"Customers can request refunds within 30 days"
                      ↓
               Embedding model
                      ↓
[0.021, -0.448, 0.192, 0.771, ...]
```
> That vector may contain hundreds or thousands of numbers.

> The important idea is that semantically similar text tends to produce vectors that are close together.

* So these sentences:
```
"How long do I have to return my purchase?"

"Customers may request refunds within 30 days."
```
> can be close in vector space even though they don't share many exact words.

## 7. Store the chunk, metadata, and embedding
> If you're using pgvector, your table might conceptually look like:
```
chunks

id
document_id
organization_id
content
page_number
metadata

embedding vector(...)
```
> Then create a vector index, often HNSW.
* **HNSW:** HNSW (Hierarchical Navigable Small World) is one of the most widely used graph-based algorithms for Approximate Nearest Neighbor (ANN) search in vector databases and search engines.

* Your architecture now looks like:
```
Documents
    ↓
Extract
    ↓
Clean
    ↓
Chunk
    ↓
Embeddings
    ↓
PostgreSQL + pgvector
```
---

> This completes the ingestion pipeline 

---

## 8. A user asks a question
> Now the runtime RAG flow starts.

* Suppose the user asks:
```
"How long do I have to request a refund?"
```
* Your backend receives the query.
```
User
  ↓
API
  ↓
RAG system
```

## 9. Optionally rewrite or understand the query

> Sometimes the raw user query is not ideal for retrieval.

* For example, in a conversation:
```
User: What is the refund policy?
Assistant: ...
User: What about enterprise customers?

Searching for:

"What about enterprise customers?"
```
isn't very useful.

* You may rewrite it as:
```
"What is the refund policy for enterprise customers?"
```
> This is often called query rewriting or query contextualization.

## 10. Create an embedding for the user's query

> Use the same embedding model family you used during ingestion.
```
"How long do I have to request a refund?"
                      ↓
               Embedding model
                      ↓
[0.018, -0.402, 0.211, 0.743, ...]
```
> Now you can compare that query vector against your stored chunk vectors.

## 11. Retrieve the most similar chunks

> Your vector database searches for the closest embeddings.

* With pgvector, conceptually:
```sql
SELECT
    content,
    document_id,
    page_number
FROM chunks
WHERE organization_id = 42
ORDER BY embedding <=> :query_embedding
LIMIT 10;
```
* You might retrieve:
```text
#1
"Customers can request refunds within 30 days..."

#2
"Refund requests should include the original order..."

#3
"Enterprise customers with annual contracts..."
```
> This is the core retrieval part of RAG.

## 12. Apply metadata and permission filters

> This is especially important in a SaaS product.

* Imagine company A and company B both upload private documents. You must never retrieve company B's chunks for company A.

* So your query may enforce:
```sql
WHERE organization_id = :current_org
```
* You may additionally filter by:
```
user permissions
project
document type
language
date
department
customer
region
```
> Security should normally happen during retrieval, not after sending data to the LLM.
## 13. Optionally perform hybrid search

> Pure vector search isn't always enough.

* Consider a query such as:
```
"Error code XJ-4921"
```
* Exact keyword matching may be much better than semantic similarity.

* So many production RAG systems combine:
```
Vector search
       +
Keyword/BM25 search
       ↓
Hybrid retrieval
```
> This is one area where OpenSearch can be particularly useful.
## 14. Rerank the retrieved chunks

>  Your vector search might retrieve 20 chunks, but some will be more relevant than others.

* A reranker evaluates the query against each candidate more carefully.
```
Initial retrieval

20 chunks
   ↓
Reranker
   ↓
Best 5 chunks
```
* For example:

* Vector search ranking:
```
Chunk A  0.87
Chunk B  0.84
Chunk C  0.82
```
* Reranker ranking:
```
Chunk C  0.96
Chunk A  0.91
Chunk B  0.61
```
> Reranking can significantly improve RAG quality.
## 15. Build the context for the LLM

> Now you take the best chunks and construct the model input.

* For example:
```
SYSTEM:

Answer the user's question using only the supplied
company documentation. If the answer isn't available,
say that you don't know.

CONTEXT:

[Source 1 - refund_policy.pdf, page 7]
Customers can request refunds within 30 days...

[Source 2 - refund_policy.pdf, page 8]
Enterprise customers with annual contracts...

USER:

How long do I have to request a refund?
```
> This is where the Augmented part of Retrieval-Augmented Generation happens.
## Send the prompt to the LLM

* Your LLM receives:
```
Instructions
    +
Conversation
    +
Retrieved context
    +
User question
```
and produces an answer.

* For example:
```
You can request a refund within 30 days of purchase.
```
> The LLM isn't expected to remember your private company documentation. You're supplying the relevant information dynamically.
## 17. Generate citations

> Because every chunk has metadata, you can tell the user where the answer came from.

* For example:
```
You can request a refund within 30 days of purchase.

Source:
refund_policy.pdf, page 7
```
> This is extremely useful for enterprise RAG because users want to verify the answer.
## 18. Return the answer to the user

> The final application flow becomes:
```
User question
    ↓
Query embedding
    ↓
Vector search
    ↓
Metadata filtering
    ↓
Top 20 chunks
    ↓
Reranking
    ↓
Top 5 chunks
    ↓
Prompt construction
    ↓
LLM
    ↓
Answer + citations
```
## 19. Log what happened

> A production RAG application should usually record information such as retrieval results, latency, document IDs used, retrieval scores, reranking scores, model, token usage, user feedback, and whether the answer had citations.

* Be careful about storing sensitive user questions and model outputs, particularly for enterprise customers.

* These logs help you understand whether failures are coming from retrieval or generation.

## 20. Evaluate the RAG system

* RAG quality isn't just:
```
"Does the answer sound good?"
```
* You need to evaluate separate pieces.

* Suppose your system gives the wrong answer.

* The reason could be:
```
Wrong document uploaded
       ↓
Bad extraction
       ↓
Bad chunking
       ↓
Embedding problem
       ↓
Retrieval missed correct chunk
       ↓
Reranker selected wrong chunk
       ↓
Prompt was poor
       ↓
LLM ignored correct context
```
> This distinction is extremely important.
* A good evaluation dataset might contain:
```
Question
Expected answer
Expected source/document
Expected relevant chunks
```
> Then measure retrieval and answer quality separately.
```
                     INGESTION

Customer uploads PDF
        ↓
Object storage
        ↓
Document parser
        ↓
Text extraction
        ↓
Cleaner
        ↓
Chunker
        ↓
Embedding model
        ↓
PostgreSQL + pgvector
        ├── chunks
        ├── embeddings
        ├── metadata
        └── permissions


                     QUESTION

User
 ↓
Frontend
 ↓
FastAPI backend
 ↓
Query rewriting
 ↓
Embedding model
 ↓
pgvector search
 ↓
Permission filtering
 ↓
Top 20 chunks
 ↓
Reranker
 ↓
Top 5 chunks
 ↓
Prompt builder
 ↓
LLM
 ↓
Answer + citations
 ↓
Frontend
```

![Alt text description](https://miro.medium.com/0*7OaGfO2DctgswevJ.jpeg)

| RAG Stage                            | Popular models / algorithms                                                   | Popular frameworks / infrastructure                                 | 
| ------------------------------------ | ----------------------------------------------------------------------------- | ------------------------------------------------------------------- | 
| **1. Document Ingestion & Chunking** | Docling VLM models, OCR models; usually no LLM for normal text                | **Docling**, Unstructured, LangChain, LlamaIndex, Haystack, PyMuPDF | 
| **2. Embedding Generation**          | OpenAI `text-embedding-3-small/large`, Voyage 4, Cohere Embed, BGE-M3, MiniLM | SentenceTransformers, Hugging Face, OpenAI/Voyage/Cohere SDKs       | 
| **3. Vector Storage**                | HNSW, IVFFlat                                                                 | **pgvector**, Qdrant, Pinecone, Weaviate, Milvus, OpenSearch        |   
| **4. Query Understanding**           | GPT family, Claude, Gemini, small local LLMs                                  | LangChain, LlamaIndex, Haystack                                     | 
| **5. Permission-Aware Retrieval**    | Dense retrieval, cosine similarity, BM25, hybrid search                       | **pgvector**, OpenSearch, Qdrant + application ACLs                 |  
| **6. Reranking**                     | Cohere Rerank v4, Voyage Rerank, BGE rerankers, CrossEncoder MiniLM           | Cohere, Voyage, SentenceTransformers, Haystack                      |  
| **7. Grounding / Context Building**  | Usually no special model; sometimes LLM context compression                   | LangChain, LlamaIndex, Haystack                                     | 
| **8. Generation**                    | GPT family, Claude Sonnet/Opus, Gemini Flash/Pro, Mistral, open-weight models | OpenAI/Anthropic/Google APIs, LangChain, LlamaIndex                 | 
| **9. Citation**                      | Usually **no model**                                                          | Your metadata + LangChain/LlamaIndex                                | 
| **10. Evaluation & Monitoring**      | LLM-as-judge models                                                           | LangSmith, RAGAS, Phoenix, DeepEval, TruLens                        | 

> Popular model families

* For hosted RAG generation, common choices include:

| Provider      | Model family       | Current model ID            |
| ------------- | ------------------ | --------------------------- |
| **OpenAI**    | GPT-5.6 Sol        | `gpt-5.6-sol`               |
|               | GPT-5.6 Terra      | `gpt-5.6-terra`             |
|               | GPT-5.6 Luna       | `gpt-5.6-luna`              |
| **Anthropic** | Claude Opus 5      | `claude-opus-5`             |
|               | Claude Sonnet 5    | `claude-sonnet-5`           |
|               | Claude Haiku 4.5   | `claude-haiku-4-5-20251001` |
| **Google**    | Gemini Pro         | `gemini-3.1-pro-preview`    |
|               | Gemini Flash       | `gemini-3.8-flash`          |
|               | Gemini Flash-Lite  | `gemini-3.5-flash-lite`     |
| **Mistral**   | Mistral Medium 3.5 | `mistral-medium-latest`     |
|               | Mistral Small 4    | `mistral-small-latest`      |
|               | Mistral Large 3    | `mistral-large-latest`      |


* Frameworks: 
```
LangChain
LlamaIndex
Haystack

or

direct provider SDK
```
