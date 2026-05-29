# WHMCS Tenant Billing Skill

## Purpose
Provides patterns for implementing multi-tenant billing in WHMCS SaaS environments, managing billing for multiple clients, handling shared resources, and calculating tenant-specific charges.

## Implementation Patterns

### Multi-Tenant Billing
```php
<?php
class TenantBilling {
    private $db;
    
    public function generateTenantInvoice($tenantId, $period) {
        $tenant = $this->getTenant($tenantId);
        $lineItems = [];
        
        // Base subscription
        $lineItems[] = $this->calculateSubscription($tenant);
        
        // Usage-based charges
        $usageCharges = $this->calculateUsageCharges($tenantId, $period);
        $lineItems = array_merge($lineItems, $usageCharges);
        
        // Quota overages
        $overages = $this->calculateOverageCharges($tenantId);
        $lineItems = array_merge($lineItems, $overages);
        
        // Create invoice
        return $this->createInvoice($tenantId, $lineItems, $period);
    }
    
    private function calculateSubscription($tenant) {
        return [
            'description' => "{$tenant['plan_name']} Plan - {$tenant['billing_cycle']}",
            'amount' => $tenant['monthly_rate'],
            'type' => 'subscription'
        ];
    }
    
    private function calculateUsageCharges($tenantId, $period) {
        $usage = $this->getTenantUsage($tenantId, $period);
        $charges = [];
        
        foreach ($usage as $resource) {
            $rate = $this->getUsageRate($resource['type']);
            $included = $this->getIncludedAmount($tenantId, $resource['type']);
            $billable = max(0, $resource['amount'] - $included);
            
            if ($billable > 0) {
                $charges[] = [
                    'description' => ucfirst($resource['type']) . " usage",
                    'amount' => $billable * $rate,
                    'type' => 'usage',
                    'metadata' => ['amount' => $billable, 'rate' => $rate]
                ];
            }
        }
        
        return $charges;
    }
    
    private function calculateOverageCharges($tenantId) {
        $quotas = $this->getTenantQuotas($tenantId);
        $overages = [];
        
        foreach ($quotas as $quota) {
            if ($quota['usage'] > $quota['limit']) {
                $overage = $quota['usage'] - $quota['limit'];
                $rate = $quota['overage_rate'];
                
                $overages[] = [
                    'description' => "{$quota['name']} overage",
                    'amount' => $overage * $rate,
                    'type' => 'overage'
                ];
            }
        }
        
        return $overages;
    }
    
    public function getTenantBalance($tenantId) {
        $invoices = $this->db->select(
            "SELECT SUM(total) as total FROM tblinvoices WHERE userid = ?",
            [$tenantId]
        );
        
        $payments = $this->db->select(
            "SELECT SUM(amount) as total FROM tblaccounts WHERE userid = ?",
            [$tenantId]
        );
        
        return ($invoices['total'] ?? 0) - ($payments['total'] ?? 0);
    }
}
```

## Database Schema
```sql
CREATE TABLE mod_tenant_plans (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    monthly_rate DECIMAL(10,2),
    billing_cycle VARCHAR(20),
    included_resources JSON,
    overage_rates JSON
);

CREATE TABLE mod_tenant_usage (
    id INT AUTO_INCREMENT PRIMARY KEY,
    tenant_id INT NOT NULL,
    resource_type VARCHAR(50),
    amount DECIMAL(15,4),
    period_start DATE,
    period_end DATE
);
```
