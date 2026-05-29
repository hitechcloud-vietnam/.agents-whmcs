# WHMCS SSL Setup Workflow

## Purpose
Configure SSL certificates for secure WHMCS access

## Prerequisites
- WHMCS installed
- Domain pointed to server
- Root/sudo access

## Step 1: Check Current SSL Status

```bash
# Test current SSL connection
curl -I https://yourdomain.com

# Check SSL certificate details
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com
```

## Step 2: Install Let's Encrypt SSL (Recommended)

### Ubuntu/Debian
```bash
apt update
apt install -y certbot python3-certbot-apache
```

### CentOS/AlmaLinux
```bash
dnf install -y certbot python3-certbot-apache
```

### Generate Certificate
```bash
certbot --apache -d yourdomain.com -d www.yourdomain.com
```

### Follow Prompts
1. Enter email for renewal alerts
2. Accept Terms of Service
3. Choose: Redirect HTTP to HTTPS

### Verify Auto-Renewal
```bash
systemctl status certbot.timer
certbot renew --dry-run
```

## Step 3: Configure Apache SSL

```bash
nano /etc/apache2/sites-available/whmcs-ssl.conf
```

```apache
<VirtualHost *:443>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com
    DocumentRoot /var/www/whmcs

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem
    SSLCertificateChainFile /etc/letsencrypt/live/yourdomain.com/chain.pem

    Protocols h2 http/1.1

    <Directory /var/www/whmcs>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Security Headers
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

    ErrorLog ${APACHE_LOG_DIR}/whmcs_ssl_error.log
    CustomLog ${APACHE_LOG_DIR}/whmcs_ssl_access.log combined
</VirtualHost>
```

```bash
a2ensite whmcs-ssl.conf
a2enmod ssl headers
systemctl reload apache2
```

## Step 4: Configure HTTP to HTTPS Redirect

```bash
nano /etc/apache2/sites-available/whmcs.conf
```

```apache
<VirtualHost *:80>
    ServerName yourdomain.com
    ServerAlias www.yourdomain.com
    Redirect permanent / https://yourdomain.com/
</VirtualHost>
```

```bash
systemctl reload apache2
```

## Step 5: Configure Nginx (Alternative)

```bash
nano /etc/nginx/sites-available/whmcs
```

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;

    root /var/www/whmcs;
    index index.php;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/yourdomain.com/chain.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/whmcs /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

## Step 6: Update WHMCS Configuration

Navigate to: Setup > General Settings > General

```
Website Address: https://yourdomain.com/
Site SSL URL: https://yourdomain.com/
```

## Step 7: Configure WHMCS SSL Settings

Navigate to: Setup > General Settings > Security

```
Force SSL: Yes
SSL Certificate Path: /etc/letsencrypt/live/yourdomain.com/
```

## Step 8: Update configuration.php

```bash
nano /var/www/whmcs/configuration.php
```

Ensure SSL settings:
```php
$gk = 'YourLicenseKey';
$mysql_host = 'localhost';
$mysql_username = 'whmcs_user';
$mysql_password = 'YourPassword';
$mysql_database = 'whmcs_db';

$cc_username = 'yourdomain.com';
$cc_encryption_hash = md5(uniqid(rand(), TRUE));

$config['log_days'] = 30;
$config['display_errors'] = false;
```

## Step 9: Set Proper Permissions

```bash
cd /var/www/whmcs
chown -R www-data:www-data .
chmod 400 configuration.php
chmod 755 .
chmod 755 admin
chmod 755 templates_c
chmod 755 cache
```

## Step 10: Test SSL Configuration

```bash
# Check SSL grade
curl -I https://yourdomain.com

# Test with SSL Labs
# Visit: https://www.ssllabs.com/ssltest/

# Verify certificate chain
openssl s_client -connect yourdomain.com:443 -showcerts
```

## Step 11: Configure HSTS

```bash
nano /etc/apache2/sites-available/whmcs-ssl.conf
```

Add within VirtualHost:
```apache
Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "SAMEORIGIN"
Header always set X-XSS-Protection "1; mode=ban"
Header always set Referrer-Policy "strict-origin-when-cross-origin"
Header always set Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'"
```

```bash
systemctl reload apache2
```

## Step 12: Verify WHMCS SSL

Navigate to: Setup > General Settings > Security

Verify:
- SSL is enabled
- Force SSL enabled
- All URLs use https://

## Verification Checklist

- [ ] SSL certificate installed
- [ ] HTTPS working for main domain
- [ ] HTTP redirects to HTTPS
- [ ] WHMCS URLs updated to https://
- [ ] Security headers configured
- [ ] HSTS enabled
- [ ] SSL grade A or higher
- [ ] Certificate auto-renewal enabled

## Troubleshooting

### Mixed Content Errors
Check browser console for mixed content warnings. Update any hardcoded http:// URLs in templates and configuration.

### Certificate Chain Issues
```bash
# Verify certificate chain
openssl verify -CAfile /etc/letsencrypt/live/yourdomain.com/fullchain.pem /etc/letsencrypt/live/yourdomain.com/cert.pem
```

### SSL Not Working
```bash
# Check Apache SSL config
apache2ctl -t

# Check SSL module
a2enmod ssl
```
