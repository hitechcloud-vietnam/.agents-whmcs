# WHMCS Database Server Setup Workflow

## Description
Configure a dedicated database server for WHMCS to improve performance and scalability.

## Prerequisites
- Dedicated server for MySQL/MariaDB
- SSH access to both servers
- Root privileges
- Private network between servers (recommended)

## Architecture
```
[WHMCS App Server]  <--- Private Network --->  [MySQL Server]
    10.0.0.10                                    10.0.0.20
```

## Steps

### Step 1: Prepare Database Server
```bash
# Update system
apt update && apt upgrade -y

# Install MariaDB (recommended) or MySQL
# MariaDB
apt install -y mariadb-server mariadb-client

# Or MySQL 8.0
# apt install -y mysql-server mysql-client
```

### Step 2: Configure MySQL/MariaDB
```bash
# Edit my.cnf or create custom config
cat > /etc/mysql/mariadb.conf.d/99-whmcs.cnf << 'EOF'
[mysqld]
# Connection settings
max_connections = 500
wait_timeout = 600
interactive_timeout = 600
connect_timeout = 10

# Buffer settings
innodb_buffer_pool_size = 4G  # 70-80% of RAM
innodb_log_file_size = 1G
innodb_log_buffer_size = 64M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT

# Query cache (MariaDB 10.1+)
# query_cache_type = 1
# query_cache_size = 256M
# query_cache_limit = 2M

# Table settings
table_open_cache = 4000
table_definition_cache = 2000

# Temp tables
tmp_table_size = 256M
max_heap_table_size = 256M

# Logging
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
log_queries_not_using_indexes = 0

# Binary logging (for replication/backups)
log_bin = /var/log/mysql/mysql-bin.log
expire_logs_days = 7
max_binlog_size = 100M
binlog_format = row

# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
EOF

# Restart MySQL
systemctl restart mysql
```

### Step 3: Create WHMCS Database and User
```bash
mysql -u root -p << 'EOF'
-- Create database
CREATE DATABASE whmcs CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user with remote access
CREATE USER 'whmcs'@'10.0.0.10' IDENTIFIED BY 'strong_password_here';
CREATE USER 'whmcs'@'%' IDENTIFIED BY 'strong_password_here';

-- Grant privileges
GRANT ALL PRIVILEGES ON whmcs.* TO 'whmcs'@'10.0.0.10';
GRANT ALL PRIVILEGES ON whmcs.* TO 'whmcs'@'%';

-- Apply changes
FLUSH PRIVILEGES;

-- Show grants
SHOW GRANTS FOR 'whmcs'@'10.0.0.10';
EOF
```

### Step 4: Configure Firewall
```bash
# On database server - allow only WHMCS app server
ufw allow from 10.0.0.10 to any port 3306

# Or for all connections (use strong passwords)
ufw allow 3306/tcp
```

### Step 5: Optimize for Private Network
```bash
# On database server
cat >> /etc/mysql/mariadb.conf.d/99-whmcs.cnf << 'EOF'

# Network optimizations
skip-name-resolve
bind-address = 0.0.0.0
net_buffer_length = 16K
max_allowed_packet = 64M

# Connection pooling settings
thread_pool_size = 32
thread_pool_max_threads = 20000
thread_pool_stall_limit = 500
EOF

systemctl restart mysql
```

### Step 6: Update WHMCS Configuration
```php
// Update configuration.php on app server

$db_host = '10.0.0.20';  // Database server IP
$db_username = 'whmcs';
$db_password = 'strong_password_here';
$db_name = 'whmcs';

// Optional: Use socket for local connections (if same server)
// $db_host = 'localhost';
// $db_socket = '/var/run/mysqld/mysqld.sock';
```

### Step 7: Test Connection
```bash
# From WHMCS app server
apt install -y mysql-client

# Test connection
mysql -h 10.0.0.20 -u whmcs -p -e "SHOW DATABASES;"

# Test query performance
mysql -h 10.0.0.20 -u whmcs -p whmcs -e "SELECT COUNT(*) FROM tblclients;"
```

