# WHMCS Saved Replies Manager Devkit

## Overview
A saved replies manager for WHMCS with organization, search, quick insertion, and personalization features for support staff.

## Features
- Saved replies management
- Category organization
- Quick search
- Variable placeholders
- Personalization tokens
- Favorites management
- Team sharing
- Usage statistics
- Import/export
- Rich text support
- Attachment support
- Version history

## WHMCS Integration Points
- Module: addon/CannedResponses
- Hook: TicketReply
- Hook: TicketOpen

## Database Schema
```sql
CREATE TABLE mod_canned_responses (
    id INT AUTO_INCREMENT PRIMARY KEY,
    category_id INT,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    shortcode VARCHAR(50) UNIQUE,
    variables JSON,
    use_count INT DEFAULT 0,
    is_favorite TINYINT(1) DEFAULT 0,
    is_shared TINYINT(1) DEFAULT 0,
    created_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE mod_canned_response_categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INT,
    sort_order INT DEFAULT 0,
    color VARCHAR(20)
);
```

## Testing Checklist
- [ ] Reply creation
- [ ] Category management
- [ ] Variable substitution
- [ ] Quick insertion
- [ ] Usage tracking

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
