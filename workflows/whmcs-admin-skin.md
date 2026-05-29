# WHMCS Admin Skin Workflow

## Purpose
Guide developers through customizing the WHMCS admin area appearance.

## Prerequisites
- WHMCS installation
- Admin access
- CSS/HTML knowledge
- Understanding of admin template structure

## Steps

### Phase 1: Admin Template Structure

1. Admin template locations
   ```
   /whmcs/admin/templates/
   ├── default/
   │   ├── header.tpl
   │   ├── footer.tpl
   │   ├── sidebar.tpl
   │   └── styles/
   └── portal/
   ```

2. Admin skinning areas
   ```
   Admin Sections:
   ├── Login page
   ├── Dashboard
   ├── Sidebar navigation
   ├── Content tables
   ├── Forms
   └── Modals
   ```

### Phase 2: Admin CSS Customization

1. Admin CSS file
   ```
   /whmcs/admin/templates/default/styles/
   └── admin.css
   ```

2. Basic admin styles
   ```css
   /* Custom admin theme */
   
   /* Primary brand colors */
   :root {
       --admin-primary: #667eea;
       --admin-secondary: #764ba2;
       --admin-dark: #1a1a2e;
       --admin-light: #f8f9fa;
   }
   
   /* Sidebar styling */
   .sidebar {
       background: var(--admin-dark);
   }
   
   /* Navigation items */
   .sidebar-menu li a {
       color: rgba(255,255,255,0.8);
   }
   
   .sidebar-menu li a:hover,
   .sidebar-menu li.active a {
       background: var(--admin-primary);
       color: #fff;
   }
   
   /* Buttons */
   .btn-primary {
       background: linear-gradient(135deg, #667eea, #764ba2);
       border: none;
   }
   
   /* Cards */
   .card {
       border-radius: 10px;
       box-shadow: 0 2px 10px rgba(0,0,0,0.05);
   }
   ```

### Phase 3: Admin Header Customization

1. Admin header template
   ```smarty
   {* /admin/templates/default/header.tpl *}
   <nav class="navbar admin-header">
       <a class="navbar-brand" href="{$admin_folder}/">
           <img src="{$logo_url}" alt="Admin" class="admin-logo">
       </a>
       
       <div class="header-actions">
           <a href="{$WEB_ROOT}/" target="_blank" class="btn btn-sm btn-outline-secondary">
               <i class="fa fa-external-link-alt"></i>
               View Site
           </a>
           
           <div class="dropdown">
               <button class="btn btn-sm dropdown-toggle" data-toggle="dropdown">
                   <i class="fa fa-user"></i>
                   {$admin_name}
               </button>
               <div class="dropdown-menu">
                   <a class="dropdown-item" href="{$admin_folder}/user/profile.php">Profile</a>
                   <a class="dropdown-item" href="{$admin_folder}/logout.php">Logout</a>
               </div>
           </div>
       </div>
   </nav>
   ```

2. Admin header CSS
   ```css
   .admin-header {
       background: #fff;
       padding: 10px 20px;
       box-shadow: 0 2px 10px rgba(0,0,0,0.05);
       position: sticky;
       top: 0;
       z-index: 100;
   }
   
   .admin-logo {
       max-height: 35px;
   }
   
   .header-actions {
       display: flex;
       gap: 10px;
       align-items: center;
   }
   ```

### Phase 4: Admin Sidebar Customization

1. Sidebar styling
   ```css
   .admin-sidebar {
       background: #1a1a2e;
       min-height: calc(100vh - 60px);
       width: 240px;
       position: fixed;
       left: 0;
       top: 60px;
   }
   
   .admin-sidebar .sidebar-menu {
       padding: 0;
       margin: 0;
       list-style: none;
   }
   
   .admin-sidebar .menu-header {
       color: rgba(255,255,255,0.4);
       font-size: 11px;
       font-weight: 600;
       text-transform: uppercase;
       letter-spacing: 1px;
       padding: 20px 20px 10px;
   }
   
   .admin-sidebar .menu-item a {
       display: flex;
       align-items: center;
       padding: 12px 20px;
       color: rgba(255,255,255,0.7);
       text-decoration: none;
       transition: all 0.2s;
   }
   
   .admin-sidebar .menu-item a:hover {
       background: rgba(255,255,255,0.1);
       color: #fff;
   }
   
   .admin-sidebar .menu-item.active a {
       background: #667eea;
       color: #fff;
   }
   
   .admin-sidebar .menu-item a i {
       width: 20px;
       margin-right: 12px;
   }
   ```

