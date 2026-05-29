# WHMCS SSL Certificate Renewal Workflow

## Purpose
Renew SSL certificates for WHMCS

## Prerequisites
- SSH access
- Certbot installed (for Let's Encrypt)

## Step 1: Check Certificate Expiration

```bash
# Check expiration date
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com 2>/dev/null | openssl x509 -noout -dates

# Or use
certbot certificates
```

## Step 2: Renew Certificate

### Let's Encrypt
```bash
certbot renew
```

### Force Renew (if needed)
```bash
certbot renew --force-renewal
```

## Step 3: Verify Renewal

```bash
certbot certificates
```

Check:
- Valid from: [date]
- Expires: [date]
- Domains covered

## Step 4: Reload Web Server

```bash
# Apache
systemctl reload apache2

# Nginx
systemctl reload nginx
```

## Step 5: Test SSL

Visit: https://www.ssllabs.com/ssltest/

Verify:
- Certificate valid
- No mixed content
- Grade A or higher

## Step 6: Set Up Auto-Renewal

```bash
# Check certbot timer
systemctl status certbot.timer

# If not enabled
systemctl enable certbot.timer
systemctl start certbot.timer
```

## SSL Renewal Checklist

- [ ] Expiration checked
- [ ] Certificate renewed
- [ ] Renewal verified
- [ ] Web server reloaded
- [ ] SSL tested
- [ ] Auto-renewal enabled
