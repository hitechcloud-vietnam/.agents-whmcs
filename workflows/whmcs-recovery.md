# WHMCS Disaster Recovery Workflow

## Description
Complete disaster recovery procedures for WHMCS system failures.

## Scenario 1: Complete Server Failure

### Step 1: Provision New Server
```bash
# Setup new LAMP/LEMP stack
apt update && apt upgrade -y
apt install -y php mysql-server nginx letsencrypt
```

### Step 2: Restore from Offsite Backup
```bash
# Download latest backup from offsite storage
rsync -avz backup@offsite-server:/backups/whmcs/ /tmp/whmcs_backup/

# Extract backup
tar -xzvf /tmp/whmcs_backup/whmcs_full_backup_latest.tar.gz -C /var/www/
```

### Step 3: Restore Database
```bash
mysql -u root -p -e "CREATE DATABASE whmcs CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p whmcs < /tmp/whmcs_backup/database_backup.sql
```

### Step 4: Recreate Environment
```bash
# Reinstall same PHP version
apt install -y php8.2 php8.2-mysql php8.2-gd php8.2-curl php8.2-xml php8.2-zip

# Restore web server config
cp /tmp/whmcs_backup/nginx.conf /etc/nginx/sites-available/whmcs
```

### Step 5: Update DNS and SSL
```bash
# Setup SSL
certbot --nginx -d whmcs.example.com

# Update DNS
# (Update A record to new server IP)
```

## Scenario 2: Database Corruption

### Step 1: Identify Corruption
```bash
# Check MySQL error logs
tail -100 /var/log/mysql/error.log

# Test database integrity
mysqlcheck -u root -p --check whmcs
```

### Step 2: Restore Database
```bash
# Stop MySQL
systemctl stop mysql

# Drop corrupted database
mysql -u root -p -e "DROP DATABASE whmcs;"

# Restore from backup
mysql -u root -p -e "CREATE DATABASE whmcs;"
mysql -u root -p whmcs < /path/to/backup/database.sql
```

### Step 3: Verify Tables
```sql
USE whmcs;
SHOW TABLES;
CHECK TABLE tblclients;
CHECK TABLE tblinvoices;
CHECK TABLE tblhosting;
-- Check all critical tables
```

## Scenario 3: Ransomware/Intrusion

### Step 1: Isolate System
```bash
# Disconnect from network
iptables -I INPUT -j DROP
# Keep SSH access for yourself
iptables -I INPUT -p tcp --dport 22 -j ACCEPT
```

### Step 2: Forensic Analysis
```bash
# Check for unauthorized files
find /var/www/whmcs -name "*.encrypted" -o -name "*.locked"
ls -la /var/www/whmcs/*.php  # Check for suspicious PHP files
```

### Step 3: Clean Restore
```bash
# Never pay ransom
# Restore from known-good backup before incident
rm -rf /var/www/whmcs
tar -xzvf /path/to/clean_backup/whmcs.tar.gz -C /var/www/

# Change all passwords
mysql -u root -p -e "UPDATE whmcs.tbladmins SET password = MD5('newpassword') WHERE id=1;"
```

### Step 4: Security Hardening
```bash
# Update all passwords
# Enable 2FA for all admins
# Review and block suspicious IPs
# Update firewall rules
```

## Recovery Time Objectives

| Component | RTO | RPO |
|-----------|-----|-----|
| Full System | 4-8 hours | 24 hours |
| Database Only | 1-2 hours | 24 hours |
| File Restore | 1 hour | 24 hours |

## Prevention Checklist
- [ ] Daily automated backups
- [ ] Offsite backup storage
- [ ] Database replication
- [ ] Regular backup testing
- [ ] Monitoring and alerts
- [ ] Security hardening
- [ ] 2FA enabled
- [ ] Firewall configured

## Emergency Contacts
- WHMCS Support: https://support.whmcs.com
- Hosting Provider Support
- Security Team (if breach occurred)

## Tags
- disaster-recovery
- backup
- emergency
- business-continuity