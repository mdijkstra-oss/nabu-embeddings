# nabu-embeddings

The `/embeddings` route [nabu-frontend](https://github.com/mdijkstra-oss/nabu-frontend) calls, and nothing else.

A browser cannot hold a provider key. This is a Caddy listener that takes an OpenAI-shaped embeddings request from the frontend's origin, adds `Authorization` from the environment, and forwards it to OpenAI unchanged. The body is never read, so what the frontend sends is what OpenAI answers.

> [!WARNING]
> The frontend does not call this yet. It sends `{input}` with no `model`, which OpenAI rejects, and it has one host variable for both the agent gateway and this. [Pointing the frontend at it](#pointing-the-frontend-at-it) is what has to change.

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

That envelope is what `app/lib/embeddings/client.ts` already parses: `data` sorted by `index`, one float array each. A thousand and twenty-four of them, so the arrays are cut short above.

With no key set, the same call answers `401` from OpenAI. An error carrying OpenAI's own wording rather than Caddy's is the thing to look for, because it means the path rewrite, the `Host` override and the injected header all arrived.

## Pointing the frontend at it

Three changes, all in `nabu-frontend`, none of them possible from here.

**The request needs a `model`.** `app/lib/embeddings/client.ts` builds its body as `JSON.stringify({ input })`. Sent through this listener with a working key, that body comes back:

```json
{"error": {"message": "you must provide a model parameter", "type": "invalid_request_error"}}
```

A proxy that forwards bodies untouched cannot supply the field.

**It has to be `text-embedding-3-large` at `dimensions: 1024`.** Every vector already written to a `.embeddings.hidden.md` companion came from that model at that width. Sending anything else produces vectors that are compared against the existing ones by cosine similarity and mean nothing next to them.

**The host is a second variable.** `VITE_LLM_HOST` is the agent gateway, and `getLlmHost()` in `app/lib/agent/env.ts` feeds both the chat client and every embeddings call site. Splitting them means a `VITE_EMBEDDINGS_HOST` with its own accessor, and nine call sites across eight files switched to it. Only `app/lib/agent/client/fetch.ts` keeps the old one.

> [!NOTE]
> The embedding cache keys on chunk hash alone — no model, no dimension — so changing either silently keeps every stale vector rather than re-embedding. That is a `nabu-frontend` concern, and it decides whether a model change needs the companion files wiped.

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

`app/lib/embeddings/client.ts` is where the two body fields go, and `app/lib/agent/env.ts` is where the host splits in two. Neither is large; the decision that comes first is what happens to the companion files already holding 1024-float vectors, because a re-embed is cheap only while the corpus is small.
