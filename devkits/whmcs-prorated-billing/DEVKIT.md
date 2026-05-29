# WHMCS Pro-rated Billing Calculator Devkit

## Overview
A pro-rated billing calculator for WHMCS that accurately calculates partial billing periods, mid-cycle upgrades/downgrades, and prorated refunds for cancelled services.

## Features
- Pro-rated billing calculation
- Mid-cycle upgrade/downgrade pricing
- Partial period refunds
- Prorated credit calculations
- Billing cycle alignment tools
- Minimum billing period enforcement
- Prorate summary display
- Invoice line item breakdown
- Configurable rounding rules
- Prorate templates for products

## WHMCS Integration Points
- Hook: ServiceUpgrade
- Hook: ServiceDowngrade
- Hook: InvoiceCreation
- Hook: OrderPricing
- Custom pricing module

## Database Schema
```sql
CREATE TABLE mod_prorate_calculations (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    service_id INT NOT NULL,
    calculation_type ENUM('upgrade', 'downgrade', 'cancel', 'mid_cycle') NOT NULL,
    original_price DECIMAL(10,2),
    new_price DECIMAL(10,2),
    days_used INT,
    days_remaining INT,
    prorated_amount DECIMAL(10,2),
    is_credit TINYINT(1) DEFAULT 0,
    invoice_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE mod_prorate_rules (
    id INT AUTO_INCREMENT PRIMARY KEY,
    product_id INT,
    billing_cycle ENUM('monthly', 'quarterly', 'semi_annual', 'annual') NOT NULL,
    rounding_method ENUM('round', 'floor', 'ceil') DEFAULT 'round',
    decimal_places INT DEFAULT 2,
    minimum_charge DECIMAL(10,2) DEFAULT 0.00,
    grace_period_days INT DEFAULT 0
);
```

## API Endpoints
- POST /api/prorate/calculate - Calculate prorated amount
- GET /api/prorate/summary/:service_id - Get prorate summary
- POST /api/prorate/apply - Apply prorated amount to invoice
- GET /api/prorate/history/:user_id - Get calculation history

## Testing Checklist
- [ ] Mid-month billing calculation
- [ ] Upgrade pricing
- [ ] Downgrade pricing
- [ ] Cancellation refund
- [ ] Rounding accuracy
- [ ] Invoice integration

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
