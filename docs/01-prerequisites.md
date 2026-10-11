# 01. Prerequisites

A document-Q&A agent needs three things in place before it can run on Kortix: a host, a model key, and a document-source connection. Kortix is the open-source AI Operating System, and it self-hosts as one Docker Compose stack on a Linux box ([self-hosting](https://kortix.com/docs/host)).

## The host

A supported self-hosted Kortix instance needs a Linux machine with a persistent domain. The domain gives Caddy a stable name for ACME TLS and gives agent sandboxes a stable URL to call back to; without a domain or a tunnel, sessions cannot run ([self-hosting architecture](https://kortix.com/docs/host/architecture)).

The self-hosting page documents a one-shot bootstrap on a bare Linux box that installs Docker, installs the `kortix` CLI, and starts the stack in a single command ([self-hosting](https://kortix.com/docs/host)). The script runs on Linux only. On another operating system, install the CLI directly and use the manual path:

```bash
curl -fsSL https://kortix.com/install | bash      # install the CLI
# create A/AAAA records for <domain> and api.<domain>, and open ports 80 and 443
kortix self-host init --domain kortix.example.com
kortix self-host start
```

Create the DNS records before `init`, because Caddy issues a TLS certificate over ports 80 and 443. To evaluate without a domain, use `kortix self-host init --tunnel cloudflare`; the tunnel URL changes on every restart, so it is not for production.

Memory is the one hardware number the docs pin down. Each API container has a 640 MiB memory limit by default, which is the right default on an 8 GiB host. On a 16 GiB host, raise it once API traffic reaches that limit:

```bash
kortix self-host env set KORTIX_API_MEMORY_LIMIT=1024m
docker stats --no-stream
```

Agent sessions do not run on the self-host box. They run on a separate sandbox provider, and the default is Daytona, with Platinum and E2B also supported. After the stack starts, `kortix self-host configure` prompts for the sandbox provider key ([self-hosting](https://kortix.com/docs/host)).

If you already run AnythingLLM, its host requirements differ. AnythingLLM publishes a Docker image you pull with `docker pull mintplexlabs/anythingllm:latest` and run as a single service on port 3001, with a host folder mounted at `/app/server/storage` so data survives a restart ([local Docker installation](https://docs.anythingllm.com/installation-docker/local-docker)). The published image runs as UID/GID 1000, and the default vector database needs a CPU with the AVX2 instruction set, which older or virtualized hardware often hides.

## Model keys

Kortix runs each session on a model, and it accepts any model provider with your own keys, or the ChatGPT plan you already pay for, or a sign-in with an OpenCode Console account for OpenCode Zen and Go ([models](https://kortix.com/docs/project/models)).

A bring-your-own-key model is named `provider/model`, for example `anthropic/claude-opus-4-8`. Each provider reads its own key from an environment variable such as `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `OPENROUTER_API_KEY`. You add the key on the project's Model settings page or set it directly as a secret. A ChatGPT plan is connected once through OAuth, and Kortix stores the model ids you can bring rather than the vendor's account.

A Kortix-managed model uses a bare id such as `deepseek-v4.1-flash` (the platform default), and Kortix supplies the credentials; a cloud account pays with Kortix credits. Kortix does not offer OpenAI or Anthropic models as managed models, so those arrive through your own key or a ChatGPT plan. Provider keys are stored as secrets on the project, never in the repository.

AnythingLLM takes the same kind of provider credentials: it connects local models through Ollama or LM Studio and cloud models through your own accounts, with an AnythingLLM native embedder as the default ([AnythingLLM features](https://docs.anythingllm.com)). Both products let you point at a model you already have a key for.

## Connectors the agent needs

A document-Q&A agent needs one connector to reach a document source and, optionally, one channel to be reached from. Kortix declares most connectors in `kortix.yaml`, and it browses a managed catalog of 3,000+ apps, plus MCP, OpenAPI, Postman, GraphQL, and raw HTTP ([connectors](https://kortix.com/docs/connect/connectors)). Google Drive and Notion are both in the catalog, so either can serve as the document source for this template.

Connector credentials are brokered server-side and never enter the machine, so the agent gets a token and never the key. A connector can hold several accounts, and each account carries a "who can use it" answer of only you, everyone in the project, or specific people and groups ([connectors](https://kortix.com/docs/connect/connectors)).

The four prerequisites to gather before the first session are a Linux host with a domain, a model key or plan, a document-source connection, and the sandbox provider key. Once they exist, [02-docker-compose.md](02-docker-compose.md) stands the stack up.

Related reading: [self-hosting an AnythingLLM alternative](https://anythingllm-alternative.com/self-hosting-anythingllm-alternative).
