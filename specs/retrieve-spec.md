# Spec: `retrieve()`

**File:** `retriever.py`
**Status:** Spec incomplete — fill in all blank fields before implementing

---

## Purpose

Given a user's natural language query, find the most relevant chunks from the vector store using semantic similarity search. Return them ranked by relevance so that `generate_response()` can use them as context.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | `str` | The user's natural language question |
| `n_results` | `int` | Maximum number of chunks to return (default: `N_RESULTS` from `config.py`) |

**Output:** `list[dict]`

Each dict in the returned list must contain exactly these keys:

| Key | Type | Description |
|-----|------|-------------|
| `"text"` | `str` | The chunk text |
| `"game"` | `str` | The game name this chunk came from |
| `"distance"` | `float` | Cosine distance score — lower means more similar to the query |

Results should be ordered from most to least relevant (lowest to highest distance). Returns an empty list `[]` if the collection contains no documents.

---

## Design Decisions

*Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours.*

---

### Query approach

*Describe how you will use `_collection.query()` to find relevant chunks. What arguments will you pass, and why?*

```
We will be passing three arguments:

query_texts=[query]
- Wraps the user's string in a list (the API expects a list to support batch qqueries, even if we're only processing one query).

n_results=n_results
- Caps how many chunks come back. Passed through from the function parameter so the caller (and config.py's N_RESULTS default) controls the limit without touching this function.

include=["documents", "metadatas", "distances"]
- Requests exactly the three fields needed to build the return dicts. "embeddings" is omitted to keep the response payload small.
```

---

### Return structure

*Sketch out what one item in your return list looks like as a concrete example. Where does each field come from in the query results?*

```
{
    "text":     "Roll both dice to produce resources.",  # results["documents"][0][i]
    "game":     "Catan",                                 # results["metadatas"][0][i]["game"]
    "distance": 0.21,                                    # results["distances"][0][i]
}
```

---

### Handling the nested result structure

*`_collection.query()` returns nested lists. Describe what index you need to access to get the actual list of results for a single query, and why the nesting exists.*

```
_collection.query() accepts a list of query strings so it can batch multiple
queries in one call. Every field in the response is therefore a list-of-lists. The outer index picks the query, the inner index picks the hit.

Since we pass exactly one query string, we use [0] on every field to unwrap
our results:

    results["documents"][0]  → ["chunk text 1", "chunk text 2", ...]
    results["metadatas"][0]  → [{"game": "Catan"}, {"game": "Uno"}, ...]
    results["distances"][0]  → [0.21, 0.45, ...]

ChromaDB returns these already sorted lowest-distance-first, so no manual
sorting is needed.
```

---

### Relevance threshold

*Will you filter out results above a certain distance score, or return all `n_results` regardless of how relevant they are? What are the tradeoffs of each approach?*

```
Return all n_results below a hard threshold.

No threshold:
  + Simple — no constant to pick or calibrate.
  + generate_response() always receives some context to work with.
  - Weak, noisy matches may reach the LLM if no closely relevant chunk exists.

Hard threshold (e.g. drop anything above distance 0.7):
  + Limits noise; helps when the user asks about a game not in the database.
  - Requires tuning against real queries — wrong cutoff silently drops good
    results or lets bad ones through.
```

---

### Edge cases

*How does your implementation behave when: (a) the collection is empty, (b) the query matches no chunks well, (c) the query matches chunks from multiple games?*

```
(a) Empty collection:
    The guard at the top of retrieve() catches this — _collection.count() == 0
    returns [] immediately. Calling .query() on an empty collection raises an
    error, so the early return is necessary, not just defensive.

(b) Query matches no chunks well (all distances are high):
    The 0.7 threshold filters out anything above that distance, so if all
    n_results hits are weak matches, retrieve() returns []. generate_response()
    will receive no context and should handle that gracefully.

(c) Query matches chunks from multiple games:
    Results are ranked purely by distance regardless of game, so a cross-game
    query (e.g. "how do I win?") can return chunks from Catan, Monopoly, and
    Uno in the same list. The "game" field on each dict lets generate_response()
    attribute each chunk to the right rulebook.
```

---

## Implementation Notes

*Fill this in after implementing, before moving to Milestone 3.*

**Test query and top result returned:**

```
Query: What happens when you roll a 7?
Top result game: Catan
Distance score: 0.466
Does it make sense? Yes, the top result seems to be mapping to a chunk of the Catan rules that mentions what happens when you roll a 7.
```

**One thing about the query results that surprised you:**

```
I expected the top 3 chunks returned from the query to have lower distance scores. In the example they seemed to range from 0.1-0.2 while my results seemed to range from 0.4-0.6. Though This might be a result of different chunk sizes.
```
