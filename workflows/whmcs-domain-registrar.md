# WHMCS Domain Registrar Setup Workflow

## Purpose
Configure domain registrar modules for domain registration/transfer/renewal

## Prerequisites
- WHMCS installed
- Registrar account (enom, ResellerClub, etc.)
- Admin access

## Step 1: Navigate to Domain Registrars

Navigate to: Setup > Products/Services > Domain Registrars

## Step 2: Choose Registrar Module

Available registrars include:
- Enom
- ResellerClub
- OpenSRS
- RealtimeRegister
- hexonet
- OVH
- Custom API

## Step 3: Activate Registrar

1. Click registrar name
2. Enter credentials:
   ```
   Username: [registrar-username]
   Password: [registrar-password]
   API Key: [if applicable]
   ```
3. Save configuration

## Step 4: Configure Registrar Settings

Navigate to: Setup > Products/Services > Domain Registrars > Registrar Settings

```
Auto-registration: Yes
Default Registration Years: 1
Enable Domain Sync: Yes
Sync Interval: Daily
```

## Step 5: Configure TLDs

Navigate to: Setup > Products/Services > Domain Pricing

### Add TLD
1. Click "Add TLD"
2. Configure:
   ```
   TLD: .com
   Registration Price: $10.00
   Renewal Price: $12.00
   Transfer Price: $12.00
   ```
3. Enable registrar sync

## Step 6: Set Up Domain Pricing Groups

Navigate to: Setup > Products/Services > Domain Pricing

### Create Pricing Group
1. Click "Add Group"
2. Name: "Standard"
3. Add TLDs with pricing
4. Save

## Step 7: Configure Domain Sync

Navigate to: Setup > Products/Services > Domain Registrars > Sync

```bash
# Manual domain sync
/usr/bin/php /var/www/whmcs/crons/domainssync.php
```

### Cron for Domain Sync
```cron
# Run domain sync every 6 hours
0 */6 * * * /usr/bin/php /var/www/whmcs/crons/domainssync.php
```

## Step 8: Configure Domain Transfer

Navigate to: Setup > Products/Services > Domain Pricing

### Transfer Settings
```
Allow Transfers: Yes
Transfer Lock Required: Yes
EPP Code Required: Yes
```

## Step 9: Set Up Domain Addons

Navigate to: Setup > Products/Services > Domain Pricing > Addons

```
DNS Management: $2.00/year
Email Forwarding: $2.00/year
ID Protection: $8.00/year
```

## Step 10: Configure Domain Renewals

Navigate to: Setup > Products/Services > Domain Pricing

### Renewal Settings
```
Auto-renewal Reminder Days: 30, 14, 7, 1
Auto-renewal Enabled: Optional
Auto-renewal Grace Period: 30 days
```

## Step 11: Test Domain Registration

1. Search for domain
2. Attempt registration
3. Verify in registrar account

## Step 12: Configure Domain Transfer-In

Navigate to: Setup > Products/Services > Domain Registrars

Enable transfer-in notifications and EPP verification.

## Domain Registrar Troubleshooting

### Registration Failed
- Verify registrar credentials
- Check API permissions
- Review transfer authorization

### Sync Not Working
```bash
# Enable debug
nano /var/www/whmcs/includes/registrar.php
# Set debug mode

# Test sync manually
/usr/bin/php /var/www/whmcs/crons/domainssync.php
```

## Verification Checklist

- [ ] Registrar module activated
- [ ] API credentials configured
- [ ] TLDs added with pricing
- [ ] Domain sync working
- [ ] Transfer settings configured
- [ ] Addons pricing set
- [ ] Test registration successful
