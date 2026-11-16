# n8n-toolbox

Gives an ADI agent a door onto your n8n instance's workflows, over n8n's official
instance-level MCP server. No hive service of its own — this bundle is two elements:

| kind | name | what it is |
| --- | --- | --- |
| mcp | `n8n` | the MCP server definition: n8n's streamable-HTTP endpoint, bearer auth |
| agent | `n8n-operator` | `harness:claude-sdk`, attached to the `n8n` server above |

**Guides:** [n8n → ADI → n8n: process data in ADI and send the result back](https://docs.withadi.dev/n8n-to-adi/) — this bundle covers ADI driving n8n; that guide covers the other direction, n8n handing data to an ADI agent.

## Install it

```sh
adi-mono marketplace install <source>/n8n-toolbox
```

Lands both elements globally under their published ids (`n8n`, `n8n-operator`) — installing
twice, or into a project, mints past a collision the ordinary way (see `[[Marketplace]]`). The
`mcp` element always lands verbatim-or-refuses like a tool does: a second global install collides
on the exact id `n8n` unless you give it a project.

## Secrets

- `N8N_MCP_ACCESS_TOKEN` — the bearer token n8n's MCP server checks on every request. Get it from
  n8n: **Settings → Instance-level MCP → Connect → API key tab**. If you installed the `n8n`
  package in this same marketplace, `bin/provision.sh` already minted and set this for you under
  that project — in that case there's nothing to do here but attach it (next section).

  **Set it to the bare token**, exactly as n8n showed it to you — never with `Bearer ` written
  into it:

  ```sh
  adi-mono secrets set N8N_MCP_ACCESS_TOKEN "<the token n8n showed you>" --project <id>
  ```

  The bundled `mcp/n8n.toml` names this secret with a `prefix`:

  ```toml
  Authorization = { secret = "N8N_MCP_ACCESS_TOKEN", prefix = "Bearer " }
  ```

  so `Bearer ` is glued onto the secret's resolved value at launch — the header n8n's MCP
  middleware actually checks for is built for you, and the stored secret stays the plain token
  n8n handed out (also what `process:codex`'s own bearer-token knob needs, raw). The CLI
  equivalent of the line above is `adi-mono mcp save n8n --header
  "Authorization=bearer:N8N_MCP_ACCESS_TOKEN"` — see "Pointing it at a different n8n" below.

- The agent needs the secret **attached**, not just the server declaring it — registering a
  secret name on an `mcp` element never hands it to anybody (same rule a tool's `bin_tools`
  follows). `n8n-operator` already lists it in `[[secrets]]`, so once the secret itself exists
  under whichever scope you installed into, it resolves at launch with nothing further to do.
  `adi-mono mcp show n8n` says whether the server itself still reports it missing.

## Pointing it at a different n8n

The bundled `mcp/n8n.toml` defaults to `http://n8n.adi/mcp-server/http` — this marketplace's own
`n8n` package, on this machine. n8n Cloud, a different self-hosted instance, or a different host
on this same machine all need the URL changed after install — the `mcp` kind supports exactly
this, no file edit required:

```sh
adi-mono mcp save n8n --url https://your-instance.app.n8n.cloud/mcp-server/http \
  --header "Authorization=bearer:N8N_MCP_ACCESS_TOKEN"
```

(`adi-mono mcp show n8n` to confirm what's currently set.) Works with the `n8n` package here, or
any n8n instance at or past the version that added instance-level MCP (n8n ≥2.20.0) — the server
side is n8n's own, not something this bundle provides.

## The agent

`n8n-operator` is deliberately small: find a workflow, run it, report what actually came back,
never invent one. It runs on `harness:claude-sdk`, which needs its own LLM backend and Claude
credentials configured separately — `backend = "harness:claude-sdk"` with no `[[backends]]` row
is legal (the file still parses and lands), but a run on it fails immediately with "Not logged in"
until it answers on a real backend. If this machine already runs other `harness:claude-sdk`
agents, point `n8n-operator` at the same one:

```sh
adi-mono agents save n8n-operator --llm <an-existing-harness-claude-sdk-backend-id>
```

Otherwise, set one up first — see `[[Agents]]`'s own backends section. This isn't something the
bundle can do for you: a named LLM backend is never assumed to exist on the machine an install
lands on, and this one carries no `llm/*.toml` of its own to create it (see `[[Building a
marketplace bundle]]`, "Published names and installed ids").

Once it can actually answer, run it with a task:

```sh
adi-mono agents run n8n-operator --message "run my daily digest workflow"
```
