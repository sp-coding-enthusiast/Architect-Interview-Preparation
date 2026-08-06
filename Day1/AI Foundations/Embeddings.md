# Embeddings and Semantic Search

# Introduction

Traditional search looks for **exact keyword matches**.

Semantic search goes one step further by understanding the **meaning (semantics)** of the text rather than just matching words.

The technology that makes semantic search possible is called **Embeddings**.

Embeddings convert text into **numerical vectors** that capture the meaning of the text. Similar meanings produce vectors that are close together in a high-dimensional space.

---

# Traditional Search vs Semantic Search

## Traditional Search

Search is based on keywords.

Example

```text
Query:
Cheap cars

Document:
Affordable vehicles
```

Keyword search may fail because:

- "Cheap" ≠ "Affordable"
- "Cars" ≠ "Vehicles"

No exact keyword match means poor results.

---

## Semantic Search

The search engine understands that:

```text
Cheap ≈ Affordable

Cars ≈ Vehicles
```

It returns the correct document because it understands the meaning.

---

# What are Embeddings?

An **Embedding** is a numerical representation (vector) of text, images, audio, or other data.

Instead of storing words as plain text, AI models convert them into a list of numbers.

Example

```text
"Cat"

↓

[0.12, -0.83, 0.45, ...]
```

```text
"Dog"

↓

[0.10, -0.79, 0.43, ...]
```

The vectors are close together because **cats and dogs are semantically similar**.

---

# Why Convert Text into Numbers?

Computers cannot understand natural language directly.

Instead, they understand mathematics.

```text
Sentence

↓

Embedding Model

↓

Vector

↓

Mathematical Comparison
```

This allows AI systems to measure similarity between pieces of text.

---

# High-Level Flow

```text
Text

↓

Embedding Model

↓

Vector

↓

Vector Database

↓

Similarity Search

↓

Relevant Results
```

---

# Example

Sentence 1

```text
The weather is nice today.
```

Embedding

```text
[0.32, -0.41, 0.81, ...]
```

Sentence 2

```text
Today has beautiful weather.
```

Embedding

```text
[0.30, -0.39, 0.79, ...]
```

Even though the words are different, the vectors are very close because the meaning is similar.

---

# What is a Vector?

A vector is simply a list of numbers.

Example

```text
[1.2, 3.4, 5.6]
```

Real AI models use vectors with hundreds or thousands of dimensions.

Examples:

- 384 dimensions
- 768 dimensions
- 1024 dimensions
- 1536 dimensions
- 3072 dimensions

Each dimension captures part of the semantic meaning.

---

# Visualizing Embeddings

Imagine a 2D space.

```text
            Animal

 Dog ●

        Cat ●


                    Car ●

                         Truck ●
```

- Cat and Dog are close.
- Car and Truck are close.
- Animals are far from vehicles.

Real embedding spaces have hundreds or thousands of dimensions.

---

# How Embeddings are Generated

```text
Sentence

↓

Tokenizer

↓

Transformer Model

↓

Embedding Layer

↓

Vector
```

Popular embedding models include:

- OpenAI text-embedding models
- Sentence Transformers
- BERT-based embedding models
- E5
- BGE (BAAI General Embedding)

---

# Why Semantic Search Works

Semantic search compares **meaning**, not words.

Example

Query

```text
How do I reset my password?
```

Document

```text
Forgot your account credentials?
```

Keyword search:

```text
Password

↓

Not Found
```

Semantic search:

```text
Reset Password

↓

Forgot Credentials

↓

Same Meaning

↓

Match Found
```

---

# Similarity Search

Once text is converted into vectors:

```text
Query Vector

↓

Compare

↓

Document Vectors

↓

Most Similar Results
```

The nearest vectors are considered the most relevant.

---

# Distance Between Vectors

Vectors are compared using mathematical distance.

Common methods:

- Cosine Similarity
- Euclidean Distance
- Dot Product

The smaller the distance (or higher the cosine similarity), the more similar the meaning.

---

# Cosine Similarity

Most widely used.

It measures the angle between vectors.

```text
Vector A

↗

Vector B

↗
```

Small angle

↓

High similarity

---

Example

```text
Cosine = 1.0

Exactly Same Meaning
```

```text
Cosine = 0.95

Very Similar
```

```text
Cosine = 0.20

Mostly Different
```

---

# Example Search

Documents

```text
1. Apple releases new iPhone

2. Tesla launches EV

3. Microsoft introduces AI Copilot
```

Query

```text
Latest Apple phone
```

Keyword search may not understand:

```text
Phone

↓

iPhone
```

Semantic search recognizes:

```text
Phone ≈ iPhone

Apple ≈ Apple

↓

Returns Document 1
```

---

# Embeddings in RAG

