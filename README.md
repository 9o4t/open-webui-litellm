# Deploy and Host Open WebUI + LiteLLM on Railway

Team chat over **your** models: [Open WebUI](https://github.com/open-webui/open-webui) talks only to [LiteLLM](https://www.litellm.ai/). This is the ChatGPT Team replacement on [c17.io/stacks/open-webui-litellm](https://c17.io/stacks/open-webui-litellm).

**Provider API keys are BYO and optional.** The stack boots without `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`. You add keys in LiteLLM (or as optional env vars) when you want models to answer.

## About Hosting Open WebUI + LiteLLM

Open WebUI is the ChatGPT-style UI. LiteLLM is an OpenAI-compatible proxy: one URL, many providers, virtual keys, spend logs. Agents (Cursor, n8n, your bots) should point at LiteLLM, not at Open WebUI.

On Railway this is three services:

| Service | Source | Public | Notes |
|---|---|---|---|
| `open-webui` | `ghcr.io/open-webui/open-webui` | Yes | `PORT` native. Health `GET /health`. Volume `/app/backend/data`. |
| `litellm` | `docker.litellm.ai/berriai/litellm:main-stable` | Yes | So agents can hit the proxy. Health `GET /health/liveliness`. |
| `Postgres` | Railway plugin | No | LiteLLM models, virtual keys, spend. Open WebUI stays on SQLite on the volume unless you later set `DATABASE_URL`. |

Open WebUI is pre-wired to LiteLLM over Railway private networking. First Open WebUI signup becomes admin; then disable public signup. LiteLLM admin UI is `/ui` on the LiteLLM domain; password is `LITELLM_MASTER_KEY`.

## Common Use Cases

- Replace ChatGPT Team seats: one UI, many models, your keys
- Shared proxy for Cursor, n8n, and bots (`OPENAI_API_BASE` = LiteLLM)
- Add a local Ollama later without changing the UI (LiteLLM model row)
- Audit prompts on a box you own instead of a vendor log

## Dependencies for Open WebUI + LiteLLM Hosting

### Required (wired or generated — you do not type these)

| Variable | Service | Source |
|---|---|---|
| `PORT` | both apps | Injected. Open WebUI reads `PORT`. LiteLLM start: `litellm --port $PORT --host 0.0.0.0` (or target port `4000`). |
| `WEBUI_SECRET_KEY` | open-webui | Generated. JWT / session secret. |
| `WEBUI_AUTH` | open-webui | `true` |
| `ENABLE_OLLAMA_API` | open-webui | `false` (no host Ollama on Railway) |
| `OPENAI_API_BASE_URL` | open-webui | `http://${{litellm.RAILWAY_PRIVATE_DOMAIN}}:${{litellm.PORT}}/v1` |
| `OPENAI_API_KEY` | open-webui | `${{litellm.LITELLM_MASTER_KEY}}` (the proxy key, not OpenAI) |
| `LITELLM_MASTER_KEY` | litellm | Generated. Prefix `sk-`. Admin UI password. |
| `LITELLM_SALT_KEY` | litellm | Generated. Encrypts stored provider keys. |
| `STORE_MODEL_IN_DB` | litellm | `True` |
| `DATABASE_URL` | litellm | `${{Postgres.DATABASE_URL}}` |

### Optional — BYO provider keys

None of these are required to boot. The UI comes up with an empty model list until you add a key in LiteLLM or set one of these.

| Variable | Required | Why |
|---|---|---|
| `OPENAI_API_KEY` | No | OpenAI / compatible. Set on **litellm**, then add the model in LiteLLM UI (`os.environ/OPENAI_API_KEY`). |
| `ANTHROPIC_API_KEY` | No | Anthropic. Same pattern. |
| `GEMINI_API_KEY` | No | Google Gemini. Same pattern. |
| `AZURE_API_KEY` / `AZURE_API_BASE` | No | Azure OpenAI. Same pattern. |
| `OPEN_WEBUI_DATABASE_URL` | No | Only if you move Open WebUI off SQLite. Do **not** reuse LiteLLM's `DATABASE_URL` — different schemas. |

### Volume

| Mount | Service | Why |
|---|---|---|
| `/app/backend/data` | open-webui | Users, chats, uploads, SQLite. Without it, the admin account resets every deploy. |

LiteLLM state is in Postgres. It does not need a volume.

## Why Deploy Open WebUI + LiteLLM on Railway?

ChatGPT Team is $30/user/month and you still cannot point it at last week's bench winner. This stack is a UI + a proxy. You pay Railway plus the tokens you burn. Ten seats of ChatGPT Team is the ~$3,600/year line on the c17 stack page. Trial credits cover the first deploy for most people. c17 never charges you — Railway usage is the referral / template kickback.

See `TEMPLATE.md` for the marketplace / composer shape.

## Run locally

```bash
cp .env.example .env
# set WEBUI_SECRET_KEY, LITELLM_SALT_KEY (openssl rand -hex 32)
# set LITELLM_MASTER_KEY=sk-$(openssl rand -hex 32)
# set POSTGRES_PASSWORD
# optional: OPENAI_API_KEY / ANTHROPIC_API_KEY
docker compose up
```

- Open WebUI: http://localhost:3000
- LiteLLM admin: http://localhost:4000/ui (user `admin`, password = `LITELLM_MASTER_KEY`)

The UI works without provider keys. Chats fail until you add a model in LiteLLM.

## License

Open WebUI and LiteLLM keep their upstream licenses. This repo is config and docs only.
