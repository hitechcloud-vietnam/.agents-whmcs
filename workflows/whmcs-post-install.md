# WHMCS Post-Installation Workflow

## Purpose
Complete essential post-installation configuration and hardening

## Prerequisites
- WHMCS installation completed
- Admin access to WHMCS
- WHMCS license active

## Step 1: Log into WHMCS Admin

1. Navigate to: `https://yourdomain.com/admin/`
2. Enter admin credentials
3. Complete two-factor authentication if enabled

## Step 2: Initial Security Hardening

### Change Admin URL
Navigate to: Setup > General Settings > Security

1. Enable "Admin Directory Alias"
2. Set custom admin path (e.g., `/admin_secure_xyz123/`)

**Manual Configuration:**
```bash
cd /var/www/whmcs/admin
mv index.php index.php.bak
cp ../index.php index.php
nano ../admin/index.php
# Add authentication check
```

### Set Admin Session Security
Navigate to: Setup > General Settings > Security

- Enable "Require Two-Factor Authentication for Admin Login"
- Set "Concurrent Admin Sessions" to 1
- Enable "Admin IP Address Restriction" (optional)

### Configure Failed Login Limits
Navigate to: Setup > General Settings > Security

- Max failed login attempts: 5
- Lockout duration: 15 minutes
- Enable CAPTCHA on login

## Step 3: Configure General Settings

Navigate to: Setup > General Settings

### Basic Settings Tab
```
Company Name: [Your Company Name]
Website Address: https://yourdomain.com/
Site SSL URL: https://yourdomain.com/
Server IP Address: [Your Server IP]
Support Phone Number: [Phone Number]
Email Address: [Admin Email]
Charset: UTF-8
```

### Security Tab
```
Require Email Verification: Yes
Require Two-Factor Authentication: Yes
Enable API Access: [As needed]
```

### Maintenance Mode
```
Enable Maintenance Mode: Off (unless needed)
Maintenance Mode Message: [Custom message]
Allowed IPs for Maintenance: [Your IP]
```

## Step 4: Configure Localization

Navigate to: Setup > General Settings > Localisation

```
Default Language: English
Additional Languages: [Add as needed]
Timezone: [Your timezone]
Date Format: dd/MM/yyyy
Currency: USD
```

## Step 5: Set Up Payment Gateways

Navigate to: Setup > Payments > Payment Gateways

### Activate PayPal
1. Click "All Payment Gateways"
2. Find "PayPal" and click "Activate"
3. Configure:
   - PayPal Email Address
   - Payment Gateway Mode: Live/Sandbox

### Activate Stripe
1. Click "All Payment Gateways"
2. Find "Stripe" and click "Activate"
3. Configure:
   - Publishable Key
   - Secret Key
   - Webhook Secret

### Configure Bank Transfer
1. Activate "Bank Transfer"
2. Configure:
   - Bank Name
   - Account Name
   - Account Number
   - Sort Code/IBAN
   - SWIFT/BIC Code

## Step 6: Configure Tax Settings

Navigate to: Setup > Payments > Tax Rules

### Add Tax Rule
1. Click "Add Tax Rule"
2. Configure:
   - Tax Name: VAT / Sales Tax
   - Tax Rate: 20%
   - Country: [Default Country]
   - State/Region: [All or specific]
   - Products: Yes
   - Late Fees: Yes

### Tax Configuration
- Enable "Tax Exempt" toggle
- Set default tax status for new clients

## Step 7: Configure Email Settings

Navigate to: Setup > Email > Mail Configuration

### SMTP Configuration (Recommended)
```
Mail Type: SMTP
Mail Encoding: UTF-8
SMTP Host: smtp.yourprovider.com
SMTP Port: 587
SMTP Username: your-email@domain.com
SMTP Password: [your-password]
SMTP Security: TLS
```

### Email Branding
Navigate to: Setup > Email > Email Templates

1. Select each template
2. Customize with:
   - Company logo
   - Brand colors
   - Footer information
3. Save changes

### Essential Templates to Customize
- Client Welcome Email
- Password Reset Email
- Invoice Created
- Invoice Paid Confirmation
- Support Ticket Opened
- Support Ticket Reply

## Step 8: Configure Support Department

Navigate to: Setup > Support > Support Departments

### Create Support Department
1. Click "Create New Department"
2. Configure:
   - Department Name
   - Department Email
   - Administrator
   - Client visibility: Yes

### Department Settings
```
Ticket Import: Enabled
Private Notes: Yes
Feedback Survey: Yes
Auto-Lock Tickets: Yes (after 7 days)
```

## Step 9: Configure Client Groups

Navigate to: Setup > Clients > Client Groups

### Create Groups
1. Click "Create New Group"
2. Configure:
   - Group Name
   - Group Colour
   - Discount: [As needed]
   - Suspension Fee Override: [As needed]

### Default Groups
- Default (no discount)
- VIP (5% discount)
- Reseller (10% discount)

## Step 10: Set Up Currency

