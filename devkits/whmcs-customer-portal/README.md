# WHMCS Custom Customer Portal Module

A feature-rich custom client portal dashboard for WHMCS with customizable widgets, themes, and enhanced user experience.

## Features

- Customizable dashboard with drag-and-drop widgets
- Multiple theme support (Default, Dark, Light, Custom)
- User preference persistence
- Quick action shortcuts
- Real-time widget data loading
- Personal quick links management
- Activity tracking
- Responsive design ready

## Installation

1. Copy the module to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/customerportal/
   ```

2. Activate through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "Custom Customer Portal"
   - Click **Activate**
   - Configure settings

3. Module Settings:
   - **Enable Dashboard**: Toggle custom dashboard
   - **Dashboard Theme**: Choose portal theme
   - **Enable Widgets**: Allow widget customization
   - **Enable Quick Actions**: Show quick action buttons
   - **Recent Activity Limit**: Number of activities to show

## Available Widgets

| Widget ID | Name | Description |
|-----------|------|-------------|
| `overview` | Account Overview | Account summary and key stats |
| `services` | My Services | Active services and products |
| `invoices` | Recent Invoices | Invoice history |
| `tickets` | Support Tickets | Open support tickets |
| `domains` | My Domains | Registered domains |
| `billing` | Billing Summary | Payment methods and totals |
| `usage` | Resource Usage | Bandwidth and storage |
| `announcements` | Announcements | Latest news |
| `activity` | Recent Activity | Account activity log |
| `quickactions` | Quick Actions | Shortcut buttons |
| `affiliate` | Affiliate Program | Affiliate statistics |
| `security` | Security Status | Account security |

## Usage

### Getting Dashboard Data

```php
// Get complete dashboard data for a user
$data = customerportal_GetDashboardData($userId);

// Access components
$widgets = $data['widgets'];
$settings = $data['settings'];
$quickActions = $data['quick_actions'];
$user = $data['user'];
```

### Rendering the Portal

```php
// Render complete portal HTML
$html = customerportal_RenderPortal($userId, array(
    'theme' => 'dark',
));

echo $html;
```

### Managing Widgets

```php
// Get available widgets
$widgets = customerportal_GetWidgets();

// Get user's widget configuration
$userWidgets = customerportal_GetUserWidgets($userId);

// Save widget preferences
customerportal_SaveWidgetPreferences($userId, array(
    'overview' => array('position' => 0, 'is_visible' => true),
    'services' => array('position' => 1, 'is_visible' => true),
    'domains' => array('position' => 2, 'is_visible' => false),
));

// Get widget content
$content = customerportal_GetWidgetContent($userId, 'services', array(
    'limit' => 10,
));
```

### User Settings

```php
// Get user settings
$settings = customerportal_GetUserSettings($userId);

// Save user settings
customerportal_SaveUserSettings($userId, array(
    'theme' => 'dark',
    'language' => 'english',
    'timezone' => 'America/New_York',
    'date_format' => 'm/d/Y',
    'notifications' => array('email' => true, 'sms' => false),
));
```

### Quick Links

```php
// Add custom quick link
customerportal_AddQuickLink($userId, array(
    'title' => 'My Custom Link',
    'url' => 'custom-page.php',
    'icon' => 'fa-star',
    'category' => 'custom',
    'sort_order' => 5,
));

// Get quick actions
$quickActions = customerportal_GetQuickActions($userId);

// Remove quick link
customerportal_RemoveQuickLink($linkId, $userId);
```

## Template Integration

Add to your clientarea template:

```smarty
<div id="custom-customer-portal">
    {$portalHtml}
</div>
```

In PHP:

```php
$portalHtml = customerportal_RenderPortal($userId);
$smarty->assign('portalHtml', $portalHtml);
```

## Database Tables

- `mod_customerportal_widget_prefs` - Widget preferences per user
- `mod_customerportal_settings` - User portal settings
- `mod_customerportal_quicklinks` - Custom quick links

## API Functions

| Function | Description |
|----------|-------------|
| `customerportal_GetWidgets()` | Get all available widgets |
| `customerportal_GetDashboardData()` | Get complete dashboard data |
| `customerportal_GetUserWidgets()` | Get user's widget config |
| `customerportal_SaveWidgetPreferences()` | Save widget preferences |
| `customerportal_GetUserSettings()` | Get user settings |
| `customerportal_SaveUserSettings()` | Save user settings |
| `customerportal_GetQuickActions()` | Get quick action buttons |
| `customerportal_AddQuickLink()` | Add custom quick link |
| `customerportal_RemoveQuickLink()` | Remove quick link |
| `customerportal_GetWidgetContent()` | Get widget data |
| `customerportal_RenderPortal()` | Render portal HTML |

## Version History

- **1.0.0** - Initial release
  - Customizable dashboard widgets
  - Multiple theme support
  - User preferences
  - Quick actions
  - Quick links management
