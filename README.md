# Paperclip on Fly.io

Deploy [Paperclip](https://docs.paperclip.ing) — an AI agent orchestration platform — as a Fly.io Machine app.

## Prerequisites

- [flyctl](https://fly.io/docs/flyctl/install/) installed and authenticated
- A Fly.io account with a payment method on file (for Machines)

## Setup

Clone the Paperclip source into this repo. The deploy builds upstream's Dockerfile directly, with one patch: a replacement `docker-entrypoint.sh` that fixes Fly volume ownership on boot (upstream's only chowns when UID/GID remapping occurs, but Fly volumes always mount as root).

```bash
git clone https://github.com/paperclipai/paperclip.git
```

To update Paperclip later:

```bash
git -C paperclip pull
```

## Deploy

> **App names are globally unique on Fly.io.** Pick your own name and update `fly.toml` — both the `app` field and `PAPERCLIP_AUTH_PUBLIC_BASE_URL`. The examples below use `my-paperclip` as a placeholder.

```bash
# 1. Pick a name and update fly.toml
#    app = "my-paperclip"
#    PAPERCLIP_AUTH_PUBLIC_BASE_URL = "https://my-paperclip.fly.dev"

# 2. Create the app
flyctl apps create my-paperclip

# 3. Create a persistent volume for Paperclip data (embedded DB, uploads, encryption keys)
#    --size is in GB. 10 GB is a reasonable starting point; expand later with `flyctl volumes extend`.
#    --vm-cpu-kind and --vm-cpus hint at the machine size so the volume lands on the right host.
flyctl volumes create paperclip_data \
  --region sjc \
  --size 10 \
  --vm-cpu-kind shared \
  --vm-cpus 2 \
  --app my-paperclip

# 4. Set the auth secret (required for authenticated mode)
flyctl secrets set BETTER_AUTH_SECRET=$(openssl rand -hex 32) --app my-paperclip

# 5. (Optional) Set API keys for agent adapters
flyctl secrets set ANTHROPIC_API_KEY=sk-ant-... --app my-paperclip
flyctl secrets set OPENAI_API_KEY=sk-... --app my-paperclip

# 6. Deploy
#    Copies fly.toml and our entrypoint fix into paperclip/ so the build
#    context has everything it needs.
cp fly.toml paperclip/ && cp docker-entrypoint.sh paperclip/scripts/ && flyctl deploy paperclip/
```

## First Run

After the first deploy, SSH into the machine to initialize Paperclip and create the admin account:

```bash
# Run these commands from this repo directory (or pass `--config /path/to/fly.toml`).
# `flyctl` reads `./fly.toml` from your current working directory, even for
# commands like `flyctl ssh console`.
#
# 1. Run onboard to generate the config file and auth secrets.
#    Use --yes to accept defaults. When it offers to start the server, say no —
#    the container's main process is already running.
# SSH runs as root, but Paperclip's data dir is owned by the node user.
# Run these commands as node so files are created with the right ownership.
# You may have to ctrl-c out of the booted server which it seems to do automatically
# at the moment but we don't need a server because it's already running on this machine
# Depending on your proxy setup, you may need the bind flag.
flyctl ssh console --app my-paperclip -C "su node - bash -c 'pnpm paperclipai onboard --yes --bind lan'"

# 2. (check output of above for a first admin URL or) Generate the first admin invite URL.
flyctl ssh console --app my-paperclip -C "su node - bash -c 'pnpm paperclipai auth bootstrap-ceo'"
```

Open the invite URL from step 2 in your browser to create the admin account.

### LLM adapters

Both of these happen in the console, so log into your Fly console first:

```bash
flyctl ssh console --app my-paperclip
```

#### Claude Code

To authenticate Claude Code for the adapter, SSH in and run through setup interactively:

Run `claude auth login --console` as node user in the fly console and follow its instructions.

Walk through the login prompts, then `/exit` when done.

#### OpenAI

Run `codex login --device-auth` as node user in the fly console and follow its instructions.

Enable device code authorization for Codex in ChatGPT Security Settings, then run "codex login --device-auth" again

---

Then open your app:

```bash
flyctl open
```
## MCP servers

While the official MCP server is in development, use this [third party one](https://github.com/zelenkoff/flyio-mcp-server).

## Configuration

Environment variables are set in `fly.toml` under `[env]`. Secrets should be set via `flyctl secrets set`.

| Variable | Default | Description |
|----------|---------|-------------|
| `PAPERCLIP_DEPLOYMENT_MODE` | `authenticated` | `local_trusted` or `authenticated` |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `public` | `private` or `public` |
| `PAPERCLIP_AUTH_BASE_URL_MODE` | `explicit` | Must be `explicit` for public exposure |
| `PAPERCLIP_AUTH_PUBLIC_BASE_URL` | — | Your app's public URL (e.g. `https://my-paperclip.fly.dev`) |
| `BETTER_AUTH_SECRET` | — | Secret key for auth sessions (set via `flyctl secrets set`) |

See the full list of environment variables in the [Paperclip docs](https://docs.paperclip.ing/deploy/environment-variables).

## Scaling

```bash
# Resize the machine
flyctl scale vm shared-cpu-4x --memory 4096 --app my-paperclip

# Expand the volume
flyctl volumes extend <volume-id> --size 20 --app my-paperclip
```

## How It Works

`flyctl deploy paperclip/` builds upstream's Dockerfile with `paperclip/` as the build context. Before deploying, `fly.toml` and a patched `docker-entrypoint.sh` are copied in. The entrypoint patch is minimal — it adds an unconditional `chown -R node:node /paperclip` because Fly volumes mount as root on first boot and upstream's entrypoint only fixes ownership during UID/GID remapping. Everything else is upstream. Data is persisted to a Fly volume mounted at `/paperclip`.
