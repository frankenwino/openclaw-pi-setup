# Self-Hosted AI Stack on Raspberry Pi 4

A guide to running [OpenClaw](https://openclaw.ai) on a Raspberry Pi 4, using Matrix (Synapse + Element) as the messaging layer, secured with HTTPS and accessible remotely via Tailscale.

## Contents

1. [OS Setup](guide/01-os-setup.md)
2. [Docker Setup](guide/02-docker-setup.md)
3. [Tailscale Setup](guide/03-tailscale.md)
4. [Stack Deployment](guide/04-stack-deployment.md)
5. [HTTPS with mkcert](guide/05-https-mkcert.md)
6. [OpenClaw Configuration](guide/06-openclaw.md)
7. [Client Setup](guide/07-client-setup.md)
8. [Maintenance](guide/08-maintenance.md)

---

## Goal

The primary goal is a self-hosted OpenClaw instance running on your own hardware.

---

## Architecture

```
┌─────────┐     ┌─────────┐     ┌───────────┐     ┌─────────┐
│ Element │────▶│ Synapse │────▶│ OpenClaw  │────▶│ LLM API │
│ (client)│◀────│(Matrix) │◀────│ (gateway) │◀────│         │
└─────────┘     └─────────┘     └───────────┘     └─────────┘

All services run in Docker on the Pi, behind nginx, accessible over Tailscale.
```

---

## Why Self-Host?

The simplest alternative is to spin up a VPS, install OpenClaw, and point it at Telegram or Discord as the messaging channel. That works fine and takes less effort. If you just want OpenClaw running quickly, that's a reasonable path.

This setup takes a different approach, for a few reasons:

**Cost** — A VPS costs money. A Pi you already own costs nothing to run. While you're still experimenting and figuring out what you actually want from an OpenClaw, there's no point paying for infrastructure.

**It's fun** — Running your own stack is interesting. You learn how the pieces fit together.

**Privacy** — This is the more principled reason. A VPS host with access to the machine can read your data. Telegram and Discord require trusting those platforms with your conversation history. With a self-hosted Synapse server, you own the message store and the encryption keys. Matrix and all the software in this stack are open source — you can verify what they actually do.

The one compromise is the LLM provider — see [The Trade-off: LLM Provider](#the-trade-off-llm-provider) below.

---

## The Trade-off: LLM Provider

Ideally, the LLM would also run locally via [Ollama](https://ollama.com), keeping everything on your own hardware. In practice, the Pi 4 is too underpowered for useful local inference — and OpenClaw requires a model with at least a 16k context window, which rules out most models that would run acceptably on low-spec hardware.

For now, this setup uses GitHub Copilot (GPT-4o) as the LLM backend. This is the one part of the stack that phones home. OpenClaw also supports OpenAI, Anthropic, and others. Local inference via Ollama is worth revisiting as hardware improves.

---

## Prerequisites

### Hardware

- Raspberry Pi 4 (2GB RAM minimum, 4GB recommended)
- SD card (16GB+) — for the OS only
- USB stick (8GB+) — stores all Docker volume data and container configs. If you need to rebuild the system, everything is on the USB stick rather than baked into the OS.

### Accounts

- [GitHub](https://github.com) account — used for Tailscale SSO and OpenClaw's Copilot authentication
- [Tailscale](https://tailscale.com) account (free tier is fine)

### What Gets Installed

| Software | Purpose |
|----------|---------|
| Ubuntu Server 24.04 LTS | Operating system |
| Docker Engine + Compose plugin | Container runtime |
| Tailscale | Private network access |
| mkcert | Locally-trusted TLS certificates |
| Synapse | Matrix homeserver (Docker) |
| Element | Matrix web client (Docker) |
| nginx | Reverse proxy / HTTPS termination (Docker) |
| PostgreSQL | Synapse database (Docker) |
| OpenClaw | AI gateway (Docker) |

---

## Naming Conventions

Throughout these docs, `nexus.local` refers to the Pi's hostname on the local network. Your Pi's hostname is set during Ubuntu Server setup. Replace `nexus.local` with your own hostname wherever it appears.

```bash
hostname                                  # check current hostname
sudo hostnamectl set-hostname your-hostname  # change it
```

---

## Alternative: Running in a VirtualBox VM

These docs work on a VirtualBox VM with minor adjustments.

1. Create a VM: Ubuntu Server 24.04 LTS, 2GB+ RAM, 20GB+ disk, network set to **Bridged Adapter**
2. Follow these docs from [02-docker-setup.md](guide/02-docker-setup.md) onwards
3. Instead of a USB stick, create a local directory:
   ```bash
   sudo mkdir -p /opt/docker-volumes/{synapse,element,openclaw,nginx/certs}
   sudo chown -R 1000:1000 /opt/docker-volumes
   ```
4. Update all volume paths in `docker-compose.yml` from `/mnt/docker-volumes/` to `/opt/docker-volumes/`

Get the VM's IP with:
```bash
ip addr show | grep "inet " | grep -v 127
```

## Alternative: Native Install

If you prefer to run OpenClaw natively on the Pi without Docker, see the [OpenClaw Raspberry Pi install guide](https://docs.openclaw.ai/install/raspberry-pi.md).
