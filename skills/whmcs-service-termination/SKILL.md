# WHMCS Service Termination

## Overview
Master skill for service termination in WHMCS. Covers termination workflows, data handling, and final processes.

## Termination Hooks

```php
<?php
// /includes/hooks/service_termination_hooks.php

add_hook("PreServiceTermination", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    $clientId = $params["userid"];
    
    if (hasOutstandingBalance($clientId)) {
        return ["error" => "Cannot terminate with outstanding balance"];
    }
    
    if (hasActiveLegalHold($serviceId)) {
        return ["error" => "Service has legal hold"];
    }
    
    return ["success" => true];
});

add_hook("ServiceTerminated", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    
    backupServiceData($serviceId);
    
    removeDnsRecords($params["domain"]);
    
    releaseServerResources($serviceId);
    
    sendTerminationConfirmation($params["userid"], $serviceId);
    
    return true;
});
```

## Termination Manager

```php
<?php
// /includes/managers/TerminationManager.php

namespace WHMCS\Services;

class TerminationManager
{
    public function terminateService(int $serviceId, string $reason, bool $immediate = false): array
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        if ($service->status === "Terminated") {
            return ["success" => false, "error" => "Already terminated"];
        }
        
        if (!$immediate) {
            $this->scheduleTermination($serviceId, $reason);
            return ["success" => true, "scheduled" => true];
        }
        
        return $this->executeTermination($serviceId, $reason);
    }
    
    private function executeTermination(int $serviceId, string $reason): array
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        if ($service->product->module) {
            $module = new \WHMCS\Module\Server();
            $module->load($service->product->module);
            
            $params = $this->buildModuleParams($service);
            $result = $module->terminateAccount($params);
            
            if ($result !== "success") {
                return ["success" => false, "error" => $result];
            }
        }
        
        \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->where("id", $serviceId)
            ->update([
                "domainstatus" => "Terminated",
                "termination_date" => date("Y-m-d"),
            ]);
        
        return ["success" => true];
    }
    
    private function scheduleTermination(int $serviceId, string $reason): void
    {
        $terminationDate = date("Y-m-d", strtotime("+7 days"));
        
        \Illuminate\Database\Capsule\Manager::table("mod_termination_queue")
            ->insert([
                "service_id" => $serviceId,
                "reason" => $reason,
                "scheduled_date" => $terminationDate,
                "status" => "pending",
            ]);
    }
}
```

## Best Practices

1. **Warning Period**: Provide termination notice period
2. **Data Backup**: Backup data before termination
3. **Outstanding Balance**: Handle outstanding payments
4. **Legal Holds**: Respect legal requirements
5. **Confirmation**: Require confirmation
6. **Final Invoice**: Generate final invoice
7. **Communication**: Clear communication throughout
8. **Recovery**: Consider recovery options
