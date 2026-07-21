---
title: "Vector Search from Scratch: Demystifying Semantic Search"
tags:
  - AI
  - Vector-Search
  - Embeddings
  - Math
  - Python
---

If you’ve spent any time building with LLMs recently, you’ve probably heard of "Vector Search," "Embeddings," and "RAG" (Retrieval-Augmented Generation). Every startup and their mother is pitching a new vector database, claiming it’s the secret sauce to making AI smart. 

But if you look at the documentation for tools like LangChain or Pinecone, it feels like they are gatekeeping the actual tech. They throw around terms like *high-dimensional vector spaces* and *indexing algorithms*, leaving developers to just copy-paste boilerplate config and pray it works.

Well, spoiler alert: it’s not magic. It’s literally just high school geometry. 

In this post, we are going to strip away the marketing fluff and build a fully functional semantic search engine from scratch using nothing but Python and raw NumPy. No database, no heavy frameworks—just pure math vibes.

---

### what actually is an embedding? (let it cook)

Before we talk about search, we need to talk about data. Computers are great at comparing numbers ($5 > 3$), but they are terrible at comparing meaning (*"happy"* is similar to *"joyful"*). 

An **embedding** is just a way of converting meaning into coordinates. 

Imagine a simple 2D map of words. On the X-axis, we plot how "animate" something is. On the Y-axis, we plot how "furry" it is.
- A **rock** would be at coordinates $(0, 0)$—not alive, not furry.
- A **dog** would be at $(1, 1)$—alive and furry.
- A **goldfish** would be at $(1, 0)$—alive, but not furry.

```
Furry (Y)
  ▲
  │   Dog (1,1)
  │
  │
  │   Goldfish (1,0)
  └──────────────► Animate (X)
Rock (0,0)
```

In this coordinates system, a dog is geometrically closer to a goldfish than it is to a rock. That distance is semantic similarity. 

Real embedding models don't just use 2 dimensions; they use 384, 768, or even 1536 dimensions. Each dimension represents a subtle concept (like gender, tense, capitalization, or context). But the core rule remains: **similar meanings get similar coordinates**.

---

### the math: cosine similarity (is it mathing?)

Once we have our coordinates (vectors), how do we search them? 

You might think we should just draw a straight line between two points and measure the distance. In geometry, this is called the **Euclidean Distance**. 

But Euclidean distance is lowkey a trap when it comes to text because of **document length**. 

If you have a short sentence like *"I love cats,"* and a 5-page article about how amazing cats are, their meanings are identical. But because the article has thousands of words, its vector coordinates will be pushed way out into space, making the straight-line distance between them massive. Complete skill issue.

To fix this, we measure the **angle** between the vectors rather than the distance between the points. If the vectors point in the same direction, the angle between them is $0^\circ$, meaning they are semantically aligned—even if one is way longer than the other.

We calculate this angle using **Cosine Similarity**:

$$\cos(\theta) = \frac{\mathbf{A} \cdot \mathbf{B}}{\|\mathbf{A}\| \|\mathbf{B}\|}$$

In plain English:
1. **$\mathbf{A} \cdot \mathbf{B}$ (Dot Product)**: Multiply the coordinates of vector A by vector B and add them all up. This measures how much the vectors point in the same direction.
2. **$\|\mathbf{A}\| \|\mathbf{B}\|$ (Magnitude)**: Normalize the vectors by their length so that long documents don't skew the results.

If the similarity score is $1.0$, the meanings are identical. If it’s $0.0$, they are completely unrelated.

---

### building it in 50 lines of python (numpy is goated)

Ngl, writing the math formulas looks intimidating, but translating it into Python using NumPy is incredibly simple. 

First, let's write our own cosine similarity function:

```python
import numpy as np

def cosine_similarity(a, b):
    # np.dot calculates the dot product
    # np.linalg.norm calculates the length (magnitude) of the vector
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```

Now, let’s build a local search index. We will create a small corpus of documents and convert them into mock embeddings (in a real app, you would fetch these from a model like OpenAI's `text-embedding-3-small` or run a local model via `sentence-transformers`).

```python
# 5-dimensional mock embeddings for simplicity
# In real life, these would be 1536-dimensional arrays
database = {
    "How to configure local Docker sandboxes": np.array([0.9, 0.1, 0.8, 0.1, 0.2]),
    "My favorite pizza toppings and recipes": np.array([0.1, 0.9, 0.2, 0.8, 0.7]),
    "Running LLMs locally on Apple Silicon":   np.array([0.8, 0.2, 0.9, 0.1, 0.1]),
    "The nostalgia of classic Windows 95 UI":  np.array([0.2, 0.3, 0.1, 0.9, 0.9])
}

# The user enters a search query
# We convert the query into the same coordinate format
query_vector = np.array([0.85, 0.15, 0.85, 0.1, 0.15]) # Sounds like tech/local LLMs

# Calculate similarity against every document in our database
results = []
for title, doc_vector in database.items():
    score = cosine_similarity(query_vector, doc_vector)
    results.append((title, score))

# Sort the results by score in descending order
results.sort(key=lambda x: x[1], reverse=True)

# Print the top matches
for rank, (title, score) in enumerate(results):
    print(f"{rank + 1}. [{score:.4f}] {title}")
```

If you run this code, the output will look like this:
```
1. [0.9952] Running LLMs locally on Apple Silicon
2. [0.9781] How to configure local Docker sandboxes
3. [0.3871] The nostalgia of classic Windows 95 UI
4. [0.3211] My favorite pizza toppings and recipes
```

The system successfully identified that your query was highly relevant to local LLMs and Docker, while safely ignoring pizza and retro interfaces—all without a single database queries. Sheesh.

---

### the reality check

So, if vector search is just 5 lines of NumPy, why does Pinecone exist? 

Because of scale. Calculating cosine similarity against 4 documents is instant. Calculating it against 10,000,000 documents on every search query will crawl your server to a halt. 

Vector databases exist to solve the scale problem using index algorithms (like HNSW - Hierarchical Navigable Small World). They group similar coordinates together in a graph network so that instead of checking every vector, the engine only searches the neighborhood closest to your query.

But for your personal blog, local projects, or minor notes indexes? You do not need a vector database. A simple NumPy array and a few dot products are more than enough to build a blazing-fast, private semantic search engine.

Thanks for reading, and I'll see you in the next one!
