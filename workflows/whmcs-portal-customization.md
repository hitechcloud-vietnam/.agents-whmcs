# WHMCS Portal Customization Workflow

## Purpose
Guide developers through customizing the WHMCS client portal.

## Prerequisites
- WHMCS installation
- Smarty template knowledge
- CSS/JavaScript skills
- Client area understanding

## Steps

### Phase 1: Portal Structure Analysis

1. Client area pages
   ```
   /whmcs/templates/six/
   ├── clienthome.tpl          # Dashboard
   ├── clientareaproducts.tpl  # Services list
   ├── clientareadomains.tpl   # Domains list
   ├── clientareabilling.tpl   # Invoices
   ├── clientareaaffiliates.tpl # Affiliates
   └── supporttickets.tpl      # Tickets
   ```

2. Portal sections
   ```
   Client Portal Sections:
   ├── Dashboard (home)
   ├── Services
   ├── Domains
   ├── Billing
   │   ├── Invoices
   │   ├── Quotes
   │   └── Payment Methods
   ├── Support
   │   ├── Tickets
   │   ├── Knowledge Base
   │   └── Downloads
   └── Account Settings
   ```

### Phase 2: Dashboard Customization

1. Dashboard widget structure
   ```smarty
   {* Custom dashboard *}
   <div class="dashboard container">
       <div class="dashboard-header">
           <h1>Welcome back, {$client->firstname}!</h1>
           <p>Here's an overview of your account</p>
       </div>
       
       <div class="dashboard-grid">
           <div class="dashboard-card">
               <div class="card-icon">
                   <i class="fa fa-server"></i>
               </div>
               <div class="card-content">
                   <h3>{$active_services}</h3>
                   <p>Active Services</p>
               </div>
           </div>
           
           <div class="dashboard-card">
               <div class="card-icon">
                   <i class="fa fa-globe"></i>
               </div>
               <div class="card-content">
                   <h3>{$total_domains}</h3>
                   <p>Domains</p>
               </div>
           </div>
           
           <div class="dashboard-card">
               <div class="card-icon">
                   <i class="fa fa-file-invoice-dollar"></i>
               </div>
               <div class="card-content">
                   <h3>{$open_invoices}</h3>
                   <p>Open Invoices</p>
               </div>
           </div>
           
           <div class="dashboard-card">
               <div class="card-icon">
                   <i class="fa fa-ticket-alt"></i>
               </div>
               <div class="card-content">
                   <h3>{$open_tickets}</h3>
                   <p>Open Tickets</p>
               </div>
           </div>
       </div>
   </div>
   ```

2. Dashboard CSS
   ```css
   .dashboard {
       padding: 30px 0;
   }
   
   .dashboard-header {
       margin-bottom: 30px;
   }
   
   .dashboard-header h1 {
       font-size: 28px;
       font-weight: 600;
       margin-bottom: 5px;
   }
   
   .dashboard-grid {
       display: grid;
       grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
       gap: 20px;
   }
   
   .dashboard-card {
       background: #fff;
       border-radius: 12px;
       padding: 25px;
       display: flex;
       align-items: center;
       box-shadow: 0 2px 15px rgba(0,0,0,0.05);
       transition: transform 0.2s;
   }
   
   .dashboard-card:hover {
       transform: translateY(-3px);
   }
   
   .dashboard-card .card-icon {
       width: 60px;
       height: 60px;
       border-radius: 12px;
       background: linear-gradient(135deg, #667eea, #764ba2);
       color: #fff;
       display: flex;
       align-items: center;
       justify-content: center;
       font-size: 24px;
       margin-right: 20px;
   }
   
   .dashboard-card .card-content h3 {
       font-size: 28px;
       font-weight: 700;
       margin: 0;
   }
   
   .dashboard-card .card-content p {
       color: #6c757d;
       margin: 5px 0 0;
   }
   ```

### Phase 3: Services Page Customization

