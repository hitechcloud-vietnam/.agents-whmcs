# WHMCS Attachments Schema

Complete reference for attachment storage tables in WHMCS.

## Main Tables

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

### tblknowledgebaseattachments

Knowledge base attachments.

```sql
CREATE TABLE `tblknowledgebaseattachments` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `article_id` INT(10) UNSIGNED NOT NULL,
  `filename` VARCHAR(255) NOT NULL,
  `filepath` VARCHAR(255) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_article_id` (`article_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblquoteattachments

Quote attachments.

```sql
CREATE TABLE `tblquoteattachments` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `quote_id` INT(10) UNSIGNED NOT NULL,
  `filename` VARCHAR(255) NOT NULL,
  `filetype` VARCHAR(50) DEFAULT NULL,
  `data` MEDIUMBLOB DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_quote_id` (`quote_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblannouncements attachments

Announcement attachments.

```sql
CREATE TABLE `tblannouncements_attachments` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `announcement_id` INT(10) UNSIGNED NOT NULL,
  `filename` VARCHAR(255) NOT NULL,
  `filepath` VARCHAR(255) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_announcement_id` (`announcement_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## File Storage Functions

### saveTicketAttachment()

```php
function saveTicketAttachment(
    int $ticketId,
    int $replyId,
    string $filename,
    string $data,
    string $contentType
): int {
    return Capsule::table('tblticketattachments')->insertGetId([
        'ticketid' => $ticketId,
        'replyid' => $replyId,
        'filename' => $filename,
        'filetype' => $contentType,
        'data' => $data,
        'created_at' => date('Y-m-d H:i:s')
    ]);
}
```

### getTicketAttachments()

```php
function getTicketAttachments(int $ticketId): array
{
    return Capsule::table('tblticketattachments')
        ->where('ticketid', $ticketId)
        ->get();
}
```

## Related Documentation

- [whmcs-schema-tickets.md](whmcs-schema-tickets.md)