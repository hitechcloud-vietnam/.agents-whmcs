# WHMCS SLA Compliance Monitoring Devkit

## Overview
An SLA compliance monitoring system for WHMCS tracking response times, resolution times, and escalation triggers with real-time dashboard and reporting.

## Features
- SLA definition and management
- Response time tracking
- Resolution time monitoring
- Escalation triggers
- Real-time dashboard
- Compliance reports
- Penalty calculation
- SLA tier management
- Holiday scheduling
- Business hours configuration
- Breach notifications
- Performance analytics

## WHMCS Integration Points
- Module: addon/SlaMonitor
- Hook: TicketOpen
- Hook: TicketReply
- Hook: TicketClose

## Database Schema
```sql
CREATE TABLE mod_sla_policies (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    priority VARCHAR(20) NOT NULL,
    first_response_hours INT NOT NULL,
    resolution_hours INT NOT NULL,
    next_escalation_hours INT,
    business_hours_only TINYINT(1) DEFAULT 1,
    is_default TINYINT(1) DEFAULT 0,
    is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE mod_sla_tracking (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL UNIQUE,
    policy_id INT NOT NULL,
    sla_deadline TIMESTAMP,
    first_response_deadline TIMESTAMP,
    resolution_deadline TIMESTAMP,
    first_response_at TIMESTAMP,
    resolved_at TIMESTAMP,
    first_response_met TINYINT(1),
    resolution_met TINYINT(1),
    time_to_first_response INT,
    time_to_resolution INT,
    escalation_count INT DEFAULT 0,
    last_escalation_at TIMESTAMP
);

CREATE TABLE mod_sla_breaches (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ticket_id INT NOT NULL,
    breach_type ENUM('first_response', 'resolution') NOT NULL,
    breach_time TIMESTAMP,
    penalty_amount DECIMAL(10,2),
    is_waived TINYINT(1) DEFAULT 0,
    waiver_reason TEXT,
    waived_by INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Testing Checklist
- [ ] SLA deadline calculation
- [ ] Response time tracking
- [ ] Breach detection
- [ ] Escalation triggers
- [ ] Compliance reports

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
