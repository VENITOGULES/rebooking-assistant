# Report: rebooking assistant exercise

All three checks pass:

```
T1 ✓ booking system down: gives up after 3 tries
T2 ✓ the answer lists its sources
T3 ✓ recall_at_k and the better retriever
tests and check: 6ca0f8f7
```

`check.py` and `tests/` were not changed. Code changes are only in `assistant/graph.py` and `assistant/retrieval.py`.

## How I approached it

1. Read the README, the graph and the three test files before touching code, so the fix matches what "done" means.
2. Ran the check to see the three failures.
3. Fixed one task at a time and reran the check after each one.

## Task 1: the booking system is down

**What was wrong.** `after_check_seats` sent every failure straight back to `check_seats`, with no limit. For flights starting with `UA9`, the loop never ended until LangGraph hit its recursion limit and raised an error.

**Fix.** The state now counts attempts:

- `booking_attempts: int` in `State`, and `MAX_BOOKING_ATTEMPTS = 3`.
- `check_seats` adds one on every call, whether it succeeds or fails.
- The router retries only while the call failed and attempts are under 3. Otherwise it goes to `write_answer` with `seats=None`, so the answer says "I could not check seats right now."

A healthy booking system is still called exactly once.

**In production I would add:** exponential backoff with jitter and a timeout per call, instead of retrying immediately. I would also log every failure, and add a circuit breaker so we stop calling a server we already know is down. LangGraph's `RetryPolicy` is an option too. I kept an explicit counter because it makes the fallback path visible in the graph and easy to test.

## Task 2: `sources` was always empty

**What was wrong.** `find_rules` already returned `sources`, but the key was not declared in the `State` TypedDict. LangGraph keeps only keys that exist in the state schema, so the update was dropped without any error.

**Fix.** I added `sources: list` to `State`. `ask.py` now prints the 3 passage ids.

**Lesson.** In LangGraph the state schema is the contract between nodes. A key that isn't declared disappears silently, so I'd add a test for every key that downstream code relies on.

## Task 3: recall@k and the better retriever

**recall@k** is the share of questions whose relevant passage id appears in the top k results:

```python
hits = sum(relevant in ranked[:k] for ranked, relevant in zip(ranked_ids, relevant_ids))
return hits / len(relevant_ids)   # 0.0 when there are no questions
```

**Results** (`python evaluate.py`):

| Retriever | recall@3 |
|---|---|
| by_title | 0.80 |
| **by_title_and_text** | **1.00** |

I set `CHOSEN_RETRIEVER = "by_title_and_text"`. Titles alone miss words that only appear in the passage text, such as "dog", "miles" or "voucher".

**Caveats and next steps**

- Ten questions is a small eval set: the 0.20 gap is two questions. I'd grow it with real, harder questions before trusting small differences.
- Recall only says whether the right passage was retrieved. I'd also track MRR (how high it ranks) and check that the answer is faithful to the passage it cites.
- On thousands of pages of real manuals, keyword overlap won't be enough. I'd use hybrid search (BM25 plus embeddings) with a reranker, and metadata filters such as fare type and policy version.

## How to run

```bash
docker compose run --rm check
docker compose run --rm check python evaluate.py
docker compose run --rm check python ask.py "Is a cancelled flight free to change?" UA900
```
