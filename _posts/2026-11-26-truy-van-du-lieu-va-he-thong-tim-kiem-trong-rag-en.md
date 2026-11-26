---
title: "Data Retrieval & Search Systems in RAG"
description: "Theory and practical operations of Data Retrieval in RAG: BM25, Learning to Rank, Dense Retrieval, ColBERT, Cross-Encoders, GraphRAG, RRF, and evaluation metrics."
date: 2026-11-26 17:54:00 +0700
categories: [AI & Agent, RAG]
tags: [rag, data-retrieval, bm25, semantic-search, graphrag, reranking]
pin: false
toc: true
math: true
---

## 1. Overview

**Data Retrieval** is the process of extracting specific, relevant information from a large data repository based on a user or system query. In modern AI applications — especially **RAG (Retrieval-Augmented Generation)** architectures — retrieval quality directly determines the accuracy of the final response, as a Large Language Model (LLM) can only synthesize reliable answers when grounded in the right context.

Target data typically falls into three categories:
- **Structured Data:** Organized under a strict schema, such as tables in SQL databases.
- **Unstructured Data:** Raw text, PDFs, images, and audio.
- **Semi-structured Data:** Flexible, tagged, or nested formats such as HTML, JSON, and XML.

Depending on the use case, retrieval systems power tasks ranging from Passage Retrieval and Text-to-SQL to Image Retrieval and Multimodal Search (querying across both text and images).

### Popular Data Retrieval Approaches Today

In modern search and RAG deployments, no single retrieval technique handles every scenario well on its own. Industry practice relies on multi-stage combinations:

1. **Hybrid Search + Cross-Encoder Reranking (Most Popular RAG Standard):** Running keyword search (**BM25**) and semantic search (**Dense Vector Search**) in parallel, merging candidates with **Reciprocal Rank Fusion (RRF)**, and re-scoring the top results with a **Cross-Encoder Reranker**. This balances speed, semantic understanding, and exact-match precision for proper nouns or technical codes.
2. **Dense Retrieval with Metadata Filtering (Common for Small-to-Medium Systems):** Using an Embedding Model paired with metadata filters (date, source, access permissions) inside a Vector Database (e.g., Qdrant, Milvus, pgvector) when deployment simplicity and low cost are priorities.
3. **Late Interaction / Multi-Vector — ColBERT:** Increasingly adopted for workloads requiring high token-level precision alongside contextual understanding.
4. **GraphRAG & Agentic Retrieval:** Used for multi-hop reasoning and corpus-wide synthesis (**GraphRAG**), or when an LLM dynamically plans, rewrites queries, and iterates across retrieval steps (**Agentic RAG**).
5. **BM25 + Learning to Rank (LTR):** Still the backbone of traditional large-scale industrial search (e.g., e-commerce, web search) thanks to ultra-low latency and seamless integration of non-textual signals like recency, popularity, and click behavior.

---

## 2. Main Components and Methods

A modern retrieval stack brings together complementary techniques, spanning exact lexical matching, learned ranking models, dense semantic embeddings, and knowledge graphs.

### 2.1. Keyword / Lexical Search

Traditional search relies on exact term matching between the query and documents. To achieve instant lookup across millions of records, systems store data in an **Inverted Index**: for each term $t$ in the vocabulary, the index stores its **IDF** along with a posting list of `doc_id`s containing $t$ and their corresponding **TF** values.

- **TF (Term Frequency):** Measures how often a term appears in a document, usually normalized by document length to avoid favoring longer texts.
- **IDF (Inverse Document Frequency):** Measures how rare a term is across the entire corpus via $\log\left(\frac{\text{total documents}}{\text{documents containing } t}\right)$ — rarer terms receive higher discriminative weight.

For ranking, classic **TF-IDF** represents both query and document as $\text{TF} \times \text{IDF}$ weight vectors in a Vector Space Model and scores them using **Cosine Similarity**. In practice, systems favor **BM25 (Okapi BM25)**, a probabilistic improvement over TF-IDF. Rather than computing Cosine Similarity, BM25 sums the probabilistic relevance contributions of each matching term $t \in q \cap d$, introducing **TF saturation** (diminishing returns for repeated terms to prevent keyword stuffing) and **document length normalization**:

$$\text{BM25}(q, d) = \sum_{t \in q \cap d} \text{IDF}(t) \cdot \frac{\text{tf}(t, d) \cdot (k_1 + 1)}{\text{tf}(t, d) + k_1 \cdot \left(1 - b + b \cdot \frac{|d|}{\text{avgdl}}\right)}$$

*(Where $k_1 \approx 1.2\text{--}2.0$ controls TF saturation speed, and $b \approx 0.75$ controls length penalty based on document length $|d|$ and corpus average length $\text{avgdl}$).*

