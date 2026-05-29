# WHMCS Header Customization Workflow

## Purpose
Guide developers through customizing the WHMCS header/navigation area.

## Prerequisites
- WHMCS installation
- Smarty template knowledge
- HTML/CSS/JS skills
- FTP or file manager access

## Steps

### Phase 1: Header Structure Analysis

1. Locate header templates
   ```
   /whmcs/templates/six/layout/
   ├── header.tpl          # Main header
   ├── header-v2.tpl       # Alternative header
   └── navigation.tpl      # Navigation component
   ```

2. Header components
   ```
   Header structure:
   ├── Top bar (optional)
   │   ├── Contact info
   │   ├── Language selector
   │   └── Login/Account links
   ├── Main header
   │   ├── Logo
   │   ├── Search (optional)
   │   ├── Navigation menu
   │   └── Mobile menu toggle
   └── Announcement bar (optional)
   ```

3. Hook points for header
   ```
   WHMCS header hooks:
   - ClientAreaPageHead
   - ClientAreaHeader
   - ClientAreaNavBar
   ```

### Phase 2: Basic Header Modifications

1. Modify logo container
   ```smarty
   {* In header.tpl *}
   <a href="{$WEB_ROOT}/" class="navbar-brand">
       {if $companyLogoUrl}
           <img src="{$companyLogoUrl}" alt="{$companyname}" class="logo">
       {else}
           <span class="logo-text">{$companyname}</span>
       {/if}
   </a>
   ```

2. Add search bar
   ```smarty
   <form class="header-search" action="{$WEB_ROOT}/search.php" method="get">
       <input type="text" name="q" placeholder="Search..." class="form-control">
       <button type="submit" class="btn btn-link">
           <i class="fa fa-search"></i>
       </button>
   </form>
   ```

3. Add contact info
   ```smarty
   <div class="header-contact">
       <a href="tel:{$companyPhone}">
           <i class="fa fa-phone"></i>
           {$companyPhone}
       </a>
   </div>
   ```

### Phase 3: Header CSS Styling

1. Main header styles
   ```css
   .site-header {
       background: #fff;
       box-shadow: 0 2px 10px rgba(0,0,0,0.08);
       position: sticky;
       top: 0;
       z-index: 1000;
   }
   
   .navbar {
       padding: 15px 0;
   }
   
   .navbar-brand img {
       max-height: 50px;
       width: auto;
   }
   ```

2. Navigation styles
   ```css
   .header-nav .nav-link {
       padding: 10px 15px;
       color: #333;
       font-weight: 500;
       transition: color 0.3s;
   }
   
   .header-nav .nav-link:hover,
   .header-nav .nav-link.active {
       color: #007bff;
   }
   
   .header-nav .dropdown-menu {
       border: none;
       box-shadow: 0 4px 20px rgba(0,0,0,0.1);
       border-radius: 8px;
   }
   ```

3. Top bar styles
   ```css
   .header-top {
       background: #1a1a2e;
       color: #fff;
       padding: 8px 0;
       font-size: 13px;
   }
   
   .header-top a {
       color: rgba(255,255,255,0.8);
   }
   
   .header-top a:hover {
       color: #fff;
   }
   ```

### Phase 4: Header Customization Hooks

1. Hook-based header modification
   ```php
   // hooks/header_customization.php
   <?php
   add_hook('ClientAreaHeader', 1, function($vars) {
       return '<div class="custom-header-element"></div>';
   });
   ```

2. Add custom header content
   ```php
   add_hook('ClientAreaPageHead', 1, function($vars) {
       echo '<style>
           .custom-header-banner {
               background: linear-gradient(135deg, #667eea, #764ba2);
               color: #fff;
               padding: 10px;
               text-align: center;
           }
       </style>';
   });
   ```

3. Conditional header content
   ```php
   add_hook('ClientAreaHeader', 1, function($vars) {
       $output = '';
       
       if (!defined('CLIENTAREA')) {
           $output = '<div class="promo-banner">Special Offer!</div>';
       }
       
       return $output;
   });
   ```

### Phase 5: Mobile Header

1. Mobile hamburger menu
   ```css
   .header-toggler {
       display: none;
   }
   
   @media (max-width: 991px) {
       .header-toggler {
           display: flex;
           flex-direction: column;
           justify-content: center;
           align-items: center;
           width: 44px;
           height: 44px;
           padding: 8px;
           background: transparent;
           border: none;
           cursor: pointer;
       }
       
       .header-toggler span {
           display: block;
           width: 24px;
           height: 2px;
           background: #333;
           margin: 3px 0;
           transition: transform 0.3s;
       }
       
       .header-toggler.active span:nth-child(1) {
           transform: rotate(45deg) translate(5px, 5px);
       }
       
       .header-toggler.active span:nth-child(2) {
           opacity: 0;
       }
       
       .header-toggler.active span:nth-child(3) {
           transform: rotate(-45deg) translate(5px, -5px);
       }
   }
   ```

