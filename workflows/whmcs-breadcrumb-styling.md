# WHMCS Breadcrumb Styling Workflow

## Purpose
Guide developers through customizing breadcrumbs in WHMCS.

## Prerequisites
- WHMCS installation
- HTML/CSS knowledge
- Smarty template understanding

## Steps

### Phase 1: Breadcrumb Structure

1. Breadcrumb template location
   ```
   /whmcs/templates/six/
   └── includes/breadcrumb.tpl
   ```

2. Default breadcrumb structure
   ```smarty
   {if $breadcrumb}
   <nav aria-label="breadcrumb" class="breadcrumb-wrapper">
       <ol class="breadcrumb">
           {foreach $breadcrumb as $crumb}
               <li class="breadcrumb-item {if $crumb@last}active{/if}">
                   {if !$crumb@last}
                       <a href="{$crumb.link}">{$crumb.label}</a>
                   {else}
                       {$crumb.label}
                   {/if}
               </li>
           {/foreach}
       </ol>
   </nav>
   {/if}
   ```

### Phase 2: Basic Breadcrumb Styling

1. Breadcrumb CSS
   ```css
   .breadcrumb-wrapper {
       padding: 15px 0;
       margin-bottom: 20px;
   }
   
   .breadcrumb {
       background: transparent;
       padding: 0;
       margin: 0;
       font-size: 14px;
   }
   
   .breadcrumb-item {
       color: #6c757d;
   }
   
   .breadcrumb-item a {
       color: #007bff;
       text-decoration: none;
   }
   
   .breadcrumb-item a:hover {
       text-decoration: underline;
   }
   
   .breadcrumb-item + .breadcrumb-item::before {
       content: "/";
       color: #adb5bd;
       padding: 0 8px;
   }
   
   .breadcrumb-item.active {
       color: #495057;
       font-weight: 500;
   }
   ```

### Phase 3: Modern Breadcrumb Styles

1. With icons
   ```smarty
   <nav aria-label="breadcrumb" class="breadcrumb-nav">
       <ol class="breadcrumb modern">
           <li class="breadcrumb-item">
               <a href="{$WEB_ROOT}/">
                   <i class="fa fa-home"></i>
               </a>
           </li>
           {foreach $breadcrumb as $crumb}
               {if !$crumb@last}
                   <li class="breadcrumb-item">
                       <a href="{$crumb.link}">{$crumb.label}</a>
                       <i class="fa fa-chevron-right"></i>
                   </li>
               {else}
                   <li class="breadcrumb-item active">
                       {$crumb.label}
                   </li>
               {/if}
           {/foreach}
       </ol>
   </nav>
   ```

2. Modern breadcrumb CSS
   ```css
   .breadcrumb.modern {
       display: flex;
       align-items: center;
       flex-wrap: wrap;
       gap: 8px;
   }
   
   .breadcrumb.modern .breadcrumb-item {
       display: flex;
       align-items: center;
       gap: 8px;
   }
   
   .breadcrumb.modern .breadcrumb-item a {
       color: #6c757d;
       transition: color 0.2s;
   }
   
   .breadcrumb.modern .breadcrumb-item a:hover {
       color: #007bff;
   }
   
   .breadcrumb.modern .breadcrumb-item i {
       font-size: 10px;
       color: #adb5bd;
   }
   
   .breadcrumb.modern .breadcrumb-item.active {
       color: #212529;
       font-weight: 500;
   }
   ```

### Phase 4: With Separator Icons

1. Chevron separator
   ```css
   .breadcrumb.chevron .breadcrumb-item + .breadcrumb-item::before {
       content: "";
       font-family: 'Font Awesome 5 Free';
       content: "\f054";
       font-weight: 900;
       font-size: 10px;
       color: #adb5bd;
       padding: 0;
       margin: 0 5px;
   }
   ```

2. Arrow separator
   ```css
   .breadcrumb.arrow .breadcrumb-item + .breadcrumb-item::before {
       content: "\2192";
       font-size: 14px;
       color: #ced4da;
       padding: 0 10px;
   }
   ```

3. Dot separator
   ```css
   .breadcrumb.dots .breadcrumb-item + .breadcrumb-item::before {
       content: "\2022";
       font-size: 18px;
       color: #adb5bd;
       padding: 0 10px;
       vertical-align: middle;
   }
   ```

### Phase 5: Pill Style Breadcrumbs

