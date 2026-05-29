# WHMCS Client Hooks

## Overview

Client hooks allow you to execute custom code during client-related operations in WHMCS.

## Available Client Hooks

### Client Created Hook

```php
<?php
// Triggered when a new client is created
add_hook('ClientCreated', 1, function(array $vars) {
    $clientId = $vars['client_id'];
    $email = $vars['email'];
    $firstName = $vars['firstname'];
    $lastName = $vars['lastname'];
    
    // Send welcome email
    sendWelcomeEmail($clientId, $email);
    
    // Add to CRM
    syncToCrm($vars);
    
    // Create default notes
    createDefaultClientNotes($clientId);
    
    return [
        'success' => true,
        'client_id' => $clientId,
    ];
});
```

### Client Updated Hook

```php
<?php
// Triggered when client details are updated
add_hook('ClientUpdated', 1, function(array $vars) {
    $clientId = $vars['client_id'];
    $changes = $vars['changes'] ?? [];
    
    // Log changes for audit
    logClientChanges($clientId, $changes);
    
    // Update external systems
    if (isset($changes['email'])) {
        updateExternalEmail($clientId, $changes['email']);
    }
    
    // Trigger notifications
    if (isset($changes['status'])) {
        notifyStatusChange($clientId, $changes['status']);
    }
    
    return ['success' => true];
});

function logClientChanges(int $clientId, array $changes): void
{
    $logData = [
        'client_id' => $clientId,
        'changes' => json_encode($changes),
        'timestamp' => date('Y-m-d H:i:s'),
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
    ];
    
    Capsule::table('mod_client_change_logs')->insert($logData);
}
```

### Client Deleted Hook

```php
<?php
// Triggered when a client is deleted
add_hook('ClientDelete', 1, function(array $vars) {
    $clientId = $vars['client_id'];
    
    // Archive client data for compliance
    archiveClientData($clientId);
    
    // Remove from external systems
    removeFromCrm($clientId);
    
    // Cancel pending services first
    cancelPendingServices($clientId);
    
    // Log deletion for audit
    logClientDeletion($clientId);
    
    return ['success' => true, 'client_id' => $clientId];
});

function archiveClientData(int $clientId): void
{
    $client = Capsule::table('tblclients')
        ->where('id', $clientId)
        ->first();
    
    if ($client) {
        Capsule::table('mod_client_archive')->insert([
            'original_id' => $clientId,
            'archived_data' => json_encode($client),
            'deleted_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

### Client Login Hook

```php
<?php
// Triggered on client login
add_hook('ClientLogin', 1, function(array $vars) {
    $clientId = $vars['user_id'];
    
    // Update last login
    updateLastLogin($clientId);
    
    // Check for pending actions
    $pendingActions = checkPendingActions($clientId);
    if (!empty($pendingActions)) {
        // Store for display after redirect
        $_SESSION['pending_actions'] = $pendingActions;
    }
    
    // Track analytics
    trackLoginAnalytics($clientId);
    
    // Check for suspicious activity
    checkLoginSecurity($clientId);
    
    return ['success' => true];
});

function updateLastLogin(int $clientId): void
{
    Capsule::table('tblclients')
        ->where('id', $clientId)
        ->update([
            'lastlogin' => date('Y-m-d H:i:s'),
            'lastloginip' => $_SERVER['REMOTE_ADDR'] ?? null,
        ]);
}

function checkLoginSecurity(int $clientId): bool
{
    $currentIp = $_SERVER['REMOTE_ADDR'] ?? '';
    
    // Check for login from new location
    $recentLogins = Capsule::table('tblactivitylog')
        ->where('userid', $clientId)
        ->where('description', 'like', '%Login%')
        ->orderBy('id', 'desc')
        ->limit(5)
        ->pluck('ipaddr')
        ->toArray();
    
    if (!empty($recentLogins) && !in_array($currentIp, $recentLogins)) {
        // New location detected - could send alert
        sendSecurityAlert($clientId, $currentIp);
    }
    
    return true;
}
```

### Client Logout Hook

```php
<?php
// Triggered on client logout
add_hook('ClientLogout', 1, function(array $vars) {
    $clientId = $vars['user_id'] ?? 0;
    
    // Clear session data
    clearClientSession($clientId);
    
    // Log activity
    logLogoutActivity($clientId);
    
    // Update online status
    updateOnlineStatus($clientId, false);
    
    return ['success' => true];
});
```

### Client Password Change Hook

```php
<?php
// Triggered when client changes password
add_hook('ClientPasswordChange', 1, function(array $vars) {
    $clientId = $vars['client_id'];
    $newPasswordHash = $vars['password_hash'] ?? '';
    
    // Update external systems
    syncPasswordToExternal($clientId, $vars['email']);
    
    // Invalidate other sessions
    invalidateOtherSessions($clientId);
    
    // Send confirmation email
    sendPasswordChangeConfirmation($clientId);
    
    // Log for security audit
    logPasswordChange($clientId);
    
    return ['success' => true];
});

