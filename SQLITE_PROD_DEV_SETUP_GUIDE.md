# SQLite Production and Development Setup Guide

Complete guide for setting up isolated production and development SQLite databases with Python, including backup and recovery strategies.

## Table of Contents
- [SQLite vs PostgreSQL Architecture](#sqlite-vs-postgresql-architecture)
- [Why SQLite is Different](#why-sqlite-is-different)
- [Production/Development Setup](#productiondevelopment-setup)
- [Python Integration](#python-integration)
- [Backup Strategies](#backup-strategies)
- [Point-in-Time Recovery for SQLite](#point-in-time-recovery-for-sqlite)
- [Data Migration: PostgreSQL to SQLite](#data-migration-postgresql-to-sqlite)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

---

## SQLite vs PostgreSQL Architecture

### PostgreSQL (Client-Server)

```
┌─────────────────────────────────────┐
│   PostgreSQL Server (Process)      │
│   ├── Port 5432 (Production)       │
│   │   └── vessel_analytics_prod    │
│   └── Port 5433 (Development)      │
│       └── vessel_analytics_dev     │
└─────────────────────────────────────┘
        ↑
        │ Network connection (psql client)
        │
┌───────────────────┐
│   Your App        │
└───────────────────┘
```

**Characteristics:**
- Server process always running
- Client-server architecture
- Multiple concurrent connections
- Port-based isolation
- WAL archiving built-in
- PITR support built-in

### SQLite (Embedded/Serverless)

```
┌─────────────────────────────────────┐
│   Your App (Process)                │
│   ├── opens: prod.db                │
│   └── opens: dev.db                 │
└─────────────────────────────────────┘
        ↑
        │ Direct file access
        │
┌─────────────────────────────────────┐
│   Filesystem                        │
│   ├── /data/prod.db                 │
│   ├── /data/prod.db-wal             │
│   ├── /data/prod.db-shm             │
│   ├── /data/dev.db                  │
│   ├── /data/dev.db-wal              │
│   └── /data/dev.db-shm              │
└─────────────────────────────────────┘
```

**Characteristics:**
- No server process (embedded)
- Database is a single file
- Library runs in your app's process
- File-based isolation
- WAL mode for crash recovery (not archiving)
- PITR requires custom solution

---

## Why SQLite is Different

### Key Differences:

| Aspect | PostgreSQL | SQLite |
|--------|------------|--------|
| **Architecture** | Client-Server | Embedded library |
| **Database** | Server-managed objects | Single file on disk |
| **Isolation** | Different ports/instances | Different files |
| **Connections** | Network (localhost or remote) | Direct file access |
| **Concurrency** | Many concurrent writers | One writer at a time |
| **Backup** | pg_dump, pg_basebackup | File copy |
| **PITR** | Built-in WAL archiving | Custom implementation |
| **Setup** | Install, configure, start service | Just open a file |
| **When to use** | High traffic, multiple apps | Embedded, single app, < 1TB |

### SQLite Advantages:

✅ **Simplicity:**
- No server to install/configure
- No ports, users, permissions
- Database is just a file
- Zero configuration

✅ **Portability:**
- Copy file = copy database
- Works on any platform
- Self-contained

✅ **Performance:**
- No network overhead
- No client-server communication
- Very fast for reads

✅ **Reliability:**
- ACID compliant
- Crash-safe with WAL mode
- Battle-tested (used in billions of devices)

### SQLite Limitations:

⚠️ **Concurrency:**
- Only one writer at a time
- Readers can work during writes (WAL mode)
- Not suitable for high-concurrency writes

⚠️ **Size:**
- Recommended < 1 TB
- Performance degrades with very large databases

⚠️ **No Built-in Replication:**
- No master-slave replication
- No built-in PITR
- Need custom solutions

---

## Production/Development Setup

### Directory Structure

```
/home/gitgud/Projects/sql-revision/
├── data/
│   ├── prod/
│   │   ├── vessel_analytics.db          # Production database
│   │   ├── vessel_analytics.db-wal      # Write-Ahead Log
│   │   └── vessel_analytics.db-shm      # Shared memory file
│   └── dev/
│       ├── vessel_analytics.db          # Development database
│       ├── vessel_analytics.db-wal
│       └── vessel_analytics.db-shm
├── backups/
│   ├── prod/
│   │   ├── daily/                       # Daily full backups
│   │   ├── wal_archives/                # WAL file archives (for PITR)
│   │   └── snapshots/                   # Point-in-time snapshots
│   └── scripts/
│       ├── backup_prod.sh
│       ├── restore_prod.sh
│       └── archive_wal.sh
├── scripts/
│   ├── init_prod.py                     # Initialize production DB
│   ├── init_dev.py                      # Initialize development DB
│   ├── refresh_dev_from_prod.py         # Copy prod to dev
│   └── db_utils.py                      # Common utilities
└── schema.sql                           # Database schema
```

### Step 1: Create Directory Structure

```bash
# From your project directory
cd /home/gitgud/Projects/sql-revision/

# Create directories
mkdir -p data/prod data/dev
mkdir -p backups/prod/daily backups/prod/wal_archives backups/prod/snapshots
mkdir -p backups/scripts
mkdir -p scripts

# Set permissions (if needed)
chmod 755 data/ backups/ scripts/
```

### Step 2: Database Configuration Module

Create `scripts/db_config.py`:

```python
"""
Database configuration for production and development.
"""
import os
from pathlib import Path

# Project root
PROJECT_ROOT = Path(__file__).parent.parent

# Database paths
PROD_DB_PATH = PROJECT_ROOT / "data" / "prod" / "vessel_analytics.db"
DEV_DB_PATH = PROJECT_ROOT / "data" / "dev" / "vessel_analytics.db"

# Backup paths
BACKUP_ROOT = PROJECT_ROOT / "backups" / "prod"
DAILY_BACKUP_PATH = BACKUP_ROOT / "daily"
WAL_ARCHIVE_PATH = BACKUP_ROOT / "wal_archives"
SNAPSHOT_PATH = BACKUP_ROOT / "snapshots"

# Ensure directories exist
for path in [PROD_DB_PATH.parent, DEV_DB_PATH.parent,
             DAILY_BACKUP_PATH, WAL_ARCHIVE_PATH, SNAPSHOT_PATH]:
    path.mkdir(parents=True, exist_ok=True)

# SQLite configuration
SQLITE_CONFIG = {
    "timeout": 10.0,           # Wait up to 10 seconds for lock
    "isolation_level": None,   # Autocommit mode (manage transactions manually)
    "check_same_thread": False # Allow use from multiple threads (be careful!)
}

# WAL mode configuration (recommended for better concurrency)
WAL_MODE_PRAGMAS = [
    "PRAGMA journal_mode=WAL;",           # Enable WAL mode
    "PRAGMA synchronous=NORMAL;",         # Balance safety and speed
    "PRAGMA cache_size=-64000;",          # 64MB cache
    "PRAGMA temp_store=MEMORY;",          # Use memory for temp tables
    "PRAGMA mmap_size=268435456;",        # 256MB memory-mapped I/O
]

# Read-only configuration
READONLY_CONFIG = {
    "uri": True,  # Enable URI filenames
}

def get_db_uri(db_path: Path, readonly: bool = False) -> str:
    """
    Get SQLite URI connection string.

    Args:
        db_path: Path to database file
        readonly: If True, open in read-only mode

    Returns:
        URI connection string
    """
    mode = "?mode=ro" if readonly else ""
    return f"file:{db_path}{mode}"


def get_environment_marker(db_path: Path) -> str:
    """
    Determine if database is production or development.

    Args:
        db_path: Path to database file

    Returns:
        "production" or "development"
    """
    if "prod" in str(db_path):
        return "production"
    elif "dev" in str(db_path):
        return "development"
    else:
        return "unknown"
```

### Step 3: Database Utilities Module

Create `scripts/db_utils.py`:

```python
"""
Common database utilities for SQLite.
"""
import sqlite3
import shutil
from pathlib import Path
from datetime import datetime
from typing import Optional
from contextlib import contextmanager

from db_config import (
    SQLITE_CONFIG, WAL_MODE_PRAGMAS, READONLY_CONFIG,
    get_db_uri, get_environment_marker
)


@contextmanager
def get_connection(db_path: Path, readonly: bool = False):
    """
    Context manager for SQLite connections.

    Args:
        db_path: Path to database file
        readonly: If True, open in read-only mode

    Yields:
        sqlite3.Connection

    Example:
        with get_connection(PROD_DB_PATH) as conn:
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM vessels")
    """
    if readonly:
        # Read-only connection using URI
        uri = get_db_uri(db_path, readonly=True)
        conn = sqlite3.connect(uri, **READONLY_CONFIG, **SQLITE_CONFIG)
    else:
        # Read-write connection
        conn = sqlite3.connect(db_path, **SQLITE_CONFIG)

        # Enable WAL mode and optimizations
        cursor = conn.cursor()
        for pragma in WAL_MODE_PRAGMAS:
            cursor.execute(pragma)
        cursor.close()

    # Enable foreign keys
    conn.execute("PRAGMA foreign_keys=ON;")

    try:
        yield conn
    finally:
        conn.close()


def verify_environment(conn: sqlite3.Connection) -> str:
    """
    Check which environment the database belongs to.

    Args:
        conn: Database connection

    Returns:
        "production", "development", or "unknown"
    """
    cursor = conn.cursor()

    # Check if _environment table exists
    cursor.execute("""
        SELECT name FROM sqlite_master
        WHERE type='table' AND name='_environment'
    """)

    if cursor.fetchone():
        cursor.execute("SELECT name FROM _environment LIMIT 1")
        row = cursor.fetchone()
        if row:
            return row[0]

    return "unknown"


def mark_environment(conn: sqlite3.Connection, environment: str):
    """
    Mark database as production or development.

    Args:
        conn: Database connection
        environment: "production" or "development"
    """
    cursor = conn.cursor()

    # Create table if it doesn't exist
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS _environment (
            name TEXT PRIMARY KEY,
            marked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)

    # Insert or replace marker
    cursor.execute("""
        INSERT OR REPLACE INTO _environment (name) VALUES (?)
    """, (environment,))

    conn.commit()


def get_db_info(db_path: Path) -> dict:
    """
    Get information about a database file.

    Args:
        db_path: Path to database file

    Returns:
        Dictionary with database information
    """
    if not db_path.exists():
        return {"exists": False}

    stat = db_path.stat()

    with get_connection(db_path, readonly=True) as conn:
        cursor = conn.cursor()

        # Get page count and page size
        cursor.execute("PRAGMA page_count;")
        page_count = cursor.fetchone()[0]

        cursor.execute("PRAGMA page_size;")
        page_size = cursor.fetchone()[0]

        # Get table count
        cursor.execute("""
            SELECT COUNT(*) FROM sqlite_master
            WHERE type='table' AND name NOT LIKE 'sqlite_%'
        """)
        table_count = cursor.fetchone()[0]

        # Get environment
        environment = verify_environment(conn)

        # Get journal mode
        cursor.execute("PRAGMA journal_mode;")
        journal_mode = cursor.fetchone()[0]

    return {
        "exists": True,
        "path": str(db_path),
        "size_bytes": stat.st_size,
        "size_mb": stat.st_size / (1024 * 1024),
        "modified": datetime.fromtimestamp(stat.st_mtime),
        "page_count": page_count,
        "page_size": page_size,
        "db_size_mb": (page_count * page_size) / (1024 * 1024),
        "table_count": table_count,
        "environment": environment,
        "journal_mode": journal_mode,
    }


def print_db_info(db_path: Path, label: str = "Database"):
    """
    Print formatted database information.

    Args:
        db_path: Path to database file
        label: Label to print
    """
    info = get_db_info(db_path)

    print(f"\n{'='*60}")
    print(f"{label.upper()}")
    print(f"{'='*60}")

    if not info["exists"]:
        print("❌ Database does not exist")
        print(f"Path: {db_path}")
        return

    print(f"✓ Path: {info['path']}")
    print(f"✓ Size: {info['size_mb']:.2f} MB")
    print(f"✓ Tables: {info['table_count']}")
    print(f"✓ Environment: {info['environment']}")
    print(f"✓ Journal Mode: {info['journal_mode']}")
    print(f"✓ Last Modified: {info['modified']}")
    print(f"{'='*60}\n")


def copy_database(source: Path, dest: Path, verify: bool = True) -> bool:
    """
    Safely copy a database from source to destination.

    This performs a hot backup - the source database can be in use.

    Args:
        source: Source database path
        dest: Destination database path
        verify: If True, verify the copy is valid

    Returns:
        True if successful
    """
    if not source.exists():
        raise FileNotFoundError(f"Source database not found: {source}")

    print(f"Copying database...")
    print(f"  From: {source}")
    print(f"  To:   {dest}")

    # Ensure destination directory exists
    dest.parent.mkdir(parents=True, exist_ok=True)

    # Use SQLite backup API for hot backup (safe even if DB is in use)
    with get_connection(source, readonly=True) as src_conn:
        with sqlite3.connect(dest) as dest_conn:
            src_conn.backup(dest_conn)

    print(f"✓ Copy completed")

    # Verify the copy
    if verify:
        print(f"Verifying copy...")
        with get_connection(dest, readonly=True) as conn:
            cursor = conn.cursor()
            cursor.execute("PRAGMA integrity_check;")
            result = cursor.fetchone()[0]

            if result == "ok":
                print(f"✓ Integrity check passed")
            else:
                print(f"❌ Integrity check failed: {result}")
                return False

    return True
```

### Step 4: Initialize Production Database

Create `scripts/init_prod.py`:

```python
"""
Initialize production database with schema and sample data.
"""
import sqlite3
from pathlib import Path

from db_config import PROD_DB_PATH, PROJECT_ROOT
from db_utils import get_connection, mark_environment, print_db_info


def init_production():
    """Initialize production database."""
    print("Initializing production database...")

    # Check if database already exists
    if PROD_DB_PATH.exists():
        response = input(f"⚠️  Production database already exists at {PROD_DB_PATH}\n"
                        f"   Do you want to recreate it? (yes/no): ")
        if response.lower() != 'yes':
            print("Aborted.")
            return

        # Remove existing database and WAL files
        PROD_DB_PATH.unlink()
        (PROD_DB_PATH.parent / f"{PROD_DB_PATH.name}-wal").unlink(missing_ok=True)
        (PROD_DB_PATH.parent / f"{PROD_DB_PATH.name}-shm").unlink(missing_ok=True)

    # Load schema
    schema_path = PROJECT_ROOT / "schema.sql"
    if not schema_path.exists():
        print(f"❌ Schema file not found: {schema_path}")
        return

    with open(schema_path, 'r') as f:
        schema_sql = f.read()

    # Create database and load schema
    with get_connection(PROD_DB_PATH) as conn:
        cursor = conn.cursor()

        # Execute schema (might have multiple statements)
        cursor.executescript(schema_sql)

        # Mark as production
        mark_environment(conn, "production")

        conn.commit()
        print("✓ Schema loaded")

    # Print info
    print_db_info(PROD_DB_PATH, "Production Database")

    print("\n✓ Production database initialized successfully!")
    print(f"\nNext steps:")
    print(f"  1. Load data: python load_data_sqlite.py")
    print(f"  2. Create dev database: python scripts/init_dev.py")


if __name__ == "__main__":
    init_production()
```

### Step 5: Copy Production to Development

Create `scripts/refresh_dev_from_prod.py`:

```python
"""
Refresh development database from production.

This script safely copies production to development without affecting production.
"""
from pathlib import Path
from datetime import datetime

from db_config import PROD_DB_PATH, DEV_DB_PATH
from db_utils import (
    copy_database, mark_environment, print_db_info,
    get_connection, verify_environment
)


def refresh_dev_from_prod():
    """
    Refresh development database from production.

    This is 100% safe:
    - Only READS from production (no modifications)
    - Only WRITES to development
    - Production database can be in use during copy
    """
    print("="*60)
    print("REFRESH DEVELOPMENT FROM PRODUCTION")
    print("="*60)

    # Check if production database exists
    if not PROD_DB_PATH.exists():
        print(f"❌ Production database not found: {PROD_DB_PATH}")
        print(f"\n   Run: python scripts/init_prod.py")
        return

    # Verify production environment
    with get_connection(PROD_DB_PATH, readonly=True) as conn:
        env = verify_environment(conn)
        if env != "production":
            print(f"⚠️  Warning: Source database is marked as '{env}', not 'production'")
            response = input("   Continue anyway? (yes/no): ")
            if response.lower() != 'yes':
                print("Aborted.")
                return

    # Show what will happen
    print("\nThis will:")
    print("  1. Read from production database (no changes to production)")
    print("  2. Overwrite development database")
    print("  3. Mark new database as 'development'")
    print("\n✅ Production database will NOT be modified.")

    # Confirm
    response = input("\nContinue? (yes/no): ")
    if response.lower() != 'yes':
        print("Aborted.")
        return

    print("\n" + "="*60)

    # Show source info
    print_db_info(PROD_DB_PATH, "Source (Production)")

    # Remove old development database and WAL files
    if DEV_DB_PATH.exists():
        print("Removing old development database...")
        DEV_DB_PATH.unlink()
        (DEV_DB_PATH.parent / f"{DEV_DB_PATH.name}-wal").unlink(missing_ok=True)
        (DEV_DB_PATH.parent / f"{DEV_DB_PATH.name}-shm").unlink(missing_ok=True)
        print("✓ Old development database removed")

    # Copy production to development
    print(f"\n[1/2] Copying database...")
    success = copy_database(PROD_DB_PATH, DEV_DB_PATH, verify=True)

    if not success:
        print("❌ Copy failed")
        return

    # Mark as development
    print(f"\n[2/2] Marking as development...")
    with get_connection(DEV_DB_PATH) as conn:
        mark_environment(conn, "development")
        conn.commit()
    print("✓ Marked as development")

    # Show destination info
    print_db_info(DEV_DB_PATH, "Destination (Development)")

    # Verify production unchanged
    print("Verifying production database was not modified...")
    with get_connection(PROD_DB_PATH, readonly=True) as conn:
        env = verify_environment(conn)
        if env == "production":
            print("✅ Production database verified (still marked as 'production')")
        else:
            print(f"⚠️  Warning: Production database now marked as '{env}'")

    print("\n" + "="*60)
    print("✅ DEVELOPMENT DATABASE REFRESHED SUCCESSFULLY")
    print("="*60)

    print(f"\nProduction: {PROD_DB_PATH}")
    print(f"Development: {DEV_DB_PATH}")

    print(f"\nYou can now safely experiment with the development database.")
    print(f"Production database was not modified and remains available.")


if __name__ == "__main__":
    refresh_dev_from_prod()
```

### Step 6: Application Code with Environment Safety

Create `scripts/app_example.py`:

```python
"""
Example application code demonstrating safe database access.
"""
import sqlite3
from pathlib import Path

from db_config import PROD_DB_PATH, DEV_DB_PATH
from db_utils import get_connection, verify_environment


def connect_to_database(environment: str = "development"):
    """
    Connect to the appropriate database.

    Args:
        environment: "production" or "development"

    Returns:
        Database connection

    Raises:
        ValueError: If environment is invalid
    """
    if environment == "production":
        db_path = PROD_DB_PATH
    elif environment == "development":
        db_path = DEV_DB_PATH
    else:
        raise ValueError(f"Invalid environment: {environment}")

    if not db_path.exists():
        raise FileNotFoundError(f"Database not found: {db_path}")

    conn = sqlite3.connect(db_path)

    # Verify we're connected to the right environment
    actual_env = verify_environment(conn)
    if actual_env != environment:
        conn.close()
        raise RuntimeError(
            f"Environment mismatch! "
            f"Requested {environment} but connected to {actual_env}"
        )

    print(f"✓ Connected to {environment} database")
    return conn


def safe_query_example():
    """Example of safely querying the database."""

    # Always specify environment explicitly
    with get_connection(DEV_DB_PATH) as conn:
        # Verify environment before making changes
        env = verify_environment(conn)
        print(f"Connected to: {env}")

        if env != "development":
            raise RuntimeError(f"Expected development, got {env}!")

        # Safe to make changes now
        cursor = conn.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS test (
                id INTEGER PRIMARY KEY,
                data TEXT
            )
        """)
        conn.commit()
        print("✓ Test table created in development")


def production_readonly_example():
    """Example of reading from production safely."""

    # Open production in read-only mode
    with get_connection(PROD_DB_PATH, readonly=True) as conn:
        cursor = conn.cursor()

        # This works (read operation)
        cursor.execute("SELECT COUNT(*) FROM vessel_characteristics")
        count = cursor.fetchone()[0]
        print(f"Production has {count} vessels")

        # This will fail (write operation on read-only connection)
        try:
            cursor.execute("DELETE FROM vessel_characteristics")
            print("❌ This shouldn't happen!")
        except sqlite3.OperationalError as e:
            print(f"✓ Write blocked on read-only connection: {e}")


if __name__ == "__main__":
    print("Example 1: Safe query with environment check")
    safe_query_example()

    print("\nExample 2: Read-only production access")
    production_readonly_example()
```

---

## Backup Strategies

### Strategy 1: Simple File Copy (Full Backup)

Create `backups/scripts/backup_prod_simple.sh`:

```bash
#!/bin/bash
# Simple full backup of production database

PROD_DB="/home/gitgud/Projects/sql-revision/data/prod/vessel_analytics.db"
BACKUP_DIR="/home/gitgud/Projects/sql-revision/backups/prod/daily"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/vessel_analytics_$DATE.db"

echo "=== Production Database Backup ==="
echo "Source: $PROD_DB"
echo "Destination: $BACKUP_FILE"
echo ""

# Ensure backup directory exists
mkdir -p "$BACKUP_DIR"

# Method 1: SQLite backup command (hot backup - safe while DB is in use)
sqlite3 "$PROD_DB" ".backup '$BACKUP_FILE'"

if [ $? -eq 0 ]; then
    echo "✓ Backup completed successfully"

    # Verify integrity
    INTEGRITY=$(sqlite3 "$BACKUP_FILE" "PRAGMA integrity_check;")
    if [ "$INTEGRITY" = "ok" ]; then
        echo "✓ Integrity check passed"
    else
        echo "❌ Integrity check failed: $INTEGRITY"
        exit 1
    fi

    # Show backup info
    SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    echo "✓ Backup size: $SIZE"

    # Keep only last 7 days
    find "$BACKUP_DIR" -name "*.db" -mtime +7 -delete
    echo "✓ Old backups cleaned up (kept last 7 days)"
else
    echo "❌ Backup failed"
    exit 1
fi

echo "=== Backup Complete ==="
```

Make it executable:

```bash
chmod +x backups/scripts/backup_prod_simple.sh
```

### Strategy 2: Python Backup Script

Create `scripts/backup_prod.py`:

```python
"""
Backup production database.
"""
import sqlite3
from pathlib import Path
from datetime import datetime, timedelta

from db_config import PROD_DB_PATH, DAILY_BACKUP_PATH
from db_utils import get_connection


def backup_database(keep_days: int = 7):
    """
    Create a backup of the production database.

    Args:
        keep_days: Number of days to keep old backups
    """
    if not PROD_DB_PATH.exists():
        print(f"❌ Production database not found: {PROD_DB_PATH}")
        return

    # Create backup filename with timestamp
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_file = DAILY_BACKUP_PATH / f"vessel_analytics_{timestamp}.db"

    print(f"=== Backup Production Database ===")
    print(f"Source: {PROD_DB_PATH}")
    print(f"Destination: {backup_file}")
    print()

    # Perform hot backup using SQLite backup API
    with get_connection(PROD_DB_PATH, readonly=True) as src_conn:
        with sqlite3.connect(backup_file) as backup_conn:
            src_conn.backup(backup_conn)

    print(f"✓ Backup created")

    # Verify integrity
    with sqlite3.connect(backup_file) as conn:
        cursor = conn.cursor()
        cursor.execute("PRAGMA integrity_check;")
        result = cursor.fetchone()[0]

        if result == "ok":
            print(f"✓ Integrity check passed")
        else:
            print(f"❌ Integrity check failed: {result}")
            return

    # Show backup size
    size_mb = backup_file.stat().st_size / (1024 * 1024)
    print(f"✓ Backup size: {size_mb:.2f} MB")

    # Clean up old backups
    cutoff_date = datetime.now() - timedelta(days=keep_days)
    deleted_count = 0

    for old_backup in DAILY_BACKUP_PATH.glob("*.db"):
        if datetime.fromtimestamp(old_backup.stat().st_mtime) < cutoff_date:
            old_backup.unlink()
            deleted_count += 1

    if deleted_count > 0:
        print(f"✓ Cleaned up {deleted_count} old backups (kept last {keep_days} days)")

    print(f"\n=== Backup Complete ===")


if __name__ == "__main__":
    backup_database()
```

### Strategy 3: Incremental Backup with WAL Files

SQLite's WAL (Write-Ahead Log) can be used for incremental backups:

Create `scripts/backup_with_wal.py`:

```python
"""
Incremental backup using WAL files for point-in-time recovery.
"""
import shutil
from pathlib import Path
from datetime import datetime

from db_config import PROD_DB_PATH, WAL_ARCHIVE_PATH, SNAPSHOT_PATH


def create_snapshot():
    """
    Create a full snapshot of the database.
    This is the base for incremental backups.
    """
    if not PROD_DB_PATH.exists():
        print(f"❌ Production database not found: {PROD_DB_PATH}")
        return None

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    snapshot_file = SNAPSHOT_PATH / f"snapshot_{timestamp}.db"

    print(f"Creating snapshot: {snapshot_file}")

    # Force checkpoint to ensure all WAL changes are in main DB
    import sqlite3
    with sqlite3.connect(PROD_DB_PATH) as conn:
        conn.execute("PRAGMA wal_checkpoint(TRUNCATE);")

    # Copy database file
    shutil.copy2(PROD_DB_PATH, snapshot_file)

    print(f"✓ Snapshot created: {snapshot_file}")
    return snapshot_file


def archive_wal_file():
    """
    Archive current WAL file for incremental backup.

    This allows point-in-time recovery between snapshots.
    """
    wal_file = Path(str(PROD_DB_PATH) + "-wal")

    if not wal_file.exists():
        print("No WAL file to archive (database might not be in WAL mode)")
        return

    # Check if WAL file has content
    if wal_file.stat().st_size == 0:
        print("WAL file is empty, nothing to archive")
        return

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    archived_wal = WAL_ARCHIVE_PATH / f"wal_{timestamp}.wal"

    # Force checkpoint to create new WAL
    import sqlite3
    with sqlite3.connect(PROD_DB_PATH) as conn:
        conn.execute("PRAGMA wal_checkpoint(RESTART);")

    # Copy WAL file
    shutil.copy2(wal_file, archived_wal)

    print(f"✓ WAL archived: {archived_wal}")
    return archived_wal


def setup_pitr_backup():
    """
    Set up point-in-time recovery backup strategy.

    Strategy:
    1. Create full snapshot daily
    2. Archive WAL files hourly
    3. To recover: restore snapshot + replay WAL files
    """
    print("="*60)
    print("POINT-IN-TIME RECOVERY SETUP")
    print("="*60)

    # Create initial snapshot
    snapshot = create_snapshot()

    # Archive current WAL
    wal = archive_wal_file()

    print("\n" + "="*60)
    print("Setup complete!")
    print("\nRecovery process:")
    print("  1. Start with snapshot (full database)")
    print("  2. Apply WAL files in order")
    print("  3. Replay to desired point in time")
    print("\nAutomation:")
    print("  - Run create_snapshot() daily (cron)")
    print("  - Run archive_wal_file() hourly (cron)")
    print("="*60)


if __name__ == "__main__":
    setup_pitr_backup()
```

---

## Point-in-Time Recovery for SQLite

### Understanding SQLite WAL Mode

SQLite's Write-Ahead Logging (WAL) mode:

```
┌─────────────────────────────────────────┐
│   vessel_analytics.db (Main database)  │
│   - Contains committed transactions     │
│   - Updated during checkpoints          │
└─────────────────────────────────────────┘
           ↑
           │ Checkpoint (merge WAL into main)
           │
┌─────────────────────────────────────────┐
│   vessel_analytics.db-wal (WAL file)   │
│   - All recent writes go here first    │
│   - Provides crash recovery             │
│   - Can be archived for PITR            │
└─────────────────────────────────────────┘
```

### PITR Implementation

Create `scripts/restore_pitr.py`:

```python
"""
Point-in-time recovery for SQLite using snapshots and WAL archives.
"""
import sqlite3
import shutil
from pathlib import Path
from datetime import datetime

from db_config import SNAPSHOT_PATH, WAL_ARCHIVE_PATH


def list_snapshots():
    """List available snapshots."""
    snapshots = sorted(SNAPSHOT_PATH.glob("snapshot_*.db"))

    if not snapshots:
        print("No snapshots available")
        return []

    print("\nAvailable Snapshots:")
    print("="*60)
    for i, snapshot in enumerate(snapshots, 1):
        timestamp_str = snapshot.stem.replace("snapshot_", "")
        timestamp = datetime.strptime(timestamp_str, "%Y%m%d_%H%M%S")
        size_mb = snapshot.stat().st_size / (1024 * 1024)
        print(f"{i}. {timestamp} ({size_mb:.2f} MB)")
    print("="*60)

    return snapshots


def list_wal_archives():
    """List available WAL archives."""
    wals = sorted(WAL_ARCHIVE_PATH.glob("wal_*.wal"))

    if not wals:
        print("No WAL archives available")
        return []

    print("\nAvailable WAL Archives:")
    print("="*60)
    for i, wal in enumerate(wals, 1):
        timestamp_str = wal.stem.replace("wal_", "")
        timestamp = datetime.strptime(timestamp_str, "%Y%m%d_%H%M%S")
        size_kb = wal.stat().st_size / 1024
        print(f"{i}. {timestamp} ({size_kb:.2f} KB)")
    print("="*60)

    return wals


def restore_to_snapshot(snapshot_path: Path, output_path: Path):
    """
    Restore database from a snapshot.

    Args:
        snapshot_path: Path to snapshot file
        output_path: Path for restored database
    """
    print(f"\nRestoring from snapshot: {snapshot_path}")
    shutil.copy2(snapshot_path, output_path)

    # Verify integrity
    with sqlite3.connect(output_path) as conn:
        cursor = conn.cursor()
        cursor.execute("PRAGMA integrity_check;")
        result = cursor.fetchone()[0]

        if result == "ok":
            print(f"✓ Database restored and verified")
        else:
            print(f"❌ Integrity check failed: {result}")
            return False

    return True


def restore_with_wal_replay(snapshot_path: Path, wal_archives: list,
                            output_path: Path, stop_at: datetime = None):
    """
    Restore database from snapshot and replay WAL files.

    Args:
        snapshot_path: Path to base snapshot
        wal_archives: List of WAL files to replay
        output_path: Path for restored database
        stop_at: Stop replaying at this timestamp
    """
    print(f"\nPoint-in-Time Recovery")
    print(f"="*60)

    # Step 1: Restore from snapshot
    print(f"\n[1/2] Restoring base snapshot...")
    if not restore_to_snapshot(snapshot_path, output_path):
        return False

    # Step 2: Replay WAL files
    print(f"\n[2/2] Replaying WAL archives...")

    for wal_path in wal_archives:
        # Check timestamp
        timestamp_str = wal_path.stem.replace("wal_", "")
        wal_timestamp = datetime.strptime(timestamp_str, "%Y%m%d_%H%M%S")

        if stop_at and wal_timestamp > stop_at:
            print(f"  Stopped at {wal_timestamp} (after requested time)")
            break

        print(f"  Replaying: {wal_path.name} ({wal_timestamp})")

        # To replay WAL: copy it as the WAL file for the restored database
        restored_wal = Path(str(output_path) + "-wal")
        shutil.copy2(wal_path, restored_wal)

        # Open database to force WAL integration
        with sqlite3.connect(output_path) as conn:
            conn.execute("PRAGMA wal_checkpoint(FULL);")

        # Remove WAL file after integration
        restored_wal.unlink(missing_ok=True)

    print(f"\n✓ Point-in-time recovery complete")
    print(f"✓ Restored database: {output_path}")

    return True


def interactive_restore():
    """Interactive PITR restoration."""
    print("="*60)
    print("SQLITE POINT-IN-TIME RECOVERY")
    print("="*60)

    # List available snapshots
    snapshots = list_snapshots()
    if not snapshots:
        return

    # Select snapshot
    choice = input("\nSelect snapshot number: ")
    try:
        snapshot_idx = int(choice) - 1
        snapshot_path = snapshots[snapshot_idx]
    except (ValueError, IndexError):
        print("Invalid selection")
        return

    # List WAL archives
    wal_archives = list_wal_archives()

    # Ask if user wants to replay WALs
    if wal_archives:
        replay = input("\nReplay WAL archives for point-in-time recovery? (yes/no): ")
        use_wal = replay.lower() == 'yes'
    else:
        use_wal = False

    # Get output path
    output_path = Path(input("\nOutput path for restored database: "))

    # Perform restoration
    if use_wal:
        # TODO: Ask for stop time
        restore_with_wal_replay(snapshot_path, wal_archives, output_path)
    else:
        restore_to_snapshot(snapshot_path, output_path)


if __name__ == "__main__":
    interactive_restore()
```

---

## Data Migration: PostgreSQL to SQLite

If you have data in PostgreSQL and want to migrate to SQLite:

Create `scripts/migrate_pg_to_sqlite.py`:

```python
"""
Migrate data from PostgreSQL to SQLite.
"""
import psycopg
import sqlite3
from pathlib import Path

# PostgreSQL connection
PG_PARAMS = {
    "dbname": "vessel_analytics",
    "user": "gitgud",
    "host": "localhost",
    "port": 5432,
}

# SQLite path
SQLITE_PATH = Path("data/prod/vessel_analytics.db")


def migrate_table(pg_conn, sqlite_conn, table_name: str):
    """
    Migrate a single table from PostgreSQL to SQLite.

    Args:
        pg_conn: PostgreSQL connection
        sqlite_conn: SQLite connection
        table_name: Name of table to migrate
    """
    print(f"\nMigrating table: {table_name}")

    # Get data from PostgreSQL
    with pg_conn.cursor() as pg_cur:
        pg_cur.execute(f"SELECT * FROM {table_name}")
        rows = pg_cur.fetchall()

        if not rows:
            print(f"  No data to migrate")
            return

        # Get column names
        column_names = [desc[0] for desc in pg_cur.description]

    # Insert into SQLite
    sqlite_cur = sqlite_conn.cursor()
    placeholders = ','.join(['?' for _ in column_names])
    insert_sql = f"INSERT INTO {table_name} ({','.join(column_names)}) VALUES ({placeholders})"

    sqlite_cur.executemany(insert_sql, rows)
    sqlite_conn.commit()

    print(f"  ✓ Migrated {len(rows)} rows")


def migrate_all():
    """Migrate all tables from PostgreSQL to SQLite."""
    print("="*60)
    print("MIGRATE POSTGRESQL TO SQLITE")
    print("="*60)

    # Ensure SQLite database exists with schema
    if not SQLITE_PATH.exists():
        print(f"❌ SQLite database not found: {SQLITE_PATH}")
        print(f"   Run: python scripts/init_prod.py")
        return

    # Connect to both databases
    with psycopg.connect(**PG_PARAMS) as pg_conn:
        with sqlite3.connect(SQLITE_PATH) as sqlite_conn:
            # Migrate tables
            tables = ["vessel_characteristics", "raw_vessel_events"]

            for table in tables:
                migrate_table(pg_conn, sqlite_conn, table)

    print(f"\n" + "="*60)
    print(f"✓ Migration complete")
    print(f"="*60)


if __name__ == "__main__":
    migrate_all()
```

---

## Best Practices

### 1. Always Enable WAL Mode

```python
def enable_wal_mode(db_path: Path):
    """Enable WAL mode for better concurrency."""
    with sqlite3.connect(db_path) as conn:
        conn.execute("PRAGMA journal_mode=WAL;")
```

**Benefits:**
- Better concurrency (readers don't block writers)
- Faster writes
- Crash safety
- Can archive WAL for PITR

### 2. Use Context Managers

```python
# Good - automatic cleanup
with get_connection(db_path) as conn:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM table")

# Bad - manual cleanup, risk of leaks
conn = sqlite3.connect(db_path)
cursor = conn.cursor()
cursor.execute("SELECT * FROM table")
conn.close()  # Might not be called if exception
```

### 3. Always Verify Environment

```python
def dangerous_operation(conn):
    """Perform operation that modifies data."""
    # ALWAYS check environment first
    env = verify_environment(conn)
    if env != "development":
        raise RuntimeError(f"Cannot run on {env}!")

    # Safe to proceed
    conn.execute("DELETE FROM test_table")
```

### 4. Use Read-Only Connections for Production

```python
# Open production in read-only mode
with get_connection(PROD_DB_PATH, readonly=True) as conn:
    # Can only read, cannot write
    results = conn.execute("SELECT * FROM vessels").fetchall()
```

### 5. Regular Backups

```bash
# Daily full backup (cron)
0 2 * * * /usr/bin/python3 /home/gitgud/Projects/sql-revision/scripts/backup_prod.py

# Hourly WAL archive (for PITR)
0 * * * * /usr/bin/python3 /home/gitgud/Projects/sql-revision/scripts/backup_with_wal.py
```

### 6. Test Restore Procedures

```python
def test_backup_restore():
    """Regularly test that backups can be restored."""
    # Get latest backup
    backups = sorted(DAILY_BACKUP_PATH.glob("*.db"))
    if not backups:
        print("No backups to test")
        return

    latest_backup = backups[-1]

    # Try to open and query
    with sqlite3.connect(latest_backup) as conn:
        cursor = conn.cursor()

        # Check integrity
        cursor.execute("PRAGMA integrity_check;")
        assert cursor.fetchone()[0] == "ok"

        # Check can query data
        cursor.execute("SELECT COUNT(*) FROM vessel_characteristics")
        count = cursor.fetchone()[0]
        assert count > 0

    print(f"✓ Backup {latest_backup.name} is valid")
```

### 7. Monitor Database Size

```python
def check_db_size():
    """Check if database is getting too large."""
    MAX_SIZE_MB = 10000  # 10 GB warning threshold

    if PROD_DB_PATH.exists():
        size_mb = PROD_DB_PATH.stat().st_size / (1024 * 1024)
        print(f"Production database: {size_mb:.2f} MB")

        if size_mb > MAX_SIZE_MB:
            print(f"⚠️  Database exceeds {MAX_SIZE_MB} MB")
            print(f"   Consider:")
            print(f"   - Archiving old data")
            print(f"   - Running VACUUM")
            print(f"   - Migrating to PostgreSQL")
```

---

## Troubleshooting

### Issue: Database is Locked

**Symptom:**
```
sqlite3.OperationalError: database is locked
```

**Causes:**
- Another process has the database open for writing
- WAL mode not enabled
- Long-running transaction

**Solutions:**

1. Enable WAL mode:
```python
conn.execute("PRAGMA journal_mode=WAL;")
```

2. Increase timeout:
```python
conn = sqlite3.connect(db_path, timeout=30.0)
```

3. Find locking process:
```bash
lsof | grep vessel_analytics.db
```

### Issue: Disk Space

**Symptom:**
```
sqlite3.OperationalError: disk I/O error
```

**Solutions:**

1. Check disk space:
```bash
df -h
```

2. Run VACUUM to reclaim space:
```python
conn.execute("VACUUM;")
```

3. Clean up old backups:
```bash
find backups/prod/daily/ -name "*.db" -mtime +7 -delete
```

### Issue: Corruption

**Symptom:**
```
database disk image is malformed
```

**Solutions:**

1. Check integrity:
```python
cursor.execute("PRAGMA integrity_check;")
print(cursor.fetchone()[0])
```

2. Attempt recovery:
```bash
# Dump to SQL
sqlite3 corrupted.db ".dump" > dump.sql

# Create new database
sqlite3 recovered.db < dump.sql
```

3. Restore from backup:
```python
# Use latest backup
from scripts.restore_pitr import restore_to_snapshot
```

---

## Summary

### SQLite Production/Development Setup:

✅ **Simple:**
- No server installation
- Databases are just files
- Copy file = copy database

✅ **Safe:**
- Separate files = complete isolation
- Read-only mode prevents accidents
- Environment markers for double-checking

✅ **Flexible:**
- Full backups (simple copy)
- Incremental backups (WAL archiving)
- Point-in-time recovery possible

### Key Commands:

```bash
# Initialize
python scripts/init_prod.py

# Refresh dev from prod
python scripts/refresh_dev_from_prod.py

# Backup production
python scripts/backup_prod.py

# Restore PITR
python scripts/restore_pitr.py
```

### Safety Guarantees:

✅ **Production database is safe because:**
- Development uses a different file
- Can open production read-only
- File copy operations are safe
- Environment markers prevent mistakes

### When to Use SQLite vs PostgreSQL:

**Use SQLite for:**
- Single-user applications
- Embedded applications
- Mobile apps
- Development/testing
- Datasets < 1 TB
- Mostly reads, occasional writes

**Use PostgreSQL for:**
- Multi-user applications
- High write concurrency
- Distributed systems
- Advanced features (replication, partitioning)
- Datasets > 1 TB
- Production web applications

---

**Last Updated:** 2025-11-22

This setup provides a production-ready SQLite environment with complete separation between production and development, comprehensive backup strategies, and point-in-time recovery capabilities.
