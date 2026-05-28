# WHMCS Admin Menu DevKit

A comprehensive custom admin menu module for WHMCS that enables adding custom navigation items, organizing menus into groups, and managing permissions.

## Features

- Custom menu groups with icons
- Drag-and-drop menu item ordering
- Permission-based menu visibility
- Badge support for notifications
- Multi-level menu support (dropdowns)
- Module integration
- Admin configuration interface
- Import/export menu configuration

## Installation

1. Copy module files to:
   ```
   modules/addons/whmcs_admin_menu/
   ```

2. Activate the module in WHMCS Admin > Addon Modules

3. Configure menu items via the admin interface

## Configuration

### Menu Groups

Menu items are organized into groups:

| Setting | Description |
|---------|-------------|
| Group Name | Display name for the group |
| Slug | Unique identifier (URL-safe) |
| Icon | FontAwesome icon class |
| Sort Order | Display order (lower = first) |
| Active | Enable/disable the group |

### Menu Items

| Setting | Description |
|---------|-------------|
| Name | Display name |
| URL | Link destination |
| Icon | FontAwesome icon |
| Group | Parent group |
| Sort Order | Display order |
| Target | _self (same window) or _blank (new window) |
| Permissions | Required permissions (JSON array) |
| Badge | Optional badge text/count |

## Usage

### Adding Menu Items via Code

```php
use AdminMenu\MenuBuilder;

// Add a menu item
MenuBuilder::addItem([
    'name' => 'API Manager',
    'slug' => 'api-manager',
    'url' => 'addonmodules.php?module=api_manager',
    'icon' => 'fa-key',
    'type' => 'main',
    'group_slug' => 'custom-tools',
    'permissions' => ['API Access'],
    'sort_order' => 10,
]);

// Update a menu item
MenuBuilder::updateItem($id, [
    'menu_name' => 'Updated Name',
    'sort_order' => 5,
]);

// Delete a menu item
MenuBuilder::deleteItem($id);

// Reorder items
MenuBuilder::reorder([1, 3, 2, 5, 4]); // id in desired order
```

### Get Menu as Tree

```php
$menuTree = MenuBuilder::getTree('main');

// Returns nested structure:
[
    [
        'id' => 1,
        'name' => 'Custom Tools',
        'url' => '#',
        'icon' => 'fa-tools',
        'children' => [
            ['id' => 2, 'name' => 'API Manager', 'url' => '...'],
            ['id' => 3, 'name' => 'Backup Manager', 'url' => '...'],
        ],
    ],
]
```

### Hook Integration

```php
// Register menu items via hook
add_hook('AdminAreaNavItems', 1, function($vars) {
    return [
        [
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
                ],
            ],
        ],
    ];
});
```

## Permissions

Menu items can require specific permissions:

```json
["full_admin", "support_access"]
```

Only users with these permissions will see the menu item.

### Available Permission Keys

| Permission | Description |
|-----------|-------------|
| `full_admin` | Full administrator access |
| `support_access` | Support department access |
| `client_management` | Client management |
| `billing_management` | Billing/invoice access |
| `server_management` | Server management |
| `module_xyz` | Specific module access |

## Menu Types

- **main** - Primary sidebar menu
- **topbar** - Top navigation bar
- **context** - Context-specific menus

## Icon Reference

Uses FontAwesome 5.x icons. Examples:
- `fa-home` - Dashboard
- `fa-users` - Clients
- `fa-credit-card` - Billing
- `fa-life-ring` - Support
- `fa-server` - Services
- `fa-globe` - Domains
- `fa-chart-bar` - Reports
- `fa-cogs` - Settings
- `fa-plug` - Integrations
- `fa-key` - Security/API

## Database Schema

### menu_groups table
- `id` - Group ID
- `group_name` - Display name
- `group_slug` - Unique identifier
- `icon` - FontAwesome icon
- `sort_order` - Display order
- `is_active` - Enable/disable

### menu_items table
- `id` - Item ID
- `menu_name` - Display name
- `menu_slug` - URL-safe identifier
- `menu_type` - main, sidebar, topbar
- `menu_icon` - FontAwesome class
- `menu_url` - Link destination
- `target` - _self, _blank
- `parent_id` - Parent item ID (for submenus)
- `sort_order` - Display order
- `permissions` - Required permissions (JSON)
- `badge` - Badge text
- `is_active` - Enable/disable
- `module` - Associated module

## Admin Interface

### Menu Groups Management

- View all menu groups
- Add new groups
- Edit group settings
- Enable/disable groups
- Delete groups (with confirmation)

### Menu Items Management

- View all menu items
- Add new items
- Edit item settings
- Drag-and-drop reordering
- Set permissions
- Add badges
- Delete items

### Preview

Live preview of menu changes in admin area.

## File Structure

```
whmcs-admin-menu/
├── admin-menu.php            # Main module
├── lib/
│   ├── MenuBuilder.php       # Menu builder
│   └── MenuItems.php         # Menu item definitions
└── templates/
    └── menu-config.tpl       # Admin configuration
```

## Hooks

| Hook | Description |
|------|-------------|
| `AdminAreaNavItems` | Add menu items to navigation |
| `AdminAreaHeaderOutput` | Modify header (add badges) |

## Requirements

- WHMCS 7.0+
- PHP 7.4+

## Support

For issues and feature requests, please contact the developer.