1. Pill breadcrumb CSS
   ```css
   .breadcrumb.pills {
       background: #f8f9fa;
       padding: 12px 20px;
       border-radius: 25px;
   }
   
   .breadcrumb.pills .breadcrumb-item a {
       background: #fff;
       padding: 6px 15px;
       border-radius: 15px;
       color: #495057;
       box-shadow: 0 1px 3px rgba(0,0,0,0.1);
       transition: all 0.2s;
   }
   
   .breadcrumb.pills .breadcrumb-item a:hover {
       background: #007bff;
       color: #fff;
   }
   
   .breadcrumb.pills .breadcrumb-item.active a {
       background: #007bff;
       color: #fff;
       cursor: default;
   }
   
   .breadcrumb.pills .breadcrumb-item + .breadcrumb-item::before {
       content: "\2192";
       color: #adb5bd;
   }
   ```

### Phase 6: With Background

1. Background breadcrumb
   ```css
   .breadcrumb.with-bg {
       background: #e9ecef;
       padding: 15px 20px;
       border-radius: 8px;
   }
   
   .breadcrumb.with-bg .breadcrumb-item {
       font-size: 13px;
   }
   ```

2. Dark variant
   ```css
   .breadcrumb.dark {
       background: #212529;
       padding: 15px 20px;
       border-radius: 8px;
   }
   
   .breadcrumb.dark .breadcrumb-item {
       color: rgba(255,255,255,0.7);
   }
   
   .breadcrumb.dark .breadcrumb-item a {
       color: rgba(255,255,255,0.7);
   }
   
   .breadcrumb.dark .breadcrumb-item a:hover {
       color: #fff;
   }
   
   .breadcrumb.dark .breadcrumb-item + .breadcrumb-item::before {
       color: rgba(255,255,255,0.4);
   }
   
   .breadcrumb.dark .breadcrumb-item.active {
       color: #fff;
   }
   ```

### Phase 7: Custom Breadcrumb Template

1. Create custom template
   ```smarty
   {* Custom breadcrumb template *}
   <nav class="custom-breadcrumb" aria-label="breadcrumb">
       <div class="breadcrumb-container">
           <a href="{$WEB_ROOT}/" class="breadcrumb-home">
               <i class="fa fa-home"></i>
               <span>Home</span>
           </a>
           
           {foreach $breadcrumb as $crumb}
               <div class="breadcrumb-separator">
                   <i class="fa fa-angle-right"></i>
               </div>
               
               {if $crumb@last}
                   <span class="breadcrumb-current">{$crumb.label}</span>
               {else}
                   <a href="{$crumb.link}" class="breadcrumb-link">
                       {$crumb.label}
                   </a>
               {/if}
           {/foreach}
       </div>
   </nav>
   ```

2. Custom breadcrumb CSS
   ```css
   .custom-breadcrumb {
       padding: 0;
       margin-bottom: 25px;
   }
   
   .breadcrumb-container {
       display: flex;
       align-items: center;
       flex-wrap: wrap;
   }
   
   .breadcrumb-home {
       display: flex;
       align-items: center;
       gap: 8px;
       color: #007bff;
       text-decoration: none;
       font-size: 14px;
   }
   
   .breadcrumb-home:hover {
       text-decoration: underline;
   }
   
   .breadcrumb-separator {
       color: #adb5bd;
       margin: 0 10px;
   }
   
   .breadcrumb-link {
       color: #6c757d;
       text-decoration: none;
       font-size: 14px;
   }
   
   .breadcrumb-link:hover {
       color: #007bff;
   }
   
   .breadcrumb-current {
       color: #212529;
       font-weight: 500;
       font-size: 14px;
   }
   ```

### Phase 8: Responsive Breadcrumbs

1. Mobile breadcrumb
   ```css
   @media (max-width: 576px) {
       .breadcrumb-wrapper {
           overflow-x: auto;
           -webkit-overflow-scrolling: touch;
       }
       
       .breadcrumb {
           white-space: nowrap;
           flex-wrap: nowrap;
       }
       
       .breadcrumb-item {
           font-size: 12px;
       }
       
       .breadcrumb-item a,
       .breadcrumb-item span {
           display: inline-block;
       }
   }
   ```

## Related Workflows
- whmcs-template-modification
- whmcs-css-customization
- whmcs-layout-adjustments
- whmcs-navigation-tuning