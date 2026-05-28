# WHMCS Admin Menu DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-admin-menu/
├── admin-menu.php            # Main module
├── lib/
│   ├── MenuBuilder.php       # Menu builder
│   └── MenuItems.php         # Menu item definitions
└── templates/
    └── menu-config.tpl       # Admin configuration
```

## Main Admin Menu Module

```php
<?php
/**
 * WHMCS Custom Admin Menu Module
 * DevKit Template
 * 
 * Provides custom admin navigation menu items
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

use WHMCS\Database\Capsule;

/**
 * Config function
 */
function {module}_config(): array {
    return [
        'name' => '{Admin Menu}',
        'description' => 'Custom admin navigation menu items',
        'version' => '1.0',
        'author' => '{Author}',
    ];
}

/**
 * Activate
 */
function {module}_activate(): array {
    Capsule::schema()->create('mod_{module}_menu_items', function($t) {
        $t->increments('id');
        $t->string('menu_name');
        $t->string('menu_slug');
        $t->string('menu_type'); // main, sidebar, topbar
        $t->string('menu_icon'); // FontAwesome class
        $t->string('menu_url');
        $t->string('target'); // _self, _blank
        $t->integer('parent_id')->default(0);
        $t->integer('sort_order')->default(0);
        $t->text('permissions'); // JSON array of required permissions
        $t->string('badge'); // Badge text/count
        $t->boolean('is_active');
        $t->string('module'); // Associated module
        $t->timestamp('created_at');
    });
    
    Capsule::schema()->create('mod_{module}_menu_groups', function($t) {
        $t->increments('id');
        $t->string('group_name');
        $t->string('group_slug')->unique();
        $t->string('icon');
        $t->integer('sort_order')->default(0);
        $t->boolean('is_active');
        $t->timestamp('created_at');
    });
    
    // Register default menu groups
    {module}_registerDefaultMenuGroups();
    
    return ['status' => 'success', 'description' => 'Admin Menu activated'];
}

/**
 * Deactivate
 */
function {module}_deactivate(): array {
    Capsule::schema()->dropIfExists('mod_{module}_menu_items');
    Capsule::schema()->dropIfExists('mod_{module}_menu_groups');
    
    return ['status' => 'success'];
}

/**
 * Register default menu groups
 */
