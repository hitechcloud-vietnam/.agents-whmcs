# WHMCS CSS Customization Workflow

## Purpose
Guide developers through customizing CSS styles in WHMCS to achieve desired visual appearance.

## Prerequisites
- WHMCS installation
- Basic CSS knowledge
- Browser developer tools
- Understanding of CSS specificity

## Steps

### Phase 1: Understanding WHMCS CSS Architecture

1. Locate CSS files
   ```
   /whmcs/templates/
   └── six/                    # Default theme
       ├── css/
       │   ├── bootstrap.css   # Bootstrap framework (v4.3)
       │   ├── fontawesome.css # Font Awesome icons
       │   ├── pleeskamp.css   # Custom WHMCS styles
       │   └── custom.css      # User custom styles
       └── scss/
           └── custom.scss     # Source SCSS files
   ```

2. CSS loading order (priority)
   - Bootstrap framework (loaded first)
   - WHMCS default styles
   - Theme custom CSS
   - Inline styles

3. Identify framework
   - WHMCS 8.x uses Bootstrap 4.3
   - WHMCS 7.x uses Bootstrap 3
   - Custom themes may use different frameworks

### Phase 2: CSS Customization Methods

#### Method 1: Custom CSS File
1. Create custom.css in your theme
   ```css
   /* templates/yourtheme/css/custom.css */
   
   /* Override primary colors */
   :root {
       --primary-color: #007bff;
       --secondary-color: #6c757d;
   }
   
   /* Custom button styles */
   .btn-primary {
       background-color: var(--primary-color);
       border-color: var(--primary-color);
       transition: all 0.3s ease;
   }
   
   .btn-primary:hover {
       background-color: #0056b3;
       transform: translateY(-2px);
   }
   ```

2. Enqueue custom CSS
   ```php
   // hooks/clientarea_page_head.php
   <?php
   add_hook('ClientAreaPageHead', 1, function($vars) {
       $version = '1.0.1';
       echo '<link rel="stylesheet" href="' . $vars['WEB_ROOT'] 
           . '/templates/yourtheme/css/custom.css?v=' . $version . '">';
   });
   ```

#### Method 2: SCSS/SASS Compilation
1. Set up SCSS structure
   ```
   templates/yourtheme/
   ├── scss/
   │   ├── _variables.scss
   │   ├── _mixins.scss
   │   ├── _components.scss
   │   ├── _pages.scss
   │   └── main.scss
   └── css/
       └── compiled.css
   ```

2. Create SCSS files
   ```scss
   // _variables.scss
   $primary-color: #007bff;
   $secondary-color: #6c757d;
   $font-family-base: 'Open Sans', sans-serif;
   $border-radius: 0.25rem;
   
   // Override Bootstrap variables
   $theme-colors: (
       "primary": $primary-color,
       "secondary": $secondary-color
   );
   ```

3. Compile SCSS
   ```bash
   # Install Sass
   npm install -g sass
   
   # Compile
   sass --watch scss:css --style=compressed
   ```

#### Method 3: Live CSS Customization
1. Use browser developer tools
   - Inspect elements (F12)
   - Identify CSS selectors
   - Modify styles live
   - Copy effective styles

2. Create override file
   ```css
   /* Override generated from DevTools */
   .card {
       border: 1px solid #e0e0e0;
       box-shadow: 0 2px 8px rgba(0,0,0,0.1);
   }
   ```

### Phase 3: Common CSS Customizations

#### Button Styling
```css
/* Custom button styles */
.btn {
    border-radius: 4px;
    padding: 10px 20px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    transition: all 0.3s ease;
}

.btn-primary {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    border: none;
}

.btn-primary:hover {
    box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
    transform: translateY(-2px);
}
```

#### Card Styling
```css
/* Product/service cards */
.product-card, .service-card {
    background: #fff;
    border-radius: 12px;
    box-shadow: 0 2px 20px rgba(0,0,0,0.08);
    transition: all 0.3s ease;
    overflow: hidden;
}

.product-card:hover, .service-card:hover {
    transform: translateY(-5px);
    box-shadow: 0 8px 30px rgba(0,0,0,0.12);
}
```

