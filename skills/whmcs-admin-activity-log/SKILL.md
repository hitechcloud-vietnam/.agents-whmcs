# WHMCS Admin Activity Log

## Overview
Guide for implementing comprehensive activity logging for WHMCS admin area. Covers action tracking, log querying, and reporting.

## Activity Logging

### Log Admin Action

```php
<?php
// /includes/hooks/admin_activity_log.php

function logAdminAction(
    string $action,
    ?int $adminId = null,
    ?int $targetId = null,
    string $targetType = "",
    array $details = []
): int {
    $adminId = $adminId ?? $_SESSION["adminid"];
    
    return Capsule::table("mod_admin_activity_log")->insertGetId([
        "admin_id" => $adminId,
        "action" => $action,
        "target_id" => $targetId,
        "target_type" => $targetType,
        "details" => json_encode($details),
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "created_at" => date("Y-m-d H:i:s")
    ]);
}

// Action constants
define("ADMIN_ACTIONS", [
    // Authentication
    "admin_login",
    "admin_logout",
    "admin_login_failed",
    
    // Client Management
    "client_created",
    "client_updated",
    "client_deleted",
    "client_merged",
    
    // Services
    "service_created",
    "service_updated",
    "service_suspended",
    "service_unsuspended",
    "service_terminated",
    "service_upgraded",
    "service_downgraded",
    
    // Billing
    "invoice_created",
    "invoice_updated",
    "invoice_sent",
    "invoice_paid",
    "invoice_cancelled",
    "refund_processed",
    
    // Support
    "ticket_created",
    "ticket_replied",
    "ticket_closed",
    
    // System
    "settings_changed",
    "permission_changed",
    "api_key_created",
    "api_key_revoked",
    "mass_operation"
]);
```

### Automatic Logging

```php
add_hook("AdminClientCreated", 1, function(array $params) {
    logAdminAction("client_created", null, $params["client_id"], "client", [
        "email" => $params["email"]
    ]);
    return $params;
});

add_hook("AdminClientUpdated", 1, function(array $params) {
    $changes = array_diff_assoc($params["new_data"], $params["old_data"]);
    
    logAdminAction("client_updated", null, $params["client_id"], "client", [
        "changes" => $changes
    ]);
    return $params;
});

add_hook("AdminServiceTerminated", 1, function(array $params) {
    logAdminAction("service_terminated", null, $params["service_id"], "service", [
        "domain" => $params["domain"]
    ]);
    return $params;
});
```

### Query Activity Logs

```php
function queryAdminActivity(array $filters = []): array
{
    $query = Capsule::table("mod_admin_activity_log")
        ->join("tbladmins", "mod_admin_activity_log.admin_id", "=", "tbladmins.id")
        ->select([
            "mod_admin_activity_log.*",
            "tbladmins.username"
        ]);
    
    // Apply filters
    if (!empty($filters["admin_id"])) {
        $query->where("admin_id", $filters["admin_id"]);
    }
    
    if (!empty($filters["action"])) {
        $query->where("action", $filters["action"]);
    }
    
    if (!empty($filters["target_type"])) {
        $query->where("target_type", $filters["target_type"]);
    }
    
    if (!empty($filters["target_id"])) {
        $query->where("target_id", $filters["target_id"]);
    }
    
    if (!empty($filters["date_from"])) {
        $query->where("created_at", ">=", $filters["date_from"]);
    }
    
    if (!empty($filters["date_to"])) {
        $query->where("created_at", "<=", $filters["date_to"]);
    }
    
    return $query
        ->orderBy("created_at", "desc")
        ->limit(100)
        ->get();
}
```

### Activity Statistics

