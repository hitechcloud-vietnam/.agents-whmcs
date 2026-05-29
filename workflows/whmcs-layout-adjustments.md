# WHMCS Layout Adjustments Workflow

## Purpose
Guide developers through modifying the layout structure of WHMCS pages and components.

## Prerequisites
- WHMCS installation
- HTML/CSS knowledge
- Smarty template understanding
- Bootstrap grid system knowledge

## Steps

### Phase 1: Layout Architecture Analysis

1. Understand WHMCS layout structure
   ```
   Page structure:
   ├── Header
   │   ├── Top bar
   │   ├── Logo
   │   └── Navigation
   ├── Main content
   │   ├── Sidebar (optional)
   │   │   ├── Navigation
   │   │   └── Quick links
   │   └── Content area
   │       ├── Breadcrumbs
   │       ├── Page title
   │       └── Content
   └── Footer
       ├── Company info
       ├── Links
       └── Copyright
   ```

2. Bootstrap grid system
   ```
   Grid options:
   - Extra small (<576px): col-*
   - Small (>=576px): sm-*
   - Medium (>=768px): md-*
   - Large (>=992px): lg-*
   - Extra large (>=1200px): xl-*
   
   Container widths:
   - .container: max-width varies by breakpoint
   - .container-fluid: 100% width
   ```

### Phase 2: Container and Grid Adjustments

1. Full-width layout
   ```css
   /* Full width container */
   .whmcs-container,
   .main-content {
       max-width: 100%;
       padding-left: 0;
       padding-right: 0;
   }
   
   .whmcs-container .container {
       max-width: 100%;
   }
   ```

2. Constrained layout
   ```css
   /* Narrower content area */
   .content-area {
       max-width: 720px;
       margin: 0 auto;
   }
   
   .sidebar-area {
       max-width: 280px;
   }
   ```

3. Side-by-side layout
   ```css
   /* Two-column layout */
   .main-layout {
       display: flex;
       flex-wrap: wrap;
   }
   
   .content-area {
       flex: 1;
       min-width: 0;
   }
   
   .sidebar {
       width: 300px;
       flex-shrink: 0;
   }
   
   @media (max-width: 991px) {
       .sidebar {
           width: 100%;
       }
   }
   ```

### Phase 3: Header Customization

1. Sticky header
   ```css
   /* Sticky navigation */
   .site-header {
       position: sticky;
       top: 0;
       z-index: 1000;
       background: #fff;
       box-shadow: 0 2px 10px rgba(0,0,0,0.1);
   }
   
   body {
       padding-top: 70px; /* Offset for sticky header */
   }
   ```

2. Transparent header
   ```css
   /* Transparent header for landing pages */
   .site-header.header-transparent {
       position: absolute;
       width: 100%;
       background: transparent;
   }
   
   .site-header.header-transparent .nav-link {
       color: #fff;
   }
   ```

3. Header with mega menu
   ```css
   /* Mega menu dropdown */
   .nav-item.mega-menu {
       position: static;
   }
   
   .mega-menu .dropdown-menu {
       width: 100%;
       left: 0;
       right: 0;
       padding: 20px;
   }
   
   .mega-menu .dropdown-menu .row {
       display: flex;
   }
   ```

### Phase 4: Sidebar Customization

1. Collapsible sidebar
   ```css
   /* Collapsible sidebar */
   .sidebar-toggle {
       display: none;
   }
   
   @media (max-width: 991px) {
       .sidebar {
           position: fixed;
           left: -280px;
           top: 0;
           height: 100vh;
           z-index: 1001;
           transition: left 0.3s ease;
           background: #fff;
           box-shadow: 2px 0 10px rgba(0,0,0,0.1);
       }
       
       .sidebar.active {
           left: 0;
       }
       
       .sidebar-toggle {
           display: block;
       }
   }
   ```

2. Right sidebar layout
   ```css
   /* Right sidebar */
   .main-container {
       display: flex;
   }
   
   .main-content {
       order: 1;
   }
   
   .sidebar {
       order: 2;
       margin-left: 30px;
   }
   
   @media (max-width: 991px) {
       .main-container {
           flex-direction: column;
       }
       
       .sidebar {
           order: 2;
           margin-left: 0;
           margin-top: 30px;
       }
   }
   ```

### Phase 5: Content Area Adjustments

1. Card-based layout
   ```css
   /* Card grid layout */
   .content-grid {
       display: grid;
       grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
       gap: 24px;
       padding: 20px 0;
   }
   
   .content-card {
       background: #fff;
       border-radius: 8px;
       padding: 24px;
       box-shadow: 0 2px 8px rgba(0,0,0,0.1);
       transition: transform 0.2s ease, box-shadow 0.2s ease;
   }
   
   .content-card:hover {
       transform: translateY(-4px);
       box-shadow: 0 4px 16px rgba(0,0,0,0.12);
   }
   ```

