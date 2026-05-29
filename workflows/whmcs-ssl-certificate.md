# WHMCS SSL Certificate Installation Workflow

## Description
Step-by-step guide for installing SSL certificates on WHMCS server.

## Prerequisites
- Domain validated or purchased SSL certificate
- Private key
- Certificate files (CRT, CA Bundle)
- SSH access to server
- Web server (Apache/Nginx)

## Certificate Types
- Single Domain
- Wildcard (*.example.com)
- Multi-Domain (SAN)

## Steps

### Step 1: Generate CSR (Certificate Signing Request)
```bash
# Create private key
openssl genrsa -out /etc/ssl/private/whmcs.key 2048

# Create CSR
openssl req -new -key /etc/ssl/private/whmcs.key -out /etc/ssl/certs/whmcs.csr

# Fill in details:
# Country Name: US
# State: California
# Locality: San Francisco
# Organization: Your Company Name
# Common Name: whmcs.example.com
```

### Step 2: Order Certificate
```bash
# Display CSR for copy
cat /etc/ssl/certs/whmcs.csr

# Or use Let's Encrypt (free)
apt install -y certbot python3-certbot-nginx python3-certbot-apache

# Generate certificate
certbot certonly --webroot -w /var/www/whmcs -d whmcs.example.com

# For Apache
certbot --apache -d whmcs.example.com

# For Nginx
certbot --nginx -d whmcs.example.com
```

### Step 3: Combine Certificate Files
```bash
# For commercial certificates
# Combine certificate and CA bundle
cat /path/to/your_certificate.crt /path/to/ca_bundle.crt > /etc/ssl/certs/whmcs-combined.crt

# Verify certificate
openssl verify -CAfile /path/to/ca_bundle.crt /etc/ssl/certs/whmcs-combined.crt

# Check certificate details
openssl x509 -in /etc/ssl/certs/whmcs-combined.crt -text -noout
```

### Step 4: Configure Nginx
```bash
cat > /etc/nginx/sites-available/whmcs << 'EOF'
server {
    listen 80;
    server_name whmcs.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name whmcs.example.com;
    root /var/www/whmcs;
    index index.php;

    # SSL Configuration
    ssl_certificate /etc/ssl/certs/whmcs-combined.crt;
    ssl_certificate_key /etc/ssl/private/whmcs.key;
    
    # SSL Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=63072000" always;
    
    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # PHP Configuration
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Timeouts
        fastcgi_connect_timeout 60s;
        fastcgi_send_timeout 300s;
        fastcgi_read_timeout 300s;
    }
    
    # Deny access to sensitive files
    location ~ /\.(?!well-known) {
        deny all;
    }
    
    location ~* \.(env|log|conf)$ {
        deny all;
    }
    
    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;
}
EOF

# Enable site
ln -s /etc/nginx/sites-available/whmcs /etc/nginx/sites-enabled/whmcs

# Test and reload
nginx -t
systemctl reload nginx
```

### Step 5: Configure Apache
```bash
cat > /etc/apache2/sites-available/whmcs.conf << 'EOF'
<VirtualHost *:80>
    ServerName whmcs.example.com
    Redirect permanent / https://whmcs.example.com/
</VirtualHost>

<VirtualHost *:443>
    ServerName whmcs.example.com
    DocumentRoot /var/www/whmcs
    
    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/whmcs-combined.crt
    SSLCertificateKeyFile /etc/ssl/private/whmcs.key
    SSLCertificateChainFile /etc/ssl/certs/ca_bundle.crt
    
    # SSL Settings
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256
    SSLHonorCipherOrder on
    
    # HSTS
    Header always set Strict-Transport-Security "max-age=63072000"
    
    <Directory /var/www/whmcs>
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog ${APACHE_LOG_DIR}/whmcs_error.log
    CustomLog ${APACHE_LOG_DIR}/whmcs_access.log combined
</VirtualHost>
EOF

# Enable modules and site
a2enmod ssl rewrite headers
a2ensite whmcs.conf
a2dissite 000-default.conf

# Test and reload
apachectl configtest
systemctl reload apache2
```

### Step 6: Update WHMCS Configuration
```php
// In configuration.php
// Ensure these are set correctly
$whmcs_config = [
    'SystemSSLURL' => 'https://whmcs.example.com',
    'SystemURL' => 'https://whmcs.example.com',
    'AdminURL' => 'https://whmcs.example.com/admin',
];
```

### Step 7: Update Cron for SSL
```bash
# If using Let's Encrypt, auto-renewal is automatic
# Verify renewal works
certbot renew --dry-run

# Add to crontab if needed
# 0 0 * * * certbot renew --quiet
```

### Step 8: Verify SSL Installation
```bash
# Check SSL grade
# Visit: https://www.ssllabs.com/ssltest/
# Target: A or A+

# Command line verification
echo | openssl s_client -connect whmcs.example.com:443 -servername whmcs.example.com 2>/dev/null | openssl x509 -noout -dates -issuer

# Check certificate chain
echo | openssl s_client -connect whmcs.example.com:443 -showcerts 2>/dev/null | grep -E "subject=|issuer="
```

### Step 9: Force HTTPS in WHMCS
1. Login to WHMCS Admin
2. Go to Configuration > System Settings > General
3. Ensure both System URL and SSL URL are set to HTTPS
4. Save settings

### Step 10: Test Complete Flow
```bash
# Test full HTTPS connection
curl -I https://whmcs.example.com
curl -I https://whmcs.example.com/clientarea.php
curl -I https://whmcs.example.com/admin

# Test mixed content
# Install browser extension or use:
curl -sS https://whmcs.example.com | grep -i "http://" | head -10
```

## Let's Encrypt Automated Setup
```bash
# Complete Let's Encrypt setup
apt install -y certbot python3-certbot-nginx

# Create webroot for validation
mkdir -p /var/www/whmcs/.well-known/acme-challenge

# Generate certificate
certbot certonly --webroot -w /var/www/whmcs \
    -d whmcs.example.com \
    --agree-tos -m admin@example.com \
    --non-interactive

# Auto-renewal (usually automatic, but verify)
systemctl status certbot.timer
```

## Troubleshooting SSL Issues

### Certificate Chain Issues
```bash
# If SSL Labs shows incomplete chain
# Combine files in correct order
cat your_domain.crt bundle.crt > combined.crt
```

### Mixed Content Issues
```php
// Force HTTPS for all resources in WHMCS
// Add to configuration.php
if (empty($_SERVER['HTTPS']) || $_SERVER['HTTPS'] !== 'on') {
    header("Location: https://${_SERVER['HTTP_HOST']}${_SERVER['REQUEST_URI']}");
    exit();
}
```

### Permission Issues
```bash
chmod 600 /etc/ssl/private/whmcs.key
chmod 644 /etc/ssl/certs/whmcs-combined.crt
chown root:root /etc/ssl/private/whmcs.key
chown root:root /etc/ssl/certs/whmcs-combined.crt
```

## Tags
- ssl
- tls
- https
- security
- certificate