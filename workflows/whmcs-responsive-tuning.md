# WHMCS Responsive Tuning Workflow

## Purpose
Guide developers through optimizing WHMCS for mobile and responsive design.

## Prerequisites
- WHMCS installation
- CSS knowledge (Media queries)
- Browser developer tools
- Mobile testing devices or emulators

## Steps

### Phase 1: Responsive Framework Analysis

1. Identify current framework
   ```
   WHMCS 8.x uses Bootstrap 4.3
   - Grid system: 12 columns
   - Breakpoints: sm(576px), md(768px), lg(992px), xl(1200px)
   - Container max-widths vary by breakpoint
   ```

2. Default container widths
   ```
   Bootstrap 4 containers:
   - Extra small (<576px): 100%
   - Small (>=576px): 540px
   - Medium (>=768px): 720px
   - Large (>=992px): 960px
   - Extra large (>=1200px): 1140px
   ```

3. Inspect current responsiveness
   - Resize browser window
   - Check all pages on different sizes
   - Identify breaking points

### Phase 2: Viewport Configuration

1. Ensure proper viewport meta
   ```html
   <!-- In header.tpl - must be first in head -->
   <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
   ```

2. Enable responsive features
   ```css
   /* CSS for responsive behavior */
   *, *::before, *::after {
       box-sizing: border-box;
   }
   
   html {
       font-size: 16px;
       -webkit-text-size-adjust: 100%;
   }
   ```

### Phase 3: Grid System Implementation

1. Bootstrap responsive classes
   ```html
   <!-- Mobile-first approach -->
   <div class="container">
       <!-- Stack on mobile, side-by-side on larger -->
       <div class="row">
           <div class="col-12 col-md-8">Main content</div>
           <div class="col-12 col-md-4">Sidebar</div>
       </div>
   </div>
   ```

2. Column combinations
   ```html
   <!-- Full width on mobile, half on tablet, third on desktop -->
   <div class="col-12 col-sm-6 col-lg-4">Content</div>
   
   <!-- Auto-fit columns -->
   <div class="col-sm">Flexible</div>
   <div class="col-sm">Flexible</div>
   ```

3. Flexbox utilities
   ```html
   <!-- Responsive flex layout -->
   <div class="d-flex flex-column flex-md-row">
       <div>Item 1</div>
       <div>Item 2</div>
   </div>
   ```

### Phase 4: Mobile Navigation

1. Navbar collapse behavior
   ```css
   /* Mobile navigation */
   .navbar-collapse {
       max-height: 0;
       overflow: hidden;
       transition: max-height 0.3s ease;
   }
   
   .navbar-collapse.show {
       max-height: 500px;
       overflow-y: auto;
   }
   ```

2. Hamburger menu styling
   ```css
   /* Mobile menu toggle */
   .navbar-toggler {
       padding: 10px 12px;
       border-radius: 4px;
       border: 1px solid rgba(0,0,0,0.1);
   }
   
   .navbar-toggler:focus {
       outline: none;
       box-shadow: 0 0 0 3px rgba(0,123,255,0.25);
   }
   
   .navbar-toggler-icon {
       width: 24px;
       height: 2px;
       background: currentColor;
       position: relative;
   }
   
   .navbar-toggler-icon::before,
   .navbar-toggler-icon::after {
       content: '';
       position: absolute;
       left: 0;
       width: 100%;
       height: 2px;
       background: currentColor;
       transition: transform 0.3s;
   }
   
   .navbar-toggler-icon::before {
       top: -8px;
   }
   
   .navbar-toggler-icon::after {
       bottom: -8px;
   }
   ```

3. Mobile dropdown menus
   ```css
   /* Touch-friendly dropdowns */
   .nav-item.dropdown {
       position: relative;
   }
   
   .dropdown-menu {
       position: absolute;
       top: 100%;
       left: 0;
       min-width: 200px;
   }
   
   @media (max-width: 991px) {
       .dropdown-menu {
           position: static;
           width: 100%;
           background: #f8f9fa;
           border: none;
           box-shadow: none;
       }
   }
   ```

### Phase 5: Touch-Friendly Elements

1. Button sizing
   ```css
   /* Minimum touch target size */
   .btn,
   button,
   a.btn,
   input[type="submit"] {
       min-height: 44px;
       min-width: 44px;
       padding: 10px 20px;
   }
   
   /* Larger buttons on mobile */
   @media (max-width: 576px) {
       .btn-lg {
           width: 100%;
           padding: 14px 24px;
       }
   }
   ```

2. Form inputs
   ```css
   /* Touch-friendly inputs */
   .form-control,
   select,
   textarea {
       min-height: 44px;
       padding: 12px 16px;
       font-size: 16px; /* Prevents zoom on iOS */
   }
   
   /* Larger checkboxes/radios */
   .custom-control-label::before,
   .custom-control-label::after {
       width: 22px;
       height: 22px;
       top: 0.125rem;
       left: -1.5rem;
   }
   ```

3. Links spacing
   ```css
   /* Touch-friendly links */
   a,
   .nav-link {
       min-height: 44px;
       display: inline-flex;
       align-items: center;
       padding: 8px 12px;
   }
   
   @media (max-width: 576px) {
       .list-inline-item {
           display: block;
           margin-bottom: 10px;
       }
   }
   ```

### Phase 6: Responsive Typography