RAG (Retrieval-Augmented Generation) uses embeddings extensively.

Flow

```text
Documents

↓

Chunking

↓

Embedding Model

↓

Vector Database

↓

Query

↓

Embedding

↓

Similarity Search

↓

Relevant Chunks

↓

LLM

↓

Answer
```

Embeddings are the foundation of the retrieval process.

---

# Vector Database

Embeddings are stored in a vector database.

Popular choices:

- Azure AI Search
- Pinecone
- Weaviate
- Milvus
- Qdrant
- Chroma
- FAISS (library)

---

# Example

Document

```text
Natural Gas Prices
```

↓

Embedding

```text
[0.82, -0.15, 0.44, ...]
```

Stored inside a vector database.

When the user searches:

```text
Gas Market

↓

Embedding

↓

Similarity Search

↓

Natural Gas Prices
```

The match is found even without exact keywords.

---

# Why Not Store Plain Text?

Searching millions of documents with text comparisons is slow and limited.

Vectors enable:

- Fast nearest-neighbor search
- Better relevance
- Language understanding
- Synonym matching
- Context-aware retrieval

---

# Semantic Search Workflow

```text
User Query

↓

Embedding Model

↓

Query Vector

↓

Vector Database

↓

Top-K Similar Documents

↓

LLM

↓

Final Answer
```

---

# Real-World Examples

## ChatGPT with RAG

```text
Company Documents

↓

Embeddings

↓

Vector Database

↓

Question

↓

Relevant Documents

↓

Answer
```

---

## E-Commerce

Query

```text
Comfortable running shoes
```

Semantic search finds:

```text
Lightweight athletic sneakers
```

even if "running shoes" isn't written exactly.

---

## Healthcare

Query

```text
Heart attack symptoms
```

Matches documents containing:

```text
Myocardial infarction signs
```

because the meanings are related.

---

# Advantages of Embeddings

- Understand semantic meaning
- Handle synonyms
- Better search relevance
- Support multilingual search
- Power RAG systems
- Improve recommendation engines
- Enable semantic clustering and classification

---

# Limitations

- Require storage for vectors
- Need an embedding model
- Semantic similarity is not perfect
- Embeddings must be regenerated when documents change
- High-dimensional vector search requires specialized indexes

---

# Best Practices

- Use high-quality embedding models.
- Chunk documents appropriately before generating embeddings.
- Store vectors in a vector database optimized for similarity search.
- Retrieve the Top-K most relevant chunks.
- Re-embed documents when content changes.
- Combine semantic search with keyword filtering when precision is important.

---

# Common Mistakes

## Using Keyword Search for Semantic Problems

```text
Query

↓

Keyword Match

↓

Poor Results
```

---

## Large Document Chunks

```text
50 Pages

↓

One Embedding
```

This reduces retrieval accuracy.

---

## Not Updating Embeddings

If documents change but embeddings are not regenerated:

```text
Old Embedding

↓

New Document

↓

Incorrect Search Results
```

---

## Ignoring Metadata

Store metadata (e.g., document ID, title, date, permissions) alongside embeddings to enable filtering and security.

---

# Interview Questions

## Q1. What are embeddings?

Embeddings are dense numerical vectors that represent the semantic meaning of data such as text, images, or audio. Similar items have vectors that are close together in vector space.

---

## Q2. Why does semantic search work?

Semantic search works because both the user's query and the documents are converted into embeddings. Similar meanings produce similar vectors, allowing relevant documents to be found even when exact keywords differ.

---

## Q3. What is the difference between keyword search and semantic search?

| Keyword Search | Semantic Search |
|----------------|-----------------|
| Matches exact words | Matches meaning |
| Poor with synonyms | Handles synonyms naturally |
| No context understanding | Understands context and relationships |
| Uses text indexes | Uses vector similarity |

---

## Q4. What is the role of embeddings in RAG?

Embeddings convert document chunks and user queries into vectors. A vector database retrieves the most semantically similar chunks, which are then provided to the LLM to generate grounded answers.

---

## Q5. Which similarity metric is most commonly used?

**Cosine Similarity** is the most widely used metric because it measures how similar two vectors are regardless of their magnitude, making it effective for semantic search.

---

# Key Takeaways

- **Embeddings** convert text into high-dimensional numerical vectors that capture semantic meaning.
- **Semantic search** compares vectors rather than keywords, allowing it to understand synonyms, context, and related concepts.
- Similar meanings produce vectors that are close together in vector space.
- **Cosine Similarity** is the most common technique for measuring vector similarity.
- Embeddings are the foundation of **RAG**, recommendation systems, semantic search, clustering, and many modern AI applications.
- A **vector database** stores embeddings and enables efficient nearest-neighbor searches across millions of documents.