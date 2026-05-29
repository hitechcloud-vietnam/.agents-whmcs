# WHMCS Domain Checker Configuration Workflow

## Purpose
Set up domain availability checking and registration

## Prerequisites
- WHMCS installed
- Domain registrar module configured
- Admin access

## Step 1: Navigate to Domain Checker Settings

Navigate to: Setup > Products/Services > Domain Checker

## Step 2: Enable Domain Checker

```
Enable Domain Checker: Yes
Show in Client Area: Yes
Allow Domain Registration: Yes
Allow Domain Transfer: Yes
```

## Step 3: Configure Domain Pricing

Navigate to: Setup > Products/Services > Domain Pricing

### Add TLDs
1. Click "Add TLD"
2. Configure:
   ```
   TLD: .com
   Registration Price: $10.00
   Renewal Price: $12.00
   Transfer Price: $12.00
   Restore Price: $50.00
   ```
3. Save

### Common TLDs to Add
```
.com, .net, .org, .info, .biz
.co, .io, .app, .dev, .tech
.uk, .eu, .de, .fr, .ca
```

## Step 4: Configure Domain Pricing Groups

Navigate to: Setup > Products/Services > Domain Pricing > Pricing Groups

### Create Group
```
Group Name: Standard TLDs
TLDs: .com, .net, .org
```

### Default Groups
```
Standard: .com, .net, .org
Premium: .io, .app, .dev
Regional: .uk, .eu, .de
```

## Step 5: Set Up Domain Addons

Navigate to: Setup > Products/Services > Domain Pricing > Addons

### Configure Addons
```
ID Protection:
  - Annual Price: $8.00
  - Enable by Default: No

DNS Management:
  - Annual Price: $2.00
  - Enable by Default: Yes

Email Forwarding:
  - Annual Price: $2.00
```

## Step 6: Configure Domain Search Options

Navigate to: Setup > Products/Services > Domain Checker > Search Options

```
Bulk Search Limit: 20
IDN Support: Yes
Search Suggestions: Yes
Show Alternative TLDs: Yes
Auto-Select Alternatives: No
```

## Step 7: Set Up Domain Transfer

Navigate to: Setup > Products/Services > Domain Pricing > Transfer Settings

```
Allow Transfers: Yes
Transfer Lock Required: Yes
EPP Code Required: Yes
Transfer Price: [set per TLD]
```

## Step 8: Configure Domain Renewal

Navigate to: Setup > Products/Services > Domain Pricing > Renewal Settings

```
Auto-Renewal: Enabled
Renewal Grace Period: 30 days
Redemption Grace Period: 30 days
Renewal Reminders: 30, 14, 7, 1 days
```

## Step 9: Set Up Domain Sync

Navigate to: Setup > Products/Services > Domain Registrars

Enable auto-sync for registered domains:
```
Auto-Sync Domains: Yes
Sync Interval: Every 6 hours
Update Expiry Dates: Yes
Update Nameservers: Yes
```

## Step 10: Configure Domain Ordering

Navigate to: Setup > Products/Services > Domain Pricing > Order Settings

```
Require Hosting Package: No
Allow Pre-registration: Yes
Pre-Registration TLDs: [select]
Pre-Registration Price: $15.00
```

## Step 11: Set Up Premium Domain Pricing

Navigate to: Setup > Products/Services > Domain Pricing > Premium Domains

```
Enable Premium Domains: Yes
Sync with Registrar: Yes
Price Markups: [percentage or fixed]
```

## Step 12: Customize Domain Checker Template

Navigate to: Setup > Client Area Design > Order Form Templates

Modify domain checker appearance:
```smarty
<div class="domain-checker">
    <input type="text" name="domain" placeholder="Enter domain name">
    <button type="submit">Search</button>
</div>
```

## Domain Checker Checklist

- [ ] Domain checker enabled
- [ ] TLDs added with pricing
- [ ] Pricing groups created
- [ ] Addons configured
- [ ] Search options set
- [ ] Transfer settings configured
- [ ] Renewal settings set
- [ ] Domain sync enabled
- [ ] Premium domains configured
- [ ] Template customized
