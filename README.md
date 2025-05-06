# 🛠️ Setting Up PostgreSQL Master-Replica Setup Locally

## Master Setup

### Step 1: Switch to the `postgres` User
```bash
sudo -i -u postgres
```

### Step 2: Create Cluster Directory Structure
```bash
mkdir -p ~/postgres_cluster/{master,replica1,replica2}
```

### Step 3: Initialize Master Database
```bash
initdb -D ~/postgres_cluster/master
```
**If `initdb` is not found:**
```bash
find /usr -name initdb 2>/dev/null
export PATH=/usr/lib/postgresql/14/bin:$PATH  # Temporary fix
```
To make it permanent:
```bash
echo 'export PATH=/usr/lib/postgresql/14/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

### Step 4: Configure `postgresql.conf` for Replication
```bash
nano ~/postgres_cluster/master/postgresql.conf
```
Add or update the following lines:
```conf
listen_addresses = 'localhost'
port = 5440  # or change to avoid port conflicts
wal_level = replica
max_wal_senders = 5
wal_keep_size = 64MB
hot_standby = on
```

### Step 5: Configure `pg_hba.conf` for Replication
```bash
nano ~/postgres_cluster/master/pg_hba.conf
```
Ensure this line is added:
```conf
host    replication     all     127.0.0.1/32     trust
```

### Step 6: Start the Master Server
```bash
pg_ctl -D ~/postgres_cluster/master -l ~/postgres_cluster/master/logfile start
```

**If it fails:**
- Check the logs:
  ```bash
  cat ~/postgres_cluster/master/logfile
  ```
- Common issue: port already in use. Change the port in `postgresql.conf` (e.g., to `5440`) and restart.

### Step 7: Connect to Master
```bash
psql -p 5440 -U postgres
```

### Step 8: Create Replication User
```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replica_pass';
```

---

## Replica1 Setup

### Step 1: Create Base Backup
```bash
pg_basebackup -h localhost -p 5440 -D ~/postgres_cluster/replica1 -U replicator -Fp -Xs -P -R
```

### Step 2: Configure Replica
```bash
nano ~/postgres_cluster/replica1/postgresql.conf
```
Add:
```conf
port = 5433
hot_standby = on
```

### Step 3: Fix Directory Permissions (if needed)
```bash
chmod 700 ~/postgres_cluster/replica1
chown -R postgres:postgres ~/postgres_cluster/replica1
chmod 700 ~/postgres_cluster
chmod 700 ~/postgres_cluster/*
```

### Step 4: Start Replica1
```bash
pg_ctl -D ~/postgres_cluster/replica1 -l ~/postgres_cluster/replica1/logfile start
```

### Step 5: Connect to Replica1
```bash
psql -p 5433 -U postgres
```

---

## Replica2 Setup

### Step 1: Create Base Backup
```bash
pg_basebackup -h localhost -p 5440 -D ~/postgres_cluster/replica2 -U replicator -Fp -Xs -P -R
```

### Step 2: Configure Replica
```bash
nano ~/postgres_cluster/replica2/postgresql.conf
```
Add:
```conf
port = 5434
hot_standby = on
```

### Step 3: Start Replica2
```bash
pg_ctl -D ~/postgres_cluster/replica2 -l ~/postgres_cluster/replica2/logfile start
```

### Step 4: Connect to Replica2
```bash
psql -p 5434 -U postgres
```

---

## ✅ Verify Replication

### On Master:
```sql
-- Check replication status
SELECT * FROM pg_stat_replication;

-- Test replication
CREATE TABLE test_replication(id SERIAL PRIMARY KEY, msg TEXT);
INSERT INTO test_replication(msg) VALUES ('Hello from master!');
```

This change should reflect on both replicas (`replica1` and `replica2`) when you query the `test_replication` table.

