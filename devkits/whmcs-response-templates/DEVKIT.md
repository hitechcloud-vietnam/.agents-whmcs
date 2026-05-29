# WHMCS Canned Response System Devkit

## Overview
A comprehensive canned response system for WHMCS with template management, variable substitution, category organization, and usage analytics.

## Features
- Template management
- Variable substitution
- Category organization
- Rich text editing
- Attachment support
- Usage analytics
- Team sharing
- Version control
- Search functionality
- Quick insert
- Template rating
- GDPR compliance

## WHMCS Integration Points
- Module: addon/ResponseTemplates
- Hook: TicketReply
- Hook: TicketOpen
- Hook: AdminAreaPageRun

## Database Schema
```sql
CREATE TABLE mod_response_templates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    category_id INT,
    title VARCHAR(255) NOT NULL,
    subject VARCHAR(255),
    content TEXT NOT NULL,
    format ENUM('plain', 'html', 'markdown') DEFAULT 'plain',
    is_global TINYINT(1) DEFAULT 0,
    department_ids JSON,
    use_count INT DEFAULT 0,
    rating_sum INT DEFAULT 0,
    rating_count INT DEFAULT 0,
    last_used_at TIMESTAMP,
    created_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_category (category_id)
);

CREATE TABLE mod_response_template_categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INT,
    sort_order INT DEFAULT 0,
    icon VARCHAR(50)
);

CREATE TABLE mod_response_variables (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    value_source ENUM('static', 'ticket', 'client', 'system') DEFAULT 'system',
    default_value TEXT
);

CREATE TABLE mod_response_usage_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    template_id INT NOT NULL,
    ticket_id INT,
    used_by INT,
    used_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Testing Checklist
- [ ] Template creation
- [ ] Variable substitution
- [ ] Quick insert
- [ ] Usage tracking
- [ ] Category management

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
