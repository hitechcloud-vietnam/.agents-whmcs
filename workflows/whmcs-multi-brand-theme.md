# WHMCS Multi-Brand Theme Workflow

## Purpose
Guide developers through implementing multi-brand theming in WHMCS.

## Prerequisites
- WHMCS installation
- Theme development experience
- Understanding of brand management
- PHP/hook system knowledge

## Steps

### Phase 1: Multi-Brand Architecture

1. Brand theme structure
   ```
   /whmcs/templates/
   ├── brands/
   │   ├── brand-a/
   │   │   ├── theme.json
   │   ├── brand-b/
   │   │   ├── theme.json
   │   └── brand-c/
   │       ├── theme.json
   ```

2. Brand configuration
   ```json
   {
       "name": "Brand A Theme",
       "slug": "brand-a",
       "brand": {
           "name": "Brand A Company",
           "logo": "/img/brand-a-logo.png",
           "colors": {
               "primary": "#667eea",
               "secondary": "#764ba2"
           }
       },
       "domains": ["brand-a.com", "brand-a.net"],
       "parent": "six"
   }
   ```

### Phase 2: Brand Switching System

1. Brand detection hook
   ```php
   // hooks/brand_detection.php
   <?php
   add_hook('ClientAreaHeadOutput', 1, function($vars) {
       $currentDomain = $_SERVER['HTTP_HOST'];
       
       $brandConfig = [
           'brand-a.com' => [
               'theme' => 'brand-a',
               'colors' => ['primary' => '#667eea', 'secondary' => '#764ba2'],
               'logo' => '/img/brand-a-logo.png'
           ],
           'brand-b.com' => [
               'theme' => 'brand-b',
               'colors' => ['primary' => '#00b894', 'secondary' => '#00cec9'],
               'logo' => '/img/brand-b-logo.png'
           ],
           'brand-c.com' => [
               'theme' => 'brand-c',
               'colors' => ['primary' => '#e17055', 'secondary' => '#fab1a0'],
               'logo' => '/img/brand-c-logo.png'
           ]
       ];
       
       if (isset($brandConfig[$currentDomain])) {
           $brand = $brandConfig[$currentDomain];
           
           return '<style>
               :root {
                   --brand-primary: ' . $brand['colors']['primary'] . ';
                   --brand-secondary: ' . $brand['colors']['secondary'] . ';
               }
           </style>';
       }
   });
   ```

### Phase 3: Brand Theme Template

1. Base brand template
   ```smarty
   {* templates/brands/brand-base/theme.json *}
   {
       "extends": "six",
       "name": "{$brand_name}",
       "brand": {
           "id": "{$brand_id}",
           "name": "{$brand_display_name}",
           "logo_url": "{$brand_logo_url}",
           "favicon_url": "{$brand_favicon_url}"
       },
       "colors": {
           "primary": "{$brand_primary_color}",
           "secondary": "{$brand_secondary_color}",
           "accent": "{$brand_accent_color}"
       },
       "fonts": {
           "heading": "{$brand_heading_font}",
           "body": "{$brand_body_font}"
       }
   }
   ```

2. Brand header template
   ```smarty
   {* Brand-specific header *}
   <header class="site-header brand-header" 
           style="background: linear-gradient(135deg, {$brand.primary}, {$brand.secondary});">
       <nav class="navbar">
           <a class="navbar-brand" href="{$WEB_ROOT}/">
               <img src="{$brand.logo_url}" alt="{$brand.name}" class="brand-logo">
           </a>
           
           <div class="brand-nav">
               {include file="$template/navigation.tpl"}
           </div>
       </nav>
   </header>
   ```

### Phase 4: Dynamic Brand CSS

1. Brand CSS variables
   ```css
   .brand-theme {
       --brand-primary: {$brand.primary|default:'#007bff'};
       --brand-secondary: {$brand.secondary|default:'#6c757d'};
       --brand-accent: {$brand.accent|default:'#28a745'};
       
       --brand-heading-font: {$brand.fonts.heading|default:'Poppins'};
       --brand-body-font: {$brand.fonts.body|default:'Inter'};
   }
   
   /* Brand colors applied */
   .btn-brand {
       background: var(--brand-primary);
       color: #fff;
   }
   
   .btn-brand:hover {
       background: var(--brand-secondary);
   }
   
   .brand-accent-text {
       color: var(--brand-accent);
   }
   ```

