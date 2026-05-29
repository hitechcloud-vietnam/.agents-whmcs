# WHMCS Orders Database Schema

Complete reference for the orders table and related structures in WHMCS.

## Overview

The orders table tracks all customer orders including products, domains, and services.

## Main Tables

### tblorders

Primary orders table.

```sql
CREATE TABLE `tblorders` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `ordernum` VARCHAR(50) NOT NULL,
  `userid` INT(10) UNSIGNED NOT NULL,
  `status` VARCHAR(50) NOT NULL DEFAULT 'Pending',
  `invoicenum` VARCHAR(100) DEFAULT NULL,
  `invoiceid` INT(10) UNSIGNED DEFAULT NULL,
  `date` DATETIME NOT NULL,
  `renewdate` DATE DEFAULT NULL,
  `paymentmethod` VARCHAR(50) NOT NULL,
  `paymentmethodname` VARCHAR(255) DEFAULT NULL,
  `amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `amountpaid` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `ipaddress` VARCHAR(45) NOT NULL,
  `affiliateid` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `affiliatetrack` TEXT DEFAULT NULL,
  `orderdetails` TEXT DEFAULT NULL,
  `nameservers` TEXT DEFAULT NULL,
  `transfersecret` VARCHAR(255) DEFAULT NULL,
  `transfersecret_encrypted` VARCHAR(255) DEFAULT NULL,
  `renewals` VARCHAR(100) DEFAULT NULL,
  `promocode` VARCHAR(50) DEFAULT NULL,
  `promotioncode` VARCHAR(50) DEFAULT NULL,
  `promotiondiscount` DECIMAL(10,2) DEFAULT NULL,
  `backup_id` INT(10) UNSIGNED DEFAULT NULL,
  `is_request` TINYINT(1) NOT NULL DEFAULT 0,
  `orderstatus` VARCHAR(50) DEFAULT NULL,
  `orderdata` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  `completeddate` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `ordernum` (`ordernum`),
  KEY `idx_userid` (`userid`),
  KEY `idx_status` (`status`),
  KEY `idx_date` (`date`),
  KEY `idx_paymentmethod` (`paymentmethod`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Order Statuses

| Status | Description |
|--------|-------------|
| Pending | Order awaiting processing |
| Active | Order completed successfully |
| Suspended | Order/services suspended |
| Cancelled | Order cancelled |
| Fraud | Order flagged as fraudulent |
| Expired | Order expired |

### tblorderitems

Order line items.

```sql
CREATE TABLE `tblorderitems` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `orderid` INT(10) UNSIGNED NOT NULL,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `orderkey` VARCHAR(50) DEFAULT NULL,
  `type` VARCHAR(50) NOT NULL DEFAULT 'Hosting',
  `relid` INT(10) UNSIGNED NOT NULL DEFAULT 0,
  `productid` INT(10) UNSIGNED DEFAULT NULL,
  `domain` VARCHAR(255) DEFAULT NULL,
  `billingcycle` VARCHAR(50) NOT NULL DEFAULT 'Monthly',
  `amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `amountsetup` DECIMAL(10,2) DEFAULT NULL,
  `description` TEXT DEFAULT NULL,
  `qty` INT(10) NOT NULL DEFAULT 1,
  `downloaded` INT(10) NOT NULL DEFAULT 0,
  `downloads` INT(10) NOT NULL DEFAULT 0,
  `itemdata` TEXT DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_orderid` (`orderid`),
  KEY `idx_type` (`type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Item Types:**
- `Hosting` - Standard hosting account
- `ResellerHosting` - Reseller hosting
- `VPS` - Virtual server
- `Server` - Dedicated server
- `Other` - Other product types
- `DomainRegister` - Domain registration
- `DomainTransfer` - Domain transfer
- `Domain` - Domain service
- `Addon` - Product addon
- `Upgrade` - Product upgrade

### tblorders Affiliatetracking

```sql
CREATE TABLE `tblorders_affiliate` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `order_id` INT(10) UNSIGNED NOT NULL,
  `affiliate_id` INT(10) UNSIGNED NOT NULL,
  `visitation_id` INT(10) UNSIGNED DEFAULT NULL,
  `ip_address` VARCHAR(45) DEFAULT NULL,
  `referred_by` INT(10) UNSIGNED DEFAULT NULL,
  `referral_date` DATETIME DEFAULT NULL,
  `conversion_date` DATETIME DEFAULT NULL,
  `commission` DECIMAL(10,2) DEFAULT NULL,
  `pending` TINYINT(1) NOT NULL DEFAULT 1,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_order_id` (`order_id`),
  KEY `idx_affiliate_id` (`affiliate_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## Relationships

### Order -> Client
```php
$order = Capsule::table('tblorders')
    ->select('tblorders.*', 'tblclients.firstname', 'tblclients.lastname')
    ->join('tblclients', 'tblclients.id', '=', 'tblorders.userid')
    ->where('tblorders.id', $orderId)
    ->first();
```

### Order -> Invoice
```php
$invoice = Capsule::table('tblinvoices')
    ->where('id', $order->invoiceid)
    ->first();
```

### Order -> Items
```php
$items = Capsule::table('tblorderitems')
    ->where('orderid', $orderId)
    ->get();
```

## Query Examples

### Get order with all details

```php
function getOrderDetails(int $orderId): array
{
    $order = Capsule::table('tblorders')
        ->where('id', $orderId)
        ->first();
    
    $client = Capsule::table('tblclients')
        ->where('id', $order->userid)
        ->first();
    
    $items = Capsule::table('tblorderitems')
        ->where('orderid', $orderId)
        ->get();
    
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $order->invoiceid)
        ->first();
    
    return [
        'order' => $order,
        'client' => $client,
        'items' => $items,
        'invoice' => $invoice
    ];
}
```

### Order statistics

```php
function getOrderStats(string $startDate, string $endDate): array
{
    return Capsule::table('tblorders')
        ->selectRaw("
            COUNT(*) as total_orders,
            SUM(amount) as total_revenue,
            COUNT(CASE WHEN status = 'Pending' THEN 1 END) as pending,
            COUNT(CASE WHEN status = 'Active' THEN 1 END) as active,
            COUNT(CASE WHEN status = 'Cancelled' THEN 1 END) as cancelled,
            COUNT(CASE WHEN status = 'Fraud' THEN 1 END) as fraud
        ")
        ->whereBetween('date', [$startDate, $endDate])
        ->first();
}
```

### Order search

```php
function searchOrders(string $term): array
{
    return Capsule::table('tblorders')
        ->select('tblorders.*', 'tblclients.firstname', 'tblclients.lastname')
        ->leftJoin('tblclients', 'tblclients.id', '=', 'tblorders.userid')
        ->where('tblorders.ordernum', 'like', '%' . $term . '%')
        ->orWhere('tblclients.firstname', 'like', '%' . $term . '%')
        ->orWhere('tblclients.lastname', 'like', '%' . $term . '%')
        ->get();
}
```

## Indexes

```sql
CREATE INDEX idx_orders_userid ON tblorders(userid);
CREATE INDEX idx_orders_status ON tblorders(status);
CREATE INDEX idx_orders_date ON tblorders(date);
CREATE INDEX idx_orders_paymentmethod ON tblorders(paymentmethod);

CREATE INDEX idx_orderitems_orderid ON tblorderitems(orderid);
CREATE INDEX idx_orderitems_type ON tblorderitems(type);
```

## Related Documentation

- [whmcs-functions-orders.md](../functions/whmcs-functions-orders.md)
- [whmcs-schema-clients.md](whmcs-schema-clients.md)
- [whmcs-schema-invoices.md](whmcs-schema-invoices.md)