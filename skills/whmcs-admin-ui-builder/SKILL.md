# WHMCS Admin UI Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for building professional admin interfaces in WHMCS modules.

## When to Use

- Creating admin dashboards
- Building configuration pages
- Designing data tables

## Admin UI Patterns

### Admin Controller

```php
function {module}_output(array $vars): void {
    $tab = $_GET['tab'] ?? 'dashboard';

    echo '<div class="admin-container">';

    // Header
    echo '<div class="admin-header">';
    echo '<h1>{Module Name}</h1>';
    echo '<a href="?module={module}&action=settings" class="btn btn-settings">Settings</a>';
    echo '</div>';

    // Navigation tabs
    echo '<div class="admin-tabs">';
    echo '<a href="?module={module}&tab=dashboard" class="' . ($tab === 'dashboard' ? 'active' : '') . '">Dashboard</a>';
    echo '<a href="?module={module}&tab=data" class="' . ($tab === 'data' ? 'active' : '') . '">Data</a>';
    echo '<a href="?module={module}&tab=logs" class="' . ($tab === 'logs' ? 'active' : '') . '">Logs</a>';
    echo '</div>';

    // Content
    echo '<div class="admin-content">';
    include __DIR__ . '/templates/admin/' . $tab . '.tpl';
    echo '</div>';

    echo '</div>';
}
```

### Data Table

```smarty
<div class="data-table-container">
    <table class="data-table">
        <thead>
            <tr>
                <th>ID</th>
                <th>Name</th>
                <th>Status</th>
                <th>Created</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {foreach $items as $item}
            <tr>
                <td>{$item.id}</td>
                <td>{$item.name|escape:'html'}</td>
                <td>
                    <span class="badge badge-{$item.status|lower}">{$item.status}</span>
                </td>
                <td>{$item.created_at|date_format:'%Y-%m-%d'}</td>
                <td>
                    <a href="?module={module}&action=edit&id={$item.id}" class="btn btn-sm">Edit</a>
                    <a href="?module={module}&action=delete&id={$item.id}" class="btn btn-sm btn-danger"
                       onclick="return confirm('Delete?')">Delete</a>
                </td>
            </tr>
            {/foreach}
        </tbody>
    </table>

    {$pagination}
</div>
```

---

**Related Skills:**
- whmcs-template-styling
- whmcs-clientarea-builder
- whmcs-ajax-patterns