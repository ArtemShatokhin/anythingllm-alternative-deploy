# 02. Docker Compose: the self-hosted stack

Kortix is the open-source AI Operating System, and its supported self-host path is one Docker Compose stack. The Kortix docs draw a line between the commands they document and the operator choices around them, and the Compose map below shows the shape of the stack ([self-hosting](https://kortix.com/docs/host); [self-hosting architecture](https://kortix.com/docs/host/architecture)).

## The documented path

Kortix does not ask you to hand-write a Compose file. `kortix self-host init` renders a `docker-compose.yml`, an `.env` file, and (when you set a domain) a `Caddyfile` and `updater.sh` into `~/.config/kortix/self-host/<instance>/`. `kortix self-host start` then runs `docker compose up` against that generated file. The same artifact runs on a laptop, a VPS, or any cloud VM; a domain is only the `KORTIX_DOMAIN` environment variable and not a different setup.

```bash
curl -fsSL https://kortix.com/install | bash   # install the CLI
kortix self-host init --domain kortix.example.com
kortix self-host start
kortix self-host status      # also: kortix self-host logs, kortix self-host doctor
```

Images come from the `docker.io/kortix/*` registry, need no credentials, and are pinned by digest in the generated file. Every instance updates itself automatically on a channel, and you can pin an exact version or turn the updater off:

```bash
kortix self-host update --tag 0.9.84
kortix self-host update --auto-update off
```

## What the stack contains

The generated file stands up the frontend, the API (`kortix-api`), and the LLM gateway as the three application images, plus the Supabase Docker distribution (Kong, GoTrue auth, PostgREST, Storage, Realtime, Studio, imgproxy, meta, functions, and the Supavisor connection pooler), vendored from upstream and pinned by digest. A `kortix-updater` container watches for a new image once a day. Caddy is rendered only when a domain is set, and it terminates TLS on ports 80 and 443.

Agent sessions run outside the box. `kortix-api` reaches the sandbox provider over egress, and sandbox compute never runs on the self-host box, so the Compose stack is the control plane and not the execution plane.

## A minimal Compose map

The file below is a readable map of the stack, not a file you run. Read it to understand the services; run the generated file for a supported install. Every value that is a placeholder belongs to operator choice and is labeled as such.

```yaml
# ILLUSTRATIVE MAP, NOT THE OFFICIAL FILE.
# `kortix self-host init` renders the authoritative docker-compose.yml and
# pins each image by digest. Placeholder image refs are set by the generated
# .env; service names, Caddy's role, and the data paths follow the docs.
services:
  caddy:                   # documented: reverse proxy + ACME TLS, domain mode only
    image: caddy:2
    restart: unless-stopped
    ports: ["80:80", "443:443"]   # documented: the bundled proxy issues TLS here
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro

  kortix-api:              # documented application image
    image: ${KORTIX_API_IMAGE:?set by the generated .env}
    restart: unless-stopped
    env_file: [.env]
    depends_on: [db]

  llm-gateway:             # documented application image
    image: ${KORTIX_LLM_GATEWAY_IMAGE:?set by the generated .env}
    restart: unless-stopped
    env_file: [.env]

  frontend:                # documented application image
    image: ${KORTIX_FRONTEND_IMAGE:?set by the generated .env}
    restart: unless-stopped
    env_file: [.env]

  db:                      # the vendored Supabase distribution runs over Postgres
    image: postgres:16
    restart: unless-stopped
    volumes:
      - ./volumes/db/data:/var/lib/postgresql/data
```

## Documented parts and operator choices

Documented by the Kortix self-hosting pages: that the stack is one generic Compose system; the three application services plus the Supabase distribution and the updater; Caddy only in domain mode; ports 80 and 443; images from `docker.io/kortix/*` pinned by digest; and the two data directories.

Operator choices, which is why the map above labels its values: the domain name; the host hardware beyond the memory guidance in [01-prerequisites.md](01-prerequisites.md); the update channel or a pinned version; and the backup cadence. The placeholder image references in the map stand in for whatever the generated `.env` resolves, so copy the real values from that file rather than trusting the map.

## Backups

Kortix has no separate backup system. Each instance stores its data as two directories under `~/.config/kortix/self-host/<instance>/`: `volumes/db/data` for the Postgres database and `volumes/storage` for file storage. The instance's `.env` file holds every secret and signing key it uses. Back up all three before you run a destructive command ([self-hosting](https://kortix.com/docs/host)).

## If you already run AnythingLLM in Compose

AnythingLLM ships as a single Docker service. The docs pull `mintplexlabs/anythingllm:latest` and run it with a host folder mounted at `/app/server/storage`, then open the app on port 3001 ([local Docker installation](https://docs.anythingllm.com/installation-docker/local-docker)). Building from source uses a Compose file under the repo's `docker/` directory with a `.env.example` you copy first.

The Kortix stack is larger because it is a company control plane rather than one chat app: it adds the API, the LLM gateway, authentication, and a generated Compose file that is maintained for you. Both self-host, and self-host on Kortix is free.

Related reading: [Kortix vs AnythingLLM](https://anythingllm-alternative.com/kortix-vs-anythingllm).
