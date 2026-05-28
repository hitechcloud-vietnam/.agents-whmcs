# WHMCS SSL Management Workflow

## Purpose

Implement comprehensive SSL certificate management for WHMCS including certificate installation, renewal automation, monitoring, and handling mixed content issues. Ensures secure HTTPS connections for all WHMCS installations.

## Prerequisites

- WHMCS installation with web server access
- Domain validated and pointing to server
- Root/sudo access on server
- SSL provider account (optional for automation)

## Workflow Steps

### Step 1: Generate CSR and Install Certificate

```bash
#!/bin/bash
# /opt/scripts/generate_csr.sh

DOMAIN="whmcs.example.com"
COUNTRY="US"
STATE="California"
CITY="San Francisco"
ORG="Your Company"
EMAIL="admin@example.com"
KEY_SIZE=4096

# Generate private key
openssl genrsa -out /etc/ssl/private/${DOMAIN}.key $KEY_SIZE

# Generate CSR
openssl req -new -key /etc/ssl/private/${DOMAIN}.key \
    -out /etc/ssl/csr/${DOMAIN}.csr \
    -subj "/C=$COUNTRY/ST=$STATE/L=$CITY/O=$ORG/CN=$DOMAIN/emailAddress=$EMAIL"

# Display CSR for certificate provider
echo "=== CSR for $DOMAIN ==="
cat /etc/ssl/csr/${DOMAIN}.csr

# For Let's Encrypt (automated)
apt-get install -y certbot python3-certbot-nginx

certbot --nginx -d $DOMAIN -d www.$DOMAIN
```

### Step 2: Nginx SSL Configuration

```bash
# /etc/nginx/sites-available/whmcs-ssl

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name whmcs.example.com;
    
    # SSL Certificate
    ssl_certificate /etc/ssl/certs/${DOMAIN}.crt;
    ssl_certificate_key /etc/ssl/private/${DOMAIN}.key;
    
    # SSL Settings
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    
    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;
    
    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    root /var/www/whmcs;
    index index.php index.html;
    
    # ... rest of configuration
}

# HTTP to HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name whmcs.example.com;
    
    location / {
        return 301 https://$host$request_uri;
    }
}
```

### Step 3: Let's Encrypt Auto-Renewal

```bash
#!/bin/bash
# /opt/scripts/letsencrypt_renew.sh
# Automated certificate renewal

set -euo pipefail

LOG_FILE="/var/log/letsencrypt-renew.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "Starting certificate renewal check"

# Renew certificates
certbot renew --quiet --post-hook "systemctl reload nginx"

# Check if renewed
if [ $? -eq 0 ]; then
    log "Certificate renewal successful"
    
    # Reload web server
    systemctl reload nginx
    
    # Notify success
    echo "SSL certificate renewed successfully" | mail -s "SSL Renewal Success" admin@example.com
else
    log "No certificates due for renewal"
fi

# Cleanup old certificates
find /etc/letsencrypt -name "*.pem" -mtime +30 -delete 2>/dev/null || true
```

```bash
# /etc/cron.d/letsencrypt-renew
# Run renewal check twice daily
0 0,12 * * * root /opt/scripts/letsencrypt_renew.sh >> /var/log/letsencrypt-renew.log 2>&1
```

### Step 4: WHMCS SSL Configuration

```php
# configuration.php - Force HTTPS

// Force all traffic to HTTPS
$_SERVER['HTTPS'] = 'on';
$_SERVER['SERVER_PORT'] = 443';

// For Apache
// .htaccess additions:
RewriteEngine On
RewriteCond %{HTTPS} off
RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
```

```php
<?php
// /var/www/whmcs/includes/hooks/ssl_hook.php
// SSL enforcement hook

add_hook('ClientAreaPage', 1, function($vars) {
    // Check if running on HTTPS
    if (empty($_SERVER['HTTPS']) || $_SERVER['HTTPS'] !== 'on') {
        // Only redirect if not CLI and not already being redirected
        if (php_sapi_name() !== 'cli') {
            $redirectUrl = 'https://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
            header('Location: ' . $redirectUrl, true, 301);
            exit;
        }
    }
});

add_hook('AdminAreaPage', 1, function($vars) {
    // Admin SSL enforcement
    if (empty($_SERVER['HTTPS']) || $_SERVER['HTTPS'] !== 'on') {
        if (php_sapi_name() !== 'cli') {
            $redirectUrl = 'https://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
            header('Location: ' . $redirectUrl, true, 301);
            exit;
        }
    }
});
```