2. Mobile menu dropdown
   ```javascript
   (function($) {
       'use strict';
       
       $(document).ready(function() {
           $('.header-toggler').on('click', function() {
               $(this).toggleClass('active');
               $('.header-nav').toggleClass('show');
           });
       });
   })(jQuery);
   ```

### Phase 6: Mega Menu Implementation

1. Mega menu structure
   ```smarty
   <li class="nav-item dropdown mega-menu">
       <a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown">
           Products
       </a>
       <div class="dropdown-menu">
           <div class="container">
               <div class="row">
                   <div class="col-md-4">
                       <h5 class="dropdown-header">Hosting</h5>
                       <a class="dropdown-item" href="#">Shared Hosting</a>
                       <a class="dropdown-item" href="#">VPS Hosting</a>
                       <a class="dropdown-item" href="#">Dedicated Servers</a>
                   </div>
                   <div class="col-md-4">
                       <h5 class="dropdown-header">Domains</h5>
                       <a class="dropdown-item" href="#">Domain Registration</a>
                       <a class="dropdown-item" href="#">Domain Transfer</a>
                   </div>
                   <div class="col-md-4">
                       <h5 class="dropdown-header">SSL</h5>
                       <a class="dropdown-item" href="#">SSL Certificates</a>
                   </div>
               </div>
           </div>
       </div>
   </li>
   ```

2. Mega menu CSS
   ```css
   .mega-menu .dropdown-menu {
       width: 100%;
       padding: 20px;
       border-radius: 8px;
   }
   
   .mega-menu .dropdown-header {
       font-weight: 600;
       color: #333;
       padding: 5px 15px;
       margin-bottom: 10px;
   }
   
   .mega-menu .dropdown-item {
       padding: 8px 15px;
       white-space: normal;
   }
   
   @media (max-width: 991px) {
       .mega-menu .dropdown-menu {
           position: static;
           background: #f8f9fa;
           border: none;
           box-shadow: none;
       }
   }
   ```

### Phase 7: Transparent Header

1. For landing pages
   ```css
   .header-transparent {
       position: absolute;
       width: 100%;
       background: transparent;
       box-shadow: none;
   }
   
   .header-transparent .navbar-brand,
   .header-transparent .nav-link {
       color: #fff;
   }
   
   .header-transparent .header-contact a {
       color: rgba(255,255,255,0.9);
   }
   ```

2. Transparent header with scroll
   ```javascript
   (function($) {
       'use strict';
       
       $(window).on('scroll', function() {
           var scrollPos = $(window).scrollTop();
           
           if (scrollPos > 50) {
               $('.header-transparent')
                   .addClass('header-solid')
                   .removeClass('header-transparent');
           } else {
               $('.header-solid')
                   .addClass('header-transparent')
                   .removeClass('header-solid');
           }
       });
   })(jQuery);
   ```

### Phase 8: Header Enhancement

1. Add shopping cart indicator
   ```smarty
   <div class="header-cart">
       <a href="{$WEB_ROOT}/cart.php" class="cart-link">
           <i class="fa fa-shopping-cart"></i>
           <span class="cart-count" id="cartItemCount">{$cartitemcount|default:'0'}</span>
       </a>
   </div>
   ```

2. Add user account menu
   ```smarty
   {if $loggedin}
       <div class="header-account dropdown">
           <a href="#" class="dropdown-toggle" data-toggle="dropdown">
               <i class="fa fa-user"></i>
               {$client->firstname}
           </a>
           <div class="dropdown-menu">
               <a class="dropdown-item" href="{$WEB_ROOT}/clientarea.php">Dashboard</a>
               <a class="dropdown-item" href="{$WEB_ROOT}/clientarea.php?action=details">Profile</a>
               <a class="dropdown-item" href="{$WEB_ROOT}/logout.php">Logout</a>
           </div>
       </div>
   {else}
       <a href="{$WEB_ROOT}/login.php" class="btn btn-sm btn-outline-primary">
           Login
       </a>
   {/if}
   ```

3. Add language selector
   ```smarty
   {if $languages}
       <div class="header-language dropdown">
           <a href="#" class="dropdown-toggle" data-toggle="dropdown">
               <i class="fa fa-globe"></i>
               {$currentLanguage}
           </a>
           <div class="dropdown-menu dropdown-menu-right">
               {foreach $languages as $lang}
                   <a class="dropdown-item" href="?lang={$lang.id}">
                       {$lang.localizedName}
                   </a>
               {/foreach}
           </div>
       </div>
   {/if}
   ```

## Testing Checklist
- [ ] Logo displays correctly
- [ ] Navigation links work
- [ ] Mobile menu functions
- [ ] Dropdowns work
- [ ] Cart icon updates
- [ ] User menu shows/hides
- [ ] Language selector works
- [ ] No horizontal scroll

## Related Workflows
- whmcs-theme-customization
- whmcs-navigation-tuning
- whmcs-sidebar-modification
- whmcs-logo-branding