# WHMCS Scheduled Report Generation Devkit

## Overview
A scheduled report generation system for WHMCS that creates reports on schedule, supports multiple formats, and delivers reports via email or storage.

## Features
- Report templates
- Scheduled generation
- Multiple output formats (PDF, CSV, Excel)
- Email delivery
- Storage management
- Report history
- Report customization
- Data retention policies
- Report sharing
- Report access control
- Incremental reports
- Report analytics

## WHMCS Integration Points
- Module: addon/ReportScheduler
- Hook: DailyCronJob
- Email integration

## Database Schema
```sql
CREATE TABLE mod_report_schedules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    report_type VARCHAR(100) NOT NULL,
    schedule_type ENUM('daily', 'weekly', 'monthly', 'quarterly') DEFAULT 'daily',
    schedule_time TIME,
    schedule_day INT,
    format ENUM('pdf', 'csv', 'excel', 'html') DEFAULT 'pdf',
    recipients JSON,
    storage_location VARCHAR(255),
    retention_days INT DEFAULT 30,
    is_active TINYINT(1) DEFAULT 1,
    last_run_at TIMESTAMP,
    next_run_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_report_templates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    report_type VARCHAR(100) NOT NULL,
    template_config JSON NOT NULL,
    columns JSON,
    filters JSON,
    grouping VARCHAR(100),
    sorting VARCHAR(100),
    created_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_report_history (
    id INT AUTO_INCREMENT PRIMARY KEY,
    schedule_id INT NOT NULL,
    report_name VARCHAR(255),
    file_path VARCHAR(500),
    file_size INT,
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('success', 'failed') DEFAULT 'success',
    error_message TEXT
);

CREATE TABLE mod_report_data (
    id INT AUTO_INCREMENT PRIMARY KEY,
    report_id INT NOT NULL,
    data JSON NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_report (report_id)
);
```

## API Endpoints
- POST /api/reports/schedules - Create schedule
- GET /api/reports/schedules/:id - Get schedule
- POST /api/reports/generate - Generate report
- GET /api/reports/history - Get history
- GET /api/reports/:id/download - Download report

## Testing Checklist
- [ ] Schedule execution
- [ ] Report generation
- [ ] Email delivery
- [ ] Format conversion
- [ ] Storage management

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
