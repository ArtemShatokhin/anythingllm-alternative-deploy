# 04. Operations and FAQ

Three things decide whether a self-hosted Kortix instance is safe to run day to day: how secrets are handled, how work reaches the default branch, and how sessions scale. Kortix is the open-source AI Operating System, and each session runs on its own isolated machine.

## Secrets and credentials

A secret is a per-project credential, such as an API key or a connection string, that a session needs but that must not live in the repository. Kortix stores secrets on the project rather than on the account, and it encrypts every value with AES-256-GCM using a key derived per project ([secrets](https://kortix.com/docs/project/secrets)).

For connectors the rule is stricter: connector credentials are brokered server-side and never enter the machine. The agent gets a token and never the key.

A secret carries two independent settings. Exposure answers whether agent code can read the real value, and it takes one of `environment`, `egress-enforced`, or `none`. Usage says who spends it: agent code, the LLM gateway, a connector, or Git. Environment is the default, and it is the only exposure that can serve a credential the sandbox has to compute with. Computed secrets such as AWS SigV4 signing keys, HMAC webhook-signing secrets, JWT client assertions, and SSH private keys must stay on environment, because the value never travels on the wire and no network boundary helps. Non-HTTPS protocols such as a Postgres connection string are in the same position.

Egress-enforced exposure is experimental. The sandbox receives a handle worth nothing on its own, and Kortix substitutes the real value outside the sandbox, only on the exact HTTPS hosts you list. A host matches exactly, so `api.example.com` does not cover `uploads.api.example.com`, and wildcards are rejected. An echoed value comes back as `[REDACTED]`. The mechanism is identical on every sandbox provider.

Who can use a secret is a separate answer of everyone in the project (the default), only you, or specific people and groups. Shared sessions, triggers, schedules, and webhooks act for nobody, so they receive only values shared with everyone.

## The change-request workflow

A session runs in a sandbox on its own branch, and the branch is named after the session ID. Every change, including code, agents, skills, and the manifest, reaches the default branch through a change request and nothing else ([change requests](https://kortix.com/docs/work/change-requests)).

An agent opens a change request and stops. It commits on `$KORTIX_BRANCH_NAME`, pushes with `git push origin HEAD`, opens the request with `kortix cr open --title "..." --description "..."`, tells you the number, and does not merge it. Merge is default-deny for agents, and a session can never merge a change request it opened itself, whatever it has been granted. Opening and merging are separate capabilities: `project.gitops.push` opens and `project.gitops.merge` merges.

Review from the CLI:

```bash
kortix cr ls              # list what sessions propose
kortix cr diff <cr>       # show the unified patch
kortix cr merge <cr>      # land it on the default branch
```

A merge reads the manifest from the head branch and validates it, so a branch with an invalid `kortix.yaml` stops with a `MANIFEST_INVALID` error rather than landing. A closed change request can be reopened; a merged one is terminal, and a new request is opened against the merged state instead.

## Scaling sessions

Thousands of agents run in parallel on one config, each on its own cloud computer. One session maps to one isolated sandbox with its own machine and branch, so sessions do not collide, and each new session starts from the default branch.

Session compute stays off the self-host box and runs on a sandbox provider, whose default is Daytona, with Platinum and E2B also supported. On the host side, the API's memory limit is the tunable that matters: the default is 640 MiB per API container, raised with `kortix self-host env set KORTIX_API_MEMORY_LIMIT=1024m` on a larger box ([self-hosting](https://kortix.com/docs/host)).

Work can start three ways. A person starts a session on demand; a cron trigger or a signed webhook starts a session with nobody present; and a human-assisted run pauses at an approval gate. A trigger has exactly two types, `cron` and `webhook`, and it starts sessions with a `session_mode` of `fresh`, `reuse`, `pinned`, or `keyed` (the default is `fresh`).

## FAQ

**Do connector credentials ever reach the sandbox?** No. Kortix brokers every connector call server-side, so the credential never enters the machine, and the agent gets a token instead of the key ([connectors](https://kortix.com/docs/connect/connectors)).

**Can I run Kortix beside AnythingLLM while I compare?** Yes. Self-host is free, so a Kortix project can run next to an existing AnythingLLM setup, and you can move one workflow at a time. Your documents stay where they are while you copy the ones you need into the repo.

**What happens to a secret the agent has to compute with?** It stays on the `environment` exposure. A signing key or a non-HTTPS connection string must be a plain environment value, because the sandbox holds the value to do the math and no network boundary can substitute for it.

**How do I reverse a bad change?** A change request is a diff you read before it merges, so a bad change is caught at review. A closed request can be reopened, and a merged one is terminal, so the fix is a new change request against the merged state.

Related reading: [AnythingLLM alternatives compared](https://anythingllm-alternative.com/best-anythingllm-alternatives).
