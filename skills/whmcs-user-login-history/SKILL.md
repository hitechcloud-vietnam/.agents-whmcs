# WHMCS User Login History

## Overview
Guide for implementing comprehensive login history tracking for WHMCS users. Covers login logging, failed attempt tracking, and audit reports.

## Login History System

### Log Login Attempts

```php
<?php
// /includes/hooks/login_history.php

add_hook("ClientLogin", 1, function(array $params) {
    $userId = $params["userid"];
    
    Capsule::table("mod_login_history")->insert([
        "user_id" => $userId,
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "browser" => parseBrowser(),
        "os" => parseOS(),
        "location" => getLocationFromIP($_SERVER["REMOTE_ADDR"]),
        "success" => 1,
        "login_time" => date("Y-m-d H:i:s"),
        "session_id" => session_id()
    ]);
    
    // Update last login
    Capsule::table("tblclients")
        ->where("id", $userId)
        ->update([
            "lastlogin" => date("Y-m-d H:i:s"),
            "lastloginip" => $_SERVER["REMOTE_ADDR"]
        ]);
    
    return $params;
});

add_hook("FailedLoginAttempt", 1, function(array $params) {
    Capsule::table("mod_login_history")->insert([
        "user_id" => null,
        "email" => $params["username"],
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "browser" => parseBrowser(),
        "os" => parseOS(),
        "location" => getLocationFromIP($_SERVER["REMOTE_ADDR"]),
        "success" => 0,
        "failure_reason" => $params["reason"] ?? "Invalid credentials",
        "login_time" => date("Y-m-d H:i:s")
    ]);
    
    // Track failed attempts for rate limiting
    trackFailedAttempt($_SERVER["REMOTE_ADDR"]);
    
    return $params;
});

function trackFailedAttempt(string $ipAddress): void
{
    $recentAttempts = Capsule::table("mod_failed_login_attempts")
        ->where("ip_address", $ipAddress)
        ->where("attempt_time", ">", date("Y-m-d H:i:s", strtotime("-15 minutes")))
        ->count();
    
    if ($recentAttempts >= 5) {
        // Block IP temporarily
        Capsule::table("mod_blocked_ips")->insert([
            "ip_address" => $ipAddress,
            "reason" => "Too many failed login attempts",
            "blocked_until" => date("Y-m-d H:i:s", strtotime("+30 minutes")),
            "created_at" => date("Y-m-d H:i:s")
        ]);
    }
}
```

### Get Login History

```php
function getLoginHistory(
    int $userId,
    ?string $startDate = null,
    ?string $endDate = null,
    bool $successfulOnly = false,
    int $limit = 50,
    int $offset = 0
): array {
    $query = Capsule::table("mod_login_history")
        ->where("user_id", $userId);
    
    if ($startDate) {
        $query->where("login_time", ">=", $startDate);
    }
    
    if ($endDate) {
        $query->where("login_time", "<=", $endDate);
    }
    
    if ($successfulOnly) {
        $query->where("success", 1);
    }
    
    $total = $query->count();
    
    $history = $query
        ->orderBy("login_time", "desc")
        ->limit($limit)
        ->offset($offset)
        ->get();
    
    return [
        "total" => $total,
        "history" => $history,
        "has_more" => ($offset + $limit) < $total
    ];
}

function getLoginSummary(int $userId, int $days = 30): array
{
    $startDate = date("Y-m-d H:i:s", strtotime("-{$days} days"));
    
    $successful = Capsule::table("mod_login_history")
        ->where("user_id", $userId)
        ->where("success", 1)
        ->where("login_time", ">=", $startDate)
        ->count();
    
    $failed = Capsule::table("mod_login_history")
        ->where("user_id", $userId)
        ->where("success", 0)
        ->where("login_time", ">=", $startDate)
        ->count();
    
    $uniqueIPs = Capsule::table("mod_login_history")
        ->where("user_id", $userId)
        ->where("success", 1)
        ->where("login_time", ">=", $startDate)
        ->distinct("ip_address")
        ->count("ip_address");
    
    $locations = Capsule::table("mod_login_history")
        ->where("user_id", $userId)
        ->where("success", 1)
        ->where("login_time", ">=", $startDate)
        ->distinct("location")
        ->count("location");
    
    return [
        "successful_logins" => $successful,
        "failed_attempts" => $failed,
        "unique_ips" => $uniqueIPs,
        "unique_locations" => $locations
    ];
}
```

### Suspicious Activity Detection

