# Vector Similarity, Chunking, Tokenization & Context Windows

# Introduction

Large Language Models (LLMs) do not search documents using keywords.

Instead, they:

1. Convert text into **embeddings (vectors)**
2. Compare vectors using **similarity algorithms**
3. Retrieve the most relevant chunks
4. Send those chunks to the LLM

The quality of a RAG (Retrieval-Augmented Generation) system depends heavily on:

- Embeddings
- Similarity Metrics
- Chunking Strategy
- Tokenization
- Context Window

---

# Vector Similarity

After converting text into embeddings, we need a way to measure how similar two vectors are.

The three most common similarity metrics are:

- Cosine Similarity
- Dot Product
- Euclidean Distance

---

# 1. Cosine Similarity

## What is Cosine Similarity?

Cosine Similarity measures the **angle between two vectors**, ignoring their length (magnitude).

Two vectors pointing in the same direction have high similarity.

---

## Visualization

```text
Vector A

↗

↗ Vector B
```

Small angle

↓

High similarity

---

If vectors point in opposite directions

```text
↗ Vector A



↙ Vector B
```

Large angle

↓

Low similarity

---

## Formula

Cosine Similarity is calculated as:

```
Cos(A, B) = (A · B) / (||A|| × ||B||)
```

Where:

- **A · B** = Dot Product
- **||A||** = Magnitude of A
- **||B||** = Magnitude of B

---

## Interpretation

| Value | Meaning |
|---------|---------|
| 1 | Exactly same direction |
| 0.9 | Very similar |
| 0.7 | Similar |
| 0 | Unrelated |
| -1 | Opposite direction |

---

## Example

Query

```text
Cheap cars
```

Document

```text
Affordable vehicles
```

Even though no keywords match,

their embeddings point in nearly the same direction.

```
Cosine Similarity ≈ 0.95
```

Document is retrieved.

---

## Advantages

- Most commonly used
- Ignores vector length
- Excellent for semantic search
- Works well with normalized embeddings

---

## Disadvantages

- Slightly more computational work than dot product
- Requires vector normalization for best performance

---

# 2. Dot Product

## What is Dot Product?

Dot Product measures how much two vectors point in the same direction **while also considering their magnitude**.

Unlike cosine similarity,

vector length matters.

---

## Formula

```
A · B = A1B1 + A2B2 + A3B3 + ...
```

---

## Example

Vector A

```
[2,3]
```

Vector B

```
[4,5]
```

Dot Product

```
2×4 + 3×5

=

23
```

Larger values generally indicate stronger similarity (assuming vectors are comparable).

---

## Visualization

```text
Vector A

↗

↗ Vector B

Large Dot Product
```

---

## Advantages

- Extremely fast
- Preferred by many vector databases
- Works well when embeddings are already normalized
- Efficient on GPUs

---

## Disadvantages

Magnitude affects the score.

Longer vectors may receive larger scores even if their semantic meaning isn't more similar.

---

## Where Used

- OpenAI embeddings (normalized)
- FAISS
- Pinecone
- Milvus
- Vector search engines

---

# Cosine Similarity vs Dot Product

| Cosine | Dot Product |
|----------|-------------|
| Uses angle | Uses angle + magnitude |
| Ignores length | Includes length |
| Common in semantic search | Common in vector databases |
| Range: -1 to 1 | No fixed range |

---

# 3. Euclidean Distance

## What is Euclidean Distance?

Measures the **straight-line distance** between two vectors.

Smaller distance means greater similarity.

---

## Visualization

```text
A ●

      \

       \

        ● B
```

Shortest distance between two points.

---

## Formula

```
Distance = √((x₂-x₁)² + (y₂-y₁)²)
```

---

## Example

Vector A

```
(2,3)
```

Vector B

```
(4,6)
```

Distance

```
√((4-2)²+(6-3)²)

=

√13

≈3.6
```

---

## Interpretation

| Distance | Meaning |
|-----------|----------|
| 0 | Same vector |
| Small | Similar |
| Large | Different |

---

## Advantages

- Simple
- Easy to understand
- Useful for clustering

---

## Disadvantages

- Less effective in very high-dimensional spaces
- Sensitive to vector magnitude

---

# Comparison

| Metric | Uses Angle | Uses Magnitude | Best For |
|----------|------------|----------------|-----------|
| Cosine Similarity | Yes | No | Semantic Search |
| Dot Product | Yes | Yes | Vector Databases |
| Euclidean Distance | No | Yes | Clustering |

---

# Chunking

## What is Chunking?

LLMs cannot process an entire book or large document efficiently.

Documents are divided into **smaller pieces called chunks**.

```text
Large Document

↓

Chunk 1

Chunk 2

Chunk 3

Chunk 4
```

Each chunk gets its own embedding.

---

# Why Chunking?

Suppose a PDF has

```
300 pages
```

Embedding the entire document creates one huge vector.

Instead,

split into:

```text
Page 1-2

↓

Chunk

↓

Embedding
```

Now search becomes more accurate.

---

# Fixed-Size Chunking

Example

```text
Every 500 words

↓

One Chunk
```

Advantages

- Simple
- Fast

Disadvantages

May split sentences in the middle.

---

# Sentence-Based Chunking

```text
Sentence 1

Sentence 2

Sentence 3

↓

Chunk
```

Advantages

Keeps natural meaning.

---

# Paragraph Chunking

```text
Paragraph

↓

Chunk
```

Best for documentation.

---

# Section-Based Chunking

