# WHMCS Installment Payment Plans Devkit

## Overview
A comprehensive installment payment plan system for WHMCS that enables flexible payment scheduling, down payment collection, automated payment collection, and plan management.

## Features
- Flexible payment schedules
- Down payment collection
- Automated payment collection
- Plan modification options
- Early payoff discounts
- Late payment handling
- Payment method integration
- Plan templates
- Credit check integration
- Payment history tracking
- Auto-financing approval

## WHMCS Integration Points
- Module: addon/PaymentPlans
- Hook: OrderCreated
- Hook: InvoiceCreation
- Hook: PaymentReceived

## Database Schema
```sql
CREATE TABLE mod_payment_plans (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    plan_name VARCHAR(255),
    total_amount DECIMAL(12,2) NOT NULL,
    down_payment DECIMAL(12,2),
    installments INT NOT NULL,
    interest_rate DECIMAL(5,2) DEFAULT 0.00,
    payment_frequency ENUM('weekly', 'biweekly', 'monthly') DEFAULT 'monthly',
    start_date DATE,
    status ENUM('active', 'completed', 'defaulted', 'cancelled') DEFAULT 'active',
    auto_charge TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_payment_plan_schedules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    plan_id INT NOT NULL,
    installment_number INT NOT NULL,
    amount_due DECIMAL(12,2) NOT NULL,
    principal_amount DECIMAL(12,2),
    interest_amount DECIMAL(12,2),
    due_date DATE NOT NULL,
    status ENUM('pending', 'paid', 'late', 'failed') DEFAULT 'pending',
    invoice_id INT,
    paid_at TIMESTAMP,
    INDEX idx_plan (plan_id)
);
```

## Testing Checklist
- [ ] Plan creation
- [ ] Payment scheduling
- [ ] Auto-charge execution
- [ ] Late payment handling
- [ ] Plan modification
- [ ] Early payoff

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
