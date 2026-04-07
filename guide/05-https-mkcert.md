# 05 - HTTPS with mkcert

Generate locally-trusted TLS certificates for the Pi and install the root CA on your client devices so browsers trust the connection.

mkcert creates locally-trusted certificates without browser warnings. It's free and works on your local network and Tailscale.

## Install mkcert on the Pi

```bash
sudo apt install mkcert libnss3-tools -y
mkcert -install
```

## Generate Certificate

Replace the IPs with your actual Tailscale IP, local IP, and hostname:

```bash
mkcert <TAILSCALE_IP> <LOCAL_IP> nexus.local localhost
```

Example:
```bash
mkcert <PI_TAILSCALE_IP> <PI_LOCAL_IP> nexus.local localhost
```

This creates two files in the current directory:
- `<IP>+3.pem` — certificate
- `<IP>+3-key.pem` — private key

## Copy Certs to nginx

```bash
sudo cp <IP>+3.pem /mnt/docker-volumes/nginx/certs/self-signed.crt
sudo cp <IP>+3-key.pem /mnt/docker-volumes/nginx/certs/self-signed.key
sudo docker compose restart nginx
```

Test HTTPS:

```bash
curl https://<TAILSCALE_IP>
```

## Root CA Location

The root CA that signs your certificate is at:

```bash
mkcert -CAROOT
# /home/pi/.local/share/mkcert/
```

You need to install `rootCA.pem` on every device that will access your services.

## Install Root CA on Client Devices

### Copy CA from Pi

```bash
scp pi@<PI_IP>:/home/pi/.local/share/mkcert/rootCA.pem ~/rootCA.pem
```

### Firefox (all platforms)

1. Open Firefox → Settings → Privacy & Security
2. Scroll to Certificates → click **View Certificates**
3. Click **Authorities** tab → **Import**
4. Select `rootCA.pem`
5. Check **Trust this CA to identify websites**
6. Click OK and restart Firefox

### Linux (system-wide)

```bash
sudo cp ~/rootCA.pem /usr/local/share/ca-certificates/nexus-local-ca.crt
sudo update-ca-certificates
```

### Linux (Electron apps like Element desktop)

```bash
certutil -d sql:$HOME/.pki/nssdb -A -t "CT,," -n "nexus-local-ca" -i ~/rootCA.pem
```

Restart the Electron app after running this.

### macOS

Double-click `rootCA.pem` to open Keychain Access, then:
1. Find the certificate in the keychain
2. Right-click → Get Info
3. Expand Trust → Set to **Always Trust**

### iOS / Android

Transfer `rootCA.pem` to the device and open it. Follow the prompts to install as a trusted CA.

## Certificate Renewal

Certificates expire after 2 years. To renew:

```bash
mkcert <TAILSCALE_IP> <LOCAL_IP> nexus.local localhost
sudo cp <IP>+3.pem /mnt/docker-volumes/nginx/certs/self-signed.crt
sudo cp <IP>+3-key.pem /mnt/docker-volumes/nginx/certs/self-signed.key
sudo docker compose restart nginx
```