### Step 5: SSL Certificate Monitoring

```php
<?php
// /opt/scripts/ssl_monitor.php
// Monitor SSL certificate expiration

class SSLCertificateMonitor {
    private $pdo;
    private $warningThreshold = 30; // days
    private $criticalThreshold = 7;  // days
    
    public function __construct() {
        $this->pdo = new PDO('mysql:host=localhost', 'whmcs', 'password');
    }
    
    public function checkCertificates(): array {
        $domains = $this->getMonitoredDomains();
        $results = [];
        
        foreach ($domains as $domain) {
            $cert = $this->getCertificateInfo($domain);
            $results[$domain] = $this->evaluateCertificate($cert);
        }
        
        $this->alertExpiring($results);
        
        return $results;
    }
    
    private function getCertificateInfo(string $domain): array {
        $context = stream_context_create([
            'ssl' => [
                'verify_peer' => false,
                'verify_peer_name' => false
            ]
        ]);
        
        $remote = stream_socket_client(
            "ssl://{$domain}:443",
            $errno,
            $errstr,
            5,
            STREAM_CLIENT_CONNECT,
            $context
        );
        
        if (!$remote) {
            return ['error' => $errstr];
        }
        
        $cert = stream_socket_get_transport($remote);
        preg_match('/ subject=.*issuer=.*/', $cert, $matches);
        
        $certInfo = openssl_x509_parse($cert);
        
        return [
            'subject' => $certInfo['subject']['CN'],
            'issuer' => $certInfo['issuer']['CN'],
            'valid_from' => date('Y-m-d', $certInfo['validFrom_time_t']),
            'valid_to' => date('Y-m-d', $certInfo['validTo_time_t']),
            'days_remaining' => floor(($certInfo['validTo_time_t'] - time()) / 86400)
        ];
    }
    
    private function evaluateCertificate(array $cert): string {
        if (isset($cert['error'])) {
            return 'ERROR';
        }
        
        if ($cert['days_remaining'] <= $this->criticalThreshold) {
            return 'CRITICAL';
        }
        
        if ($cert['days_remaining'] <= $this->warningThreshold) {
            return 'WARNING';
        }
        
        return 'OK';
    }
    
    private function alertExpiring(array $results): void {
        foreach ($results as $domain => $status) {
            if (in_array($status, ['CRITICAL', 'WARNING'])) {
                $this->sendAlert($domain, $status);
            }
        }
    }
    
    private function sendAlert(string $domain, string $status): void {
        $message = "SSL Certificate $status: $domain";
        
        // Send email
        mail('admin@example.com', "SSL Alert: $status", $message);
        
        // Send to monitoring system
        logActivity($message);
    }
    
    private function getMonitoredDomains(): array {
        $stmt = $this->pdo->query("SELECT domain FROM mod_ssl_monitor");
        return $stmt->fetchAll(PDO::FETCH_COLUMN);
    }
}

// Run check
$monitor = new SSLCertificateMonitor();
$monitor->checkCertificates();
```

### Step 6: SSL Certificate Installation Script

