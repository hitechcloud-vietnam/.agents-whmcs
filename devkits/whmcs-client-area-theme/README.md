# WHMCS Client Area Theme DevKit

A comprehensive custom client area theme for WHMCS that enables full customization of colors, typography, layouts, and appearance.

## Features

- Full color customization with CSS variables
- Typography customization (fonts, sizes)
- Layout customization (widths, spacing)
- Custom CSS/JS support
- Asset versioning for caching
- Template compilation with caching
- Admin configuration interface
- Live preview
- Responsive design

## Installation

1. Copy theme files to:
   ```
   templates/{theme_name}/
   ```

2. Activate the theme in WHMCS Admin:
   - Go to Configuration > System Settings > Client Area Templates
   - Select your theme and click Activate

3. Configure via the admin interface

## Configuration

### Colors

| Setting | Description |
|---------|-------------|
| Primary Color | Main brand color (buttons, links, headers) |
| Secondary Color | Secondary elements |
| Accent Color | Highlights and call-to-actions |

### Typography

| Setting | Description |
|---------|-------------|
| Font Family | Google Font or system fonts |
| Base Font Size | 12px, 14px, or 16px |

### Layout

| Setting | Description |
|---------|-------------|
| Logo | Upload custom logo |
| Favicon | Upload custom favicon |
| Container Width | Max content width |

### Custom CSS

Add custom CSS rules for further customization.

## Usage

### Theme Configuration

```php
use ClientTheme\ThemeConfig;

// Initialize theme
ThemeConfig::init('theme_name');

// Get config value
$primaryColor = ThemeConfig::get('primary_color');

// Set config value
ThemeConfig::set('primary_color', '#ff5733');

// Get CSS variables
$css = ThemeConfig::getCssVariables();
```

### Template Compilation

```php
use ClientTheme\TemplateCompiler;

$compiler = new TemplateCompiler();
$compiler->setMany([
    'title' => 'Page Title',
    'content' => 'Page content here',
    'items' => $items,
]);

$html = $compiler->render('page.tpl');
```

### Asset Management

```php
use ClientTheme\AssetManager;

// Enqueue CSS
echo AssetManager::enqueueCss('theme_name', 'custom.css');

// Enqueue JS
echo AssetManager::enqueueJs('theme_name', 'custom.js');

// Get file URLs with versioning
$assets = new AssetManager('theme_name');
$cssUrl = $assets->css('custom.css');
$jsUrl = $assets->js('custom.js');
```

## CSS Variables

The theme uses CSS custom properties (variables) for easy customization:

```css
:root {
    /* Colors */
    --primary-color: #007bff;
    --secondary-color: #6c757d;
    --accent-color: #28a745;
    --success-color: #28a745;
    --warning-color: #ffc107;
    --danger-color: #dc3545;
    --info-color: #17a2b8;
    
    /* Typography */
    --font-family: 'Poppins', sans-serif;
    --font-size-base: 14px;
    --font-weight-normal: 400;
    --font-weight-bold: 600;
    
    /* Spacing */
    --spacing-xs: 0.25rem;
    --spacing-sm: 0.5rem;
    --spacing-md: 1rem;
    --spacing-lg: 1.5rem;
    --spacing-xl: 2rem;
    
    /* Border Radius */
    --border-radius: 0.25rem;
    --border-radius-lg: 0.5rem;
    
    /* Shadows */
    --box-shadow: 0 0.125rem 0.25rem rgba(0,0,0,0.075);
    --box-shadow-lg: 0 0.5rem 1rem rgba(0,0,0,0.15);
}
```

## Hooks

The theme uses WHMCS hooks for customization:

```php
// Add custom head content
add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="custom.css">';
});

// Add custom footer content
add_hook('ClientAreaFooterOutput', 1, function($vars) {
    return '<script src="custom.js"></script>';
});
```

## Theme Manifest

Create a `config.json` in your theme directory:

```json
{
    "name": "Custom Theme",
    "version": "1.0.0",
    "author": "Your Name",
    "description": "Custom client area theme",
    "min_whmcs_version": "7.0",
    "assets": {
        "css": ["custom.css"],
        "js": ["custom.js"]
    }
}
```

## File Structure

```
templates/{theme_name}/
├── theme.php                 # Theme configuration
├── config.json               # Theme manifest
├── lib/
│   ├── ThemeConfig.php       # Configuration handler
│   ├── TemplateCompiler.php   # Template compilation
│   └── AssetManager.php      # Asset management
├── templates/
│   ├── client/
│   │   └── layout.tpl        # Main layout
│   └── overrides/
│       ├── header.tpl
│       └── footer.tpl
├── assets/
│   ├── css/
│   │   └── custom.css
│   ├── js/
│   │   └── custom.js
│   └── images/
│       ├── logo.svg
│       └── favicon.ico
└── preview.png              # Theme preview image
```

## Responsive Breakpoints

| Breakpoint | Width | Devices |
|------------|-------|---------|
| xs | < 576px | Mobile |
| sm | 576px - 768px | Tablet |
| md | 768px - 992px | Small Desktop |
| lg | 992px - 1200px | Desktop |
| xl | > 1200px | Large Desktop |

## Browser Support

- Chrome (latest)
- Firefox (latest)
- Safari (latest)
- Edge (latest)
- IE 11 (limited support)

## Requirements

- WHMCS 7.0+
- PHP 7.4+

## Support

For issues and feature requests, please contact the developer.