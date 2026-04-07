# 07 - Client Setup

This doc covers setting up Element on the devices you'll use to chat with OpenClaw — your laptop, phone, or any other machine on your Tailscale network.

## Element Web (Browser)

Access via browser at `https://<PI_TAILSCALE_IP>`.

**Requirements:**
- Install mkcert root CA in Firefox (see [04-https-mkcert.md](04-https-mkcert.md))
- Connect to Tailscale

## Element Desktop App (Linux)

### Install

Download from [element.io/download](https://element.io/download) or:

```bash
sudo apt install element-desktop
```

### Configure

1. Launch Element desktop
2. Click **Edit** next to the homeserver field
3. Select **Other homeserver**
4. Enter `https://<PI_TAILSCALE_IP>`
5. Click **Continue**

### Trust the Certificate

The desktop app uses Chromium's NSS certificate store. Install the root CA:

```bash
# Copy CA from Pi
scp pi@<PI_IP>:/home/pi/.local/share/mkcert/rootCA.pem ~/rootCA.pem

# Install into NSS store
certutil -d sql:$HOME/.pki/nssdb -A -t "CT,," -n "nexus-local-ca" -i ~/rootCA.pem
```

Restart Element desktop after running this.

## Element Mobile (iOS / Android)

1. Install Element from App Store / Play Store
2. Tap **Edit** on the homeserver screen
3. Enter `https://<PI_TAILSCALE_IP>`
4. Install the mkcert root CA on your device (see [04-https-mkcert.md](04-https-mkcert.md))

## Logging In

Use the credentials you created during Synapse setup:

- **Username**: your Matrix username (e.g. `admin`)
- **Password**: your Synapse admin password
- **Homeserver**: `https://<PI_TAILSCALE_IP>`

## Chatting with OpenClaw

1. Start a Direct Message with `@openclaw:matrix.local`
2. Send any message
3. If prompted with a pairing code, run on the Pi:
   ```bash
   sudo docker compose exec openclaw node /app/dist/index.js pairing approve matrix <CODE>
   ```
4. OpenClaw will now respond to your messages

## Troubleshooting

### "Cannot connect to homeserver"
- Check you're connected to Tailscale
- Verify the Pi is running: `sudo docker compose ps`
- Check the cert is trusted in your browser/app

### "Certificate not trusted" in Element desktop
- Run the `certutil` command above and restart Element

### OpenClaw not responding
- Check logs: `sudo docker compose logs openclaw | tail -20`
- Verify Matrix channel is connected: `sudo docker compose logs openclaw | grep matrix`
