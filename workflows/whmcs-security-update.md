# WHMCS Security Update Workflow

## Purpose
Apply security updates and harden WHMCS installation

## Prerequisites
- WHMCS installed
- Admin access
- SSH access

## Step 1: Check WHMCS Version

Navigate to: Utilities > System Health Status

Verify version and check for security updates.

## Step 2: Review Security Announcements

Visit: https://whmcs.com/security/

Check for:
- Recent security patches
- Vulnerability disclosures
- Recommended actions

## Step 3: Enable Automatic Updates (if available)

Navigate to: Setup > General Settings > Security

Enable automatic security updates if supported.

## Step 4: Update WHMCS

```bash
# Create backup first
/usr/local/bin/whmcs-backup.sh

# Download latest version
cd /tmp
wget https://downloads.whmcs.com/whmcs-latest.zip
unzip -o whmcs-latest.zip

# Apply update
cd /var/www/whmcs
cp -r /tmp/whmcs-latest/* .
```

## Step 5: Update PHP

```bash
# Check current version
php -v

# Update PHP (Ubuntu)
apt update && apt install php8.1

# Update PHP (CentOS)
dnf update php-8.1
```

## Step 6: Update MySQL/MariaDB

```bash
# Check current version
mysql --version

# Update MySQL (Ubuntu)
apt update && apt install mysql-server

# Update MariaDB (Ubuntu)
apt update && apt install mariadb-server
```

## Step 7: Review File Permissions

```bash
cd /var/www/whmcs

# Set secure permissions
chown -R www-data:www-data .
find . -type d -exec chmod 755 {} \;
find . -type f -exec chmod 644 {} \;

# Critical files
chmod 400 configuration.php
chmod 644 .htaccess
chmod 755 admin
```

## Step 8: Review Admin Access

Navigate to: Configuration > System Settings > Administrators

1. Review all admin accounts
2. Remove unused accounts
3. Enable 2FA for all admins
4. Check permission levels

## Step 9: Update Admin Passwords

```bash
# Force password reset via WHMCS admin
# Navigate to: Configuration > System Settings > Administrators > [User] > Password
```

## Step 10: Review API Keys

Navigate to: Setup > System Settings > API Credentials

1. Review active API keys
2. Remove unused keys
3. Update key permissions
4. Rotate keys if needed

## Step 11: Configure Firewall

```bash
# UFW (Ubuntu)
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable

# firewalld (CentOS)
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-service=ssh
firewall-cmd --reload
```

## Step 12: Enable SSL/TLS

```bash
# Check SSL certificate
openssl s_client -connect yourdomain.com:443 2>/dev/null | openssl x509 -noout -dates

# Renew if expired
certbot renew
```

## Step 13: Review Security Headers

```bash
nano /var/www/whmcs/.htaccess
```

Add security headers:
```apache
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-Content-Type-Options "nosniff"
Header always set X-XSS-Protection "1; mode=block"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Content-Security-Policy "default-src 'self'"
```

## Step 14: Review and Update .htaccess

```bash
nano /var/www/whmcs/.htaccess
```

Ensure protection:
```apache
# Prevent directory listing
Options -Indexes

# Protect configuration
<FilesMatch "^\.ht">
    Require all denied
</FilesMatch>

# Block access to sensitive files
<FilesMatch "^(configuration\.php|init\.php)">
    Require all denied
</FilesMatch>
```

## Step 15: Update SSL Configuration

```bash
nano /etc/apache2/sites-available/whmcs-ssl.conf
```

```apache
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
SSLHonorCipherOrder on
```

```bash
systemctl restart apache2
```

## Step 16: Review Failed Login Attempts

Navigate to: Configuration > System Settings > Admin Activity Log

Look for:
- Multiple failed attempts
- Unknown IP addresses
- Suspicious patterns

## Step 17: Configure Intrusion Detection

```bash
# Install fail2ban
apt install fail2ban

# Configure
nano /etc/fail2ban/jail.local
```

```ini
[whmcs]
enabled = true
port = http,https
filter = whmcs
logpath = /var/www/whmcs/logs/error.log
maxretry = 5
bantime = 3600
```

```bash
systemctl enable fail2ban
systemctl start fail2ban
```

## Step 18: Test Security

1. Run security scan
2. Test admin login
3. Test client login
4. Verify SSL works
5. Check headers present

## Security Update Checklist

- [ ] Version checked
- [ ] Security announcements reviewed
- [ ] WHMCS updated
- [ ] PHP updated
- [ ] MySQL updated
- [ ] Permissions reviewed
- [ ] Admin access reviewed
- [ ] Passwords updated
- [ ] API keys reviewed
- [ ] Firewall configured
- [ ] SSL enabled and tested
- [ ] Security headers added
- [ ] .htaccess reviewed
- [ ] SSL configured
- [ ] Failed logins reviewed
- [ ] Intrusion detection enabled
- [ ] Security tested
