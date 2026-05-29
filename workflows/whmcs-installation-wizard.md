# WHMCS Installation Wizard Workflow

## Purpose
Complete WHMCS installation using the web-based installation wizard

## Prerequisites
- Server requirements verified
- Prerequisites installed
- WHMCS license key
- Database credentials ready

## Step 1: Download WHMCS

### Method A: Download from WHMCS Website
1. Log in to your WHMCS account at https://www.whmcs.com/members/
2. Navigate to Downloads
3. Download the latest WHMCS release
4. Upload via FTP/SFTP to your server

### Method B: Command Line Download
```bash
cd /var/www/whmcs
wget https://downloads.whmcs.com/whmcs-latest.zip
unzip whmcs-latest.zip
mv whmcs/* ./
mv whmcs/.* . 2>/dev/null || true
rm -rf whmcs whmcs-latest.zip
```

## Step 2: Set Permissions

```bash
cd /var/www/whmcs

# Set ownership
chown -R www-data:www-data .

# Set directory permissions
chmod 755 .
chmod 755 admin
chmod 755 apertures
chmod 755 attachments
chmod 755 cache
chmod 755 certificates
chmod 755 downloads
chmod 755 logs
chmod 755 templates_c
chmod 755 resources/domains

# Set file permissions
find . -type f -exec chmod 644 {} \;

# Set specific required permissions
chmod 644 configuration.php.new
chmod 750 admin
chmod 640 configuration.php
```

## Step 3: Create Empty configuration.php

```bash
cd /var/www/whmcs
touch configuration.php
chown www-data:www-data configuration.php
chmod 400 configuration.php
```

## Step 4: Access Installation Wizard

1. Open your browser
2. Navigate to: `https://yourdomain.com/install/install.php`
3. OR navigate to: `https://yourdomain.com/admin/install.php` for fresh install

## Step 5: Installation Wizard - Welcome Screen

1. Review license agreement
2. Check system requirements (all should show green checkmarks)
3. Click "Next" to continue

## Step 6: License Key Entry

1. Enter your WHMCS license key
2. Click "Next"

**License Key Location:**
- WHMCS Members Area > License Key Management
- Format: XXXX-XXXX-XXXX-XXXX

## Step 7: Database Configuration

Enter your database details:

| Field | Value |
|-------|-------|
| Database Host | localhost |
| Database Name | whmcs_db |
| Database Username | whmcs_user |
| Database Password | [your-password] |
| Table Prefix | whmcs_ |

Click "Next" to test connection and create tables.

**Troubleshooting:**
- If connection fails, verify MySQL is running
- Check credentials match what you created earlier
- Ensure MySQL user has proper permissions

## Step 8: WHMCS Admin Account

Create your primary admin account:

| Field | Value |
|-------|-------|
| Admin Username | admin (or custom) |
| Admin Password | [strong-password] |
| Confirm Password | [same-password] |
| Admin Email | admin@yourdomain.com |

**Password Requirements:**
- Minimum 12 characters
- At least one uppercase letter
- At least one number
- At least one special character

## Step 9: Company Information

Enter your company details:

| Field | Value |
|-------|-------|
| Company Name | Your Company Name |
| Email Address | admin@yourdomain.com |
| Domain | https://yourdomain.com |
| Phone Number | +1-555-555-5555 |
| Address | Full street address |
| City | City |
| State/Region | State/Province |
| Country | Select from dropdown |
| Postcode | Postal code |

## Step 10: Cron Job Configuration

The wizard will display the required cron command:

```bash
0,30 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php
```

Copy this command - you will configure it in Step 12.

## Step 11: Complete Installation

1. Review all settings
2. Click "Complete Installation"
3. Wait for database tables to be created

## Step 12: Configure Cron Job

```bash
crontab -e
```

Add the cron job:
```cron
# WHMCS Cron - Every 30 minutes
0,30 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php

# Alternative: Run every 5 minutes
*/5 * * * * /usr/bin/php /var/www/whmcs/crons/cron.php
```

## Step 13: Secure configuration.php

After installation, the system may update permissions automatically. Verify:

```bash
cd /var/www/whmcs
chmod 400 configuration.php
chown www-data:www-data configuration.php
```

## Step 14: Remove Installation Directory

```bash
rm -rf /var/www/whmcs/install
```

**Important:** This is a security requirement. Do not skip this step.

## Step 15: Test Admin Access

1. Navigate to: `https://yourdomain.com/admin/`
2. Log in with your admin credentials
3. Verify dashboard loads correctly

## Step 16: Initial WHMCS Configuration

### Configure General Settings
Navigate to: Setup > General Settings

1. **Basic Settings Tab:**
   - Company Name
   - Logo
   - Email Address
   - Domain URL

2. **Localisation Tab:**
   - Default Language
   - Timezone
   - Date Format
   - Currency

3. **Security Tab:**
   - Enable two-factor authentication
   - Set session timeout
   - Configure IP restriction

### Configure Payments
Navigate to: Setup > Payments > Payment Gateways

1. Click "Available Gateways"
2. Activate desired gateways:
   - PayPal
   - Stripe
   - Bank Transfer
3. Configure gateway credentials

### Configure Email Templates
Navigate to: Setup > Email Templates > Email Templates

1. Review default templates
2. Customize for your brand
3. Configure SMTP settings if needed

## Step 17: Post-Installation Checklist

- [ ] Installation wizard completed
- [ ] Admin account created
- [ ] Cron job configured
- [ ] Install directory removed
- [ ] configuration.php secured
- [ ] Admin area accessible
- [ ] Basic settings configured
- [ ] Payment gateways configured
- [ ] Email templates reviewed

## Step 18: Verify System Health

Navigate to: Utilities > System Health Status

Check:
- System requirements
- Cron status
- Database connection
- License status

## Troubleshooting Common Issues

### Blank Page After Installation
```bash
# Enable error display temporarily
nano /var/www/whmcs/configuration.php
# Add: $display_errors = true;

# Check PHP error log
tail -f /var/log/apache2/error.log
```

### Database Connection Error
```bash
# Test MySQL connection
mysql -u whmcs_user -p whmcs_db
```

### Permission Denied Errors
```bash
# Reset permissions
cd /var/www/whmcs
chown -R www-data:www-data .
chmod 755 .
chmod 755 admin
chmod 755 templates_c
chmod 755 cache
chmod 755 downloads
chmod 755 attachments
chmod 644 *.php
```

## Next Steps
Proceed to: `whmcs-post-install.md`
