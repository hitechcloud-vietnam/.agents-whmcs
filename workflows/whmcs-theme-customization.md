# WHMCS Theme Customization Workflow

## Purpose
Guide developers through creating a custom WHMCS theme from scratch or modifying existing themes.

## Prerequisites
- WHMCS installation (v8.x recommended)
- PHP 7.4+ / 8.x
- FTP/SSH access to WHMCS files
- Basic HTML/CSS/PHP knowledge
- Smarty template engine understanding

## Steps

### Phase 1: Environment Setup
1. Create development environment
   ```bash
   # Clone WHMCS or use existing installation
   cd /var/www/whmcs
   
   # Create theme directory
   mkdir -p templates/mycustomtheme
   ```
2. Enable debug mode in WHMCS
   - Navigate to: Configuration > System Settings > General Settings
   - Enable Smarty template debugging
   - Enable WHMCS debug mode for development

3. Set up version control
   ```bash
   cd templates/mycustomtheme
   git init
   echo "*.log" > .gitignore
   echo "/node_modules" >> .gitignore
   ```

### Phase 2: Theme Structure Analysis
1. Examine existing theme structure
   ```
   templates/
   ├── six/                    # Default WHMCS 8 theme
   │   ├── layout/
   │   │   ├── header.tpl
   │   │   ├── footer.tpl
   │   │   ├── sidebar.tpl
   │   │   └── navigation.tpl
   │   ├── css/
   │   │   ├── bootstrap.css
   │   │   └── custom.css
   │   ├── js/
   │   │   └── custom.js
   │   └── templates/          # Page templates
   │       ├── clienthome.tpl
   │       ├── cart.tpl
   │       └── invoices.tpl
   └── orderforms/             # Cart/ordering themes
       └── default/
   ```

2. Document current theme features
   - List all template files
   - Note CSS framework used (Bootstrap 3/4/5)
   - Identify JavaScript dependencies
   - Map template inheritance patterns

### Phase 3: Theme Creation
1. Create theme configuration file
   ```php
   // templates/mycustomtheme/theme.php
   <?php
   return [
       'name' => 'My Custom Theme',
       'author' => 'Your Name',
       'version' => '1.0.0',
       'description' => 'Custom WHMCS theme with brand identity',
       'supports' => '8.x',
       'parents' => ['six'],  // Extend existing theme
       'basedOn' => 'six',
   ];
   ```

2. Create directory structure
   ```bash
   mkdir -p templates/mycustomtheme/{layout,css,js,img,scss,components}
   ```

3. Create layout files
   - header.tpl: Main HTML head, navigation, top bar
   - footer.tpl: Footer content, scripts, copyright
   - sidebar.tpl: Sidebar components
   - navigation.tpl: Menu structure

4. Copy base templates to modify
   ```bash
   cp -r templates/six/* templates/mycustomtheme/
   ```

### Phase 4: Template Customization
1. Modify header.tpl
   ```smarty
   {assign var="logo_url" value=$BASE_URL_CUSTOM_TEMPLATE|cat:"/img/logo.svg"}
   <header class="site-header">
       <nav class="navbar navbar-expand-lg">
           <a class="navbar-brand" href="{$WEB_ROOT}/">
               <img src="{$logo_url}" alt="{$companyname}" height="40">
           </a>
       </nav>
   </header>
   ```

2. Customize footer.tpl
   ```smarty
   <footer class="site-footer bg-dark text-white py-4">
       <div class="container">
           <div class="row">
               <div class="col-md-6">
                   <p>&copy; {$date_year} {$companyname}. All rights reserved.</p>
               </div>
               <div class="col-md-6 text-md-right">
                   <a href="{$WEB_ROOT}/privacy.php">Privacy Policy</a>
               </div>
           </div>
       </div>
   </footer>
   ```

3. Create custom page templates
   - clienthome.tpl: Client area homepage
   - supporttickets.tpl: Ticket list
   - invoices.tpl: Invoice listing
   - domainchecker.tpl: Domain search

### Phase 5: CSS Customization
1. Create SCSS structure
   ```
   scss/
   ├── _variables.scss      # Custom variables
   ├── _mixins.scss         # Mixins
   ├── _typography.scss     # Font settings
   ├── _colors.scss        # Color scheme
   ├── _components.scss     # Component styles
   ├── _layout.scss        # Layout styles
   └── main.scss           # Main import file
   ```

2. Compile CSS
   ```bash
   # Install dependencies
   npm init -y
   npm install sass --save-dev
   
   # Compile
   npx sass scss:css
   ```

3. Enqueue styles in WHMCS
   ```php
   // hooks/clientarea_page_head.php
   <?php
   use WHMCS\View\Markup\MarkupResolver;
   
   add_hook('ClientAreaPageHead', 1, function($vars) {
       $version = '1.0.0';
       echo '<link rel="stylesheet" href="/templates/mycustomtheme/css/main.min.css?v='.$version.'">';
   });
   ```

### Phase 6: JavaScript Customizations
1. Create custom.js
   ```javascript
   // Custom JavaScript functionality
   (function($) {
       'use strict';
       
       // Initialize custom components
       $(document).ready(function() {
           initCustomNavigation();
           initCustomModals();
           initCustomValidation();
       });
       
       function initCustomNavigation() {
           // Custom navigation behavior
       }
       
       function initCustomModals() {
           // Custom modal handlers
       }
       
       function initCustomValidation() {
           // Custom form validation
       }
   })(jQuery);
   ```

2. Enqueue scripts
   ```php
   add_hook('ClientAreaPageHead', 1, function($vars) {
       echo '<script src="/templates/mycustomtheme/js/custom.js"></script>';
   });
   ```

### Phase 7: Testing
1. Activate theme in WHMCS
   - Go to: Configuration > System Settings > General Settings > Customizing
   - Select your custom theme
   - Save settings

2. Test across pages
   - Homepage
   - Client area pages
   - Order process
   - Cart/checkout
   - Invoice pages
   - Support tickets

3. Browser testing
   - Chrome, Firefox, Safari, Edge
   - Mobile responsive testing
   - Accessibility testing

### Phase 8: Deployment
1. Production checklist
   - Minify CSS/JS
   - Compress images
   - Update version numbers
   - Create backup of previous theme

2. Deploy files
   ```bash
   # Via FTP/SFTP
   rsync -avz --delete templates/mycustomtheme/ user@server:/var/www/whmcs/templates/mycustomtheme/
   ```

3. Post-deployment verification
   - Clear WHMCS cache
   - Test critical user flows
   - Monitor for errors

## Troubleshooting
- Enable Smarty debugging: Shows template variables and includes
- Check error logs: /whmcs/logs/
- Clear cache: Configuration > System Settings > Cache Management

## Related Workflows
- whmcs-template-modification
- whmcs-css-customization
- whmcs-logo-branding
- whmcs-color-scheme