# WHMCS Service Upgrade

## Overview
Master skill for service upgrades and downgrades in WHMCS. Covers plan changes, prorating, and immediate vs scheduled upgrades.

## Upgrade Hooks

```php
<?php
// /includes/hooks/service_upgrade_hooks.php

add_hook("ServiceUpgrade", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    $newPackageId = $params["new_package_id"];
    
    $prorateAmount = calculateUpgradeProrate($serviceId, $newPackageId);
    
    if ($prorateAmount > 0) {
        chargeUpgradeDifference($serviceId, $prorateAmount);
    } else {
        applyUpgradeCredit($serviceId, abs($prorateAmount));
    }
    
    executeUpgrade($serviceId, $newPackageId);
    
    return true;
});

add_hook("ScheduledServiceUpgrade", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    
    executeScheduledUpgrade($serviceId);
    
    return true;
});
```

## Upgrade Manager

```php
<?php
// /includes/managers/UpgradeManager.php

namespace WHMCS\Services;

class UpgradeManager
{
    public function upgradeService(int $serviceId, int $newPackageId, bool $immediate = true): array
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        $newPackage = \WHMCS\Product\Product::find($newPackageId);
        
        if (!$service || !$newPackage) {
            return ["success" => false, "error" => "Invalid service or package"];
        }
        
        $prorateAmount = $this->calculateProrate($service, $newPackage);
        
        if ($immediate) {
            if ($prorateAmount > 0) {
                $this->chargeUpgrade($serviceId, $prorateAmount);
            }
            
            return $this->executeUpgrade($serviceId, $newPackageId);
        }
        
        $this->scheduleUpgrade($serviceId, $newPackageId);
        
        return ["success" => true, "scheduled" => true];
    }
    
    private function calculateProrate(\WHMCS\Service\Service $service, \WHMCS\Product\Product $newPackage): float
    {
        $daysRemaining = $this->getDaysRemaining($service);
        $daysInPeriod = 30;
        
        $oldDaily = $service->recurringamount / $daysInPeriod;
        $newDaily = $newPackage->getPrice($service->client->currency) / $daysInPeriod;
        
        return round(($newDaily - $oldDaily) * $daysRemaining, 2);
    }
    
    private function executeUpgrade(int $serviceId, int $newPackageId): array
    {
        $module = new \WHMCS\Module\Server();
        $service = \WHMCS\Service\Service::find($serviceId);
        
        if ($service->product->module) {
            $module->load($service->product->module);
            $result = $module->changePackage([
                "serviceid" => $serviceId,
                "configoption1" => $newPackageId,
            ]);
            
            if ($result !== "success") {
                return ["success" => false, "error" => $result];
            }
        }
        
        \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->where("id", $serviceId)
            ->update(["packageid" => $newPackageId]);
        
        return ["success" => true];
    }
    
    private function getDaysRemaining(\WHMCS\Service\Service $service): int
    {
        $today = new \DateTime();
        $nextDue = new \DateTime($service->nextduedate);
        return max(0, $today->diff($nextDue)->days);
    }
}
```

## Best Practices

1. **Prorating**: Calculate prorate correctly
2. **Immediate vs Scheduled**: Offer both options
3. **Feature Changes**: Handle module configuration changes
4. **Communication**: Inform customers of changes
5. **Reversibility**: Consider downgrade options
6. **Testing**: Test upgrades thoroughly
7. **Rollback**: Have rollback plan
8. **Documentation**: Document upgrade process
