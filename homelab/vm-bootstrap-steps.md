# Debian VM Bootstrap Steps

Using Debian cloud images with cloud-init on Proxmox. Cloud images are pre-built minimal images that boot directly to a running system - no installer, no interaction needed.

## Why Cloud Images?

| Aspect | Standard ISO | Cloud Image |
|--------|-------------|-------------|
| Setup time per VM | 10-15 min | ~30 seconds |
| Repeatability | Manual each time | Clone from template |
| Customization | Full control during install | Post-boot or baked into template |
| Size | ~600MB+ | ~250-350MB |

Cloud images align with the DevOps mindset - treat VMs as cattle, not pets.

---

## One-Time: Create Proxmox Template

Run these commands on your Proxmox host.

### 1. Download Debian Cloud Image

```bash
cd /var/lib/vz/template/iso
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2
```

### 2. Create VM and Import Disk

```bash
# Create VM (adjust ID as needed)
qm create 9000 --name debian-cloud-template --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0

# Import the cloud image as the VM's disk
qm importdisk 9000 debian-12-genericcloud-amd64.qcow2 local-lvm

# Attach the disk to the VM
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0

# Set boot order
qm set 9000 --boot c --bootdisk scsi0

# Add cloud-init drive
qm set 9000 --ide2 local-lvm:cloudinit

# Enable serial console (required for cloud images)
qm set 9000 --serial0 socket --vga serial0

# Enable QEMU guest agent
qm set 9000 --agent enabled=1
```

### 3. Configure Cloud-Init Defaults

```bash
# Set default user and SSH key
qm set 9000 --ciuser youruser
qm set 9000 --sshkeys ~/.ssh/authorized_keys
qm set 9000 --ipconfig0 ip=dhcp
```

Or configure via Proxmox UI: VM → Cloud-Init tab.

### 4. Convert to Template

```bash
qm template 9000
```

---

## Deploying a New VM

### Clone from Template

```bash
# Full clone (independent copy)
qm clone 9000 101 --name my-new-vm --full

# Or linked clone (faster, shares base disk)
qm clone 9000 101 --name my-new-vm
```

### Customize Cloud-Init for This VM

```bash
qm set 101 --ipconfig0 ip=192.168.1.101/24,gw=192.168.1.1
qm set 101 --nameserver 1.1.1.1
qm set 101 --searchdomain home.local
```

### Start the VM

```bash
qm start 101
```

The VM boots in ~30 seconds with your user, SSH key, and network configured.

---

## Post-Boot Configuration

After the VM boots, SSH in and run any additional setup.

### 1. Install Additional Packages

```bash
apt update
apt install -y curl wget git qemu-guest-agent
systemctl enable --now qemu-guest-agent
```

### 2. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --authkey=tskey-auth-xxxxx  # Use auth key for automation
```

### 3. Connect to GitHub

Generate a deploy key:
```bash
ssh-keygen -t ed25519 -C "homelab-vm-deploy" -f ~/.ssh/github_deploy -N ""
cat ~/.ssh/github_deploy.pub
```

Add the public key to GitHub (Settings → Deploy keys), then configure SSH:
```bash
cat >> ~/.ssh/config << 'EOF'
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_deploy
    IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
```

Clone the repo:
```bash
git clone git@github.com:USERNAME/ops.git ~/stacks
```

---

## Advanced: Custom Cloud-Init User Data

For more complex bootstrapping, create a custom cloud-init config:

```yaml
#cloud-config
package_update: true
packages:
  - curl
  - wget
  - git
  - qemu-guest-agent

runcmd:
  - systemctl enable --now qemu-guest-agent
  - curl -fsSL https://tailscale.com/install.sh | sh
  # Add more bootstrap commands here

write_files:
  - path: /root/.ssh/config
    permissions: '0600'
    content: |
      Host github.com
          HostName github.com
          User git
          IdentityFile ~/.ssh/github_deploy
          IdentitiesOnly yes
```

Save this as a snippet in Proxmox and reference it:
```bash
qm set 101 --cicustom "user=local:snippets/my-cloud-init.yaml"
```

---

## Ansible Playbook (Post-Boot)

For VMs created from the cloud template:

```yaml
---
- name: Configure Debian Cloud VM
  hosts: all
  become: yes
  tasks:
    - name: Install packages
      ansible.builtin.apt:
        name:
          - curl
          - wget
          - git
          - qemu-guest-agent
        state: present
        update_cache: yes

    - name: Enable QEMU guest agent
      ansible.builtin.systemd:
        name: qemu-guest-agent
        enabled: yes
        state: started

    - name: Install Tailscale
      ansible.builtin.shell: curl -fsSL https://tailscale.com/install.sh | sh
      args:
        creates: /usr/bin/tailscale

    - name: Generate GitHub deploy key
      community.crypto.openssh_keypair:
        path: ~/.ssh/github_deploy
        type: ed25519
        comment: "homelab-vm-deploy"

    - name: Configure SSH for GitHub
      ansible.builtin.copy:
        dest: ~/.ssh/config
        mode: '0600'
        content: |
          Host github.com
              HostName github.com
              User git
              IdentityFile ~/.ssh/github_deploy
              IdentitiesOnly yes

    - name: Clone ops repo
      ansible.builtin.git:
        repo: git@github.com:USERNAME/ops.git
        dest: ~/stacks
        accept_hostkey: yes
```

---

## Notes

- **Tailscale**: For full automation, generate an auth key in the Tailscale admin console
- **GitHub deploy key**: Must be added manually via GitHub UI or API
- **Template updates**: Periodically download fresh cloud images and recreate the template
- **Disk resize**: Cloud images have small disks by default. Resize after cloning:
  ```bash
  qm resize 101 scsi0 +20G
  ```