1. Services list styling
   ```smarty
   <div class="services-list">
       {foreach $services as $service}
           <div class="service-card">
               <div class="service-header">
                   <h3>{$service->productName}</h3>
                   <span class="status-badge {$service->status}">
                       {$service->status}
                   </span>
               </div>
               <div class="service-details">
                   <p class="domain">{$service->domain}</p>
                   <p class="billing-cycle">
                       Next Due: {$service->nextDueDate}
                   </p>
               </div>
               <div class="service-actions">
                   <a href="{$WEB_ROOT}/clientarea.php?action=productdetails&id={$service->id}" 
                      class="btn btn-sm btn-outline-primary">
                       View Details
                   </a>
               </div>
           </div>
       {/foreach}
   </div>
   ```

2. Service card CSS
   ```css
   .service-card {
       background: #fff;
       border-radius: 10px;
       padding: 20px;
       margin-bottom: 15px;
       border-left: 4px solid #667eea;
       box-shadow: 0 2px 10px rgba(0,0,0,0.05);
   }
   
   .service-header {
       display: flex;
       justify-content: space-between;
       align-items: flex-start;
       margin-bottom: 15px;
   }
   
   .service-header h3 {
       font-size: 18px;
       font-weight: 600;
       margin: 0;
   }
   
   .status-badge {
       padding: 4px 12px;
       border-radius: 20px;
       font-size: 12px;
       font-weight: 600;
       text-transform: uppercase;
   }
   
   .status-badge.active {
       background: #d4edda;
       color: #155724;
   }
   
   .status-badge.pending {
       background: #fff3cd;
       color: #856404;
   }
   
   .status-badge.suspended {
       background: #f8d7da;
       color: #721c24;
   }
   ```

### Phase 4: Ticket System Styling

1. Ticket list styling
   ```smarty
   <div class="tickets-list">
       {foreach $tickets as $ticket}
           <a href="{$WEB_ROOT}/supporttickets.php?action=view&id={$ticket->id}" 
              class="ticket-item">
               <div class="ticket-status {$ticket->status}">
                   <i class="fa fa-circle"></i>
               </div>
               <div class="ticket-content">
                   <h4>{$ticket->subject}</h4>
                   <p class="ticket-meta">
                       Ticket #{$ticket->id} | 
                       {$ticket->created_date}
                   </p>
               </div>
               <div class="ticket-priority {$ticket->priority}">
                   {$ticket->priority}
               </div>
           </a>
       {/foreach}
   </div>
   ```

2. Ticket CSS
   ```css
   .ticket-item {
       display: flex;
       align-items: center;
       padding: 20px;
       background: #fff;
       border-radius: 8px;
       margin-bottom: 10px;
       text-decoration: none;
       color: inherit;
       transition: all 0.2s;
       box-shadow: 0 2px 8px rgba(0,0,0,0.05);
   }
   
   .ticket-item:hover {
       background: #f8f9fa;
       transform: translateX(5px);
   }
   
   .ticket-status {
       width: 12px;
       height: 12px;
       border-radius: 50%;
       margin-right: 15px;
   }
   
   .ticket-status.open {
       background: #007bff;
   }
   
   .ticket-status.answered {
       background: #28a745;
   }
   
   .ticket-content {
       flex: 1;
   }
   
   .ticket-content h4 {
       font-size: 16px;
       font-weight: 500;
       margin: 0 0 5px;
   }
   
   .ticket-meta {
       font-size: 13px;
       color: #6c757d;
       margin: 0;
   }
   
   .ticket-priority {
       padding: 4px 12px;
       border-radius: 4px;
       font-size: 11px;
       font-weight: 600;
       text-transform: uppercase;
   }
   
   .ticket-priority.high {
       background: #f8d7da;
       color: #721c24;
   }
   
   .ticket-priority.medium {
       background: #fff3cd;
       color: #856404;
   }
   
   .ticket-priority.low {
       background: #d1ecf1;
       color: #0c5460;
   }
   ```

### Phase 5: Navigation Customization

