# SIS4 LDOC
Scripts that run on SIS4
# === STEP 1: Create backups directory ===
sudo mkdir -p /var/backups
sudo chown root:root /var/backups
sudo chmod 755 /var/backups


# === STEP 2: Create Database Backup Script ===
sudo tee /usr/local/bin/backup_db.sh > /dev/null <<'SH'
#!/bin/bash
# backup_db.sh — backup PostgreSQL 'football' DB (plain SQL format)
OUTDIR="/var/backups"
TIMESTAMP=$(date +%F_%H-%M-%S)

# perform PostgreSQL dump
sudo -u postgres pg_dump -h localhost -p 5432 -U postgres football > "${OUTDIR}/football_${TIMESTAMP}.sql"

# optional: delete backups older than 30 days
find "${OUTDIR}" -name "football_*.sql" -type f -mtime +30 -delete
SH

# set execute permission
sudo chmod +x /usr/local/bin/backup_db.sh


# === STEP 3: Create Log Cleanup Script ===
sudo tee /usr/local/bin/cleanup_logs.sh > /dev/null <<'SH'
#!/bin/bash
# cleanup_logs.sh — truncate or remove old log files

LOGPATH="/var/log"

# Truncate .log files older than 7 days
find "$LOGPATH" -type f -name "*.log" -mtime +7 -exec sh -c '> "{}"' \;

# Remove compressed logs older than 30 days
find "$LOGPATH" -type f -name "*.gz" -mtime +30 -delete
SH

# set execute permission
sudo chmod +x /usr/local/bin/cleanup_logs.sh


# === STEP 4: Add tasks to CRON (root crontab) ===
sudo bash -c 'cat <<EOF >> /var/spool/cron/crontabs/root
# Daily DB backup at 02:00
0 2 * * * /usr/local/bin/backup_db.sh

# Weekly logs cleanup on Sunday at 03:00
0 3 * * 0 /usr/local/bin/cleanup_logs.sh
EOF'

# restart cron service
sudo systemctl restart cron