```php
function detectSuspiciousLogins(int $userId): array
{
    $suspicious = [];
    $history = getLoginHistory($userId, null, null, false, 100)["history"];
    
    // Group by IP
    $ipLogins = [];
    foreach ($history as $login) {
        if ($login->success) {
            $ipLogins[$login->ip_address][] = $login;
        }
    }
    
    foreach ($ipLogins as $ip => $logins) {
        // Check for impossible travel
        if (count($logins) >= 2) {
            for ($i = 0; $i < count($logins) - 1; $i++) {
                $current = strtotime($logins[$i]->login_time);
                $previous = strtotime($logins[$i + 1]->login_time);
                $timeDiff = $current - $previous;
                
                // If logins are less than 2 hours apart but different locations
                if ($timeDiff < 7200) {
                    $currentLoc = getCountryFromIP($logins[$i]->ip_address);
                    $prevLoc = getCountryFromIP($logins[$i + 1]->ip_address);
                    
                    if ($currentLoc !== $prevLoc && $currentLoc && $prevLoc) {
                        $suspicious[] = [
                            "type" => "impossible_travel",
                            "ip1" => $logins[$i]->ip_address,
                            "location1" => $logins[$i]->location,
                            "time1" => $logins[$i]->login_time,
                            "ip2" => $logins[$i + 1]->ip_address,
                            "location2" => $logins[$i + 1]->location,
                            "time2" => $logins[$i + 1]->login_time,
                            "severity" => "high"
                        ];
                    }
                }
            }
        }
    }
    
    // Check for many failed attempts
    $failedLogins = array_filter($history, fn($h) => !$h->success);
    if (count($failedLogins) > 10) {
        $suspicious[] = [
            "type" => "multiple_failed_attempts",
            "count" => count($failedLogins),
            "severity" => "medium"
        ];
    }
    
    return $suspicious;
}
```

## Login History Template

```smarty
<!-- /templates/clientarea_login_history.tpl -->
<div class="login-history-container">
    <h2>Login History</h2>
    
    <div class="history-summary">
        <div class="summary-card">
            <h4>Last 30 Days</h4>
            <div class="stats">
                <div class="stat">
                    <strong>{$summary.successful_logins}</strong>
                    <span>Successful Logins</span>
                </div>
                <div class="stat">
                    <strong>{$summary.failed_attempts}</strong>
                    <span>Failed Attempts</span>
                </div>
                <div class="stat">
                    <strong>{$summary.unique_ips}</strong>
                    <span>Unique IPs</span>
                </div>
            </div>
        </div>
    </div>
    
    {if $suspicious}
        <div class="alert alert-warning">
            <h4><i class="fa fa-exclamation-triangle"></i> Suspicious Activity Detected</h4>
            {foreach $suspicious as $alert}
                <div class="alert-item">
                    <strong>{$alert.type|ucwords|replace:'_':' '}</strong>
                    {if $alert.type eq 'impossible_travel'}
                        <p>Login from {$alert.location1} at {$alert.time1} 
                           and {$alert.location2} at {$alert.time2}</p>
                    {/if}
                </div>
            {/foreach}
        </div>
    {/if}
    
    <div class="history-filters">
        <form method="get" class="filter-form">
            <input type="hidden" name="action" value="login_history">
            <input type="date" name="start_date" value="{$filters.start_date}">
            <input type="date" name="end_date" value="{$filters.end_date}">
            <label>
                <input type="checkbox" name="successful_only" value="1"
                       {if $filters.successful_only}checked{/if}>
                Successful logins only
            </label>
            <button type="submit" class="btn btn-secondary">Filter</button>
        </form>
    </div>
    
    <table class="login-history-table">
        <thead>
            <tr>
                <th>Date & Time</th>
                <th>Status</th>
                <th>IP Address</th>
                <th>Location</th>
                <th>Browser / OS</th>
                <th>Details</th>
            </tr>
        </thead>
        <tbody>
            {foreach $history as $login}
                <tr class="{if !$login->success}failed{/if}">
                    <td>{$login->login_time}</td>
                    <td>
                        {if $login->success}
                            <span class="badge badge-success">Success</span>
                        {else}
                            <span class="badge badge-danger">Failed</span>
                            <br><small>{$login->failure_reason}</small>
                        {/if}
                    </td>
                    <td>{$login->ip_address}</td>
                    <td>{$login->location ?? 'Unknown'}</td>
                    <td>{$login->browser} / {$login->os}</td>
                    <td>
                        {if !$login->success && $login->email}
                            <small>Trying: {$login->email}</small>
                        {/if}
                    </td>
                </tr>
            {/foreach}
        </tbody>
    </table>
    
    {if $has_more}
        <div class="pagination">
            <a href="?page={$page + 1}" class="btn btn-secondary">
                Load More
            </a>
        </div>
    {/if}
</div>
```

## Best Practices

1. **Comprehensive Logging**: Log all login attempts (success and failure)
2. **Location Detection**: Track IP-based location information
3. **Device Info**: Record browser and OS details
4. **Failed Attempt Tracking**: Monitor and limit failed attempts
5. **Suspicious Activity**: Detect impossible travel and anomalies
6. **User Alerts**: Notify users of suspicious activity
7. **Retention Policy**: Define and enforce log retention
8. **Export**: Allow users to export their login history
