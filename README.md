# Traefik Labelizer

Traefik Labelizer helps you take an existing `docker-compose.yml` and automatically add production‑ready [Traefik](https://doc.traefik.io/traefik/) labels for routing, TLS, and authentication middlewares.

You can run it entirely locally (instant, deterministic) or via an AI engine (Gemini, OpenAI, OpenRouter) for more complex, heuristic transformations.

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
