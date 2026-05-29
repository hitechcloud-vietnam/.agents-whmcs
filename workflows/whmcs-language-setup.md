# WHMCS Language Setup Workflow

## Purpose
Configure multiple languages for WHMCS

## Prerequisites
- WHMCS installed
- Admin access

## Step 1: Navigate to Language Settings

Navigate to: Setup > General Settings > Localisation

## Step 2: Set Default Language

```
Default Language: English (UK) / English (US)
Available Languages: [select installed]
```

## Step 3: Install New Language

### Download Language Pack
1. Get language pack from WHMCS Marketplace or community
2. Upload to: `/var/www/whmcs/langauges/`

### Install via Admin
Navigate to: Setup > General Settings > Localisation

1. Click "Install Language"
2. Select language file
3. Install

## Step 4: Configure Language Files

```bash
cd /var/www/whmcs/langauges
ls -la
```

### Common Languages
- English (UK) - default
- English (US)
- German
- French
- Spanish
- Dutch
- Italian
- Portuguese

## Step 5: Customize Language Strings

Navigate to: Setup > General Settings > Localisation > Language Strings

### Edit Language
1. Select language
2. Search for string
3. Edit translation
4. Save

### Example Strings
```php
$_LANG['accountstats'] = 'Account Statistics';
$_LANG['accountoverview'] = 'Account Overview';
$_LANG['addfunds'] = 'Add Funds';
```

## Step 6: Configure Language Switcher

Navigate to: Setup > Client Area Design > Theme Settings

```
Show Language Selector: Yes
Language Selector Position: Header / Footer
Default Client Language: [auto-detect]
```

## Step 7: Set Up RTL Languages

For Arabic, Hebrew, etc.:

Navigate to: Setup > General Settings > Localisation

```
RTL Mode: Enabled
Default RTL Language: Arabic
```

## Step 8: Configure Email Language

Navigate to: Setup > Email > Email Templates

For each language, configure email templates in that language.

## Step 9: Test Languages

1. Go to client area
2. Switch languages
3. Verify all strings translate
4. Check layout (RTL)

## Language Configuration Checklist

- [ ] Default language set
- [ ] Additional languages installed
- [ ] Strings customized
- [ ] Language switcher enabled
- [ ] RTL configured (if needed)
- [ ] Email templates translated
- [ ] Languages tested
