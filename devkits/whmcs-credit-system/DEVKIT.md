# WHMCS Account Credit System Devkit

## Overview
A comprehensive credit system for WHMCS enabling clients to add funds, apply credits to invoices, earn credits through promotions, and manage credit limits with notifications.

## Features
- Client credit accounts
- Add funds functionality
- Credit-to-invoice application
- Promotional credits
- Credit limits and caps
- Auto-credit application
- Credit transaction history
- Credit expiration handling
- Partial credit usage
- Referral credits
- Monthly credit summaries

## WHMCS Integration Points
- Module: addon/CreditSystem
- Hook: InvoiceCreation
- Hook: PaymentReceived
- Hook: OrderCreated

## Database Schema
```sql
CREATE TABLE mod_client_credits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL UNIQUE,
    current_balance DECIMAL(12,2) DEFAULT 0.00,
    credit_limit DECIMAL(12,2),
    lifetime_earned DECIMAL(12,2) DEFAULT 0.00,
    lifetime_used DECIMAL(12,2) DEFAULT 0.00,
    auto_apply_threshold DECIMAL(12,2),
    notification_threshold DECIMAL(12,2),
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE mod_credit_transactions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    amount DECIMAL(12,2) NOT NULL,
    transaction_type ENUM('deposit', 'withdrawal', 'invoice_apply', 'refund', 'promotion', 'expiration', 'adjustment') NOT NULL,
    reference_type VARCHAR(50),
    reference_id INT,
    description TEXT,
    balance_after DECIMAL(12,2),
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_client (client_id)
);

CREATE TABLE mod_credit_promotions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    credit_type ENUM('fixed', 'percentage') NOT NULL,
    credit_value DECIMAL(10,2),
    min_deposit DECIMAL(10,2),
    max_credit DECIMAL(10,2),
    requires_code TINYINT(1) DEFAULT 0,
    usage_limit INT,
    used_count INT DEFAULT 0,
    start_date DATETIME,
    end_date DATETIME,
    is_active TINYINT(1) DEFAULT 1
);
```

## Testing Checklist
- [ ] Add funds
- [ ] Credit application to invoice
- [ ] Promotional credits
- [ ] Auto-credit application
- [ ] Credit expiration
- [ ] Transaction history

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
