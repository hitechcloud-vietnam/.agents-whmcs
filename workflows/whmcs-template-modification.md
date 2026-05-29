# WHMCS Template Modification Workflow

## Purpose
Guide developers through modifying Smarty templates in WHMCS to customize appearance and functionality.

## Prerequisites
- WHMCS installation
- Basic Smarty template knowledge
- Access to template files
- Understanding of WHMCS template hierarchy

## Steps

### Phase 1: Template Discovery
1. Locate template files
   ```
   /whmcs/templates/
   ├── six/                    # Default theme (WHMCS 8)
   ├── twenty-one/            # WHMCS 2021 theme
   ├── portal/               # Legacy portal theme
   └── orderforms/           # Cart order forms
       ├── classic/           # Classic cart
       ├── default/           # Default cart
       └── twenty-one/        # Modern cart
   ```

2. Identify template types
   - **Layout templates**: header, footer, navigation
   - **Page templates**: client area pages
   - **Email templates**: Email notifications
   - **PDF templates**: Invoice PDFs
   - **Order form templates**: Shopping cart

3. Understand template inheritance
   - Child templates extend parent templates
   - Six theme is base for most templates
   - Override specific files as needed

### Phase 2: Template Modification Basics
1. Modify existing template
   ```smarty
   {extends file="parent:layout.tpl"}
   
   {block name="content"}
   <div class="custom-content">
       {$content}
   </div>
   {/block}
   ```

2. Access WHMCS variables
   ```smarty
   {$client->firstname}           {* Get client data *}
   {$companyname}                 {* Company name *}
   {$BASE_URL_CUSTOM_TEMPLATE}    {* Template assets URL *}
   {$LANG.somekey}               {* Language string *}
   ```

3. Use conditional logic
   ```smarty
   {if $loggedin}
       <p>Welcome, {$client->firstname}!</p>
   {else}
       <p>Please log in.</p>
   {/if}
   ```

### Phase 3: Common Template Modifications

#### Header Modification
```smarty
{* Custom header hooks *}
<header class="site-header">
    {include file="$template/header.tpl"}
    
    {* Add custom announcement bar *}
    {if $announcement}
        <div class="announcement-bar">
            {$announcement}
        </div>
    {/if}
</header>
```

#### Footer Modification
```smarty
{assign var="current_year" value=$smarty.now|date_format:'%Y'}
<footer class="site-footer">
    <div class="container">
        <p>&copy; {$current_year} {$companyname}. All rights reserved.</p>
        
        {* Add social links *}
        <div class="social-links">
            <a href="https://facebook.com/example"><i class="fab fa-facebook"></i></a>
            <a href="https://twitter.com/example"><i class="fab fa-twitter"></i></a>
        </div>
    </div>
</footer>
```

#### Cart Template Modification
```smarty
{* Custom cart item display *}
{foreach $cartitems as $item}
    <div class="cart-item" data-id="{$item.id}">
        <span class="item-name">{$item.name}</span>
        <span class="item-price">{($item.price)|number_format:2}</span>
        
        {* Add custom remove button *}
        <button class="remove-item" data-item-id="{$item.id}">
            <i class="fa fa-trash"></i>
        </button>
    </div>
{/foreach}
```

### Phase 4: Advanced Template Techniques

#### Creating Reusable Blocks
```smarty
{* Create custom component *}
{block name="product-card"}
<div class="product-card">
    <img src="{$product->image}" alt="{$product->name}">
    <h3>{$product->name}</h3>
    <p class="price">{($product->price)|currency}</p>
    <a href="{routePath('product-view', $product->id)}" class="btn btn-primary">
        View Details
    </a>
</div>
{/block}
```

#### Using Hooks in Templates
```smarty
{* Execute hook in template *}
{hook key="clientAreaHomepagePanels"}

{* Custom hook output *}
{foreach $hookOutput as $panel}
    <div class="homepage-panel">
        {$panel}
    </div>
{/foreach}
```

#### Template Pagination
```smarty
{if $paginator}
    <div class="pagination">
        {if $paginator->getPrevUrl()}
            <a href="{$paginator->getPrevUrl()}">&laquo; Previous</a>
        {/if}
        
        {foreach $paginator->getPages() as $page}
            <a href="{$page.url}" class="{if $page.isCurrent}active{/if}">
                {$page.num}
            </a>
        {/foreach}
        
        {if $paginator->getNextUrl()}
            <a href="{$paginator->getNextUrl()}">Next &raquo;</a>
        {/if}
    </div>
{/if}
```

### Phase 5: Template Debugging
1. Enable Smarty debugging
   - WHMCS Admin > Configuration > System Settings > General Settings
   - Enable "Enable Smarty Template Debugging"
   - Debug console appears in new window

2. Variable inspection
   ```smarty
   {* Debug: Show all variables *}
   {$var_dump = $smarty.server}
   <pre>{$var_dump|var_export}</pre>
   
   {* Debug: Show specific variable *}
   {$smarty.get|var_export}
   ```

3. Template logging
   ```php
   // hooks/debug_template.php
   <?php
   add_hook('PostTemplateRender', 1, function($vars) {
       logActivity("Template rendered: " . $vars['templatefile']);
   });
   ```

### Phase 6: Template Version Control
1. Git workflow
   ```bash
   cd /whmcs/templates/mycustom
   git init
   git add .
   git commit -m "Initial template customization"
   
   # Create modification branch
   git checkout -b feature/modification-name
   ```

2. Template backup
   ```bash
   # Backup before modifications
   tar -czf template-backup-$(date +%Y%m%d).tar.gz /whmcs/templates/
   ```

### Phase 7: Performance Optimization
1. Template caching
   - WHMCS caches compiled templates
   - Clear cache after modifications
   - Use {nocache} for dynamic content

2. Minify output
   ```smarty
   {nocache}
   {$dynamicContent}
   {/nocache}
   ```

### Phase 8: Testing Template Changes
1. Local testing
   - Use development environment
   - Test all user flows
   - Check responsive design

2. Staging deployment
   - Deploy to staging server
   - User acceptance testing
   - Cross-browser testing

3. Production deployment
   - Backup current templates
   - Deploy changes
   - Monitor for errors

## Best Practices
- Always use child themes when possible
- Document template modifications
- Use version control for templates
- Test in development before production
- Keep custom code in separate files

## Related Workflows
- whmcs-theme-customization
- whmcs-css-customization
- whmcs-javascript-extensions
- whmcs-email-template-design