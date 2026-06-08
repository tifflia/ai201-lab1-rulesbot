# Spec: `generate_response()`

**File:** `generator.py`
**Status:** Spec incomplete — fill in all blank fields before implementing

---

## Purpose

Given a user query and a list of retrieved rule chunks, generate a response that directly answers the question using only the retrieved text as context. The response must be grounded — it should not draw on the model's general knowledge of board games, only on what was retrieved.

---

## Input / Output Contract

**Inputs:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `query` | `str` | The user's original question |
| `retrieved_chunks` | `list[dict]` | Ranked list of chunks from `retrieve()`, each with `"text"`, `"game"`, and `"distance"` |

**Output:** `str`

A plain string containing the response to show the user. The response should:
- Answer the question using only the retrieved rule text
- Identify which game the answer comes from
- Acknowledge clearly when the answer is not found in the loaded rules

Returns a fallback string (not an error) when `retrieved_chunks` is empty.

---

## Design Decisions

*Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours.*

---

### Context formatting

*How will you format the retrieved chunks before passing them to the LLM? Describe the structure — not the code. Consider: will you label chunks by game? Include distance scores? Separate chunks with delimiters?*

```
For the context, I will label the retrieved chunks by game so that they can be cited correctly (i.e. [Catan]). I will not include distance scores as this ends up being unnecessary noise. I will also try to separate the chunks with some sort of delimiter like '---', to keep the boundaries clear.
```

---

### System prompt — grounding instruction

*Write the exact system prompt instruction you will use to prevent the model from answering beyond the retrieved text. This is the most important design decision in this function.*

```
Answer using only the rule text provided below. If the answer is not contained in the provided text, say so explicitly — do not draw on outside knowledge or fill in gaps from what you know about board games.
```

---

### System prompt — citation instruction

*Write the exact instruction you will use to tell the model to identify which game its answer comes from.*

```
"Always identify which game your answer comes from."
```

---

### Fallback behavior

*What should the response say when the answer isn't found in the loaded rule books? Write the exact fallback message.*

```
"I couldn't find anything relevant in the loaded rule books. Try rephrasing your question — or check that your ingestion pipeline is working."
```

---

### Handling low-relevance chunks

*`retrieved_chunks` may include chunks with high distance scores (weak relevance). Will you filter these out before building context, pass them all in, or handle them another way? What are the tradeoffs?*

```
Low-relevance chunks are handled in retriever.py in the retrieve() function. All chunks with distance scores greater than 0.7 are filtered out. Doing this prevents noise in the context we provide for the generation, but also means we can potentially drop relevant or good results.
```

---

### Message structure

*Describe how you will structure the messages list for the API call — what goes in the system message vs. the user message?*

```
The system messages contains the initial grounding instructions, citation instructions, and fallback behavior. The user message contains the rules context chunks and the actual query.
```

---

## Implementation Notes

*Fill this in after implementing and testing.*

**Test query and response:**

```
Query: What happens if you roll a 7 in Catan?
Response: In Catan, when a 7 is rolled, no resources are produced. Additionally, every player with more than 7 resource cards in hand must discard half (rounded down) of their resource cards. The player who rolled the 7 also gets to move the robber to any terrain hex and steal one resource card from another player.
Correctly grounded? yes
Cited the right game? yes
```

**One thing you changed from your original spec after seeing the actual output:**

```
Explicitly including the citation instructions in the system message.
```
