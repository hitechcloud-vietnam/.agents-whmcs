# WHMCS IP Whitelist Module

IP whitelist protection for admin access control with support for IP ranges, subnets, time-based access, and comprehensive logging.

## Features

- Single IP whitelist
- IP range support
- Subnet/CIDR support
- Time-based access (expiration)
- Access logging
- Admin notifications on blocked access
- 2FA requirement for whitelisted IPs
- Access statistics
- Expired entry cleanup

## Installation

1. Copy the module to your WHMCS installation:
   ```
   /path/to/whmcs/modules/servers/ipwhitelist/
   ```

2. Activate through WHMCS Admin:
   - Go to **Setup > Addon Modules**
   - Find "IP Whitelist"
   - Click **Activate**
   - Configure settings

3. Add to admin area for protection (requires hook integration).

## Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| EnableWhitelist | yes | Enable IP whitelist |
| EnableLogging | yes | Log access attempts |
| DefaultAction | block | Default for non-listed IPs |
| NotifyOnBlock | no | Email admin on blocked access |
| AdminEmail | - | Notification email |
| Enable2FA | no | Require 2FA for whitelisted IPs |

## Usage

### Adding IPs to Whitelist

```php
// Add single IP
ipwhitelist_AddIP('192.168.1.100', 'Office IP', array(
    'created_by' => $adminId,
));

// Add IP range
ipwhitelist_AddIP('192.168.1.0', 'Office Network', array(
    'type' => 'range',
    'ip_end' => '192.168.1.255',
));

// Add subnet (CIDR notation)
ipwhitelist_AddIP('10.0.0.0', 'VPN Network', array(
    'type' => 'subnet',
    'subnet_mask' => 24, // 10.0.0.0/24
));

// Add with expiration
ipwhitelist_AddIP('203.0.113.50', 'Contractor IP', array(
    'expires_at' => '2026-12-31 23:59:59',
    'description' => 'Contractor temporary access',
));
```

### Managing Whitelist

```php
// Get all entries
$entries = ipwhitelist_GetEntries(array('active_only' => true));

// Get specific entry
$entry = ipwhitelist_GetEntry('192.168.1.100');

// Update entry
ipwhitelist_UpdateIP('192.168.1.100', array(
    'description' => 'Updated description',
    'expires_at' => '2027-01-01 00:00:00',
    'require_2fa' => 1,
));

// Remove from whitelist
ipwhitelist_RemoveIP('192.168.1.100');
```

### Validating Access

```php
// Check if IP is whitelisted
$result = ipwhitelist_IsWhitelisted('192.168.1.100');

if ($result['whitelisted']) {
    $entry = $result['entry'];
    echo "IP is whitelisted: " . $entry->description;
} else {
    echo "IP is not whitelisted";
}

// Validate full access request
$result = ipwhitelist_ValidateAccess('192.168.1.100', $userId);

if ($result['allowed']) {
    if (!empty($result['requires_2fa'])) {
        // IP is whitelisted but requires 2FA
    }
    // Grant access
} else {
    // Block access
}
```

### Access Logging

```php
// Get access logs
$logs = ipwhitelist_GetLogs(array(
    'since' => '2026-05-01 00:00:00',
    'action' => 'blocked',
    'limit' => 100,
));

// Get statistics
$stats = ipwhitelist_GetStats(30);
// Returns: total_attempts, allowed_attempts, blocked_attempts, unique_ips, top_blocked_ips
```

### Maintenance

```php
// Clean expired entries
$deleted = ipwhitelist_CleanExpired();
echo "Removed {$deleted} expired entries";
```

## IP Types

### Single IP
```
192.168.1.100
```

### IP Range
```
192.168.1.0 - 192.168.1.255
```

### Subnet (CIDR)
```
10.0.0.0/24  (10.0.0.0 - 10.0.0.255)
10.0.0.0/16  (10.0.0.0 - 10.0.255.255)
172.16.0.0/12 (172.16.0.0 - 172.31.255.255)
```

## Hook Integration

To protect admin area, create a hook:

```php
<?php
// includes/hooks/ipwhitelist_protection.php
add_hook('AdminAreaPage', 1, function($vars) {
    $ip = $_SERVER['REMOTE_ADDR'];
    $result = ipwhitelist_ValidateAccess($ip);
    
    if (!$result['allowed']) {
        http_response_code(403);
        die('Access denied. Your IP is not whitelisted.');
    }
    
    if (!empty($result['requires_2fa']) && !twofactorauth_IsEnabled($_SESSION['adminid'], 'admin')) {
        // Redirect to 2FA setup
    }
});
```

## Database Tables

- `mod_ipwhitelist_entries` - Whitelist entries
- `mod_ipwhitelist_logs` - Access logs

## API Functions

| Function | Description |
|----------|-------------|
| `ipwhitelist_AddIP()` | Add IP to whitelist |
| `ipwhitelist_RemoveIP()` | Remove IP from whitelist |
| `ipwhitelist_UpdateIP()` | Update whitelist entry |
| `ipwhitelist_GetEntry()` | Get specific entry |
| `ipwhitelist_GetEntries()` | Get all entries |
| `ipwhitelist_IsWhitelisted()` | Check if IP is whitelisted |
| `ipwhitelist_ValidateAccess()` | Full access validation |
| `ipwhitelist_LogAccess()` | Log access attempt |
| `ipwhitelist_GetLogs()` | Get access logs |
| `ipwhitelist_GetStats()` | Get statistics |
| `ipwhitelist_CleanExpired()` | Clean expired entries |

## Version History

- **1.0.0** - Initial release
  - IP whitelist management
  - IP range and subnet support
  - Access logging
  - Admin notifications
  - Statistics
  - Expiration management
