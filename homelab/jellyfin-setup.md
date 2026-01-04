# Jellyfin LXC Setup on Proxmox (with iGPU Passthrough)

## Prerequisites

- Proxmox host with Intel iGPU (e.g., N95 with Quick Sync)
- Media stored on host at `/mnt/mediastore`

## Step 1: Download Debian Template

Proxmox UI → Node → local storage → CT Templates → Templates button → download `debian-12-standard`

## Step 2: Create the LXC Container

Proxmox UI → Create CT

- Hostname: `jellyfin`
- Template: debian-12-standard
- Disk: 8GB
- CPU: 2 cores
- Memory: 2048MB
- Network: DHCP on vmbr0

Leave "Unprivileged container" checked. Create but don't start.

## Step 3: Create Media User on Host (for UID Mapping)

Unprivileged LXC containers map UIDs for security - container root (UID 0) becomes UID 100000+ on the host. This means container processes can't access files owned by regular host users.

To allow the container to access media files while staying unprivileged, we create a dedicated user and map that specific UID into the container.

On the Proxmox host:

```bash
useradd -u 1100 -M -s /usr/sbin/nologin jellyfin
chown -R 1100:1100 /mnt/mediastore
```

Then allow the container to use this UID. Add to `/etc/subuid`:

```bash
echo "root:1100:1" >> /etc/subuid
```

And `/etc/subgid`:

```bash
echo "root:1100:1" >> /etc/subgid
```

## Step 4: Configure iGPU Passthrough, Media Mount, and UID Mapping

SSH into Proxmox host:

```bash
nano /etc/pve/lxc/100.conf
```

Add these lines at the bottom:

```
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri dev/dri none bind,optional,create=dir
mp0: /mnt/mediastore,mp=/media

# UID/GID mapping: pass through UID 1100 for media access
lxc.idmap: u 0 100000 1100
lxc.idmap: g 0 100000 1100
lxc.idmap: u 1100 1100 1
lxc.idmap: g 1100 1100 1
lxc.idmap: u 1101 101101 64435
lxc.idmap: g 1101 101101 64435
```

**What this mapping does:**
- UIDs 0-1099 in container → UIDs 100000-101099 on host (isolated)
- UID 1100 in container → UID 1100 on host (direct passthrough for media access)
- UIDs 1101-65535 in container → UIDs 101101-165535 on host (isolated)

## Step 4: Start Container and Fix Networking (if needed)

```bash
pct start 100
pct enter 100
```

If no IP on eth0, add network config:

```bash
cat >> /etc/network/interfaces << EOF
auto eth0
iface eth0 inet dhcp
EOF

systemctl restart networking
```

Verify connectivity:

```bash
ping -c 3 8.8.8.8
```

## Step 5: Install Jellyfin

```bash
apt update && apt install -y curl gnupg
curl -fsSL https://repo.jellyfin.org/install-debuntu.sh | bash
```

## Step 6: Access Jellyfin

Open `http://<container-ip>:8096` in browser.

Media is available inside the container at `/media`.

## Step 7: Enable Hardware Transcoding

In Jellyfin dashboard:

Dashboard → Playback → Transcoding → Hardware acceleration → Intel QuickSync (QSV)

## Notes

- Container IP can be found with `ip a` inside the container
- Jellyfin service starts automatically on boot
- For remote access, install Tailscale inside the container