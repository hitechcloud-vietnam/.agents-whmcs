# WHMCS MX Records Setup Workflow

## Purpose
Configure and manage MX (Mail Exchange) records for domain email routing.

## Prerequisites
- WHMCS installation
- DNS management module
- Mail server information available

## Step-by-Step Process

### Step 1: Access MX Record Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > DNS Records`
3. Select MX record configuration

### Step 2: Configure Default MX Records
1. Set default MX:
   - Primary mail server (mx1.domain.com)
   - Priority (10-50)
   - Secondary mail server (mx2.domain.com)
   - Backup mail server (mx3.domain.com)
2. Set MX priority defaults

### Step 3: Configure MX Templates
1. Set up templates:
   - Default email template
   - Google Workspace template
   - Microsoft 365 template
   - Custom mail provider templates
   - Multiple MX templates
2. Set template variables

### Step 4: Configure MX Priority
1. Set priority rules:
   - Lower number = higher priority
   - Primary (10)
   - Secondary (20)
   - Tertiary (30)
   - Backup (40-50)
2. Set up failover

### Step 5: Configure Mail Server Settings
1. Set mail server info:
   - Mail server hostname
   - IP addresses
   - SSL/TLS requirements
   - Port settings
   - Authentication requirements
2. Set up validation

### Step 6: Configure MX Propagation
1. Set up propagation:
   - TTL values for MX records
   - Propagation time expectations
   - Verification methods
   - Propagation monitoring
   - Update timing
2. Set up conflict handling

### Step 7: Configure Email Security
1. Set up security:
   - SPF record for mail servers
   - DKIM record configuration
   - DMARC record setup
   - Mail encryption requirements
   - Anti-spoofing measures
2. Set security validation

### Step 8: Configure Customer MX Setup
1. Set up customer interface:
   - MX record view
   - MX template selection
   - Custom MX configuration
   - Email provider presets
   - Validation feedback
2. Set up self-service

### Step 9: Configure Monitoring
1. Set up monitoring:
   - MX record status check
   - Mail server connectivity
   - Delivery tracking
   - Error detection
   - Blacklist monitoring
2. Set up alerts

### Step 10: Configure Troubleshooting
1. Set up diagnostics:
   - MX lookup tool
   - Propagation checker
   - Mail server tester
   - SPF/DKIM/DMARC validator
   - Common issue resolution
2. Set up support tools

## Verification Checklist
- [ ] MX records configured
- [ ] Priority set correctly
- [ ] Propagation complete
- [ ] Email routing works
- [ ] Security records added

## Related Workflows
- whmcs-zone-management
- whmcs-txt-records
- whmcs-a-record-config
- whmcs-dnssec-setup

## MX Records Best Practices
- Use multiple MX servers
- Set proper priorities
- Always include SPF record
- Set reasonable TTL values
- Monitor mail delivery