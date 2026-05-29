# WHMCS Usage-Based Billing Meter Devkit

## Overview
A comprehensive usage metering and billing system for WHMCS that tracks metered resources (API calls, storage, bandwidth) and generates accurate usage-based invoices.

## Features
- Real-time resource metering
- Usage tracking and aggregation
- Tiered usage pricing
- Overage billing
- Usage dashboards and reports
- Threshold alerts
- Usage forecasting
- API usage tracking
- Storage usage metering
- Bandwidth monitoring
- Custom meter definitions
- Usage-based upselling

## WHMCS Integration Points
- Module: addon/UsageMetering
- Hook: ServiceCreate
- Hook: InvoiceCreation
- Custom meter integration

## Database Schema
```sql
CREATE TABLE mod_usage_meters (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    meter_type ENUM('api_calls', 'storage', 'bandwidth', 'compute', 'custom') NOT NULL,
    current_usage BIGINT DEFAULT 0,
    unit VARCHAR(50) NOT NULL,
    reset_period ENUM('daily', 'weekly', 'monthly', 'never') DEFAULT 'monthly',
    reset_date DATE,
    last_reset_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_service (service_id)
);

CREATE TABLE mod_usage_records (
    id INT AUTO_INCREMENT PRIMARY KEY,
    meter_id INT NOT NULL,
    usage_amount BIGINT NOT NULL,
    recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_meter_time (meter_id, recorded_at)
);

CREATE TABLE mod_usage_pricing_tiers (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    meter_type VARCHAR(50) NOT NULL,
    from_unit INT NOT NULL,
    to_unit INT,
    unit_price DECIMAL(10,4),
    tier_order INT DEFAULT 0
);

CREATE TABLE mod_usage_alerts (
    id INT AUTO_INCREMENT PRIMARY KEY,
    service_id INT NOT NULL,
    meter_id INT,
    threshold_percent INT DEFAULT 80,
    alert_type ENUM('email', 'sms', 'both', 'none') DEFAULT 'email',
    is_active TINYINT(1) DEFAULT 1
);
```

## API Endpoints
- POST /api/usage/record - Record usage
- GET /api/usage/current/:service_id - Get current usage
- GET /api/usage/history/:service_id - Get usage history
- POST /api/usage/meters - Create meter
- GET /api/usage/forecast/:service_id - Get usage forecast

## Testing Checklist
- [ ] Usage recording accuracy
- [ ] Tiered pricing calculation
- [ ] Overage billing
- [ ] Reset period handling
- [ ] Alert notifications
- [ ] Invoice generation

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
