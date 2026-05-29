# WHMCS Admin Session Management

## Overview
Guide for implementing admin session management in WHMCS. Covers session tracking, timeout handling, and concurrent session control.

## Session Management

### Track Admin Sessions

```php
<?php
// /includes/hooks/admin_session_mgmt.php

add_hook("AdminSessionStart", 1, function(array $params) {
    $adminId = $params["admin_id"];
    $sessionId = session_id();
    
    // Store session in database
    Capsule::table("mod_admin_sessions")->insert([
        "session_id" => $sessionId,
        "admin_id" => $adminId,
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "created_at" => date("Y-m-d H:i:s"),
        "last_activity" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+2 hours"))
    ]);
    
    return $params;
});

add_hook("AdminSessionActivity", 1, function(array $params) {
    $sessionId = session_id();
    
    Capsule::table("mod_admin_sessions")
        ->where("session_id", $sessionId)
        ->update([
            "last_activity" => date("Y-m-d H:i:s")
        ]);
});
```

### Session Validation

```php
function validateAdminSession(string $sessionId): ?array
{
    $session = Capsule::table("mod_admin_sessions")
        ->where("session_id", $sessionId)
        ->where("active", 1)
        ->first();
    
    if (!$session) {
        return null;
    }
    
    // Check expiration
    if (strtotime($session->expires_at) < time()) {
        expireSession($sessionId);
        return null;
    }
    
    // Check IP consistency
    if ($session->ip_address !== $_SERVER["REMOTE_ADDR"]) {
        // Log potential security issue but don't block
        Capsule::table("mod_admin_session_events")->insert([
            "session_id" => $sessionId,
            "event_type" => "ip_mismatch",
            "ip_before" => $session->ip_address,
            "ip_after" => $_SERVER["REMOTE_ADDR"],
            "created_at" => date("Y-m-d H:i:s")
        ]);
    }
    
    return [
        "admin_id" => $session->admin_id,
        "session_id" => $sessionId,
        "last_activity" => $session->last_activity
    ];
}
```

### Concurrent Session Control

```php
function checkConcurrentSessions(int $adminId): array
{
    $sessions = Capsule::table("mod_admin_sessions")
        ->where("admin_id", $adminId)
        ->where("active", 1)
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->get();
    
    return $sessions->toArray();
}

function limitConcurrentSessions(int $adminId, int $maxSessions = 3): bool
{
    $sessions = checkConcurrentSessions($adminId);
    
    if (count($sessions) >= $maxSessions) {
        // Keep most recent, expire others
        usort($sessions, function($a, $b) {
            return strtotime($b->last_activity) - strtotime($a->last_activity);
        });
        
        $toExpire = array_slice($sessions, $maxSessions);
        foreach ($toExpire as $session) {
            expireSession($session->session_id);
        }
        
        return true;
    }
    
    return false;
}
```

### Session Cleanup

```php
function cleanupExpiredSessions(): int
{
    $deleted = Capsule::table("mod_admin_sessions")
        ->where("expires_at", "<", date("Y-m-d H:i:s"))
        ->delete();
    
    return $deleted;
}

add_hook("DailyCronJob", 1, function(array $params) {
    $cleaned = cleanupExpiredSessions();
    logActivity("Cleaned up {$cleaned} expired admin sessions");
    
    return $params;
});
```

## Session Management Template

```smarty
<!-- /admin/templates/admin_sessions.tpl -->
<div class="admin-sessions-container">
    <h2>Active Sessions</h2>
    
    <div class="sessions-info">
        <p>You are currently logged in from this device.</p>
    </div>
    
    <table class="table sessions-table">
        <thead>
            <tr>
                <th>Device</th>
                <th>IP Address</th>
                <th>Location</th>
                <th>Last Activity</th>
                <th>Status</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {foreach $sessions as $session}
                <tr class="{if $session.session_id eq $current_session}current{/if}">
                    <td>
                        <i class="fa fa-{if strpos($session.user_agent, 'Mobile')}mobile{else}desktop{/if}"></i>
                        {if $session.session_id eq $current_session}
                            <strong>This device</strong>
                        {else}
                            {$session.user_agent|truncate:50}
                        {/if}
                    </td>
                    <td>{$session.ip_address}</td>
                    <td>{$session.location|default:'Unknown'}</td>
                    <td>{$session.last_activity}</td>
                    <td>
                        <span class="badge badge-success">Active</span>
                    </td>
                    <td>
                        {if $session.session_id neq $current_session}
                            <a href="?revoke={$session.session_id}" class="btn btn-sm btn-danger">
                                Revoke
                            </a>
                        {/if}
                    </td>
                </tr>
            {/foreach}
        </tbody>
    </table>
    
    <div class="session-actions">
        <a href="?revoke_all=1" class="btn btn-outline" 
           onclick="return confirm('Revoke all other sessions?');">
            Revoke All Other Sessions
        </a>
    </div>
</div>
```

## Best Practices

1. **Session Timeout**: Implement reasonable timeout
2. **Activity Tracking**: Update last activity on each request
3. **IP Validation**: Check IP consistency
4. **Concurrent Limits**: Limit concurrent sessions per admin
5. **Cleanup**: Regular cleanup of expired sessions
6. **Audit Trail**: Log session events
7. **Remember Me**: Secure implementation
8. **Revocation**: Allow session revocation
