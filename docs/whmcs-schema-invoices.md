# WHMCS Invoices Database Schema

Complete reference for the invoices table and related structures in WHMCS.

## Overview

The invoices table manages all billing documents including invoices, bills, and credit notes.

## Main Tables

### tblinvoices

Primary invoices table.

```sql
CREATE TABLE `tblinvoices` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `userid` INT(10) UNSIGNED NOT NULL,
  `invoicenum` VARCHAR(100) DEFAULT NULL,
  `uuid` CHAR(36) DEFAULT NULL,
  `status` VARCHAR(50) NOT NULL DEFAULT 'Draft',
  `date` DATE NOT NULL,
  `duedate` DATE DEFAULT NULL,
  `datepaid` DATETIME DEFAULT NULL,
  `datecleared` DATETIME DEFAULT NULL,
  `date_refunded` DATETIME DEFAULT NULL,
  `subtotal` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `taxrate` DECIMAL(5,2) DEFAULT NULL,
  `taxrate2` DECIMAL(5,2) DEFAULT NULL,
  `tax` DECIMAL(10,2) DEFAULT 0.00,
  `tax2` DECIMAL(10,2) DEFAULT 0.00,
  `total` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `credit` DECIMAL(10,2) DEFAULT 0.00,
  `taxexempt` TINYINT(1) DEFAULT 0,
  `paymentmethod` VARCHAR(50) DEFAULT NULL,
  `paymentmethodname` VARCHAR(255) DEFAULT NULL,
  `notes` TEXT DEFAULT NULL,
  `currency` INT(10) UNSIGNED DEFAULT 1,
  `prjid` INT(10) UNSIGNED DEFAULT NULL,
  `lineitems` TEXT DEFAULT NULL,
  `deleted_at` DATETIME DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  `updated_at` DATETIME DEFAULT NULL,
  `refund_id` INT(10) UNSIGNED DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_userid` (`userid`),
  KEY `idx_status` (`status`),
  KEY `idx_date` (`date`),
  KEY `idx_duedate` (`duedate`),
  KEY `idx_invoicenum` (`invoicenum`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Invoice Statuses:**
| Status | Description |
|--------|-------------|
| Draft | Not yet finalized |
| Unpaid | Awaiting payment |
| Paid | Payment received |
| Partial | Partially paid |
| Payment Pending | Payment processing |
| Cancelled | Cancelled invoice |
| Refunded | Payment refunded |

### tblinvoiceitems

Invoice line items.

```sql
CREATE TABLE `tblinvoiceitems` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `invoiceid` INT(10) UNSIGNED NOT NULL,
  `userid` INT(10) UNSIGNED DEFAULT NULL,
  `type` VARCHAR(50) NOT NULL DEFAULT 'Hosting',
  `relid` INT(10) UNSIGNED DEFAULT NULL,
  `description` TEXT NOT NULL,
  `amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `taxed` TINYINT(1) NOT NULL DEFAULT 0,
  `qty` INT(10) NOT NULL DEFAULT 1,
  `duedate` DATE DEFAULT NULL,
  `period` VARCHAR(100) DEFAULT NULL,
  `created_at` DATETIME DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_invoiceid` (`invoiceid`),
  KEY `idx_type` (`type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

**Item Types:**
- `Hosting` - Hosting service
- `ResellerHosting` - Reseller account
- `VPS` - Virtual server
- `Server` - Dedicated server
- `DomainRegister` - Domain registration
- `DomainTransfer` - Domain transfer
- `Domain` - Domain renewal
- `Addon` - Service addon
- `Upgrade` - Product upgrade
- `LateFee` - Late payment fee
- `Credit` - Credit applied
- `Custom` - Custom line item

### tblinvoiceitems Lineitemtax

```sql
CREATE TABLE `tblinvoiceitems_lineitemtax` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `invoice_item_id` INT(10) UNSIGNED NOT NULL,
  `tax_id` INT(10) UNSIGNED DEFAULT NULL,
  `name` VARCHAR(100) NOT NULL,
  `rate` DECIMAL(5,2) NOT NULL DEFAULT 0.00,
  `amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  PRIMARY KEY (`id`),
  KEY `idx_invoice_item_id` (`invoice_item_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblinvoicepayments

Payment records against invoices.

```sql
CREATE TABLE `tblinvoicepayments` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `invoice_id` INT(10) UNSIGNED NOT NULL,
  `trans_id` INT(10) UNSIGNED DEFAULT NULL,
  `amount` DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  `date` DATETIME DEFAULT NULL,
  `gateway` VARCHAR(50) DEFAULT NULL,
  `ref` VARCHAR(255) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_invoice_id` (`invoice_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### tblinvoicestatus

Invoice status definitions.

```sql
CREATE TABLE `tblinvoicestatus` (
  `id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `name` VARCHAR(50) NOT NULL,
  `color` VARCHAR(20) DEFAULT NULL,
  `weight` INT(10) DEFAULT 0,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

INSERT INTO `tblinvoicestatus` (`name`, `color`, `weight`) VALUES
('Draft', '#999999', 10),
('Unpaid', '#cc0000', 20),
('Paid', '#009900', 30),
('Payment Pending', '#f0ad4e', 25),
('Cancelled', '#555555', 50),
('Refunded', '#993399', 40);
```

## Relationships

### Invoice -> Client
```php
$invoice = Capsule::table('tblinvoices')
    ->select('tblinvoices.*', 'tblclients.firstname', 'tblclients.lastname')
    ->join('tblclients', 'tblclients.id', '=', 'tblinvoices.userid')
    ->where('tblinvoices.id', $invoiceId)
    ->first();
```

### Invoice -> Items
```php
$items = Capsule::table('tblinvoiceitems')
    ->where('invoiceid', $invoiceId)
    ->get();
```

### Invoice -> Payments
```php
$payments = Capsule::table('tblinvoicepayments')
    ->where('invoice_id', $invoiceId)
    ->get();
```

## Query Examples

### Get invoice with full details

```php
function getInvoiceDetails(int $invoiceId): array
{
    $invoice = Capsule::table('tblinvoices')
        ->where('id', $invoiceId)
        ->first();
    
    $client = Capsule::table('tblclients')
        ->where('id', $invoice->userid)
        ->first();
    
    $items = Capsule::table('tblinvoiceitems')
        ->where('invoiceid', $invoiceId)
        ->get();
    
    $payments = Capsule::table('tblinvoicepayments')
        ->where('invoice_id', $invoiceId)
        ->get();
    
    $totalPaid = array_sum(array_column($payments, 'amount'));
    
    return [
        'invoice' => $invoice,
        'client' => $client,
        'items' => $items,
        'payments' => $payments,
        'balance_due' => $invoice->total - $totalPaid - $invoice->credit
    ];
}
```

### Get overdue invoices

```php
function getOverdueInvoices(): array
{
    return Capsule::table('tblinvoices')
        ->select('tblinvoices.*', 'tblclients.firstname', 'tblclients.lastname')
        ->join('tblclients', 'tblclients.id', '=', 'tblinvoices.userid')
        ->where('tblinvoices.status', 'Unpaid')
        ->where('tblinvoices.duedate', '<', date('Y-m-d'))
        ->get();
}
```

### Calculate invoice totals

```php
function calculateInvoiceTotal(int $invoiceId): array
{
    $items = Capsule::table('tblinvoiceitems')
        ->where('invoiceid', $invoiceId)
        ->get();
    
    $subtotal = 0;
    $taxable = 0;
    $tax = 0;
    
    foreach ($items as $item) {
        $subtotal += $item->amount * $item->qty;
        
        if ($item->taxed) {
            $taxable += $item->amount * $item->qty;
        }
    }
    
    $invoice = Capsule::table('tblinvoices')->where('id', $invoiceId)->first();
    
    $tax = $invoice->tax + $invoice->tax2;
    $total = $subtotal + $tax;
    
    return [
        'subtotal' => $subtotal,
        'taxable' => $taxable,
        'tax' => $tax,
        'total' => $total
    ];
}
```

## Indexes

```sql
CREATE INDEX idx_invoices_userid ON tblinvoices(userid);
CREATE INDEX idx_invoices_status ON tblinvoices(status);
CREATE INDEX idx_invoices_date ON tblinvoices(date);
CREATE INDEX idx_invoices_duedate ON tblinvoices(duedate);
CREATE INDEX idx_invoices_invoicenum ON tblinvoices(invoicenum);

CREATE INDEX idx_invoiceitems_invoiceid ON tblinvoiceitems(invoiceid);
CREATE INDEX idx_invoiceitems_type ON tblinvoiceitems(type);

CREATE INDEX idx_invoicepayments_invoice_id ON tblinvoicepayments(invoice_id);
```

## Related Documentation

- [whmcs-functions-invoices.md](../functions/whmcs-functions-invoices.md)
- [whmcs-schema-clients.md](whmcs-schema-clients.md)
- [whmcs-schema-transactions.md](whmcs-schema-transactions.md)