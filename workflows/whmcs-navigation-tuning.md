# WHMCS Navigation Tuning Workflow

## Purpose
Guide developers through customizing WHMCS navigation menus.

## Prerequisites
- WHMCS installation
- HTML/CSS/JS knowledge
- Smarty template experience
- Understanding of navigation structure

## Steps

### Phase 1: Navigation Structure

1. Navigation locations
   ```
   WHMCS Navigation:
   ├── Header navigation (top bar)
   ├── Main navigation (navbar)
   ├── Client area navigation (sidebar)
   ├── Footer navigation
   └── Mobile navigation (hamburger)
   ```

2. Navigation templates
   ```
   /whmcs/templates/six/layout/
   ├── navigation.tpl
   ├── header-nav.tpl
   └── client-nav.tpl
   ```

3. Navigation hooks
   ```
   WHMCS Nav Hooks:
   - ClientAreaPrimaryNavbar
   - ClientAreaSecondaryNavbar
   - ClientAreaPrimarySidebar
   - ClientAreaSecondarySidebar
   - PreClientAreaNavMenuItems
   - PostClientAreaNavMenuItems
   ```

### Phase 2: Top Bar Navigation

1. Top bar structure
   ```smarty
   <div class="header-top">
       <div class="container">
           <div class="row align-items-center">
               <div class="col-md-6">
                   <div class="top-contact">
                       <a href="tel:{$companyPhone}">
                           <i class="fa fa-phone"></i> {$companyPhone}
                       </a>
                       <a href="mailto:{$companyEmail}">
                           <i class="fa fa-envelope"></i> {$companyEmail}
                       </a>
                   </div>
               </div>
               <div class="col-md-6 text-md-right">
                   <div class="top-menu">
                       <a href="{$WEB_ROOT}/contact.php">Contact</a>
                       <a href="{$WEB_ROOT}/supporttickets.php">Support</a>
                       {if !$loggedin}
                           <a href="{$WEB_ROOT}/login.php">Login</a>
                           <a href="{$WEB_ROOT}/register.php" class="btn btn-sm btn-primary">Sign Up</a>
                       {else}
                           <a href="{$WEB_ROOT}/clientarea.php">My Account</a>
                       {/if}
                   </div>
               </div>
           </div>
       </div>
   </div>
   ```

2. Top bar CSS
   ```css
   .header-top {
       background: #1a1a2e;
       color: #fff;
       padding: 8px 0;
       font-size: 13px;
   }
   
   .top-contact a {
       color: rgba(255,255,255,0.8);
       margin-right: 20px;
   }
   
   .top-contact i {
       margin-right: 5px;
   }
   
   .top-menu a {
       color: rgba(255,255,255,0.8);
       margin-left: 15px;
   }
   
   .top-menu a:hover {
       color: #fff;
   }
   ```

### Phase 3: Main Navigation

1. Main navbar structure
   ```smarty
   <nav class="navbar navbar-expand-lg main-nav">
       <div class="container">
           <a class="navbar-brand" href="{$WEB_ROOT}/">
               <img src="{$logoUrl}" alt="{$companyname}">
           </a>
           
           <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#mainNav">
               <span class="navbar-toggler-icon"></span>
           </button>
           
           <div class="collapse navbar-collapse" id="mainNav">
               <ul class="navbar-nav ml-auto">
                   <li class="nav-item">
                       <a class="nav-link" href="{$WEB_ROOT}/">Home</a>
                   </li>
                   <li class="nav-item dropdown">
                       <a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown">
                           Services
                       </a>
                       <div class="dropdown-menu">
                           <a class="dropdown-item" href="{$WEB_ROOT}/cart.php?gid=hosting">Hosting</a>
                           <a class="dropdown-item" href="{$WEB_ROOT}/cart.php?gid=vps">VPS</a>
                           <a class="dropdown-item" href="{$WEB_ROOT}/cart.php?gid=servers">Servers</a>
                       </div>
                   </li>
                   <li class="nav-item dropdown">
                       <a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown">
                           Domains
                       </a>
                       <div class="dropdown-menu">
                           <a class="dropdown-item" href="{$WEB_ROOT}/cart.php?a=add&domain=register">Register</a>
                           <a class="dropdown-item" href="{$WEB_ROOT}/cart.php?a=add&domain=transfer">Transfer</a>
                       </div>
                   </li>
                   <li class="nav-item">
                       <a class="nav-link" href="{$WEB_ROOT}/supporttickets.php">Support</a>
                   </li>
                   <li class="nav-item">
                       <a class="nav-link" href="{$WEB_ROOT}/knowledgebase.php">KB</a>
                   </li>
                   <li class="nav-item">
                       <a class="nav-link btn btn-primary btn-sm" href="{$WEB_ROOT}/cart.php">Order Now</a>
                   </li>
               </ul>
           </div>
       </div>
   </nav>
   ```