function {module}_registerDefaultMenuGroups(): void {
    $groups = [
        ['name' => 'Custom Tools', 'slug' => 'custom-tools', 'icon' => 'fa-tools', 'sort' => 90],
        ['name' => 'Reports', 'slug' => 'reports', 'icon' => 'fa-chart-bar', 'sort' => 80],
        ['name' => 'Integrations', 'slug' => 'integrations', 'icon' => 'fa-plug', 'sort' => 70],
    ];
    
    foreach ($groups as $group) {
        Capsule::table('mod_{module}_menu_groups')->insert([
            'group_name' => $group['name'],
            'group_slug' => $group['slug'],
            'icon' => $group['icon'],
            'sort_order' => $group['sort'],
            'is_active' => 1,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
}

/**
 * Output function (Admin Interface)
 */
function {module}_output(array $vars): void {
    $action = $_REQUEST['action'] ?? 'dashboard';
    
    if ($_SERVER['REQUEST_METHOD'] === 'POST') {
        check_token('WHMCS.admin.default');
    }
    
    switch ($action) {
        case 'groups':
            {module}_manageGroups();
            break;
        case 'items':
            {module}_manageItems();
            break;
        case 'settings':
            {module}_showSettings();
            break;
        default:
            {module}_showDashboard();
    }
}

/**
 * Menu Item Registration Hook
 */
function {module}_registerMenu(): array {
    $items = Capsule::table('mod_{module}_menu_items')
        ->where('is_active', 1)
        ->orderBy('sort_order')
        ->get();
    
    $menuItems = [];
    
    foreach ($items as $item) {
        // Check permissions
        $permissions = json_decode($item->permissions, true) ?? [];
        
        if (!empty($permissions) && !{module}_hasPermission($permissions)) {
            continue;
        }
        
        $menuItems[] = [
            'name' => $item->menu_name,
            'label' => $item->menu_name,
            'uri' => $item->menu_url,
            'icon' => $item->menu_icon,
            'order' => $item->sort_order,
            'badge' => $item->badge,
        ];
    }
    
    return $menuItems;
}
add_hook('AdminAreaNavItems', 1, function($vars) {
    return {module}_registerMenu();
});
```

## Menu Builder

```php
<?php
/**
 * Menu Builder
 * Builds and manages admin menu structure
 */

namespace AdminMenu;

use WHMCS\Database\Capsule;

class MenuBuilder {
    
    private array $menu = [];
    private string $menuType;
    
    /**
     * Build menu from database
     */
    public function build(string $menuType = 'main'): array {
        $this->menuType = $menuType;
        
        $groups = Capsule::table('mod_{module}_menu_groups')
            ->where('is_active', 1)
            ->where(function($query) use ($menuType) {
                $query->where('group_slug', 'LIKE', '%' . $menuType . '%')
                      ->orWhere('group_slug', '=', 'all');
            })
            ->orderBy('sort_order')
            ->get();
        
        $menu = [];
        
        foreach ($groups as $group) {
            $items = Capsule::table('mod_{module}_menu_items')
                ->where('menu_type', $menuType)
                ->where('group_slug', $group->group_slug)
                ->where('is_active', 1)
                ->orderBy('sort_order')
                ->get();
            
            $groupItems = [];
            
            foreach ($items as $item) {
                if (!$this->checkPermissions($item->permissions)) {
                    continue;
                }
                
                $groupItems[] = [
                    'id' => $item->id,
                    'name' => $item->menu_name,
                    'slug' => $item->menu_slug,
                    'url' => $item->menu_url,
                    'icon' => $item->menu_icon,
                    'badge' => $item->badge,
                    'target' => $item->target,
                ];
            }
            
            if (!empty($groupItems)) {
                $menu[] = [
                    'group' => $group->group_name,
                    'slug' => $group->group_slug,
                    'icon' => $group->icon,
                    'items' => $groupItems,
                ];
            }
        }
        
        return $menu;
    }
    
    /**
     * Check user permissions
     */
    private function checkPermissions(string $permissionsJson): bool {
        $permissions = json_decode($permissionsJson, true) ?? [];
        
        if (empty($permissions)) {
            return true; // No restrictions
        }
        
        // Check if user has any of the required permissions
        foreach ($permissions as $permission) {
            if (defined('ADMIN_ROLE')) {
                // Check admin role permissions
                // This would need to be adapted to your permission system
            }
        }
        
        return true; // Simplified - implement proper permission check
    }
    
    /**
     * Add menu item
     */
    public static function addItem(array $data): int {
        return Capsule::table('mod_{module}_menu_items')->insertGetId([
            'menu_name' => $data['name'],
            'menu_slug' => $data['slug'] ?? '',
            'menu_type' => $data['type'] ?? 'main',
            'menu_icon' => $data['icon'] ?? 'fa-link',
            'menu_url' => $data['url'],
            'target' => $data['target'] ?? '_self',
            'parent_id' => $data['parent_id'] ?? 0,
            'sort_order' => $data['sort_order'] ?? 0,
            'permissions' => json_encode($data['permissions'] ?? []),
            'badge' => $data['badge'] ?? '',
            'is_active' => $data['is_active'] ?? 1,
            'module' => $data['module'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Update menu item
     */
    public static function updateItem(int $id, array $data): bool {
        $update = [];
        
        foreach (['menu_name', 'menu_url', 'menu_icon', 'sort_order', 'badge', 'is_active'] as $field) {
            if (isset($data[$field])) {
                $update[$field] = $data[$field];
            }
        }
        
        if (!empty($data['permissions'])) {
            $update['permissions'] = json_encode($data['permissions']);
        }
        
        return Capsule::table('mod_{module}_menu_items')
            ->where('id', $id)
            ->update($update) > 0;
    }
    
    /**
     * Delete menu item
     */
    public static function deleteItem(int $id): bool {
        return Capsule::table('mod_{module}_menu_items')
            ->where('id', $id)
            ->delete() > 0;
    }
    
    /**
     * Reorder menu items
     */
    public static function reorder(array $items): void {
        foreach ($items as $order => $id) {
            Capsule::table('mod_{module}_menu_items')
                ->where('id', $id)
                ->update(['sort_order' => $order]);
        }
    }
    
    /**
     * Get all menu items as tree
     */
    public static function getTree(string $menuType = 'main'): array {
        $items = Capsule::table('mod_{module}_menu_items')
            ->where('menu_type', $menuType)
            ->where('is_active', 1)
            ->orderBy('sort_order')
            ->get();
        
        $tree = [];
        
        foreach ($items as $item) {
            if ($item->parent_id == 0) {
                $tree[$item->id] = [
                    'id' => $item->id,
                    'name' => $item->menu_name,
                    'url' => $item->menu_url,
                    'icon' => $item->menu_icon,
                    'children' => [],
                ];
            }
        }
        
        foreach ($items as $item) {
            if ($item->parent_id > 0 && isset($tree[$item->parent_id])) {
                $tree[$item->parent_id]['children'][] = [
                    'id' => $item->id,
                    'name' => $item->menu_name,
                    'url' => $item->menu_url,
                    'icon' => $item->menu_icon,
                ];
            }
        }
        
        return array_values($tree);
    }
}

/**
 * Menu Item Definitions
 */
class MenuItems {
    
    /**
     * Get default WHMCS menu items
     */
    public static function getDefaultItems(): array {
        return [
            [
                'name' => 'Dashboard',
                'slug' => 'dashboard',
                'icon' => 'fa-home',
                'url' => 'index.php',
            ],
            [
                'name' => 'Clients',
                'slug' => 'clients',
                'icon' => 'fa-users',
                'url' => 'clients.php',
                'children' => [
                    ['name' => 'Search Clients', 'url' => 'clients.php?search='],
                    ['name' => 'Add New Client', 'url' => 'clientsadd.php'],
                    ['name' => 'Client Groups', 'url' => 'clientgroups.php'],
                ],
            ],
            [
                'name' => 'Billing',
                'slug' => 'billing',
                'icon' => 'fa-credit-card',
                'url' => => 'invoices.php',
                'children' => [
                    ['name' => 'Invoices', 'url' => 'invoices.php'],
                    ['name' => 'Transactions', 'url' => 'transactions.php'],
                    ['name' => 'Quotes', 'url' => 'quotes.php'],
                    ['name' => 'Credit', 'url' => 'credits.php'],
                ],
            ],
            [
                'name' => 'Support',
                'slug' => 'support',
                'icon' => 'fa-life-ring',
                'url' => 'supporttickets.php',
                'children' => [
                    ['name' => 'All Tickets', 'url' => 'supporttickets.php'],
                    ['name' => 'New Ticket', 'url' => 'supporttickets.php?action=open'],
                    ['name' => 'Pre-Sales Questions', 'url' => 'supporttickets.php?view=presales'],
                ],
            ],
            [
                'name' => 'Services',
                'slug' => 'services',
                'icon' => 'fa-server',
                'url' => 'services.php',
            ],
            [
                'name' => 'Domains',
                'slug' => 'domains',
                'icon' => 'fa-globe',
                'url' => 'domains.php',
            ],
            [
                'name' => 'Orders',
                'slug' => 'orders',
                'icon' => 'fa-shopping-cart',
                'url' => 'orders.php',
            ],
            [
                'name' => 'Reports',
                'slug' => 'reports',
                'icon' => 'fa-chart-bar',
                'url' => 'reports.php',
            ],
        ];
    }
    
    /**
     * Get custom module menu items
     */
    public static function getModuleMenuItems(): array {
        $modules = Capsule::table('tbladdon_modules')
            ->where('status', 'Active')
            ->get();
        
        $items = [];
        
        foreach ($modules as $module) {
            $items[] = [
                'name' => $module->name,
                'slug' => 'module-' . $module->systemname,
                'icon' => 'fa-puzzle-piece',
                'url' => 'addonmodules.php?module=' . $module->systemname,
            ];
        }
        
        return $items;
    }
}
```

## Hook Integration

```php
<?php
/**
 * Admin Menu Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Add menu items to sidebar
add_hook('AdminAreaNavItems', 1, function($vars) {
    $menuItems = [];
    
    // Add custom tools group
    $menuItems[] = [
        'name' => 'Custom Tools',
        'label' => 'Custom Tools',
        'uri' => '#',
        'icon' => 'fa-tools',
        'order' => 100,
        'children' => [
            [
                'name' => 'API Manager',
                'label' => 'API Manager',
                'uri' => 'addonmodules.php?module=api_manager',
                'icon' => 'fa-key',
                'order' => 1,
            ],
            [
                'name' => 'Backup Manager',
                'label' => 'Backup Manager',
                'uri' => 'addonmodules.php?module=backup_manager',
                'icon' => 'fa-database',
                'order' => 2,
            ],
        ],
    ];
    
    // Add quick links
    $menuItems[] = [
        'name' => 'Quick Links',
        'label' => 'Quick Links',
        'uri' => '#',
        'icon' => 'fa-link',
        'order' => 90,
    ];
    
    return $menuItems;
});

// Modify existing menu items
add_hook('AdminAreaHeaderOutput', 1, function($vars) {
    // Add badge to support menu
    $openTickets = Capsule::table('tbltickets')
        ->where('status', 'Open')
        ->count();
    
    if ($openTickets > 0) {
        // This would modify the existing menu to show badge
        // Implementation depends on WHMCS version
    }
});
```

## Admin Configuration Template

```smarty
<div class="admin-menu-module">
    <h2>Admin Menu Configuration</h2>
    
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Menu Groups</h3>
            <a href="?module={module}&action=groups&sub=add" class="btn btn-success btn-sm pull-right">
                <i class="fa fa-plus"></i> Add Group
            </a>
        </div>
        <div class="panel-body">
            <table class="table table-striped">
                <thead>
                    <tr>
                        <th>Group Name</th>
                        <th>Slug</th>
                        <th>Icon</th>
                        <th>Order</th>
                        <th>Items</th>
                        <th>Status</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $groups as $group}
                    <tr>
                        <td>
                            <i class="fa {$group.icon}"></i>
                            {$group.group_name}
                        </td>
                        <td><code>{$group.group_slug}</code></td>
                        <td>{$group.icon}</td>
                        <td>{$group.sort_order}</td>
                        <td>
                            <span class="badge">{$group.item_count}</span>
                        </td>
                        <td>
                            {if $group.is_active}
                                <span class="label label-success">Active</span>
                            {else}
                                <span class="label label-default">Inactive</span>
                            {/if}
                        </td>
                        <td>
                            <a href="?module={module}&action=groups&sub=edit&id={$group.id}" class="btn btn-xs">
                                <i class="fa fa-edit"></i>
                            </a>
                            <a href="?module={module}&action=groups&sub=delete&id={$group.id}" class="btn btn-xs btn-danger">
                                <i class="fa fa-trash"></i>
                            </a>
                        </td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
        </div>
    </div>
    
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Menu Items</h3>
            <a href="?module={module}&action=items&sub=add" class="btn btn-success btn-sm pull-right">
                <i class="fa fa-plus"></i> Add Item
            </a>
        </div>
        <div class="panel-body">
            <table class="table table-striped">
                <thead>
                    <tr>
                        <th>Name</th>
                        <th>URL</th>
                        <th>Icon</th>
                        <th>Group</th>
                        <th>Order</th>
                        <th>Permissions</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody>
                    {foreach $items as $item}
                    <tr>
                        <td>
                            <i class="fa {$item.menu_icon}"></i>
                            {$item.menu_name}
                        </td>
                        <td><code>{$item.menu_url}</code></td>
                        <td>{$item.menu_icon}</td>
                        <td>{$item.group_name}</td>
                        <td>{$item.sort_order}</td>
                        <td>
                            {if $item.permissions}
                                <span class="badge" title="{$item.permissions}">
                                    {$item.permissions_count} required
                                </span>
                            {else}
                                <span class="text-muted">None</span>
                            {/if}
                        </td>
                        <td>
                            <a href="?module={module}&action=items&sub=edit&id={$item.id}" class="btn btn-xs">
                                <i class="fa fa-edit"></i>
                            </a>
                            <a href="?module={module}&action=items&sub=delete&id={$item.id}" class="btn btn-xs btn-danger">
                                <i class="fa fa-trash"></i>
                            </a>
                        </td>
                    </tr>
                    {/foreach}
                </tbody>
            </table>
        </div>
    </div>
    
    <div class="panel panel-default">
        <div class="panel-heading">
            <h3 class="panel-title">Add/Edit Menu Item</h3>
        </div>
        <div class="panel-body">
            <form method="post" action="?module={module}&action=items&sub=save">
                <input type="hidden" name="csrf_token" value="{$csrf_token}">
                <input type="hidden" name="id" value="{$edit_item.id}">
                
                <div class="row">
                    <div class="col-md-6">
                        <div class="form-group">
                            <label>Menu Name</label>
                            <input type="text" name="menu_name" class="form-control" 
                                   value="{$edit_item.menu_name}" required>
                        </div>
                    </div>
                    <div class="col-md-6">
                        <div class="form-group">
                            <label>URL</label>
                            <input type="text" name="menu_url" class="form-control"
                                   value="{$edit_item.menu_url}" required>
                        </div>
                    </div>
                </div>
                
                <div class="row">
                    <div class="col-md-6">
                        <div class="form-group">
                            <label>Icon (FontAwesome class)</label>
                            <input type="text" name="menu_icon" class="form-control"
                                   value="{$edit_item.menu_icon|default:'fa-link'}"
                                   placeholder="fa-link">
                        </div>
                    </div>
                    <div class="col-md-6">
                        <div class="form-group">
                            <label>Menu Group</label>
                            <select name="group_slug" class="form-control">
                                {foreach $groups as $group}
                                <option value="{$group.group_slug}" 
                                        {if $edit_item.group_slug eq $group.group_slug}selected{/if}>
                                    {$group.group_name}
                                </option>
                                {/foreach}
                            </select>
                        </div>
                    </div>
                </div>
                
                <div class="row">
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Sort Order</label>
                            <input type="number" name="sort_order" class="form-control"
                                   value="{$edit_item.sort_order|default:0}">
                        </div>
                    </div>
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Target</label>
                            <select name="target" class="form-control">
                                <option value="_self" {if $edit_item.target eq '_self'}selected{/if}>
                                    Same Window
                                </option>
                                <option value="_blank" {if $edit_item.target eq '_blank'}selected{/if}>
                                    New Window
                                </option>
                            </select>
                        </div>
                    </div>
                    <div class="col-md-4">
                        <div class="form-group">
                            <label>Active</label>
                            <select name="is_active" class="form-control">
                                <option value="1" {if $edit_item.is_active}selected{/if}>Yes</option>
                                <option value="0" {if !$edit_item.is_active}selected{/if}>No</option>
                            </select>
                        </div>
                    </div>
                </div>
                
                <button type="submit" class="btn btn-primary">
                    <i class="fa fa-save"></i> Save Menu Item
                </button>
            </form>
        </div>
    </div>
</div>
```

## Checklist

```
Pre-Dev:
□ Define menu structure
□ Plan menu groups
□ Identify required permissions
□ Design menu item attributes

Development:
□ Create module with database tables
□ Implement MenuBuilder class
□ Create MenuItems helper class
□ Add menu registration hook
□ Build admin configuration UI
□ Add group management
□ Add item management
□ Implement drag-drop reordering
□ Add permission checking

Testing:
□ Test menu display
□ Verify permissions
□ Test reordering
□ Test new menu items
□ Verify icons
□ Check badge display
```