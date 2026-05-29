# WHMCS User Activity Log

## Overview
Guide for implementing comprehensive user activity logging in WHMCS. Covers activity tracking, storage, and retrieval.

## Activity Logging System

### Log Activity

```php
<?php
// /includes/hooks/activity_log.php

function logUserActivity(
    int $userId,
    string $action,
    string $category,
    array $details = [],
    ?string $ipAddress = null
): int {
    return Capsule::table("mod_user_activity_log")->insertGetId([
        "user_id" => $userId,
        "action" => $action,
        "category" => $category,
        "details" => json_encode($details),
        "ip_address" => $ipAddress ?? $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "created_at" => date("Y-m-d H:i:s")
    ]);
}

// Define activity categories and actions
define("ACTIVITY_CATEGORIES", [
    "account" => ["login", "logout", "register", "profile_update", "password_change"],
    "billing" => ["invoice_viewed", "payment_made", "refund_requested"],
    "services" => ["service_ordered", "service_cancelled", "service_upgraded"],
    "support" => ["ticket_created", "ticket_replied", "ticket_closed"],
    "api" => ["api_request", "api_key_created", "api_key_revoked"],
]);

// Hook into common actions
add_hook("ClientLogin", 1, function(array $params) {
    logUserActivity($params["userid"], "login", "account", [
        "ip" => $_SERVER["REMOTE_ADDR"]
    ]);
    return $params;
});

add_hook("ClientLogout", 1, function(array $params) {
    logUserActivity($params["userid"], "logout", "account");
    return $params;
});

add_hook("ClientUpdate", 1, function(array $params) {
    logUserActivity($params["userid"], "profile_update", "account", [
        "updated_fields" => array_keys($params)
    ]);
    return $params;
});

add_hook("InvoicePaid", 1, function(array $params) {
    $invoice = Capsule::table("tblinvoices")->where("id", $params["invoiceid"])->first();
    logUserActivity($invoice->userid, "payment_made", "billing", [
        "invoice_id" => $params["invoiceid"],
        "amount" => $invoice->total
    ]);
    return $params;
});

add_hook("TicketOpen", 1, function(array $params) {
    logUserActivity($params["userid"], "ticket_created", "support", [
        "ticket_id" => $params["ticketid"],
        "subject" => $params["subject"]
    ]);
    return $params;
});
```

### Query Activity Log

```php
function getUserActivity(
    int $userId,
    ?string $category = null,
    ?string $startDate = null,
    ?string $endDate = null,
    int $limit = 50,
    int $offset = 0
): array {
    $query = Capsule::table("mod_user_activity_log")
        ->where("user_id", $userId);
    
    if ($category) {
        $query->where("category", $category);
    }
    
    if ($startDate) {
        $query->where("created_at", ">=", $startDate);
    }
    
    if ($endDate) {
        $query->where("created_at", "<=", $endDate);
    }
    
    $total = $query->count();
    
    $activities = $query
        ->orderBy("created_at", "desc")
        ->limit($limit)
        ->offset($offset)
        ->get();
    
    return [
        "total" => $total,
        "activities" => $activities,
        "has_more" => ($offset + $limit) < $total
    ];
}

function getRecentActivity(int $userId, int $limit = 10): array
{
    return Capsule::table("mod_user_activity_log")
        ->where("user_id", $userId)
        ->orderBy("created_at", "desc")
        ->limit($limit)
        ->get()
        ->map(function($item) {
            $item->details = json_decode($item->details, true);
            return $item;
        })
        ->toArray();
}
```

### Activity Cleanup

```php
add_hook("DailyCronJob", 1, function(array $params) {
    $retentionDays = getConfig("activity_log_retention_days", 90);
    $cutoffDate = date("Y-m-d H:i:s", strtotime("-{$retentionDays} days"));
    
    $deleted = Capsule::table("mod_user_activity_log")
        ->where("created_at", "<", $cutoffDate)
        ->whereNotIn("category", ["billing"]) // Always keep billing records
        ->delete();
    
    logActivity("Cleaned up {$deleted} old activity log entries");
    
    return $params;
});
```

