# WHMCS Branding Configuration Workflow

## Purpose
Configure company branding across WHMCS

## Prerequisites
- WHMCS installed
- Admin access
- Brand assets (logo, colors)

## Step 1: Configure Basic Settings

Navigate to: Setup > General Settings > General

```
Company Name: Your Company Name
Website Address: https://yourdomain.com
Email Address: info@yourdomain.com
Phone Number: +1-555-555-5555
Fax Number: [if applicable]
```

## Step 2: Upload Company Logo

Navigate to: Setup > Client Area Design > Theme Settings > Logo

Upload:
```
Logo (Light): [PNG, recommended 200x60px]
Logo (Dark): [PNG for dark backgrounds]
Favicon: [32x32 ICO/PNG]
Apple Touch Icon: [180x180 PNG]
```

## Step 3: Set Company Address

Navigate to: Setup > General Settings > General

```
Company Address: 123 Business Street
City: New York
State/Region: NY
Postcode: 10001
Country: United States
```

## Step 4: Configure Business Hours

Navigate to: Setup > General Settings > Business Hours

```
Monday: 9:00 AM - 6:00 PM
Tuesday: 9:00 AM - 6:00 PM
Wednesday: 9:00 AM - 6:00 PM
Thursday: 9:00 AM - 6:00 PM
Friday: 9:00 AM - 6:00 PM
Saturday: Closed
Sunday: Closed
Holiday Mode: [list holidays]
```

## Step 5: Set VAT/Tax Number

Navigate to: Setup > Payments > Tax Rules

```
Tax ID/VAT Number: GB123456789
Apply Tax: Yes/No
Tax Exemptions: [as needed]
```

## Step 6: Configure Email Branding

Navigate to: Setup > Email > Email Templates

For each template:
1. Select template
2. Upload logo
3. Customize colors
4. Update footer

### Email Header
```html
<div style="background:#0073aa;padding:20px;text-align:center;">
    <img src="{$logo}" alt="{$company_name}" height="50">
</div>
```

### Email Footer
```html
<div style="background:#f5f5f5;padding:20px;text-align:center;">
    <p>{$company_name}<br>
    {$company_address}<br>
    {$company_email} | {$company_phone}</p>
    <p>Company Number: 12345678 | VAT: GB123456789</p>
</div>
```

## Step 7: Configure Invoice Branding

Navigate to: Setup > Payments > Invoice Settings > Invoice Design

```
Invoice Header: [upload or customize]
Invoice Footer: [add company details]
Show Company Logo: Yes
Show Company Address: Yes
Show VAT Number: Yes
```

## Step 8: Set Social Media Links

Navigate to: Setup > General Settings > Social Links

```
Facebook: https://facebook.com/yourcompany
Twitter: https://twitter.com/yourcompany
LinkedIn: https://linkedin.com/company/yourcompany
YouTube: https://youtube.com/yourcompany
Instagram: https://instagram.com/yourcompany
```

## Step 9: Configure Legal Pages

Navigate to: Setup > General Settings > Legal

Link to:
```
Terms of Service: /terms.php
Privacy Policy: /privacy.php
Refund Policy: /refund.php
```

## Step 10: Update System Notifications

Navigate to: Configuration > System Settings > Notifications

Customize all system notifications with brand voice.

## Branding Checklist

- [ ] Company details configured
- [ ] Logos uploaded
- [ ] Address set
- [ ] Business hours configured
- [ ] Tax ID set
- [ ] Email templates branded
- [ ] Invoice branding configured
- [ ] Social links added
- [ ] Legal pages linked
