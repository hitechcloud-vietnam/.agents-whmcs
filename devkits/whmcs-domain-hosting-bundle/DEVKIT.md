# WHMCS Domain + Hosting Bundle Devkit

## Overview
A bundled product system offering domains combined with hosting packages, featuring automated provisioning, cross-selling recommendations, and unified management.

## Features
- Domain + hosting bundle products
- Automated provisioning on bundle purchase
- Bundle pricing tiers
- Cross-sell recommendations
- Renewal synchronization
- Unified billing for bundles
- Custom bundle builder
- Trial-to-paid conversion
- Add-on services within bundles
- Bundle analytics and ROI tracking

## WHMCS Integration Points
- Custom product: domain_hosting_bundle
- Hook: OrderCreated
- Hook: ServiceCreate
- Automated provisioning triggers

## Database Schema
```sql
CREATE TABLE mod_bundle_products (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    domain_tlds JSON,
    hosting_product_id INT,
    bundle_price DECIMAL(10,2),
    discount_percent DECIMAL(5,2),
    is_active TINYINT(1) DEFAULT 1,
    sort_order INT DEFAULT 0
);

CREATE TABLE mod_bundle_orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    bundle_id INT NOT NULL,
    domain_name VARCHAR(255),
    hosting_service_id INT,
    provisioning_status ENUM('pending', 'domain_only', 'complete', 'failed') DEFAULT 'pending',
    provisioned_at TIMESTAMP
);

CREATE TABLE mod_bundle_addons (
    id INT AUTO_INCREMENT PRIMARY KEY,
    bundle_id INT NOT NULL,
    addon_product_id INT NOT NULL,
    included_quantity INT DEFAULT 1,
    discount_percent DECIMAL(5,2),
    is_default TINYINT(1) DEFAULT 0
);
```

## API Endpoints
- POST /api/bundles - Create bundle product
- GET /api/bundles - List all bundles
- POST /api/bundles/order - Place bundle order
- GET /api/bundles/:id/analytics - Get bundle analytics

## Testing Checklist
- [ ] Bundle order creation
- [ ] Domain provisioning
- [ ] Hosting provisioning
- [ ] Bundle pricing calculation
- [ ] Renewal synchronization
- [ ] Add-on attachment
- [ ] Cross-sell recommendations

## Dependencies
- WHMCS 8.x+
- PHP 8.1+