1. Fluid typography
   ```css
   /* Responsive font sizes using clamp() */
   h1 {
       font-size: clamp(1.75rem, 4vw, 3rem);
   }
   
   h2 {
       font-size: clamp(1.5rem, 3vw, 2.25rem);
   }
   
   h3 {
       font-size: clamp(1.25rem, 2.5vw, 1.75rem);
   }
   
   body {
       font-size: clamp(14px, 2vw, 16px);
   }
   ```

2. Media query typography
   ```css
   /* Mobile typography */
   @media (max-width: 767px) {
       body {
           font-size: 15px;
           line-height: 1.6;
       }
       
       h1 { font-size: 1.75rem; }
       h2 { font-size: 1.5rem; }
       h3 { font-size: 1.25rem; }
       
       p {
           margin-bottom: 1rem;
       }
   }
   
   /* Tablet typography */
   @media (min-width: 768px) and (max-width: 991px) {
       body {
           font-size: 16px;
       }
   }
   ```

### Phase 7: Responsive Components

1. Cards on mobile
   ```css
   /* Stack cards on mobile */
   .card-grid {
       display: grid;
       gap: 16px;
   }
   
   @media (max-width: 575px) {
       .card-grid {
           grid-template-columns: 1fr;
       }
   }
   
   @media (min-width: 576px) and (max-width: 991px) {
       .card-grid {
           grid-template-columns: repeat(2, 1fr);
       }
   }
   
   @media (min-width: 992px) {
       .card-grid {
           grid-template-columns: repeat(3, 1fr);
       }
   }
   ```

2. Tables on mobile
   ```css
   /* Horizontal scroll for tables */
   .table-responsive {
       overflow-x: auto;
       -webkit-overflow-scrolling: touch;
   }
   
   /* Card view for table data on mobile */
   @media (max-width: 767px) {
       .table-mobile-cards {
           border: 0;
       }
       
       .table-mobile-cards thead {
           display: none;
       }
       
       .table-mobile-cards tbody tr {
           display: block;
           margin-bottom: 16px;
           border: 1px solid #dee2e6;
           border-radius: 8px;
       }
       
       .table-mobile-cards td {
           display: flex;
           justify-content: space-between;
           padding: 12px 16px;
           border-bottom: 1px solid #dee2e6;
       }
       
       .table-mobile-cards td::before {
           content: attr(data-label);
           font-weight: 600;
       }
   }
   ```

3. Modals on mobile
   ```css
   /* Full-screen modal on mobile */
   @media (max-width: 575px) {
       .modal-dialog {
           max-width: 100%;
           height: 100vh;
           margin: 0;
       }
       
       .modal-content {
           height: 100%;
           border-radius: 0;
       }
       
       .modal-body {
           overflow-y: auto;
       }
   }
   ```

### Phase 8: Image and Media Responsive

1. Responsive images
   ```css
   /* Fluid images */
   img {
       max-width: 100%;
       height: auto;
   }
   
   /* Lazy load images */
   img[loading="lazy"] {
       opacity: 0;
       transition: opacity 0.3s;
   }
   
   img[loading="lazy"].loaded {
       opacity: 1;
   }
   ```

2. Picture element for art direction
   ```html
   <picture>
       <source media="(max-width: 575px)" srcset="/img/product-mobile.jpg">
       <source media="(min-width: 576px)" srcset="/img/product-desktop.jpg">
       <img src="/img/product-desktop.jpg" alt="Product">
   </picture>
   ```

3. Video embeds
   ```css
   /* Responsive video container */
   .video-container {
       position: relative;
       padding-bottom: 56.25%; /* 16:9 aspect ratio */
       height: 0;
       overflow: hidden;
   }
   
   .video-container iframe,
   .video-container video {
       position: absolute;
       top: 0;
       left: 0;
       width: 100%;
       height: 100%;
   }
   ```

### Phase 9: Performance Optimization

1. Reduce payload on mobile
   ```css
   /* Hide heavy elements on mobile */
   @media (max-width: 767px) {
       .hide-mobile {
           display: none !important;
       }
       
       /* Lazy load background images */
       .bg-image-mobile {
           background-image: none !important;
       }
   }
   ```

2. Optimize loading
   ```html
   <!-- Async CSS for non-critical styles -->
   <link rel="preload" href="/css/mobile.css" as="style" media="screen and (max-width: 767px)" onload="this.onload=null;this.rel='stylesheet'">
   ```

3. Touch optimization
   ```css
   /* Remove tap highlight */
   * {
       -webkit-tap-highlight-color: transparent;
   }
   
   /* Smooth scrolling */
   html {
       scroll-behavior: smooth;
   }
   
   /* Prevent zoom on inputs */
   @media (max-width: 767px) {
       input, select, textarea {
           font-size: 16px; /* iOS requires 16px to prevent zoom */
       }
   }
   ```

### Phase 10: Testing and Validation

1. Browser testing tools
   - Chrome DevTools Device Mode
   - Firefox Responsive Design Mode
   - Safari Web Inspector

2. Real device testing
   - iOS Safari (iPhone/iPad)
   - Android Chrome
   - Samsung Internet

3. Accessibility testing
   - Touch target sizes
   - Readable text sizes
   - Color contrast

## Responsive Design Checklist
- [ ] Viewport meta tag present
- [ ] Grid system works at all breakpoints
- [ ] Navigation collapses properly
- [ ] Touch targets are 44px minimum
- [ ] Font sizes readable on mobile
- [ ] Tables scroll horizontally
- [ ] Images scale responsively
- [ ] No horizontal scroll on mobile

## Related Workflows
- whmcs-css-customization
- whmcs-layout-adjustments
- whmcs-theme-customization
- whmcs-button-styling