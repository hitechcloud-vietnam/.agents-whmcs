# WHMCS Service Suspension

## Overview
Master skill for service suspension in WHMCS. Covers suspension reasons, automated suspension, and reactivation.

## Suspension Hooks

```php
<?php
// /includes/hooks/service_suspension_hooks.php

add_hook("ServiceSuspended", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    
    sendSuspensionNotice($params["userid"], $serviceId, $params["suspendreason"]);
    
    disableServiceAccess($serviceId);
    
    logSuspension($serviceId, $params["suspendreason"]);
    
    return true;
});

add_hook("ServiceUnsuspended", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    
    enableServiceAccess($serviceId);
    
    sendReactivationNotice($params["userid"], $serviceId);
    
    return true;
});
```

## Suspension Manager

```php
<?php
// /includes/managers/SuspensionManager.php

namespace WHMCS\Services;

class SuspensionManager
{
    public function suspendService(int $serviceId, string $reason): array
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        if ($service->status !== "Active") {
            return ["success" => false, "error" => "Service not active"];
        }
        
        $module = new \WHMCS\Module\Server();
        $module->load($service->product->module);
        
        $params = $this->buildModuleParams($service);
        $result = $module->suspendAccount($params);
        
        if ($result === "success") {
            $this->updateServiceStatus($serviceId, "Suspended", $reason);
            return ["success" => true];
        }
        
        return ["success" => false, "error" => $result];
    }
    
    public function unsuspendService(int $serviceId): array
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        if ($service->status !== "Suspended") {
            return ["success" => false, "error" => "Service not suspended"];
        }
        
        $module = new \WHMCS\Module\Server();
        $module->load($service->product->module);
        
        $params = $this->buildModuleParams($service);
        $result = $module->unsuspendAccount($params);
        
        if ($result === "success") {
            $this->updateServiceStatus($serviceId, "Active", "");
            return ["success" => true];
        }
        
        return ["success" => false, "error" => $result];
    }
    
    private function updateServiceStatus(int $serviceId, string $status, string $reason): void
    {
        \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->where("id", $serviceId)
            ->update([
                "domainstatus" => $status,
                "suspendreason" => $reason,
            ]);
    }
}
```

## Best Practices

1. **Grace Period**: Allow grace period before suspension
2. **Notifications**: Send warnings before suspension
3. **Reasons**: Track suspension reasons
4. **Automation**: Automate based on overdue invoices
5. **Reactivation**: Make reactivation easy
6. **Data**: Protect customer data during suspension
7. **Communication**: Communicate clearly
8. **Recovery**: Have clear recovery process
