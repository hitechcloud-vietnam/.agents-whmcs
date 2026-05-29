# WHMCS Tenant Billing DevKit

## Overview

Per-tenant billing system for WHMCS multi-tenant SaaS operations, enabling granular billing, usage-based charges, seat-based licensing, and consolidated invoicing.

## Features

- Per-tenant billing
- Usage-based charges
- Seat-based licensing
- Tiered pricing
- Prorated billing
- Consolidated invoicing
- Revenue attribution
- Feature-based billing

## Module Class

```php
<?php
/**
 * WHMCS Tenant Billing Module
 */

if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

require_once __DIR__ . '/lib/TenantBillingManager.php';

function whmcs_tenant_billing_activate() {
    $manager = new TenantBillingManager();
    return $manager->activate();
}

function whmcs_tenant_billing_charge($tenantId, $amount, $description, $type = 'usage') {
    $manager = new TenantBillingManager();
    return $manager->addCharge($tenantId, $amount, $description, $type);
}

function whmcs_tenant_billing_get_invoice($tenantId) {
    $manager = new TenantBillingManager();
    return $manager->generateInvoice($tenantId);
}

function whmcs_tenant_billing_get_usage($tenantId, $period = 'monthly') {
    $manager = new TenantBillingManager();
    return $manager->getUsageReport($tenantId, $period);
}
```

### lib/TenantBillingManager.php

```php
<?php
namespace WHMCS\Module\TenantBilling;

use Illuminate\Database\Capsule\Manager as Capsule;
use Carbon\Carbon;

class TenantBillingManager {
    
    public function activate() {
        try {
            $this->createTables();
            return ['success' => true, 'msg' => 'Tenant Billing module activated'];
        } catch (\Exception $e) {
            return ['success' => false, 'msg' => $e->getMessage()];
        }
    }
    
    protected function createTables() {
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_tenant_billing` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `tenant_id` INT UNSIGNED NOT NULL,
                `billing_type` VARCHAR(30) NOT NULL DEFAULT 'usage',
                `base_amount` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
                `usage_amount` DECIMAL(12,2) NOT NULL DEFAULT 0.00,
                `period_start` DATE NOT NULL,
                `period_end` DATE NOT NULL,
                `invoice_id` INT UNSIGNED NULL,
                `status` VARCHAR(20) NOT NULL DEFAULT 'pending',
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
        
        Capsule::statement("
            CREATE TABLE IF NOT EXISTS `mod_tenant_charges` (
                `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
                `tenant_billing_id` INT UNSIGNED NOT NULL,
                `charge_type` VARCHAR(30) NOT NULL,
                `description` VARCHAR(255) NOT NULL,
                `quantity` DECIMAL(12,4) NOT NULL DEFAULT 1,
                `unit_price` DECIMAL(12,4) NOT NULL,
                `total_amount` DECIMAL(12,2) NOT NULL,
                `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                PRIMARY KEY (`id`)
            ) ENGINE=DEFAULT CHARSET=utf8mb4
        ");
    }
    
    public function addCharge($tenantId, $amount, $description, $type = 'usage') {
        $billing = $this->getOrCreateCurrentBilling($tenantId);
        
        $chargeId = Capsule::table('mod_tenant_charges')->insertGetId([
            'tenant_billing_id' => $billing->id,
            'charge_type' => $type,
            'description' => $description,
            'total_amount' => $amount,
        ]);
        
        Capsule::table('mod_tenant_billing')
            ->where('id', $billing->id)
            ->increment('usage_amount', $amount);
        
        return ['success' => true, 'charge_id' => $chargeId];
    }
    
    protected function getOrCreateCurrentBilling($tenantId) {
        $now = Carbon::now();
        $periodStart = $now->copy()->startOfMonth()->toDateString();
        $periodEnd = $now->copy()->endOfMonth()->toDateString();
        
        $billing = Capsule::table('mod_tenant_billing')
            ->where('tenant_id', $tenantId)
            ->where('period_start', $periodStart)
            ->first();
        
        if ($billing) {
            return $billing;
        }
        
        // Get base pricing from service
        $service = Capsule::table('tblhosting')
            ->where('userid', $tenantId)
            ->where('domainstatus', 'Active')
            ->first();
        
        $baseAmount = $service->amount ?? 0;
        
        $id = Capsule::table('mod_tenant_billing')->insertGetId([
            'tenant_id' => $tenantId,
            'base_amount' => $baseAmount,
            'period_start' => $periodStart,
            'period_end' => $periodEnd,
        ]);
        
        return Capsule::table('mod_tenant_billing')->where('id', $id)->first();
    }
    
    public function getUsageReport($tenantId, $period = 'monthly') {
        $startDate = $period === 'monthly' 
            ? Carbon::now()->startOfMonth()->toDateString()
            : Carbon::now()->subDays(7)->toDateString();
        
        $billing = Capsule::table('mod_tenant_billing')
            ->where('tenant_id', $tenantId)
            ->where('period_start', $startDate)
            ->first();
        
        if (!$billing) {
            return ['base_amount' => 0, 'usage_amount' => 0, 'total' => 0];
        }
        
        $charges = Capsule::table('mod_tenant_charges')
            ->where('tenant_billing_id', $billing->id)
            ->get();
        
        return [
            'base_amount' => $billing->base_amount,
            'usage_amount' => $billing->usage_amount,
            'total' => $billing->base_amount + $billing->usage_amount,
            'charges' => $charges,
        ];
    }
    
    public function calculateProration($tenantId, $days, $monthlyAmount) {
        $daysInMonth = Carbon::now()->daysInMonth;
        return ($monthlyAmount / $daysInMonth) * $days;
    }
}
```

## API Endpoints

```
POST /api/v1/tenant-billing/charge       - Add usage charge
GET  /api/v1/tenant-billing/{tenantId}/usage - Get usage report
GET  /api/v1/tenant-billing/{tenantId}/invoice - Generate invoice
POST /api/v1/tenant-billing/{tenantId}/seats - Update seat count
```
