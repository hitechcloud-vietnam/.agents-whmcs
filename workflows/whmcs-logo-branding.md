# WHMCS Logo and Branding Workflow

## Purpose
Guide developers and administrators through customizing WHMCS logo and branding elements.

## Prerequisites
- WHMCS admin access
- Logo files (SVG, PNG, or JPEG)
- Basic image editing knowledge
- FTP/File manager access

## Steps

### Phase 1: Logo Requirements and Preparation

1. Required logo formats
   ```
   Formats needed:
   - Main logo (header): PNG/SVG, transparent background
   - Footer logo: PNG with white/dark background
   - Favicon: ICO/PNG, 16x16, 32x32, 64x64 sizes
   - Apple touch icon: PNG, 180x180
   - Social media og:image: PNG, 1200x630
   ```

2. Logo dimensions
   - Header logo: Max width 250px, height proportional
   - Email logo: Max width 200px
   - Invoice logo: Max width 300px, max height 100px
   - Favicon: 32x32 or 48x48 for ICO format

3. Preparation checklist
   - Export vector files (SVG/AI)
   - Optimize file sizes (<100KB for web)
   - Prepare light and dark versions
   - Create retina versions (@2x)

### Phase 2: Uploading Logos in WHMCS Admin

1. Upload via WHMCS Admin Panel
   - Navigate to: Configuration > System Settings > General Settings
   - Click "Customizing" or "Branding" tab
   - Upload logo files in appropriate sections
   - Save settings

2. Logo locations in WHMCS
   ```
   Admin Panel Settings:
   ├── Site Logo (Header)
   ├── Email Header Logo
   ├── PDF Invoice Logo
   ├── Login Page Logo
   ├── Admin System Logo
   └── Favicon
   ```

3. Direct file upload
   ```
   System logos location:
   /whmcs/assets/img/
   ├── logo.png          # Main header logo
   ├── logo-order.png    # Order form logo
   ├── logo-dark.png     # Dark theme logo
   └── logo-light.png    # Light theme logo
   ```

### Phase 3: Theme Logo Customization

#### Six Theme Logo
1. Override logo in custom theme
   ```smarty
   {* In header.tpl *}
   <a href="{$WEB_ROOT}/" class="navbar-brand">
       {if $custom_logo_url}
           <img src="{$custom_logo_url}" alt="{$companyname}" class="custom-logo">
       {else}
           <img src="{$BASE_URL_CUSTOM_TEMPLATE}img/logo.svg" 
                alt="{$companyname}" 
                class="custom-logo"
                height="40">
       {/if}
   </a>
   ```

2. CSS for logo styling
   ```css
   /* Logo container styles */
   .navbar-brand .custom-logo {
       max-height: 50px;
       width: auto;
       max-width: 200px;
       padding: 5px 0;
   }
   
   /* Responsive logo */
   @media (max-width: 768px) {
       .navbar-brand .custom-logo {
           max-height: 35px;
           max-width: 150px;
       }
   }
   ```

#### Order Form Logo
```smarty
{* In order form header *}
<div class="order-header">
    <a href="{$WEB_ROOT}/">
        <img src="{$logo_url}" 
             alt="{$companyname}" 
             class="order-logo"
             style="max-height: 60px;">
    </a>
</div>
```

### Phase 4: Dynamic Logo Switching

#### Theme-Based Logo
```php
// hooks/theme_logo.php
<?php
add_hook('ClientAreaPageHead', 1, function($vars) {
    $currentTheme = isset($vars['template']) ? $vars['template'] : 'six';
    
    $logoMap = [
        'dark-theme' => '/img/logo-dark.png',
        'light-theme' => '/img/logo-light.png',
        'default' => '/img/logo.png'
    ];
    
    $logoPath = $logoMap[$currentTheme] ?? $logoMap['default'];
    
    echo '<style>
        .site-logo {
            background-image: url("' . $logoPath . '");
        }
    </style>';
});
```

#### Dark/Light Mode Logo
```javascript
// Dynamic logo switching based on theme
(function($) {
    'use strict';
    
    function updateLogoForTheme(isDark) {
        var logoSrc = isDark ? '/img/logo-dark.png' : '/img/logo-light.png';
        $('.navbar-brand img').attr('src', logoSrc);
    }
    
    // Check system preference
    if (window.matchMedia) {
        var darkQuery = window.matchMedia('(prefers-color-scheme: dark)');
        darkQuery.addEventListener('change', function(e) {
            updateLogoForTheme(e.matches);
        });
        updateLogoForTheme(darkQuery.matches);
    }
})(jQuery);
```

