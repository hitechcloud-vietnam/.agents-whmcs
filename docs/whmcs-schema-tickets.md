# WHMCS Tickets Database Schema

Complete reference for the support tickets table and related structures in WHMCS.

## Main Tables

### tbltickets

Primary tickets table.

```sql
CREATE TABLE `tbltickets` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `tid` VARCHAR(20) NOT NULL,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `contactid` INT(10) UNSIGNED DEFAULT NULL,
  `did` INT(10) UNSIGNED DEFAULT NULL,
  `subject` VARCHAR(255) NOT NULL,
  `status` VARCHAR(50) NOT NULL DEFAULT 'Open',
  `priority` VARCHAR(20) NOT NULL DEFAULT 'Medium',
  `admin` VARCHAR(50) DEFAULT NULL,
  `admin_unread` TINYINT(1) NOT NULL DEFAULT 0,
  `flag` VARCHAR(50) DEFAULT NULL,
  `lastreply` DATETIME DEFAULT NULL,
  `subject_encrypted` VARCHAR(255) DEFAULT NULL,
  `urgency` INT(10) DEFAULT 0,
  `ipaddress` VARCHAR(45) DEFAULT NULL,
  `locale` VARCHAR(50) DEFAULT NULL,
  `client_unread` TINYINT(1) NOT NULL DEFAULT 1,
  `merged_ticket_id` INT(10) UNSIGNED DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  `master_ticket_id` INT(10) UNSIGNED DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `tid` (`tid`),
  KEY `idx_userid` (`userid`),
  KEY `idx_status` (`status`),
  KEY `idx_did` (`did`),
  KEY `idx_priority` (`priority`),
  KEY `idx_flag` (`flag`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Ticket Statuses:**
| Status | Description |
|--------|-------------|
| Open | Awaiting response |
| Answered | Answered by staff |
| Customer Reply | Awaiting staff response |
| Closed | Ticket closed |
| Merged | Merged with another ticket |

**Ticket Priorities:**
| Priority | Description |
|----------|-------------|
| Low | Low priority |
| Medium | Medium priority |
| High | High priority |
| Urgent | Urgent priority |

### tblticketreplies

Ticket replies.

```sql
CREATE TABLE `tblticketreplies` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `tid` INT(10) UNSIGNED NOT NULL,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `contactid` INT(10) UNSIGNED DEFAULT NULL,
  `admin` VARCHAR(50) DEFAULT NULL,
  `name` VARCHAR(255) DEFAULT NULL,
  `email` VARCHAR(255) DEFAULT NULL,
  `date` DATETIME DEFAULT NULL,
  `message` TEXT DEFAULT NULL,
  `message_md` TEXT DEFAULT NULL,
  `clientread` TINYINT(1) NOT NULL DEFAULT 1,
  `adminread` TINYINT(1) NOT NULL DEFAULT 0,
  `attachment` VARCHAR(255) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_tid` (`tid`),
  KEY `idx_userid` (`userid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblticketnotes

Internal ticket notes.

