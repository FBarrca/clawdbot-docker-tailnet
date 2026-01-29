# Clawdbot + Tailscale

A Docker setup for running [moltbot](https://github.com/moltbot/moltbot) (clawdbot) securely on your private Tailscale network. The gateway is exposed **only via Tailscale** with no host port mapping, making it accessible from any device on your tailnet.

<!-- admotion -->

> [!NOTE]
> Moltbot technically supports Tailscale natively, but I don't trust it, so I opted for this setup for better isolation and control.

## Quick Start

1. **Clone and initialize submodule:**

   ```bash
   git clone <repo-url>
   cd clawdbot-docker-tailnet
   git submodule update --init
   ```

2. **Build the moltbot image:**

```bash
cd moltbot
docker build \
  -t "moltbot:local" \
  -f "Dockerfile" \
  "."
cd ..
```

You can add --build-arg "CLAWDBOT_DOCKER_APT_PACKAGES=${CLAWDBOT_DOCKER_APT_PACKAGES}" \
to the build command to install extra packages. But you should not need to do this most of the time.

3. **Configure environment:**
   - Copy `docker/.env.example` to `docker/.env`
   - Set `TS_AUTHKEY=` to your reusable auth key ([tailscale.com admin → Keys](https://login.tailscale.com/admin/settings/keys))
   - Leave `CLAUDE_AI_SESSION_KEY`, `CLAUDE_WEB_SESSION_KEY`, and `CLAUDE_WEB_COOKIE` blank if you don't use the Claude integration (they can be set later)

4. **Start the stack:**

   ```bash
   docker compose -f docker/docker-compose.yml --env-file docker/.env up -d
   ```

5. **Access the gateway:**
   From any device on your tailnet, open `http://clawdbot:18789` or `http://clawdbot.<your-tailnet>.ts.net:18789`

## Architecture

- **Tailscale sidecar** (`ts-clawdbot`): Joins your tailnet and provides the network for the gateway
- **Gateway** (`moltbot-gateway`): Shares the Tailscale network; no host port mapping (only accessible via tailnet)
- **CLI** (`moltbot-cli`): Communicates with the gateway via the Docker bridge network
- **Custom bridge network** (`clawdbot`): Isolates the stack from other containers

## Tailscale Setup

### First Time Setup

1. **Sign up** at [tailscale.com](https://tailscale.com)
2. **Install Tailscale** on your devices to join the tailnet
3. **Generate auth key:**
   - Go to [admin console → Keys](https://login.tailscale.com/admin/settings/keys)
   - Click **Generate auth key**
   - Set: **Reusable** = yes, **Expiration** = 90 days
   - (Optional) Add a tag like `tag:docker` for access control
   - Copy the key (starts with `tskey-auth-`)

### Configuration

Set `TS_AUTHKEY` in `docker/.env`:

```env
TS_AUTHKEY=tskey-auth-...
```

The Tailscale state is persisted in the `tailscale-data-clawdbot` volume, so the node stays on your tailnet even after key expiry.

### Verify Setup

- Check [admin console → Machines](https://login.tailscale.com/admin/machines) for the **clawdbot** node
- Access the gateway from any tailnet device at `clawdbot:18789`
- Run CLI commands from the tailnet: `--host clawdbot` (ensure gateway token is set)

## Configuration

### Docker Compose (moltbot/docker-compose.yml)

| Variable                   | Default           | Description                                                 |
| -------------------------- | ----------------- | ----------------------------------------------------------- |
| **CLAWDBOT_GATEWAY_TOKEN** | _(none)_          | Gateway auth token (required). Generate via docker-setup.sh |
| **CLAUDE_AI_SESSION_KEY**  | _(none)_          | Claude AI session key                                       |
| **CLAUDE_WEB_SESSION_KEY** | _(none)_          | Claude web session key                                      |
| **CLAUDE_WEB_COOKIE**      | _(none)_          | Claude web cookie                                           |
| **CLAWDBOT_CONFIG_DIR**    | `$HOME/.clawdbot` | Host path for config                                        |
| **CLAWDBOT_WORKSPACE_DIR** | `$HOME/clawd`     | Host path for workspace                                     |

Set these in `docker/.env` (copy from `docker/.env.example`). The stack is run with `--env-file docker/.env`, so all variables used by the compose file must be defined there—use empty values for optional ones like the Claude keys to avoid warnings.

### Dockerfile (moltbot/Dockerfile)

| Variable                         | Default      | Description                                   |
| -------------------------------- | ------------ | --------------------------------------------- |
| **CLAWDBOT_DOCKER_APT_PACKAGES** | `""`         | Extra Debian packages (space-separated)       |
| **CLAWDBOT_PREFER_PNPM**         | `1`          | Prefer pnpm over Bun (useful on ARM/Synology) |
| **NODE_ENV**                     | `production` | Node environment                              |

## Submodule Management

[moltbot](https://github.com/moltbot/moltbot) is included as a git submodule that way you can easily update it to the latest version.

**Initialize (first time):**

```bash
git submodule update --init
```

**Update to latest:**

```bash
git submodule update --remote moltbot
```

## Accessing the container via Tailscale SSH

You can get a shell on the Clawdbot node from any device on your tailnet using [Tailscale SSH](https://tailscale.com/kb/1193/tailscale-ssh/).

1. **Enable Tailscale SSH** for the clawdbot machine:
   - Open [admin console → Machines](https://login.tailscale.com/admin/machines), select the **clawdbot** machine, and enable **SSH**.
   - Or enable tailnet-wide SSH under [Settings → SSH](https://login.tailscale.com/admin/settings/ssh).

2. **Connect from a device on your tailnet:**

   ```bash
   ssh clawdbot
   ```

   Or use the full name: `ssh clawdbot.<your-tailnet>.ts.net`

3. **Note:** The clawdbot node is the Tailscale container (`ts-clawdbot`). The default Tailscale image does not run an SSH server, so `ssh clawdbot` will only work if you run an SSH server inside that container (e.g. a custom image with `openssh-server`).  
   To get a shell in the container from the **host** where Docker runs, use:

   ```bash
   docker exec -it docker-ts-clawdbot-1 sh
   ```

   (Container name may vary; use `docker ps` to confirm.)
