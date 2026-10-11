# 03. A document-Q&A agent preset

A document-Q&A agent in Kortix is two files that meet at one path: a governance block in `kortix.yaml` and a behavior file at `agents/<name>.md`. Kortix is the open-source AI Operating System, and a project is a git repo plus a manifest at the root ([manifest reference](https://kortix.com/docs/project/manifest); [agents](https://kortix.com/docs/project/agents)).

## The manifest: `kortix.yaml`

```yaml
# yaml-language-server: $schema=https://kortix.com/schema/kortix.v2.schema.json
kortix_version: 2
default_agent: doc-qa

project:
  name: doc-qa
  description: A document-Q&A agent that answers over a connected document source.

env:
  # BYOK model-provider keys are set as secrets on the project, never committed.
  # List the names only. These are the provider environment variables from the
  # Models docs (ANTHROPIC_API_KEY, OPENAI_API_KEY, and so on).
  optional: [ANTHROPIC_API_KEY, OPENAI_API_KEY]

agents:
  doc-qa:
    file: agents/doc-qa.md
    connectors: all
    secrets: all
    skills: all

connectors:
  # ILLUSTRATIVE document-source connector. The block shape follows the
  # connector definition in the manifest reference; replace slug, provider,
  # app, and the tool matches with the row you see under Connectors -> Add app
  # in your own project. The manifest reference shows this same shape with
  # provider: pipedream and app: gmail.
  - slug: docs-read
    name: Document source, read only
    provider: pipedream
    app: google-drive
    authorization_strategy: user
    policies:
      - match: "*"
        action: always_run
      - match: "delete_*"
        action: block
```

Each field above comes from a documented place. `kortix_version: 2` and `default_agent` are required, and `default_agent` must name an enabled agent. The `project` block is display metadata. `env.required` and `env.optional` hold variable names only, and `required` is advisory and never enforced at session start. The `agents` map is the governance half: it says which agent may launch and what it may reach, and it accepts only governance keys, never a prompt, model, or tool ([manifest reference](https://kortix.com/docs/project/manifest)).

Governance is deny-by-default. An agent with no `connectors`, `secrets`, `skills`, `apps`, or `kortix_permissions` key gets none of that access, so the `agents.doc-qa` block above grants reach deliberately. Set a field to `all` to grant full access or list specific names. `connectors.policies` is a list of `{match, action}` pairs, where `match` is a glob over tool names and `action` is `always_run`, `require_approval`, or `block`; the policy belongs to the connector and every connection under it uses it ([connectors](https://kortix.com/docs/connect/connectors)).

## The agent markdown: `agents/doc-qa.md`

```markdown
---
model: anthropic/claude-opus-4-8
temperature: 0.2
---

You are the document Q&A agent for this project.

Answer questions from the documents the project can read. Work in this order:

1. Find the passages that support an answer. Use the connected document source
   and the files in the repo.
2. Answer in your own words, one claim per sentence, and cite the source file
   and section for every factual claim.
3. When the documents do not support an answer, say what is missing instead of
   filling the gap. Never invent a policy, a number, or a date.
4. When a question needs a document that is not in the repo, open a change
   request that adds it, and say so in the same reply.

Keep answers short. Lead with the answer, then the source lines under it.
```

The body is the system prompt. The frontmatter holds only behavioral fields: `model`, `mode`, `temperature`, `top_p`, `steps`, and `permission`. The `model` value follows the `provider/model` shape from the Models docs and is an example here; replace it with any id your own key connects. Nothing in `kortix.yaml` may restate a behavioral field, because the validator rejects a v2 agent block that carries one and points at the agent's own file ([agents](https://kortix.com/docs/project/agents); [models](https://kortix.com/docs/project/models)).

Tool scoping uses the `permission` key in this frontmatter, not a `tools` key. The `.md` file never sets platform access, and the manifest never sets a prompt or tool; the two surfaces do not overlap.

## A skill the agent may load: `skills/cite-sources/SKILL.md`

A skill is a markdown file at `skills/<name>/SKILL.md`, and the harness loads it on demand rather than injecting it into every prompt. The `skills: all` grant above lets the agent load it.

```markdown
---
name: cite-sources
description: Answer document questions with a source line per claim.
---

When you answer from documents, attach one source line per factual claim.

- Name the file and the section the claim comes from.
- Quote at most one short phrase per claim, and paraphrase the rest.
- List the sources at the end of the answer, one per line, in the order you
  used them.
```

## Ship the preset

With the CLI installed, scaffold a project, replace the starter files with the two files above, and push:

```bash
curl -fsSL https://kortix.com/install | bash
kortix init my-doc-qa
cd my-doc-qa
# write kortix.yaml, agents/doc-qa.md, and skills/cite-sources/SKILL.md
kortix ship
```

`kortix ship` lints `kortix.yaml`, commits local changes, pushes the branch, and prompts for any missing secret or connection. New or changed agents and skills reach future sessions only after a change request merges them into the default branch, so review the first one with `kortix cr ls` and `kortix cr diff <cr>`, then land it with `kortix cr merge <cr>` ([CLI](https://kortix.com/docs/cli); [change requests](https://kortix.com/docs/work/change-requests)).

Related reading: [the open-source AnythingLLM alternative](https://anythingllm-alternative.com/open-source-anythingllm-alternative).