### Step 8: Import Data (if new install)
```bash
# Backup from old server
mysqldump -h old_host -u root -p whmcs > whmcs_database.sql

# Restore to new server
mysql -h 10.0.0.20 -u whmcs -p whmcs < whmcs_database.sql

# Verify import
mysql -h 10.0.0.20 -u whmcs -p whmcs -e "SELECT COUNT(*) FROM tblclients;"
```

### Step 9: Configure Backups
```bash
# On database server - create backup script
cat > /usr/local/bin/mysql-backup.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/backups/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
DB_NAME="whmcs"
DB_USER="whmcs"
DB_PASS="strong_password_here"
REMOTE_HOST="backup-server"
REMOTE_USER="backup"

mkdir -p $BACKUP_DIR

# Lock tables and backup
mysqldump -h 10.0.0.20 -u $DB_USER -p$DB_PASS \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    $DB_NAME | gzip > $BACKUP_DIR/mysql_${DATE}.sql.gz

# Verify backup
if [ $? -eq 0 ]; then
    echo "Backup completed: $DATE"
    # Upload to remote
    rsync -az $BACKUP_DIR/mysql_${DATE}.sql.gz $REMOTE_USER@$REMOTE_HOST:/backups/mysql/
    # Keep local for 7 days
    find $BACKUP_DIR -name "mysql_*.sql.gz" -mtime +7 -delete
else
    echo "Backup failed!"
    exit 1
fi
EOF

chmod +x /usr/local/bin/mysql-backup.sh

# Add to cron
echo "0 3 * * * /usr/local/bin/mysql-backup.sh" >> /etc/crontab
```

### Step 10: Setup Monitoring
```bash
# Create monitoring script
cat > /usr/local/bin/mysql-monitor.sh << 'EOF'
#!/bin/bash

ALERT_EMAIL="admin@example.com"
DB_HOST="10.0.0.20"

# Get MySQL status
STATUS=$(mysql -h $DB_HOST -u whmcs -p'password' -e "SHOW GLOBAL STATUS LIKE 'Threads_connected';" 2>/dev/null)

# Check connection count
CONNECTIONS=$(echo "$STATUS" | grep Threads_connected | awk '{print $2}')

if [ $CONNECTIONS -gt 450 ]; then
    echo "WARNING: High connection count: $CONNECTIONS" | \
        mail -s "MySQL Alert: High Connections" $ALERT_EMAIL
fi

# Check replication lag if applicable
# mysql -h $DB_HOST -e "SHOW SLAVE STATUS\G"

# Log to monitoring system
echo "$(date): Connections=$CONNECTIONS" >> /var/log/mysql_monitor.log
EOF
```

## Performance Tuning Tips

### For High-Traffic Sites
```bash
# Additional optimizations in my.cnf
cat >> /etc/mysql/mariadb.conf.d/99-whmcs.cnf << 'EOF'

# InnoDB settings for high traffic
innodb_read_io_threads = 16
innodb_write_io_threads = 16
innodb_io_capacity = 2000
innodb_io_capacity_max = 4000

# Connection handling
thread_cache_size = 50
sort_buffer_size = 4M
read_buffer_size = 4M
read_rnd_buffer_size = 8M
join_buffer_size = 4M
EOF

systemctl restart mysql
```

## Troubleshooting
```bash
# Check MySQL logs
tail -f /var/log/mysql/error.log
tail -f /var/log/mysql/slow.log

# Check connections
mysql -h 10.0.0.20 -u root -p -e "SHOW PROCESSLIST;"

# Check status
mysql -h 10.0.0.20 -u root -p -e "SHOW GLOBAL STATUS;"
mysql -h 10.0.0.20 -u root -p -e "SHOW ENGINE INNODB STATUS;"
```

## Tags
- database
- mysql
- mariadb
- performance
- dedicated-server