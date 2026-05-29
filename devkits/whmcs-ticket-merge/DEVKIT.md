# WHMCS Ticket Merging Tool Devkit

## Overview
A ticket merging tool for WHMCS that combines duplicate tickets, maintains relationship history, and handles merged ticket notifications.

## Features
- Duplicate ticket detection
- Ticket merging interface
- Merge preview
- Content consolidation
- Reply history preservation
- Merge notifications
- Merge undo capability
- Merge criteria settings
- Bulk merge operations
- Merge audit log

## WHMCS Integration Points
- Module: addon/TicketMerge
- Hook: TicketSave
- Hook: AdminAreaPageRun

## Database Schema
```sql
CREATE TABLE mod_ticket_merges (
    id INT AUTO_INCREMENT PRIMARY KEY,
    primary_ticket_id INT NOT NULL,
    merged_ticket_ids JSON NOT NULL,
    merged_by INT NOT NULL,
    merge_reason TEXT,
    merged_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_primary (primary_ticket_id)
);

CREATE TABLE mod_ticket_merge_criteria (
    id INT AUTO_INCREMENT PRIMARY KEY,
    criteria_type ENUM('subject_similarity', 'customer_email', 'subject_keywords') NOT NULL,
    criteria_value VARCHAR(255),
    similarity_threshold DECIMAL(3,2),
    auto_merge TINYINT(1) DEFAULT 0
);
```

## Testing Checklist
- [ ] Merge interface
- [ ] History preservation
- [ ] Notification sending
- [ ] Undo capability
- [ ] Audit logging

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
