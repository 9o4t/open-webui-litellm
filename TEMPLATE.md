# Railway template config (Open WebUI + LiteLLM)

Publish from Dave's Railway account using official images plus Railway Postgres. Then put the published template code in `9o4t/c17` `internal/railway/templates.go` as `"open-webui-litellm": "<code>"`. Do not invent a code.

Public source / docs: GitHub [`9o4t/open-webui-litellm`](https://github.com/9o4t/open-webui-litellm). The running services are Docker images, not a build of this repo.

Provider keys are **optional BYO**. Do not mark `OPENAI_API_KEY` required.

## Services

| Field | open-webui | litellm | Postgres |
|---|---|---|---|
| Name | `open-webui` | `litellm` | `Postgres` |
| Source | `ghcr.io/open-webui/open-webui` | `docker.litellm.ai/berriai/litellm:main-stable` | Railway plugin |
| Public | Yes | Yes (agents use this) | No |
| Healthcheck | `GET /health` | `GET /health/liveliness` | plugin default |
| Healthcheck timeout | 60s | 60s | — |
| Restart | on failure | on failure | plugin default |
| Volume | `/app/backend/data` | none | plugin volume |
| Start | image default (`PORT`) | `litellm --port $PORT --host 0.0.0.0` | plugin default |

If the LiteLLM image ignores `PORT`, set the service target port to `4000` and point Open WebUI at `${{litellm.RAILWAY_PRIVATE_DOMAIN}}:4000`.

## Variables Railway provides

| Variable | Service | Value |
|---|---|---|
| `PORT` | both apps | Injected |
| `WEBUI_SECRET_KEY` | open-webui | Generated |
| `WEBUI_AUTH` | open-webui | `true` |
| `ENABLE_OLLAMA_API` | open-webui | `false` |
| `OPENAI_API_BASE_URL` | open-webui | `http://${{litellm.RAILWAY_PRIVATE_DOMAIN}}:${{litellm.PORT}}/v1` |
| `OPENAI_API_KEY` | open-webui | `${{litellm.LITELLM_MASTER_KEY}}` |
| `LITELLM_MASTER_KEY` | litellm | Generated, prefix `sk-` |
| `LITELLM_SALT_KEY` | litellm | Generated |
| `STORE_MODEL_IN_DB` | litellm | `True` |
| `DATABASE_URL` | litellm | `${{Postgres.DATABASE_URL}}` |

## Variables the operator may provide

None are required to boot.

| Variable | Required | Why |
|---|---|---|
| `OPENAI_API_KEY` | No | Set on **litellm**. Empty = no OpenAI models until you add a key in the LiteLLM UI. |
| `ANTHROPIC_API_KEY` | No | Same. |
| `GEMINI_API_KEY` | No | Same. |

Do not share LiteLLM's `DATABASE_URL` with Open WebUI.

## Composer / dashboard publish checklist

1. Sign in as Dave on [railway.com](https://railway.com).
2. New project. Add image `ghcr.io/open-webui/open-webui`. Add image `docker.litellm.ai/berriai/litellm:main-stable`. Add Railway Postgres.
3. Volume `/app/backend/data` on `open-webui`.
4. Set the variables in the table. Generate the three secrets. Wire LiteLLM `DATABASE_URL`. Public domains on both app services.
5. Healthchecks: Open WebUI `GET /health`, LiteLLM `GET /health/liveliness`. Timeout 60s.
6. Confirm both UIs boot **without** provider keys. Open WebUI first user = admin. LiteLLM `/ui` accepts `admin` + `LITELLM_MASTER_KEY`.
7. Generate a template: dashboard **Generate Template**, or `railway templates create --project <id> --json`.
8. Mark provider keys optional. Description (≤75 chars): `Open WebUI + LiteLLM. BYO LLM keys. Optional Postgres.`
9. Publish, category **AI/ML** (or **Starters**):

    ```
    railway templates publish <template-id> --category AI/ML \
      --description "Open WebUI + LiteLLM. BYO LLM keys. Optional Postgres." \
      --readme-file README.md --json
    ```

10. Copy the published **template code**.
11. In `9o4t/c17`, set `internal/railway/templates.go`:

    `"open-webui-litellm": "<code>"`

    That entry switches `/stacks/open-webui-litellm`, process **AI chat**, and `/replace/chatgpt` from **Start on Railway** to the official Deploy button with `referralCode=4xHDyE`.
12. Run `go test ./...` in c17. Merge only when the code is the real published one.
