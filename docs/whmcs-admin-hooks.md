# WHMCS Admin Hooks

## Overview

Admin hooks allow customization of administrative area operations including authentication, navigation, and dashboard functionality.

## Available Admin Hooks

### Admin Login Hook

```php
<?php
// Triggered on admin login
add_hook('AdminLogin', 1, function(array $vars) {
    $adminId = $vars['admin_id'];
    $username = $vars['username'];
    
    // Log login
    logAdminLogin($adminId);
    
    // Update last login
    updateLastLogin($adminId);
    
    // Check for security issues
    checkSecurityStatus($adminId);
    
    // Load preferences
    loadAdminPreferences($adminId);
    
    return ['success' => true];
});

function logAdminLogin(int $adminId): void
{
    Capsule::table('mod_admin_activity')->insert([
        'admin_id' => $adminId,
        'action' => 'login',
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}
```

### Admin Logout Hook

```php
<?php
// Triggered on admin logout
add_hook('AdminLogout', 1, function(array $vars) {
    $adminId = $vars['admin_id'];
    
    // Clear session data
    clearAdminSession($adminId);
    
    // Log logout
    logAdminLogout($adminId);
    
    return ['success' => true];
});
```

### Pre-Admin Login Hook

```php
<?php
// Runs before admin login - validation
add_hook('PreAdminLogin', 1, function(array $vars) {
    $username = $vars['username'];
    $password = $vars['password'];
    
    // Check IP whitelist
    if (!isIpWhitelisted($_SERVER['REMOTE_ADDR'])) {
        return [
            'abort' => true,
            'error_msg' => 'IP address not authorized',
        ];
    }
    
    // Check for locked account
    if (isAccountLocked($username)) {
        return [
            'abort' => true,
            'error_msg' => 'Account temporarily locked',
        ];
    }
    
    // Check 2FA if required
    if (requires2FA($username)) {
        // Set flag for 2FA prompt
        $_SESSION['require_2fa'] = true;
    }
    
    return ['abort' => false];
});
```

### Admin Authentication Hook

```php
<?php
// Custom admin authentication
add_hook('AdminAuth', 1, function(array $vars) {
    // Return ['abort' => true] to prevent login
    // Return ['abort' => false] to continue default auth
    
    // Custom LDAP authentication
    if (useLdapAuth()) {
        $result = ldapAuthenticate($vars['username'], $vars['password']);
        if ($result['success']) {
            return ['abort' => false];
        }
    }
    
    return ['abort' => false];
});
```

### Admin Dashboard Hook

```php
<?php
// Add widgets to admin dashboard
add_hook('AdminHomepage', 1, function(array $vars) {
    return [
        'sidebar' => [
            [
                'title' => 'Quick Stats',
                'template' => 'widgets/quick-stats',
            ],
        ],
        'widgets' => [
            [
                'title' => 'Pending Orders',
                'count' => getPendingOrderCount(),
                'url' => 'orders.php?status=Pending',
            ],
            [
                'title' => 'Open Tickets',
                'count' => getOpenTicketCount(),
                'url' => 'supporttickets.php?status=Open',
            ],
        ],
    ];
});
```

### Admin Navigation Hook

```php
<?php
// Modify admin navigation
add_hook('AdminAreaNav', 1, function(array $vars) {
    $userId = (int) $_SESSION['adminid'];
    $roleId = (int) $_SESSION['admin_role'];
    
    // Add custom navigation items
    $customNav = [];
    
    if ($roleId === 1) { // Full admin
        $customNav[] = [
            'name' => 'Custom Reports',
            'label' => 'Reports',
            'uri' => 'reports.php',
            'order' => 50,
        ];
    }
    
    // Add module navigation
    $moduleNav = getModuleNavigation($roleId);
    
    return [
        'add' => array_merge($customNav, $moduleNav),
    ];
});
```

### Admin Product Management Hook

```php
<?php
// Product configuration display
add_hook('ProductConfigControlOutput', 1, function(array $vars) {
    $productId = $vars['pid'];
    
    return [
        'tabContents' => [
            'custom_settings' => [
                'title' => 'Custom Settings',
                'content' => renderCustomSettings($productId),
            ],
        ],
    ];
});
```

### Admin Service Actions Hook

