# WHMCS Automated Tax Calculations Devkit

## Overview
An automated tax calculation system for WHMCS integrating with tax services, handling multiple tax jurisdictions, exempt handling, and real-time tax rate updates.

## Features
- Real-time tax calculation
- Multi-jurisdiction support
- Tax exemption handling
- VAT/GST compliance
- US sales tax integration
- EU VAT handling
- Product tax classification
- Customer tax ID validation
- Tax exemption certificates
- Tax-inclusive pricing
- Tax reporting
- Automatic rate updates

## WHMCS Integration Points
- Module: addon/TaxAutomation
- Hook: OrderPricing
- Hook: InvoiceCreation
- Hook: CheckoutComplete

## Database Schema
```sql
CREATE TABLE mod_tax_rates (
    id INT AUTO_INCREMENT PRIMARY KEY,
    jurisdiction VARCHAR(255) NOT NULL,
    tax_type ENUM('sales', 'vat', 'gst', 'custom') NOT NULL,
    rate_percent DECIMAL(5,4) NOT NULL,
    combined_rate DECIMAL(5,4),
    state_code CHAR(2),
    country_code CHAR(2),
    postal_code_pattern VARCHAR(50),
    is_compound TINYINT(1) DEFAULT 0,
    effective_date DATE,
    end_date DATE,
    is_active TINYINT(1) DEFAULT 1,
    source VARCHAR(100),
    last_synced TIMESTAMP
);

CREATE TABLE mod_tax_exemptions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    client_id INT NOT NULL,
    exemption_type ENUM('resale', 'government', 'nonprofit', 'industrial', 'other') NOT NULL,
    certificate_number VARCHAR(100),
    issuing_authority VARCHAR(255),
    state VARCHAR(100),
    valid_from DATE,
    valid_to DATE,
    document_url VARCHAR(500),
    is_verified TINYINT(1) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE idx_client (client_id)
);

CREATE TABLE mod_tax_jurisdictions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type ENUM('country', 'state', 'county', 'city') NOT NULL,
    country_code CHAR(2),
    state_code CHAR(50),
    nexus_type ENUM('physical', 'economic', 'none') DEFAULT 'none',
    collection_threshold DECIMAL(12,2),
    is_active TINYINT(1) DEFAULT 1
);
```

## API Endpoints
- GET /api/tax/calculate - Calculate tax
- POST /api/tax/validate - Validate tax ID
- GET /api/tax/rates/:jurisdiction - Get rates
- POST /api/tax/exemption - Apply exemption

## Testing Checklist
- [ ] Tax calculation accuracy
- [ ] Multi-jurisdiction handling
- [ ] Exemption validation
- [ ] Rate updates
- [ ] Tax reporting

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
