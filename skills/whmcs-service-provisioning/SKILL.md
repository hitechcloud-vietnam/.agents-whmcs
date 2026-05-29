# WHMCS Service Provisioning

## Overview
Master skill for service provisioning in WHMCS. Covers automatic setup, module integration, and provisioning workflows.

## Provisioning Hooks

```php
<?php
// /includes/hooks/service_provisioning_hooks.php

add_hook("PreServiceCreate", 1, function(array $params) {
    $product = \WHMCS\Product\Product::find($params["pid"]);
    
    if ($product->stockcontrol && $product->stock <= 0) {
        return ["error" => "Product out of stock"];
    }
    
    return ["success" => true];
});

add_hook("ServiceProvisioned", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    
    setupServiceMonitoring($serviceId);
    
    configureServiceBackups($serviceId);
    
    sendWelcomeEmail($params["userid"], $serviceId);
    
    return true;
});

add_hook("AfterServiceCreate", 1, function(array $params) {
    $serviceId = $params["serviceid"];
    $service = \WHMCS\Service\Service::find($serviceId);
    
    createDnsRecords($service);
    
    setupSslCertificate($service);
    
    return true;
});
```

## Provisioning Manager

```php
<?php
// /includes/managers/ProvisioningManager.php

namespace WHMCS\Services;

class ProvisioningManager
{
    public function provisionService(int $serviceId): array
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        if (!$service) {
            return ["success" => false, "error" => "Service not found"];
        }
        
        if ($service->status !== "Pending") {
            return ["success" => false, "error" => "Service not pending"];
        }
        
        $module = new \WHMCS\Module\Server();
        $module->load($service->product->module);
        
        $params = $this->buildModuleParams($service);
        
        $result = $module->createAccount($params);
        
        if ($result === "success") {
            $this->activateService($serviceId, $params);
            return ["success" => true];
        }
        
        return ["success" => false, "error" => $result];
    }
    
    private function buildModuleParams(\WHMCS\Service\Service $service): array
    {
        $server = $service->server;
        
        return [
            "serviceid" => $service->id,
            "userid" => $service->clientId,
            "domain" => $service->domain,
            "username" => generateServiceUsername($service),
            "password" => generateSecurePassword(),
            "server" => $server->id,
            "serverip" => $server->ipaddress,
            "serverhostname" => $server->hostname,
            "configoption1" => $service->configoption1,
            "configoption2" => $service->configoption2,
            "configoption3" => $service->configoption3,
            "configoption4" => $service->configoption4,
        ];
    }
    
    private function activateService(int $serviceId, array $params): void
    {
        \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->where("id", $serviceId)
            ->update([
                "domainstatus" => "Active",
                "username" => $params["username"],
                "password" => encrypt($params["password"]),
                "completed_date" => date("Y-m-d"),
            ]);
    }
}
```

## Best Practices

1. **Validation**: Validate before provisioning
2. **Automation**: Automate where possible
3. **Error Handling**: Handle provisioning errors
4. **Rollback**: Implement rollback on failure
5. **Notifications**: Notify on provisioning status
6. **Logging**: Log all provisioning attempts
7. **Testing**: Test provisioning thoroughly
8. **Monitoring**: Monitor provisioned services