```php
function getActivityStatistics(string $period = "30 days"): array
{
    $startDate = date("Y-m-d H:i:s", strtotime("-{$period}"));
    
    // Total actions
    $totalActions = Capsule::table("mod_admin_activity_log")
        ->where("created_at", ">=", $startDate)
        ->count();
    
    // Actions by admin
    $actionsByAdmin = Capsule::table("mod_admin_activity_log")
        ->select("admin_id", "tbladmins.username")
        ->join("tbladmins", "mod_admin_activity_log.admin_id", "=", "tbladmins.id")
        ->where("mod_admin_activity_log.created_at", ">=", $startDate)
        ->groupBy("admin_id")
        ->selectRaw("COUNT(*) as action_count")
        ->orderBy("action_count", "desc")
        ->get();
    
    // Actions by type
    $actionsByType = Capsule::table("mod_admin_activity_log")
        ->select("action")
        ->where("created_at", ">=", $startDate)
        ->groupBy("action")
        ->selectRaw("COUNT(*) as count")
        ->orderBy("count", "desc")
        ->get();
    
    // Recent activity
    $recentActivity = Capsule::table("mod_admin_activity_log")
        ->join("tbladmins", "mod_admin_activity_log.admin_id", "=", "tbladmins.id")
        ->select("mod_admin_activity_log.*", "tbladmins.username")
        ->where("mod_admin_activity_log.created_at", ">=", $startDate)
        ->orderBy("created_at", "desc")
        ->limit(20)
        ->get();
    
    return [
        "period" => $period,
        "total_actions" => $totalActions,
        "actions_by_admin" => $actionsByAdmin,
        "actions_by_type" => $actionsByType,
        "recent_activity" => $recentActivity
    ];
}
```

## Activity Log Template

```smarty
<!-- /admin/templates/admin_activity_log.tpl -->
<div class="activity-log-container">
    <div class="page-header">
        <h2>Admin Activity Log</h2>
        <div class="actions">
            <a href="export_activity_log.php" class="btn btn-default">
                <i class="fa fa-download"></i> Export
            </a>
        </div>
    </div>
    
    <div class="activity-stats">
        <div class="stat-card">
            <h4>Total Actions</h4>
            <span class="stat-value">{$stats.total_actions}</span>
            <span class="stat-period">Last {$stats.period}</span>
        </div>
    </div>
    
    <div class="activity-filters">
        <form method="get" class="form-inline">
            <select name="admin_id" class="form-control">
                <option value="">All Admins</option>
                {foreach $admins as $admin}
                    <option value="{$admin.id}">{$admin.username}</option>
                {/foreach}
            </select>
            
            <select name="action" class="form-control">
                <option value="">All Actions</option>
                {foreach $actions as $action}
                    <option value="{$action}">{$action|ucfirst|replace:'_':' '}</option>
                {/foreach}
            </select>
            
            <input type="date" name="date_from" class="form-control">
            <input type="date" name="date_to" class="form-control">
            
            <button type="submit" class="btn btn-primary">Filter</button>
        </form>
    </div>
    
    <table class="table activity-table">
        <thead>
            <tr>
                <th>Time</th>
                <th>Admin</th>
                <th>Action</th>
                <th>Target</th>
                <th>Details</th>
                <th>IP</th>
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
                        <span class="action-badge">
                            {$log.action|ucfirst|replace:'_':' '}
                        </span>
                    </td>
                    <td>
                        {if $log.target_id}
                            <a href="{$log.target_type}_{$log.target_id}.php">
                                #{$log.target_id}
                            </a>
                        {else}
                            -
                        {/if}
                    </td>
                    <td>
                        {if $log.details}
                            <button class="btn btn-xs btn-link details-toggle"
                                    data-details='{$log.details}'>
                                View
                            </button>
                        {/if}
                    </td>
                    <td>{$log.ip_address}</td>
                </tr>
            {/foreach}
        </tbody>
    </table>
</div>
```

## Best Practices

1. **Automatic Logging**: Hook into all admin actions
2. **Detail Capture**: Log relevant details for each action
3. **Retention**: Define log retention policy
4. **Indexing**: Index commonly queried columns
5. **Export**: Support export for analysis
6. **Dashboard**: Show activity dashboard
7. **Alerts**: Alert on suspicious activity
8. **Compliance**: Support compliance reporting