```bash
#!/bin/bash
# /opt/scripts/install_ssl.sh

DOMAIN=$1
CERT_FILE=$2
KEY_FILE=$3
CHAIN_FILE=$4

if [ -z "$DOMAIN" ] || [ -z "$CERT_FILE" ] || [ -z "$KEY_FILE" ]; then
    echo "Usage: $0 <domain> <cert_file> <key_file> [chain_file]"
    exit 1
fi

CERT_DIR="/etc/ssl/certs"
KEY_DIR="/etc/ssl/private"

# Backup existing
if [ -f "$CERT_DIR/${DOMAIN}.crt" ]; then
    cp "$CERT_DIR/${DOMAIN}.crt" "$CERT_DIR/${DOMAIN}.crt.bak-$(date +%Y%m%d)"
fi

if [ -f "$KEY_DIR/${DOMAIN}.key" ]; then
    cp "$KEY_DIR/${DOMAIN}.key" "$KEY_DIR/${DOMAIN}.key.bak-$(date +%Y%m%d)"
fi

# Install certificate
cp "$CERT_FILE" "$CERT_DIR/${DOMAIN}.crt"
chmod 644 "$CERT_DIR/${DOMAIN}.crt"

# Install private key
cp "$KEY_FILE" "$KEY_DIR/${DOMAIN}.key"
chmod 600 "$KEY_DIR/${DOMAIN}.key"

# Install chain if provided
if [ -n "$CHAIN_FILE" ]; then
    cat "$CERT_FILE" "$CHAIN_FILE" > "$CERT_DIR/${DOMAIN}.fullchain.crt"
fi

# Test nginx config
nginx -t

if [ $? -eq 0 ]; then
    # Reload nginx
    systemctl reload nginx
    echo "SSL certificate installed successfully for $DOMAIN"
else
    echo "Nginx config test failed. Certificate NOT applied."
    exit 1
fi
```

### Step 7: Mixed Content Fix

```php
<?php
// /var/www/whmcs/includes/hooks/mixed_content_hook.php
// Fix mixed content issues in WHMCS

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return <<<HTML
<script>
// Fix mixed content by upgrading all resources to HTTPS
(function() {
    // Fix inline styles and attributes
    document.querySelectorAll('[src^="http://"], [href^="http://"]').forEach(function(el) {
        var attr = el.hasAttribute('src') ? 'src' : 'href';
        var url = el.getAttribute(attr);
        if (!url.match(/^(https?:)?\/\//)) return;
        if (url.startsWith('//')) {
            el.setAttribute(attr, 'https:' + url);
        } else if (url.startsWith('http://')) {
            el.setAttribute(attr, url.replace('http:', 'https:'));
        }
    });
})();
</script>
<meta http-equiv="Content-Security-Policy" content="upgrade-insecure-requests">
HTML;
});

// Fix WHMCS asset URLs
add_hook('SmartyOutput', 0, function($vars) {
    $output = $vars['smarty']->fetch('eval:' . $vars['output']);
    return str_replace('http://', 'https://', $output);
});
```

## SSL/TLS Configuration Guidelines

| Setting | Recommended Value |
|---------|-------------------|
| Protocol | TLS 1.2, TLS 1.3 only |
| Ciphers | Modern suites only |
| Key Size | RSA 4096-bit or ECDSA P-256 |
| Certificate | 256-bit signed |
| HSTS | Enabled, max-age 1 year |
| OCSP | Stapling enabled |

## Best Practices

1. **Use Let's Encrypt**: Free automated certificates
2. **Enable HSTS**: Force HTTPS with HSTS header
3. **OCSP Stapling**: Improve SSL handshake performance
4. **Certificate Monitoring**: Alert before expiration
5. **Mixed Content Fix**: Upgrade all HTTP to HTTPS
6. **Regular Updates**: Keep SSL configuration current

## Common Pitfalls

- **Expired Certificates**: Monitor and auto-renew
- **Mixed Content**: Causes browser warnings
- **Weak Ciphers**: Security vulnerability
- **Missing Chain**: Incomplete certificate chain
- **HTTP to HTTPS Loop**: Redirect loop issues

## Verification Checklist

- [ ] SSL installed and working
- [ ] HTTPS redirects working
- [ ] HSTS header enabled
- [ ] No mixed content warnings
- [ ] OCSP stapling enabled
- [ ] Certificate monitoring configured
- [ ] Auto-renewal working
- [ ] Grade A on SSL Labs test

## Related Documentation

- [WHMCS Security Audit](whmcs-security-audit.md)
- [SSL Labs Test](https://www.ssllabs.com/ssltest/)
- [Let's Encrypt Documentation](https://letsencrypt.org/docs/)