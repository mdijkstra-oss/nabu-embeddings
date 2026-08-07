# nabu-embeddings

nabu-embeddings is a Caddy proxy in front of OpenAI's embeddings endpoint. It attaches the API key server-side, so the browser never holds it.

## Running it

```sh
cp .env.example .env       # fill in OPENAI_API_KEY
docker compose up
```

It listens on `127.0.0.1:8082`. `/health` answers `ok` once it is up.

## Calling it

`POST /embeddings` takes an OpenAI embeddings request body and returns OpenAI's response unchanged. The proxy rewrites the path to `/v1/embeddings` upstream, so ask for `/embeddings` here rather than OpenAI's own path.

```sh
curl http://127.0.0.1:8082/embeddings \
  -H 'Content-Type: application/json' \
  -d '{"model": "text-embedding-3-small", "input": "hello"}'
```

## No client authentication

The listener checks nothing about the caller, so anything that reaches the port spends the key behind it. Loopback is what limits reach: `compose.yaml` publishes to `127.0.0.1` rather than `0.0.0.0`. Reaching this from another machine means putting something in front of it that authenticates.

Setting `CORS_ORIGIN` is not that. A browser honours it; anything else ignores it.

## Configuration

| variable | default | what it does |
|---|---|---|
| `OPENAI_API_KEY` | — | the key the proxy sends as `Authorization: Bearer`. Requests fail at OpenAI without it |
| `CORS_ORIGIN` | `*` | the origin allowed to call this from a browser. `*` allows any, and a value must be one origin, not a list |
| `EMBEDDINGS_PORT` | `8082` | the host port. Caddy listens on `8082` inside the container whatever this says |
