# WHMCS Client Area Theming Workflow

## Purpose
Customize the WHMCS client area appearance

## Prerequisites
- WHMCS installed
- Admin access
- Basic CSS/HTML knowledge

## Step 1: Access Theme Settings

Navigate to: Setup > Client Area Design > Theme Settings

## Step 2: Choose Base Theme

Select from available themes:
- Default Six
- Twenty-One
- Custom Theme

## Step 3: Configure Colors

Navigate to: Setup > Client Area Design > Theme Settings > Colours

```
Primary Color: #0073aa
Secondary Color: #23282d
Accent Color: #00a0d2
Success Color: #46b450
Warning Color: #ffb900
Danger Color: #dc3232
```

## Step 4: Configure Logo

Navigate to: Setup > Client Area Design > Theme Settings > Logo

Upload:
```
Header Logo: [PNG/SVG, 200x60px]
Footer Logo: [smaller version]
Favicon: [32x32 ICO]
```

## Step 5: Set Up Custom CSS

Navigate to: Setup > Client Area Design > Customize Theme

### Add Custom CSS
```css
/* Custom Client Area Styles */
body {
    font-family: 'Open Sans', sans-serif;
}

.navbar {
    background-color: #0073aa;
}

.btn-primary {
    background-color: #0073aa;
    border-color: #006799;
}

.card {
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}
```

## Step 6: Configure Typography

Navigate to: Setup > Client Area Design > Theme Settings > Typography

```
Heading Font: Open Sans
Body Font: Open Sans
Font Size Scale: 1.125rem
```

## Step 7: Set Up Footer Content

Navigate to: Setup > Client Area Design > Theme Settings > Footer

Configure:
```
Company Name: Your Company
Address: 123 Main St, City, Country
Phone: +1-555-555-5555
Email: info@yourdomain.com
VAT Number: GB123456789
```

## Step 8: Configure Social Links

Navigate to: Setup > Client Area Design > Theme Settings > Social

```
Facebook: https://facebook.com/yourcompany
Twitter: https://twitter.com/yourcompany
LinkedIn: https://linkedin.com/company/yourcompany
```

## Step 9: Create Custom Template

### Copy Default Template
```bash
cd /var/www/whmcs/templates
cp -r six custom_theme
```

### Edit Template Files
```bash
nano /var/www/whmcs/templates/custom_theme/header.tpl
nano /var/www/whmcs/templates/custom_theme/footer.tpl
```

### Register Template
Navigate to: Setup > Client Area Design > Order Form Templates

Activate custom theme.

## Step 10: Add Custom CSS Variables

Navigate to: Setup > Client Area Design > Customize Theme

```css
:root {
    --primary-color: #0073aa;
    --secondary-color: #23282d;
    --accent-color: #00a0d2;
    --font-family: 'Inter', sans-serif;
    --border-radius: 8px;
    --box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}
```

## Client Area Theme Checklist

- [ ] Base theme selected
- [ ] Colors configured
- [ ] Logo uploaded
- [ ] Custom CSS added
- [ ] Typography set
- [ ] Footer configured
- [ ] Social links added
- [ ] Custom template created (optional)
