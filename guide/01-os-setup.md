# 01 - OS Setup

Flash Ubuntu Server onto the Pi, format the USB stick, and mount it as persistent storage for Docker volumes.

## Flash Ubuntu Server

1. Download [Ubuntu Server 24.04 LTS](https://ubuntu.com/download/raspberry-pi) for Raspberry Pi
2. Flash to SD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
3. Boot the Pi and complete initial setup

## Initial System Update

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

## Format and Mount USB Stick

Find the USB device:

```bash
lsblk
```

Format as ext4 (replace `sda` with your device):

```bash
sudo mkfs.ext4 /dev/sda1
```

Create mount point and mount:

```bash
sudo mkdir -p /mnt/docker-volumes
sudo mount /dev/sda1 /mnt/docker-volumes
```

Get the UUID for auto-mount:

```bash
sudo blkid /dev/sda1
```

Add to `/etc/fstab` for auto-mount on boot:

```bash
sudo nano /etc/fstab
```

Add this line (replace UUID with yours):

```
UUID=your-uuid-here /mnt/docker-volumes ext4 defaults 0 2
```

Create volume directories:

```bash
sudo mkdir -p /mnt/docker-volumes/{synapse,element,openclaw,nginx/certs}
sudo chown -R 1000:1000 /mnt/docker-volumes
```

Verify after reboot:

```bash
sudo reboot
df -h | grep docker-volumes
```
