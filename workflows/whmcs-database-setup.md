# WHMCS Database Setup Workflow

## Purpose
Configure MySQL/MariaDB database for WHMCS

## Prerequisites
- MySQL/MariaDB installed
- Root MySQL access
- WHMCS installed

## Step 1: Access MySQL

```bash
mysql -u root -p
```

## Step 2: Create Database

```sql
CREATE DATABASE whmcs_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

Verify:
```sql
SHOW DATABASES;
```

## Step 3: Create User

```sql
CREATE USER 'whmcs_user'@'localhost' IDENTIFIED BY 'StrongPassword123!';
```

## Step 4: Grant Privileges

```sql
GRANT ALL PRIVILEGES ON whmcs_db.* TO 'whmcs_user'@'localhost';
FLUSH PRIVILEGES;
```

Verify:
```sql
SHOW GRANTS FOR 'whmcs_user'@'localhost';
```

## Step 5: Configure MySQL for WHMCS

```bash
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Update values:
```ini
[mysqld]
max_connections = 200
max_allowed_packet = 64M
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
query_cache_type = 0
query_cache_size = 0
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

```bash
systemctl restart mysql
```

## Step 6: Update WHMCS Configuration

```bash
nano /var/www/whmcs/configuration.php
```

```php
$mysql_host = 'localhost';
$mysql_username = 'whmcs_user';
$mysql_password = 'StrongPassword123!';
$mysql_database = 'whmcs_db';
```

## Step 7: Database Maintenance

### Backup Database
```bash
mysqldump -u whmcs_user -p whmcs_db > /backup/whmcs_db_$(date +%Y%m%d).sql
```

### Optimize Tables
```bash
mysqlcheck -u root -p --optimize whmcs_db
```

### Repair Tables
```bash
mysqlcheck -u root -p --repair whmcs_db
```

## Step 8: Remote Database (Optional)

For remote MySQL:
```sql
CREATE USER 'whmcs_user'@'%' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON whmcs_db.* TO 'whmcs_user'@'%';
FLUSH PRIVILEGES;
```

```php
$mysql_host = 'remote.server.com';
$mysql_port = '3306';
```

## Database Credentials Reference

| Setting | Value |
|---------|-------|
| Host | localhost |
| Database | whmcs_db |
| User | whmcs_user |
| Charset | utf8mb4 |

## Verification

Test connection:
```bash
mysql -u whmcs_user -p whmcs_db -e "SELECT COUNT(*) FROM whmcs_users;"
```