```sql
CREATE TABLE `tblticketnotes` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `tid` INT(10) UNSIGNED NOT NULL,
  `admin` VARCHAR(50) DEFAULT NULL,
  `date` DATETIME DEFAULT NULL,
  `message` TEXT DEFAULT NULL,
  `visibility` VARCHAR(20) DEFAULT 'internal',
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_tid` (`tid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblticketpredefinedcats

Predefined categories.

```sql
CREATE TABLE `tblticketpredefinedcats` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `parent_id` INT(10) UNSIGNED DEFAULT NULL,
  `name` VARCHAR(255) NOT NULL,
  `description` TEXT DEFAULT NULL,
  `order` INT(10) DEFAULT 0,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblticketpredefinedreplies

Predefined replies.

```sql
CREATE TABLE `tblticketpredefinedreplies` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `catid` INT(10) UNSIGNED DEFAULT NULL,
  `name` VARCHAR(255) NOT NULL,
  `message` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_catid` (`catid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tbldepartments

Support departments.

```sql
CREATE TABLE `tbldepartments` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  `description` TEXT DEFAULT NULL,
  `email` VARCHAR(255) DEFAULT NULL,
  `host` VARCHAR(255) DEFAULT NULL,
  `port` INT(10) DEFAULT NULL,
  `username` VARCHAR(255) DEFAULT NULL,
  `password` VARCHAR(255) DEFAULT NULL,
  `delimiter` VARCHAR(10) DEFAULT NULL,
  `mailbox` VARCHAR(255) DEFAULT NULL,
  `signature` TEXT DEFAULT NULL,
  `hidden` TINYINT(1) NOT NULL DEFAULT 0,
  `clientsonly` TINYINT(1) NOT NULL DEFAULT 0,
  `order` INT(10) DEFAULT 0,
  `ticketmaskid` INT(10) UNSIGNED DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblticketfilters

Ticket filtering rules.

```sql
CREATE TABLE `tblticketfilters` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(255) NOT NULL,
  `content` TEXT DEFAULT NULL,
  `action` VARCHAR(50) DEFAULT NULL,
  `target_dept` INT(10) UNSIGNED DEFAULT NULL,
  `target_priority` VARCHAR(20) DEFAULT NULL,
  `target_admin` VARCHAR(50) DEFAULT NULL,
  `flag_to` VARCHAR(50) DEFAULT NULL,
  `auto_overdue` TINYINT(1) DEFAULT 0,
  `auto_close` TINYINT(1) DEFAULT 0,
  `order` INT(10) DEFAULT 0,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblticketattachments

Ticket attachments.

```sql
CREATE TABLE `tblticketattachments` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `ticketid` INT(10) UNSIGNED DEFAULT NULL,
  `replyid` INT(10) UNSIGNED DEFAULT NULL,
  `filename` VARCHAR(255) DEFAULT NULL,
  `filetype` VARCHAR(50) DEFAULT NULL,
  `data` MEDIUMBLOB DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_ticketid` (`ticketid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblticketlog

Ticket activity log.

```sql
CREATE TABLE `tblticketlog` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `tid` INT(10) UNSIGNED NOT NULL,
  `date` DATETIME DEFAULT NULL,
  `action` VARCHAR(50) DEFAULT NULL,
  `admin` VARCHAR(50) DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `ipaddress` VARCHAR(45) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_tid` (`tid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblticketwaiters

Ticket waiting for response.

```sql
CREATE TABLE `tblticketwaiters` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `ticket_id` INT(10) UNSIGNED NOT NULL,
  `email` VARCHAR(255) NOT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `expires_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_ticket_id` (`ticket_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Query Examples

### Get ticket with replies

```php
function getTicketWithReplies(int $ticketId): array
{
    $ticket = Capsule::table('tbltickets')
        ->select('tbltickets.*', 'tblclients.firstname', 'tblclients.lastname', 'tbldepartments.name as department')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbltickets.userid')
        ->leftJoin('tbldepartments', 'tbldepartments.id', '=', 'tbltickets.did')
        ->where('tbltickets.id', $ticketId)
        ->first();
    
    $replies = Capsule::table('tblticketreplies')
        ->where('tid', $ticketId)
        ->orderBy('date', 'asc')
        ->get();
    
    return [
        'ticket' => $ticket,
        'replies' => $replies
    ];
}
```

### Get open tickets by department

```php
function getOpenTicketsByDepartment(int $departmentId): array
{
    return Capsule::table('tbltickets')
        ->select('tbltickets.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tbltickets.userid')
        ->where('tbltickets.did', $departmentId)
        ->whereIn('tbltickets.status', ['Open', 'Answered', 'Customer Reply'])
        ->orderBy('tbltickets.urgency', 'desc')
        ->get();
}
```

## Related Documentation

- [whmcs-functions-tickets.md](../functions/whmcs-functions-tickets.md)
- [whmcs-integration-email.md](../integration/whmcs-integration-email.md)