```text
Chapter

↓

Section

↓

Chunk
```

Useful for manuals and books.

---

# Semantic Chunking

Instead of fixed size,

split when the topic changes.

Example

```text
Authentication

↓

Chunk

Caching

↓

Chunk

Redis

↓

Chunk
```

Produces higher-quality retrieval.

---

# Overlapping Chunks

Without overlap

```text
Chunk 1

Sentence A

Sentence B
```

```text
Chunk 2

Sentence C

Sentence D
```

Context may be lost.

---

With overlap

```text
Chunk 1

A

B

C
```

```text
Chunk 2

C

D

E
```

Shared content preserves context.

Typical overlap

```
10–20%
```

---

# Best Chunk Size

| Content Type | Recommended Size |
|---------------|------------------|
| FAQs | 100–300 tokens |
| API Docs | 300–500 tokens |
| Technical Docs | 500–800 tokens |
| Books | 800–1200 tokens |
| Legal Documents | 1000+ tokens |

---

# Tokenization

## What is Tokenization?

Before an LLM processes text,

it converts text into **tokens**.

```text
Sentence

↓

Tokenizer

↓

Tokens

↓

LLM
```

---

Example

```text
ChatGPT is amazing
```

May become

```text
Chat

G

PT

is

amazing
```

Tokens are **not always words**.

---

# Word vs Token

Sentence

```text
Artificial Intelligence
```

Words

```
2
```

Tokens

```
3–5
```

depending on tokenizer.

---

# Why Tokenization?

LLMs understand tokens,

not raw text.

Everything is converted into tokens before processing.

---

# Common Tokenizers

- Byte Pair Encoding (BPE)
- SentencePiece
- WordPiece
- Unigram Tokenizer

---

# Token Example

```text
NaturalGasPrice
```

May become

```text
Natural

Gas

Price
```

---

# Why Tokens Matter

LLMs have limits.

Example

```
Context Window

128K Tokens
```

If the input exceeds this,

older content must be removed or truncated.

---

# Context Window

## What is Context Window?

The **Context Window** is the maximum number of tokens an LLM can consider at one time.

It includes:

- System Prompt
- User Prompt
- Retrieved Chunks
- Conversation History
- Generated Response

---

# Visualization

```text
System Prompt

↓

Conversation

↓

Retrieved Chunks

↓

Current Question

↓

Response

↓

Total Tokens
```

Everything counts toward the context window.

---

# Example

Suppose an LLM supports

```
128K Tokens
```

Available usage

```
System Prompt

5K

Conversation

20K

Documents

80K

Response

10K

Total

115K
```

Still within the limit.

---

# Why Context Window Matters

If important information is outside the context window,

the model cannot use it.

Example

```text
Document

500 Pages

↓

Only First 100 Pages Fit

↓

Remaining Pages Ignored
```

---

# Context Window in RAG

```text
User Question

↓

Embedding

↓

Vector Search

↓

Top 5 Chunks

↓

LLM Context

↓

Answer
```

Only the retrieved chunks consume the context window, not the entire document collection.

---

# Best Practices

## Embeddings

- Use high-quality embedding models.
- Normalize vectors when appropriate.
- Store embeddings in a vector database.

---

## Similarity Search

- Prefer **Cosine Similarity** for semantic search.
- Use **Dot Product** when embeddings are normalized and supported by the vector database.
- Use **Euclidean Distance** mainly for clustering and specialized ML tasks.

---

## Chunking

- Keep chunks semantically meaningful.
- Use overlapping chunks.
- Avoid splitting sentences.
- Tune chunk size based on document type.

---

## Tokenization

- Estimate token usage before sending prompts.
- Reserve tokens for the model's response.
- Be aware that tokens are not the same as words.

---

## Context Window

- Retrieve only the most relevant chunks.
- Remove irrelevant conversation history when necessary.
- Use summarization for long conversations.
- Avoid filling the context window with unnecessary information.

---

# Interview Questions

## Q1. What is Cosine Similarity?

Cosine Similarity measures the angle between two vectors. It is the most widely used similarity metric for semantic search because it focuses on meaning rather than vector magnitude.

---

## Q2. What is the difference between Cosine Similarity and Dot Product?

| Cosine Similarity | Dot Product |
|-------------------|-------------|
| Compares vector direction | Compares direction and magnitude |
| Ignores vector length | Includes vector length |
| Output range is -1 to 1 | No fixed output range |
| Common for semantic search | Common in vector databases and ML models |

---

## Q3. Why is chunking important in RAG?

Chunking breaks large documents into manageable pieces so embeddings represent focused topics. This improves retrieval accuracy and keeps retrieved content within the model's context window.

---

## Q4. What is tokenization?

Tokenization is the process of converting text into smaller units called tokens, which are the basic input units understood by LLMs.

---

## Q5. What is a context window?

A context window is the maximum number of tokens an LLM can process in a single request. It includes prompts, retrieved documents, conversation history, and the model's response.

---

# Key Takeaways

- **Cosine Similarity**, **Dot Product**, and **Euclidean Distance** are the primary methods for comparing embedding vectors.
- **Cosine Similarity** is the preferred choice for semantic search because it compares meaning rather than vector size.
- **Chunking** improves retrieval quality by splitting large documents into meaningful sections before generating embeddings.
- **Tokenization** converts text into tokens, which are the units processed by LLMs.
- The **Context Window** limits how much information an LLM can consider at once, making efficient retrieval and prompt design essential for high-quality RAG systems.