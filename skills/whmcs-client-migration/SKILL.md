# WHMCS Client Migration

## Overview
Master skill for migrating clients between WHMCS installations or from external systems.

## Migration Helper

```php
<?php
// /includes/helpers/migration_helper.php

class ClientMigration
{
    public function migrateClient(array $clientData, bool $migrateServices = true): int
    {
        $existingClient = \Illuminate\Database\Capsule\Manager::table("tblclients")
            ->where("email", $clientData["email"])
            ->first();
        
        if ($existingClient) {
            return $this->updateClient($existingClient->id, $clientData);
        }
        
        $clientId = $this->createClient($clientData);
        
        if ($migrateServices && !empty($clientData["services"])) {
            $this->migrateServices($clientId, $clientData["services"]);
        }
        
        if (!empty($clientData["invoices"])) {
            $this->migrateInvoices($clientId, $clientData["invoices"]);
        }
        
        return $clientId;
    }
    
    private function createClient(array $data): int
    {
        return \Illuminate\Database\Capsule\Manager::table("tblclients")
            ->insertGetId([
                "uuid" => \Illuminate\Support\Str::uuid()->toString(),
                "firstname" => $data["firstname"],
                "lastname" => $data["lastname"],
                "companyname" => $data["companyname"] ?? "",
                "email" => $data["email"],
                "address1" => $data["address1"] ?? "",
                "city" => $data["city"] ?? "",
                "state" => $data["state"] ?? "",
                "postcode" => $data["postcode"] ?? "",
                "country" => $data["country"] ?? "",
                "phonenumber" => $data["phonenumber"] ?? "",
                "created_at" => date("Y-m-d H:i:s"),
            ]);
    }
    
    private function migrateServices(int $clientId, array $services): void
    {
        foreach ($services as $service) {
            $service["userid"] = $clientId;
            $this->createService($service);
        }
    }
    
    private function createService(array $data): int
    {
        return \Illuminate\Database\Capsule\Manager::table("tblhosting")
            ->insertGetId([
                "uuid" => \Illuminate\Support\Str::uuid()->toString(),
                "userid" => $data["userid"],
                "packageid" => $data["packageid"],
                "regdate" => $data["regdate"] ?? date("Y-m-d"),
                "domain" => $data["domain"],
                "paymentmethod" => $data["paymentmethod"] ?? "",
                "billingcycle" => $data["billingcycle"] ?? "Monthly",
                "nextduedate" => $data["nextduedate"] ?? date("Y-m-d"),
                "status" => $data["status"] ?? "Pending",
            ]);
    }
}
```

## Best Practices

1. **Validation**: Validate all data before import
2. **Duplicates**: Check for existing clients
3. **Order**: Migrate in correct sequence
4. **Verification**: Verify migration success
5. **Rollback**: Have rollback plans
6. **Testing**: Test with sample data first
7. **Logging**: Log all migration activities
8. **Notifications**: Notify clients of migration
