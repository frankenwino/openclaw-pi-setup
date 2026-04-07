# 04 - Stack Deployment

Create the Docker Compose configuration and start all services: PostgreSQL, Synapse, Element, nginx, and OpenClaw.

## References

- [OpenClaw Docker install](https://docs.openclaw.ai/install/docker.md)
- [Synapse Docker docs](https://hub.docker.com/r/matrixdotorg/synapse)

## Configuration Files

Create a working directory:

```bash
mkdir -p ~/docker-stack
cd ~/docker-stack
```

All config files below are created in `~/docker-stack/`.

### ~/docker-stack/docker-compose.yml

```yaml
services:
  # PostgreSQL database for Synapse
  postgres:
    image: postgres:15-alpine
    container_name: synapse-db
    environment:
      POSTGRES_DB: synapse
      POSTGRES_USER: synapse
      POSTGRES_PASSWORD: change_me_strong_password  # change this
    volumes:
      - /mnt/docker-volumes/synapse/postgres:/var/lib/postgresql/data
    networks:
      - stack-network
    restart: unless-stopped

  # Synapse Matrix homeserver
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: synapse
    depends_on:
      - postgres
    environment:
      SYNAPSE_SERVER_NAME: matrix.local
      SYNAPSE_REPORT_STATS: "no"
      SYNAPSE_CONFIG_DIR: /data
      SYNAPSE_CONFIG_PATH: /data/homeserver.yaml
    volumes:
      - /mnt/docker-volumes/synapse:/data
    ports:
      - "8008:8008"
    networks:
      - stack-network
    restart: unless-stopped

  # Element web client
  element:
    image: vectorim/element-web:latest
    container_name: element
    depends_on:
      - synapse
    volumes:
      - /mnt/docker-volumes/element/config.json:/app/config.json:ro
    ports:
      - "80:80"
    networks:
      - stack-network
    restart: unless-stopped

  # Nginx reverse proxy for HTTPS
  nginx:
    image: nginx:latest
    container_name: nginx
    depends_on:
      - synapse
    volumes:
      - /mnt/docker-volumes/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - /mnt/docker-volumes/nginx/certs:/etc/nginx/certs:ro
    ports:
      - "443:443"
    networks:
      - stack-network
    restart: unless-stopped

  # OpenClaw AI agent
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    container_name: openclaw
    environment:
      OPENCLAW_HOME: /home/node/.openclaw
      OPENCLAW_STATE_DIR: /home/node/.openclaw
    volumes:
      - /mnt/docker-volumes/openclaw:/home/node/.openclaw
    ports:
      - "18789:18789"
    networks:
      - stack-network
    restart: unless-stopped

networks:
  stack-network:
    driver: bridge
```

### ~/docker-stack/element-config.json

Replace `<YOUR_TAILSCALE_IP>` with your Pi's Tailscale IP (see [03-tailscale.md](03-tailscale.md)):

```json
{
  "default_server_config": {
    "m.homeserver": {
      "base_url": "https://<YOUR_TAILSCALE_IP>",
      "server_name": "matrix.local"
    },
    "m.identity_server": {
      "base_url": "https://vector.im"
    }
  },
  "brand": "Element",
  "integrations_ui_url": "https://scalar.vector.im/",
  "integrations_rest_url": "https://scalar.vector.im/api",
  "bug_report_endpoint_url": "https://element.io/bugreports/submit",
  "showLabsSettings": true,
  "features": {},
  "default_country_code": "GB",
  "roomDirectory": {
    "servers": ["matrix.org"]
  }
}
```

### ~/docker-stack/nginx.conf

First, get your Pi's local IP:

```bash
hostname -I | awk '{print $1}'
```

Then create `~/docker-stack/nginx.conf`, replacing `<YOUR_TAILSCALE_IP>` and `<YOUR_LOCAL_IP>` with your actual IPs:

```
server {
    listen 443 ssl;
    server_name <YOUR_TAILSCALE_IP> <YOUR_LOCAL_IP> nexus.local localhost;

    ssl_certificate /etc/nginx/certs/self-signed.crt;
    ssl_certificate_key /etc/nginx/certs/self-signed.key;

    # Synapse Matrix API
    location /_matrix {
        proxy_pass http://synapse:8008;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Element web client (default)
    location / {
        proxy_pass http://element:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Generate Synapse Config

Run this once to generate Synapse's initial configuration directly to USB:

```bash
sudo docker run -it --rm \
  --mount type=bind,src=/mnt/docker-volumes/synapse,dst=/data \
  -e SYNAPSE_SERVER_NAME=matrix.local \
  -e SYNAPSE_REPORT_STATS=no \
  matrixdotorg/synapse:latest generate
```

Fix permissions (Synapse runs as uid 991):

```bash
sudo chown -R 991:991 /mnt/docker-volumes/synapse/
sudo chmod -R 700 /mnt/docker-volumes/synapse/
```

## Copy Config Files to USB

```bash
sudo cp ~/docker-stack/element-config.json /mnt/docker-volumes/element/config.json
sudo cp ~/docker-stack/nginx.conf /mnt/docker-volumes/nginx/nginx.conf
sudo chown 1000:1000 /mnt/docker-volumes/element/config.json
```

## Migrate Synapse to PostgreSQL

The default Synapse config uses SQLite. Migrate to PostgreSQL for better performance.

Update `/mnt/docker-volumes/synapse/homeserver.yaml` - replace the `database:` section with:

```yaml
database:
  name: psycopg2
  allow_unsafe_locale: true
  args:
    user: synapse
    password: your_postgres_password
    database: synapse
    host: synapse-db
    cp_min: 5
    cp_max: 10
```

Start the stack first, then run the migration:

```bash
sudo docker compose up -d
sudo docker compose exec synapse bash -c "synapse_port_db --sqlite-database /data/homeserver.db --postgres-config /data/homeserver.yaml"
```

Change the default PostgreSQL password:

```bash
sudo docker exec synapse-db psql -U synapse -c "ALTER USER synapse PASSWORD 'your_new_password';"
```

Update `POSTGRES_PASSWORD` in `docker-compose.yml` to match, then restart:

```bash
sudo docker compose restart synapse
```

## Start the Stack

```bash
cd ~/docker-stack
sudo docker compose up -d
```

Wait 30 seconds, then verify all services are running:

```bash
sudo docker compose ps
```

All containers should show `Up` status.

## Create Synapse Admin User

```bash
sudo docker compose exec synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```

When prompted:
- **Username**: `root` (this becomes your Matrix ID: `@root:matrix.local`)
- **Password**: use a strong password
- **Make admin**: `yes`

> **Note**: The username `root` is just a Matrix account name - it has no relation to the Linux root user. You can use any username you prefer.

## Configure OpenClaw Gateway Binding

OpenClaw defaults to loopback only. Set it to LAN mode:

```bash
sudo docker compose exec openclaw node /app/dist/index.js config set gateway.bind lan
sudo docker compose restart openclaw
```

Verify it's accessible:

```bash
curl http://localhost:18789/healthz
```

Should return `{"ok":true,"status":"live"}`.

## Verify Access from Another Device

From another device connected to Tailscale, verify the stack is reachable:

```bash
curl https://<PI_TAILSCALE_IP>
curl https://<PI_TAILSCALE_IP>/_matrix/client/versions
curl http://<PI_TAILSCALE_IP>:18789/healthz
```

> **Note**: The HTTPS endpoints will show a certificate warning until you install the mkcert root CA — see [05-https-mkcert.md](05-https-mkcert.md).