### Phase 5: Admin Tables

1. Admin table styles
   ```css
   .admin-table {
       width: 100%;
       border-collapse: collapse;
       background: #fff;
       border-radius: 10px;
       overflow: hidden;
   }
   
   .admin-table thead {
       background: #f8f9fa;
   }
   
   .admin-table th {
       padding: 15px;
       text-align: left;
       font-size: 12px;
       font-weight: 600;
       text-transform: uppercase;
       color: #495057;
       border-bottom: 2px solid #dee2e6;
   }
   
   .admin-table td {
       padding: 15px;
       border-bottom: 1px solid #f0f0f0;
       vertical-align: middle;
   }
   
   .admin-table tbody tr:hover {
       background: #f8f9fa;
   }
   
   .admin-table .status-badge {
       padding: 4px 12px;
       border-radius: 15px;
       font-size: 11px;
       font-weight: 600;
   }
   ```

### Phase 6: Admin Forms

1. Admin form styling
   ```css
   .admin-form .form-group {
       margin-bottom: 20px;
   }
   
   .admin-form label {
       font-weight: 500;
       margin-bottom: 8px;
       color: #495057;
   }
   
   .admin-form .form-control {
       border-radius: 6px;
       padding: 10px 15px;
       border: 1px solid #ced4da;
   }
   
   .admin-form .form-control:focus {
       border-color: #667eea;
       box-shadow: 0 0 0 3px rgba(102,126,234,0.15);
   }
   ```

### Phase 7: Admin Cards/Panels

1. Admin dashboard cards
   ```css
   .admin-card {
       background: #fff;
       border-radius: 12px;
       padding: 20px;
       box-shadow: 0 2px 15px rgba(0,0,0,0.05);
   }
   
   .admin-card .card-header {
       display: flex;
       justify-content: space-between;
       align-items: center;
       margin-bottom: 15px;
       padding-bottom: 15px;
       border-bottom: 1px solid #f0f0f0;
   }
   
   .admin-card .card-title {
       font-size: 14px;
       font-weight: 600;
       margin: 0;
   }
   
   .admin-card .card-value {
       font-size: 32px;
       font-weight: 700;
       color: #212529;
   }
   
   .admin-card .card-footer {
       margin-top: 15px;
       padding-top: 15px;
       border-top: 1px solid #f0f0f0;
       font-size: 13px;
       color: #6c757d;
   }
   ```

### Phase 8: Admin Modals

1. Admin modal styling
   ```css
   .admin-modal .modal-content {
       border: none;
       border-radius: 12px;
       box-shadow: 0 10px 40px rgba(0,0,0,0.2);
   }
   
   .admin-modal .modal-header {
       padding: 20px 25px;
       border-bottom: 1px solid #e9ecef;
       background: #f8f9fa;
       border-radius: 12px 12px 0 0;
   }
   
   .admin-modal .modal-title {
       font-weight: 600;
   }
   
   .admin-modal .modal-footer {
       padding: 20px 25px;
       border-top: 1px solid #e9ecef;
       background: #f8f9fa;
   }
   ```

### Phase 9: Admin Theme Hook

1. Custom admin CSS via hook
   ```php
   // hooks/admin_custom_css.php
   <?php
   add_hook('AdminAreaHeadOutput', 1, function($vars) {
       return '<link rel="stylesheet" href="/templates/admin-custom.css">';
   });
   ```

## Related Workflows
- whmcs-theme-customization
- whmcs-css-customization
- whmcs-sidebar-modification
- whmcs-form-styling