function logPasswordChange(int $clientId): void
{
    Capsule::table('mod_security_logs')->insert([
        'client_id' => $clientId,
        'action' => 'password_change',
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? 'unknown',
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Client Merge Hook

```php
<?php
// Triggered when merging clients
add_hook('ClientMerge', 1, function(array $vars) {
    $sourceId = $vars['source_client_id'];
    $targetId = $vars['target_client_id'];
    
    // Transfer all services
    transferServices($sourceId, $targetId);
    
    // Transfer invoices
    transferInvoices($sourceId, $targetId);
    
    // Merge notes
    mergeNotes($sourceId, $targetId);
    
    // Archive source client
    archiveMergedClient($sourceId);
    
    return ['success' => true];
});
```

### Pre-Client Validation Hook

```php
<?php
// Run before client operations (validation)
add_hook('preCreateClient', 1, function(array $vars) {
    $errors = [];
    
    // Custom validation - email domain blacklist
    $emailDomain = substr(strrchr($vars['email'], '@'), 1);
    $blockedDomains = getBlockedEmailDomains();
    
    if (in_array($emailDomain, $blockedDomains)) {
        $errors[] = 'Email domain is not allowed';
    }
    
    // Check for duplicate email
    if (emailExists($vars['email'])) {
        $errors[] = 'Email address already registered';
    }
    
    // Custom field validation
    foreach ($vars['customfields'] ?? [] as $fieldId => $value) {
        if (!validateCustomField($fieldId, $value)) {
            $errors[] = "Invalid value for custom field";
        }
    }
    
    if (!empty($errors)) {
        return [
            'abort' => false, // false = continue but show warnings
            'warnings' => $errors,
        ];
    }
    
    return ['abort' => false];
});
```

## Comprehensive Hook Handler

```php
<?php
class ClientHookHandler {
    private array $hooks = [];
    
    public function register(): void
    {
        // Client Created
        add_hook('ClientCreated', 1, [$this, 'handleClientCreated']);
        
        // Client Updated
        add_hook('ClientUpdated', 1, [$this, 'handleClientUpdated']);
        
        // Client Deleted
        add_hook('ClientDelete', 1, [$this, 'handleClientDeleted']);
        
        // Client Login
        add_hook('ClientLogin', 1, [$this, 'handleClientLogin']);
        
        // Client Logout
        add_hook('ClientLogout', 1, [$this, 'handleClientLogout']);
        
        // Password Change
        add_hook('ClientPasswordChange', 1, [$this, 'handlePasswordChange']);
    }
    
    public function handleClientCreated(array $vars): array
    {
        // Log the creation
        $this->logActivity('client_created', $vars['client_id'], $vars);
        
        // Sync to CRM
        $this->syncToCrm($vars);
        
        // Send welcome sequence
        $this->triggerWelcomeSequence($vars['client_id']);
        
        // Setup default preferences
        $this->setupDefaultPreferences($vars['client_id']);
        
        return ['success' => true];
    }
    
    public function handleClientUpdated(array $vars): array
    {
        $this->logActivity('client_updated', $vars['client_id'], $vars);
        
        // Sync changes
        if (!empty($vars['changes'])) {
            $this->syncChangesToCrm($vars['client_id'], $vars['changes']);
        }
        
        return ['success' => true];
    }
    
    public function handleClientDeleted(array $vars): array
    {
        $this->archiveClient($vars['client_id']);
        $this->logActivity('client_deleted', $vars['client_id'], $vars);
        
        return ['success' => true];
    }
    
    public function handleClientLogin(array $vars): array
    {
        $this->updateLoginMetrics($vars['user_id']);
        $this->checkAccountStatus($vars['user_id']);
        
        return ['success' => true];
    }
    
    public function handleClientLogout(array $vars): array
    {
        $this->logActivity('client_logout', $vars['user_id'] ?? 0);
        
        return ['success' => true];
    }
    
    public function handlePasswordChange(array $vars): array
    {
        $this->logSecurityEvent('password_changed', $vars['client_id']);
        $this->invalidateSessions($vars['client_id']);
        
        return ['success' => true];
    }
    
    private function logActivity(string $action, int $clientId, array $data = []): void
    {
        Capsule::table('mod_activity_log')->insert([
            'action' => $action,
            'client_id' => $clientId,
            'data' => json_encode($data),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function logSecurityEvent(string $event, int $clientId): void
    {
        Capsule::table('mod_security_log')->insert([
            'event' => $event,
            'client_id' => $clientId,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function syncToCrm(array $vars): void
    {
        // CRM sync implementation
    }
    
    private function syncChangesToCrm(int $clientId, array $changes): void
    {
        // Sync specific changes
    }
    
    private function triggerWelcomeSequence(int $clientId): void
    {
        // Queue welcome emails
    }
    
    private function setupDefaultPreferences(int $clientId): void
    {
        Capsule::table('mod_client_preferences')->insert([
            'client_id' => $clientId,
            'language' => 'english',
            'email_notifications' => true,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function archiveClient(int $clientId): void
    {
        // Archive client data
    }
    
    private function updateLoginMetrics(int $clientId): void
    {
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update([
                'lastlogin' => date('Y-m-d H:i:s'),
            ]);
    }
    
    private function checkAccountStatus(int $clientId): void
    {
        // Check for issues
    }
    
    private function invalidateSessions(int $clientId): void
    {
        // Invalidate other sessions
    }
}

// Register hooks
$handler = new ClientHookHandler();
$handler->register();
```

## Hook Priority

Hooks are executed in priority order (lower numbers run first):

| Priority | Use Case |
|----------|----------|
| 1-10 | Core validations and checks |
| 11-50 | Data preparation and transformation |
| 51-100 | External integrations |
| 101+ | Logging and notifications |

## Best Practices

1. **Return arrays** - Always return an array from hook functions
2. **Handle errors gracefully** - Use try-catch blocks
3. **Check required vars** - Validate all required variables exist
4. **Log hook execution** - Track hook performance
5. **Avoid infinite loops** - Don't trigger actions that fire the same hook

## Related Documentation

- [WHMCS Admin Hooks](/docs/whmcs-admin-hooks.md)
- [WHMCS Cron Hooks](/docs/whmcs-cron-hooks.md)