2. Brand stylesheet
   ```css
   /* Brand-specific styles */
   .brand-header {
       background: linear-gradient(135deg, var(--brand-primary), var(--brand-secondary));
   }
   
   .brand-logo {
       max-height: 50px;
       filter: brightness(0) invert(1);
   }
   
   .brand-nav .nav-link:hover {
       color: var(--brand-primary);
   }
   
   .brand-hero {
       background: linear-gradient(135deg, var(--brand-primary) 0%, var(--brand-secondary) 100%);
   }
   ```

### Phase 5: Email Brand Templates

1. Brand email headers
   ```php
   add_hook('EmailPreSend', 1, function($vars) {
       $clientBrand = getClientBrand($vars['clientid']);
       
       return [
           'brand_logo' => $clientBrand['logo_url'],
           'brand_colors' => $clientBrand['colors'],
           'brand_company_name' => $clientBrand['company_name']
       ];
   });
   ```

2. Brand email template
   ```smarty
   {* Brand email header *}
   <table role="presentation" width="100%" cellpadding="0" cellspacing="0" 
          style="background: linear-gradient(135deg, {$brand_colors.primary}, {$brand_colors.secondary});">
       <tr>
           <td align="center" style="padding: 30px 20px;">
               <img src="{$brand_logo}" alt="{$brand_company_name}" 
                    style="max-width: 180px; height: auto;">
           </td>
       </tr>
   </table>
   ```

### Phase 6: Admin Brand Management

1. Brand settings page
   ```php
   // Admin brand configuration
   add_hook('AdminAreaConfigSidebar', 1, function($vars) {
       return [
           [
               'name' => 'Branding',
               'label' => 'Multi-Brand Settings',
               'uri' => '/admin/config-brand.php',
               'order' => 100
           ]
       ];
   });
   ```

2. Brand CRUD operations
   ```php
   function saveBrand($brandData) {
       // Validate brand data
       // Save to database
       // Generate theme files
       // Clear cache
   }
   
   function switchBrand($brandId) {
       // Update active brand
       // Refresh configuration
       // Clear template cache
   }
   ```

### Phase 7: Per-Client Branding

1. Client brand assignment
   ```php
   function assignClientBrand($clientId, $brandId) {
       // Update client record with brand
       // Set brand cookie
       // Apply brand-specific theming
   }
   ```

2. Client brand detection
   ```php
   add_hook('ClientAreaHeadOutput', 1, function($vars) {
       if (isset($vars['userid'])) {
           $clientBrand = Capsule::table('clients')
               ->where('id', $vars['userid'])
               ->first();
           
           if ($clientBrand && $clientBrand->brand_id) {
               $brand = getBrandById($clientBrand->brand_id);
               return applyBrandStyles($brand);
           }
       }
   });
   ```

### Phase 8: Domain-Based Routing

1. Domain routing hook
   ```php
   // hooks/domain_routing.php
   add_hook('SystemPreModuleCreate', 1, function($vars) {
       $domain = $_SERVER['HTTP_HOST'];
       $brand = findBrandByDomain($domain);
       
       if ($brand) {
           $_SESSION['active_brand'] = $brand;
       }
   });
   ```

2. URL routing
   ```php
   function routeToBrand() {
       $brandSlug = $_GET['brand'] ?? null;
       
       if ($brandSlug) {
           $brand = getBrandBySlug($brandSlug);
           loadBrandTheme($brand);
       }
   }
   ```

## Brand Configuration Example

```php
// Brand configuration
$brands = [
    'techcorp' => [
        'name' => 'TechCorp',
        'theme' => 'techcorp-theme',
        'company_name' => 'TechCorp Hosting',
        'logo_url' => '/img/techcorp-logo.svg',
        'colors' => [
            'primary' => '#2563eb',
            'secondary' => '#1d4ed8',
            'accent' => '#06b6d4'
        ],
        'domains' => ['techcorp.example.com', 'techcorp.io'],
        'email_from_name' => 'TechCorp Support'
    ],
    'cloudnine' => [
        'name' => 'CloudNine',
        'theme' => 'cloudnine-theme',
        'company_name' => 'CloudNine Solutions',
        'logo_url' => '/img/cloudnine-logo.svg',
        'colors' => [
            'primary' => '#8b5cf6',
            'secondary' => '#7c3aed',
            'accent' => '#10b981'
        ],
        'domains' => ['cloudnine.example.com'],
        'email_from_name' => 'CloudNine Support'
    ]
];
```

## Related Workflows
- whmcs-theme-customization
- whmcs-color-scheme
- whmcs-logo-branding
- whmcs-email-template-design