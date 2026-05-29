# WHMCS Maintenance Mode Switch Devkit

## Overview
A maintenance mode management system for WHMCS that enables scheduled maintenance windows, allows whitelisted IP access, and provides status notifications.

## Features
- Maintenance mode toggle
- Scheduled maintenance windows
- IP whitelist management
- Bypass authentication
- Custom maintenance page
- Message customization
- Progress indicator
- End time estimation
- Email notifications
- API endpoints
- Scheduled tasks
- Maintenance history

## WHMCS Integration Points
- Module: addon/MaintenanceMode
- Hook: PreCalculations
- Hook: ClientAreaPage

## Database Schema
```sql
CREATE TABLE mod_maintenance_settings (
    id INT AUTO_INCREMENT PRIMARY KEY,
    is_enabled TINYINT(1) DEFAULT 0,
    start_time DATETIME,
    end_time DATETIME,
    message TEXT,
    allow_whitelisted TINYINT(1) DEFAULT 1,
    allow_logged_in TINYINT(1) DEFAULT 0,
    custom_page_html TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_maintenance_whitelist (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ip_address VARCHAR(45) NOT NULL,
    description VARCHAR(255),
    added_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_maintenance_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    action VARCHAR(50) NOT NULL,
    performed_by INT,
    ip_address VARCHAR(45),
    details TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Testing Checklist
- [ ] Toggle functionality
- [ ] Schedule maintenance
- [ ] IP whitelist access
- [ ] Custom page display
- [ ] Scheduled end

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
