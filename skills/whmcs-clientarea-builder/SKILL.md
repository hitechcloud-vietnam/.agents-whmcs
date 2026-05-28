# WHMCS Client Area Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building WHMCS client area pages for service management and information display.

## When to Use

- Creating custom service management pages
- Building client dashboards for hosting services
- Adding service-specific actions in client portal

## Client Area Functions

```php
// modules/servers/{module}/{module}.php

/**
 * Main client area page
 */
function {module}_ClientArea(array $params): array {
    $action = $_GET['action'] ?? 'overview';

    return match ($action) {
        'overview' => renderOverview($params),
        'manage' => renderManage($params),
        'console' => renderConsole($params),
        'usage' => renderUsage($params),
        default => renderOverview($params),
    };
}

/**
 * Custom button array for client area
 */
function {module}_ClientAreaCustomButtonArray(): array {
    return [
        'Restart Server' => 'restart',
        'Reinstall OS' => 'reinstall',
        'VNC Console' => 'console',
    ];
}

/**
 * Allowed functions in client area
 */
function {module}_ClientAreaAllowedFunctions(): array {
    return [
        'overview' => 'Overview',
        'manage' => 'Manage',
        'backup' => 'Backup Management',
    ];
}
```

## Template Rendering

```php
// templates/overview.tpl
<div class="service-overview">
    <h2>{$product_name}</h2>

    <div class="service-status">
        <span class="label">Status:</span>
        <span class="badge badge-{$status|lower}">{$status}</span>
    </div>

    <div class="service-details">
        <div class="detail-row">
            <span class="label">Hostname:</span>
            <span class="value">{$hostname|escape:'html'}</span>
        </div>
        <div class="detail-row">
            <span class="label">IP Address:</span>
            <span class="value">{$server_ip|escape:'html'}</span>
        </div>
        <div class="detail-row">
            <span class="label">Plan:</span>
            <span class="value">{$plan|escape:'html'}</span>
        </div>
    </div>

    <div class="service-actions">
        <a href="?action=manage" class="btn btn-primary">Manage</a>
        <a href="?action=console" class="btn btn-secondary">Console</a>
    </div>
</div>
```

## Action Handling

```php
// modules/servers/{module}/actions.php
if (!defined("WHMCS")) { die("Direct access denied"); }

// Handle client area actions
$action = $_REQUEST['action'] ?? '';
check_token('WHMCS.default');

switch ($action) {
    case 'restart':
        handleRestart();
        break;
    case 'reinstall':
        handleReinstall();
        break;
    case 'console':
        handleConsole();
        break;
    case 'backup_create':
        handleBackupCreate();
        break;
}

function handleRestart() {
    $serviceId = (int) $_REQUEST['serviceid'];
    $serverId = getCustomFieldValue($serviceId, 'server_id');

    try {
        $api = new ApiClient($this->params);
        $api->rebootServer($serverId);

        logActivity('Server restart initiated: ' . $serverId);
        redirect('clientarea.php?action=manage&success=restart');
    } catch (\Exception $e) {
        redirect('clientarea.php?action=manage&error=' . urlencode($e->getMessage()));
    }
}
```

## Checklist

- [ ] ClientArea function returns array
- [ ] Template files in templates/
- [ ] Custom buttons array
- [ ] Allowed functions array
- [ ] CSRF protection on actions
- [ ] Output escaping
- [ ] Error handling with redirect

---

**Related Skills:**
- whmcs-server-builder
- whmcs-template-styling
- whmcs-clientarea-ajax