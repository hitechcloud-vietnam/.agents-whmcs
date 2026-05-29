# WHMCS Color Scheme Customization Workflow

## Purpose
Guide developers through implementing custom color schemes in WHMCS for brand consistency.

## Prerequisites
- WHMCS installation
- CSS/SCSS knowledge
- Understanding of color theory
- Design system or brand guidelines

## Steps

### Phase 1: Color System Planning

1. Define brand colors
   ```
   Primary color palette:
   ├── Primary: #007bff (Main brand color)
   ├── Primary Dark: #0056b3 (Hover/active states)
   ├── Primary Light: #cce5ff (Backgrounds)
   ├── Secondary: #6c757d (Secondary elements)
   └── Accent: #28a745 (Success/positive actions)
   ```

2. Color structure
   ```
   System colors:
   ├── Success: #28a745
   ├── Warning: #ffc107
   ├── Danger: #dc3545
   ├── Info: #17a2b8
   ├── Light: #f8f9fa
   └── Dark: #343a40
   ```

3. Create color variables
   ```scss
   // _color-variables.scss
   :root {
       // Primary colors
       --color-primary: #007bff;
       --color-primary-dark: #0056b3;
       --color-primary-light: #cce5ff;
       
       // Secondary colors
       --color-secondary: #6c757d;
       --color-secondary-dark: #545b62;
       --color-secondary-light: #e9ecef;
       
       // Accent colors
       --color-success: #28a745;
       --color-warning: #ffc107;
       --color-danger: #dc3545;
       --color-info: #17a2b8;
       
       // Neutral colors
       --color-white: #ffffff;
       --color-black: #000000;
       --color-gray-100: #f8f9fa;
       --color-gray-200: #e9ecef;
       --color-gray-300: #dee2e6;
       --color-gray-400: #ced4da;
       --color-gray-500: #adb5bd;
       --color-gray-600: #6c757d;
       --color-gray-700: #495057;
       --color-gray-800: #343a40;
       --color-gray-900: #212529;
       
       // Semantic colors
       --color-text: #212529;
       --color-text-muted: #6c757d;
       --color-background: #ffffff;
       --color-border: #dee2e6;
   }
   ```

### Phase 2: Bootstrap Variable Override

1. Override Bootstrap 4 variables
   ```scss
   // Override Bootstrap variables before import
   // Custom Bootstrap variables
   $primary: #007bff;
   $secondary: #6c757d;
   $success: #28a745;
   $info: #17a2b8;
   $warning: #ffc107;
   $danger: #dc3545;
   $light: #f8f9fa;
   $dark: #343a40;
   
   // Theme defaults
   $body-bg: #ffffff;
   $body-color: #212529;
   
   // Component defaults
   $border-radius: 0.25rem;
   $border-radius-lg: 0.3rem;
   $border-radius-sm: 0.2rem;
   
   // Button styles
   $btn-border-radius: 0.25rem;
   $btn-padding-y: 0.5rem;
   $btn-padding-x: 1rem;
   ```

2. Generate new Bootstrap CSS
   ```bash
   # Compile with custom variables
   sass custom-bootstrap.scss assets/css/bootstrap-custom.css
   ```

### Phase 3: WHMCS Six Theme Colors

1. Six theme color overrides
   ```css
   /* Six theme specific colors */
   .whmcs-container {
       /* Background colors */
       --bg-primary: #ffffff;
       --bg-secondary: #f8f9fa;
       --bg-dark: #1a1a2e;
       
       /* Text colors */
       --text-primary: #212529;
       --text-secondary: #6c757d;
       --text-light: #ffffff;
       
       /* Accent colors */
       --accent-primary: #007bff;
       --accent-hover: #0056b3;
   }
   ```

2. Navigation colors
   ```css
   /* Header/navigation colors */
   #header {
       background-color: var(--header-bg);
   }
   
   .header-nav {
       background-color: var(--nav-bg);
   }
   
   .nav-link {
       color: var(--nav-link-color);
   }
   
   .nav-link:hover,
   .nav-link.active {
       color: var(--nav-link-hover);
   }
   ```

### Phase 4: Button Color Customization

1. Primary button
   ```css
   .btn-primary {
       background-color: var(--color-primary);
       border-color: var(--color-primary);
       color: #ffffff;
   }
   
   .btn-primary:hover,
   .btn-primary:focus {
       background-color: var(--color-primary-dark);
       border-color: var(--color-primary-dark);
   }
   
   .btn-primary:active {
       background-color: #004a99;
       border-color: #004a99;
   }
   ```

2. Secondary button
   ```css
   .btn-secondary {
       background-color: var(--color-secondary);
       border-color: var(--color-secondary);
   }
   
   .btn-outline-primary {
       color: var(--color-primary);
       border-color: var(--color-primary);
   }
   
   .btn-outline-primary:hover {
       background-color: var(--color-primary);
       color: #ffffff;
   }
   ```

