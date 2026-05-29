# WHMCS Sidebar Modification Workflow

## Purpose
Guide developers through customizing the WHMCS sidebar component.

## Prerequisites
- WHMCS installation
- HTML/CSS knowledge
- Smarty template understanding
- JavaScript basics

## Steps

### Phase 1: Sidebar Structure Analysis

1. Locate sidebar templates
   ```
   /whmcs/templates/six/layout/
   ├── sidebar.tpl
   └── sidebar-nav.tpl
   ```

2. Sidebar components
   ```
   Sidebar structure:
   ├── Widget containers
   │   ├── Quick links widget
   │   ├── Services widget
   │   ├── Status widget
   │   └── Announcements widget
   ├── Navigation
   │   ├── Main menu
   │   └── Sub-menu items
   └── Collapsible sections
   ```

### Phase 2: Sidebar Layout

1. Basic sidebar structure
   ```smarty
   <aside class="sidebar" id="sidebar">
       <div class="sidebar-section">
           <h5 class="sidebar-title">Quick Links</h5>
           <ul class="sidebar-menu">
               <li><a href="{$WEB_ROOT}/clientarea.php">Dashboard</a></li>
               <li><a href="{$WEB_ROOT}/clientarea.php?action=products">Services</a></li>
               <li><a href="{$WEB_ROOT}/clientarea.php?action=invoices">Invoices</a></li>
               <li><a href="{$WEB_ROOT}/supporttickets.php">Tickets</a></li>
           </ul>
       </div>
       
       <div class="sidebar-section">
           <h5 class="sidebar-title">Support</h5>
           <ul class="sidebar-menu">
               <li><a href="{$WEB_ROOT}/knowledgebase.php">Knowledge Base</a></li>
               <li><a href="{$WEB_ROOT}/downloads.php">Downloads</a></li>
               <li><a href="{$WEB_ROOT}/networkstatus.php">Network Status</a></li>
           </ul>
       </div>
   </aside>
   ```

2. Sidebar CSS
   ```css
   .sidebar {
       width: 260px;
       background: #fff;
       border-radius: 8px;
       padding: 20px;
       box-shadow: 0 2px 10px rgba(0,0,0,0.05);
   }
   
   .sidebar-section {
       margin-bottom: 25px;
   }
   
   .sidebar-section:last-child {
       margin-bottom: 0;
   }
   
   .sidebar-title {
       font-size: 13px;
       font-weight: 600;
       text-transform: uppercase;
       color: #6c757d;
       margin-bottom: 15px;
       letter-spacing: 0.5px;
   }
   ```

### Phase 3: Navigation Sidebar

1. Client area navigation
   ```smarty
   <nav class="sidebar-nav">
       <ul class="nav flex-column">
           <li class="nav-item {if $active eq 'home'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php">
                   <i class="fa fa-home"></i> Dashboard
               </a>
           </li>
           <li class="nav-item {if $active eq 'services'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=products">
                   <i class="fa fa-server"></i> My Services
               </a>
           </li>
           <li class="nav-item {if $active eq 'domains'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=domains">
                   <i class="fa fa-globe"></i> Domains
               </a>
           </li>
           <li class="nav-item {if $active eq 'billing'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=billing">
                   <i class="fa fa-credit-card"></i> Billing
               </a>
           </li>
           <li class="nav-item {if $active eq 'tickets'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/supporttickets.php">
                   <i class="fa fa-ticket-alt"></i> Support Tickets
               </a>
           </li>
           <li class="nav-item {if $active eq 'settings'}active{/if}">
               <a class="nav-link" href="{$WEB_ROOT}/clientarea.php?action=details">
                   <i class="fa fa-cog"></i> Account Settings
               </a>
           </li>
       </ul>
   </nav>
   ```

2. Navigation CSS
   ```css
   .sidebar-nav .nav-item {
       margin-bottom: 5px;
   }
   
   .sidebar-nav .nav-link {
       display: flex;
       align-items: center;
       padding: 12px 15px;
       color: #495057;
       border-radius: 6px;
       transition: all 0.2s;
   }
   
   .sidebar-nav .nav-link i {
       margin-right: 12px;
       width: 20px;
       text-align: center;
   }
   
   .sidebar-nav .nav-link:hover {
       background: #f8f9fa;
       color: #007bff;
   }
   
   .sidebar-nav .nav-item.active .nav-link {
       background: #007bff;
       color: #fff;
   }
   ```

### Phase 4: Collapsible Sidebar

1. Accordion sidebar
   ```smarty
   <div class="sidebar-accordion">
       <div class="accordion-item">
           <button class="accordion-header" type="button">
               <i class="fa fa-home"></i> Dashboard
               <i class="fa fa-chevron-down"></i>
           </button>
           <div class="accordion-body">
               <!-- Content -->
           </div>
       </div>
       
       <div class="accordion-item active">
           <button class="accordion-header" type="button">
               <i class="fa fa-server"></i> Services
               <i class="fa fa-chevron-up"></i>
           </button>
           <div class="accordion-body show">
               <a href="#">View All</a>
               <a href="#">Upgrade</a>
           </div>
       </div>
   </div>
   ```