Although lexical search cannot capture synonyms or deeper context, engines like **Elasticsearch** (built on Apache Lucene) remain indispensable for exact matches on proper nouns, error codes, IDs, and domain-specific identifiers.

### 2.2. Learning to Rank (LTR)

**Learning to Rank** frames search ranking as a supervised machine learning problem. The system extracts features from each `(Query, Document)` pair — such as BM25/TF-IDF scores, document length, title match ratio, and user click signals — and trains against relevance labels (binary `0/1` or graded `0–3`).

LTR approaches fall into three loss formulations:
- **Pointwise:** Predicts a relevance score for each document independently (e.g., Linear Regression, Logistic Regression).
- **Pairwise:** Learns which document is more relevant within a pair (e.g., RankSVM).
- **Listwise:** Directly optimizes the ordering of an entire candidate list against ranking metrics like nDCG or MAP (e.g., ensemble models such as **GBDT / LambdaMART** via XGBoost or LightGBM).

Because LTR requires manual feature engineering and lacks the deep semantic understanding of Transformers, it is rarely used as the primary retriever in RAG. However, it remains widely used at the **Re-ranking** stage of large-scale search engines due to its speed and ability to blend non-textual signals.

### 2.3. Semantic Search / Dense Retrieval

Instead of matching surface tokens, Semantic Search uses language/embedding models (e.g., BERT, RoBERTa, BGE, E5) to encode queries and documents into **dense embeddings** in a semantic vector space, measuring similarity via **Cosine Similarity** or **Dot Product**. This enables matching synonyms and intent even without overlapping keywords.

Key building blocks of dense retrieval include:

- **Bi-Encoder Architecture & Domain Adaptation:** General pre-trained models often struggle with specialized jargon, so they are fine-tuned using a **Bi-Encoder** setup with two independent encoding branches for `Query` and `Document` — either sharing weights (**Siamese**) or using separate weights (**Dual-Encoder / Asymmetric Bi-Encoder**). Training typically uses **Contrastive Learning / Triplet Loss** on `(Query, Positive Doc, Negative Doc)` triplets to pull relevant pairs closer and push irrelevant ones apart. *(Pre-training techniques like **Condenser / coCondenser** further improve how information is compressed into the `[CLS]` vector).*
- **Approximate Nearest Neighbor (ANN) Indexing:** Brute-force scanning across millions of vectors is too slow, so vector databases rely on ANN indexes:
  - **IVF (Inverted File Index):** Uses K-Means (or Hierarchical K-Means) to partition the vector space into clusters; queries only scan vectors inside the nearest cluster centroids.
  - **HNSW (Hierarchical Navigable Small World):** Connects vectors into a multi-layer proximity graph (sparse upper layers for fast long-range hops, dense lower layers for precise local search), delivering high speed and recall at the cost of high RAM usage.
- **Late Interaction — ColBERT:** Addresses the bottleneck of compressing an entire passage into a single vector by storing **per-token vectors** (multi-vector representation) for documents. At query time, each query token computes **MaxSim** (maximum similarity across all document tokens), and the results are summed for the final score. ColBERT achieves high accuracy and preserves fine-grained keyword sensitivity, but demands significantly more RAM and storage.
- **Reranking with Cross-Encoders (Full Interaction):** Concatenates `[CLS] Query [SEP] Document [SEP]` and passes the pair jointly through BERT so all tokens attend to each other across every layer before a classification/regression head scores relevance. Cross-Encoder rerankers can be trained with three neural loss paradigms: **Pointwise** (BCE or MSE per `(Query, Document)` pair), **Pairwise** (Margin Ranking Loss or Pairwise Cross-Entropy comparing `Doc+` vs. `Doc-`), or **Listwise** (InfoNCE, ListNet, or ListMLE optimizing over the full candidate list). Because document vectors cannot be pre-computed, Cross-Encoders are too slow for full-corpus search and are applied exclusively to **rerank** the top 20–100 candidates from initial retrieval.

### 2.4. GraphRAG (Knowledge Graph-Enhanced RAG)

**GraphRAG** builds a **Knowledge Graph** of **Entities** and **Relationships** alongside raw text chunks to enable deeper reasoning. It addresses two major limitations of standard Vector RAG: **multi-hop reasoning** (connecting facts across intermediate entities $A \to B \to C$ scattered across documents) and **global queries** (corpus-wide summarization questions that top-$k$ local chunks cannot answer).

A typical workflow (such as Microsoft GraphRAG) has two phases:
1. **Indexing:** An LLM extracts entities and relationships from chunks $\to$ a graph clustering algorithm (e.g., **Leiden**) groups nodes into hierarchical **Communities** $\to$ an LLM pre-generates summaries for each community.
2. **Querying:** **Local Search** locates query-relevant entities and traverses neighboring edges for detailed multi-hop context, whereas **Global Search** synthesizes answers from the pre-computed community summaries.

