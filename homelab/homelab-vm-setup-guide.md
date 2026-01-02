# Homelab VM Setup Guide

## Part 1: Proxmox Cloud Image Setup

### Download Cloud Image (on Proxmox host)

```bash
cd /var/lib/vz/template/iso/
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2
```

### Create VM Shell

```bash
# Create VM (adjust VM_ID as needed)
VM_ID=101
VM_NAME=homelab

qm create $VM_ID --name $VM_NAME --memory 6144 --cores 3 --net0 virtio,bridge=vmbr0
```

### Import Cloud Image as Disk

```bash
qm importdisk $VM_ID /var/lib/vz/template/iso/debian-12-generic-amd64.qcow2 local-lvm
```

### Attach Disk and Configure Boot

```bash
# Attach the imported disk as scsi0
qm set $VM_ID --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-$VM_ID-disk-0

# Add cloud-init drive
qm set $VM_ID --ide2 local-lvm:cloudinit

# Set boot order
qm set $VM_ID --boot c --bootdisk scsi0

# Enable serial console (required for cloud-init)
qm set $VM_ID --serial0 socket --vga serial0
```

### Configure Cloud-Init (via UI or CLI)

```bash
qm set $VM_ID --ciuser youruser
qm set $VM_ID --cipassword yourpassword  # or use --sshkeys
qm set $VM_ID --ipconfig0 ip=dhcp
# Or static: --ipconfig0 ip=192.168.1.100/24,gw=192.168.1.1
```

### Resize Disk (optional)

```bash
qm resize $VM_ID scsi0 +50G
```

### Start VM

```bash
qm start $VM_ID
```

---

## Part 2: Ansible Automation Steps

### Inventory File (inventory.yml)

```yaml
all:
  hosts:
    homelab:
      ansible_host: <VM_IP_OR_TAILSCALE_IP>
      ansible_user: youruser
```

### Playbook Structure

```
homelab-ansible/
├── inventory.yml
├── playbook.yml
├── roles/
│   ├── base/
│   ├── tailscale/
│   ├── docker/
│   └── traefik/
└── group_vars/
    └── all.yml
```

### Tasks to Automate

#### 1. Base System Setup

- Update apt cache and upgrade packages
- Install essential packages: `curl`, `git`, `vim`, `htop`, `jq`
- Configure timezone
- Set hostname
- Create non-root user with sudo

#### 2. Tailscale Installation

```yaml
# roles/tailscale/tasks/main.yml
- name: Add Tailscale signing key
  ansible.builtin.shell: |
    curl -fsSL https://pkgs.tailscale.com/stable/debian/bookworm.noarmor.gpg | tee /usr/share/keyrings/tailscale-archive-keyring.gpg

- name: Add Tailscale repo
  ansible.builtin.apt_repository:
    repo: "deb [signed-by=/usr/share/keyrings/tailscale-archive-keyring.gpg] https://pkgs.tailscale.com/stable/debian bookworm main"
    state: present

- name: Install Tailscale
  ansible.builtin.apt:
    name: tailscale
    state: present
    update_cache: yes

- name: Start Tailscale service
  ansible.builtin.systemd:
    name: tailscaled
    state: started
    enabled: yes

# Note: `tailscale up` requires interactive auth or auth key
# Use: tailscale up --authkey=<key> for automation
```

#### 3. Docker Installation

```yaml
# roles/docker/tasks/main.yml
- name: Install Docker prerequisites
  ansible.builtin.apt:
    name:
      - ca-certificates
      - curl
      - gnupg
    state: present

- name: Add Docker GPG key
  ansible.builtin.shell: |
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    chmod a+r /etc/apt/keyrings/docker.gpg

- name: Add Docker repo
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian bookworm stable"
    state: present

- name: Install Docker
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present
    update_cache: yes

- name: Add user to docker group
  ansible.builtin.user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Start Docker service
  ansible.builtin.systemd:
    name: docker
    state: started
    enabled: yes
```

#### 4. Traefik Setup

```yaml
# roles/traefik/tasks/main.yml
- name: Create Traefik directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - /opt/traefik
    - /opt/traefik/config
    - /opt/traefik/certs

- name: Copy Traefik docker-compose
  ansible.builtin.template:
    src: docker-compose.yml.j2
    dest: /opt/traefik/docker-compose.yml

- name: Copy Traefik config
  ansible.builtin.template:
    src: traefik.yml.j2
    dest: /opt/traefik/config/traefik.yml

- name: Start Traefik
  community.docker.docker_compose_v2:
    project_src: /opt/traefik
    state: present
```

---

## Part 3: Traefik Configuration

### docker-compose.yml

```yaml
services:
  traefik:
    image: traefik:v3.2
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      - PORKBUN_API_KEY=${PORKBUN_API_KEY}
      - PORKBUN_SECRET_API_KEY=${PORKBUN_SECRET_API_KEY}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./config/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./certs:/certs
    networks:
      - traefik

networks:
  traefik:
    name: traefik
    external: false
```

### traefik.yml

```yaml
api:
  dashboard: true

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: traefik

certificatesResolvers:
  letsencrypt:
    acme:
      email: you@example.com
      storage: /certs/acme.json
      dnsChallenge:
        provider: porkbun
        delayBeforeCheck: 60
        resolvers:
          - "1.1.1.1:53"
          - "8.8.8.8:53"
```

### Example Service with Traefik Labels

```yaml
services:
  whoami:
    image: traefik/whoami
    container_name: whoami
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.yourdomain.com`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=letsencrypt"
    networks:
      - traefik

networks:
  traefik:
    external: true
```

---

## Part 4: Pi-hole DNS Setup

Add wildcard A record in Pi-hole:

1. Pi-hole Admin → Local DNS → DNS Records
2. Add: `*.yourdomain.com` → `100.x.x.x` (VM's Tailscale IP)

Note: Pi-hole doesn't support wildcard records natively. Instead, add individual records per service, or use dnsmasq config:

```bash
# /etc/dnsmasq.d/02-custom.conf
address=/yourdomain.com/100.x.x.x
```

This resolves `yourdomain.com` and all subdomains to the specified IP.

Restart Pi-hole DNS: `pihole restartdns`

---

## Quick Reference

| Step | Manual Command |
|------|----------------|
| SSH to VM | `ssh user@<tailscale-ip>` |
| Check Tailscale | `tailscale status` |
| Docker status | `docker ps` |
| Traefik logs | `docker logs traefik` |
| Renew certs | Automatic via Traefik |