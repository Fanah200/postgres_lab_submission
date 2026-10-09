# Hands‑On Lab: Backups, Point‑in‑Time Recovery, and Replication

**Author:** Farah  
**Date:** October 9, 2026  
**Database System:** PostgreSQL  

---

## Task 1: Take and Verify a Logical Backup

### Objective
Create an isolated SQL-level structural copy of the `bootcamp` database, verify its contents, and test a restoration on a dummy database to ensure backup integrity.

### Execution Commands
```bash
# 1. Create a directory to house backups
mkdir -p ~/backups

# 2. Take a custom-formatted, compressed logical dump
pg_dump -Fc -f ~/backups/bootcamp.dump bootcamp

# 3. Read the internal Table of Contents to verify file integrity
pg_restore --list ~/backups/bootcamp.dump | head

# 4. Create a test cluster database and dry-run restoration
createdb bootcamp_check && pg_restore -d bootcamp_check ~/backups/bootcamp.dump
```

### Verification Output
```text
; Archive created by pg_dump version 15.4
; TOC Entries: 8
;
215; 1259 TABLE public students postgres
216; 1259 TABLE public courses postgres
```

---

## Task 2: Enable WAL Archiving

### Objective
Configure continuous archiving of transaction Write-Ahead Logs (WAL) to a secure destination directory and capture a physical block-level baseline backup (`pg_basebackup`).

### Configuration Changes (`postgresql.conf`)
The following rules were appended to the primary server initialization file:
```ini
wal_level = replica
archive_mode = on
archive_command = 'cp %p /home/postgres/backups/wal/%f'
```

### Execution Commands
```bash
# 1. Instantiate the WAL target repository
mkdir -p ~/backups/wal

# 2. Cycle service engine to instantiate changes
sudo systemctl restart postgresql

# 3. Create compressed physical base template
pg_basebackup -D ~/backups/base -Ft -z -Xs -P
```

---

## Task 3: Simulate a Disaster and Recover (PITR)

### Objective
Simulate a catastrophic data deletion event and use Point-in-Time Recovery to roll back the database cluster to the exact second before the incident.

### Disaster Simulation Log
1. **Target Point in Time Capture:** `2026-10-09 18:50:22.123456+03`
2. **Destructive Statement Executed:**
```sql
DELETE FROM students;
```

### Recovery Implementation Steps
1. Halted active instance: `sudo systemctl stop postgresql`
2. Purged corrupted runtime active data cluster directory.
3. Extracted baseline physical snapshot `~/backups/base/base.tar.gz` back into active directory.
4. Created an empty file named `recovery.signal` to initialize PITR mode.
5. Injected timeline target thresholds into configuration engine:
```ini
restore_command = 'cp /home/postgres/backups/wal/%f %p'
recovery_target_time = '2026-10-09 18:50:21' # 1 second prior to disaster
```
6. Re-instantiated system cluster: `sudo systemctl start postgresql`

### Validation Query Result
```sql
SELECT count(*) FROM students;
```
* **Output:** `50` (Data successfully recovered!)

---

## Task 4: Set Up a Streaming Standby & Replication Watch

### Objective
Establish a persistent replication path to feed transaction data blocks directly to a secondary, read-only slave node.

### System Configuration
* **Access Rules (`pg_hba.conf`):**
  ```text
  host replication replicator 127.0.0.1/32 md5
  ```
* **Replication Target Instantiation Command:**
  ```bash
  pg_basebackup -h 127.0.0.1 -U replicator -D ~/standby -R -P
  ```

### Live Replication Status Verification
Executed from Primary Node console:
```sql
SELECT application_name, state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

### Final Health Status
```text
 application_name |   state   | lag_bytes 
------------------+-----------+-----------
 walreceiver      | streaming |         0
```
---
