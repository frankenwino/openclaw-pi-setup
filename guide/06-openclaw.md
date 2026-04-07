# 06 - OpenClaw Configuration

Run OpenClaw onboarding, connect it to your Synapse server, and pair your Matrix account so you can start chatting.

## References

- [OpenClaw Docker install](https://docs.openclaw.ai/install/docker.md)
- [OpenClaw Matrix channel docs](https://docs.openclaw.ai/channels/matrix.md)
- [OpenClaw gateway configuration](https://docs.openclaw.ai/gateway/configuration.md)

## Run Onboarding

```bash
sudo docker compose exec -it openclaw node /app/dist/index.js onboard
```

Follow the prompts to:
- Select your LLM provider (OpenAI, Anthropic, GitHub Copilot, etc.)
- Configure basic settings

## Configure Gateway Binding

By default OpenClaw only listens on loopback. Set it to LAN mode so it's accessible from outside the container:

```bash
sudo docker compose exec openclaw node /app/dist/index.js config set gateway.bind lan
sudo docker compose restart openclaw
sleep 15
curl http://localhost:18789/healthz
```

## Connect OpenClaw to Synapse (Matrix)

### Create a Matrix Bot User

```bash
sudo docker compose exec synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```

- **Username**: `openclaw` (or any name)
- **Password**: use a strong password
- **Make admin**: `no`

### Configure OpenClaw Matrix Channel

```bash
# Allow connection to private/internal network
sudo docker compose exec openclaw node /app/dist/index.js config set channels.matrix.allowPrivateNetwork true

# Set homeserver — OpenClaw and Synapse are on the same Docker network,
# so we use the service name "synapse" instead of localhost or an IP
sudo docker compose exec openclaw node /app/dist/index.js config set channels.matrix.homeserver http://synapse:8008

# Set bot credentials
sudo docker compose exec openclaw node /app/dist/index.js config set channels.matrix.userId @openclaw:matrix.local
sudo docker compose exec openclaw node /app/dist/index.js config set channels.matrix.password 'your_bot_password'

# Allow auto-joining rooms
sudo docker compose exec openclaw node /app/dist/index.js config set channels.matrix.autoJoin always

# Allow messages from all users in rooms
sudo docker compose exec openclaw node /app/dist/index.js config set channels.matrix.groupPolicy open
```

Restart to apply:

```bash
sudo docker compose restart openclaw
```

## Pair Your Matrix Account

Start a DM with `@openclaw:matrix.local` in Element. OpenClaw will send a pairing code. Approve it:

```bash
sudo docker compose exec openclaw node /app/dist/index.js pairing approve matrix <PAIRING_CODE>
```

## Verify OpenClaw is Responding

Send a message to `@openclaw:matrix.local` in Element. It should respond.

## OpenClaw Web UI

The OpenClaw web UI is available at `https://<PI_TAILSCALE_IP>:18790/chat?session=main`.

### First-time access

The UI requires a gateway token. Get it by running on the Pi:

```bash
sudo docker compose exec openclaw node /app/dist/index.js dashboard --no-open
```

This prints a URL like:
```
http://127.0.0.1:18789/#token=<your-token>
```

Open the UI and paste the token when prompted, or open:
```
https://<PI_TAILSCALE_IP>:18790/#token=<your-token>
```

The token is also stored in `/mnt/docker-volumes/openclaw/openclaw.json` under `gateway.auth.token`.



```
http://localhost:18789
```

Or configure `allowedOrigins` for remote access:

```bash
sudo docker compose exec openclaw node /app/dist/index.js config set gateway.controlUi.allowedOrigins '["https://<YOUR_IP>:18789"]' --strict-json
```

## Useful Commands

```bash
# View current config
sudo docker compose exec openclaw node /app/dist/index.js config get

# Check channel status
sudo docker compose exec openclaw node /app/dist/index.js channels status

# View logs
sudo docker compose logs openclaw -f
```
