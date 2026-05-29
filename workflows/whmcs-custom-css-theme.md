# WHMCS Custom CSS Theme Workflow

## Purpose
Customize WHMCS appearance using CSS overrides without modifying core files.

## Prerequisites
- WHMCS installed and accessible
- FTP/cPanel file access or file manager
- Basic CSS knowledge
- Child theme or custom template setup (recommended)

## Step-by-Step Process

### Step 1: Access Template Directory
```
1. Connect to your WHMCS server via FTP or cPanel File Manager
2. Navigate to /whmcs/templates/
3. Identify your active template folder
```

### Step 2: Create Custom CSS File
```
1. Create a new file named "custom.css" in your template directory
2. Path should be: /whmcs/templates/your_template/custom.css
3. Leave file empty initially
```

### Step 3: Hook CSS Into Template
```
1. Edit your template's template.php or header.tpl file
2. Add this line in the <head> section before </head>:
   <link rel="stylesheet" href="{$BASE_PATH_4}templates/{$template}/custom.css">
```

### Step 4: Use WHMCS Hook System (Alternative)
```
1. Create file: /whmcs/includes/hooks/custom_css.php
2. Add the following code:
```

```php
<?php
use WHMCS\View\Asset;

add_hook('ClientAreaHeadOutput', 1, function($vars) {
    return '<link rel="stylesheet" href="' . Asset::url('/templates/' . $vars['template'] . '/custom.css') . '">';
});
```

### Step 5: Write Custom CSS Rules

**Common Customizations:**

```css
/* Brand Colors */
:root {
    --primary-color: #007bff;
    --secondary-color: #6c757d;
    --accent-color: #28a745;
}

/* Header Styling */
#header {
    background-color: var(--primary-color);
    box-shadow: 0 2px 10px rgba(0,0,0,0.1);
}

/* Button Customization */
.btn-primary {
    background-color: var(--primary-color);
    border-radius: 8px;
    transition: all 0.3s ease;
}

.btn-primary:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0,123,255,0.4);
}

/* Card Styling */
.card {
    border: none;
    border-radius: 12px;
    box-shadow: 0 4px 6px rgba(0,0,0,0.05);
}

/* Navigation */
.navbar-default {
    background: linear-gradient(135deg, var(--primary-color), var(--secondary-color));
}

/* Typography */
body {
    font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
}
```

### Step 6: Test and Refine
```
1. Clear WHMCS template cache (Setup > System Settings > Cache)
2. View changes in incognito/private browser
3. Use browser DevTools to inspect elements
4. Add !important flag sparingly if needed
```

### Step 7: Performance Optimization
```
1. Minify CSS for production
2. Consider using CSS variables for easy updates
3. Group related styles together
4. Remove unused rules periodically
```

## Best Practices
- Always use a child theme or custom template folder
- Never modify WHMCS core files directly
- Use CSS variables for maintainability
- Test across multiple browsers and devices
- Document your CSS changes
- Use specific selectors to avoid conflicts
- Consider mobile-first approach

## File Structure
```
/whmcs/templates/your_template/
├── custom.css          <- Your custom styles
├── header.tpl          <- Modified to include CSS
└── [other template files]
```

## Troubleshooting
- **Styles not loading**: Clear cache and verify file path
- **Conflicts**: Use more specific selectors or !important
- **Cache issues**: Disable browser cache during development
