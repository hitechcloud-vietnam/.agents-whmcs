# WHMCS Accounts Receivable Dunning Devkit

## Overview
A comprehensive dunning management system for WHMCS automating payment collection sequences, late fee application, escalation procedures, and debt recovery workflows.

## Features
- Dunning sequences
- Payment reminder automation
- Late fee application
- Escalation workflows
- Email/SMS notifications
- Collection agency integration
- Payment plan offers
- Debt recovery tracking
- Dunning performance analytics
- Custom message templates
- Retry scheduling
- Dispute handling

## WHMCS Integration Points
- Module: addon/DunningManagement
- Hook: InvoiceOverdue
- Hook: PaymentReceived
- Cron job integration

## Database Schema
```sql
CREATE TABLE mod_dunning_sequences (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    initial_days_overdue INT DEFAULT 1,
    total_steps INT DEFAULT 5,
    is_active TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_dunning_steps (
    id INT AUTO_INCREMENT PRIMARY KEY,
    sequence_id INT NOT NULL,
    step_number INT NOT NULL,
    days_overdue INT NOT NULL,
    action_type ENUM('reminder', 'late_fee', 'suspension', 'termination', 'escalation') NOT NULL,
    late_fee_percent DECIMAL(5,2),
    late_fee_flat DECIMAL(10,2),
    email_template VARCHAR(100),
    sms_template VARCHAR(100),
    suspend_service TINYINT(1) DEFAULT 0,
    terminate_service TINYINT(1) DEFAULT 0
);

CREATE TABLE mod_dunning_actions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    invoice_id INT NOT NULL,
    sequence_id INT,
    step_id INT,
    action_type VARCHAR(50),
    action_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('pending', 'executed', 'failed', 'skipped') DEFAULT 'pending',
    executed_at TIMESTAMP,
    error_message TEXT
);
```

## Testing Checklist
- [ ] Dunning sequence execution
- [ ] Email/SMS notifications
- [ ] Late fee application
- [ ] Service suspension
- [ ] Recovery analytics

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
