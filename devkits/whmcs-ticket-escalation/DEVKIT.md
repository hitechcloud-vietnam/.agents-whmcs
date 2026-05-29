# WHMCS Automatic Ticket Escalation Devkit

## Overview
An automatic ticket escalation system for WHMCS that monitors ticket activity, triggers escalations based on rules, notifies responsible parties, and tracks escalation history.

## Features
- Rule-based escalation triggers
- Priority escalation
- Department escalation
- Manager notification
- Time-based escalation
- Inactivity escalation
- SLA breach escalation
- Multi-level escalation paths
- Escalation history tracking
- Escalation prevention rules
- Auto-escalation disable
- Escalation analytics

## WHMCS Integration Points
- Module: addon/TicketEscalation
- Hook: TicketSave
- Hook: TicketReply
- Hook: DailyCronJob

## Database Schema
```sql
CREATE TABLE mod_escalation_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    trigger_type ENUM('time', 'sla', 'priority', 'inactivity') NOT NULL,
    trigger_value INT,
    trigger_unit ENUM('minutes', 'hours', 'days') DEFAULT 'hours',
    source_department INT,
    source_priority VARCHAR(20),
    action_type ENUM('change_department', 'change_priority', 'assign_user', 'notify', 'merge') NOT NULL,
    action_value VARCHAR(255),
    is_active TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_escalation_log (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    rule_id INT,
    escalation_type VARCHAR(50),
    from_value VARCHAR(255),
    to_value VARCHAR(255),
    escalated_by VARCHAR(50),
    escalated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_ticket (ticket_id)
);

CREATE TABLE mod_escalation_prevention (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    reason VARCHAR(255),
    prevented_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Testing Checklist
- [ ] Rule evaluation
- [ ] Escalation execution
- [ ] Notification delivery
- [ ] Prevention rules
- [ ] History tracking

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