2. Accordion JavaScript
   ```javascript
   (function($) {
       'use strict';
       
       $('.sidebar-accordion .accordion-header').on('click', function() {
           var $item = $(this).closest('.accordion-item');
           var $body = $item.find('.accordion-body');
           
           if ($item.hasClass('active')) {
               $item.removeClass('active');
               $body.slideUp();
           } else {
               $('.accordion-item').removeClass('active');
                   $('.accordion-body').slideUp();
               
               $item.addClass('active');
               $body.slideDown();
           }
       });
   })(jQuery);
   ```

### Phase 5: Widgets

1. Service status widget
   ```smarty
   <div class="sidebar-widget service-status">
       <h5 class="widget-title">Services</h5>
       {foreach $services as $service}
           <div class="service-item">
               <div class="service-info">
                   <span class="service-name">{$service->productName}</span>
                   <span class="service-domain">{$service->domain}</span>
               </div>
               <span class="service-status-badge {$service->status}">
                   {$service->status}
               </span>
           </div>
       {/foreach}
   </div>
   ```

2. Quick links widget
   ```smarty
   <div class="sidebar-widget quick-links">
       <h5 class="widget-title">Quick Links</h5>
       <div class="link-grid">
           <a href="{$WEB_ROOT}/cart.php" class="quick-link">
               <i class="fa fa-shopping-cart"></i>
               <span>Order New</span>
           </a>
           <a href="{$WEB_ROOT}/supporttickets.php" class="quick-link">
               <i class="fa fa-plus"></i>
               <span>New Ticket</span>
           </a>
           <a href="{$WEB_ROOT}/clientarea.php?action=details" class="quick-link">
               <i class="fa fa-user"></i>
               <span>Profile</span>
           </a>
           <a href="{$WEB_ROOT}/logout.php" class="quick-link">
               <i class="fa fa-sign-out-alt"></i>
               <span>Logout</span>
           </a>
       </div>
   </div>
   ```

3. Widget CSS
   ```css
   .sidebar-widget {
       background: #f8f9fa;
       border-radius: 8px;
       padding: 15px;
       margin-bottom: 15px;
   }
   
   .widget-title {
       font-size: 14px;
       font-weight: 600;
       margin-bottom: 12px;
       color: #212529;
   }
   
   .link-grid {
       display: grid;
       grid-template-columns: repeat(2, 1fr);
       gap: 10px;
   }
   
   .quick-link {
       display: flex;
       flex-direction: column;
       align-items: center;
       padding: 15px;
       background: #fff;
       border-radius: 6px;
       color: #495057;
       transition: all 0.2s;
   }
   
   .quick-link:hover {
       background: #007bff;
       color: #fff;
   }
   
   .quick-link i {
       font-size: 24px;
       margin-bottom: 8px;
   }
   
   .quick-link span {
       font-size: 12px;
   }
   ```

### Phase 6: Responsive Sidebar

1. Mobile sidebar
   ```css
   @media (max-width: 991px) {
       .sidebar {
           position: fixed;
           left: -280px;
           top: 0;
           height: 100vh;
           z-index: 1000;
           background: #fff;
           box-shadow: 2px 0 15px rgba(0,0,0,0.1);
           transition: left 0.3s ease;
           padding-top: 60px;
       }
       
       .sidebar.active {
           left: 0;
       }
       
       .sidebar-overlay {
           position: fixed;
           top: 0;
           left: 0;
           right: 0;
           bottom: 0;
           background: rgba(0,0,0,0.5);
           z-index: 999;
           display: none;
       }
       
       .sidebar-overlay.active {
           display: block;
       }
   }
   ```

2. Sidebar toggle button
   ```javascript
   (function($) {
       'use strict';
       
       $(document).ready(function() {
           $('.sidebar-toggle').on('click', function() {
               $('.sidebar').toggleClass('active');
               $('.sidebar-overlay').toggleClass('active');
           });
           
           $('.sidebar-overlay').on('click', function() {
               $('.sidebar').removeClass('active');
               $(this).removeClass('active');
           });
       });
   })(jQuery);
   ```

### Phase 7: Sidebar Customization Hooks

1. Add content via hook
   ```php
   // hooks/sidebar_widget.php
   <?php
   add_hook('ClientAreaSidebars', 1, function($vars) {
       return [
           'action' => 'widget',
           'template' => 'widgets/custom-support',
           'vars' => [
               'supportHours' => '24/7',
               'contactMethod' => 'Tickets'
           ]
       ];
   });
   ```

## Testing Checklist
- [ ] Sidebar loads on all pages
- [ ] Links navigate correctly
- [ ] Collapsible sections work
- [ ] Mobile sidebar toggle works
- [ ] Widgets display properly

## Related Workflows
- whmcs-layout-adjustments
- whmcs-header-customization
- whmcs-navigation-tuning
- whmcs-widget-styling