# WHMCS DNS Configuration Workflow

## Description
Configure DNS records for WHMCS server with proper setup for email, subdomains, and services.

## Prerequisites
- Domain name registered
- Access to DNS management ( registrar or third-party )
- Understanding of DNS record types

## Required DNS Records

### Core Records
| Type | Name | Value | Priority/TTL |
|------|------|-------|--------------|
| A | @ | Server IP | 3600 |
| A | www | Server IP | 3600 |
| A | whmcs | Server IP | 3600 |
| A | admin | Server IP | 3600 (optional) |
| AAAA | @ | IPv6 Address | 3600 |

### Email Records (Required)
| Type | Name | Value | Priority |
|------|------|-------|----------|
| MX | @ | mail.example.com | 10 |
| TXT | @ | v=spf1 mx a:mail.example.com ~all | - |
| TXT | @ | v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com | - |
| TXT | default._domainkey | (DKIM public key) | - |

## Steps

### Step 1: Configure A Record
```bash
# Point domain to WHMCS server
# In your DNS provider (CloudFlare, Route53, etc.)

# Primary A record
Name: @ (or example.com)
Type: A
Value: 203.0.113.50 (your server IP)
TTL: 3600 (1 hour)

# WWW subdomain
Name: www
Type: A
Value: 203.0.113.50
TTL: 3600

# WHMCS subdomain
Name: whmcs
Type: A
Value: 203.0.113.50
TTL: 3600
```

### Step 2: Configure MX Record for Email
```bash
# MX Record
Name: @ (or example.com)
Type: MX
Value: mail.example.com
Priority: 10
TTL: 3600

# Secondary MX (optional)
Name: @ (or example.com)
Type: MX
Value: mail2.example.com
Priority: 20
TTL: 3600
```

### Step 3: Configure SPF Record
```bash
# SPF TXT Record
Name: @ (or example.com)
Type: TXT
Value: v=spf1 mx a:mail.example.com include:_spf.example.com ~all
TTL: 3600

# More examples:
# Basic - only use servers listed in MX
v=spf1 mx ~all

# With additional mail providers
v=spf1 mx a:mail.example.com include:spf.protection.outlook.com ~all

# Allow specific IPs
v=spf1 ip4:203.0.113.50 mx ~all

# With third-party services (SendGrid, Mailgun)
v=spf1 mx include:sendgrid.net include:mailgun.org ~all
```

### Step 4: Configure DKIM Record
```bash
# Generate DKIM key pair on mail server
# (See whmcs-email-server.md for full setup)

# Public DKIM TXT Record
Name: default._domainkey
Type: TXT
Value: v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC... (your key)
TTL: 3600

# Alternative selector
Name: mail._domainkey
Type: TXT
Value: v=DKIM1; k=rsa; p=your_public_key
TTL: 3600
```

### Step 5: Configure DMARC Record
```bash
# Basic DMARC
Name: _dmarc
Type: TXT
Value: v=DMARC1; p=none; rua=mailto:dmarc@example.com
TTL: 3600

# Strict DMARC
Name: _dmarc
Type: TXT
Value: v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com; ruf=mailto:dmarc-forensics@example.com; pct=100
TTL: 3600

# Monitor only (for testing)
Name: _dmarc
Type: TXT
Value: v=DMARC1; p=none; rua=mailto:dmarc@example.com
TTL: 3600
```

### Step 6: Configure CAA Record (Security)
```bash
# CAA Record - specifies allowed certificate authorities
Name: @
Type: CAA
Value: 0 issue "letsencrypt.org"
Value: 0 issue "digicert.com"
TTL: 3600

# With email for reporting issues
Name: @
Type: CAA
Value: 0 issue "letsencrypt.org"
Value: 0 iodef "mailto:security@example.com"
TTL: 3600
```

### Step 7: Configure Reverse DNS (PTR)
```bash
# Set up at your hosting provider
# Point to your domain

# For cloud providers:
# AWS Route53 - Create PTR record in reverse lookup zone
# DigitalOcean - Set reverse DNS in droplet settings
# Linode - Set reverse DNS in node settings

# Verify reverse DNS
dig -x 203.0.113.50 +short
# Expected: whmcs.example.com
```

### Step 8: DNS Propagation Check
```bash
# Check DNS propagation
dig whmcs.example.com A
dig whmcs.example.com MX
dig whmcs.example.com TXT

# Use multiple DNS servers
dig @8.8.8.8 whmcs.example.com A
dig @1.1.1.1 whmcs.example.com A
dig @9.9.9.9 whmcs.example.com A

# Online tools:
# https://dnschecker.org
# https://www.whatsmydns.net
```

### Step 9: Configure WHMCS DNS Settings
```php
// In configuration.php
$whmcs_config = [
    'SystemURL' => 'https://whmcs.example.com',
    'SystemSSLURL' => 'https://whmcs.example.com',
    'Domain' => 'whmcs.example.com',
];

// For domain registrar module DNS
// Go to Configuration > System Settings > Domain Registrar
// Configure DNS template
```

### Step 10: Advanced: Third-Party DNS Providers

#### CloudFlare Setup
```php
// If using CloudFlare proxy
// Set proxy status to "Proxied" for CDN benefits
// DNS Only for subdomains needing direct IP

// CloudFlare API integration
$cloudflare_api_key = 'your_api_key';
$cloudflare_email = 'admin@example.com';
```

#### AWS Route53 Setup
```bash
# Install AWS CLI
apt install -y awscli

# Configure AWS credentials
aws configure

# Create hosted zone
aws route53 create-hosted-zone \
    --name whmcs.example.com \
    --caller-reference $(date +%s)

# Add records
aws route53 change-resource-record-sets \
    --hosted-zone-id Z1234567890ABC \
    --change-batch file://dns-changes.json
```

### Step 11: DNSSEC Configuration
```bash
# Enable DNSSEC at your registrar

# Get DS records from your DNS provider
# CloudFlare: Dashboard > DNS > DNSSEC > Enable
# AWS Route53: Create DS record in registrar

# Example DS record format
# Flags: 257
# Protocol: 3
# Algorithm: 8 (RSA/SHA-256)
# Digest Type: 2 (SHA-256)
# Digest: your_digest_value

# Verify DNSSEC
dig DS whmcs.example.com +short
dnssec-verifier whmcs.example.com
```

## Troubleshooting DNS Issues

### Common Problems
```bash
# DNS not resolving
dig +trace whmcs.example.com

# Check nameservers
whois example.com | grep -i "name server"

# Verify A record propagation
for ns in $(dig NS example.com +short); do
    echo "NS: $ns"
    dig @$ns whmcs.example.com A +short
done

# Email issues
dig MX example.com +short
dig TXT example.com +short
```

### DNS Propagation Times
| Record Type | Typical Propagation |
|-------------|-------------------|
| A/AAAA | 5 minutes - 48 hours |
| MX | 15 minutes - 24 hours |
| TXT | 5 minutes - 48 hours |
| CNAME | 5 minutes - 48 hours |
| NS | 24 - 72 hours |

## Checklist
- [ ] A record configured
- [ ] MX record configured
- [ ] SPF record configured
- [ ] DKIM record configured
- [ ] DMARC record configured
- [ ] CAA record configured
- [ ] Reverse DNS verified
- [ ] DNSSEC enabled
- [ ] Propagation tested
- [ ] Email deliverability verified

## Tags
- dns
- configuration
- domain
- email
- security