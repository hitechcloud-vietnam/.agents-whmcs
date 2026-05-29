# WHMCS Calendar Schema

Complete reference for calendar and event tables in WHMCS.

## Main Tables

### tblcalendar

Calendar entries.

```sql
CREATE TABLE `tblcalendar` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `title` VARCHAR(255) NOT NULL,
  `description` TEXT DEFAULT NULL,
  `start_date` DATETIME NOT NULL,
  `end_date` DATETIME DEFAULT NULL,
  `all_day` TINYINT(1) DEFAULT 1,
  `reminder` TINYINT(1) DEFAULT 0,
  `reminder_date` DATETIME DEFAULT NULL,
  `color` VARCHAR(20) DEFAULT NULL,
  `priority` VARCHAR(20) DEFAULT 'normal',
  `admin_id` INT(10) UNSIGNED DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_start_date` (`start_date`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblcalendar_events

Event details.

```sql
CREATE TABLE `tblcalendar_events` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `calendar_id` INT(10) UNSIGNED NOT NULL,
  `event_type` VARCHAR(50) DEFAULT NULL,
  `reference_id` INT(10) UNSIGNED DEFAULT NULL,
  `reference_type` VARCHAR(50) DEFAULT NULL,
  `location` VARCHAR(255) DEFAULT NULL,
  `attendees` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_calendar_id` (`calendar_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get calendar events

```php
function getCalendarEvents(
    string $startDate,
    string $endDate,
    ?int $adminId = null
): array {
    $query = Capsule::table('tblcalendar')
        ->whereBetween('start_date', [$startDate, $endDate]);
    
    if ($adminId) {
        $query->where('admin_id', $adminId);
    }
    
    return $query->orderBy('start_date', 'asc')
        ->get();
}
```

### Create calendar event

```php
function createCalendarEvent(array $data): int
{
    return Capsule::table('tblcalendar')->insertGetId([
        'title' => $data['title'],
        'description' => $data['description'] ?? null,
        'start_date' => $data['start_date'],
        'end_date' => $data['end_date'] ?? null,
        'all_day' => $data['all_day'] ?? 1,
        'color' => $data['color'] ?? null,
        'admin_id' => $data['admin_id'] ?? null,
        'created_at' => date('Y-m-d H:i:s')
    ]);
}
```

## Related Documentation

- [whmcs-advanced-scheduling.md](../advanced/whmcs-advanced-scheduling.md)