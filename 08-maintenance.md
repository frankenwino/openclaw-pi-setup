# 08 - Maintenance

Common commands for managing the stack — logs, restarts, updates, backups, and troubleshooting.

## Common Commands

### View service status
```bash
cd ~/docker-stack
sudo docker compose ps
```

### View logs
```bash
sudo docker compose logs -f synapse
sudo docker compose logs -f element
sudo docker compose logs -f openclaw
sudo docker compose logs -f nginx
```

### Restart all services
```bash
sudo docker compose restart
```

### Restart a single service
```bash
sudo docker compose restart openclaw
```

### Stop all services
```bash
sudo docker compose down
```

### Start all services
```bash
sudo docker compose up -d
```

## Updating Docker Images

```bash
cd ~/docker-stack
sudo docker compose pull
sudo docker compose up -d
```

## Backup USB Volumes

```bash
sudo tar -czf ~/backup-$(date +%Y%m%d).tar.gz /mnt/docker-volumes/
```

Store backups off the Pi (e.g. copy to another machine):

```bash
scp pi@<PI_IP>:~/backup-*.tar.gz ~/backups/
```

## Check Disk Usage

```bash
df -h /mnt/docker-volumes/
du -sh /mnt/docker-volumes/*
```

## Renew mkcert Certificate

Certificates expire after 2 years. To renew:

```bash
cd ~
mkcert <TAILSCALE_IP> <LOCAL_IP> nexus.local localhost
sudo cp <IP>+3.pem /mnt/docker-volumes/nginx/certs/self-signed.crt
sudo cp <IP>+3-key.pem /mnt/docker-volumes/nginx/certs/self-signed.key
sudo docker compose restart nginx
```

## Add a New Matrix User

```bash
sudo docker compose exec synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008
```

## Reset OpenClaw Config

If OpenClaw config gets corrupted:

```bash
sudo docker compose stop openclaw
sudo rm /mnt/docker-volumes/openclaw/openclaw.json
sudo docker compose start openclaw
# Re-run onboarding
sudo docker compose exec -it openclaw node /app/dist/index.js onboard
```

## Troubleshooting

### Synapse won't start
```bash
sudo docker compose logs synapse | tail -30
# Check permissions
ls -la /mnt/docker-volumes/synapse/
# Fix permissions if needed
sudo chown -R 991:991 /mnt/docker-volumes/synapse/
```

### OpenClaw not receiving Matrix messages
```bash
sudo docker compose logs openclaw | grep -v "\[ws\]" | tail -20
# Restart if Matrix sync stalled
sudo docker compose restart openclaw
```

### nginx cert errors
```bash
sudo docker compose logs nginx | tail -10
# Verify cert files exist
ls -la /mnt/docker-volumes/nginx/certs/
```

### USB not mounted after reboot
```bash
sudo mount -a
df -h | grep docker-volumes
# If missing, check /etc/fstab UUID matches
sudo blkid
```
