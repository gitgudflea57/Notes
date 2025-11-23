# Production and Development PostgreSQL Setup Guide

Complete guide for setting up isolated production and development PostgreSQL instances with systemd on Linux.

## Table of Contents
- [Why Separate Instances?](#why-separate-instances)
- [Architecture Overview](#architecture-overview)
- [Step 1: Create Production Instance](#step-1-create-production-instance)
- [Step 2: Create Development Instance](#step-2-create-development-instance)
- [Step 3: Configure systemd Services](#step-3-configure-systemd-services)
- [Step 4: Create Databases](#step-4-create-databases)
- [Step 5: Copy Production to Development](#step-5-copy-production-to-development)
- [Verification and Testing](#verification-and-testing)
- [Daily Operations](#daily-operations)
- [Backup and Recovery](#backup-and-recovery)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## Why Separate Instances?

### Complete Isolation Benefits:

| Aspect | Same Instance | Separate Instances |
|--------|---------------|-------------------|
| **Data Files** | Shared directory | Separate directories |
| **WAL Files** | Mixed (both DBs) | Isolated per instance |
| **PITR Recovery** | Restores ALL databases | Restore each independently |
| **Configuration** | Shared settings | Independent settings |
| **Restart Impact** | Affects both DBs | Independent restarts |
| **Resource Limits** | Shared memory pool | Configurable per instance |
| **Monitoring** | Combined metrics | Separate metrics |

### Key Advantage: WAL File Isolation

**Problem with single instance:**
```
Port 5432 (Single Instance)
├── Database: prod
├── Database: dev
└── WAL: /var/lib/postgres/data/pg_wal/
    └── Mixed transactions from BOTH databases
    └── PITR restores BOTH to same point in time
```

**Solution with separate instances:**
```
Port 5432 (Production Instance)
├── Database: prod
└── WAL: /var/lib/postgres/prod_data/pg_wal/
    └── Only production transactions
    └── PITR restores only production

Port 5433 (Development Instance)
├── Database: dev
└── WAL: /var/lib/postgres/dev_data/pg_wal/
    └── Only development transactions
    └── PITR restores only development
```

---

## Architecture Overview

### Target Setup:

```
Production Instance
├── Port: 5432
├── Data Directory: /var/lib/postgres/prod_data/
├── Database: vessel_analytics_prod
├── WAL Archive: /backup/wal/prod/
├── systemd Service: postgresql@prod.service
└── Purpose: Production workload, full backups

Development Instance
├── Port: 5433
├── Data Directory: /var/lib/postgres/dev_data/
├── Database: vessel_analytics_dev
├── WAL Archive: Disabled (optional)
├── systemd Service: postgresql@dev.service
└── Purpose: Development, testing, experimentation
```

### Resource Usage:

- **Production Instance:** ~200-500 MB RAM (depends on configuration)
- **Development Instance:** ~200-500 MB RAM (depends on configuration)
- **Total Overhead:** ~400-1000 MB RAM (negligible on modern systems)

---

## Step 1: Create Production Instance

### 1.1 Initialize Production Cluster

```bash
# Switch to postgres user
sudo -u postgres bash

# Create production data directory
initdb -D /var/lib/postgres/prod_data

# Exit postgres user
exit
```

**Expected output:**
```
The files belonging to this database system will be owned by user "postgres".
...
Success. You can now start the database server using:
    pg_ctl -D /var/lib/postgres/prod_data -l logfile start
```

### 1.2 Configure Production Instance

```bash
# Set port to 5432 (standard PostgreSQL port)
sudo -u postgres bash -c "cat >> /var/lib/postgres/prod_data/postgresql.conf << EOF

# Production Instance Configuration
port = 5432

# Memory Configuration (adjust based on your system)
shared_buffers = 256MB
work_mem = 8MB
maintenance_work_mem = 128MB

# WAL Configuration
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backup/wal/prod/%f && cp %p /backup/wal/prod/%f'
archive_timeout = 300

# Connection Settings
max_connections = 100
EOF"
```

### 1.3 Create WAL Archive Directory

```bash
# Create archive directory
sudo mkdir -p /backup/wal/prod
sudo chown postgres:postgres /backup/wal/prod
sudo chmod 755 /backup/wal/prod
```

### 1.4 Set Up systemd Service for Production

Create the systemd service file:

```bash
# Create the service file
sudo nano /etc/systemd/system/postgresql@prod.service
```

Paste this configuration:

```ini
[Unit]
Description=PostgreSQL database server - Production Instance
After=network.target

[Service]
Type=forking
User=postgres
Group=postgres

Environment=PGDATA=/var/lib/postgres/prod_data
ExecStart=/usr/bin/pg_ctl -D /var/lib/postgres/prod_data -l /var/lib/postgres/prod_data/logfile start
ExecStop=/usr/bin/pg_ctl -D /var/lib/postgres/prod_data stop
ExecReload=/usr/bin/pg_ctl -D /var/lib/postgres/prod_data reload

Restart=on-failure
RestartSec=5s

OOMScoreAdjust=-900
TimeoutSec=0

RuntimeDirectory=postgresql

[Install]
WantedBy=multi-user.target
```

Save and exit (Ctrl+O, Enter, Ctrl+X).

### 1.5 Enable and Start Production Service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable production service (auto-start on boot)
sudo systemctl enable postgresql@prod

# Start production service
sudo systemctl start postgresql@prod

# Check status
sudo systemctl status postgresql@prod
```

**Expected output:**
```
● postgresql@prod.service - PostgreSQL database server - Production Instance
     Loaded: loaded (/etc/systemd/system/postgresql@prod.service; enabled)
     Active: active (running) since ...
```

---

## Step 2: Create Development Instance

### 2.1 Initialize Development Cluster

```bash
# Switch to postgres user
sudo -u postgres bash

# Create development data directory
initdb -D /var/lib/postgres/dev_data

# Exit postgres user
exit
```

### 2.2 Configure Development Instance

```bash
# Set port to 5433 (different from production)
sudo -u postgres bash -c "cat >> /var/lib/postgres/dev_data/postgresql.conf << EOF

# Development Instance Configuration
port = 5433

# Memory Configuration (lighter than production)
shared_buffers = 128MB
work_mem = 4MB
maintenance_work_mem = 64MB

# WAL Configuration (optional - you may not need archiving for dev)
wal_level = replica
# archive_mode = off  # Disable archiving for dev to save disk space

# Connection Settings
max_connections = 50
EOF"
```

**Note:** We disable WAL archiving for dev to save disk space and reduce overhead.

### 2.3 Set Up systemd Service for Development

Create the systemd service file:

```bash
# Create the service file
sudo nano /etc/systemd/system/postgresql@dev.service
```

Paste this configuration:

```ini
[Unit]
Description=PostgreSQL database server - Development Instance
After=network.target

[Service]
Type=forking
User=postgres
Group=postgres

Environment=PGDATA=/var/lib/postgres/dev_data
ExecStart=/usr/bin/pg_ctl -D /var/lib/postgres/dev_data -l /var/lib/postgres/dev_data/logfile start
ExecStop=/usr/bin/pg_ctl -D /var/lib/postgres/dev_data stop
ExecReload=/usr/bin/pg_ctl -D /var/lib/postgres/dev_data reload

Restart=on-failure
RestartSec=5s

OOMScoreAdjust=-900
TimeoutSec=0

RuntimeDirectory=postgresql

[Install]
WantedBy=multi-user.target
```

Save and exit.

### 2.4 Enable and Start Development Service

```bash
# Reload systemd
sudo systemctl daemon-reload

# Enable development service (auto-start on boot)
sudo systemctl enable postgresql@dev

# Start development service
sudo systemctl start postgresql@dev

# Check status
sudo systemctl status postgresql@dev
```

---

## Step 3: Configure systemd Services

Both services are now configured. Here's how to manage them:

### Service Management Commands:

```bash
# Production Instance
sudo systemctl start postgresql@prod      # Start
sudo systemctl stop postgresql@prod       # Stop
sudo systemctl restart postgresql@prod    # Restart
sudo systemctl status postgresql@prod     # Status
sudo systemctl enable postgresql@prod     # Auto-start on boot
sudo systemctl disable postgresql@prod    # Disable auto-start

# Development Instance
sudo systemctl start postgresql@dev       # Start
sudo systemctl stop postgresql@dev        # Stop
sudo systemctl restart postgresql@dev     # Restart
sudo systemctl status postgresql@dev      # Status
sudo systemctl enable postgresql@dev      # Auto-start on boot
sudo systemctl disable postgresql@dev     # Disable auto-start

# Both instances
sudo systemctl status 'postgresql@*'      # Status of all
```

### View Logs:

```bash
# Production logs
sudo journalctl -u postgresql@prod -f

# Development logs
sudo journalctl -u postgresql@dev -f

# Or check log files directly
sudo tail -f /var/lib/postgres/prod_data/logfile
sudo tail -f /var/lib/postgres/dev_data/logfile
```

---

## Step 4: Create Databases

### 4.1 Create Users on Both Instances

**Production Instance (port 5432):**

```bash
# Connect as postgres superuser
psql -p 5432 -U postgres

# Create your user
CREATE USER gitgud WITH SUPERUSER;

# Exit
\q
```

**Development Instance (port 5433):**

```bash
# Connect as postgres superuser
psql -p 5433 -U postgres

# Create your user
CREATE USER gitgud WITH SUPERUSER;

# Exit
\q
```

### 4.2 Create Production Database

```bash
# Create production database
createdb -p 5432 -U postgres -O gitgud vessel_analytics_prod

# Connect to verify
psql -p 5432 -U gitgud -d vessel_analytics_prod
```

Inside psql, verify you're on the right instance:

```sql
-- Check connection info
\conninfo

-- Should show:
-- You are connected to database "vessel_analytics_prod" as user "gitgud" on host "localhost" at port "5432"

-- Check data directory (confirms instance)
SHOW data_directory;
-- Should show: /var/lib/postgres/prod_data

-- Exit
\q
```

### 4.3 Load Schema and Data into Production

```bash
# Load schema
psql -p 5432 -U gitgud -d vessel_analytics_prod -f schema.sql

# Load data (adjust path as needed)
# python load_data.py  # Make sure it connects to port 5432
```

**Important:** Update your `load_data.py` to connect to the correct port:

```python
# In load_data.py
DB_PARAMS = {
    "dbname": "vessel_analytics_prod",
    "user": "gitgud",
    "host": "localhost",
    "port": 5432,  # Production port
}
```

---

## Step 5: Copy Production to Development

### Method 1: Using pg_dump and pg_restore (Recommended)

This method is **100% safe** - it only reads from production and writes to development.

#### 5.1 Dump Production Database

```bash
# Create dump file
pg_dump -p 5432 -U gitgud -d vessel_analytics_prod -F c -f /tmp/prod_backup.dump

# Verify dump was created
ls -lh /tmp/prod_backup.dump
```

**What this does:**
- ✅ Only **READS** from production
- ✅ No modifications to production
- ✅ Creates a compressed backup file
- ⚠️  Takes brief ACCESS SHARE lock (allows other reads/writes)

#### 5.2 Create Development Database

```bash
# Create development database
createdb -p 5433 -U gitgud vessel_analytics_dev

# Verify it was created
psql -p 5433 -U gitgud -l | grep vessel_analytics_dev
```

#### 5.3 Restore to Development

```bash
# Restore production backup to development
pg_restore -p 5433 -U gitgud -d vessel_analytics_dev /tmp/prod_backup.dump

# Check for errors
echo $?  # Should be 0 if successful
```

**What this does:**
- ✅ Only **WRITES** to development database
- ✅ No interaction with production
- ✅ Safe to run multiple times (idempotent with --clean option)

#### 5.4 Clean Up

```bash
# Remove backup file
rm /tmp/prod_backup.dump
```

#### 5.5 Verify Development Database

```bash
# Connect to development
psql -p 5433 -U gitgud -d vessel_analytics_dev
```

Inside psql:

```sql
-- Verify connection
\conninfo
-- Should show port "5433" and database "vessel_analytics_dev"

-- Verify data directory
SHOW data_directory;
-- Should show: /var/lib/postgres/dev_data

-- Check tables
\dt

-- Count records (compare with production)
SELECT COUNT(*) FROM vessel_characteristics;
SELECT COUNT(*) FROM raw_vessel_events;

-- Mark as development environment
CREATE TABLE _environment (name TEXT);
INSERT INTO _environment VALUES ('development');
COMMENT ON TABLE _environment IS 'Marker to identify this as development database';

-- Exit
\q
```

### Method 2: Automated Script

Create a reusable script:

```bash
# Create script
nano ~/refresh_dev_from_prod.sh
```

Paste this content:

```bash
#!/bin/bash
# Refresh development database from production

set -e  # Exit on error

PROD_PORT=5432
DEV_PORT=5433
PROD_DB="vessel_analytics_prod"
DEV_DB="vessel_analytics_dev"
BACKUP_FILE="/tmp/prod_backup_$(date +%Y%m%d_%H%M%S).dump"

echo "=== Refreshing Development from Production ==="
echo "This script will:"
echo "  1. Dump production database (read-only, no changes)"
echo "  2. Drop existing development database"
echo "  3. Create fresh development database"
echo "  4. Restore production data to development"
echo ""
echo "Production database will NOT be modified."
echo ""
read -p "Continue? (y/n) " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo "Aborted."
    exit 1
fi

echo ""
echo "[1/5] Dumping production database..."
pg_dump -p $PROD_PORT -U gitgud -d $PROD_DB -F c -f $BACKUP_FILE
echo "✓ Production dumped to $BACKUP_FILE"

echo ""
echo "[2/5] Dropping existing development database..."
dropdb -p $DEV_PORT -U gitgud --if-exists $DEV_DB
echo "✓ Old development database dropped"

echo ""
echo "[3/5] Creating new development database..."
createdb -p $DEV_PORT -U gitgud $DEV_DB
echo "✓ Development database created"

echo ""
echo "[4/5] Restoring production data to development..."
pg_restore -p $DEV_PORT -U gitgud -d $DEV_DB $BACKUP_FILE
echo "✓ Data restored to development"

echo ""
echo "[5/5] Marking as development environment..."
psql -p $DEV_PORT -U gitgud -d $DEV_DB <<EOF
DROP TABLE IF EXISTS _environment;
CREATE TABLE _environment (name TEXT);
INSERT INTO _environment VALUES ('development');
COMMENT ON TABLE _environment IS 'Marker to identify this as development database';
EOF
echo "✓ Development environment marked"

echo ""
echo "Cleaning up..."
rm $BACKUP_FILE
echo "✓ Backup file removed"

echo ""
echo "=== Development Database Ready ==="
echo "Connect with: psql -p $DEV_PORT -U gitgud -d $DEV_DB"
echo ""
echo "Production database at port $PROD_PORT was not modified."
```

Save and make executable:

```bash
chmod +x ~/refresh_dev_from_prod.sh
```

Run it:

```bash
~/refresh_dev_from_prod.sh
```

---

## Verification and Testing

### Verify Complete Isolation

Run these tests to confirm production and development are completely isolated:

#### Test 1: Different Data Directories

```bash
echo "=== Production Instance ==="
psql -p 5432 -U gitgud -d vessel_analytics_prod -c "SHOW data_directory;"

echo ""
echo "=== Development Instance ==="
psql -p 5433 -U gitgud -d vessel_analytics_dev -c "SHOW data_directory;"
```

**Expected output:**
```
=== Production Instance ===
 data_directory
---------------------------------
 /var/lib/postgres/prod_data

=== Development Instance ===
 data_directory
--------------------------------
 /var/lib/postgres/dev_data
```

#### Test 2: Different WAL Files

```bash
echo "=== Production WAL Directory ==="
ls -lh /var/lib/postgres/prod_data/pg_wal/ | head -5

echo ""
echo "=== Development WAL Directory ==="
ls -lh /var/lib/postgres/dev_data/pg_wal/ | head -5
```

These should show completely different files.

#### Test 3: Modify Development Without Affecting Production

```bash
# Connect to development
psql -p 5433 -U gitgud -d vessel_analytics_dev
```

```sql
-- Make destructive changes to dev
CREATE TABLE test_isolation (id INT);
INSERT INTO test_isolation VALUES (1), (2), (3);

-- Delete some data (safe in dev!)
DELETE FROM raw_vessel_events WHERE random() > 0.5;

-- Check remaining data
SELECT COUNT(*) FROM raw_vessel_events;

-- Exit
\q
```

Now check production is untouched:

```bash
# Connect to production
psql -p 5432 -U gitgud -d vessel_analytics_prod
```

```sql
-- Verify test table doesn't exist
\dt test_isolation
-- Should show: "Did not find any relation named test_isolation"

-- Verify all data still there
SELECT COUNT(*) FROM raw_vessel_events;
-- Should show full count (no deletions)

-- Exit
\q
```

✅ **Production is completely unaffected!**

#### Test 4: Verify Archive Isolation

```bash
# Check production archives
echo "Production WAL Archives:"
ls -lh /backup/wal/prod/ | wc -l

# Development shouldn't have archives (we disabled archiving)
echo "Development WAL Archives:"
ls -lh /backup/wal/dev/ 2>&1 || echo "Not configured (expected)"
```

---

## Daily Operations

### Connecting to the Right Database

**Always verify before running commands!**

```bash
# Production
psql -p 5432 -U gitgud -d vessel_analytics_prod

# Development
psql -p 5433 -U gitgud -d vessel_analytics_dev
```

**Inside psql, always check:**

```sql
\conninfo  -- Shows port, database, user
SHOW data_directory;  -- Confirms instance
```

### Set Your Prompt to Show Which Database

Add to `~/.psqlrc`:

```
\set PROMPT1 '%n@%M:%>/%/ [%x] %# '
```

Your prompt will now show:
```
gitgud@localhost:5432/vessel_analytics_prod [TX] #
gitgud@localhost:5433/vessel_analytics_dev [TX] #
```

### Quick Reference Aliases

Add to your `~/.bashrc` or `~/.bash_aliases`:

```bash
# PostgreSQL aliases
alias psql-prod='psql -p 5432 -U gitgud -d vessel_analytics_prod'
alias psql-dev='psql -p 5433 -U gitgud -d vessel_analytics_dev'

# Systemd aliases
alias pg-prod-status='sudo systemctl status postgresql@prod'
alias pg-dev-status='sudo systemctl status postgresql@dev'
alias pg-prod-restart='sudo systemctl restart postgresql@prod'
alias pg-dev-restart='sudo systemctl restart postgresql@dev'
alias pg-prod-logs='sudo journalctl -u postgresql@prod -f'
alias pg-dev-logs='sudo journalctl -u postgresql@dev -f'

# Refresh dev from prod
alias refresh-dev='~/refresh_dev_from_prod.sh'
```

Reload your shell:

```bash
source ~/.bashrc
```

Now you can use:

```bash
psql-prod     # Connect to production
psql-dev      # Connect to development
refresh-dev   # Refresh dev from prod
```

---

## Backup and Recovery

### Production Backups

#### Daily Logical Backup (pg_dump)

```bash
# Create backup script
sudo nano /usr/local/bin/backup-postgres-prod.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/backup/postgres/prod"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="vessel_analytics_prod"

mkdir -p $BACKUP_DIR

pg_dump -p 5432 -U postgres -d $DB_NAME -F c -f $BACKUP_DIR/${DB_NAME}_${DATE}.dump

# Keep only last 7 days
find $BACKUP_DIR -name "*.dump" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/${DB_NAME}_${DATE}.dump"
```

```bash
sudo chmod +x /usr/local/bin/backup-postgres-prod.sh
```

**Add to cron** (daily at 2 AM):

```bash
sudo crontab -e
```

Add:

```
0 2 * * * /usr/local/bin/backup-postgres-prod.sh
```

#### Point-in-Time Recovery (PITR)

Production has WAL archiving enabled, so you can restore to any point in time.

**See `WAL_ARCHIVING_GUIDE.md` for complete PITR instructions.**

### Development Backups

**Development doesn't need regular backups** because:
- It's disposable - can be recreated from production anytime
- You can refresh it daily/weekly from production
- No critical data lives only in development

If you do need to backup dev for some reason:

```bash
pg_dump -p 5433 -U gitgud -d vessel_analytics_dev -F c -f /tmp/dev_backup.dump
```

---

## Best Practices

### 1. Never Run Production Commands on Development (and vice versa)

❌ **BAD:**
```bash
# Might accidentally run on wrong instance
psql -d vessel_analytics_prod  # Which port? Easy to forget!
```

✅ **GOOD:**
```bash
# Always specify port explicitly
psql -p 5432 -U gitgud -d vessel_analytics_prod
```

### 2. Use Environment Markers

```sql
-- In production
CREATE TABLE _environment (name TEXT);
INSERT INTO _environment VALUES ('production');

-- In development
CREATE TABLE _environment (name TEXT);
INSERT INTO _environment VALUES ('development');

-- Before running dangerous commands, check:
SELECT * FROM _environment;
```

### 3. Different Color Schemes in psql

Production (red prompt):

```bash
export PSQL_PAGER='less -S -R'
export PGHOST=localhost
export PGPORT=5432
export PGDATABASE=vessel_analytics_prod
export PROMPT_COMMAND='echo -ne "\033]0;PRODUCTION - PostgreSQL\007"'
```

Development (green prompt):

```bash
export PSQL_PAGER='less -S -R'
export PGHOST=localhost
export PGPORT=5433
export PGDATABASE=vessel_analytics_dev
export PROMPT_COMMAND='echo -ne "\033]0;DEVELOPMENT - PostgreSQL\007"'
```

### 4. Sanitize Sensitive Data in Development

After copying prod to dev:

```sql
-- Anonymize PII
UPDATE users SET
    email = 'dev_user_' || id || '@example.com',
    phone = NULL,
    password = 'dev_password_hash';

-- Remove sensitive data
DELETE FROM payment_information;
DELETE FROM personal_documents;
```

### 5. Regular Refresh Schedule

Refresh development from production regularly:

```bash
# Weekly on Mondays at 6 AM
0 6 * * 1 /home/gitgud/refresh_dev_from_prod.sh
```

### 6. Monitor Resource Usage

```bash
# Check memory usage
ps aux | grep postgres | awk '{sum+=$6} END {print "Total RSS:", sum/1024, "MB"}'

# Check disk usage
du -sh /var/lib/postgres/prod_data/
du -sh /var/lib/postgres/dev_data/
du -sh /backup/wal/prod/

# Check active connections per instance
psql -p 5432 -U gitgud -d vessel_analytics_prod -c "SELECT count(*) FROM pg_stat_activity;"
psql -p 5433 -U gitgud -d vessel_analytics_dev -c "SELECT count(*) FROM pg_stat_activity;"
```

---

## Troubleshooting

### Issue: Can't Connect to Instance

**Symptom:**
```
psql: error: connection to server on socket "/run/postgresql/.s.PGSQL.5432" failed
```

**Solutions:**

1. Check if instance is running:
```bash
sudo systemctl status postgresql@prod
sudo systemctl status postgresql@dev
```

2. Check port configuration:
```bash
sudo grep "^port" /var/lib/postgres/prod_data/postgresql.conf
sudo grep "^port" /var/lib/postgres/dev_data/postgresql.conf
```

3. Check logs:
```bash
sudo journalctl -u postgresql@prod -n 50
sudo tail -50 /var/lib/postgres/prod_data/logfile
```

### Issue: Port Already in Use

**Symptom:**
```
FATAL:  could not create lock file "/run/postgresql/.s.PGSQL.5432.lock"
```

**Solution:**

Check what's using the port:
```bash
sudo ss -tlnp | grep 5432
sudo ss -tlnp | grep 5433
```

If another instance is using it, change the port in postgresql.conf.

### Issue: Permission Denied

**Symptom:**
```
FATAL:  role "gitgud" does not exist
```

**Solution:**

Create the user on that specific instance:
```bash
psql -p 5432 -U postgres -c "CREATE USER gitgud WITH SUPERUSER;"
psql -p 5433 -U postgres -c "CREATE USER gitgud WITH SUPERUSER;"
```

### Issue: WAL Archiving Failing

**Symptom:**
```
LOG: archive command failed with exit code 1
```

**Solutions:**

1. Check archive directory exists and has correct permissions:
```bash
ls -ld /backup/wal/prod/
sudo chown postgres:postgres /backup/wal/prod/
sudo chmod 755 /backup/wal/prod/
```

2. Check archive_command syntax:
```bash
psql -p 5432 -U gitgud -d vessel_analytics_prod -c "SHOW archive_command;"
```

3. Check logs:
```bash
sudo grep "archive" /var/lib/postgres/prod_data/logfile | tail -20
```

### Issue: Ran Command on Wrong Instance

**Symptom:**
"Oh no, I just deleted data from production instead of dev!"

**Solution:**

If you have WAL archiving (production):
1. Stop accepting new connections immediately
2. Use PITR to restore to just before the mistake
3. See `WAL_ARCHIVING_GUIDE.md` for recovery procedure

If you don't have backups:
- Check if you have a recent pg_dump backup
- Restore from backup to a new database
- Investigate what was lost

**Prevention:**
- Always check `\conninfo` before destructive operations
- Use transaction blocks: `BEGIN; ... ROLLBACK;` for testing
- Require explicit confirmation for destructive operations

### Issue: Out of Disk Space

**Symptom:**
```
ERROR: could not extend file: No space left on device
```

**Solutions:**

1. Check disk usage:
```bash
df -h
du -sh /var/lib/postgres/*/
du -sh /backup/wal/prod/
```

2. Clean up old WAL archives:
```bash
# Keep only last 7 days
find /backup/wal/prod/ -name "0*" -mtime +7 -delete
```

3. Clean up old pg_dump backups:
```bash
find /backup/postgres/ -name "*.dump" -mtime +30 -delete
```

4. Vacuum databases:
```sql
VACUUM FULL;  -- Reclaims space but locks tables
```

---

## Summary

### What You've Built:

✅ **Complete Isolation:**
- Separate data directories
- Separate WAL files
- Independent PITR
- Independent configuration

✅ **Safe Development:**
- Can't accidentally affect production
- Easy to refresh from production
- Disposable - recreate anytime

✅ **Production Ready:**
- WAL archiving enabled
- Point-in-Time Recovery possible
- Auto-start on boot
- systemd management

### Key Commands Reference:

```bash
# Service Management
sudo systemctl start/stop/restart postgresql@prod
sudo systemctl start/stop/restart postgresql@dev
sudo systemctl status 'postgresql@*'

# Connections
psql -p 5432 -U gitgud -d vessel_analytics_prod
psql -p 5433 -U gitgud -d vessel_analytics_dev

# Refresh Dev from Prod
~/refresh_dev_from_prod.sh

# Backups
pg_dump -p 5432 -U gitgud -d vessel_analytics_prod -F c -f backup.dump
pg_restore -p 5433 -U gitgud -d vessel_analytics_dev backup.dump

# Monitoring
du -sh /var/lib/postgres/*/
ls -lh /backup/wal/prod/
sudo systemctl status 'postgresql@*'
```

### Resource Usage:

- **Memory:** ~400-1000 MB total (both instances)
- **Disk:** Depends on data size + WAL archives
- **CPU:** Minimal when idle, depends on workload when active
- **Processes:** ~12-20 background processes total

### Safety Guarantees:

✅ **pg_dump from prod:** Only reads, never modifies
✅ **pg_restore to dev:** Only writes to dev, never touches prod
✅ **Separate instances:** Physically impossible to share data
✅ **Different ports:** Explicit targeting required

**The ONLY way to affect production is user error (connecting to wrong port).**

---

## Related Documentation

- `POSTGRESQL_INSTANCES_GUIDE.md` - Detailed guide on managing multiple PostgreSQL instances
- `WAL_ARCHIVING_GUIDE.md` - Complete guide for WAL archiving and Point-in-Time Recovery
- `STUDY_PLAN.md` - SQL revision exercises using your vessel analytics data

---

**Last Updated:** 2025-11-22

This setup provides production-grade isolation between production and development environments while maintaining ease of use and safety. You can now confidently experiment in development knowing production is completely protected.
