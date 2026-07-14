# Ory Agent Plugin: Codex

Security and developer experience for [Codex](https://github.com/openai/codex), powered by [Ory](https://ory.com).

**Security.** Codex runs real actions on your machine — editing files, running shell commands, calling APIs. The plugin gives every session a verifiable identity (you sign in once; Codex and any sub-agents it spawns each get their own), checks every tool call against permissions you control, and records each decision as an audit trace you can ship to your observability stack. It starts in watch mode so nothing is blocked on day one, and if Ory is ever unreachable it steps aside rather than locking you out.

**Developer experience.** A single command installs the plugin and walks you through connecting — choose Ory Network, a local Docker stack, or audit-only, and it wires up the project, sign-in client, login, and permissions for you. It also helps you build Ory into your own app: ask in plain language to scaffold login, registration, and recovery pages, run a local Ory, or manage identities and permissions through the bundled MCP server.

## What you'll need

- [Codex](https://github.com/openai/codex), installed and signed in
- Node.js **22 or newer**
- [Docker](https://docs.docker.com/get-docker/) — only if you want to run Ory locally
- macOS or Linux (Windows works via WSL2)

## Get started

Run one command. It installs the plugin and walks you through connecting:

```bash
npx -y -p @ory/codex ory-codex install
```

This registers the Ory plugin with Codex (hooks, skills, and a bundled Ory tool server) and then asks how you want to connect — **press Enter for the default**:

- **Ory Network** *(default)* — sign in, or create a free account, in your browser. The project, keys, permissions, and login are all set up for you. Nothing to configure by hand.
- **Local** — run a complete Ory on your laptop with Docker. No account, no signup, no keys. Great for trying it out.
- **Audit-only** — skip Ory entirely and just log what Codex does.

That's it. Confirm everything landed with:

```bash
npx -y -p @ory/codex ory-codex status
```

`status` is your one-stop check: what's configured, who's signed in, which tools are covered by permissions, and recent activity. Anything not set up yet shows as `(unset)`.

> **First launch: trust the Ory hooks.** Codex treats a freshly installed plugin's hooks as untrusted, so your first Codex session asks you to review and trust them. Sign-in and per-tool checks only start running once you do. In the TUI, that first check kicks in on your opening turn.

Re-run install with `--reconfigure` to change your connection later, or `--no-configure` to skip the wizard and connect by hand.

## What you get

Once connected, every tool Codex runs is governed by Ory — three things happen automatically:

- **Who's driving.** You sign in once in your browser (a standard secure browser sign-in — no tokens to copy around). Codex itself registers its own identity automatically the first time it runs. The "who acted on whose behalf" trail stays queryable later.
- **What it's allowed to do.** Before a tool runs, Ory checks whether it's permitted. It starts in **watch mode** — nothing is blocked, you just *see* what would be — so it never gets in your way on day one.
- **A record of everything.** Every decision (allowed, denied, skipped) is logged as a trace you can send to Jaeger, Honeycomb, Grafana, or just a file.

If Ory is ever unreachable, the plugin gets out of the way and lets Codex keep working — so it can't lock you out. That also means enforcement is only as strong as the permissions you grant.

### See what's happening

Everything the plugin does is observable out of the box — no configuration required:

- **Status at a glance.** `npx -y -p @ory/codex ory-codex status` shows what's configured, who's signed in, how many built-in tools your permissions cover, and the most recent tool-call activity.
- **Live traces.** Every tool call is recorded as an OpenTelemetry-style span. Watch them stream as the agent works:

  ```bash
  npx -y -p @ory/codex ory-codex watch
  ```

  Spans are also written to `~/.config/ory-agent-plugins/codex/ory-agent-trace.ndjson` (NDJSON, one span per line) — tail that file, or point `OTEL_EXPORTER_OTLP_ENDPOINT` at a collector to ship them straight to Jaeger, Honeycomb, or Grafana.
- **Debug log.** For a verbose play-by-play, set `ORY_AGENT_DEBUG=true`; structured logs land in `~/.config/ory-agent-plugins/codex/ory-agent-debug.log`.

### Ready to enforce?

When the watch-mode logs look right, turn on blocking with one command (setup already granted you the built-in tools):

```bash
npx -y -p @ory/codex ory-codex permissions enforce
```

Now a denied tool is actually blocked and Codex shows why. Go back to watch mode anytime with `permissions observe`. Use `permissions status` to see what's covered and `permissions bootstrap` to (re-)grant the built-in tools — or just ask Codex in chat, e.g. *"grant me use of the shell tool."*

## Also: add login to your own app

Beyond securing Codex, the plugin helps you build Ory into whatever you're working on. Ask Codex *"add Ory login to this app"* — or pick `ory-auth-setup` from the `/skills` menu — and it scaffolds the login, registration, recovery, and settings pages (using [Ory Elements](https://github.com/ory/elements)) wired to a local Ory, so no signup or keys are needed. Start that local Ory with the `ory-local-up` skill (it prints a test email + password to sign in with) and tear it down with `ory-local-down`.

Bundled **skills** (ask in plain language, or pick from `/skills`) cover more: `ory-auth-setup`, `ory-login-flow`, `ory-social-login` (Google, GitHub, Apple…), `ory-permissions-onboarding`, and playbooks for wiring Ory into your own agents, E2B sandboxes, or Temporal workers. A built-in **Ory tool server** lets Codex manage identities, projects, and permissions straight from chat.

## Configure by hand (CI / advanced)

The guided setup covers most people. For scripted or CI setups, or to point at an existing Ory Network project, connect directly. Settings are saved to `~/.config/ory-agent-plugins/config.json` and shared across all your Ory agent plugins; environment variables win when both are set.

```bash
npx -y -p @ory/codex ory-codex configure \
  --project-url https://<slug>.projects.oryapis.com \
  --oauth2-client-id <sign-in client id> \
  --user-login
```

Codex's own identity registers itself automatically on first run — nothing to create. The `--oauth2-client-id` is the one piece browser sign-in needs; the guided setup makes it for you, or see below to do it by hand. For logging-only with no checks, use `--audit-only`.

<details>
<summary>Create the sign-in client by hand</summary>

The guided setup normally does this. To do it yourself, create a **public** OAuth2 client (no secret) listing all four loopback URLs — the plugin tries each in turn at runtime so sign-in survives a busy port, and Ory only accepts a callback on a URL you registered:

```bash
ory create oauth2-client --project <project-id> \
  --name "ory-agent-plugin" \
  --grant-type authorization_code,refresh_token \
  --response-type code \
  --scope openid,offline_access \
  --token-endpoint-auth-method none \
  --redirect-uri http://127.0.0.1:47823/callback \
  --redirect-uri http://127.0.0.1:47824/callback \
  --redirect-uri http://127.0.0.1:47825/callback \
  --redirect-uri http://127.0.0.1:47826/callback
```

…or in the [Ory Console](https://console.ory.sh) under *OAuth2* → *Clients* → *Create client* (pick "Public client", set "Authorization Code" + "Refresh Token" grants, scopes `openid offline_access`, paste the four URLs above). Pass the resulting id to `configure --oauth2-client-id`. Running headless with a session token already? Set `ORY_USER_SESSION_TOKEN` and skip the browser step entirely.

</details>

With nothing configured, the plugin still loads and runs in **pass-through mode**: skills, commands, and logging work, but no checks run and nothing is blocked. Perfectly fine if you only want the app-building features.

## Commands

```
ory-codex install | uninstall        Install/remove; --reconfigure re-runs setup, --no-configure skips it
ory-codex status                     Show configuration, identities, permission coverage, recent activity
ory-codex watch                      Tail the live trace stream (OTel spans)
ory-codex permissions <cmd>          status | bootstrap | observe (watch) | enforce (block)
ory-codex configure <flags>          Point at a project by hand (--project-url, --oauth2-client-id, --user-login, --audit-only)
ory-codex agent <status|unregister>  Manage Codex's own auto-created identity
ory-codex local <up|down|status|…>   Run / manage a local Ory in Docker
```

All prefixed with `npx -y -p @ory/codex`.

The local stack runs a complete Ory on your laptop: the Ory APIs at `http://localhost:4000`, a login UI on `:4455` (not :3000, to avoid Next.js port conflicts), the Ory Console on `:4100`, and Jaeger (the trace viewer) on `:16686`.

## Troubleshooting

- **`local up` fails** — make sure Docker is running and ports `4000`, `4100`, `4455`, and `16686` are free.
- **Browser sign-in loops** — reset with `ory-codex agent unregister` and try again.
- **Install ran but never asked how to connect** — you're almost certainly on a stale `npx` cache. `npx -p @ory/codex` (no version pin) reuses a previously-downloaded copy instead of re-resolving to the latest, so an older CLI whose install predates the setup wizard can run while you believe you're on the current release. The install banner prints the running version; confirm it with `npx -y -p @ory/codex ory-codex version`. To force the current release, clear the cache and reinstall: `rm -rf ~/.npm/_npx` then `npx -y -p @ory/codex ory-codex install`. Pinning an exact version (`@ory/codex@<version>`) also bypasses the cached copy.
- **`npm install … ENOVERSIONS`** — if your `~/.npmrc` sets `min-release-age`, npm hides versions newer than that. Override per-call: `npm_config_min_release_age=0 npx -y -p @ory/codex ory-codex install`.
- **`codex doctor` says `ory-mcp-server is not resolvable`** — the bundled tool server is fetched on demand via `npx`, so make sure `npm`/`npx` is on your PATH. The first session downloads it; later ones reuse the cache.
- **Want to see what's happening** — `npx -y -p @ory/codex ory-codex status` for a snapshot, `npx -y -p @ory/codex ory-codex watch` for the live trace stream, or set `ORY_AGENT_DEBUG=true` for a verbose log. Traces and logs live under `~/.config/ory-agent-plugins/codex/` (see [See what's happening](#see-whats-happening)).

## Learn more

- [Ory documentation](https://www.ory.com/docs/) · [Ory Console](https://console.ory.sh) · [Ory Elements](https://github.com/ory/elements)
- [Codex documentation](https://github.com/openai/codex)

## License

Apache-2.0
