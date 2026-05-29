# WHMCS Nameserver Change Workflow

## Purpose
Configure and manage nameserver changes for domains through WHMCS.

## Prerequisites
- WHMCS installation
- DNS management module
- Domain registrar integration
- DNS hosting configured

## Step-by-Step Process

### Step 1: Access Nameserver Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > Domain Nameservers`
3. Review nameserver options

### Step 2: Configure Default Nameservers
1. Set default nameservers:
   - Primary nameserver (ns1)
   - Secondary nameserver (ns2)
   - Additional nameservers
   - Default glue records
   - IP addresses for glue
2. Set nameserver templates

### Step 3: Configure Nameserver Options
1. Set up options:
   - Allow customer nameserver changes
   - Require verification for changes
   - Set allowed nameservers
   - Configure custom nameservers
   - Set child nameserver support
2. Set up nameserver validation

### Step 4: Configure Nameserver Selection
1. Set selection interface:
   - Default nameservers option
   - Custom nameservers option
   - DNS template selection
   - Registrar default option
   - Pre-configured options
2. Set display options

### Step 5: Configure Registrar Integration
1. Set registrar settings:
   - Update nameservers via registrar API
   - Set registrar-specific rules
   - Configure API authentication
   - Set update frequency
2. Set up sync with registrar

### Step 6: Configure DNS Template
1. Set up DNS templates:
   - Template for default nameservers
   - Template for custom nameservers
   - Zone file configuration
   - Record templates
   - Email configuration
2. Set up template assignment

### Step 7: Configure Customer Workflow
1. Set up customer interface:
   - Nameserver change form
   - Change request submission
   - Verification process
   - Confirmation display
   - Change history
2. Set up self-service

### Step 8: Configure Validation
1. Set validation rules:
   - Nameserver format validation
   - IP address validation
   - Reverse DNS check
   - Propagation check
   - Registrar compatibility check
2. Set up error handling

### Step 9: Configure Change Process
1. Set up process:
   - Submit change to registrar
   - Verify change applied
   - Update DNS zones
   - Send confirmation email
   - Update domain status
2. Set up approval workflow

### Step 10: Monitor Nameserver Changes
1. Track changes:
   - Change frequency
   - Common issues
   - Propagation times
   - Customer requests
   - Registrar errors
2. Generate reports

## Verification Checklist
- [ ] Default nameservers configured
- [ ] Customer changes work
- [ ] Registrar updates function
- [ ] DNS zones update
- [ ] Confirmation emails send

## Related Workflows
- whmcs-zone-management
- whmcs-domain-registration-flow
- whmcs-dnssec-setup
- whmcs-subdomain-automation

## Nameserver Best Practices
- Use reliable DNS hosting
- Set up monitoring
- Communicate propagation time
- Verify changes promptly
- Keep nameserver records updated