```php
<?php
// Add custom service actions
add_hook('AdminServiceActions', 1, function(array $vars) {
    $serviceId = $vars['serviceid'];
    $service = Capsule::table('tblhosting')->where('id', $serviceId)->first();
    
    $actions = [];
    
    // Add reboot action for VPS
    if ($service->servertype === 'kvm' || $service->servertype === 'vmware') {
        $actions[] = [
            'label' => 'Reboot Server',
            'uri' => 'cmd.php?action=reboot&id=' . $serviceId,
            'class' => 'btn-danger',
            'confirm' => 'Are you sure you want to reboot this server?',
        ];
    }
    
    // Add console access
    $actions[] = [
        'label' => 'Console Access',
        'uri' => 'cmd.php?action=console&id=' . $serviceId,
        'class' => 'btn-primary',
        'target' => '_blank',
    ];
    
    return ['actions' => $actions];
});
```

## Comprehensive Admin Handler

```php
<?php
class AdminHookHandler {
    
    public function register(): void
    {
        add_hook('AdminLogin', 1, [$this, 'handleLogin']);
        add_hook('AdminLogout', 1, [$this, 'handleLogout']);
        add_hook('PreAdminLogin', 1, [$this, 'handlePreLogin']);
        add_hook('AdminAuth', 1, [$this, 'handleAuth']);
        add_hook('AdminHomepage', 1, [$this, 'handleDashboard']);
        add_hook('AdminAreaNav', 1, [$this, 'handleNavigation']);
        add_hook('AdminServiceActions', 1, [$this, 'handleServiceActions']);
    }
    
    public function handleLogin(array $vars): array
    {
        $this->logLogin($vars['admin_id']);
        $this->updateLastLogin($vars['admin_id']);
        $this->checkSecurity($vars['admin_id']);
        return ['success' => true];
    }
    
    public function handleLogout(array $vars): array
    {
        $this->logLogout($vars['admin_id']);
        $this->clearSession($vars['admin_id']);
        return ['success' => true];
    }
    
    public function handlePreLogin(array $vars): array
    {
        // Check IP whitelist
        if (!$this->isIpAllowed()) {
            return ['abort' => true, 'error_msg' => 'IP not allowed'];
        }
        
        // Check account lock
        if ($this->isLocked($vars['username'])) {
            return ['abort' => true, 'error_msg' => 'Account locked'];
        }
        
        return ['abort' => false];
    }
    
    public function handleAuth(array $vars): array
    {
        // Custom authentication logic
        return ['abort' => false];
    }
    
    public function handleDashboard(array $vars): array
    {
        return [
            'widgets' => $this->getDashboardWidgets(),
            'sidebar' => $this->getSidebarWidgets(),
        ];
    }
    
    public function handleNavigation(array $vars): array
    {
        return ['add' => $this->getNavigationItems()];
    }
    
    public function handleServiceActions(array $vars): array
    {
        return ['actions' => $this->getServiceActions($vars['serviceid'])];
    }
    
    private function logLogin(int $adminId): void
    {
        Capsule::table('mod_admin_activity')->insert([
            'admin_id' => $adminId,
            'action' => 'login',
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function logLogout(int $adminId): void
    {
        Capsule::table('mod_admin_activity')->insert([
            'admin_id' => $adminId,
            'action' => 'logout',
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? '',
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    private function updateLastLogin(int $adminId): void
    {
        Capsule::table('tbladmins')
            ->where('id', $adminId)
            ->update(['lastlogin' => date('Y-m-d H:i:s')]);
    }
    
    private function checkSecurity(int $adminId): void
    {
        // Check for unusual activity
    }
    
    private function clearSession(int $adminId): void
    {
        // Clear session data
    }
    
    private function isIpAllowed(): bool
    {
        $allowed = ['127.0.0.1', '::1'];
        return in_array($_SERVER['REMOTE_ADDR'] ?? '', $allowed);
    }
    
    private function isLocked(string $username): bool
    {
        return false;
    }
    
    private function getDashboardWidgets(): array
    {
        return [];
    }
    
    private function getSidebarWidgets(): array
    {
        return [];
    }
    
    private function getNavigationItems(): array
    {
        return [];
    }
    
    private function getServiceActions(int $serviceId): array
    {
        return [];
    }
}

$handler = new AdminHookHandler();
$handler->register();
```

## Best Practices

1. **Secure admin hooks** - Always validate admin permissions
2. **Log admin actions** - Maintain audit trail
3. **Use CSRF tokens** - Protect against CSRF attacks
4. **Limit hook scope** - Only add what's necessary
5. **Cache dashboard data** - Avoid slow dashboard loading

## Related Documentation

- [WHMCS Client Hooks](/docs/whmcs-client-hooks.md)
- [WHMCS Cron Hooks](/docs/whmcs-cron-hooks.md)