2. Navigation CSS
   ```css
   .main-nav {
       padding: 15px 0;
       background: #fff;
       box-shadow: 0 2px 10px rgba(0,0,0,0.05);
   }
   
   .main-nav .nav-item {
       margin: 0 5px;
   }
   
   .main-nav .nav-link {
       padding: 10px 15px;
       color: #333;
       font-weight: 500;
       transition: color 0.3s;
   }
   
   .main-nav .nav-link:hover {
       color: #007bff;
   }
   
   .main-nav .nav-link.btn {
       color: #fff;
       padding: 8px 20px;
       border-radius: 4px;
   }
   
   .main-nav .dropdown-menu {
       border: none;
       box-shadow: 0 4px 20px rgba(0,0,0,0.1);
       border-radius: 8px;
       padding: 10px;
   }
   
   .main-nav .dropdown-item {
       padding: 10px 15px;
       border-radius: 4px;
   }
   
   .main-nav .dropdown-item:hover {
       background: #f8f9fa;
   }
   ```

### Phase 4: Client Area Navigation

1. Client sidebar nav
   ```smarty
   <nav class="client-nav">
       <ul class="nav flex-column">
           <li class="nav-header">Main</li>
           <li class="nav-item {if $activePage eq 'home'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php">
                   <i class="fa fa-home"></i>
                   <span>Dashboard</span>
               </a>
           </li>
           <li class="nav-item {if $activePage eq 'services'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=services">
                   <i class="fa fa-server"></i>
                   <span>My Services</span>
               </a>
           </li>
           
           <li class="nav-header">Billing</li>
           <li class="nav-item {if $activePage eq 'invoices'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=invoices">
                   <i class="fa fa-file-invoice-dollar"></i>
                   <span>Invoices</span>
               </a>
           </li>
           <li class="nav-item {if $activePage eq 'payments'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=paymentmethods">
                   <i class="fa fa-credit-card"></i>
                   <span>Payment Methods</span>
               </a>
           </li>
           
           <li class="nav-header">Support</li>
           <li class="nav-item {if $activePage eq 'tickets'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/supporttickets.php">
                   <i class="fa fa-ticket-alt"></i>
                   <span>Tickets</span>
               </a>
           </li>
           <li class="nav-item {if $activePage eq 'kb'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/knowledgebase.php">
                   <i class="fa fa-book"></i>
                   <span>Knowledge Base</span>
               </a>
           </li>
           
           <li class="nav-header">Account</li>
           <li class="nav-item {if $activePage eq 'details'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=details">
                   <i class="fa fa-user-cog"></i>
                   <span>Settings</span>
               </a>
           </li>
           <li class="nav-item">
               <a class="nav-link" href="{$WEB_ROOT}/logout.php">
                   <i class="fa fa-sign-out-alt"></i>
                   <span>Logout</span>
               </a>
           </li>
       </ul>
   </nav>
   ```

2. Client nav CSS
   ```css
   .client-nav {
       background: #fff;
       border-radius: 12px;
       padding: 20px;
       box-shadow: 0 2px 15px rgba(0,0,0,0.05);
   }
   
   .client-nav .nav-header {
       font-size: 11px;
       font-weight: 600;
       text-transform: uppercase;
       color: #adb5bd;
       letter-spacing: 1px;
       padding: 15px 15px 8px;
       margin-top: 10px;
   }
   
   .client-nav .nav-header:first-child {
       padding-top: 0;
       margin-top: 0;
   }
   
   .client-nav .nav-item {
       margin-bottom: 3px;
   }
   
   .client-nav .nav-link {
       display: flex;
       align-items: center;
       padding: 12px 15px;
       color: #495057;
       border-radius: 8px;
       transition: all 0.2s;
   }
   
   .client-nav .nav-link i {
       width: 20px;
       margin-right: 12px;
       font-size: 16px;
   }
   
   .client-nav .nav-link:hover {
       background: #f8f9fa;
       color: #007bff;
   }
   
   .client-nav .nav-item.active .nav-link {
       background: linear-gradient(135deg, #667eea, #764ba2);
       color: #fff;
   }
   ```

### Phase 5: Mega Menu

