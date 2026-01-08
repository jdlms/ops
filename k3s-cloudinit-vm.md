# Proxmox Cloud-Init Template Setup

## Overview

Cloud-init images are pre-built OS images designed for cloud/VM environments. Unlike ISOs (which require manual installation), you import them directly as a VM disk and configure via cloud-init on first boot.

## Download the Cloud Image

```bash
# Create a directory for cloud images (location doesn't matter, this is just a source file)
mkdir -p /var/template/cloud
cd /var/template/cloud

# Download Debian 12 genericcloud image
# - genericcloud: optimized for VMs (no bare metal drivers)
# - qcow2: QEMU copy-on-write format, sparse/compressed, snapshot-capable
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2
```

## Create the VM

```bash
# Create base VM structure
# - 9000: VM ID (high number convention for templates)
# - memory: RAM in MB
# - cores: vCPUs
# - net0: first NIC, virtio driver, attached to vmbr0 bridge
# IMPORTANT: Do NOT add a VLAN tag unless you specifically need it
#   Adding tag=XXX puts the VM on an isolated VLAN which likely has no DHCP
qm create 9000 --name debian-cloud --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
```

## Import and Attach the Disk

```bash
# Import qcow2 into Proxmox storage
# - Creates vm-9000-disk-0 in local-lvm
# - Adjust 'local-lvm' to match your storage (check with: pvesm status)
qm importdisk 9000 debian-12-genericcloud-amd64.qcow2 local-lvm

# Attach imported disk to VM
# - scsihw: use virtio-scsi-pci controller (best performance)
# - scsi0: first SCSI disk slot
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
```

## Configure Cloud-Init

```bash
# Add cloud-init drive
# - ide2: attaches as CD-ROM drive
# - Proxmox generates NoCloud datasource here (meta-data, user-data, network-config)
qm set 9000 --ide2 local-lvm:cloudinit
```

## Set Boot Options

```bash
# Configure boot order
# - boot c: boot from disk
# - bootdisk: specify which disk to boot from
qm set 9000 --boot c --bootdisk scsi0

# Add serial console
# - Required by some cloud images for console access
# - vga serial0: redirects display to serial
qm set 9000 --serial0 socket --vga serial0
```

## Resize the Disk

```bash
# Cloud images have minimal disk size (~2-3GB)
# Resize to something usable (20GB is plenty for learning)
qm resize 9000 scsi0 20G
```

## Enable QEMU Guest Agent

```bash
# Enable on VM config (Proxmox side)
# This lets you query VM IP from host with: qm guest cmd <vmid> network-get-interfaces
qm set 9000 --agent enabled=1
```

Note: You still need to install the agent inside the VM after first boot (see Post-Boot Setup).

## Set Default Cloud-Init Parameters

```bash
# Set user (don't use root - cloud images disable root SSH by default)
# Cloud-init grants passwordless sudo automatically
qm set 9000 --ciuser youruser

# Set SSH key for passwordless login
qm set 9000 --sshkeys /root/.ssh/authorized_keys

# Set password for console access (useful for troubleshooting if SSH fails)
qm set 9000 --cipassword yourpassword

# Set network - DHCP or static
qm set 9000 --ipconfig0 ip=dhcp
# Or static: qm set 9000 --ipconfig0 ip=192.168.0.50/24,gw=192.168.0.1
```

## Convert to Template

```bash
# Lock VM as template
# - Prevents accidental modification
# - Enables clone operations
qm template 9000

# Clean up source image (optional)
rm /var/template/cloud/debian-12-genericcloud-amd64.qcow2
```

## Usage: Clone and Configure

```bash
# Full clone from template
# - 101: new VM ID
# - --full: independent copy (not linked clone)
qm clone 9000 101 --name my-vm --full

# Override cloud-init parameters if needed (can also do via UI: VM > Cloud-Init tab)
qm set 101 --ipconfig0 ip=dhcp

# IMPORTANT: Set cloud-init params BEFORE first boot
# Cloud-init only runs once on first boot. If you boot with bad config,
# you'll need to delete and re-clone.

# Regenerate cloud-init image after changes
qm cloudinit update 101

# Start the VM
qm start 101
```

## Post-Boot Setup

### Install QEMU Guest Agent

```bash
# SSH into the VM, then:
sudo apt update && sudo apt install qemu-guest-agent
sudo systemctl enable --now qemu-guest-agent
```

Now you can query VM IP from Proxmox host:
```bash
qm guest cmd 101 network-get-interfaces
```

### Enable Tailscale SSH (Optional)

If using Tailscale for access:
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
```

The `--ssh` flag enables Tailscale SSH, letting you SSH via Tailscale identity without managing keys.

## Troubleshooting

### VM Stuck on "systemd-networkd-wait-online" for Minutes

**Cause:** No DHCP response. Usually one of:
1. VM has a VLAN tag isolating it from your network
2. No DHCP server on the network segment
3. Bridge misconfigured

**Diagnose:**
```bash
# Check for VLAN tag
qm config <vmid> | grep net
# If you see tag=XXX, that's likely the problem

# Check bridge setup
cat /etc/network/interfaces
brctl show
```

**Fix VLAN issue:**
```bash
qm set <vmid> --net0 virtio,bridge=vmbr0
qm reboot <vmid>
```

**Or use static IP instead of DHCP:**
```bash
qm set <vmid> --ipconfig0 ip=192.168.0.50/24,gw=192.168.0.1
qm cloudinit update <vmid>
qm reboot <vmid>
```

### Can't Login - Only Set SSH Key, Don't Know IP

**Find IP via MAC address:**
```bash
# Get MAC from config
qm config <vmid> | grep net

# Scan network
arp-scan --interface=vmbr0 --localnet | grep -i "<mac-prefix>"
```

**Or add temporary password for console access:**
```bash
qm set <vmid> --cipassword temppass123
qm cloudinit update <vmid>
qm reboot <vmid>
# Login via console, run: ip a
```

### Serial Console Shows Nothing

Hit Enter. Serial console waits for input before showing login prompt.

### Cloud-Init Config Not Applied

Cloud-init only runs on first boot. If you changed config after booting:
```bash
# Option 1: Regenerate and hope (sometimes works)
qm cloudinit update <vmid>
qm reboot <vmid>

# Option 2: Delete and re-clone (guaranteed)
qm destroy <vmid>
qm clone 9000 <vmid> --name my-vm --full
# Set cloud-init params BEFORE starting
qm start <vmid>
```

## k3s with Tailscale (Optional)

If running k3s and want cluster traffic over Tailscale:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--node-ip=<tailscale-ip> --flannel-iface=tailscale0" sh -
```

This is useful for agents on different networks. For local-only clusters on the same Proxmox host, just use LAN IPs.

## Quick Reference

```bash
# List VMs
qm list

# Show VM config
qm config <vmid>

# Check storage
pvesm status

# Check thin pool usage
lvs -a

# Get VM IP (requires guest agent)
qm guest cmd <vmid> network-get-interfaces

# Console into VM
qm terminal <vmid>
```