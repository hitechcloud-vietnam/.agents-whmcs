# WHMCS Icon Set Integration Workflow

## Purpose
Integrate and use icon libraries in WHMCS for enhanced visual elements.

## Prerequisites
- WHMCS installation
- FTP/cPanel file access
- Basic CSS/HTML knowledge

## Step-by-Step Process

### Step 1: Choose Icon Library

**Popular Options:**
- Font Awesome 6 (comprehensive)
- Material Icons (clean, Google-maintained)
- Phosphor Icons (modern, versatile)
- Heroicons (Tailwind CSS companion)
- Feather Icons (minimalist)

### Step 2: Font Awesome Integration

**Hook File:**
```php
<?php
/**
 * Add Font Awesome 6 to WHMCS
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '<link rel="stylesheet" 
                  href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" 
                  integrity="sha512-iecdLmaskl7CVkqkXNQ/ZH/XLlvWZOJyj7Yy7tcenmpD1ypASozpmT/E0iPtmFIB46ZmdtAc9eNBvH0H/ZpiBw==" 
                  crossorigin="anonymous" 
                  referrerpolicy="no-referrer" />';
});
```

**Alternative: Self-Hosted Font Awesome:**
```bash
# Download Font Awesome
1. Download from fontawesome.com
2. Extract to /whmcs/templates/your_template/icons/
```

```php
<?php
/**
 * Self-hosted Font Awesome
 */
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    $basePath = '/templates/' . $vars['template'] . '/icons';
    return '<link rel="stylesheet" href="' . $basePath . '/css/all.min.css">';
});
```

### Step 3: SVG Sprite Setup

**Create SVG Sprite:**
```xml
<!-- /whmcs/templates/your_template/icons/sprite.svg -->
<svg xmlns="http://www.w3.org/2000/svg" style="display: none;">
    <symbol id="icon-server" viewBox="0 0 24 24">
        <path d="M12 2C6.48 2 2 4.69 2 8v8c0 3.31 4.48 6 10 6s10-2.69 10-6V8c0-3.31-4.48-6-10-6z"/>
    </symbol>
    <symbol id="icon-cloud" viewBox="0 0 24 24">
        <path d="M19.35 10.04C18.67 6.59 15.64 4 12 4 9.11 4 6.6 5.64 5.35 8.04 2.34 8.36 0 10.91 0 14c0 3.31 2.69 6 6 6h13c2.76 0 5-2.24 5-5 0-2.64-2.05-4.78-4.65-4.96z"/>
    </symbol>
    <symbol id="icon-shield" viewBox="0 0 24 24">
        <path d="M12 1L3 5v6c0 5.55 3.84 10.74 9 12 5.16-1.26 9-6.45 9-12V5l-9-4z"/>
    </symbol>
</svg>
```

**Use SVG Sprite:**
```smarty
<svg class="icon"><use xlink:href="#icon-server"></use></svg>
```

### Step 4: Icon Hook System

**Icon Service Class:**
```php
<?php
/**
 * WHMCS Icon Service
 */
namespace WHMCS\Services;

class Icons {
    private static $library = 'fontawesome';
    
    public static function setLibrary($library) {
        self::$library = $library;
    }
    
    public static function render($name, $options = []) {
        $size = $options['size'] ?? '1x';
        $class = $options['class'] ?? '';
        $style = $options['style'] ?? '';
        
        switch (self::$library) {
            case 'fontawesome':
                return self::fontAwesome($name, $size, $class, $style);
            case 'material':
                return self::materialIcon($name, $size, $class, $style);
            case 'svg':
                return self::svgIcon($name, $size, $class, $style);
            default:
                return self::fontAwesome($name, $size, $class, $style);
        }
    }
    
    private static function fontAwesome($name, $size, $class, $style) {
        $prefix = strpos($name, 'brands-') === 0 ? 'fab' : 'fas';
        return '<i class="' . $prefix . ' fa-' . $name . ' fa-' . $size . ' ' . $class . '" style="' . $style . '"></i>';
    }
    
    private static function materialIcon($name, $size, $class, $style) {
        return '<span class="material-icons ' . $class . '" style="' . $style . '">' . $name . '</span>';
    }
    
    private static function svgIcon($name, $size, $class, $style) {
        return '<svg class="icon ' . $class . '" style="' . $style . '"><use xlink:href="#icon-' . $name . '"></use></svg>';
    }
}
```

### Step 5: Category Icon Mapping

