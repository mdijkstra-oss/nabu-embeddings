# nabu-embeddings

The `/embeddings` route [nabu-frontend](https://github.com/mdijkstra-oss/nabu-frontend) calls, and nothing else.

A browser cannot hold a provider key. This is a Caddy listener that takes an OpenAI-shaped embeddings request from the frontend's origin, adds `Authorization` from the environment, and forwards it to OpenAI unchanged. The body is never read, so what the frontend sends is what OpenAI answers.

## Running it

```sh
cp .env.example .env       # OPENAI_API_KEY
docker compose up
```

The listener publishes `8082`. One call embeds two texts and shows the whole chain resolving:

```console
$ curl -s -X POST http://localhost:8082/embeddings \
    -H 'Content-Type: application/json' \
    -d '{"input":["the first text","the second text"],"model":"text-embedding-3-large","dimensions":1024}'
{
  "object": "list",
  "data": [
    {"index": 0, "object": "embedding", "embedding": [0.048004, 0.011833, -0.002096, ...]},
    {"index": 1, "object": "embedding", "embedding": [0.024200, 0.024109, -0.021210, ...]}
  ],
  "model": "text-embedding-3-large",
  "usage": {"prompt_tokens": 6, "total_tokens": 6}
}
```

That envelope is what `app/lib/embeddings/client.ts` parses: `data` sorted by `index`, one float array each. A thousand and twenty-four of them, so the arrays are cut short above.

With no key set, the same call answers `401` from OpenAI. An error carrying OpenAI's own wording rather than Caddy's is the thing to look for, because it means the path rewrite, the `Host` override and the injected header all arrived.

## Pointing the frontend at it

`VITE_EMBEDDINGS_HOST` in `nabu-frontend` is this listener's address, separate from `VITE_LLM_HOST` so that the agent gateway and this can move independently.

The model and the width travel in the body, because a proxy that forwards bodies untouched cannot supply them. `VITE_EMBEDDINGS_MODEL` and `VITE_EMBEDDINGS_DIMENSIONS` are where they come from, defaulting to `text-embedding-3-large` at `1024`.

Neither is a preference. Every vector in a `.embeddings.hidden.md` companion was written at one model and one width, and a vector scored against another width returns a number rather than an error, so changing either is a re-embedding of the whole corpus.

> [!IMPORTANT]
> A new width performs that re-embedding itself: `diffChunks` counts a stored vector of the wrong length as a chunk it has never seen. A new model at the same width does not, because nothing records which model wrote an entry. Delete every companion by hand when changing the model alone.

## What is not translated

The body passes through byte for byte, which fixes the shape of what this can serve. OpenAI's own `/v1/embeddings` and anything that copies it work; Google's native endpoints do not, because they put the model in the URL path, name the text `content` rather than `input`, and return floats at `embedding.values` rather than `data[].embedding`.

Google's OpenAI compatibility layer is the way in if it is ever wanted — it covers embeddings, and the change would be an endpoint and a key, not code.

📘 [OpenAI compatibility | Gemini API](https://ai.google.dev/gemini-api/docs/openai)
> "Text embeddings measure the relatedness of text strings"

`task_type` is the cost of staying this simple. Google's native embeddings distinguish `RETRIEVAL_QUERY` from `RETRIEVAL_DOCUMENT`, and the frontend embeds both — HyDE passages are queries, chunk text is documents. OpenAI has no such field, so the distinction is not available to send.

## Routes

| route | answers |
|---|---|
| `POST /embeddings` | rewritten to `/v1/embeddings` and forwarded to `https://api.openai.com` with the key attached |
| `OPTIONS /embeddings` | `204` with the CORS headers, so the browser's preflight passes |
| `GET /health` | `ok`, liveness only |
| anything else | `404` |

## Configuration

| variable | default | holds |
|---|---|---|
| `OPENAI_API_KEY` | — | the key, injected as `Authorization: Bearer`. Unset means every request answers `401` from OpenAI |
| `CORS_ORIGIN` | `http://localhost:5173` | the single origin allowed to call this. The frontend's dev server, or wherever it is served from |
| `PORT` | `8082` | the host port, published on `127.0.0.1` only. The listener inside the container is always `8082` |

`CORS_ORIGIN` takes one origin, not a list. The frontend sends `X-Session-ID` and `X-Project-ID` alongside `Content-Type`, which makes every call a preflighted one, and all three are named in `Access-Control-Allow-Headers`.

There is **no client authentication**, so anything that reaches the port spends the key behind it. That is why `compose.yaml` publishes to loopback rather than `0.0.0.0`, and why reaching this from another machine means putting something in front of it that authenticates as its actual job.

## Next: nabu-frontend

The app that calls this. Its README covers what the vectors are made of — how a document is chunked, what a companion file holds, and the query cascade that scores one chunk against another.
