# 03 - Tailscale Setup

Install Tailscale on the Pi and connect it to your private network. You'll need your Pi's Tailscale IP before configuring the stack in the next step.

## References

- [Tailscale install docs](https://tailscale.com/docs/install)

Tailscale creates a private mesh network between your devices. Your Pi and all your other devices get private IPs that can reach each other securely from anywhere.

## Install Tailscale on the Pi

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

A URL will appear. Open it in your browser and authenticate with your Tailscale account (GitHub SSO works).

## Check Status

```bash
sudo tailscale status
```

Note your Pi's Tailscale IP (e.g. `110.26.13.37`).

## Install Tailscale on Other Devices

Install Tailscale on any device you want to use to access Element and chat with OpenClaw.

Download from [tailscale.com/download](https://tailscale.com/download) for:
- Linux, macOS, Windows, iOS, Android

Sign in with the same account on each device.

## Notes

- **Mullvad VPN conflicts with Tailscale** — disconnect Mullvad before using Tailscale
- Tailscale free tier supports up to 3 users and 100 devices
- HTTPS certificates from Tailscale (`tailscale cert`) require a paid plan — use mkcert instead (see [05-https-mkcert.md](05-https-mkcert.md))
