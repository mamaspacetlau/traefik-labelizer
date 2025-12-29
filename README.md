# Traefik Labelizer

Traefik Labelizer helps you take an existing `docker-compose.yml` and automatically add production‑ready [Traefik](https://doc.traefik.io/traefik/) labels for routing, TLS, and authentication middlewares.

You can run it entirely locally (instant, deterministic) or via an AI engine (Gemini, OpenAI, OpenRouter) for more complex, heuristic transformations.

## Demo video

You can watch the demo video on GitHub by clicking this link:

[Demo video (MP4)](demo/demo.mp4)

---

## Running the app

### Local development

```bash
npm install
npm run dev
```

The Vite dev server will start (by default on port 5173). Open the URL it prints in your terminal.

### Docker

Build the image:

```bash
docker build -t traefik-labelizer .
```

Run it directly:

```bash
docker run --rm -p 9412:9412 traefik-labelizer
```

The container runs nginx configured to listen on port `9412`. The app is then available at:

```text
http://localhost:9412
```

### Docker Compose

This repository includes a convenience `compose.yaml`:

```yaml
services:
  traefik-labelizer:
    image: motionninja/trafik-labelizer:latest
    container_name: traefik-labelizer
    restart: unless-stopped
    ports:
      - 9412:9412
    environment:
      API_KEY: ${API_KEY}
      API_PROVIDER: ${API_PROVIDER}
      DOMAIN: ${DOMAIN:-example.com}
      MIDDLEWARE: ${MIDDLEWARE}
      COMMENT_OUT_PORTS: ${COMMENT_OUT_PORTS:-true}
      SYNC_SCROLL: ${SYNC_SCROLL:-true}
      NETWORK: ${NETWORK:-frontend}
      WORD_WRAP: ${WORD_WRAP:-false}
      AI_MODEL: ${AI_MODEL:-}
```

Create a `.env` file alongside `compose.yaml` and set the environment variables described below, then:

```bash
docker compose up -d
```

---

## Configuration overview

The app reads configuration in two layers:

1. Container environment variables
2. A generated `config.js` that is injected into the browser as `window.__TRAEFIK_LABELIZER_CONFIG__`

The flow is:

```text
docker-compose / Docker env → docker-entrypoint.sh → config.js → App UI
```

### Core environment variables

These are the main variables you will configure.

#### Domain and networking

- `DOMAIN`  
  Base domain used to generate host rules and example templates.  
  Default: `example.com`

- `NETWORK`  
  Name of the Docker network Traefik uses (`providers.docker.network`). Also used when ensuring a `networks:` block is present in the transformed YAML.  
  Default: `frontend`

#### Traefik behavior

- `COMMENT_OUT_PORTS`  
  Controls whether Traefik Labelizer comments out service `ports:` when labels are added.  
  Values: `"true"` or `"false"`  
  Default: `"true"`

- `MIDDLEWARE`  
  Enables the default authentication middleware(s) in the UI.  
  Values (comma/space insensitive): `tinyauth`, `authentik`, `voidauth`, `none`  
  Example: `MIDDLEWARE=tinyauth` will start with TinyAuth toggled on.

#### Editor behavior

- `SYNC_SCROLL`  
  When `true`, vertical scrolling between the source and generated YAML editors is synchronized.  
  Values: `"true"` or `"false"`  
  Default: `"true"`

- `WORD_WRAP`  
  Controls the initial state of the Word Wrap toggle for both YAML editors.  
  Values: `"true"` or `"false"`  
  Default: `"false"`

#### AI engine

- `API_KEY`  
  The API key used for the AI provider. If this is empty or missing, the AI Engine is disabled and only Local (live) mode is available.

- `API_PROVIDER`  
  Selects which AI provider to use.  
  Values: `gemini`, `openai`, `openrouter`  
  Default: `gemini`

- `AI_MODEL`  
  Shared default model identifier. This is provider‑specific and should be set to a value that matches your chosen provider. Examples:

  - Gemini: `gemini-3-flash-preview`
  - OpenAI: `gpt-4.1-mini` or similar
  - OpenRouter: `deepseek/deepseek-v3.2`, `bytedance-seed/seedream-4.5`, etc.

  Provider‑specific overrides are also supported via:

  - `GEMINI_MODEL`
  - `OPENAI_MODEL`
  - `OPENROUTER_MODEL`

  If a provider‑specific variable is set, it takes precedence over `AI_MODEL`. Otherwise, the runtime config uses `AI_MODEL` for that provider.

- `OPENAI_BASE_URL`  
  Optional override for the OpenAI API base URL.  
  Default (if empty): `https://api.openai.com/v1`

- `OPENROUTER_BASE_URL`  
  Optional override for the OpenRouter API base URL.  
  Default (if empty): `https://openrouter.ai/api/v1`

---

## How the configuration is wired

### docker-entrypoint.sh → config.js

At runtime, `docker-entrypoint.sh` generates `/usr/share/nginx/html/config.js`:

```sh
window.__TRAEFIK_LABELIZER_CONFIG__ = {
  domain: "${DOMAIN:-example.com}",
  middleware: "${MIDDLEWARE:-}",
  commentOutPorts: "${COMMENT_OUT_PORTS:-true}",
  syncScroll: "${SYNC_SCROLL:-true}",
  wordWrap: "${WORD_WRAP:-false}",
  network: "${NETWORK:-frontend}",
  apiKey: "${API_KEY:-}",
  apiProvider: "${API_PROVIDER:-gemini}",
  openaiBaseUrl: "${OPENAI_BASE_URL:-}",
  openrouterBaseUrl: "${OPENROUTER_BASE_URL:-}",
  geminiModel: "${GEMINI_MODEL:-${AI_MODEL:-}}",
  openaiModel: "${OPENAI_MODEL:-${AI_MODEL:-}}",
  openrouterModel: "${OPENROUTER_MODEL:-${AI_MODEL:-}}"
};
```

The React app reads from `window.__TRAEFIK_LABELIZER_CONFIG__` at startup and derives the initial UI state from it.

### App runtime config

In the main app component, these runtime config values are mapped into a strongly‑typed `TraefikConfig`:

- `commentOutPorts` and `syncScroll` are treated as `"true"`/`"false"` strings and converted to booleans.
- `wordWrap` is derived from the `"true"`/`"false"` `WORD_WRAP` value and controls the initial Word Wrap toggle state.
- `networkName` is used when generating or patching the `networks:` block in transformed YAML.
- AI settings determine whether the AI Engine option is available and which model/provider are used.

---

## UI overview

### Modes: Local vs AI Engine

In the header, you can choose between:

- **Local (Live)**  
  Uses a deterministic local transformer (`services/labelTransformer.ts`) to add Traefik labels based on your configuration. The output updates as you type.

- **AI Engine**  
  Uses the selected AI provider to generate labels and comments. Requires `API_KEY` and `API_PROVIDER`. AI runs on demand via the "Generate with AI" button.

### Configuration sidebar

On the left, the **Configuration** card controls how labels are generated:

- **Main Domain**  
  The base domain (from `DOMAIN`). Used for router `Host(...)` rules and templates.

- **Subdomain Override**  
  Allows you to override the subdomain pattern globally or per service port:

  - When multiple `service:port` combinations are detected, you can choose a specific one from a dropdown and override its subdomain individually.
  - The default entry controls the global pattern.

- **Label Generation per Port**  
  When multiple ports are present for a service, you can toggle label generation on or off per `service:containerPort`. If you disable labels for a given port, the transformer will:

  - Keep the port active in the YAML.
  - Avoid generating Traefik labels tied to that port.

- **Middlewares**  
  Toggles for:

  - TinyAuth
  - Authentik
  - VoidAuth

  These control whether corresponding middleware labels are added to generated routers.

- **Comment Out Ports**  
  Matches `COMMENT_OUT_PORTS`. When enabled, the transformer comments out service `ports:` where Traefik routers are created, while respecting ports you explicitly disabled label generation for.

- **Sync Scroll**  
  Matches `SYNC_SCROLL`. Controls whether vertical scroll is synchronized between the source and output editors.

- **Word Wrap**  
  Matches `WORD_WRAP`. When enabled, both editors wrap long lines instead of requiring horizontal scrolling.

- **Network**  
  Matches `NETWORK`. Used by the transformer to ensure the `networks:` section includes the specified network as `external: true`.

### Editors

- **Source docker-compose.yml**  
  Paste your starting compose file here. As you type or paste, the app:

  - Detects services and ports.
  - Populates the Label Generation section with `service:port` keys.
  - In Local mode, continuously regenerates the labeled YAML on the right.

- **Traefik Labelized YAML**  
  Shows the transformed YAML. You can:

  - Copy it via **Copy YAML**. After copying, you will see a Network Reminder popup telling you to create the configured Docker network (for example, `docker network create frontend`).  
  - Inspect the added labels and comments for each service and port.

### Reference panels

Below the editors, you will find optional reference panels:

- **Base Traefik Infrastructure YAML**  
  A ready‑to‑copy baseline Traefik stack using your configured domain (`Domain` field).

- **Selected Authentication Service Definitions**  
  Conditionally shows example service definitions for TinyAuth, Authentik, and VoidAuth, based on which middlewares are enabled.

---

## Networking and port behavior

- The runtime image uses `nginx:stable-alpine` and serves the static app from `/usr/share/nginx/html`.
- The Dockerfile modifies nginx’s default configuration to listen on port `9412` instead of `80`.
- `EXPOSE 9412` marks this port in the image.
- The provided `compose.yaml` maps host `9412` to container `9412`:

  ```yaml
  ports:
    - 9412:9412
  ```

If you use a different port mapping, ensure the host port correctly maps to container port `9412`.

---

## AI providers and models

The AI integration supports three providers:

- Gemini (via `@google/genai`)
- OpenAI compatible APIs
- OpenRouter

The provider is chosen via `API_PROVIDER`. The model used for each provider follows this priority:

1. Provider‑specific env (`GEMINI_MODEL`, `OPENAI_MODEL`, `OPENROUTER_MODEL`)
2. Shared `AI_MODEL` (if set)
3. Hardcoded provider defaults in `services/aiService.ts`:

   - Gemini default: `gemini-3-flash-preview`
   - OpenAI default: `gpt-4.1-mini`
   - OpenRouter default: `deepseek/deepseek-v3.2` (mapped to `deepseek/deepseek-v3.2`)

Ensure that the model name you provide matches the conventions of the selected provider.

---

## Tips for using Traefik Labelizer effectively

- Use **Local (Live)** mode to quickly iterate on label structure and port behavior.
- Switch to **AI Engine** when you need more complex, semantic refactors or detailed inline comments.
- Use the per‑port Label Generation toggles to:

  - Leave internal maintenance ports unexposed to Traefik.
  - Keep container `ports:` active only where you need them.

- Configure `WORD_WRAP=true` when working with very long labels or comments to avoid horizontal scrolling.
- Always create the Docker network indicated in the Network Reminder popup before deploying the generated YAML:

  ```bash
  docker network create <network_name>
  ```

This ensures your services can attach to the expected shared network when Traefik is running.
