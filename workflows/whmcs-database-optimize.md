# WHMCS Database Optimization Workflow

## Purpose
Optimize WHMCS database for performance

## Prerequisites
- SSH access
- MySQL root access

## Step 1: Check Database Size

```bash
mysql -u root -p -e "
SELECT 
    table_schema AS 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS 'Size (MB)'
FROM information_schema.tables 
WHERE table_schema = 'whmcs_db'
GROUP BY table_schema;"
```

## Step 2: Find Large Tables

```bash
mysql -u root -p -e "
SELECT 
    table_name AS 'Table',
    ROUND(data_length / 1024 / 1024, 2) AS 'Data (MB)',
    ROUND(index_length / 1024 / 1024, 2) AS 'Index (MB)',
    table_rows AS 'Rows'
FROM information_schema.tables 
WHERE table_schema = 'whmcs_db'
ORDER BY (data_length + index_length) DESC
LIMIT 20;"
```

## Step 3: Optimize Tables

```bash
# Optimize all tables
mysqlcheck -u root -p --optimize whmcs_db

# Optimize specific table
mysqlcheck -u root -p --optimize whmcs_db.tblactivitylog
```

## Step 4: Analyze Tables

```bash
# Analyze for query optimization
mysqlcheck -u root -p --analyze whmcs_db
```

## Step 5: Check for Table Issues

```bash
# Check all tables
mysqlcheck -u root -p --check whmcs_db

# Repair if issues found
mysqlcheck -u root -p --repair whmcs_db
```

## Step 6: Clean Up Activity Log

```sql
-- Delete logs older than 90 days
DELETE FROM tblactivitylog 
WHERE date < DATE_SUB(NOW(), INTERVAL 90 DAY);

-- Delete old admin logs
DELETE FROM tbladminlog 
WHERE logdate < DATE_SUB(NOW(), INTERVAL 90 DAY);
```

## Step 7: Clean Up Session Data

```sql
-- Clean expired sessions
DELETE FROM tblsessions 
WHERE lastvisit < DATE_SUB(NOW(), INTERVAL 1 DAY);

-- Clean remember me tokens
DELETE FROM tblemailauto生成的 
WHERE expiry < NOW();
```

## Step 8: Optimize MySQL Configuration

```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Recommended settings:
```ini
[mysqld]
max_connections = 200
table_open_cache = 2000
query_cache_type = 0
query_cache_size = 0
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT
```

```bash
systemctl restart mysql
```

## Step 9: Create Optimization Script

```bash
nano /usr/local/bin/whmcs-optimize-db.sh
```

```bash
#!/bin/bash
# WHMCS Database Optimization Script

echo "Starting WHMCS database optimization..."

# Optimize all tables
mysqlcheck -u root -p --optimize --all-databases

# Analyze tables
mysqlcheck -u root -p --analyze whmcs_db

# Clean old activity logs (90 days)
mysql -u root -p -e "
DELETE FROM whmcs_db.tblactivitylog 
WHERE date < DATE_SUB(NOW(), INTERVAL 90 DAY);"

# Clean old admin logs (90 days)
mysql -u root -p -e "
DELETE FROM whmcs_db.tbladminlog 
WHERE logdate < DATE_SUB(NOW(), INTERVAL 90 DAY);"

# Clean expired sessions
mysql -u root -p -e "
DELETE FROM whmcs_db.tblsessions 
WHERE lastvisit < DATE_SUB(NOW(), INTERVAL 1 DAY);"

echo "Database optimization complete"
```

```bash
chmod +x /usr/local/bin/whmcs-optimize-db.sh
```

## Step 10: Schedule Optimization

```bash
crontab -e
```

Add:
```cron
# Weekly database optimization - Sunday at 3 AM
0 3 * * 0 /usr/local/bin/whmcs-optimize-db.sh >> /var/log/whmcs-db-optimize.log 2>&1
```

## Step 11: Monitor Query Performance

```bash
# Enable slow query log
mysql -u root -p -e "SET GLOBAL slow_query_log = 'ON';"
mysql -u root -p -e "SET GLOBAL slow_query_log_file = '/var/log/mysql/slow.log';"
mysql -u root -p -e "SET GLOBAL long_query_time = 2;"

# View slow queries
tail -20 /var/log/mysql/slow.log
```

## Database Optimization Checklist

- [ ] Database size checked
- [ ] Large tables identified
- [ ] Tables optimized
- [ ] Tables analyzed
- [ ] Table issues resolved
- [ ] Old logs cleaned
- [ ] Sessions cleaned
- [ ] MySQL configured
- [ ] Optimization script created
- [ ] Optimization scheduled
- [ ] Query performance monitored
