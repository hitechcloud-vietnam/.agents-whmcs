# WHMCS Addon Module Setup Workflow

## Purpose
Install and configure WHMCS addon modules

## Prerequisites
- WHMCS installed
- Admin access
- Addon module files

## Step 1: Navigate to Addon Modules

Navigate to: Setup > Addon Modules

## Step 2: Install Addon Module

### Method A: Automatic Install
1. Purchase addon from WHMCS Marketplace
2. Download from client area
3. Upload to WHMCS admin

### Method B: Manual Install
```bash
# Upload addon files
cd /var/www/whmcs/modules/addons
upload your_addon/

# Set permissions
chown -R www-data:www-data your_addon/
chmod 755 your_addon/
```

## Step 3: Activate Addon

Navigate to: Setup > Addon Modules

1. Find addon
2. Click "Activate"
3. Configure license key (if required)

## Step 4: Configure Addon Settings

1. Click "Configure"
2. Set module-specific options:
   ```
   API Key: [your-key]
   Username: [username]
   Default Settings: [configure]
   ```
3. Save

## Step 5: Set Module Permissions

Navigate to: Setup > Addon Modules > Permissions

1. Select addon
2. Assign to admin roles:
   ```
   Full Access: [selected admins]
   Read-Only: [selected admins]
   ```
3. Save

## Step 6: Configure Hooks (if required)

Some addons require hook integration:

```php
// Add to configuration.php
$addons = [
    'your_addon' => true,
];
```

## Common Addon Modules

| Addon | Purpose |
|-------|---------|
| Project Management | Track client projects |
| Live Chat | Customer support chat |
| SMS Notifications | SMS alerts |
| Backup Module | Automated backups |
| Custom Widgets | Dashboard enhancements |

## Addon Configuration Checklist

- [ ] Module installed
- [ ] Module activated
- [ ] Settings configured
- [ ] Permissions set
- [ ] License validated
- [ ] Hooks configured