1. Portal navigation
   ```smarty
   <nav class="portal-nav">
       <ul class="nav-list">
           <li class="{if $active eq 'home'}active{/if}">
               <a href="{$WEB_ROOT}/clientarea.php">
                   <i class="fa fa-home"></i>
                   <span>Dashboard</span>
               </a>
           </li>
           <li class="{if $active eq 'services'}active{/if}">
               <a href="{$WEB_ROOT}/clientarea.php?action=services">
                   <i class="fa fa-server"></i>
                   <span>Services</span>
               </a>
           </li>
           <li class="{if $active eq 'domains'}active{/if}">
               <a href="{$WEB_ROOT}/clientarea.php?action=domains">
                   <i class="fa fa-globe"></i>
                   <span>Domains</span>
               </a>
           </li>
           <li class="{if $active eq 'billing'}active{/if}">
               <a href="{$WEB_ROOT}/clientarea.php?action=billing">
                   <i class="fa fa-file-invoice-dollar"></i>
                   <span>Billing</span>
               </a>
           </li>
           <li class="{if $active eq 'tickets'}active{/if}">
               <a href="{$WEB_ROOT}/supporttickets.php">
                   <i class="fa fa-ticket-alt"></i>
                   <span>Support</span>
               </a>
           </li>
       </ul>
   </nav>
   ```

2. Portal nav CSS
   ```css
   .portal-nav {
       background: #fff;
       border-radius: 12px;
       padding: 20px;
       box-shadow: 0 2px 15px rgba(0,0,0,0.05);
   }
   
   .portal-nav .nav-list {
       list-style: none;
       padding: 0;
       margin: 0;
   }
   
   .portal-nav .nav-item {
       margin-bottom: 5px;
   }
   
   .portal-nav .nav-link {
       display: flex;
       align-items: center;
       padding: 12px 15px;
       color: #495057;
       border-radius: 8px;
       transition: all 0.2s;
   }
   
   .portal-nav .nav-link i {
       width: 24px;
       font-size: 18px;
       margin-right: 12px;
   }
   
   .portal-nav .nav-link:hover {
       background: #f8f9fa;
       color: #667eea;
   }
   
   .portal-nav .nav-item.active .nav-link {
       background: linear-gradient(135deg, #667eea, #764ba2);
       color: #fff;
   }
   ```

### Phase 6: Quick Actions

1. Quick action buttons
   ```smarty
   <div class="quick-actions">
       <a href="{$WEB_ROOT}/cart.php" class="action-btn">
           <i class="fa fa-shopping-cart"></i>
           <span>Order New</span>
       </a>
       <a href="{$WEB_ROOT}/supporttickets.php?action=open" class="action-btn">
           <i class="fa fa-plus"></i>
           <span>New Ticket</span>
       </a>
       <a href="{$WEB_ROOT}/clientarea.php?action=details" class="action-btn">
           <i class="fa fa-user-cog"></i>
           <span>Settings</span>
       </a>
       <a href="{$WEB_ROOT}/logout.php" class="action-btn">
           <i class="fa fa-sign-out-alt"></i>
           <span>Logout</span>
       </a>
   </div>
   ```

2. Quick action CSS
   ```css
   .quick-actions {
       display: grid;
       grid-template-columns: repeat(4, 1fr);
       gap: 15px;
       margin: 30px 0;
   }
   
   .action-btn {
       display: flex;
       flex-direction: column;
       align-items: center;
       padding: 25px 15px;
       background: #fff;
       border-radius: 10px;
       text-decoration: none;
       color: #495057;
       box-shadow: 0 2px 10px rgba(0,0,0,0.05);
       transition: all 0.2s;
   }
   
   .action-btn:hover {
       background: #667eea;
       color: #fff;
       transform: translateY(-3px);
   }
   
   .action-btn i {
       font-size: 28px;
       margin-bottom: 10px;
   }
   
   .action-btn span {
       font-size: 13px;
       font-weight: 500;
   }
   
   @media (max-width: 767px) {
       .quick-actions {
           grid-template-columns: repeat(2, 1fr);
       }
   }
   ```

### Phase 7: Portal Hooks

1. Dashboard widget hook
   ```php
   // hooks/portal_widgets.php
   <?php
   add_hook('ClientAreaHomepagePanels', 1, function($vars) {
       return [
           'name' => 'Custom Widget',
           'template' => 'widgets/custom-widget',
           'order' => 100,
       ];
   });
   ```

## Testing Checklist
- [ ] All portal pages render correctly
- [ ] Navigation is functional
- [ ] Quick actions work
- [ ] Widgets display properly
- [ ] Mobile responsive

## Related Workflows
- whmcs-theme-customization
- whmcs-sidebar-modification
- whmcs-navigation-tuning
- whmcs-card-design