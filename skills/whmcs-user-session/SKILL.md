# WHMCS User Session Management

## Overview
Guide for implementing custom session management in WHMCS. Covers session handling, timeout, and security features.

## Session Management

### Session Configuration

```php
<?php
// /includes/hooks/session_management.php

// Custom session handler
add_hook("SessionStart", 1, function(array $params) {
    // Set session parameters
    ini_set("session.cookie_httponly", 1);
    ini_set("session.cookie_secure", isset($_SERVER["HTTPS"]));
    ini_set("session.use_strict_mode", 1);
    ini_set("session.gc_maxlifetime", 3600); // 1 hour
    
    // Regenerate session ID periodically
    if (!isset($_SESSION["last_regeneration"])) {
        $_SESSION["last_regeneration"] = time();
    } elseif (time() - $_SESSION["last_regeneration"] > 300) {
        session_regenerate_id(true);
        $_SESSION["last_regeneration"] = time();
    }
    
    return $params;
});
```

### Session Validation

```php
add_hook("ValidateSession", 1, function(array $params) {
    $sessionId = session_id();
    
    // Check session in database
    $session = Capsule::table("mod_user_sessions")
        ->where("session_id", $sessionId)
        ->where("user_type", "client")
        ->first();
    
    if (!$session) {
        return ["error" => "Invalid session."];
    }
    
    // Check expiration
    if (strtotime($session->expires_at) < time()) {
        destroySession($sessionId);
        return ["error" => "Session expired."];
    }
    
    // Check IP consistency
    if ($session->ip_address !== $_SERVER["REMOTE_ADDR"]) {
        // Log potential session hijacking
        Capsule::table("mod_session_events")->insert([
            "session_id" => $sessionId,
            "event_type" => "ip_mismatch",
            "original_ip" => $session->ip_address,
            "current_ip" => $_SERVER["REMOTE_ADDR"],
            "created_at" => date("Y-m-d H:i:s")
        ]);
        
        // Optionally invalidate session
        if (getConfig("strict_ip_check")) {
            destroySession($sessionId);
            return ["error" => "Session invalidated due to IP change."];
        }
    }
    
    // Update last activity
    Capsule::table("mod_user_sessions")
        ->where("id", $session->id)
        ->update(["last_activity" => date("Y-m-d H:i:s")]);
    
    return ["success" => true];
});
```

### Create Session

```php
add_hook("ClientLogin", 1, function(array $params) {
    $clientId = $params["userid"];
    $sessionId = session_id();
    
    // Create session record
    Capsule::table("mod_user_sessions")->insert([
        "session_id" => $sessionId,
        "user_id" => $clientId,
        "user_type" => "client",
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "created_at" => date("Y-m-d H:i:s"),
        "last_activity" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+24 hours")),
        "remember_me" => isset($_POST["rememberme"]) ? 1 : 0
    ]);
    
    // Set session variables
    $_SESSION["user_id"] = $clientId;
    $_SESSION["login_time"] = time();
    $_SESSION["session_created"] = date("Y-m-d H:i:s");
    
    return $params;
});
```

### Destroy Session

```php
function destroySession(string $sessionId): void
{
    // Remove from database
    Capsule::table("mod_user_sessions")
        ->where("session_id", $sessionId)
        ->delete();
    
    // Clear session
    session_unset();
    session_destroy();
    
    // Delete session cookie
    if (isset($_COOKIE[session_name()])) {
        setcookie(session_name(), "", time() - 3600, "/");
    }
}

add_hook("ClientLogout", 1, function(array $params) {
    $sessionId = session_id();
    
    // Mark session as logged out (for audit)
    Capsule::table("mod_user_sessions")
        ->where("session_id", $sessionId)
        ->update([
            "logged_out_at" => date("Y-m-d H:i:s"),
            "active" => 0
        ]);
    
    destroySession($sessionId);
    
    return $params;
});
```

### Session Cleanup

```php
// Cron hook for session cleanup
add_hook("DailyCronJob", 1, function(array $params) {
    $deleted = Capsule::table("mod_user_sessions")
        ->where("expires_at", "<", date("Y-m-d H:i:s"))
        ->whereOr("last_activity", "<", date("Y-m-d H:i:s", strtotime("-7 days")))
        ->delete();
    
    logActivity("Cleaned up {$deleted} expired sessions");
    
    return $params;
});

// Cleanup old anonymous sessions
function cleanupAnonymousSessions(): int
{
    return Capsule::table("mod_anonymous_sessions")
        ->where("created_at", "<", date("Y-m-d H:i:s", strtotime("-1 hour")))
        ->delete();
}
```

### Remember Me Feature

```php
add_hook("RememberClientLogin", 1, function(array $params) {
    $clientId = $params["client_id"];
    $selector = bin2hex(random_bytes(16));
    $token = bin2hex(random_bytes(32));
    $hashedToken = hash("sha256", $token);
    
    // Store remember me token
    Capsule::table("mod_remember_tokens")->insert([
        "selector" => $selector,
        "token_hash" => $hashedToken,
        "client_id" => $clientId,
        "user_agent" => $_SERVER["HTTP_USER_AGENT"] ?? "",
        "ip_address" => $_SERVER["REMOTE_ADDR"],
        "created_at" => date("Y-m-d H:i:s"),
        "expires_at" => date("Y-m-d H:i:s", strtotime("+30 days"))
    ]);
    
    // Set cookie
    setcookie("remember_me", base64_encode($selector . ":" . $token), [
        "expires" => time() + (30 * 24 * 60 * 60),
        "path" => "/",
        "secure" => isset($_SERVER["HTTPS"]),
        "httponly" => true,
        "samesite" => "Lax"
    ]);
    
    return ["success" => true];
});

add_hook("ProcessRememberToken", 1, function(array $params) {
    $cookie = $_COOKIE["remember_me"] ?? "";
    $parts = explode(":", base64_decode($cookie), 2);
    
    if (count($parts) !== 2) {
        return ["error" => "Invalid remember token."];
    }
    
    [$selector, $token] = $parts;
    
    // Find token
    $record = Capsule::table("mod_remember_tokens")
        ->where("selector", $selector)
        ->where("expires_at", ">", date("Y-m-d H:i:s"))
        ->first();
    
    if (!$record || hash("sha256", $token) !== $record->token_hash) {
        // Clear invalid token
        setcookie("remember_me", "", time() - 3600, "/");
        return ["error" => "Invalid remember token."];
    }
    
    // Log user in
    $_SESSION["uid"] = $record->client_id;
    $_SESSION["uphash"] = md5($record->client_id . session_id());
    
    // Refresh token
    add_hook("RememberClientLogin", 1, function($params) use ($record) {
        // Token will be refreshed on next login
    });
    
    return [
        "success" => true,
        "client_id" => $record->client_id
    ];
});
```

## Best Practices

1. **Secure Cookies**: HttpOnly, Secure, SameSite flags
2. **Session Regeneration**: Regenerate ID after login
3. **Timeout**: Set appropriate session expiration
4. **IP Validation**: Check IP consistency (with flexibility)
5. **Remember Me**: Secure token-based persistence
6. **Cleanup**: Regular cleanup of expired sessions
7. **Activity Tracking**: Update last activity timestamp
8. **Multiple Sessions**: Support concurrent session limits
9. **Audit Logging**: Log all session events
