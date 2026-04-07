# 02 - Docker Setup

Install Docker Engine and the Compose plugin.

## References

- [Docker Engine install for Ubuntu](https://docs.docker.com/engine/install/ubuntu.md)
- [Docker Compose install](https://docs.docker.com/compose/install.md)
- [Docker Compose v2 docs](https://docs.docker.com/compose.md)

## Install Docker Engine

```bash
# Add Docker's official GPG key
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Verify Installation

```bash
sudo systemctl status docker
sudo docker run hello-world
```

## Add User to Docker Group (optional)

```bash
sudo usermod -aG docker $USER
newgrp docker
```
