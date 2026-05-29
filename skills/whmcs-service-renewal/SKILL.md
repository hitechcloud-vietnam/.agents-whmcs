# WHMCS Service Renewal

## Overview
Master skill for service renewal in WHMCS. Covers renewal reminders, automatic renewals, and billing cycles.

## Renewal Hooks

```php
<?php
// /includes/hooks/service_renewal_hooks.php

add_hook("ServiceRenewal", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    $service = \WHMCS\Service\Service::find($serviceId);
    
    if ($service->status !== "Active") {
        return ["error" => "Service not active"];
    }
    
    $this->createRenewalInvoice($serviceId);
    
    return true;
});

add_hook("ServiceRenewed", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    $service = \WHMCS\Service\Service::find($serviceId);
    
    extendServiceExpiration($serviceId);
    
    sendRenewalConfirmation($service->clientId, $serviceId);
    
    return true;
});
```

## Renewal Manager

```php
<?php
// /includes/managers/RenewalManager.php

namespace WHMCS\Services;

class RenewalManager
{
    public function processRenewals(): array
    {
        $dueServices = $this->getDueForRenewal();
        
        $results = [
            "processed" => 0,
            "renewed" => 0,
            "failed" => 0,
        ];
        
        foreach ($dueServices as $service) {
            $results["processed"]++;
            
            $result = $this->processRenewal($service);
            
            if ($result["success"]) {
                $results["renewed"]++;
            } else {
                $results["failed"]++;
            }
        }
        
        return $results;
    }
    
    public function createRenewalInvoice(int $serviceId): int
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        $invoiceId = localAPI("CreateInvoice", [
            "userid" => $service->clientId,
            "duedate" => $service->nextduedate,
            "sendinvoice" => true,
        ]);
        
        $price = $service->recurringamount;
        $billingCycle = $service->billingcycle;
        
        addInvoiceLineItem($invoiceId, [
            "description" => "{$service->product->name} - {$service->domain} ({$billingCycle})",
            "amount" => $price,
            "taxed" => true,
        ]);
        
        return $invoiceId;
    }
    
    public function extendExpiration(int $serviceId): void
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        $days = $this->getBillingCycleDays($service->billingcycle);
        $newDueDate = date("Y-m-d", strtotime("+{$days} days"));
        
        \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->where("id", $serviceId)
            ->update([
                "nextduedate" => $newDueDate,
                "nextinvoicedate" => date("Y-m-d", strtotime("-7 days", strtotime($newDueDate))),
            ]);
    }
    
    private function getBillingCycleDays(string $cycle): int
    {
        $cycles = [
            "Monthly" => 30,
            "Quarterly" => 90,
            "Semi-Annually" => 180,
            "Annually" => 365,
            "Biennially" => 730,
        ];
        
        return $cycles[$cycle] ?? 30;
    }
    
    private function getDueForRenewal(): array
    {
        $threshold = date("Y-m-d", strtotime("+7 days"));
        
        return \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->join("tblproducts", "tblhosting.packageid", "=", "tblproducts.id")
            ->where("tblhosting.domainstatus", "Active")
            ->where("tblhosting.nextduedate", "<=", $threshold)
            ->whereNotIn("tblproducts.paytype", ["Free", "One Time"])
            ->get()
            ->toArray();
    }
    
    private function processRenewal($service): array
    {
        $this->createRenewalInvoice($service->id);
        
        return ["success" => true];
    }
}
```

## Best Practices

1. **Reminders**: Send renewal reminders in advance
2. **Auto-Renew**: Enable automatic renewals
3. **Grace Period**: Allow grace period after expiration
4. **Pricing**: Apply correct renewal pricing
5. **Notifications**: Communicate renewal status
6. **Suspension**: Plan for expiration handling
7. **Discounts**: Offer renewal discounts
8. **History**: Track renewal history
