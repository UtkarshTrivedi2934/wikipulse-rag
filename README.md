# wikipulse-rag

Chat with live data. A retrieval-augmented generation (RAG) app that answers questions about what is being edited on English Wikipedia *right now*, with cited, time-aware answers.

Built with Streamlit and Gemini.

## Why it's useful

A normal LLM knows nothing about what happened five minutes ago. RAG lets it answer questions about fresh data without retraining anything: the app retrieves the latest events, hands them to the model, and the model answers only from them. The "live" part is that the index keeps absorbing new events while you chat.

## How it works

1. **Pull.** When a question comes in (at most once every 20 seconds), the app fetches the latest ~300 edits from Wikipedia's `recentchanges` API and skips events it has already seen.
2. **Embed and index.** New events are converted into embeddings (lists of numbers representing meaning) with the Gemini embedding API and added to a shared in-memory index, capped at 3,000 events.
3. **Retrieve.** The question is expanded by the LLM into related search terms. The app then finds the 40 most relevant events using embedding similarity, a keyword boost, and a small boost for recent events.
4. **Answer.** Gemini answers using only those events, citing event IDs and how long ago each edit happened. It may use general knowledge to judge whether a page fits the topic (e.g. recognizing a footballer's name) and marks those cases as inferred.
5. **Stay honest.** If the events don't contain what you asked about, it says so instead of guessing.

Every event is reduced to three fields: `id`, `ts` (UTC ISO timestamp), `text`.

## Example

> **Q:** Which pages are being edited about football or soccer?
>
> The app finds recent edits to national team and club season pages, cites each event ID with its age, and reports how recent the newest relevant edit is.

## Known limitations

- Pages whose titles are only a person's name carry little meaning for the embedder, so some relevant pages can be missed. Query expansion and LLM judgment reduce this, but hybrid search (BM25 + embeddings) would help further.
- The index lives in memory, so it resets when the app sleeps. It backfills from the API on wake.
- Free hosting tiers sleep when idle, so the first visit after a quiet period is slow.

## Where this goes

Wikipedia is a stand-in feed. The same structure works for any source that can be turned into `id`, `ts`, `text` records: sales events, pipeline logs, support tickets, sensor readings, or news. To adapt it, replace `pull_recent()`. For production, swap the in-memory index for pgvector or a similar store, and move the logic behind a FastAPI endpoint with a streaming or database source.