Navigate to: Setup > Payments > Currencies

### Add Currencies
1. Click "Add Currency"
2. Configure:
   - Currency Code: EUR, GBP, etc.
   - Currency Symbol: €, £, etc.
   - Format: [Symbol] [Amount]
   - Update Rates: [Auto/Manual]

### Set Base Currency
```
Base Currency: USD
Auto-update exchange rates: Yes (daily)
```

## Step 11: Configure Automation (Cron)

Navigate to: Utilities > System Health Status

Verify cron is running:
1. Check "Last Cron Run" timestamp
2. Should be within last 30 minutes

### Manual Cron Verification
```bash
# Run cron manually
/usr/bin/php /var/www/whmcs/crons/cron.php

# Check output for errors
```

### Cron Automation Tasks
Ensure these are enabled in: Setup > Automation Settings
- Generate Invoices
- Payment Reminders
- Ticket Auto-Close
- Domain Status Updates
- Suspension/Deletion
- Affiliate Commission

## Step 12: Configure MarketConnect (Optional)

Navigate to: Setup > MarketConnect

1. Review available services:
   - SSL Certificates
   - SiteLock
   - CodeGuard
   - SEO Tools

2. Activate desired services
3. Configure pricing

## Step 13: Set Up Notifications

Navigate to: Setup > Notifications > Notification Preferences

### Configure Admin Notifications
```
New Order Placed: Yes
Invoice Created: Yes
Payment Received: Yes
Support Ticket Opened: Yes
Domain Renewal Reminder: Yes
```

## Step 14: Configure API Access

Navigate to: Setup > System Settings > API Credentials

### Create API Key
1. Click "Create New API Credential"
2. Configure:
   - API User: [Username]
   - Access Key: [Auto-generated]
   - Permissions: [Select as needed]
   - IP Access List: [Your IPs]

### API Access Permissions
- Full Access (for admin use)
- Specific permissions for integrations

## Step 15: Set Up Activity Log

Navigate to: Setup > System Settings > Activity Logs

```
Log Admin Activity: Yes
Log Client Activity: Yes
Log Retention: 90 days
```

## Step 16: Configure CAPTCHA

Navigate to: Setup > General Settings > Security

### reCAPTCHA Setup
1. Register at Google reCAPTCHA
2. Get Site Key and Secret Key
3. Configure:
   - reCAPTCHA Type: v2 Checkbox
   - Site Key: [Your Key]
   - Secret Key: [Your Key]
   - Enable on: Login, Register, Contact

## Step 17: Create First Product

Navigate to: Setup > Products/Services > Products/Services

### Create Product Group
1. Click "Create New Group"
2. Configure:
   - Group Name: Hosting
   - Group Headline: Web Hosting Services
   - Description: [Description]

### Create Product
1. Click "Create New Product"
2. Configure:
   - Product Type: Reseller Hosting / Shared Hosting
   - Product Group: Hosting
   - Product Name: Starter Hosting
   - Module: cPanel (if using provisioning)

## Step 18: Configure Order Form

Navigate to: Setup > Payments > Order Form Settings

### Order Form Options
```
Enable Order Form: Yes
Default Order Form Template: Six / Standard Form
Show in Client Area: Yes
Allow Quantity: No
```

### Product Ordering
- Drag and drop to set order priority
- Enable/disable products for ordering

## Step 19: Configure Domain Pricing

Navigate to: Setup > Products/Services > Domain Pricing

### TLD Configuration
1. Add new TLDs
2. Set registration prices
3. Set renewal prices
4. Set transfer prices
5. Enable/disable TLDs

### Domain Registration Settings
```
Auto-registration: Yes (with registrar)
Default registration years: 1
Enable domain privacy: Yes
```

## Step 20: Final Verification

### System Health Check
Navigate to: Utilities > System Health Status

Verify all checks pass:
- [ ] System requirements met
- [ ] Cron running correctly
- [ ] Database connection stable
- [ ] License active
- [ ] No error logs

### Test Core Functions
1. **Place Test Order**
   - Go to order page
   - Complete checkout
   - Verify invoice created

2. **Test Payment**
   - Pay test invoice
   - Verify payment recorded

3. **Test Ticket**
   - Submit support ticket
   - Reply to ticket
   - Verify email notifications

## Post-Installation Checklist

- [ ] Admin area secured (custom path)
- [ ] Two-factor authentication enabled
- [ ] General settings configured
- [ ] Payment gateways activated
- [ ] Tax rules configured
- [ ] Email/SMTP configured
- [ ] Support departments created
- [ ] Client groups set up
- [ ] Currency configured
- [ ] Cron job verified
- [ ] MarketConnect configured
- [ ] Notifications set up
- [ ] API access configured
- [ ] CAPTCHA enabled
- [ ] First product created
- [ ] Order form configured
- [ ] Domain pricing set
- [ ] System health verified

## Next Steps
Proceed to: `whmcs-ssl-setup.md` or `whmcs-cron-setup.md`