1. Mega menu structure
   ```smarty
   <li class="nav-item dropdown mega-menu">
       <a class="nav-link dropdown-toggle" href="#" data-toggle="dropdown">
           Products
       </a>
       <div class="dropdown-menu mega-menu-dropdown">
           <div class="container">
               <div class="row">
                   <div class="col-md-3">
                       <h6 class="mega-menu-title">Hosting</h6>
                       <a class="dropdown-item" href="#">Shared Hosting</a>
                       <a class="dropdown-item" href="#">WordPress Hosting</a>
                       <a class="dropdown-item" href="#">VPS Hosting</a>
                       <a class="dropdown-item" href="#">Dedicated Servers</a>
                   </div>
                   <div class="col-md-3">
                       <h6 class="mega-menu-title">Domains</h6>
                       <a class="dropdown-item" href="#">Domain Registration</a>
                       <a class="dropdown-item" href="#">Domain Transfer</a>
                       <a class="dropdown-item" href="#">WHOIS Lookup</a>
                   </div>
                   <div class="col-md-3">
                       <h6 class="mega-menu-title">Security</h6>
                       <a class="dropdown-item" href="#">SSL Certificates</a>
                       <a class="dropdown-item" href="#">Site Lock</a>
                       <a class="dropdown-item" href="#">Code Signing</a>
                   </div>
                   <div class="col-md-3">
                       <div class="mega-menu-promo">
                           <img src="/img/promo.jpg" alt="Promotion">
                           <p>Up to 50% off on VPS plans!</p>
                       </div>
                   </div>
               </div>
           </div>
       </div>
   </li>
   ```

2. Mega menu CSS
   ```css
   .mega-menu .dropdown-menu {
       width: 700px;
       padding: 25px;
       border-radius: 12px;
   }
   
   .mega-menu-title {
       font-size: 14px;
       font-weight: 600;
       color: #212529;
       margin-bottom: 15px;
       padding: 0 15px;
   }
   
   .mega-menu .dropdown-item {
       padding: 10px 15px;
   }
   
   .mega-menu-promo {
       background: linear-gradient(135deg, #667eea, #764ba2);
       border-radius: 8px;
       padding: 20px;
       color: #fff;
       text-align: center;
   }
   
   .mega-menu-promo img {
       border-radius: 4px;
       margin-bottom: 10px;
   }
   
   @media (max-width: 991px) {
       .mega-menu .dropdown-menu {
           width: 100%;
       }
   }
   ```

### Phase 6: Navigation Hooks

1. Add menu item via hook
   ```php
   // hooks/add_nav_item.php
   <?php
   add_hook('ClientAreaPrimaryNavbar', 1, function($vars) {
       return [
           'label' => 'Custom Page',
           'uri' => '/custom.php',
           'order' => 50,
           'children' => [
               [
                   'label' => 'Sub Item 1',
                   'uri' => '/custom1.php'
               ],
               [
                   'label' => 'Sub Item 2',
                   'uri' => '/custom2.php'
               ]
           ]
       ];
   });
   ```

2. Conditional menu items
   ```php
   add_hook('ClientAreaPrimaryNavbar', 1, function($vars) {
       $items = [];
       
       if (is_null($vars['userid'])) {
           $items[] = [
               'label' => 'Register',
               'uri' => '/register.php',
               'order' => 100
           ];
       } else {
           $items[] = [
               'label' => 'Dashboard',
               'uri' => '/clientarea.php',
               'order' => 100
           ];
       }
       
       return $items;
   });
   ```

### Phase 7: Mobile Navigation

1. Mobile menu
   ```css
   @media (max-width: 991px) {
       .navbar-collapse {
           position: fixed;
           top: 0;
           left: -100%;
           width: 280px;
           height: 100vh;
           background: #fff;
           box-shadow: 2px 0 15px rgba(0,0,0,0.1);
           transition: left 0.3s ease;
           padding: 80px 20px 20px;
           overflow-y: auto;
       }
       
       .navbar-collapse.show {
           left: 0;
       }
       
       .navbar-collapse .nav-item {
           margin: 0;
       }
       
       .navbar-collapse .nav-link {
           padding: 15px;
           border-bottom: 1px solid #e9ecef;
       }
       
       .navbar-toggler.active {
           position: fixed;
           left: 290px;
           z-index: 1001;
       }
   }
   ```

## Testing Checklist
- [ ] All links functional
- [ ] Dropdowns work on hover and click
- [ ] Mobile menu opens/closes
- [ ] Active states show correctly
- [ ] Mega menu displays properly

## Related Workflows
- whmcs-header-customization
- whmcs-sidebar-modification
- whmcs-theme-customization
- whmcs-breadcrumb-styling