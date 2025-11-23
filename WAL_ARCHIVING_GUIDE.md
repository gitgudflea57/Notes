# PostgreSQL WAL Archiving & Point-in-Time Recovery Guide

Complete guide for setting up WAL (Write-Ahead Logging) archiving and performing Point-in-Time Recovery (PITR) for PostgreSQL databases.

---

## Table of Contents

1. [What is WAL?](#what-is-wal)
2. [Setting Up WAL Archiving](#setting-up-wal-archiving)
3. [Creating Base Backups](#creating-base-backups)
4. [Point-in-Time Recovery (PITR)](#point-in-time-recovery-pitr)
5. [Multiple PostgreSQL Instances](#multiple-postgresql-instances)
6. [Monitoring & Testing](#monitoring--testing)
7. [Troubleshooting](#troubleshooting)

---

## What is WAL?

**Write-Ahead Logging (WAL)** is PostgreSQL's transaction log that records every change to the database **before** it's written to data files.

### How WAL Works

```
1. Transaction: INSERT INTO table VALUES (...)
   ↓
2. Written to WAL first  [SAFE - Durable]
   ↓
3. Written to data files later [Can be delayed]
```

### WAL = Time Machine for Your Database

- **Base Backup** = Snapshot at time T₀
- **WAL Files** = Recording of every change after T₀
- **Recovery** = Restore backup + replay WAL to any point in time

```
Timeline:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━→
Sunday 00:00          Tuesday 14:29:59   Tuesday 14:30:00
    ↓                       ↓                  ↓
[Base Backup]         [Good State]        [Disaster! 😱]
    │                       │                  │
    │←────── WAL Files ─────┤                  │
    └───────────────────────┴──────────────────┘
                    Replay WAL to recover
```

---

## Setting Up WAL Archiving

### Step 1: Create WAL Archive Directory

```bash
# Create directory for archived WAL files
sudo mkdir -p /backup/wal
sudo chown postgres:postgres /backup/wal
sudo chmod 700 /backup/wal
```

### Step 2: Configure PostgreSQL

Edit `/etc/postgresql/15/main/postgresql.conf` (or your version's path):

```ini
#------------------------------------------
# WAL ARCHIVING CONFIGURATION
#------------------------------------------

# Generate enough WAL information for recovery
wal_level = replica

# Enable WAL archiving
archive_mode = on

# Command to archive completed WAL files
# %f = WAL filename (e.g., 000000010000000000000001)
# %p = Full path to WAL file in pg_wal/
archive_command = 'test ! -f /backup/wal/%f && cp %p /backup/wal/%f'

# Force WAL file switch every 5 minutes (even if not full)
archive_timeout = 300

# Optional: Adjust WAL size for better performance
# wal_segment_size is compile-time, but you can tune:
max_wal_size = 2GB
min_wal_size = 1GB
```

**Archive Command Explained:**
```bash
test ! -f /backup/wal/%f  # Check file doesn't already exist
&&                         # If check passed, then...
cp %p /backup/wal/%f      # Copy WAL file to archive
```

### Step 3: Restart PostgreSQL

```bash
# Restart to apply configuration
sudo systemctl restart postgresql

# Verify PostgreSQL is running
sudo systemctl status postgresql
```

### Step 4: Verify WAL Archiving

```bash
# Force a WAL file switch to test archiving
sudo -u postgres psql -c "SELECT pg_switch_wal();"

# Wait a moment, then check for archived files
ls -lh /backup/wal/

# You should see files like:
# 000000010000000000000001
# 000000010000000000000002
```

### Step 5: Monitor Archive Status

```sql
-- Connect to PostgreSQL
psql -U postgres

-- Check archiving status
SELECT * FROM pg_stat_archiver;

-- Output should show:
-- archived_count | last_archived_time        | failed_count
-- 5              | 2025-11-21 15:30:00       | 0
```

---

## Creating Base Backups

Base backups are full snapshots of your database cluster. Combined with WAL files, they enable PITR.

### Method 1: pg_basebackup (Recommended)

```bash
# Create base backup
sudo -u postgres pg_basebackup \
    -D /backup/base_$(date +%Y%m%d_%H%M%S) \
    -F tar \
    -z \
    -X stream \
    -P

# Flags explained:
# -D = Destination directory
# -F tar = Output as tar archive
# -z = Compress with gzip
# -X stream = Include WAL files needed for consistency
# -P = Show progress
```

### Method 2: Automated Weekly Backups

Create `/usr/local/bin/weekly_backup.sh`:

```bash
#!/bin/bash
#
# Weekly base backup script
#

BACKUP_DIR="/backup/postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_WEEKS=4

# Create base backup
echo "Starting base backup at $(date)"
sudo -u postgres pg_basebackup \
    -D "${BACKUP_DIR}/base_${TIMESTAMP}" \
    -F tar \
    -z \
    -X stream \
    -P

if [ $? -eq 0 ]; then
    echo "✓ Backup completed: base_${TIMESTAMP}"
else
    echo "✗ Backup failed!"
    exit 1
fi

# Remove backups older than retention period
find "${BACKUP_DIR}" -name "base_*" -type d -mtime +$((RETENTION_WEEKS * 7)) -exec rm -rf {} \;

echo "Backup completed at $(date)"
```

Make it executable and schedule:

```bash
# Make executable
chmod +x /usr/local/bin/weekly_backup.sh

# Add to crontab (run every Sunday at 2 AM)
sudo crontab -e
# Add:
0 2 * * 0 /usr/local/bin/weekly_backup.sh >> /var/log/postgres_backup.log 2>&1
```

### Verify Base Backup

```bash
# List backup contents
tar -tzf /backup/base_20251121/base.tar.gz | head -20

# Verify backup integrity
tar -tzf /backup/base_20251121/base.tar.gz > /dev/null
echo $?  # Should be 0 for success
```

---

## Point-in-Time Recovery (PITR)

### Scenario: Disaster Recovery

**Problem:** Someone dropped a critical table at 14:30:00 on Tuesday.

**Goal:** Recover database to 14:29:59 (1 second before disaster).

### Prerequisites

- Base backup from before the disaster
- Archived WAL files covering the time period

### Recovery Procedure

#### Step 1: Stop PostgreSQL

```bash
sudo systemctl stop postgresql

# Verify it's stopped
sudo systemctl status postgresql
```

#### Step 2: Backup Current State (Optional but Recommended)

```bash
# Move current data directory to safe location
sudo mv /var/lib/postgres/data /var/lib/postgres/data_broken_$(date +%Y%m%d)

# Create fresh directory
sudo mkdir /var/lib/postgres/data
sudo chown postgres:postgres /var/lib/postgres/data
sudo chmod 700 /var/lib/postgres/data
```

#### Step 3: Restore Base Backup

```bash
# Extract base backup
cd /var/lib/postgres/data
sudo -u postgres tar -xzf /backup/base_20251117/base.tar.gz

# Extract pg_wal if it was backed up separately
sudo -u postgres tar -xzf /backup/base_20251117/pg_wal.tar.gz
```

#### Step 4: Configure Recovery

Create recovery signal file:

```bash
sudo -u postgres touch /var/lib/postgres/data/recovery.signal
```

Edit `/var/lib/postgres/data/postgresql.auto.conf`:

```ini
# Tell PostgreSQL where to find archived WAL files
restore_command = 'cp /backup/wal/%f %p'

# Target recovery time (1 second before disaster)
recovery_target_time = '2025-11-21 14:29:59'

# What to do after reaching target
recovery_target_action = 'promote'

# Optional: Recovery target name (for testing)
# recovery_target_name = 'before_disaster'

# Optional: Timeline to recover (usually default is fine)
# recovery_target_timeline = 'latest'
```

**Recovery Options:**

```ini
# Recover to specific time
recovery_target_time = '2025-11-21 14:29:59'

# Recover to specific transaction ID
recovery_target_xid = '123456'

# Recover to end of specific WAL file
recovery_target_lsn = '0/5000000'

# Recover to named restore point
recovery_target_name = 'before_migration'

# What to do after recovery:
# 'promote' = Normal operation (default)
# 'pause' = Pause for inspection
# 'shutdown' = Stop server
recovery_target_action = 'promote'
```

#### Step 5: Start PostgreSQL

```bash
# Start PostgreSQL - it will enter recovery mode
sudo systemctl start postgresql

# Watch the recovery progress
sudo tail -f /var/log/postgresql/postgresql-15-main.log
```

**What Happens:**

```
PostgreSQL recovery log:

LOG:  starting point-in-time recovery to 2025-11-21 14:29:59
LOG:  restored log file "000000010000000000000001" from archive
LOG:  redo starts at 0/1000028
LOG:  restored log file "000000010000000000000002" from archive
...
LOG:  recovery stopping before commit of transaction 123458
LOG:  recovery has paused
LOG:  redo done at 0/500014C
LOG:  selected new timeline ID: 2
LOG:  archive recovery complete
LOG:  database system is ready to accept connections
```

#### Step 6: Verify Recovery

```bash
# Connect to database
psql -U postgres -d vessel_analytics

# Check data is at correct time
SELECT NOW();
SELECT COUNT(*) FROM vessel_characteristics;
SELECT MAX(status_date_time) FROM raw_vessel_events;

# Verify the dropped table exists
\dt vessel_characteristics
```

#### Step 7: Resume Normal Operations

```bash
# If recovery was successful, remove recovery files
sudo rm /var/lib/postgres/data/recovery.signal
sudo rm /var/lib/postgres/data/recovery.conf  # If it exists

# The database is now in normal operation
# WAL archiving will resume automatically
```

---

## Multiple PostgreSQL Instances

Run separate PostgreSQL instances for complete isolation of different databases.

### Why Multiple Instances?

- **Separate WAL archiving** per instance
- **Different backup schedules**
- **Resource isolation**
- **Independent upgrades**
- **Security isolation**

### Setup Second Instance

#### Step 1: Create Data Directory

```bash
# Create directory for second instance
sudo mkdir -p /var/lib/postgres/vessel_data
sudo chown postgres:postgres /var/lib/postgres/vessel_data
sudo chmod 700 /var/lib/postgres/vessel_data
```

#### Step 2: Initialize Cluster

```bash
# Initialize new PostgreSQL cluster
sudo -u postgres initdb -D /var/lib/postgres/vessel_data

# Output:
# Success. You can now start the database server using:
#     pg_ctl -D /var/lib/postgres/vessel_data -l logfile start
```

#### Step 3: Configure Instance

Edit `/var/lib/postgres/vessel_data/postgresql.conf`:

```ini
# Different port from main instance
port = 5433

# WAL archiving for this instance
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backup/vessel_wal/%f && cp %p /backup/vessel_wal/%f'
archive_timeout = 300

# Instance-specific tuning
shared_buffers = 4GB
work_mem = 256MB
maintenance_work_mem = 1GB
```

#### Step 4: Create Systemd Service

Create `/etc/systemd/system/postgresql@vessel.service`:

```ini
[Unit]
Description=PostgreSQL Vessel Analytics Instance
After=network.target

[Service]
Type=forking
User=postgres

# Paths to PostgreSQL binaries (adjust for your version)
ExecStart=/usr/lib/postgresql/15/bin/pg_ctl start -D /var/lib/postgres/vessel_data -l /var/log/postgresql/vessel.log
ExecStop=/usr/lib/postgresql/15/bin/pg_ctl stop -D /var/lib/postgres/vessel_data
ExecReload=/usr/lib/postgresql/15/bin/pg_ctl reload -D /var/lib/postgres/vessel_data

Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

#### Step 5: Start Instance

```bash
# Reload systemd
sudo systemctl daemon-reload

# Start vessel instance
sudo systemctl start postgresql@vessel

# Enable auto-start on boot
sudo systemctl enable postgresql@vessel

# Verify both instances running
sudo systemctl status postgresql         # Main (port 5432)
sudo systemctl status postgresql@vessel  # Vessel (port 5433)
```

#### Step 6: Create Database

```bash
# Connect to vessel instance
psql -U postgres -p 5433

# Create database
CREATE DATABASE vessel_analytics;

# Create user
CREATE USER gitgud WITH PASSWORD 'your_password';
GRANT ALL PRIVILEGES ON DATABASE vessel_analytics TO gitgud;

# Exit
\q
```

#### Step 7: Update Application

Update your application connection parameters:

```python
# load_data.py
DB_PARAMS = {
    "dbname": "vessel_analytics",
    "user": "gitgud",
    "host": "localhost",
    "port": 5433  # Changed from 5432
}
```

### Managing Multiple Instances

```bash
# Check all PostgreSQL processes
ps aux | grep postgres

# Check listening ports
sudo netstat -tlnp | grep postgres
# tcp  0.0.0.0:5432  LISTEN  postgres (main)
# tcp  0.0.0.0:5433  LISTEN  postgres (vessel)

# Connect to specific instance
psql -U gitgud -d vessel_analytics -p 5432  # Main
psql -U gitgud -d vessel_analytics -p 5433  # Vessel

# Start/stop specific instance
sudo systemctl start postgresql          # Main
sudo systemctl start postgresql@vessel   # Vessel

# View logs
sudo tail -f /var/log/postgresql/postgresql-15-main.log  # Main
sudo tail -f /var/log/postgresql/vessel.log              # Vessel
```

---

## Monitoring & Testing

### Monitor WAL Archiving

```sql
-- Check archive status
SELECT * FROM pg_stat_archiver;

-- Check current WAL file and location
SELECT pg_current_wal_lsn(), pg_current_wal_insert_lsn();

-- Check WAL file size
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') / (1024*1024) AS wal_mb;

-- Force WAL switch (for testing)
SELECT pg_switch_wal();
```

### Test Archive Command

```bash
# Check if WAL files are being archived
ls -lh /backup/wal/

# Count archived files
ls /backup/wal/ | wc -l

# Check WAL file age (should be recent)
ls -lht /backup/wal/ | head -5

# Verify archive command works
sudo -u postgres sh -c 'test ! -f /backup/wal/test_file && cp /tmp/test /backup/wal/test_file'
echo $?  # Should be 0
```

### Monthly Recovery Test

Create `/usr/local/bin/test_recovery.sh`:

```bash
#!/bin/bash
#
# Monthly recovery test - verifies backups work
#

TEST_DB="vessel_analytics_recovery_test"
LATEST_BACKUP=$(ls -td /backup/base_* | head -1)

echo "Testing recovery from: $LATEST_BACKUP"

# Stop test database if exists
psql -U postgres -p 5434 -c "DROP DATABASE IF EXISTS $TEST_DB;" 2>/dev/null

# Initialize test instance (port 5434)
if [ ! -d /var/lib/postgres/test_data ]; then
    initdb -D /var/lib/postgres/test_data
    echo "port = 5434" >> /var/lib/postgres/test_data/postgresql.conf
fi

# Restore backup to test instance
pg_ctl -D /var/lib/postgres/test_data start
sleep 2

# Create database and restore
psql -U postgres -p 5434 -c "CREATE DATABASE $TEST_DB;"
pg_restore -U postgres -p 5434 -d $TEST_DB "$LATEST_BACKUP/base.tar.gz"

# Verify data
ROW_COUNT=$(psql -U postgres -p 5434 -d $TEST_DB -t -c "SELECT COUNT(*) FROM raw_vessel_events;")

if [ $ROW_COUNT -gt 0 ]; then
    echo "✓ Recovery test PASSED: $ROW_COUNT rows restored"

    # Cleanup
    psql -U postgres -p 5434 -c "DROP DATABASE $TEST_DB;"
    pg_ctl -D /var/lib/postgres/test_data stop

    exit 0
else
    echo "✗ Recovery test FAILED: No data restored"
    exit 1
fi
```

Schedule monthly:

```bash
# Add to crontab (1st of each month at 3 AM)
0 3 1 * * /usr/local/bin/test_recovery.sh >> /var/log/recovery_test.log 2>&1
```

---

## Troubleshooting

### Archive Command Failing

**Symptoms:**
- `pg_stat_archiver` shows `failed_count > 0`
- WAL files not appearing in `/backup/wal/`

**Solutions:**

```bash
# 1. Check archive directory permissions
ls -ld /backup/wal/
# Should be: drwx------ postgres postgres

# 2. Test archive command manually
sudo -u postgres sh -c 'cp /var/lib/postgres/data/pg_wal/000000010000000000000001 /backup/wal/'

# 3. Check disk space
df -h /backup

# 4. View PostgreSQL logs for errors
sudo tail -f /var/log/postgresql/postgresql-15-main.log | grep -i archive

# 5. Simplify archive command for debugging
# In postgresql.conf, temporarily change to:
archive_command = 'cp %p /backup/wal/%f 2>> /var/log/archive.log'
# Then check: tail -f /var/log/archive.log
```

### Recovery Stops Early

**Symptoms:**
- Recovery completes but data is missing
- Recovery stops before target time

**Solutions:**

```bash
# 1. Check if all required WAL files are available
ls /backup/wal/ | sort

# 2. Check recovery target is correct
cat /var/lib/postgres/data/postgresql.auto.conf | grep recovery_target

# 3. Check PostgreSQL logs for recovery messages
sudo grep -i recovery /var/log/postgresql/postgresql-15-main.log

# 4. Verify restore_command works
sudo -u postgres sh -c 'cp /backup/wal/000000010000000000000001 /tmp/test_restore'
```

### "Could not open file" During Recovery

**Problem:** `restore_command` can't find WAL files

**Solution:**

```bash
# Check restore_command has correct path
cat /var/lib/postgres/data/postgresql.auto.conf | grep restore_command

# Verify WAL files exist
ls -lh /backup/wal/

# Test restore command manually
sudo -u postgres sh -c 'cp /backup/wal/000000010000000000000001 /var/lib/postgres/data/pg_wal/test'
```

### WAL Disk Space Filling Up

**Problem:** `/backup/wal/` growing too large

**Solution:**

```bash
# 1. Check current WAL archive size
du -sh /backup/wal/

# 2. Clean old WAL files (after base backup)
# Only delete WAL files older than your oldest base backup!

# 3. Automate cleanup with script
#!/bin/bash
# Keep WAL files for last 30 days
find /backup/wal/ -name "0*" -type f -mtime +30 -delete

# 4. Add to crontab (daily at 4 AM)
0 4 * * * find /backup/wal/ -name "0*" -type f -mtime +30 -delete
```

---

## Summary Checklist

### Initial Setup
- [ ] Create WAL archive directory (`/backup/wal/`)
- [ ] Configure `postgresql.conf` (wal_level, archive_mode, archive_command)
- [ ] Restart PostgreSQL
- [ ] Verify WAL archiving working (`pg_stat_archiver`)
- [ ] Create first base backup (`pg_basebackup`)

### Regular Operations
- [ ] Weekly base backups (automated with cron)
- [ ] Monitor archive status (`SELECT * FROM pg_stat_archiver;`)
- [ ] Monitor disk space (`df -h /backup`)
- [ ] Monthly recovery tests
- [ ] Clean old WAL files (retain based on backup retention)

### Disaster Recovery
- [ ] Identify recovery target time
- [ ] Stop PostgreSQL
- [ ] Backup current state (optional)
- [ ] Restore base backup
- [ ] Create `recovery.signal`
- [ ] Configure `postgresql.auto.conf` with recovery targets
- [ ] Start PostgreSQL (enters recovery mode)
- [ ] Monitor recovery in logs
- [ ] Verify data after recovery
- [ ] Remove recovery configuration files

---

## Quick Reference Commands

```bash
# Enable WAL archiving
echo "wal_level = replica" >> /etc/postgresql/15/main/postgresql.conf
echo "archive_mode = on" >> /etc/postgresql/15/main/postgresql.conf
echo "archive_command = 'test ! -f /backup/wal/%f && cp %p /backup/wal/%f'" >> /etc/postgresql/15/main/postgresql.conf

# Create base backup
pg_basebackup -D /backup/base_$(date +%Y%m%d) -F tar -z -X stream -P

# Force WAL switch (testing)
psql -c "SELECT pg_switch_wal();"

# Check archive status
psql -c "SELECT * FROM pg_stat_archiver;"

# Recovery configuration
touch /var/lib/postgres/data/recovery.signal
echo "restore_command = 'cp /backup/wal/%f %p'" >> /var/lib/postgres/data/postgresql.auto.conf
echo "recovery_target_time = '2025-11-21 14:29:59'" >> /var/lib/postgres/data/postgresql.auto.conf

# View recovery progress
tail -f /var/log/postgresql/postgresql-15-main.log | grep -i recovery
```

---

## Additional Resources

- **PostgreSQL WAL Documentation:** https://www.postgresql.org/docs/current/wal-intro.html
- **Backup & Recovery Guide:** https://www.postgresql.org/docs/current/backup.html
- **pg_basebackup Manual:** https://www.postgresql.org/docs/current/app-pgbasebackup.html

---

**Created:** 2025-11-21
**PostgreSQL Version:** 15.x
**Last Updated:** 2025-11-21
