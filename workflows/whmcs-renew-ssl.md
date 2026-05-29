# WHMCS SSL Certificate Renewal Workflow

## Description
Guide for renewing SSL certificates on WHMCS server.

## Prerequisites
- Existing SSL certificate
- SSH access to server
- Certificate authority credentials (if not Let's Encrypt)

## Certificate Types & Validity
- Commercial: 1-2 years
- Let's Encrypt: 90 days
- Auto-renewal recommended

## Steps

### Step 1: Check Certificate Expiration
```bash
# Check days until expiration
echo | openssl s_client -connect whmcs.example.com:443 2>/dev/null | openssl x509 -noout -dates

# Expected output:
# notBefore=Jan 15 00:00:00 2024 GMT
# notAfter=Jan 15 00:00:00 2025 GMT

# Calculate days remaining
CERT="/etc/ssl/certs/whmcs-combined.crt"
EXPIRATION=$(openssl x509 -in $CERT -noout -enddate | cut -d= -f2)
DAYS_LEFT=$(( ($(date -d "$EXPIRATION" +%s) - $(date +%s)) / 86400 ))
echo "Days until expiration: $DAYS_LEFT"
```

### Step 2: Renewal for Let's Encrypt
```bash
# Method A: Automatic renewal (recommended)
systemctl status certbot.timer

# Check if timer is active
systemctl list-timers | grep certbot

# Manual renewal test
certbot renew --dry-run

# Force renewal (even if not expiring)
certbot renew --force-renewal
```

### Step 3: Renewal for Commercial Certificates
```bash
# Step 1: Generate new CSR
openssl req -new -key /etc/ssl/private/whmcs.key -out /tmp/whmcs.csr

# Step 2: Display CSR for certificate authority
cat /tmp/whmcs.csr

# Step 3: Submit CSR to your certificate authority
# Go to: DigiCert, GlobalSign, Comodo, etc.
# Generate new certificate

# Step 4: Download new certificate files
# - your_domain.crt
# - ca_bundle.crt

# Step 5: Combine certificate files
cat /path/to/new_certificate.crt /path/to/ca_bundle.crt > /etc/ssl/certs/whmcs-combined.crt

# Step 6: Verify new certificate
openssl verify -CAfile /path/to/ca_bundle.crt /etc/ssl/certs/whmcs-combined.crt
openssl x509 -in /etc/ssl/certs/whmcs-combined.crt -noout -dates
```

### Step 4: Install Renewed Certificate
```bash
# For Nginx
# Copy new certificate
cp /path/to/new_combined.crt /etc/ssl/certs/whmcs-combined.crt

# Test configuration
nginx -t

# Reload Nginx
systemctl reload nginx

# For Apache
# Copy new certificate
cp /path/to/new_combined.crt /etc/ssl/certs/whmcs-combined.crt

# Test configuration
apachectl configtest

# Reload Apache
systemctl reload apache2
```

### Step 5: Verify Renewal
```bash
# Test SSL connection
echo | openssl s_client -connect whmcs.example.com:443 2>/dev/null | openssl x509 -noout -dates

# Check with external service
curl -sI https://whmcs.example.com | grep -i "strict-transport"

# Visit SSL Labs for full analysis
# https://www.ssllabs.com/ssltest/analyze.html?d=whmcs.example.com
```

### Step 6: Update WHMCS
```php
// Usually no changes needed
// WHMCS reads URLs from configuration.php

// Verify settings are correct
// Configuration > System Settings > General
// System URL should be https://
```

### Step 7: Monitoring Setup
```bash
# Create SSL expiration monitoring script
cat > /usr/local/bin/ssl-expiry-monitor.sh << 'EOF'
#!/bin/bash

ALERT_EMAIL="admin@example.com"
DOMAIN="whmcs.example.com"
CERT_FILE="/etc/ssl/certs/whmcs-combined.crt"
DAYS_WARNING=30

# Get expiration date
EXPIRATION=$(openssl x509 -in $CERT_FILE -noout -enddate 2>/dev/null | cut -d= -f2)

if [ -z "$EXPIRATION" ]; then
    echo "Cannot read certificate" | mail -s "SSL Alert: Certificate Error" $ALERT_EMAIL
    exit 1
fi

# Calculate days
EXPIRES_EPOCH=$(date -d "$EXPIRATION" +%s)
NOW_EPOCH=$(date +%s)
DAYS_LEFT=$(( ($EXPIRES_EPOCH - $NOW_EPOCH) / 86400 ))

echo "Domain: $DOMAIN"
echo "Expires: $EXPIRATION"
echo "Days left: $DAYS_LEFT"

if [ $DAYS_LEFT -lt $DAYS_WARNING ]; then
    echo "SSL Certificate for $DOMAIN expires in $DAYS_LEFT days!" | \
        mail -s "SSL Alert: Expiring Soon" $ALERT_EMAIL
fi

if [ $DAYS_LEFT -lt 7 ]; then
    echo "URGENT: SSL Certificate for $DOMAIN expires in $DAYS_LEFT days!" | \
        mail -s "SSL Alert: URGENT - Expiring Soon!" $ALERT_EMAIL
fi
EOF

chmod +x /usr/local/bin/ssl-expiry-monitor.sh

# Add to cron - check daily
echo "0 9 * * * /usr/local/bin/ssl-expiry-monitor.sh" >> /etc/crontab
```

### Step 8: Auto-Renewal Setup (Let's Encrypt)
```bash
# Let's Encrypt automatic renewal is handled by certbot
# Verify the timer is active
systemctl status certbot.timer

# If not enabled
systemctl enable --now certbot.timer

# Check renewal configuration
cat /etc/cron.d/certbot

# Manual test
certbot renew --deploy-hook "systemctl reload nginx"
```

### Step 9: Handle Renewal Failures
```bash
# Common issues and solutions

# Issue: Webroot validation fails
# Solution: Check .well-known directory permissions
mkdir -p /var/www/whmcs/.well-known/acme-challenge
chown -R www-data:www-data /var/www/whmcs/.well-known

# Issue: DNS not propagated
# Solution: Wait and retry
dig +short CNAME _acme-challenge.whmcs.example.com

# Issue: Port 80 blocked
# Solution: Ensure HTTP is accessible
ufw allow 80/tcp

# Issue: Rate limiting
# Solution: Wait and retry (Let's Encrypt has limits)
# Check: https://letsencrypt.org/docs/rate-limits/
```

### Step 10: Post-Renewal Checklist
- [ ] Certificate installed
- [ ] Web server reloaded
- [ ] SSL Labs test shows A or higher
- [ ] HTTPS works on all pages
- [ ] No mixed content warnings
- [ ] HSTS header present
- [ ] Monitoring updated with new expiry date

## Troubleshooting

### "Certificate verify failed"
```bash
# Update CA certificates
apt update && apt install -y ca-certificates
update-ca-certificates
```

### "No vhost found"
```bash
# For Let's Encrypt, ensure Nginx/Apache is configured
# Check for running web server
systemctl status nginx
systemctl status apache2

# Check if port 80 is being used
netstat -tuln | grep :80
```

### Old certificate still showing
```bash
# Clear browser cache
# Or force reload in browser: Ctrl+Shift+R

# Restart PHP-FPM
systemctl restart php8.2-fpm

# Clear opcode cache if using OPcache
# Restart PHP-FPM will clear it
```

## Tags
- ssl
- renewal
- certificate
- security
- maintenance