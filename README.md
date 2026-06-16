# Hermes + SearXNG + Camoufox

A self-hostable web-research stack for the [Hermes Agent](https://github.com/NousResearch/hermes-agent),
built around one idea: **don't "browse the internet" as a single habit — split web work into
discovery, extraction, and interaction.**

1. **Search** with [SearXNG](https://github.com/searxng/searxng) — cheap discovery of candidate URLs.
2. **Extract** with [Firecrawl](https://github.com/firecrawl/firecrawl) — turn known URLs into clean markdown.
3. **Browse** with **Camoufox** (the stealth Firefox fork, served by
   [`jo-inc/camofox-browser`](https://github.com/jo-inc/camofox-browser)) — only when JavaScript,
   clicks, or auth state are genuinely required.

This keeps web work auditable, cheaper, and less exposed to prompt-injection hidden inside arbitrary
pages. The Hermes skill that encodes this policy lives in
[`skills/searxng-firecrawl-camoufox/SKILL.md`](skills/searxng-firecrawl-camoufox/SKILL.md).

## Components

| Service | Image | Role | Port |
|---|---|---|---|
| **hermes** | `nousresearch/hermes-agent` | Agent gateway + OpenAI-compatible API | 8642 |
| **hermes-dashboard** | `nousresearch/hermes-agent` | Web dashboard / triage UI | 9119 |
| **searxng** | `searxng/searxng` | Metasearch — the *search* layer | 8080 |
| **valkey** | `valkey/valkey` | Cache/queue for SearXNG (Redis fork) | 6379 |
| **camoufox** | `ghcr.io/jo-inc/camofox-browser` | Stealth browser automation — the *browse* layer | 9377 |
| **ollama** | `ollama/ollama` | Local LLM runtime | 11434 |

> **On the spelling:** the browser engine is **Camoufox** (with a "u"). The upstream Docker image
> (`jo-inc/camofox-browser`) and its environment variables (`CAMOFOX_*`) are spelled **without** the
> "u" — that is the maintainer's literal naming, so this repo keeps those identifiers verbatim while
> using "Camoufox" everywhere else.

> **Firecrawl is not bundled.** Hermes' `web_extract` defaults to Firecrawl, which this compose does
> not run. See [Wiring & gaps](#wiring--gaps).

## Two compose files

- **`docker-compose.yml`** — generic / portable. Publishes ports and reads a plain `.env`. Use this for a normal `docker compose up`.
- **`docker-compose.coolify.yml`** — for [Coolify](https://coolify.io). Uses Coolify's `SERVICE_*` magic variables and Traefik/Authentik labels instead of published ports.

## Quick start (generic)

Requirements: Docker + Compose v2, and enough RAM for your Ollama model (the default `gemma4:e4b`
wants roughly 16 GB).

```bash
# 1. Configure
cp .env.example .env
# Generate the two required secrets and put them in .env:
#   SEARXNG_SECRET        ->  openssl rand -hex 32
#   HERMES_API_SERVER_KEY ->  openssl rand -hex 32

# 2. Boot the stack
docker compose up -d

# 3. Pull the Ollama model (it is NOT downloaded automatically)
docker compose exec ollama ollama pull gemma4:e4b
```

Verify:

```bash
# SearXNG returns JSON (not 403) — confirms the json format is enabled
curl -s "http://localhost:8080/search?q=test&format=json" | head

# Camoufox health
docker compose exec camoufox curl -fsS http://127.0.0.1:9377/health

# Hermes dashboard
open http://localhost:9119
```

## Configuration

All settings live in `.env` (see `.env.example` for the full list with comments). Highlights:

| Variable | Purpose |
|---|---|
| `SEARXNG_SECRET` | **Required.** SearXNG secret key (`openssl rand -hex 32`). |
| `HERMES_API_SERVER_KEY` | **Required.** Auth key for the Hermes API server. |
| `OLLAMA_MODEL` | Ollama model tag (default `gemma4:e4b`). Pull it after boot. |
| `CAMOFOX_IMAGE_TAG` | Browser image tag — pin a real one from the [releases](https://github.com/jo-inc/camofox-browser/releases) (tags are arch-suffixed). |
| `SEARXNG_PORT` / `HERMES_API_PORT` | Host ports for the generic compose. |

### Hermes tool wiring

The generic compose wires Hermes to the in-network services:

```yaml
SEARXNG_URL=http://searxng:8080      # search backend
CAMOFOX_URL=http://camoufox:9377     # browser backend (env name is an upstream literal)
```

To make SearXNG the default search backend (Hermes defaults to Firecrawl), set it in your Hermes
`config.yaml` — an example is in [`hermes/config.example.yaml`](hermes/config.example.yaml):

```yaml
web:
  search_backend: "searxng"
  extract_backend: "firecrawl"   # see gaps below
```

See the [Hermes configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration)
for the authoritative reference.

### Wiring & gaps

This stack is explicit about what it does and does not include:

- **Firecrawl (extract) is not bundled.** Options: (a) [self-host Firecrawl](https://github.com/firecrawl/firecrawl/blob/main/SELF_HOST.md)
  (port 3002) and set `FIRECRAWL_API_URL=http://firecrawl:3002` with `USE_DB_AUTHENTICATION=false`;
  (b) use Firecrawl cloud via `FIRECRAWL_API_KEY`; or (c) rely on SearXNG-only search and skip `web_extract`.
- **Hermes ↔ Ollama LLM wiring is left to you.** The compose runs Ollama but does not assume your
  Hermes version's provider variables. Point Hermes at `http://ollama:11434` per the Hermes docs
  (commented placeholders are in `docker-compose.yml`).
- **The Ollama model is not auto-pulled** — run the `ollama pull` step from the quick start.
- **Pin the Camoufox image tag** — `latest` may not resolve on GHCR; tags are arch-suffixed (e.g. `135.0.1-aarch64`).

## The skill

[`skills/searxng-firecrawl-camoufox/SKILL.md`](skills/searxng-firecrawl-camoufox/SKILL.md) is a Hermes
skill that teaches the agent the **search → extract → browse** escalation policy: SearXNG first,
Firecrawl for known URLs, and Camoufox only when a real browser is unavoidable. Drop it into your
Hermes skills directory.

## Deploying on Coolify

Use `docker-compose.coolify.yml`. It relies on Coolify features instead of published ports:

- `SERVICE_URL_SEARXNG_8080` and `SERVICE_HEX_64_SEARXNG` are Coolify "magic" variables that Coolify
  generates for you (a public URL and a 64-character hex secret).
- The Traefik/Authentik labels put services behind Coolify's reverse proxy and SSO.
- Set `HERMES_API_SERVER_KEY` in the Coolify environment.

## Security

- **Change every secret.** Never commit your real `.env` (it is gitignored).
- SearXNG runs with `SEARXNG_PUBLIC_INSTANCE=false`. If you expose it publicly, enable the limiter
  (`server.limiter: true` in `searxng-settings.yml`) and keep auth in front of Hermes.
- Treat extracted web content as untrusted evidence, not as instructions.

## License

[MIT](LICENSE).

## Credits

[Hermes Agent](https://github.com/NousResearch/hermes-agent) ·
[SearXNG](https://github.com/searxng/searxng) ·
[camofox-browser](https://github.com/jo-inc/camofox-browser) (Camoufox) ·
[Firecrawl](https://github.com/firecrawl/firecrawl) ·
[Ollama](https://github.com/ollama/ollama) ·
[Valkey](https://github.com/valkey-io/valkey)
