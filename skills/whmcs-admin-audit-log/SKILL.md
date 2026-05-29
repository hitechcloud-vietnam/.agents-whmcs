# WHMCS Admin Audit Log

## Overview
Guide for implementing comprehensive audit logging in WHMCS admin area. Covers activity tracking, log queries, and compliance features.

## Audit Log System

### Log Activity

```php
<?php
// /includes/hooks/admin_audit_log.php

function logAdminActivity(
    string $action,
    ?int $relatedId = null,
    array $details = [],
    ?int $adminId = null
): int {
    $adminId = $adminId ?? $_SESSION["adminid"];
    
    return Capsule::table("mod_admin_audit_log")->insertGetId([
        "admin_id" => $adminId,
        "action" => $action,
        "related_id" => $relatedId,
        "related_type" => $details["related_type"] ?? null,
        "details" => json_encode($details),
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "created_at" => date("Y-m-d H:i:s")
    ]);
}

// Define action categories
define("AUDIT_ACTIONS", [
    "login" => "Authentication",
    "logout" => "Authentication",
    "client_create" => "Client Management",
    "client_update" => "Client Management",
    "client_delete" => "Client Management",
    "service_create" => "Services",
    "service_update" => "Services",
    "service_terminate" => "Services",
    "invoice_create" => "Billing",
    "invoice_update" => "Billing",
    "invoice_paid" => "Billing",
    "config_change" => "System",
    "permission_change" => "Security",
    "mass_action" => "Bulk Operations"
]);
```

### Automatic Logging Hooks

```php
add_hook("AdminLogin", 1, function(array $params) {
    logAdminActivity("login", $params["admin_id"], [
        "related_type" => "admin"
    ]);
    return $params;
});

add_hook("AdminLogout", 1, function(array $params) {
    logAdminActivity("logout", $params["admin_id"], [
        "related_type" => "admin"
    ]);
    return $params;
});

add_hook("AdminClientUpdate", 1, function(array $params) {
    $changedFields = [];
    foreach ($params["data"] as $field => $value) {
        if ($params["original"][$field] !== $value) {
            $changedFields[$field] = [
                "from" => $params["original"][$field],
                "to" => $value
            ];
        }
    }
    
    if (!empty($changedFields)) {
        logAdminActivity("client_update", $params["client_id"], [
            "related_type" => "client",
            "changed_fields" => $changedFields
        ]);
    }
    
    return $params;
});
```

### Query Audit Log

```php
function queryAuditLog(
    array $filters = [],
    int $page = 1,
    int $perPage = 50
): array {
    $query = Capsule::table("mod_admin_audit_log")
        ->join("tbladmins", "mod_admin_audit_log.admin_id", "=", "tbladmins.id")
        ->select([
            "mod_admin_audit_log.*",
            "tbladmins.username",
            "tbladmins.email"
        ]);
    
    // Apply filters
    if (!empty($filters["admin_id"])) {
        $query->where("admin_id", $filters["admin_id"]);
    }
    
    if (!empty($filters["action"])) {
        $query->where("action", $filters["action"]);
    }
    
    if (!empty($filters["related_type"])) {
        $query->where("related_type", $filters["related_type"]);
    }
    
    if (!empty($filters["date_from"])) {
        $query->where("created_at", ">=", $filters["date_from"]);
    }
    
    if (!empty($filters["date_to"])) {
        $query->where("created_at", "<=", $filters["date_to"]);
    }
    
    if (!empty($filters["search"])) {
        $query->where(function($q) use ($filters) {
            $q->where("tbladmins.username", "like", "%" . $filters["search"] . "%")
              ->orWhere("details", "like", "%" . $filters["search"] . "%");
        });
    }
    
    $total = $query->count();
    
    $logs = $query
        ->orderBy("created_at", "desc")
        ->offset(($page - 1) * $perPage)
        ->limit($perPage)
        ->get();
    
    return [
        "logs" => $logs,
        "total" => $total,
        "page" => $page,
        "per_page" => $perPage,
        "total_pages" => ceil($total / $perPage)
    ];
}
```

### Compliance Reports