**Configuration:**
```php
<?php
/**
 * Map product categories to icons
 */
$categoryIcons = [
    'hosting' => 'fa-server',
    'reseller' => 'fa-hand-holding-usd',
    'vps' => 'fa-cloud',
    'dedicated' => 'fa-database',
    'ssl' => 'fa-shield-alt',
    'domains' => 'fa-globe',
    'email' => 'fa-envelope',
    'software' => 'fa-code',
    'support' => 'fa-life-ring',
    'other' => 'fa-box'
];

add_hook('ClientAreaHeadOutput', 1, function($vars) use ($categoryIcons) {
    $css = '<style>';
    foreach ($categoryIcons as $category => $icon) {
        $css .= '.category-icon-' . $category . '::before { content: "\\' . substr($icon, 3) . '"; }';
    }
    $css .= '</style>';
    return $css;
});
```

**Template Usage:**
```smarty
{foreach from=$productCategories item=category}
    <div class="category-card">
        <i class="fas {$categoryIcons[$category.name]|default:'fa-folder'}"></i>
        <span>{$category.name}</span>
    </div>
{/foreach}
```

### Step 6: Status Icons

**Status Icon Configuration:**
```php
<?php
/**
 * Status-based icons
 */
$statusIcons = [
    'Active' => ['icon' => 'fa-check-circle', 'color' => '#28a745'],
    'Pending' => ['icon' => 'fa-clock', 'color' => '#ffc107'],
    'Suspended' => ['icon' => 'fa-pause-circle', 'color' => '#dc3545'],
    'Terminated' => ['icon' => 'fa-times-circle', 'color' => '#6c757d'],
    'Cancelled' => ['icon' => 'fa-ban', 'color' => '#6c757d'],
    'Expired' => ['icon' => 'fa-hourglass-end', 'color' => '#6c757d'],
];
```

**Smarty Function:**
```smarty
{function name=statusIcon}
    {assign var="statusConfig" value=$statusIcons[$status]|default:['icon' => 'fa-question', 'color' => '#6c757d']}
    <i class="fas {$statusConfig.icon}" style="color: {$statusConfig.color}"></i>
{/function}

{statusIcon status=$service.status}
```

### Step 7: Navigation Icons

**Icon Navigation Hook:**
```php
<?php
/**
 * Add icons to navigation
 */
add_hook('ClientAreaNavbars', 1, function($vars) {
    $navIcons = [
        'home' => 'fa-home',
        'account' => 'fa-user-circle',
        'services' => 'fa-server',
        'billing' => 'fa-credit-card',
        'support' => 'fa-life-ring',
        'logout' => 'fa-sign-out-alt'
    ];
    
    return ['nav_icons' => $navIcons];
});
```

**Template Integration:**
```smarty
{foreach from=$primarynav item=navitem}
    <li>
        <a href="{$navitem.uri}">
            {if $nav_icons[$navitem.name]}
                <i class="fas {$nav_icons[$navitem.name]}"></i>
            {/if}
            {$navitem.label}
        </a>
    </li>
{/foreach}
```

### Step 8: Animated Icons

**Animated Icon CSS:**
```css
/* Spinning icons */
.fa-spin {
    animation: fa-spin 2s infinite linear;
}

@keyframes fa-spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

/* Pulse animation */
.fa-pulse {
    animation: fa-pulse 1s infinite;
}

@keyframes fa-pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.5; }
}

/* Bounce animation */
.fa-bounce {
    animation: fa-bounce 1s infinite;
}

@keyframes fa-bounce {
    0%, 100% { transform: translateY(0); }
    50% { transform: translateY(-25%); }
}

/* Pulse on hover */
.icon-hover:hover {
    animation: fa-pulse 1s infinite;
}
```

### Step 9: Icon Size Classes

```css
/* Icon Size Utility Classes */
.icon-xs { font-size: 0.625rem; }   /* 10px */
.icon-sm { font-size: 0.875rem; }   /* 14px */
.icon-md { font-size: 1rem; }       /* 16px */
.icon-lg { font-size: 1.5rem; }     /* 24px */
.icon-xl { font-size: 2rem; }       /* 32px */
.icon-2xl { font-size: 3rem; }     /* 48px */

/* Fixed width icons */
.icon-fw {
    width: 1.25em;
    text-align: center;
}
```

## Best Practices
- Use consistent icon library throughout
- Prefer SVG for performance (inline when possible)
- Use appropriate icon sizes for context
- Maintain color consistency with brand
- Provide fallback for missing icons
- Consider accessibility (aria-label)
- Lazy load icons below the fold
- Optimize SVG files (remove unnecessary data)
- Use icon fonts for ease of styling
- Document icon usage in your project
