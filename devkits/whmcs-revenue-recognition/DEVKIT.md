# WHMCS Revenue Recognition Tracking Devkit

## Overview
A revenue recognition tracking system for WHMCS implementing ASC 606/IFRS 15 standards with deferred revenue management, milestone-based recognition, and financial reporting.

## Features
- ASC 606/IFRS 15 compliance
- Deferred revenue tracking
- Milestone-based recognition
- Contract asset management
- Performance obligation tracking
- Revenue allocation
- Recognition scheduling
- Financial reports
- Audit trail
- Multi-currency support
- Tax handling in recognition

## WHMCS Integration Points
- Module: addon/RevenueRecognition
- Hook: OrderCreated
- Hook: InvoicePaid
- Hook: ServiceCreate

## Database Schema
```sql
CREATE TABLE mod_revenue_contracts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    order_id INT NOT NULL,
    contract_value DECIMAL(12,2) NOT NULL,
    currency_code CHAR(3) DEFAULT 'USD',
    recognition_method ENUM('straight_line', 'milestone', 'percentage_complete') DEFAULT 'straight_line',
    start_date DATE,
    end_date DATE,
    total_periods INT,
    periods_completed INT DEFAULT 0,
    recognized_revenue DECIMAL(12,2) DEFAULT 0.00,
    deferred_revenue DECIMAL(12,2),
    status ENUM('active', 'completed', 'cancelled') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_revenue_schedules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    contract_id INT NOT NULL,
    period_number INT NOT NULL,
    period_start DATE,
    period_end DATE,
    scheduled_revenue DECIMAL(12,2) NOT NULL,
    actual_revenue DECIMAL(12,2),
    recognized_at TIMESTAMP,
    invoice_id INT,
    INDEX idx_contract (contract_id)
);

CREATE TABLE mod_revenue_allocation (
    id INT AUTO_INCREMENT PRIMARY KEY,
    contract_id INT NOT NULL,
    performance_obligation VARCHAR(255) NOT NULL,
    allocated_value DECIMAL(12,2) NOT NULL,
    allocation_percent DECIMAL(5,2),
    recognized_value DECIMAL(12,2) DEFAULT 0.00
);
```

## Testing Checklist
- [ ] Contract creation
- [ ] Revenue scheduling
- [ ] Recognition execution
- [ ] Deferred revenue tracking
- [ ] Financial reports

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