While GraphRAG excels at complex reasoning and corpus-level questions, indexing is expensive and slow (consuming substantial LLM tokens) and updates are harder to apply to rapidly changing data.

---

## 3. Usage and Practical Operations

In production, these components are assembled into an end-to-end pipeline covering data preparation, hybrid retrieval, memory optimization, and rigorous evaluation.

### 3.1. Data Preparation & Standard Retrieval Pipeline

1. **Chunking & Metadata Filtering:** Long documents are split into manageable chunks (`chunk_size` typically 256–1024 tokens with 10–20% `chunk_overlap` to preserve boundary context). Attaching metadata (`date`, `source`, `category`, `access_role`) enables **Metadata Filtering** — preferably via pre-filtering or single-stage filtering during vector search rather than post-filtering, which can leave too few results.
2. **Query Transformation:** When user queries are short, ambiguous, or multi-faceted, an LLM can improve recall before retrieval via **Query Rewriting**, **Multi-Query** decomposition, or **HyDE (Hypothetical Document Embeddings)** — generating a hypothetical answer first and using its embedding to search for real documents.
3. **Hybrid Search with RRF:** Running BM25 and Dense Vector Search in parallel yields strong candidate coverage (e.g., top 50–100). Since BM25 scores (unbounded) and Cosine Similarity ($[-1, 1]$) operate on incompatible scales, **Reciprocal Rank Fusion (RRF)** merges them cleanly using rank positions alone:
   $$\text{RRF}(d) = \sum_{i} \frac{1}{k + \text{rank}_i(d)} \quad (\text{typically } k = 60)$$
4. **Reranking & Vector Quantization:** Candidates are passed to a **Cross-Encoder** to select the top 3–10 passages for the LLM. At large scale where HNSW consumes excessive RAM, **Scalar Quantization (`int8` / SQ8)**, **Product Quantization (PQ)**, or **Binary Quantization** can shrink memory usage by 4×–32×, followed by rescoring with full-precision vectors.

### 3.2. Evaluation and Tuning

Quantitative evaluation guides model selection and hyperparameter tuning (`top_k`, chunk size, score thresholds). Relevance is typically defined as binary (relevant if a document contains any useful portion, or graded for nDCG), while **Accuracy** is avoided because the vast number of irrelevant documents inflates it artificially.

| Category | Metric | Formula / Core Meaning | When to Use |
| :--- | :--- | :--- | :--- |
| **Order-unaware** | **Precision@k** | $\frac{\text{Relevant docs in top } k}{k}$ | Measuring how clean/focused the top-$k$ results are. |
| **Order-unaware** | **Recall@k** | $\frac{\text{Relevant docs in top } k}{\text{Total relevant docs in corpus}}$ | Ensuring critical evidence is not missed before LLM/reranking. |
| **Order-unaware** | **F-score ($F_1$)** | $\frac{2 \times \text{Precision} \times \text{Recall}}{\text{Precision} + \text{Recall}}$ | Balancing Precision and Recall. |
| **Order-aware** | **AP / MAP** | $\text{AP} = \frac{1}{R} \sum_{k=1}^{K} \text{Precision}@k \times \text{rel}_k$; $\text{MAP} = \frac{1}{|Q|} \sum \text{AP}(q)$ | Finding **multiple** relevant documents and ranking them near the top. |
| **Order-aware** | **MRR** | $\frac{1}{|Q|} \sum_{q=1}^{|Q|} \frac{1}{\text{rank}_q}$ (reciprocal rank of first relevant hit) | When **one** correct result needs to appear as early as possible. |
| **Order-aware** | **nDCG@k** | $\text{DCG}@k = \sum_{i=1}^{k} \frac{\text{rel}_i}{\log_2(i + 1)}$, $\text{nDCG}@k = \frac{\text{DCG}@k}{\text{IDCG}@k}$ | Graded relevance (`0, 1, 2, 3...`), normalized to $[0, 1]$. |

Beyond offline retrieval metrics, production systems rely on:
- **Separating Retrieval vs. Generation Evaluation:** Tracking **Context Precision** and **Context Recall** (e.g., via **RAGAS**) to isolate whether errors stem from missing retrieved context or LLM hallucination.
- **A/B Testing:** Routing a small fraction of live traffic (e.g., 1%) to a new retrieval pipeline to validate real-world impact on click-through rates and user satisfaction.
- **Operational Criteria & Presentation:** Balancing indexing speed, query latency, explainability, and infrastructure/API costs, while choosing between **static snippets** (fixed openings/summaries) and **dynamic snippets** (extracting the exact passage matching the query).
