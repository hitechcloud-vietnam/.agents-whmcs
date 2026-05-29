# WHMCS Activity Log Schema

Complete reference for the activity log table and related structures in WHMCS.

## Main Tables

### tblactivitylog

System activity log.

```sql
CREATE TABLE `tblactivitylog` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `date` DATETIME NOT NULL,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `userip` VARCHAR(45) DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `username` VARCHAR(255) DEFAULT NULL,
  `sessionid` VARCHAR(255) DEFAULT NULL,
  `result` VARCHAR(20) DEFAULT NULL,
  `clientid` INT(10) UNSIGNED DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_date` (`date`),
  KEY `idx_userid` (`userid`),
  KEY `idx_clientid` (`clientid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Log Categories

| Category | Description |
|----------|-------------|
| Client | Client-related actions |
| Order | Order processing |
| Invoice | Invoice operations |
| Service | Service management |
| Domain | Domain operations |
| Ticket | Support ticket actions |
| Admin | Admin activities |
| System | System events |
| Security | Security-related events |
| Integration | Third-party integrations |

## Query Examples

### Get recent activity

```php
function getRecentActivity(int $limit = 50): array
{
    return Capsule::table('tblactivitylog')
        ->orderBy('date', 'desc')
        ->limit($limit)
        ->get();
}
```

### Get client activity

```php
function getClientActivity(int $clientId, int $limit = 100): array
{
    return Capsule::table('tblactivitylog')
        ->where('clientid', $clientId)
        ->orderBy('date', 'desc')
        ->limit($limit)
        ->get();
}
```

### Get activity by date range

```php
function getActivityByDateRange(
    string $startDate,
    string $endDate,
    int $limit = 1000
): array {
    return Capsule::table('tblactivitylog')
        ->whereBetween('date', [$startDate, $endDate])
        ->orderBy('date', 'desc')
        ->limit($limit)
        ->get();
}
```

### Log activity

```php
function logActivity(
    string $description,
    ?int $clientId = null,
    ?string $userIp = null,
    ?string $result = null
): int {
    return Capsule::table('tblactivitylog')->insertGetId([
        'date' => date('Y-m-d H:i:s'),
        'userid' => $_SESSION['adminid'] ?? null,
        'userip' => $userIp ?? $_SERVER['REMOTE_ADDR'] ?? null,
        'description' => $description,
        'username' => $_SESSION['admin_username'] ?? null,
        'result' => $result,
        'clientid' => $clientId
    ]);
}
```

### Clean old logs

```php
function cleanOldActivityLogs(int $daysOld = 90): int
{
    $cutoff = date('Y-m-d H:i:s', strtotime("-{$daysOld} days"));
    
    return Capsule::table('tblactivitylog')
        ->where('date', '<', $cutoff)
        ->delete();
}
```

## Related Documentation

- [whmcs-functions-logging.md](../functions/whmcs-functions-logging.md)