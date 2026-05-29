# WHMCS Mass Actions

## Overview
Master skill for performing bulk operations on clients and services in WHMCS.

## Mass Action Helper

```php
<?php
// /includes/helpers/mass_actions_helper.php

class MassActions
{
    public function massUpdate(array $ids, string $action, array $params = []): array
    {
        $results = [
            "success" => 0,
            "failed" => 0,
            "errors" => [],
        ];
        
        foreach ($ids as $id) {
            try {
                $this->executeAction($id, $action, $params);
                $results["success"]++;
            } catch (\Exception $e) {
                $results["failed"]++;
                $results["errors"][] = "ID {$id}: " . $e->getMessage();
            }
        }
        
        return $results;
    }
    
    private function executeAction(int $id, string $action, array $params): void
    {
        switch ($action) {
            case "suspend":
                $this->suspendService($id);
                break;
                
            case "unsuspend":
                $this->unsuspendService($id);
                break;
                
            case "terminate":
                $this->terminateService($id);
                break;
                
            case "change_package":
                $this->changePackage($id, $params["package_id"]);
                break;
                
            case "add_group":
                $this->addToGroup($id, $params["group_id"]);
                break;
                
            case "send_email":
                $this->sendBulkEmail($id, $params["template"]);
                break;
        }
    }
    
    public function massSuspend(array $serviceIds): array
    {
        return $this->massUpdate($serviceIds, "suspend");
    }
    
    public function massTerminate(array $serviceIds): array
    {
        return $this->massUpdate($serviceIds, "terminate");
    }
    
    public function massChangeGroup(array $clientIds, int $groupId): array
    {
        return $this->massUpdate($clientIds, "add_group", ["group_id" => $groupId]);
    }
    
    public function massSendEmail(array $clientIds, string $template): array
    {
        return $this->massUpdate($clientIds, "send_email", ["template" => $template]);
    }
}
```

## Best Practices

1. **Confirmation**: Require confirmation for destructive actions
2. **Preview**: Show preview before execution
3. **Progress**: Display progress for large batches
4. **Rollback**: Support rollback where possible
5. **Logging**: Log all mass operations
6. **Errors**: Handle errors gracefully
7. **Undo**: Provide undo capability
8. **Rate Limiting**: Avoid overwhelming systems