### Activity Export

```php
function exportUserActivity(int $userId, string $format = "csv"): string
{
    $activities = getUserActivity($userId);
    
    switch ($format) {
        case "csv":
            return generateActivityCSV($activities["activities"]);
        case "json":
            return json_encode($activities["activities"]);
        default:
            throw new Exception("Unsupported format");
    }
}

function generateActivityCSV(array $activities): string
{
    $output = fopen("php://temp", "r+");
    
    // Header
    fputcsv($output, ["Date", "Time", "Category", "Action", "Details", "IP Address"]);
    
    foreach ($activities as $activity) {
        $details = is_array($activity->details) 
            ? json_encode($activity->details) 
            : $activity->details;
        
        fputcsv($output, [
            date("Y-m-d", strtotime($activity->created_at)),
            date("H:i:s", strtotime($activity->created_at)),
            $activity->category,
            $activity->action,
            $details,
            $activity->ip_address
        ]);
    }
    
    rewind($output);
    $csv = stream_get_contents($output);
    fclose($output);
    
    return $csv;
}
```

## Activity Log Template

```smarty
<!-- /templates/clientarea_activity.tpl -->
<div class="activity-log-container">
    <h2>Activity History</h2>
    
    <div class="activity-filters">
        <select name="category" class="form-control">
            <option value="">All Categories</option>
            <option value="account">Account</option>
            <option value="billing">Billing</option>
            <option value="services">Services</option>
            <option value="support">Support</option>
        </select>
        
        <input type="date" name="start_date" class="form-control">
        <input type="date" name="end_date" class="form-control">
        
        <button type="submit" class="btn btn-secondary">Filter</button>
    </div>
    
    <div class="activity-list">
        {foreach $activities as $activity}
            <div class="activity-item">
                <div class="activity-icon">
                    {switch $activity.category}
                        {case "account"}<i class="fa fa-user"></i>{/case}
                        {case "billing"}<i class="fa fa-credit-card"></i>{/case}
                        {case "services"}<i class="fa fa-server"></i>{/case}
                        {case "support"}<i class="fa fa-ticket"></i>{/case}
                        {default}<i class="fa fa-info-circle"></i>
                    {/switch}
                </div>
                <div class="activity-content">
                    <div class="activity-header">
                        <strong>{$activity->action|ucwords|replace:'_':' '}</strong>
                        <span class="activity-time">
                            {$activity->created_at}
                        </span>
                    </div>
                    <div class="activity-details">
                        {if $activity->details}
                            {foreach $activity->details as $key => $value}
                                <span class="detail-item">
                                    {$key}: {$value}
                                </span>
                            {/foreach}
                        {/if}
                    </div>
                    <div class="activity-meta">
                        <span class="ip-address">IP: {$activity->ip_address}</span>
                    </div>
                </div>
            </div>
        {/foreach}
    </div>
    
    {if $has_more}
        <a href="?page={$page + 1}" class="btn btn-secondary">
            Load More
        </a>
    {/if}
    
    <div class="activity-export">
        <a href="export_activity.php?format=csv" class="btn btn-outline">
            <i class="fa fa-download"></i> Export CSV
        </a>
    </div>
</div>
```

## Database Schema

```php
Capsule::schema()->create('mod_user_activity_log', function($t) {
    $t->increments('id');
    $t->integer('user_id');
    $t->string('action', 100);
    $t->string('category', 50);
    $t->text('details')->nullable();
    $t->string('ip_address', 45);
    $t->string('user_agent', 500)->nullable();
    $t->timestamp('created_at');
    
    $t->index(['user_id', 'created_at']);
    $t->index(['category']);
    $t->index(['created_at']);
});
```

## Best Practices

1. **Comprehensive Tracking**: Log all significant user actions
2. **Structured Data**: Use consistent categories and actions
3. **Contextual Details**: Include relevant details for each action
4. **IP Tracking**: Record IP addresses for security
5. **Retention Policy**: Define and enforce log retention
6. **Performance**: Use efficient indexing and archiving
7. **Privacy**: Allow users to view and export their data
8. **Compliance**: Support GDPR data subject requests
