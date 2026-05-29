# WHMCS Alert Notification System Devkit

## Overview
A comprehensive alert notification system for WHMCS that manages alert rules, channels, escalation, and delivery with support for multiple notification methods.

## Features
- Alert rule management
- Multiple notification channels
- Alert escalation
- Channel templates
- Alert grouping
- Alert deduplication
- Alert acknowledgment
- Alert history
- Quiet hours
- Alert priorities
- Channel rotation
- Delivery status tracking

## WHMCS Integration Points
- Module: addon/AlertManager
- Hook: DailyCronJob
- Notification integration

## Database Schema
```sql
CREATE TABLE mod_alert_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    condition_type VARCHAR(100) NOT NULL,
    condition_config JSON NOT NULL,
    severity ENUM('info', 'warning', 'critical') NOT NULL,
    channels JSON NOT NULL,
    escalation_enabled TINYINT(1) DEFAULT 0,
    escalation_steps JSON,
    is_active TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_alert_channels (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    channel_type ENUM('email', 'sms', 'webhook', 'slack', 'discord') NOT NULL,
    config JSON NOT NULL,
    is_active TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_alert_notifications (
    id INT AUTO_INCREMENT PRIMARY KEY,
    rule_id INT,
    channel_id INT NOT NULL,
    recipient VARCHAR(255) NOT NULL,
    subject VARCHAR(500),
    message TEXT,
    status ENUM('pending', 'sent', 'failed', 'acknowledged') DEFAULT 'pending',
    sent_at TIMESTAMP,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_alert_escalation (
    id INT AUTO_INCREMENT PRIMARY KEY,
    original_alert_id INT NOT NULL,
    escalation_level INT DEFAULT 1,
    escalated_to VARCHAR(255),
    escalated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    acknowledged TINYINT(1) DEFAULT 0
);

CREATE TABLE mod_alert_quiet_hours (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    start_time TIME,
    end_time TIME,
    days_of_week JSON,
    timezone VARCHAR(50) DEFAULT 'UTC',
    is_active TINYINT(1) DEFAULT 1
);
```

## API Endpoints
- GET /api/alerts - List active alerts
- POST /api/alerts - Create alert
- PUT /api/alerts/:id/acknowledge - Acknowledge
- GET /api/alerts/channels - List channels
- POST /api/alerts/channels - Create channel

## Testing Checklist
- [ ] Alert rule evaluation
- [ ] Notification delivery
- [ ] Escalation handling
- [ ] Quiet hours
- [ ] Delivery tracking

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
