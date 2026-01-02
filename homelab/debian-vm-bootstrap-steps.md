# Debian VM Bootstrap Steps

Manual steps performed after fresh Debian install on Proxmox VM.

## 1. Disable GUI, Boot to Console

```bash
systemctl set-default multi-user.target
```

## 2. Skip GRUB Menu on Boot

```bash
nano /etc/default/grub
```

Set:
```
GRUB_TIMEOUT=0
```

Apply:
```bash
update-grub
```

## 3. Install Essential Packages

```bash
apt update
apt install -y curl wget sudo git
```

## 4. Install and Configure SSH Server

```bash
apt install -y openssh-server
```

Enable root login (if needed):
```bash
nano /etc/ssh/sshd_config
```

Find and change:
```
PermitRootLogin yes
```

Restart SSH:
```bash
systemctl restart sshd
```

## 5. Install Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Authenticate via the URL provided.

## 6. (Optional) Create Non-Root User

```bash
useradd -m -s /bin/bash youruser
passwd youruser
usermod -aG sudo youruser
```

## 7. Connect to GitHub

Generate a deploy key (no passphrase for automation):
```bash
ssh-keygen -t ed25519 -C "homelab-vm-deploy" -f ~/.ssh/github_deploy -N ""
```

Display the public key and add it to your GitHub repo as a deploy key:
```bash
cat ~/.ssh/github_deploy.pub
```

Configure SSH to use the deploy key for GitHub:
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
git clone git@github.com:USERNAME/ops.git /root/stacks
```

---

## Ansible Equivalent

```yaml
---
- name: Bootstrap Debian VM
  hosts: all
  become: yes
  tasks:
    - name: Set default to multi-user target
      ansible.builtin.command: systemctl set-default multi-user.target

    - name: Set GRUB timeout to 0
      ansible.builtin.lineinfile:
        path: /etc/default/grub
        regexp: '^GRUB_TIMEOUT='
        line: 'GRUB_TIMEOUT=0'
      notify: Update GRUB

    - name: Install essential packages
      ansible.builtin.apt:
        name:
          - curl
          - wget
          - sudo
          - openssh-server
          - git
        state: present
        update_cache: yes

    - name: Enable root SSH login
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin yes'
      notify: Restart SSH

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
        dest: /root/stacks
        accept_hostkey: yes

  handlers:
    - name: Update GRUB
      ansible.builtin.command: update-grub

    - name: Restart SSH
      ansible.builtin.systemd:
        name: sshd
        state: restarted
```

Note: `tailscale up` requires interactive auth or an auth key. For full automation, generate an auth key in the Tailscale admin console and use:

```bash
tailscale up --authkey=tskey-auth-xxxxx
```

Note: The deploy key must be added to GitHub manually via the web UI (Settings → Deploy keys) or via the GitHub API with a PAT.
