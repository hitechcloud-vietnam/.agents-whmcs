# WHMCS TXT Records Configuration Workflow

## Purpose
Configure and manage TXT records for domain verification, SPF, DKIM, DMARC, and other DNS-based services.

## Prerequisites
- WHMCS installation
- DNS management module
- Service requirements identified

## Step-by-Step Process

### Step 1: Access TXT Record Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > DNS Records`
3. Select TXT record configuration

### Step 2: Configure SPF Records
1. Set up SPF (Sender Policy Framework):
   - Basic SPF record
   - Include third-party mail servers
   - Include email service providers
   - Set all mechanism (+all, -all, ~all)
   - Multiple SPF records handling
2. Set up SPF template

### Step 3: Configure DKIM Records
1. Set up DKIM (DomainKeys Identified Mail):
   - DKIM selector (_domainkey)
   - Public key record
   - DKIM value format
   - Multiple DKIM selectors
   - Third-party DKIM
2. Set up DKIM templates

### Step 4: Configure DMARC Records
1. Set up DMARC (Domain-based Message Authentication):
   - DMARC policy (none, quarantine, reject)
   - Reporting URIs
   - Alignment settings
   - Percentage for testing
   - Forensic reporting
2. Set up DMARC templates

### Step 5: Configure Verification Records
1. Set up verification TXT:
   - Google Workspace verification
   - Microsoft 365 verification
   - Other service verification
   - Multiple verification records
   - Verification value handling
2. Set up verification templates

### Step 6: Configure Custom TXT Records
1. Set up custom records:
   - Domain ownership verification
   - Service configuration
   - API key verification
   - Multi-purpose TXT records
   - Custom value formatting
2. Set up record limits

### Step 7: Configure TXT Record Syntax
1. Set up validation:
   - Character limits (255 per string)
   - Multiple strings handling
   - Escape character handling
   - Quotation mark handling
   - Length validation
2. Set syntax templates

### Step 8: Configure TXT Propagation
1. Set up propagation:
   - TTL values for TXT
   - Propagation time
   - Verification after update
   - Conflict resolution
   - Update timing
2. Set propagation monitoring

### Step 9: Configure Customer Access
1. Set up customer interface:
   - TXT record view
   - Template selection
   - Custom record creation
   - Verification check
   - Documentation links
2. Set up self-service

### Step 10: Configure Monitoring
1. Set up monitoring:
   - TXT record status
   - SPF/DKIM/DMARC validation
   - Delivery issue detection
   - Configuration changes
   - Expiration tracking
2. Set up alerts

## Verification Checklist
- [ ] SPF record configured
- [ ] DKIM records added
- [ ] DMARC policy set
- [ ] Verification records added
- [ ] Propagation verified

## Related Workflows
- whmcs-mx-records
- whmcs-zone-management
- whmcs-txt-records
- whmcs-dnssec-setup

## TXT Records Best Practices
- Keep SPF records simple
- Use DKIM for email signing
- Set DMARC policy gradually
- Verify before production
- Monitor authentication results