# WHMCS Font Customization Workflow

## Purpose
Guide developers through customizing fonts and typography in WHMCS.

## Prerequisites
- WHMCS installation
- CSS/SCSS knowledge
- Web font integration experience
- Google Fonts or custom font files

## Steps

### Phase 1: Font Selection and Preparation

1. Choose web fonts
   ```
   Popular font choices:
   - Primary: Inter, Roboto, Open Sans
   - Headings: Poppins, Montserrat, Playfair Display
   - Monospace: Fira Code, Source Code Pro
   ```

2. Font file formats
   ```
   Web font formats:
   - WOFF2: Modern browsers (preferred)
   - WOFF: Older browsers support
   - TTF: Fallback for older browsers
   - EOT: IE8 and below (rarely needed)
   ```

3. Determine font weights
   ```
   Common weights needed:
   - 300: Light
   - 400: Regular (Normal)
   - 500: Medium
   - 600: Semi Bold
   - 700: Bold
   ```

### Phase 2: Google Fonts Integration

1. Select and import fonts
   ```html
   <!-- In header.tpl or via hook -->
   <link rel="preconnect" href="https://fonts.googleapis.com">
   <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
   <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Poppins:wght@400;500;600;700&display=swap" rel="stylesheet">
   ```

2. PHP hook for font loading
   ```php
   // hooks/font_loader.php
   <?php
   add_hook('ClientAreaHeadOutput', 1, function($vars) {
       return '<link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&family=Poppins:wght@400;500;600;700&display=swap" rel="stylesheet">';
   });
   ```

### Phase 3: CSS Font Variables

1. Define font variables
   ```scss
   // _typography.scss
   :root {
       // Font families
       --font-family-base: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
       --font-family-heading: 'Poppins', var(--font-family-base);
       --font-family-mono: 'Fira Code', 'Consolas', monospace;
       
       // Font sizes
       --font-size-base: 1rem;         // 16px
       --font-size-sm: 0.875rem;        // 14px
       --font-size-xs: 0.75rem;        // 12px
       --font-size-lg: 1.125rem;       // 18px
       --font-size-xl: 1.25rem;        // 20px
       --font-size-2xl: 1.5rem;        // 24px
       --font-size-3xl: 1.875rem;     // 30px
       --font-size-4xl: 2.25rem;       // 36px
       
       // Font weights
       --font-weight-light: 300;
       --font-weight-normal: 400;
       --font-weight-medium: 500;
       --font-weight-semibold: 600;
       --font-weight-bold: 700;
       
       // Line heights
       --line-height-base: 1.5;
       --line-height-heading: 1.2;
       --line-height-tight: 1.25;
       
       // Letter spacing
       --letter-spacing-tight: -0.025em;
       --letter-spacing-normal: 0;
       --letter-spacing-wide: 0.025em;
   }
   ```

### Phase 4: Typography Base Styles

1. Body typography
   ```css
   /* Body text styles */
   body {
       font-family: var(--font-family-base);
       font-size: var(--font-size-base);
       font-weight: var(--font-weight-normal);
       line-height: var(--line-height-base);
       color: var(--color-text);
       -webkit-font-smoothing: antialiased;
       -moz-osx-font-smoothing: grayscale;
   }
   
   /* Headings */
   h1, h2, h3, h4, h5, h6 {
       font-family: var(--font-family-heading);
       font-weight: var(--font-weight-semibold);
       line-height: var(--line-height-heading);
       margin-bottom: 0.5em;
   }
   
   h1 { font-size: var(--font-size-4xl); font-weight: var(--font-weight-bold); }
   h2 { font-size: var(--font-size-3xl); }
   h3 { font-size: var(--font-size-2xl); }
   h4 { font-size: var(--font-size-xl); }
   h5 { font-size: var(--font-size-lg); }
   h6 { font-size: var(--font-size-base); }
   ```

### Phase 5: Component Typography

1. Button text
   ```css
   /* Button typography */
   .btn {
       font-family: var(--font-family-base);
       font-size: var(--font-size-sm);
       font-weight: var(--font-weight-semibold);
       letter-spacing: 0.025em;
       text-transform: uppercase;
   }
   
   .btn-lg {
       font-size: var(--font-size-base);
   }
   
   .btn-sm {
       font-size: var(--font-size-xs);
   }
   ```

2. Form labels
   ```css
   /* Form typography */
   .form-label {
       font-size: var(--font-size-sm);
       font-weight: var(--font-weight-medium);
       margin-bottom: 0.25rem;
   }
   
   .form-control {
       font-family: var(--font-family-base);
       font-size: var(--font-size-base);
   }
   
   .form-control::placeholder {
       color: var(--color-text-muted);
       font-weight: var(--font-weight-light);
   }
   ```