2. Tabbed content layout
   ```css
   /* Tab-based content */
   .tab-container {
       margin-bottom: 20px;
   }
   
   .tab-content {
       background: #fff;
       padding: 20px;
       border-radius: 8px;
   }
   
   .tab-pane {
       display: none;
   }
   
   .tab-pane.active {
       display: block;
   }
   ```

3. Accordion layout
   ```css
   /* Accordion sections */
   .accordion {
       border: 1px solid var(--color-border);
       border-radius: 8px;
       overflow: hidden;
   }
   
   .accordion-item {
       border-bottom: 1px solid var(--color-border);
   }
   
   .accordion-item:last-child {
       border-bottom: none;
   }
   
   .accordion-header {
       padding: 16px 20px;
       background: var(--color-gray-100);
       cursor: pointer;
       display: flex;
       justify-content: space-between;
       align-items: center;
   }
   
   .accordion-body {
       padding: 20px;
       display: none;
   }
   
   .accordion-item.active .accordion-body {
       display: block;
   }
   ```

### Phase 6: Footer Customization

1. Multi-column footer
   ```css
   /* Footer layout */
   .site-footer {
       background: #1a1a2e;
       color: #fff;
       padding: 60px 0 30px;
   }
   
   .footer-widgets {
       display: grid;
       grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
       gap: 40px;
       margin-bottom: 40px;
   }
   
   .footer-column h4 {
       color: #fff;
       font-size: 1.125rem;
       margin-bottom: 20px;
   }
   
   .footer-column ul {
       list-style: none;
       padding: 0;
       margin: 0;
   }
   
   .footer-column li {
       margin-bottom: 10px;
   }
   
   .footer-column a {
       color: rgba(255,255,255,0.7);
       transition: color 0.2s;
   }
   
   .footer-column a:hover {
       color: #fff;
   }
   ```

2. Minimal footer
   ```css
   /* Compact footer */
   .site-footer-minimal {
       background: var(--color-gray-100);
       padding: 20px 0;
       text-align: center;
   }
   
   .footer-links {
       margin-bottom: 10px;
   }
   
   .footer-links a {
       color: var(--color-text-muted);
       margin: 0 10px;
       font-size: var(--font-size-sm);
   }
   
   .copyright {
       font-size: var(--font-size-xs);
       color: var(--color-text-muted);
   }
   ```

### Phase 7: Page-Specific Layouts

1. Dashboard layout
   ```css
   /* Client dashboard */
   .dashboard-container {
       display: grid;
       grid-template-columns: 260px 1fr;
       gap: 30px;
       min-height: 500px;
   }
   
   .dashboard-sidebar {
       background: #fff;
       border-radius: 12px;
       padding: 20px;
   }
   
   .dashboard-content {
       display: grid;
       grid-template-columns: repeat(2, 1fr);
       gap: 20px;
   }
   ```

2. Invoice layout
   ```css
   /* Invoice page layout */
   .invoice-container {
       max-width: 800px;
       margin: 0 auto;
       padding: 40px 20px;
   }
   
   .invoice-header {
       display: flex;
       justify-content: space-between;
       margin-bottom: 40px;
   }
   
   .invoice-items table {
       width: 100%;
   }
   
   .invoice-total {
       text-align: right;
       margin-top: 30px;
       padding-top: 20px;
       border-top: 2px solid var(--color-border);
   }
   ```

### Phase 8: Spacing and Padding Adjustments

1. Content spacing
   ```css
   /* Consistent spacing */
   .section {
       padding: 60px 0;
   }
   
   .section-sm {
       padding: 30px 0;
   }
   
   .section-lg {
       padding: 100px 0;
   }
   
   @media (max-width: 767px) {
       .section {
           padding: 40px 0;
       }
       
       .section-lg {
           padding: 60px 0;
       }
   }
   ```

2. Component spacing
   ```css
   /* Component margins */
   .component-group {
       margin-bottom: 24px;
   }
   
   .component-spaced {
       margin: 16px 0;
   }
   
   .elements-inline {
       display: inline-flex;
       gap: 12px;
   }
   ```

## Responsive Layout Checklist
- Mobile navigation functional
- Grid adapts to screen sizes
- Content readable on all devices
- Touch targets are adequate (44px minimum)
- No horizontal scrolling

## Related Workflows
- whmcs-responsive-tuning
- whmcs-sidebar-modification
- whmcs-header-customization
- whmcs-footer-customization