# WHMCS Automated Invoicing System Devkit

## Overview
A comprehensive automated invoicing system for WHMCS with customizable billing schedules, payment reminders, auto-charging, and invoice approval workflows.

## Features
- Automated invoice generation
- Custom billing schedules
- Payment reminder automation
- Auto-charge on due date
- Invoice approval workflows
- Recurring invoice templates
- Grace period management
- Late fee automation
- Partial payment support
- Invoice grouping
- Custom invoice notes
- Multi-currency invoicing

## WHMCS Integration Points
- Module: addon/AutoInvoicing
- Hook: InvoiceCreation
- Hook: InvoicePaid
- Hook: PaymentReminder
- Cron job integration

## Database Schema
```sql
CREATE TABLE mod_auto_invoice_schedules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    schedule_type ENUM('monthly', 'quarterly', 'semi_annual', 'annual', 'custom') NOT NULL,
    custom_days JSON,
    generation_days_before INT DEFAULT 7,
    auto_charge TINYINT(1) DEFAULT 0,
    payment_method_id INT,
    approval_required TINYINT(1) DEFAULT 0,
    late_fee_enabled TINYINT(1) DEFAULT 0,
    late_fee_amount DECIMAL(10,2),
    grace_period_days INT DEFAULT 0,
    is_active TINYINT(1) DEFAULT 1,
    next_generation_date DATE,
    INDEX idx_service (service_id)
);

CREATE TABLE mod_invoice_reminders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    invoice_id INT NOT NULL,
    reminder_type ENUM('payment', 'overdue', 'final') NOT NULL,
    reminder_number INT DEFAULT 1,
    scheduled_date DATE,
    sent_at TIMESTAMP,
    sent_via ENUM('email', 'sms', 'both') DEFAULT 'email',
    content_override TEXT
);

CREATE TABLE mod_invoice_approvals (
    id INT AUTO_INCREMENT PRIMARY KEY,
    invoice_id INT NOT NULL,
    approval_status ENUM('pending', 'approved', 'rejected') DEFAULT 'pending',
    approved_by INT,
    approved_at TIMESTAMP,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## API Endpoints
- POST /api/auto-invoice/schedules - Create schedule
- GET /api/auto-invoice/schedules/:service_id - Get schedules
- PUT /api/auto-invoice/schedules/:id - Update schedule
- POST /api/auto-invoice/generate - Generate invoices
- GET /api/invoice-reminders/pending - Get pending reminders

## Testing Checklist
- [ ] Invoice generation timing
- [ ] Payment reminders
- [ ] Auto-charge execution
- [ ] Late fee application
- [ ] Approval workflow
- [ ] Multi-currency handling

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
