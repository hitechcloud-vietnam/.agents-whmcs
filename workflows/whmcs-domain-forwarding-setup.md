# WHMCS Domain Forwarding Setup Workflow

## Purpose
Configure and manage domain forwarding (redirects) to point domains to other URLs.

## Prerequisites
- WHMCS installation
- DNS management module
- Forwarding service configured

## Step-by-Step Process

### Step 1: Access Domain Forwarding Settings
1. Log into WHMCS admin
2. Navigate to `Configuration > Products/Services > Domain Forwarding`
3. Review forwarding options

### Step 2: Enable Domain Forwarding
1. Configure forwarding:
   - Enable forwarding feature
   - Set up forwarding service
   - Configure default redirects
   - Set forwarding permissions
2. Set global forwarding settings

### Step 3: Configure Forwarding Types
1. Set up forward types:
   - 301 Permanent redirect
   - 302 Temporary redirect
   - 303 See Other
   - 307 Temporary Redirect
   - Meta refresh redirect
   - Frame redirect (masked)
2. Set default redirect type

### Step 4: Configure Forwarding Rules
1. Set up rules:
   - Source domain forwarding
   - Subdomain forwarding
   - Wildcard forwarding
   - Path forwarding (preserve path)
   - Query string forwarding
2. Set up pattern matching

### Step 5: Configure Destination URLs
1. Set up destinations:
   - Single URL destination
   - Multiple destination options
   - Random destination selection
   - Geographic-based routing
   - Time-based routing
2. Set up URL templates

### Step 6: Configure Masking Options
1. Set up masking:
   - Hide destination URL
   - Show source domain
   - Preserve in address bar
   - Frame-based masking
   - Meta tag masking
2. Set up SEO options

### Step 7: Configure SEO Settings
1. Set up SEO options:
   - Pass-through PageRank
   - Meta tag preservation
   - Canonical URL handling
   - 404 handling
   - Sitemap integration
2. Set up search engine settings

### Step 8: Configure Forwarding Limits
1. Set limits:
   - Maximum redirects per domain
   - Redirect rate limits
   - Bandwidth limits
   - Click tracking limits
   - Storage limits
2. Set up quota management

### Step 9: Configure Monitoring
1. Set up tracking:
   - Click analytics
   - Traffic reports
   - Popular destinations
   - Error tracking
   - Conversion tracking
2. Set up alerts

### Step 10: Configure Customer Interface
1. Set up customer access:
   - View forwarding status
   - Create/edit redirects
   - View analytics
   - Set up wildcards
   - Cancel forwarding
2. Set up self-service

## Verification Checklist
- [ ] Forwarding configured correctly
- [ ] Redirects work properly
- [ ] Masking functions
- [ ] Analytics tracking works
- [ ] SEO settings applied

## Related Workflows
- whmcs-zone-management
- whmcs-cname-setup
- whmcs-subdomain-automation
- whmcs-nameserver-change

## Domain Forwarding Best Practices
- Use 301 for permanent moves
- Set up proper redirects
- Monitor for loops
- Track forwarding performance
- Provide fallback URLs