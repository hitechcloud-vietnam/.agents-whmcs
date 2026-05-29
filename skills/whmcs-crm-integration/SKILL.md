# WHMCS CRM Integration

## Concept
CRM sync patterns for customer data.

## Code
```php
<?php
class CRMIntegration {
    public static function syncClientToCRM($clientId) {
        $client = getClientsDetails($clientId);
        $crmClient = $this->crm->findOrCreate($client["email"]);
        $crmClient->update(["firstname" => $client["firstname"], "lastname" => $client["lastname"]]);
        return $crmClient->save();
    }
    
    public static function syncActivity($clientId, $activity) {
        $crmClient = $this->getCRMContact($clientId);
        $crmClient->addActivity($activity);
    }
}
```