#### Form Styling
```css
/* Custom form elements */
.form-control {
    border: 2px solid #e0e0e0;
    border-radius: 8px;
    padding: 12px 16px;
    transition: border-color 0.3s ease;
}

.form-control:focus {
    border-color: #667eea;
    box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}
```

#### Table Styling
```css
/* Data tables */
.table {
    border-collapse: separate;
    border-spacing: 0;
}

.table thead th {
    background: #f8f9fa;
    border-bottom: 2px solid #dee2e6;
    font-weight: 600;
    text-transform: uppercase;
    font-size: 0.85rem;
}

.table tbody tr:hover {
    background-color: #f8f9ff;
}
```

#### Navigation Styling
```css
/* Navigation menu */
.navbar {
    padding: 1rem 0;
    box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}

.nav-link {
    font-weight: 500;
    padding: 0.75rem 1rem !important;
    transition: color 0.3s ease;
}

.nav-link:hover {
    color: #667eea !important;
}
```

### Phase 4: Theme-Specific CSS

#### Six Theme CSS Override
```css
/* Override six theme specific elements */
#header {
    background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
}

.header-nav .nav-item a {
    color: #fff;
}

#main-body {
    padding-top: 80px;
}
```

#### Order Form CSS Override
```css
/* Cart/order form customization */
.order-form-container {
    max-width: 900px;
    margin: 0 auto;
}

.cart-item-summary {
    background: #f8f9fa;
    border-radius: 8px;
    padding: 20px;
}
```

### Phase 5: CSS Organization

#### BEM Methodology
```css
/* Block-Element-Modifier structure */
.product-card {
    /* Block */
}

.product-card__image {
    /* Element */
}

.product-card__title {
    /* Element */
}

.product-card--featured {
    /* Modifier */
}
```

#### Utility Classes
```css
/* Utility classes for rapid development */
.text-primary { color: var(--primary-color) !important; }
.bg-primary { background-color: var(--primary-color) !important; }
.mt-20 { margin-top: 20px !important; }
.mb-20 { margin-bottom: 20px !important; }
.py-20 { padding-top: 20px !important; padding-bottom: 20px !important; }
```

### Phase 6: Responsive CSS

#### Media Queries
```css
/* Mobile styles */
@media (max-width: 767px) {
    .navbar-collapse {
        background: #fff;
        padding: 1rem;
        border-radius: 8px;
        box-shadow: 0 4px 20px rgba(0,0,0,0.1);
    }
    
    .product-card {
        margin-bottom: 1rem;
    }
}

/* Tablet styles */
@media (min-width: 768px) and (max-width: 991px) {
    .container {
        max-width: 720px;
    }
}
```

### Phase 7: CSS Performance

1. Minification
   ```bash
   # Using cssnano
   npm install cssnano --save-dev
   npx postcss input.css -o output.min.css
   ```

2. Critical CSS
   ```html
   <!-- Inline critical CSS in head -->
   <style>
   /* Critical above-the-fold styles */
   </style>
   ```

3. Lazy loading
   ```css
   /* Non-critical styles loaded asynchronously */
   .footer-styles { display: none; }
   .loaded .footer-styles { display: block; }
   ```

### Phase 8: Testing CSS Changes

1. Browser testing
   - Chrome, Firefox, Safari, Edge
   - DevTools element inspector
   - Responsive design mode

2. Clear cache
   - WHMCS template cache
   - Browser cache (Ctrl+Shift+R)
   - CDN cache if used

3. Verify accessibility
   - Color contrast ratios
   - Focus states visible
   - Readable text sizes

## Troubleshooting
- Specificity conflicts: Use more specific selectors or !important sparingly
- Cache issues: Clear WHMCS cache after CSS changes
- Framework conflicts: Check Bootstrap version compatibility

## Related Workflows
- whmcs-theme-customization
- whmcs-color-scheme
- whmcs-responsive-tuning
- whmcs-button-styling
- whmcs-form-styling