```php
function generateComplianceReport(
    string $startDate,
    string $endDate
): array {
    // Access patterns
    $accessPatterns = Capsule::table("mod_admin_audit_log")
        ->select(
            "admin_id",
            Capsule::raw("COUNT(*) as total_actions"),
            Capsule::raw("COUNT(DISTINCT DATE(created_at)) as active_days"),
            Capsule::raw("MIN(created_at) as first_access"),
            Capsule::raw("MAX(created_at) as last_access")
        )
        ->whereBetween("created_at", [$startDate, $endDate])
        ->groupBy("admin_id")
        ->get();
    
    // Action breakdown
    $actionBreakdown = Capsule::table("mod_admin_audit_log")
        ->select("action", Capsule::raw("COUNT(*) as count"))
        ->whereBetween("created_at", [$startDate, $endDate])
        ->groupBy("action")
        ->orderBy("count", "desc")
        ->get();
    
    // Sensitive actions
    $sensitiveActions = Capsule::table("mod_admin_audit_log")
        ->select("*")
        ->whereBetween("created_at", [$startDate, $endDate])
        ->whereIn("action", ["permission_change", "config_change", "client_delete"])
        ->get();
    
    // Failed logins
    $failedLogins = Capsule::table("mod_admin_audit_log")
        ->select("*")
        ->whereBetween("created_at", [$startDate, $endDate])
        ->where("action", "login_failed")
        ->get();
    
    return [
        "period" => ["start" => $startDate, "end" => $endDate],
        "access_patterns" => $accessPatterns,
        "action_breakdown" => $actionBreakdown,
        "sensitive_actions" => $sensitiveActions,
        "failed_logins" => $failedLogins
    ];
}
```

## Audit Log Template

```smarty
<!-- /admin/templates/audit_log.tpl -->
<div class="audit-log-container">
    <div class="audit-header">
        <h2>Admin Audit Log</h2>
        <div class="audit-actions">
            <a href="export_audit_log.php" class="btn btn-default">
                <i class="fa fa-download"></i> Export
            </a>
        </div>
    </div>
    
    <div class="audit-filters">
        <form method="get" class="form-inline">
            <select name="admin_id" class="form-control">
                <option value="">All Admins</option>
                {foreach $admins as $admin}
                    <option value="{$admin.id}">{$admin.username}</option>
                {/foreach}
            </select>
            
            <select name="action" class="form-control">
                <option value="">All Actions</option>
                {foreach $action_categories as $category => $actions}
                    <optgroup label="{$category}">
                        {foreach $actions as $action}
                            <option value="{$action}">{$action}</option>
                        {/foreach}
                    </optgroup>
                {/foreach}
            </select>
            
            <input type="date" name="date_from" class="form-control">
            <input type="date" name="date_to" class="form-control">
            
            <button type="submit" class="btn btn-primary">Filter</button>
        </form>
    </div>
    
    <table class="table audit-table">
        <thead>
            <tr>
                <th>Timestamp</th>
                <th>Admin</th>
                <th>Action</th>
                <th>Details</th>
                <th>IP Address</th>
            </tr>
        </thead>
        <tbody>
            {foreach $logs as $log}
                <tr>
                    <td>{$log.created_at}</td>
                    <td>
                        <a href="admin_profile.php?id={$log.admin_id}">
                            {$log.username}
                        </a>
                    </td>
                    <td>
                        <span class="action-badge action-{$log.action}">
                            {$log.action}
                        </span>
                    </td>
                    <td>
                        {if $log.details}
                            <button class="btn btn-xs btn-link details-toggle"
                                    data-details='{$log.details}'>
                                View Details
                            </button>
                        {/if}
                    </td>
                    <td>{$log.ip_address}</td>
                </tr>
            {/foreach}
        </tbody>
    </table>
    
    {include file="pagination.tpl" pagination=$pagination}
</div>
```

## Database Schema

```php
Capsule::schema()->create('mod_admin_audit_log', function($t) {
    $t->increments('id');
    $t->integer('admin_id');
    $t->string('action', 100);
    $t->integer('related_id')->nullable();
    $t->string('related_type', 50)->nullable();
    $t->text('details')->nullable();
    $t->string('ip_address', 45);
    $t->string('user_agent', 500)->nullable();
    $t->timestamp('created_at');
    
    $t->index(['admin_id', 'created_at']);
    $t->index(['action']);
    $t->index(['related_type', 'related_id']);
    $t->index(['created_at']);
});
```

## Best Practices

1. **Comprehensive**: Log all admin actions
2. **Immutability**: Make logs immutable (append-only)
3. **Retention**: Define retention policy
4. **Integrity**: Use checksums for log integrity
5. **Access Control**: Restrict log access to admins
6. **Real-Time**: Stream critical events
7. **Compliance**: Support compliance reporting
8. **Rotation**: Archive old logs
