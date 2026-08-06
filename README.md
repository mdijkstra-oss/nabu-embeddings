# nabu-embeddings

A Caddy proxy in front of OpenAI's embeddings endpoint. It attaches the API key server-side, so the browser never holds it. The body is forwarded byte for byte.

## Running it

```sh
cp .env.example .env       # OPENAI_API_KEY
docker compose up
```

It listens on `127.0.0.1:8082`.

> [!WARNING]
> There is no client authentication, so anything that reaches the port spends the key behind it. Loopback is the only thing that limits what can: `compose.yaml` publishes to `127.0.0.1` rather than `0.0.0.0`, and any origin is allowed by default. Reaching this from another machine means putting something in front of it that authenticates.

## Configuration

| variable | default | holds |
|---|---|---|
| `OPENAI_API_KEY` | — | the key, injected as `Authorization: Bearer`. Unset means every request answers `401` from OpenAI |
| `CORS_ORIGIN` | `*` | any origin may call this. Set it to one origin to restrict, not a list |
| `EMBEDDINGS_PORT` | `8082` | the host port. The listener inside the container is always `8082` |
