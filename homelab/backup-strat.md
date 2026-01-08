# Docker Backup Strategy

Local disk for quick restores, S3 for disaster recovery.

## Part 1: Add Second Disk in Proxmox

1. Proxmox UI → Select VM → Hardware → Add → Hard Disk
2. Storage: local-lvm
3. Size: 20-50GB (adjust based on needs)
4. Leave defaults, click Add

## Part 2: Mount Disk in VM

```bash
# Find the new disk
lsblk

# Create filesystem (assuming /dev/sdb)
mkfs.ext4 /dev/sdb

# Create mount point
mkdir /mnt/backups

# Mount
mount /dev/sdb /mnt/backups

# Make persistent
echo '/dev/sdb /mnt/backups ext4 defaults 0 2' >> /etc/fstab
```

## Part 3: Configure rclone for S3

```bash
apt install rclone -y
rclone config
```

Follow prompts:
1. `n` for new remote
2. Name: `s3backup`
3. Type: `s3`
4. Provider: your choice (AWS, Backblaze B2, etc.)
5. Enter access key and secret
6. Region and endpoint as needed
7. Leave other defaults

Test:
```bash
rclone ls s3backup:your-bucket-name
```

## Part 4: Backup Script

Create `/root/backup.sh`:

```bash
#!/bin/bash
set -e

BACKUP_DIR="/mnt/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DOCKER_DATA="/opt"  # adjust to your compose/volumes location
S3_BUCKET="your-bucket-name"
S3_PATH="homelab"

echo "Starting backup at $(date)"

# Optional: stop containers for consistent backup
# docker compose -f /opt/traefik/docker-compose.yml down

# Tar up docker data
tar -czf "$BACKUP_DIR/docker_$TIMESTAMP.tar.gz" -C "$DOCKER_DATA" .

echo "Local backup created: docker_$TIMESTAMP.tar.gz"

# Sync to S3
rclone sync "$BACKUP_DIR" "s3backup:$S3_BUCKET/$S3_PATH"

echo "Synced to S3"

# Keep only last 7 local backups
ls -t "$BACKUP_DIR"/docker_*.tar.gz | tail -n +8 | xargs -r rm

echo "Cleaned old local backups"

# Optional: restart containers
# docker compose -f /opt/traefik/docker-compose.yml up -d

echo "Backup complete at $(date)"
```

Make executable:
```bash
chmod +x /root/backup.sh
```

## Part 5: Schedule with Cron

```bash
crontab -e
```

Add (runs daily at 3am):
```
0 3 * * * /root/backup.sh >> /var/log/backup.log 2>&1
```

## Part 6: Test Restore

From local:
```bash
cd /opt
tar -xzf /mnt/backups/docker_YYYYMMDD_HHMMSS.tar.gz
```

From S3:
```bash
rclone copy s3backup:your-bucket-name/homelab/docker_YYYYMMDD_HHMMSS.tar.gz /mnt/backups/
tar -xzf /mnt/backups/docker_YYYYMMDD_HHMMSS.tar.gz -C /opt
```

## What's Backed Up

| Item | Method |
|------|--------|
| Compose files | Git + this backup |
| Bind mounts/configs | This backup |
| Named volumes | This backup (if under /opt) |
| Container images | Not backed up (pulled from registry) |

## Notes

- If using named volumes in `/var/lib/docker/volumes/`, adjust `DOCKER_DATA` or add a second tar command
- For databases, consider dumping before backup (`docker exec db pg_dump...`)
- Test restore process periodically

## Future Consideration: Separate Docker Storage

Current setup uses a simple layout for a single-VM homelab on limited hardware:
- Main disk: OS + Docker engine + all data
- Backup disk: Compressed backup tarballs only

If scaling to more machines or adding dedicated storage:

**Alternative layout**:
- Main disk (small, 20-50GB): OS + Docker engine only
- Data disk (large): Mount at `/var/lib/docker` — all images, volumes, container data
- Backup disk: Compressed tarballs or snapshots

Benefits of separation:
- Docker filling up doesn't kill the OS
- Easier to resize/migrate Docker storage independently
- Can use faster storage (SSD) for Docker, slower (HDD) for backups
- Cleaner Proxmox snapshots (exclude data disk)

To implement later:
1. Add new disk in Proxmox
2. Stop Docker: `systemctl stop docker`
3. Move data: `mv /var/lib/docker /var/lib/docker.bak`
4. Mount new disk at `/var/lib/docker`
5. Copy data back: `cp -a /var/lib/docker.bak/* /var/lib/docker/`
6. Start Docker: `systemctl start docker`
7. Verify, then remove backup: `rm -rf /var/lib/docker.bak`