# WHMCS Domain Pricing Manager Devkit

## Overview
A comprehensive domain pricing management system for WHMCS that enables dynamic pricing, bulk operations, promotional campaigns, and tiered pricing strategies for domain registration, transfer, and renewal.

## Features
- Dynamic pricing engine with bulk update capabilities
- Tiered pricing based on registration period (1-10 years)
- Promotional campaign management
- Price matching and competitor analysis
- Multi-currency support with real-time conversion
- Pricing history tracking and audit log
- Bulk pricing operations for TLD groups
- Minimum/maximum price boundaries
- Premium domain pricing management

## WHMCS Integration Points
- Custom module: registrar/domain_pricing
- Hook: DomainPriceCheck
- Admin configuration page
- Cart integration for real-time pricing

## File Structure
```
whmcs-domain-pricing-manager/
├── DEVKIT.md
├── module.php
├── WHMCS/Module/Registrar/DomainPricingManager.php
├── includes/
│   ├── PricingEngine.php
│   ├── BulkOperations.php
│   └── PromotionManager.php
└── templates/
    ├── admin pricing panel
    └── client pricing display
```

## Database Schema
```sql
CREATE TABLE mod_domain_pricing (
    id INT AUTO_INCREMENT PRIMARY KEY,
    tld VARCHAR(255) NOT NULL,
    registration_price DECIMAL(10,2),
    transfer_price DECIMAL(10,2),
    renewal_price DECIMAL(10,2),
    currency_code CHAR(3) DEFAULT 'USD',
    min_years INT DEFAULT 1,
    max_years INT DEFAULT 10,
    is_premium TINYINT(1) DEFAULT 0,
    promotion_id INT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_tld (tld),
    INDEX idx_currency (currency_code)
);

CREATE TABLE mod_pricing_promotions (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    discount_type ENUM('percentage', 'fixed', 'free_years'),
    discount_value DECIMAL(10,2),
    free_years INT DEFAULT 0,
    start_date DATETIME,
    end_date DATETIME,
    tld_filter VARCHAR(500),
    is_active TINYINT(1) DEFAULT 1
);
```

## Hooks
```php
add_hook('DomainPriceCheck', 1, function($params) {
    $pricingManager = new DomainPricingManager();
    return $pricingManager->applyDynamicPricing($params);
});
```

## API Endpoints
- POST /api/pricing/update - Update single TLD price
- POST /api/pricing/bulk-update - Bulk update prices
- GET /api/pricing/tld/:tld - Get current TLD pricing
- POST /api/promotions/create - Create new promotion

## Testing Checklist
- [ ] Single TLD price update
- [ ] Bulk price update (100+ TLDs)
- [ ] Promotional discount application
- [ ] Multi-year registration pricing
- [ ] Currency conversion accuracy
- [ ] Audit log generation
- [ ] Cart integration pricing display
- [ ] Admin panel functionality

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
