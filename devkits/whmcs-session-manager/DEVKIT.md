# WHMCS Session Manager Module

Session management with monitoring, analytics, and security.

## Features

- Session monitoring and tracking
- Session timeout management
- Session storage (database, Redis, file)
- Concurrent session limits
- Session encryption
- Session analytics
- User session list
- Force logout capability
- Remember me functionality
- Cross-device session sync
- Session activity logging
- Idle detection
- Session invalidation
- Bulk session management
- Session sharing (team features)

## Installation

1. Copy module to `/path/to/whmcs/modules/addons/sessionmanager/`
2. Activate in WHMCS Admin > System Settings > Module Addons
3. Configure session settings

## Usage

```php
// Create new session
$result = sessionmanager_CreateSession($userId, array(
    'ip_address' => '192.168.1.1',
    'user_agent' => 'Mozilla/5.0',
    'expires_in' => 3600,
    'remember_me' => false
));
// Returns: session_id, token, expires_at

// Validate session
$result = sessionmanager_ValidateSession($sessionId, $token);
if (!$result['valid']) {
    echo "Session invalid or expired";
}

// Extend session
sessionmanager_ExtendSession($sessionId, 7200);

// Get user sessions
$sessions = sessionmanager_GetUserSessions($userId);
// Returns: active_sessions, sessions[]

// Force logout
sessionmanager_ForceLogout($sessionId);

// Force logout all user sessions
sessionmanager_ForceLogoutAll($userId);

// Set concurrent session limit
sessionmanager_SetConcurrentLimit($userId, 3);

// Get session analytics
$analytics = sessionmanager_GetAnalytics($userId, 30);
// Returns: total_logins, avg_duration, favorite_devices, peak_hours

// Get session history
$history = sessionmanager_GetHistory($userId, 30);

// Get active session count
$count = sessionmanager_GetActiveCount($userId);

// Set session timeout
sessionmanager_SetTimeout($timeoutSeconds);

// Delete expired sessions
$deleted = sessionmanager_CleanExpired();

// Get session by ID
$session = sessionmanager_GetSession($sessionId);

// Share session access
sessionmanager_ShareSession($sessionId, $sharedWithUserId);

// Get shared sessions
$shared = sessionmanager_GetSharedSessions($userId);

// Revoke shared access
sessionmanager_RevokeShared($shareId);

// Block user device
sessionmanager_BlockDevice($userId, $deviceId);

// Get blocked devices
$blocked = sessionmanager_GetBlockedDevices($userId);

// Unblock device
sessionmanager_UnblockDevice($deviceId);
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| StorageType | dropdown | database | Session storage method |
| SessionTimeout | text | 1800 | Default timeout (seconds) |
| RememberMeDuration | text | 2592000 | Remember me (30 days) |
| MaxConcurrentSessions | text | 5 | Max concurrent sessions |
| EnableEncryption | yesno | yes | Encrypt session data |
| EnableIPTracking | yesno | yes | Track IP addresses |
| EnableDeviceTracking | yesno | yes | Track devices |
| ForceLogoutOnPasswordChange | yesno | yes | Logout on password change |
| IdleTimeout | text | 600 | Idle timeout (seconds) |
| RedisHost | text | localhost | Redis host |
| RedisPort | text | 6379 | Redis port |
| RedisPassword | password | - | Redis password |

## Storage Types

| Type | Description |
|------|-------------|
| database | WHMCS database storage |
| redis | Redis cache storage |
| file | File-based storage |

## Session Events

| Event | Description |
|-------|-------------|
| created | New session created |
| validated | Session validated |
| extended | Session extended |
| expired | Session expired |
| logout | User logged out |
| forced_logout | Force logout by admin |
| concurrent_limit | Concurrent limit reached |
| idle_timeout | Idle timeout reached |

## Database Tables

- `mod_sessionmanager_sessions` - Active sessions
- `mod_sessionmanager_history` - Session history
- `mod_sessionmanager_devices` - Known devices
- `mod_sessionmanager_sharing` - Shared sessions
- `mod_sessionmanager_blocks` - Blocked devices

## API Functions

| Function | Description |
|----------|-------------|
| `sessionmanager_CreateSession()` | Create new session |
| `sessionmanager_ValidateSession()` | Validate session |
| `sessionmanager_ExtendSession()` | Extend session |
| `sessionmanager_GetUserSessions()` | Get user sessions |
| `sessionmanager_GetSession()` | Get session by ID |
| `sessionmanager_GetActiveCount()` | Get active count |
| `sessionmanager_ForceLogout()` | Logout session |
| `sessionmanager_ForceLogoutAll()` | Logout all user sessions |
| `sessionmanager_SetConcurrentLimit()` | Set concurrent limit |
| `sessionmanager_GetAnalytics()` | Get analytics |
| `sessionmanager_GetHistory()` | Get session history |
| `sessionmanager_SetTimeout()` | Set timeout |
| `sessionmanager_CleanExpired()` | Clean expired sessions |
| `sessionmanager_ShareSession()` | Share session |
| `sessionmanager_GetSharedSessions()` | Get shared |
| `sessionmanager_RevokeShared()` | Revoke sharing |
| `sessionmanager_BlockDevice()` | Block device |
| `sessionmanager_GetBlockedDevices()` | Get blocked devices |
| `sessionmanager_UnblockDevice()` | Unblock device |
