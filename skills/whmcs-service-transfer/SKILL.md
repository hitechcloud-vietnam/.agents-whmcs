# WHMCS Service Transfer

## Overview
Master skill for service transfer between clients or accounts in WHMCS.

## Transfer Helper

```php
<?php
// /includes/helpers/transfer_helper.php

class ServiceTransfer
{
    public function transferToClient(int $serviceId, int $newClientId): array
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        if (!$service) {
            return ["success" => false, "error" => "Service not found"];
        }
        
        $newClient = \WHMCS\User\Client::find($newClientId);
        if (!$newClient) {
            return ["success" => false, "error" => "New client not found"];
        }
        
        \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->where("id", $serviceId)
            ->update([
                "userid" => $newClientId,
                "domainstatus" => "Pending",
            ]);
        
        sendTransferNotification($service->userid, $newClientId, $service->domain);
        
        return ["success" => true];
    }
    
    public function transferToAccount(int $serviceId, string $newUsername): array
    {
        $service = \WHMCS\Service\Service::find($serviceId);
        
        $module = new \WHMCS\Module\Server();
        $module->load($service->product->module);
        
        $result = $module->changeAccountUsername([
            "serviceid" => $serviceId,
            "username" => $newUsername,
        ]);
        
        if ($result === "success") {
            return ["success" => true];
        }
        
        return ["success" => false, "error" => $result];
    }
}
```

## Best Practices

1. **Verification**: Verify both parties consent
2. **Outstanding Balance**: Handle outstanding balances
3. **Notifications**: Notify all parties
4. **Validation**: Validate transfer request
5. **History**: Maintain service history
6. **Module Support**: Check module support
7. **Documentation**: Document transfers
8. **Audit Trail**: Keep audit trail
