# WHMCS CNAME Records Setup Workflow

## Purpose
Configure and manage CNAME (Canonical Name) records for domain aliases and service routing.

## Prerequisites
- WHMCS installation
- DNS management module
- Service integration requirements

## Step-by-Step Process

### Step 1: Access CNAME Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > DNS Records`
3. Select CNAME record configuration

### Step 2: Configure Default CNAMEs
1. Set up defaults:
   - www CNAME (to main domain)
   - mail CNAME
   - ftp CNAME
   - webmail CNAME
   - admin CNAME
2. Set up CNAME templates

### Step 3: Configure Service CNAMEs
1. Set up service records:
   - Google Workspace CNAME
   - Microsoft 365 CNAME
   - Cloudflare CNAME
   - CDN CNAMEs
   - Analytics CNAMEs
2. Set up verification

### Step 4: Configure CNAME Validation
1. Set up validation:
   - Target domain validation
   - No circular references
   - No CNAME chains (RFC compliance)
   - Target existence check
   - Alias resolution
2. Set validation rules

### Step 5: Configure CNAME Templates
1. Set up templates:
   - Common service templates
   - Provider-specific templates
   - Custom templates
   - Template versioning
   - Template inheritance
2. Set up template management

### Step 6: Configure Alias Management
1. Set up aliases:
   - Multiple aliases per service
   - Alias to alias handling
   - Alias limits
   - Alias cleanup
   - Alias history
2. Set up management rules

### Step 7: Configure CNAME Propagation
1. Set up propagation:
   - TTL values for CNAME
   - Propagation monitoring
   - Update verification
   - Target resolution check
   - Cache management
2. Set propagation alerts

### Step 8: Configure Customer Access
1. Set up customer interface:
   - CNAME view
   - Template selection
   - Custom CNAME creation
   - Alias management
   - Validation feedback
2. Set up self-service

### Step 9: Configure CNAME Limits
1. Set limits:
   - Maximum CNAMEs per domain
   - Maximum alias depth
   - Target domain restrictions
   - Reserved prefixes
   - Resource limits
2. Set up quota management

### Step 10: Monitor CNAME Usage
1. Set up monitoring:
   - CNAME status
   - Target availability
   - Propagation status
   - Change tracking
   - Error detection
2. Generate reports

## Verification Checklist
- [ ] CNAMEs configured correctly
- [ ] Templates applied properly
- [ ] Validation works
- [ ] Propagation complete
- [ ] Customer access functions

## Related Workflows
- whmcs-zone-management
- whmcs-a-record-config
- whmcs-subdomain-automation
- whmcs-domain-forwarding-setup

## CNAME Records Best Practices
- Use CNAME for aliases only
- Avoid CNAME chains
- Set reasonable TTL values
- Verify target existence
- Monitor target availability