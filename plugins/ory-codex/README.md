# Ory Agent Plugin: Codex

[Ory](https://ory.com) bundled into [Codex](https://github.com/openai/codex): skills that scaffold Ory authentication into your codebase, a local Ory stack you can spin up in one command, and (when pointed at an Ory project) authentication, authorization, and audit for every tool Codex runs.

You don't need an Ory account or any prior Ory experience to start.

## Prerequisites

- [Codex](https://github.com/openai/codex) installed and signed in
- Node.js **≥ 22**
- [Docker](https://docs.docker.com/get-docker/) (only needed for the local Ory stack)
- macOS or Linux. Windows works via WSL2.

## Install

No prior `npm install` required:

```bash
npx @ory/codex install     # registers the Ory marketplace + plugin with `codex`
npx @ory/codex uninstall   # removes both
```

Then confirm everything landed:

```bash
npx -y -p @ory/codex ory-codex status
```

`status` is the single source of truth — it prints configuration, user and agent identity, per-tool permission coverage, plugin + skill registration, and a tail of recent debug logs. Unconfigured fields show inline as `(unset)`.

## Quickstart (≈ 3 minutes)

From any project where you'd like Ory authentication, inside Codex:

1. **Start a local Ory instance.** Ask Codex *"start the local Ory stack"* or pick `ory-local-up` from the `/skills` menu.

   A banner prints the seeded test user's email and password. Note them — you'll log in with them in step 3.

2. **Scaffold Ory into your project.** Ask Codex *"add Ory auth to this app"* or pick `ory-auth-setup` from `/skills`.

   Codex installs Ory Elements, wires the SDK, generates the login / registration / recovery / verification / settings pages, and sets up session middleware. It targets the local stack from step 1, so no signup or API key is needed.

3. **Sign in.** Start your app, visit the login page Codex added, and sign in with the seeded credentials. You now have a real Ory session backed by a real Ory stack — locally, offline, with zero configuration.

4. **Turn on Ory login for the Codex session itself.** *(Optional but recommended.)* Out of the box the plugin only governs your *app*. To also attach an Ory identity to *Codex's* session — so every tool call is attributed to you, not a fallback `session:<id>` subject — opt in to the user-login flow:

   ```bash
   export ORY_USER_LOGIN=1
   ```

   User login is off by default. With it on, the next Codex session opens an Ory login in your browser; sign in with the same seeded credentials from step 1, and the token is reused on subsequent sessions until it expires. This is what makes `permissions enforce` (see [Agent security](#agent-security)) deny on the right identity later.

That's the full Ory DX path. Stop here if you're just evaluating the plugin. Continue to [Agent security](#agent-security) when you're ready to enforce.

## What's included

### Skills for scaffolding Ory into your application

Codex surfaces the skill catalog in `/skills` and auto-invokes by description. Ask Codex in natural language or invoke a skill directly:

- **`ory-auth-setup`** — full project setup. Install the Ory CLI, create an Ory Network project (or use the local one), add Ory Elements, configure the SDK, build the auth pages, wire session middleware.
- **`ory-login-flow`** — login, registration, recovery, verification, and settings pages with Ory Elements. Next.js App Router and React SPA variants.
- **`ory-social-login`** — Google, GitHub, Apple, Microsoft, Discord, and other OIDC providers with Jsonnet data mappers.
- **`ory-local-dev`** — drive the local Ory stack from within Codex to prototype and test without a remote project.
- **`ory-permissions-onboarding`** — bootstrap permission tuples for built-in tools, switch between observe and enforce mode, troubleshoot denials.
- **`ory-build-integration`** — pull the runnable subset of an `ory/integrates` template (webhook / config / http-event) into your own app and wire it to your Ory project — no contribution/registry concerns.
- **`ory-contribute-integration`** — author a brand-new integration as a contribution to `ory/integrates`, including `registry.entry.yaml`, the `Maintained by:` footer, DCO sign-off, and registry regeneration.
- **`ory-e2b-sandbox`** — scaffold an [E2B](https://e2b.dev) sandbox template that boots with this plugin preinstalled and registered, so every sandbox session is gated by Ory auth, permissions, and tracing without any per-sandbox setup.
- **`ory-build-agent`** — drop `@ory/argus` directly into a custom agent you own (Claude Agent SDK, OpenAI Agents SDK, Mastra, Vercel AI SDK, PydanticAI, LangGraph, Mistral AI, or Salesforce Agentforce) so the user is authenticated, every tool call is authorized against Ory Permissions, and the lifecycle emits trace spans.
- **`ory-temporal-worker`** — scaffold a [Temporal](https://temporal.io) TypeScript worker per the [official local-dev guide](https://docs.temporal.io/develop/typescript/set-up-your-local-typescript), with every Activity gated by an Ory permission check, the worker's agent identity resolved via DCR, and the full lifecycle emitting trace spans.

### Ory MCP server

Bundled and registered automatically. Exposes the Ory CLI and the Ory Network REST API as MCP tools so Codex can manage identities, OAuth2 clients, projects, permission tuples, and configuration without ever leaving the chat. Useful for seeding test data, verifying a scaffolded integration, or running one-off admin tasks.

### Local Ory stack

From Codex's `/skills` menu:

```
ory-local-up      # start a local Ory instance in Docker
ory-local-down    # tear it all down
ory-temporal-up   # start a local Temporal dev server (for ory-temporal-worker)
```

Or via the CLI: `npx -y -p @ory/codex ory-codex local up | down`.

`ory-local-up` brings up Ory Identities, OAuth2, and Permissions, plus a login UI on `:3000` and Jaeger on `:16686`, all reachable through `http://localhost:4000`. A test user identity is seeded and the credentials are printed for you. Use it to:

- **Learn Ory hands-on** without signing up for a hosted project.
- **Prototype** flows (login, social, MFA, recovery, permission tuples) against a real Ory backend.
- **Test** an auth integration end-to-end before pushing anything to a real environment.
- **Develop** your application against the same identity, OAuth2, and permission surfaces you'll ship with.

## Pointing at a real Ory project

The Quickstart uses the local stack. If you have a hosted [Ory Network](https://console.ory.sh) project, point the plugin at it:

```bash
npx -y -p @ory/codex ory-codex configure \
  --project-url https://<id>.projects.oryapis.com \
  --api-key ory_pat_...
```

Config is saved to `~/.config/ory-agent-plugins/config.json` and shared across every Ory agent plugin on the machine.

Without configuration the plugin still loads cleanly and runs in **pass-through mode**: skills work, but nothing is blocked. You can stay in pass-through mode indefinitely if you only want the DX features.

## Agent security

Once the plugin is pointed at an Ory project (local or hosted), Codex's session and every tool call can be governed by Ory.

- **Authentication.** Two identities. The human at the keyboard (the **user**) authenticates interactively via Ory Identities when user login is enabled (`ORY_USER_LOGIN=1`, off by default — browser PKCE flow on first session, persisted token thereafter). The Codex process (the **agent**) gets its own OAuth2 identity, self-registered via [Dynamic Client Registration (RFC 7591)](https://datatracker.ietf.org/doc/html/rfc7591) on first run.
- **Authorization.** Before any tool runs, the plugin checks [Ory Permissions](https://www.ory.com/docs/keto) (Zanzibar-style relation tuples) against the user's subject and blocks the call on `deny`. MCP tool calls additionally get a server-level check.
- **Audit.** Every decision (allow, deny, fallback) is recorded as a structured trace span: NDJSON file output and/or OTLP/HTTP export to Jaeger, Honeycomb, Grafana, and similar collectors. The user → agent delegation is written to Ory as a relation tuple so *"agent X acting on behalf of user Y"* stays queryable after tokens expire.

The plugin is **fail-open** on its own infrastructure failures (network errors, rate limits, missing config), so enforcement is only as strong as your tuples — grant explicit `invoke` relations for the tools each user should be able to run.

### Enable enforcement

After install the plugin runs in **observe mode**: every tool call is checked against Ory Permissions, but a deny is recorded as a `permission.observe_deny` audit span and the tool runs anyway. This lets you see what *would* be blocked before turning on hard blocking.

1. **Turn on user login.** It's off by default. In your shell:

   ```bash
   export ORY_USER_LOGIN=1
   ```

   The next Codex session opens a browser for PKCE login. Subsequent sessions reuse the persisted token until it expires.

2. **Bootstrap tuples for the built-in tools.** One idempotent command grants the current user `use` on every tool Codex ships with (shell, apply_patch, …):

   ```bash
   npx -y -p @ory/codex ory-codex permissions bootstrap
   ```

   If a user identity is already cached at install time, the installer runs this for you automatically — re-run after adding tools, switching subjects, or changing the namespace.

3. **Check coverage.** `permissions status` probes every tool in the harness's catalog and prints allowed / denied per tool:

   ```bash
   npx -y -p @ory/codex ory-codex permissions status
   ```

   Add tuples for any MCP server tools or custom commands by hand, or via the Ory MCP server from inside Codex (*"grant me use on the shell tool"*).

4. **Promote to enforce.** Once the observe-mode logs look right, switch over:

   ```bash
   npx -y -p @ory/codex ory-codex permissions enforce
   ```

   Denies now block the tool call; Codex shows the denial reason and the decision is recorded as a `tool.block` trace span with `blocked: true`. Switch back any time with `permissions observe`.

## CLI reference

```
npx -y -p @ory/codex ory-codex install | uninstall
npx -y -p @ory/codex ory-codex configure                            [--project-url <url>] [--api-key <key>] [--audit-only]
npx -y -p @ory/codex ory-codex agent <status|unregister>            Manage the agent's OAuth2 identity
npx -y -p @ory/codex ory-codex permissions <status|bootstrap|observe|enforce>
npx -y -p @ory/codex ory-codex local <up|down|status|seed|logs|env|configure|reset>
npx -y -p @ory/codex ory-codex status
```

Highlights:

- `agent status` — show the current persisted DCR identity for the agent.
- `permissions observe` / `permissions enforce` — switch between "log denies, allow through" (the install default) and "block denies." `permissions bootstrap` writes `use` tuples for the harness's built-in tools so the promotion path doesn't require hand-writing relationships.
- `configure --audit-only` — kill switch that disables Ory entirely (no auth, no permission checks; only audit logging of tool invocations). For phased rollouts, prefer `permissions observe` over `--audit-only`.
- `local seed` / `local env` — reseed the test user, or print env vars for pointing other tools at the local stack.

## Troubleshooting

- **`ory-local-up` fails.** Make sure Docker is running and ports `3000`, `4000`, `4100`, and `16686` are free.
- **PKCE login loops.** Clear persisted state with `npx -y -p @ory/codex ory-codex agent unregister` and retry.
- **`npx` fetches an old version.** Force a fresh fetch: `npx -y -p @ory/codex@latest ory-codex …`.
- **`npm install … ENOVERSIONS`.** If your `~/.npmrc` sets `min-release-age`, npm filters out package versions newer than that threshold. Override per-call with `npm_config_min_release_age=0 npx @ory/codex install`.
- **`codex doctor` reports `ory-mcp-server is not resolvable`.** The bundled MCP server resolves via `npx -y -p @ory/mcp-server` on demand — make sure `npm`/`npx` is on PATH. The first session start fetches the server tarball; subsequent sessions reuse the cache.
- **Need more signal.** Set `ORY_AGENT_DEBUG=true` and `ORY_AGENT_LOG_FILE=/tmp/ory.log` to capture structured logs.

## Links

- [Ory documentation](https://www.ory.com/docs/)
- [Ory Network console](https://console.ory.sh)
- [Ory Elements](https://github.com/ory/elements)
- [Codex documentation](https://github.com/openai/codex)

## License

Apache-2.0