3. Navigation
   ```css
   /* Navigation typography */
   .navbar-nav .nav-link {
       font-family: var(--font-family-base);
       font-size: var(--font-size-sm);
       font-weight: var(--font-weight-medium);
   }
   
   .dropdown-item {
       font-size: var(--font-size-sm);
   }
   ```

### Phase 6: Table and Data Typography

1. Table text styles
   ```css
   /* Table typography */
   .table {
       font-size: var(--font-size-sm);
   }
   
   .table thead th {
       font-size: var(--font-size-xs);
       font-weight: var(--font-weight-semibold);
       text-transform: uppercase;
       letter-spacing: 0.05em;
   }
   
   .table tbody td {
       vertical-align: middle;
   }
   ```

2. Price and currency typography
   ```css
   /* Price display */
   .price {
       font-family: var(--font-family-base);
       font-weight: var(--font-weight-bold);
       font-size: var(--font-size-2xl);
   }
   
   .price-small {
       font-size: var(--font-size-lg);
   }
   
   .price-large {
       font-size: var(--font-size-4xl);
   }
   
   .currency {
       font-weight: var(--font-weight-normal);
       font-size: 0.75em;
       vertical-align: super;
   }
   ```

### Phase 7: Responsive Typography

1. Mobile typography
   ```css
   /* Mobile font sizes */
   @media (max-width: 767px) {
       body {
           font-size: 15px; /* Slightly smaller for mobile */
       }
       
       h1 { font-size: 1.75rem; }
       h2 { font-size: 1.5rem; }
       h3 { font-size: 1.25rem; }
       h4 { font-size: 1.125rem; }
   }
   
   /* Tablet typography */
   @media (min-width: 768px) and (max-width: 991px) {
       body {
           font-size: 16px;
       }
   }
   
   /* Desktop typography */
   @media (min-width: 992px) {
       body {
           font-size: 16px;
       }
   }
   ```

2. Responsive headings
   ```css
   @media (max-width: 576px) {
       .display-1 { font-size: 2.5rem; }
       .display-2 { font-size: 2rem; }
       .display-3 { font-size: 1.75rem; }
       .display-4 { font-size: 1.5rem; }
   }
   ```

### Phase 8: Font Display Optimization

1. Font loading optimization
   ```html
   <!-- Preload critical fonts -->
   <link rel="preload" href="/fonts/inter.woff2" as="font" type="font/woff2" crossorigin>
   
   <!-- Font display swap for faster text rendering -->
   <style>
       @font-face {
           font-family: 'Inter';
           font-style: normal;
           font-weight: 400;
           font-display: swap;
           src: url('/fonts/inter.woff2') format('woff2');
       }
   </style>
   ```

2. Self-hosted fonts
   ```css
   /* Self-hosted font declaration */
   @font-face {
       font-family: 'Inter';
       src: url('/fonts/Inter-Light.woff2') format('woff2'),
            url('/fonts/Inter-Light.woff') format('woff');
       font-weight: 300;
       font-style: normal;
       font-display: swap;
   }
   
   @font-face {
       font-family: 'Inter';
       src: url('/fonts/Inter-Regular.woff2') format('woff2'),
            url('/fonts/Inter-Regular.woff') format('woff');
       font-weight: 400;
       font-style: normal;
       font-display: swap;
   }
   
   @font-face {
       font-family: 'Inter';
       src: url('/fonts/Inter-Bold.woff2') format('woff2'),
            url('/fonts/Inter-Bold.woff') format('woff');
       font-weight: 700;
       font-style: normal;
       font-display: swap;
   }
   ```

3. Icon font integration
   ```css
   /* Icon font sizing */
   .fa, .fas, .far, .fal, .fab {
       font-family: 'Font Awesome 5 Free', 'Font Awesome 5 Pro';
       font-weight: 900;
       font-size: 1em;
       line-height: 1;
   }
   
   .fa-sm { font-size: 0.875em; }
   .fa-lg { font-size: 1.3333em; }
   .fa-2x { font-size: 2em; }
   .fa-3x { font-size: 3em; }
   ```

## Testing Typography

1. Font loading verification
   - Check network tab for font files
   - Verify FOUT (Flash of Unstyled Text)
   - Test fallback rendering

2. Cross-browser testing
   - Chrome, Firefox, Safari, Edge
   - Windows, macOS, Linux
   - Mobile browsers

3. Performance testing
   - Page load with fonts
   - Lighthouse audit
   - Core Web Vitals

## Related Workflows
- whmcs-css-customization
- whmcs-color-scheme
- whmcs-theme-customization
- whmcs-template-modification