3. Success/Warning/Danger buttons
   ```css
   .btn-success {
       background-color: var(--color-success);
       border-color: var(--color-success);
   }
   
   .btn-warning {
       background-color: var(--color-warning);
       color: #212529;
   }
   
   .btn-danger {
       background-color: var(--color-danger);
       border-color: var(--color-danger);
   }
   ```

### Phase 5: Form Elements Colors

1. Input fields
   ```css
   .form-control {
       border-color: var(--color-border);
       background-color: var(--color-white);
   }
   
   .form-control:focus {
       border-color: var(--color-primary);
       box-shadow: 0 0 0 0.2rem rgba(0, 123, 255, 0.25);
   }
   
   .form-control:disabled {
       background-color: var(--color-gray-100);
       opacity: 0.6;
   }
   ```

2. Custom selects and checkboxes
   ```css
   /* Custom checkbox colors */
   .custom-checkbox .custom-control-input:checked ~ .custom-control-label::before {
       background-color: var(--color-primary);
       border-color: var(--color-primary);
   }
   
   /* Custom radio colors */
   .custom-radio .custom-control-input:checked ~ .custom-control-label::before {
       background-color: var(--color-primary);
       border-color: var(--color-primary);
   }
   
   /* Custom select */
   .custom-select {
       border-color: var(--color-border);
   }
   
   .custom-select:focus {
       border-color: var(--color-primary);
       box-shadow: 0 0 0 0.2rem rgba(0, 123, 255, 0.25);
   }
   ```

### Phase 6: Card and Container Colors

1. Card styling
   ```css
   .card {
       background-color: var(--color-white);
       border-color: var(--color-border);
       border-radius: var(--border-radius);
   }
   
   .card-header {
       background-color: var(--color-gray-100);
       border-bottom-color: var(--color-border);
   }
   
   .card-footer {
       background-color: var(--color-gray-100);
       border-top-color: var(--color-border);
   }
   ```

2. Table colors
   ```css
   .table {
       color: var(--color-text);
   }
   
   .table thead th {
       background-color: var(--color-gray-100);
       border-color: var(--color-border);
       color: var(--color-text);
   }
   
   .table tbody tr:hover {
       background-color: var(--color-gray-100);
   }
   
   .table-striped tbody tr:nth-of-type(odd) {
       background-color: var(--color-gray-50);
   }
   ```

### Phase 7: Alert and Notification Colors

1. Alert boxes
   ```css
   .alert {
       border-radius: var(--border-radius);
   }
   
   .alert-success {
       background-color: #d4edda;
       border-color: #c3e6cb;
       color: #155724;
   }
   
   .alert-info {
       background-color: #d1ecf1;
       border-color: #bee5eb;
       color: #0c5460;
   }
   
   .alert-warning {
       background-color: #fff3cd;
       border-color: #ffeeba;
       color: #856404;
   }
   
   .alert-danger {
       background-color: #f8d7da;
       border-color: #f5c6cb;
       color: #721c24;
   }
   ```

2. Toast notifications
   ```css
   .toast {
       background-color: var(--color-white);
       border-radius: var(--border-radius);
   }
   
   .toast-success {
       background-color: var(--color-success);
       color: var(--color-white);
   }
   
   .toast-error {
       background-color: var(--color-danger);
       color: var(--color-white);
   }
   ```

### Phase 8: Dark Mode Color Scheme

1. Dark mode variables
   ```scss
   @media (prefers-color-scheme: dark) {
       :root {
           --color-background: #1a1a2e;
           --color-surface: #16213e;
           --color-text: #e9ecef;
           --color-text-muted: #adb5bd;
           --color-border: #495057;
           
           --color-primary: #4dabf7;
           --color-primary-dark: #339af0;
           --color-primary-light: #1864ab;
       }
   }
   ```

2. Dark mode implementation
   ```css
   [data-theme="dark"] {
       --bg-body: #1a1a2e;
       --bg-card: #16213e;
       --text-color: #e9ecef;
       --border-color: #495057;
       
       .btn-primary {
           background-color: var(--color-primary);
       }
   }
   ```

### Phase 9: Color Accessibility

1. Color contrast compliance
   ```
   WCAG 2.1 contrast ratios:
   - Normal text: 4.5:1 minimum
   - Large text (18px+): 3:1 minimum
   - UI components: 3:1 minimum
   ```

2. Focus states
   ```css
   /* Visible focus indicators */
   :focus-visible {
       outline: 2px solid var(--color-primary);
       outline-offset: 2px;
   }
   
   button:focus,
   input:focus,
   select:focus {
       border-color: var(--color-primary);
       box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.25);
   }
   ```

## Verification
- Test color contrast with tools like WebAIM
- Check all button states (default, hover, active, disabled)
- Verify focus states for accessibility
- Test dark mode if implemented

## Related Workflows
- whmcs-css-customization
- whmcs-theme-customization
- whmcs-button-styling
- whmcs-form-styling