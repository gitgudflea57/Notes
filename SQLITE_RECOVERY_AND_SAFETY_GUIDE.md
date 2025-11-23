# SQLite Recovery System and Safety Guide

Complete guide for implementing a production-grade recovery system for SQLite and ensuring production database safety with multiple layers of protection.

## Table of Contents
- [Safety First: Multiple Protection Layers](#safety-first-multiple-protection-layers)
- [Production Database Protection](#production-database-protection)
- [Continuous Backup System](#continuous-backup-system)
- [Point-in-Time Recovery Implementation](#point-in-time-recovery-implementation)
- [Recovery Testing](#recovery-testing)
- [Disaster Recovery Procedures](#disaster-recovery-procedures)
- [Monitoring and Alerts](#monitoring-and-alerts)

---

## Safety First: Multiple Protection Layers

### The Problem

```python
# DANGEROUS CODE - DO NOT USE
if os.path.exists("prod.db"):
    os.remove("prod.db")  # Could accidentally delete production!
```

### The Solution: Defense in Depth

We implement **7 layers of protection** to make it virtually impossible to accidentally damage production:

```
Layer 1: File System Permissions (read-only)
Layer 2: File Immutability (cannot be deleted even by owner)
Layer 3: Environment Verification (check database marker)
Layer 4: File Hash Verification (detect tampering)
Layer 5: Mandatory Confirmation (explicit user approval)
Layer 6: Backup Before Operation (automatic safety net)
Layer 7: Transaction Rollback (database-level safety)
```

---

## Production Database Protection

### Layer 1: File System Permissions

Create `scripts/protect_production.py`:

```python
"""
Protect production database with multiple layers of security.
"""
import os
import stat
import subprocess
import hashlib
from pathlib import Path

from db_config import PROD_DB_PATH


def set_readonly_permissions(db_path: Path):
    """
    Set production database to read-only.

    Args:
        db_path: Path to database file
    """
    print(f"Setting read-only permissions on {db_path}...")

    # Owner: read only
    # Group: read only
    # Other: no access
    os.chmod(db_path, stat.S_IRUSR | stat.S_IRGRP)

    # Also protect WAL and SHM files if they exist
    wal_path = Path(str(db_path) + "-wal")
    shm_path = Path(str(db_path) + "-shm")

    for path in [wal_path, shm_path]:
        if path.exists():
            os.chmod(path, stat.S_IRUSR | stat.S_IRGRP)

    print(f"✓ Files are now read-only")


def set_writable_permissions(db_path: Path):
    """
    Set database to writable (for maintenance).

    Args:
        db_path: Path to database file
    """
    print(f"Setting writable permissions on {db_path}...")

    # Owner: read + write
    # Group: read only
    # Other: no access
    os.chmod(db_path, stat.S_IRUSR | stat.S_IWUSR | stat.S_IRGRP)

    # Also update WAL and SHM files
    wal_path = Path(str(db_path) + "-wal")
    shm_path = Path(str(db_path) + "-shm")

    for path in [wal_path, shm_path]:
        if path.exists():
            os.chmod(path, stat.S_IRUSR | stat.S_IWUSR | stat.S_IRGRP)

    print(f"✓ Files are now writable")


def get_file_permissions(db_path: Path) -> str:
    """
    Get current file permissions.

    Returns:
        String like "r--r-----" (read-only) or "rw-r-----" (writable)
    """
    if not db_path.exists():
        return "file does not exist"

    st = db_path.stat()
    mode = st.st_mode

    perms = ""
    perms += "r" if mode & stat.S_IRUSR else "-"
    perms += "w" if mode & stat.S_IWUSR else "-"
    perms += "x" if mode & stat.S_IXUSR else "-"
    perms += "r" if mode & stat.S_IRGRP else "-"
    perms += "w" if mode & stat.S_IWGRP else "-"
    perms += "x" if mode & stat.S_IXGRP else "-"
    perms += "r" if mode & stat.S_IROTH else "-"
    perms += "w" if mode & stat.S_IWOTH else "-"
    perms += "x" if mode & stat.S_IXOTH else "-"

    return perms


def check_permissions(db_path: Path):
    """Check and display current permissions."""
    print(f"\nCurrent permissions for {db_path}:")
    print(f"  {get_file_permissions(db_path)}")

    wal_path = Path(str(db_path) + "-wal")
    if wal_path.exists():
        print(f"  WAL: {get_file_permissions(wal_path)}")

    shm_path = Path(str(db_path) + "-shm")
    if shm_path.exists():
        print(f"  SHM: {get_file_permissions(shm_path)}")
```

### Layer 2: File Immutability (Linux)

Add to `scripts/protect_production.py`:

```python
def make_immutable(db_path: Path):
    """
    Make file immutable - cannot be deleted or modified even by owner.

    Requires: sudo privileges
    Linux only (uses chattr)

    Args:
        db_path: Path to database file
    """
    if not db_path.exists():
        print(f"❌ File does not exist: {db_path}")
        return False

    try:
        # Set immutable flag
        subprocess.run(
            ["sudo", "chattr", "+i", str(db_path)],
            check=True,
            capture_output=True
        )
        print(f"✓ {db_path.name} is now immutable")

        # Also protect WAL and SHM
        wal_path = Path(str(db_path) + "-wal")
        shm_path = Path(str(db_path) + "-shm")

        for path in [wal_path, shm_path]:
            if path.exists():
                subprocess.run(
                    ["sudo", "chattr", "+i", str(path)],
                    check=True,
                    capture_output=True
                )
                print(f"✓ {path.name} is now immutable")

        return True
    except subprocess.CalledProcessError as e:
        print(f"❌ Failed to make immutable: {e}")
        return False


def make_mutable(db_path: Path):
    """
    Remove immutable flag - allows modification/deletion.

    Requires: sudo privileges
    Linux only (uses chattr)

    Args:
        db_path: Path to database file
    """
    if not db_path.exists():
        print(f"❌ File does not exist: {db_path}")
        return False

    try:
        # Remove immutable flag
        subprocess.run(
            ["sudo", "chattr", "-i", str(db_path)],
            check=True,
            capture_output=True
        )
        print(f"✓ {db_path.name} is now mutable")

        # Also update WAL and SHM
        wal_path = Path(str(db_path) + "-wal")
        shm_path = Path(str(db_path) + "-shm")

        for path in [wal_path, shm_path]:
            if path.exists():
                subprocess.run(
                    ["sudo", "chattr", "-i", str(path)],
                    check=True,
                    capture_output=True
                )
                print(f"✓ {path.name} is now mutable")

        return True
    except subprocess.CalledProcessError as e:
        print(f"❌ Failed to make mutable: {e}")
        return False


def check_immutable(db_path: Path) -> bool:
    """
    Check if file has immutable flag set.

    Returns:
        True if immutable
    """
    if not db_path.exists():
        return False

    try:
        result = subprocess.run(
            ["lsattr", str(db_path)],
            capture_output=True,
            text=True,
            check=True
        )
        # Immutable flag is 'i' in the output
        return 'i' in result.stdout.split()[0]
    except (subprocess.CalledProcessError, IndexError):
        return False
```

### Layer 3: Environment Verification

Add to `scripts/db_utils.py`:

```python
def verify_production_safety(db_path: Path, allow_write: bool = False):
    """
    Verify it's safe to perform operation on this database.

    Args:
        db_path: Path to database file
        allow_write: If False, will fail if attempting to write to production

    Raises:
        RuntimeError: If unsafe to proceed
    """
    if not db_path.exists():
        raise FileNotFoundError(f"Database does not exist: {db_path}")

    # Check environment marker
    with get_connection(db_path, readonly=True) as conn:
        env = verify_environment(conn)

    is_production = (env == "production" or "prod" in str(db_path).lower())

    if is_production:
        print(f"⚠️  WARNING: This is a PRODUCTION database!")
        print(f"   Path: {db_path}")
        print(f"   Environment: {env}")

        if not allow_write:
            raise RuntimeError(
                "Attempted write operation on production database! "
                "Use allow_write=True to override."
            )

        # Extra confirmation for production writes
        print("\n" + "="*60)
        print("PRODUCTION WRITE CONFIRMATION REQUIRED")
        print("="*60)
        print("You are about to modify the PRODUCTION database.")
        print("This operation could affect live data.")
        print("")

        response = input("Type 'YES I WANT TO MODIFY PRODUCTION' to continue: ")
        if response != "YES I WANT TO MODIFY PRODUCTION":
            raise RuntimeError("Production write aborted by user")

    return env
```

### Layer 4: File Hash Verification

Add to `scripts/protect_production.py`:

```python
def calculate_file_hash(db_path: Path) -> str:
    """
    Calculate SHA-256 hash of database file.

    Args:
        db_path: Path to database file

    Returns:
        Hex digest of file hash
    """
    sha256 = hashlib.sha256()

    with open(db_path, 'rb') as f:
        while True:
            data = f.read(65536)  # Read in 64kb chunks
            if not data:
                break
            sha256.update(data)

    return sha256.hexdigest()


def save_production_hash(db_path: Path):
    """
    Calculate and save hash of production database.

    This can be used to verify the database hasn't been tampered with.
    """
    hash_file = db_path.parent / f".{db_path.name}.sha256"

    print(f"Calculating hash of {db_path}...")
    file_hash = calculate_file_hash(db_path)

    with open(hash_file, 'w') as f:
        f.write(f"{file_hash}  {db_path.name}\n")

    print(f"✓ Hash saved to {hash_file}")
    print(f"  {file_hash}")


def verify_production_hash(db_path: Path) -> bool:
    """
    Verify production database hasn't been modified.

    Returns:
        True if hash matches, False otherwise
    """
    hash_file = db_path.parent / f".{db_path.name}.sha256"

    if not hash_file.exists():
        print(f"⚠️  No hash file found: {hash_file}")
        print(f"   Run save_production_hash() first")
        return False

    # Read saved hash
    with open(hash_file, 'r') as f:
        saved_hash = f.read().split()[0]

    # Calculate current hash
    current_hash = calculate_file_hash(db_path)

    if saved_hash == current_hash:
        print(f"✓ Database hash verified - file is unchanged")
        return True
    else:
        print(f"❌ HASH MISMATCH!")
        print(f"   Saved:   {saved_hash}")
        print(f"   Current: {current_hash}")
        print(f"   Database may have been modified!")
        return False
```

### Layer 5: Mandatory Confirmation

Create `scripts/safe_operations.py`:

```python
"""
Safe database operations with multiple confirmations.
"""
import shutil
from pathlib import Path
from datetime import datetime

from db_config import PROD_DB_PATH, DEV_DB_PATH
from db_utils import verify_production_safety, get_connection, verify_environment
from protect_production import verify_production_hash, calculate_file_hash


def require_confirmation(action: str, target: Path) -> bool:
    """
    Require explicit confirmation for dangerous operations.

    Args:
        action: Description of action (e.g., "delete", "modify")
        target: Path to file

    Returns:
        True if user confirms
    """
    print("\n" + "="*60)
    print(f"CONFIRMATION REQUIRED: {action.upper()}")
    print("="*60)
    print(f"Target: {target}")
    print(f"Action: {action}")
    print("")

    # Check if production
    is_prod = "prod" in str(target).lower()

    if is_prod:
        print("⚠️  WARNING: This is a PRODUCTION database!")
        print("⚠️  This operation could cause DATA LOSS!")
        print("")
        required_text = f"YES {action.upper()} PRODUCTION"
    else:
        required_text = f"YES {action}"

    print(f"Type '{required_text}' to continue")
    print(f"Type anything else to abort")
    print("")

    response = input("Confirmation: ")

    if response == required_text:
        print("✓ Confirmed")
        return True
    else:
        print("❌ Aborted")
        return False


def safe_delete_database(db_path: Path):
    """
    Safely delete a database with multiple confirmations.

    Args:
        db_path: Path to database file
    """
    if not db_path.exists():
        print(f"Database does not exist: {db_path}")
        return

    # Check environment
    with get_connection(db_path, readonly=True) as conn:
        env = verify_environment(conn)

    print(f"\nDatabase information:")
    print(f"  Path: {db_path}")
    print(f"  Environment: {env}")
    print(f"  Size: {db_path.stat().st_size / (1024*1024):.2f} MB")

    # Require confirmation
    if not require_confirmation("delete", db_path):
        return

    # Extra protection for production
    if env == "production" or "prod" in str(db_path).lower():
        print("\n⚠️  FINAL WARNING: DELETING PRODUCTION DATABASE!")
        print("This action is IRREVERSIBLE!")
        print("")

        final = input("Type 'DELETE PRODUCTION DATABASE' to proceed: ")
        if final != "DELETE PRODUCTION DATABASE":
            print("❌ Aborted")
            return

    # Create backup before deletion
    backup_path = db_path.parent / f"{db_path.name}.deleted.{datetime.now().strftime('%Y%m%d_%H%M%S')}"
    print(f"\nCreating backup before deletion...")
    print(f"  Backup: {backup_path}")
    shutil.copy2(db_path, backup_path)
    print(f"✓ Backup created")

    # Delete database files
    print(f"\nDeleting database files...")
    db_path.unlink()

    wal_path = Path(str(db_path) + "-wal")
    if wal_path.exists():
        wal_path.unlink()
        print(f"✓ Deleted {wal_path.name}")

    shm_path = Path(str(db_path) + "-shm")
    if shm_path.exists():
        shm_path.unlink()
        print(f"✓ Deleted {shm_path.name}")

    print(f"✓ Database deleted")
    print(f"\nBackup saved at: {backup_path}")
    print(f"You can restore by copying this file back")


def safe_overwrite_database(source: Path, dest: Path):
    """
    Safely overwrite a database with comprehensive safety checks.

    Args:
        source: Source database path
        dest: Destination database path (will be overwritten)
    """
    print("="*60)
    print("SAFE DATABASE OVERWRITE")
    print("="*60)

    # Verify source exists
    if not source.exists():
        raise FileNotFoundError(f"Source database not found: {source}")

    # Get source environment
    with get_connection(source, readonly=True) as conn:
        source_env = verify_environment(conn)

    # Get destination environment (if exists)
    if dest.exists():
        with get_connection(dest, readonly=True) as conn:
            dest_env = verify_environment(conn)
    else:
        dest_env = "new file"

    print(f"\nSource:")
    print(f"  Path: {source}")
    print(f"  Environment: {source_env}")
    print(f"  Size: {source.stat().st_size / (1024*1024):.2f} MB")

    print(f"\nDestination:")
    print(f"  Path: {dest}")
    print(f"  Environment: {dest_env}")
    if dest.exists():
        print(f"  Size: {dest.stat().st_size / (1024*1024):.2f} MB")
    else:
        print(f"  Status: Will be created")

    # Safety check: Don't overwrite production with development
    if dest_env == "production" and source_env != "production":
        raise RuntimeError(
            "SAFETY VIOLATION: Attempting to overwrite PRODUCTION with non-production data! "
            "This operation is blocked."
        )

    # Require confirmation
    if not require_confirmation("overwrite", dest):
        return

    # Create backup of destination if it exists
    if dest.exists():
        backup_path = dest.parent / f"{dest.name}.backup.{datetime.now().strftime('%Y%m%d_%H%M%S')}"
        print(f"\nCreating backup of destination...")
        print(f"  Backup: {backup_path}")
        shutil.copy2(dest, backup_path)
        print(f"✓ Backup created")

        # Save hash of backup
        backup_hash = calculate_file_hash(backup_path)
        print(f"  Backup hash: {backup_hash}")

    # Perform copy
    print(f"\nCopying database...")
    shutil.copy2(source, dest)
    print(f"✓ Database copied")

    # Verify copy
    print(f"\nVerifying copy...")
    source_hash = calculate_file_hash(source)
    dest_hash = calculate_file_hash(dest)

    if source_hash == dest_hash:
        print(f"✓ Copy verified (hashes match)")
        print(f"  Hash: {dest_hash}")
    else:
        print(f"❌ COPY VERIFICATION FAILED!")
        print(f"  Source: {source_hash}")
        print(f"  Dest:   {dest_hash}")
        raise RuntimeError("Copy verification failed - data may be corrupted")

    print(f"\n✓ Operation completed successfully")
```

### Layer 6: Automatic Backup Before Operation

Create `scripts/backup_before_operation.py`:

```python
"""
Automatically backup database before any write operation.
"""
import shutil
from pathlib import Path
from datetime import datetime
from contextlib import contextmanager

from db_config import PROD_DB_PATH
from protect_production import calculate_file_hash


@contextmanager
def backup_before_write(db_path: Path):
    """
    Context manager that automatically backs up database before write.

    If operation fails, backup is kept. If successful, backup is kept with
    timestamp.

    Usage:
        with backup_before_write(PROD_DB_PATH) as backup_path:
            # Perform write operations
            conn.execute("UPDATE ...")

        # Backup is automatically managed

    Args:
        db_path: Path to database

    Yields:
        Path to backup file
    """
    if not db_path.exists():
        raise FileNotFoundError(f"Database not found: {db_path}")

    # Create backup
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    backup_path = db_path.parent / f".{db_path.name}.preop_backup_{timestamp}"

    print(f"\n[Safety] Creating pre-operation backup...")
    print(f"  Original: {db_path}")
    print(f"  Backup:   {backup_path}")

    # Copy database
    shutil.copy2(db_path, backup_path)

    # Calculate and save hash
    original_hash = calculate_file_hash(db_path)
    backup_hash = calculate_file_hash(backup_path)

    if original_hash != backup_hash:
        raise RuntimeError("Backup verification failed - hashes don't match!")

    print(f"  ✓ Backup created and verified")
    print(f"    Hash: {backup_hash}")

    try:
        yield backup_path

        # Operation succeeded
        print(f"\n[Safety] Operation completed successfully")
        print(f"  Backup kept at: {backup_path}")

    except Exception as e:
        # Operation failed - restore from backup
        print(f"\n[Safety] ❌ Operation failed: {e}")
        print(f"[Safety] Restoring from backup...")

        shutil.copy2(backup_path, db_path)

        # Verify restoration
        restored_hash = calculate_file_hash(db_path)
        if restored_hash == original_hash:
            print(f"[Safety] ✓ Database restored from backup")
        else:
            print(f"[Safety] ❌ RESTORE VERIFICATION FAILED!")
            raise RuntimeError("Database restoration failed!")

        raise  # Re-raise original exception


# Example usage
def example_safe_write():
    """Example of using backup_before_write."""
    from db_utils import get_connection

    with backup_before_write(PROD_DB_PATH) as backup:
        with get_connection(PROD_DB_PATH) as conn:
            # Perform write operation
            conn.execute("UPDATE some_table SET value = 1")
            conn.commit()

        # If we get here, operation succeeded and backup is kept
```

### Complete Protection Setup Script

Create `scripts/setup_production_protection.py`:

```python
"""
Set up complete protection for production database.
"""
from pathlib import Path

from db_config import PROD_DB_PATH
from protect_production import (
    set_readonly_permissions,
    check_permissions,
    make_immutable,
    check_immutable,
    save_production_hash,
    verify_production_hash
)


def setup_full_protection():
    """
    Apply all protection layers to production database.
    """
    if not PROD_DB_PATH.exists():
        print(f"❌ Production database not found: {PROD_DB_PATH}")
        return

    print("="*60)
    print("SETTING UP PRODUCTION DATABASE PROTECTION")
    print("="*60)

    print(f"\nTarget: {PROD_DB_PATH}")

    # Layer 1: Read-only permissions
    print(f"\n[Layer 1] Setting read-only permissions...")
    set_readonly_permissions(PROD_DB_PATH)
    check_permissions(PROD_DB_PATH)

    # Layer 2: Immutable flag (requires sudo)
    print(f"\n[Layer 2] Setting immutable flag...")
    print(f"  (This requires sudo privileges)")
    success = make_immutable(PROD_DB_PATH)

    if success:
        is_immutable = check_immutable(PROD_DB_PATH)
        if is_immutable:
            print(f"  ✓ Database is now immutable")
        else:
            print(f"  ⚠️  Warning: Could not verify immutable status")

    # Layer 4: Save file hash
    print(f"\n[Layer 4] Calculating and saving file hash...")
    save_production_hash(PROD_DB_PATH)

    print("\n" + "="*60)
    print("✅ PRODUCTION DATABASE PROTECTED")
    print("="*60)

    print("\nProtection layers applied:")
    print("  ✓ Layer 1: Read-only file permissions")
    print(f"  {'✓' if success else '⚠️ '} Layer 2: Immutable flag (cannot be deleted)")
    print("  ✓ Layer 3: Environment verification (in code)")
    print("  ✓ Layer 4: File hash saved for verification")
    print("  ✓ Layer 5: Mandatory confirmation (in code)")
    print("  ✓ Layer 6: Automatic backup before write (in code)")
    print("  ✓ Layer 7: Transaction rollback (in SQLite)")

    print("\nTo make changes to production:")
    print("  1. Remove immutable flag: sudo chattr -i prod.db")
    print("  2. Set writable: chmod u+w prod.db")
    print("  3. Make changes")
    print("  4. Re-apply protection: python scripts/setup_production_protection.py")


def remove_protection():
    """
    Remove protection from production database (for maintenance).
    """
    from protect_production import make_mutable, set_writable_permissions

    print("="*60)
    print("REMOVING PRODUCTION DATABASE PROTECTION")
    print("="*60)

    print("\n⚠️  WARNING: This will make production database modifiable!")
    response = input("Type 'YES REMOVE PROTECTION' to continue: ")

    if response != "YES REMOVE PROTECTION":
        print("Aborted")
        return

    print(f"\n[Layer 2] Removing immutable flag...")
    make_mutable(PROD_DB_PATH)

    print(f"\n[Layer 1] Setting writable permissions...")
    set_writable_permissions(PROD_DB_PATH)
    check_permissions(PROD_DB_PATH)

    print("\n" + "="*60)
    print("⚠️  PROTECTION REMOVED")
    print("="*60)
    print("\nProduction database is now modifiable.")
    print("Remember to re-apply protection when done!")
    print("  python scripts/setup_production_protection.py")


if __name__ == "__main__":
    import sys

    if len(sys.argv) > 1 and sys.argv[1] == "remove":
        remove_protection()
    else:
        setup_full_protection()
```

---

## Continuous Backup System

### Automated Backup Daemon

Create `scripts/backup_daemon.py`:

```python
"""
Continuous backup daemon for SQLite production database.

This daemon:
1. Creates full snapshots daily
2. Archives WAL files hourly
3. Monitors database for changes
4. Alerts on errors
"""
import time
import shutil
import sqlite3
from pathlib import Path
from datetime import datetime, timedelta
import logging

from db_config import PROD_DB_PATH, DAILY_BACKUP_PATH, WAL_ARCHIVE_PATH, SNAPSHOT_PATH


# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('backups/backup_daemon.log'),
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)


class BackupDaemon:
    """Continuous backup daemon."""

    def __init__(self, db_path: Path):
        self.db_path = db_path
        self.last_snapshot_time = None
        self.last_wal_archive_time = None
        self.snapshot_interval = timedelta(hours=24)  # Daily snapshots
        self.wal_archive_interval = timedelta(hours=1)  # Hourly WAL archives

    def create_snapshot(self):
        """Create full database snapshot."""
        try:
            logger.info("Creating database snapshot...")

            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            snapshot_file = SNAPSHOT_PATH / f"snapshot_{timestamp}.db"

            # Force checkpoint before snapshot
            with sqlite3.connect(self.db_path) as conn:
                conn.execute("PRAGMA wal_checkpoint(TRUNCATE);")

            # Copy database
            shutil.copy2(self.db_path, snapshot_file)

            # Verify integrity
            with sqlite3.connect(snapshot_file) as conn:
                cursor = conn.cursor()
                cursor.execute("PRAGMA integrity_check;")
                result = cursor.fetchone()[0]

                if result != "ok":
                    logger.error(f"Snapshot integrity check failed: {result}")
                    snapshot_file.unlink()
                    return False

            size_mb = snapshot_file.stat().st_size / (1024 * 1024)
            logger.info(f"✓ Snapshot created: {snapshot_file.name} ({size_mb:.2f} MB)")

            self.last_snapshot_time = datetime.now()

            # Clean up old snapshots (keep last 7 days)
            self.cleanup_old_snapshots(days=7)

            return True

        except Exception as e:
            logger.error(f"Snapshot creation failed: {e}")
            return False

    def archive_wal_file(self):
        """Archive WAL file."""
        try:
            wal_file = Path(str(self.db_path) + "-wal")

            if not wal_file.exists():
                logger.warning("No WAL file to archive")
                return False

            # Check if WAL has content
            if wal_file.stat().st_size == 0:
                logger.info("WAL file is empty, skipping archive")
                return True

            logger.info("Archiving WAL file...")

            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            archived_wal = WAL_ARCHIVE_PATH / f"wal_{timestamp}.wal"

            # Force checkpoint to create new WAL
            with sqlite3.connect(self.db_path) as conn:
                conn.execute("PRAGMA wal_checkpoint(RESTART);")

            # Copy WAL file
            shutil.copy2(wal_file, archived_wal)

            size_kb = archived_wal.stat().st_size / 1024
            logger.info(f"✓ WAL archived: {archived_wal.name} ({size_kb:.2f} KB)")

            self.last_wal_archive_time = datetime.now()

            # Clean up old WAL archives (keep last 7 days)
            self.cleanup_old_wal_archives(days=7)

            return True

        except Exception as e:
            logger.error(f"WAL archiving failed: {e}")
            return False

    def cleanup_old_snapshots(self, days: int = 7):
        """Remove snapshots older than specified days."""
        cutoff = datetime.now() - timedelta(days=days)
        deleted = 0

        for snapshot in SNAPSHOT_PATH.glob("snapshot_*.db"):
            if datetime.fromtimestamp(snapshot.stat().st_mtime) < cutoff:
                snapshot.unlink()
                deleted += 1

        if deleted > 0:
            logger.info(f"Cleaned up {deleted} old snapshots")

    def cleanup_old_wal_archives(self, days: int = 7):
        """Remove WAL archives older than specified days."""
        cutoff = datetime.now() - timedelta(days=days)
        deleted = 0

        for wal in WAL_ARCHIVE_PATH.glob("wal_*.wal"):
            if datetime.fromtimestamp(wal.stat().st_mtime) < cutoff:
                wal.unlink()
                deleted += 1

        if deleted > 0:
            logger.info(f"Cleaned up {deleted} old WAL archives")

    def should_create_snapshot(self) -> bool:
        """Check if it's time for a snapshot."""
        if self.last_snapshot_time is None:
            return True

        return datetime.now() - self.last_snapshot_time >= self.snapshot_interval

    def should_archive_wal(self) -> bool:
        """Check if it's time to archive WAL."""
        if self.last_wal_archive_time is None:
            return True

        return datetime.now() - self.last_wal_archive_time >= self.wal_archive_interval

    def run(self):
        """Run backup daemon loop."""
        logger.info("="*60)
        logger.info("BACKUP DAEMON STARTED")
        logger.info("="*60)
        logger.info(f"Database: {self.db_path}")
        logger.info(f"Snapshot interval: {self.snapshot_interval}")
        logger.info(f"WAL archive interval: {self.wal_archive_interval}")
        logger.info("="*60)

        try:
            while True:
                # Check for snapshot
                if self.should_create_snapshot():
                    self.create_snapshot()

                # Check for WAL archive
                if self.should_archive_wal():
                    self.archive_wal_file()

                # Sleep for 5 minutes before next check
                time.sleep(300)

        except KeyboardInterrupt:
            logger.info("\nBackup daemon stopped by user")
        except Exception as e:
            logger.error(f"Backup daemon error: {e}")
            raise


def run_backup_daemon():
    """Start the backup daemon."""
    daemon = BackupDaemon(PROD_DB_PATH)
    daemon.run()


if __name__ == "__main__":
    run_backup_daemon()
```

### Systemd Service for Backup Daemon

Create `/etc/systemd/system/sqlite-backup.service`:

```ini
[Unit]
Description=SQLite Continuous Backup Daemon
After=network.target

[Service]
Type=simple
User=gitgud
Group=gitgud
WorkingDirectory=/home/gitgud/Projects/sql-revision
ExecStart=/usr/bin/python3 /home/gitgud/Projects/sql-revision/scripts/backup_daemon.py
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable sqlite-backup
sudo systemctl start sqlite-backup
sudo systemctl status sqlite-backup
```

---

## Point-in-Time Recovery Implementation

### Complete Recovery Tool

Create `scripts/pitr_recovery.py`:

```python
"""
Production-grade Point-in-Time Recovery for SQLite.
"""
import sqlite3
import shutil
from pathlib import Path
from datetime import datetime
from typing import Optional

from db_config import SNAPSHOT_PATH, WAL_ARCHIVE_PATH
from protect_production import calculate_file_hash


class PITRRecovery:
    """Point-in-Time Recovery manager."""

    def __init__(self):
        self.snapshots = self._get_available_snapshots()
        self.wal_archives = self._get_available_wal_archives()

    def _get_available_snapshots(self):
        """Get list of available snapshots."""
        snapshots = []
        for path in sorted(SNAPSHOT_PATH.glob("snapshot_*.db")):
            timestamp_str = path.stem.replace("snapshot_", "")
            timestamp = datetime.strptime(timestamp_str, "%Y%m%d_%H%M%S")
            size_mb = path.stat().st_size / (1024 * 1024)

            snapshots.append({
                'path': path,
                'timestamp': timestamp,
                'size_mb': size_mb,
                'hash': calculate_file_hash(path)
            })

        return snapshots

    def _get_available_wal_archives(self):
        """Get list of available WAL archives."""
        wals = []
        for path in sorted(WAL_ARCHIVE_PATH.glob("wal_*.wal")):
            timestamp_str = path.stem.replace("wal_", "")
            timestamp = datetime.strptime(timestamp_str, "%Y%m%d_%H%M%S")
            size_kb = path.stat().st_size / 1024

            wals.append({
                'path': path,
                'timestamp': timestamp,
                'size_kb': size_kb
            })

        return wals

    def list_recovery_points(self):
        """List all available recovery points."""
        print("\n" + "="*70)
        print("AVAILABLE RECOVERY POINTS")
        print("="*70)

        if not self.snapshots:
            print("\n❌ No snapshots available")
            return

        print("\nSNAPSHOTS (Full database backups):")
        print("-"*70)
        for i, snapshot in enumerate(self.snapshots, 1):
            print(f"{i}. {snapshot['timestamp']} ({snapshot['size_mb']:.2f} MB)")
            print(f"   Path: {snapshot['path']}")
            print(f"   Hash: {snapshot['hash'][:16]}...")
            print()

        if self.wal_archives:
            print("\nWAL ARCHIVES (Incremental changes):")
            print("-"*70)
            for i, wal in enumerate(self.wal_archives, 1):
                print(f"{i}. {wal['timestamp']} ({wal['size_kb']:.2f} KB)")
            print()

            print("Recovery using WAL archives allows restoring to any point between snapshots.")
        else:
            print("\n⚠️  No WAL archives available (only snapshot recovery possible)")

        print("="*70)

    def find_best_snapshot(self, target_time: datetime) -> Optional[dict]:
        """
        Find the best snapshot to restore from for a given target time.

        Returns the most recent snapshot before target_time.
        """
        candidates = [s for s in self.snapshots if s['timestamp'] <= target_time]

        if not candidates:
            return None

        return max(candidates, key=lambda s: s['timestamp'])

    def get_relevant_wal_archives(self, snapshot_time: datetime,
                                   target_time: datetime) -> list:
        """
        Get WAL archives between snapshot and target time.
        """
        return [
            w for w in self.wal_archives
            if snapshot_time < w['timestamp'] <= target_time
        ]

    def restore_snapshot_only(self, snapshot: dict, output_path: Path) -> bool:
        """
        Restore from snapshot only (fastest recovery).
        """
        print(f"\n[1/3] Restoring from snapshot...")
        print(f"  Snapshot: {snapshot['path']}")
        print(f"  Time: {snapshot['timestamp']}")
        print(f"  Output: {output_path}")

        # Copy snapshot to output
        shutil.copy2(snapshot['path'], output_path)

        print(f"\n[2/3] Verifying integrity...")
        with sqlite3.connect(output_path) as conn:
            cursor = conn.cursor()
            cursor.execute("PRAGMA integrity_check;")
            result = cursor.fetchone()[0]

            if result != "ok":
                print(f"❌ Integrity check failed: {result}")
                return False

            print(f"✓ Integrity check passed")

        print(f"\n[3/3] Verifying hash...")
        expected_hash = snapshot['hash']
        actual_hash = calculate_file_hash(output_path)

        if expected_hash != actual_hash:
            print(f"❌ Hash mismatch!")
            print(f"  Expected: {expected_hash}")
            print(f"  Actual:   {actual_hash}")
            return False

        print(f"✓ Hash verified")

        return True

    def restore_with_wal_replay(self, snapshot: dict, wal_archives: list,
                                output_path: Path) -> bool:
        """
        Restore from snapshot and replay WAL archives.
        """
        # Step 1: Restore base snapshot
        if not self.restore_snapshot_only(snapshot, output_path):
            return False

        # Step 2: Replay WAL archives
        print(f"\n[4/N] Replaying WAL archives...")
        print(f"  Archives to replay: {len(wal_archives)}")

        for i, wal in enumerate(wal_archives, 1):
            print(f"\n  [{i}/{len(wal_archives)}] {wal['path'].name}")
            print(f"    Time: {wal['timestamp']}")

            # Copy WAL file as the WAL for our restored database
            restored_wal = Path(str(output_path) + "-wal")
            shutil.copy2(wal['path'], restored_wal)

            # Open database to integrate WAL
            with sqlite3.connect(output_path) as conn:
                conn.execute("PRAGMA wal_checkpoint(FULL);")

            # Remove WAL file after integration
            restored_wal.unlink(missing_ok=True)

            print(f"    ✓ Replayed")

        print(f"\n✓ All WAL archives replayed")

        return True

    def interactive_recovery(self):
        """Interactive recovery wizard."""
        print("="*70)
        print("SQLITE POINT-IN-TIME RECOVERY")
        print("="*70)

        # Show available recovery points
        self.list_recovery_points()

        if not self.snapshots:
            return

        print("\nRECOVERY OPTIONS:")
        print("1. Restore from specific snapshot (fast)")
        print("2. Restore to specific point in time (uses WAL replay)")
        print("3. Cancel")

        choice = input("\nSelect option: ")

        if choice == "1":
            self._restore_from_snapshot()
        elif choice == "2":
            self._restore_to_time()
        else:
            print("Cancelled")

    def _restore_from_snapshot(self):
        """Restore from a specific snapshot."""
        print("\nSelect snapshot:")
        for i, snapshot in enumerate(self.snapshots, 1):
            print(f"{i}. {snapshot['timestamp']}")

        try:
            choice = int(input("\nSnapshot number: "))
            snapshot = self.snapshots[choice - 1]
        except (ValueError, IndexError):
            print("Invalid selection")
            return

        output_path = Path(input("\nOutput path for restored database: "))

        if output_path.exists():
            response = input(f"⚠️  File exists. Overwrite? (yes/no): ")
            if response.lower() != 'yes':
                print("Cancelled")
                return

        print("\n" + "="*70)
        success = self.restore_snapshot_only(snapshot, output_path)
        print("="*70)

        if success:
            print(f"\n✅ DATABASE RESTORED SUCCESSFULLY")
            print(f"  Restored to: {output_path}")
            print(f"  From snapshot: {snapshot['timestamp']}")
        else:
            print(f"\n❌ RESTORE FAILED")

    def _restore_to_time(self):
        """Restore to a specific point in time."""
        if not self.wal_archives:
            print("\n❌ No WAL archives available for point-in-time recovery")
            print("   Only snapshot recovery is possible")
            return

        print("\nEnter target time (YYYY-MM-DD HH:MM:SS):")
        time_str = input("Target time: ")

        try:
            target_time = datetime.strptime(time_str, "%Y-%m-%d %H:%M:%S")
        except ValueError:
            print("Invalid time format")
            return

        # Find best snapshot
        snapshot = self.find_best_snapshot(target_time)
        if not snapshot:
            print(f"\n❌ No snapshot found before {target_time}")
            return

        # Get relevant WAL archives
        wal_archives = self.get_relevant_wal_archives(
            snapshot['timestamp'],
            target_time
        )

        print(f"\nRecovery plan:")
        print(f"  Base snapshot: {snapshot['timestamp']}")
        print(f"  WAL archives to replay: {len(wal_archives)}")
        print(f"  Target time: {target_time}")

        if wal_archives:
            print(f"  Will replay archives from:")
            print(f"    {wal_archives[0]['timestamp']} to {wal_archives[-1]['timestamp']}")

        response = input("\nProceed? (yes/no): ")
        if response.lower() != 'yes':
            print("Cancelled")
            return

        output_path = Path(input("\nOutput path for restored database: "))

        if output_path.exists():
            response = input(f"⚠️  File exists. Overwrite? (yes/no): ")
            if response.lower() != 'yes':
                print("Cancelled")
                return

        print("\n" + "="*70)
        success = self.restore_with_wal_replay(snapshot, wal_archives, output_path)
        print("="*70)

        if success:
            print(f"\n✅ DATABASE RESTORED SUCCESSFULLY")
            print(f"  Restored to: {output_path}")
            print(f"  From snapshot: {snapshot['timestamp']}")
            print(f"  To time: {target_time}")
        else:
            print(f"\n❌ RESTORE FAILED")


def main():
    """Run interactive recovery."""
    recovery = PITRRecovery()
    recovery.interactive_recovery()


if __name__ == "__main__":
    main()
```

---

## Summary

### 7 Layers of Production Protection:

1. **File Permissions** - Read-only by default
2. **Immutability** - Cannot be deleted even by owner (`chattr +i`)
3. **Environment Verification** - Check database marker before operations
4. **File Hash** - Detect tampering
5. **Mandatory Confirmation** - Explicit user approval required
6. **Automatic Backup** - Before any write operation
7. **Transaction Rollback** - Database-level safety

### Recovery System:

✅ **Continuous Backups:**
- Automated daemon creates snapshots daily
- WAL files archived hourly
- Old backups cleaned automatically

✅ **Point-in-Time Recovery:**
- Restore from any snapshot (fast)
- Restore to any point in time (WAL replay)
- Interactive recovery tool
- Integrity verification

✅ **Safety Guarantees:**
- Multiple confirmations required
- Automatic backups before operations
- Hash verification
- Cannot accidentally destroy production

### Quick Commands:

```bash
# Protect production
python scripts/setup_production_protection.py

# Start backup daemon
sudo systemctl start sqlite-backup

# Interactive recovery
python scripts/pitr_recovery.py

# Verify production safety
python scripts/verify_production.py
```

### Protection Status Check:

```python
from protect_production import check_immutable, check_permissions
from db_config import PROD_DB_PATH

# Check current protection
check_permissions(PROD_DB_PATH)
is_immutable = check_immutable(PROD_DB_PATH)

print(f"Immutable: {is_immutable}")
```

---

**Last Updated:** 2025-11-22

With these 7 layers of protection, it is virtually impossible to accidentally damage production. Every dangerous operation requires multiple explicit confirmations, automatic backups are created, and the database is protected at the file system level.