### Phase 5: Email Template Branding

1. Email header logo settings
   - WHMCS Admin > Configuration > Email Templates > Email Header Logo
   - Recommended size: 200px width, PNG format
   - Transparent or white background

2. Email signature logo
   ```html
   <!-- Email signature -->
   <table width="100%" cellpadding="0" cellspacing="0" border="0">
       <tr>
           <td align="left" style="padding: 20px;">
               <img src="{$email_logo_url}" 
                    alt="{$companyname}" 
                    style="max-width: 150px;">
           </td>
       </tr>
   </table>
   ```

3. Email footer logo
   ```html
   <!-- Company logo in email footer -->
   <div style="text-align: center; padding: 20px 0;">
       <img src="https://yoursite.com/assets/img/email-footer-logo.png" 
            alt="{$companyname}" 
            style="max-width: 120px;">
   </div>
   ```

### Phase 6: Invoice and PDF Branding

1. Invoice logo settings
   - WHMCS Admin > Configuration > Invoice Settings
   - Upload invoice logo (max 300px width, 100px height)
   - PNG with transparent or white background

2. PDF template logo positioning
   ```smarty
   {* In invoicepdf.tpl *}
   <div class="invoice-header">
       <table width="100%">
           <tr>
               <td width="60%">
                   <img src="{$logo_url}" 
                        alt="{$companyname}" 
                        style="max-height: 80px; max-width: 250px;">
               </td>
               <td width="40%" align="right">
                   <p class="company-name">{$companyname}</p>
                   <p class="company-address">{$companyaddress|nl2br}</p>
               </td>
           </tr>
       </table>
   </div>
   ```

3. CSS for PDF styling
   ```css
   /* PDF invoice styles */
   .invoice-header img {
       max-width: 250px;
       max-height: 80px;
   }
   
   .invoice-logo {
       float: right;
       margin: 10px;
   }
   ```

### Phase 7: Favicon and Icons

1. Generate favicon
   ```bash
   # Install favicon generator tool
   npm install -g favicons
   
   # Generate favicons
   favicons input-logo.png output-folder --preset=legacy
   ```

2. Upload favicon
   ```
   /whmcs/
   └── favicon.ico          # 16x16, 32x32, 48x48 in ICO
   └── apple-touch-icon.png # 180x180
   ```

3. Update HTML head
   ```html
   <!-- In header.tpl -->
   <link rel="icon" type="image/x-icon" href="{$WEB_ROOT}/favicon.ico">
   <link rel="apple-touch-icon" href="{$WEB_ROOT}/apple-touch-icon.png">
   <meta name="theme-color" content="#007bff">
   ```

4. Manifest for PWA
   ```json
   {
       "name": "Your Company",
       "short_name": "Company",
       "icons": [
           { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
           { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
       ],
       "theme_color": "#007bff",
       "background_color": "#ffffff",
       "display": "standalone"
   }
   ```

### Phase 8: Brand Guidelines Implementation

1. Create brand hook file
   ```php
   // hooks/brand_settings.php
   <?php
   add_hook('ClientAreaHeadOutput', 1, function($vars) {
       return <<<HTML
       <style>
           :root {
               --brand-primary: #007bff;
               --brand-secondary: #6c757d;
               --brand-logo-url: '/templates/custom/img/logo.png';
           }
           
           .navbar-brand {
               background-image: var(--brand-logo-url);
               background-size: contain;
               background-repeat: no-repeat;
           }
       </style>
   HTML;
   });
   ```

2. Update all template locations
   - Header templates
   - Footer templates
   - Email templates
   - Invoice templates
   - Admin area

## Verification Checklist
- [ ] Header logo displays correctly
- [ ] Email logo appears in all emails
- [ ] Invoice logo on PDFs
- [ ] Favicon shows in browser tab
- [ ] Mobile logo is responsive
- [ ] Dark theme logo loads correctly

## Troubleshooting
- Logo not appearing: Clear WHMCS cache
- Blurry logo: Use higher resolution image
- CORS issues: Ensure logo URLs are absolute
- File permissions: Check upload directory permissions

## Related Workflows
- whmcs-theme-customization
- whmcs-color-scheme
- whmcs-email-template-design
- whmcs-invoice-template-design