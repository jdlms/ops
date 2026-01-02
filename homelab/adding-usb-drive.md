# Proxmox USB Storage Setup

## Architecture

```
┌──────────────────────────────────┐
│         Proxmox Host             │
│                                  │
│  /mnt/mediastore                 │ ← USB drive mounted
│      │                           │
│      ├──► Jellyfin (LXC/Docker)  │  ← direct bind mount
│      │                           │
│      └──► NFS Server             │  ← exports to network
│                                  │
└──────────────┬───────────────────┘
               │
         ┌─────┴─────┐
         ▼           ▼
      VM 1         VM 2   ← mount via NFS
```

## 1. Identify the Drive

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL
```

## 2. Wipe and Partition

```bash
# Install parted if needed
apt update && apt install parted

# Wipe (replace sdX with your device)
wipefs -a /dev/sdX

# Create GPT partition table and partition
parted /dev/sdX --script mklabel gpt mkpart primary ext4 0% 100%

# Format with label
mkfs.ext4 -L mediastore /dev/sdX1
```

Alternative using fdisk (interactive):
```bash
fdisk /dev/sdX
# g - create GPT table
# n - new partition (accept defaults)
# w - write and exit
```

## 3. Mount the Drive

```bash
# Create mount point
mkdir -p /mnt/mediastore

# Get UUID
blkid /dev/sdX1

# Add to fstab (use your UUID)
echo 'UUID=your-uuid-here /mnt/mediastore ext4 defaults,nofail 0 2' >> /etc/fstab

# Mount
systemctl daemon-reload
mount -a
```

## 4. Verify Mount

```bash
# Check if mounted
df -h /mnt/mediastore

# Test write access
touch /mnt/mediastore/test && rm /mnt/mediastore/test && echo "OK"
```

## 5. NFS Setup (for VM access)

```bash
# Install NFS server
apt install nfs-kernel-server

# Export the share (adjust subnet as needed)
echo '/mnt/mediastore 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)' >> /etc/exports

# Apply and start
exportfs -a
systemctl enable --now nfs-server
```

### Mounting from a VM

```bash
# On the VM
apt install nfs-common
mkdir -p /mnt/mediastore
mount <proxmox-ip>:/mnt/mediastore /mnt/mediastore

# For persistent mount, add to VM's /etc/fstab:
# <proxmox-ip>:/mnt/mediastore /mnt/mediastore nfs defaults 0 0
```

## 6. File Transfer (via Tailscale)

From laptop to Proxmox:

```bash
# Single file
scp /path/to/file root@<proxmox-tailscale-ip>:/mnt/mediastore/

# Directory
scp -r /path/to/folder root@<proxmox-tailscale-ip>:/mnt/mediastore/

# Large/resumable transfer
rsync -avP /path/to/file root@<proxmox-tailscale-ip>:/mnt/mediastore/
```

Get Proxmox Tailscale IP:
```bash
tailscale ip -4
```

## 7. (Optional) Mount Monitoring

USB drives can disconnect unexpectedly. Basic check:

```bash
mountpoint -q /mnt/mediastore || echo "Drive disconnected!"
```

### Cron-based monitoring

```bash
# Add to crontab (crontab -e)
*/5 * * * * mountpoint -q /mnt/mediastore || echo "mediastore disconnected at $(date)" >> /var/log/mount-monitor.log
```

### Systemd timer approach

Create `/etc/systemd/system/mount-monitor.service`:
```ini
[Unit]
Description=Check mediastore mount

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'mountpoint -q /mnt/mediastore || echo "mediastore disconnected at $(date)" >> /var/log/mount-monitor.log'
```

Create `/etc/systemd/system/mount-monitor.timer`:
```ini
[Unit]
Description=Run mount check every 5 minutes

[Timer]
OnBootSec=1min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

Enable:
```bash
systemctl enable --now mount-monitor.timer
```

## Notes

- `nofail` in fstab prevents boot failure if USB drive is disconnected
- UUID-based mounting ensures consistent mount regardless of device name (sdb, sdc, etc.)
- udev is already installed (part of systemd) - no additional setup needed
- NFS uses TCP/UDP port 2049