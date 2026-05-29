# WHMCS Bulk Operations Module - DEVKIT

## Module Information
- **Name**: Bulk Operations
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Perform bulk service, domain, and invoice operations

## Installation
1. Copy to `/modules/addons/bulk_operations/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('AdminAreaPage', 1, function($vars) {
    if (strpos($vars['routeUri'] ?? '', '/bulk-operations') !== false) {
        return ['bulkOpsEnabled' => true];
    }
    return [];
});
```

### includes/BulkOperations.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class BulkOperations
{
    private static $table = 'mod_bulk_operations_log';
    
    /**
     * Bulk terminate services
     */
    public static function terminateServices($serviceIds, $reason = 'Bulk termination')
    {
        $results = ['success' => 0, 'failed' => 0, 'errors' => []];
        
        foreach ($serviceIds as $serviceId) {
            try {
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblhosting
                    SET domainstatus = 'Terminated'
                    WHERE id = " . (int)$serviceId
                );
                
                logActivity("Bulk termination: Service #{$serviceId} terminated");
                $results['success']++;
                
                self::logOperation('terminate_service', $serviceId, 'success');
            } catch (Exception $e) {
                $results['failed']++;
                $results['errors'][] = "Service #{$serviceId}: " . $e->getMessage();
                self::logOperation('terminate_service', $serviceId, 'failed', $e->getMessage());
            }
        }
        
        return $results;
    }
    
    /**
     * Bulk suspend services
     */
    public static function suspendServices($serviceIds, $suspendReason = 'Bulk suspend')
    {
        $results = ['success' => 0, 'failed' => 0, 'errors' => []];
        
        foreach ($serviceIds as $serviceId) {
            try {
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblhosting
                    SET domainstatus = 'Suspended'
                    WHERE id = " . (int)$serviceId
                ");
                
                logActivity("Bulk suspension: Service #{$serviceId} suspended");
                $results['success']++;
                
                self::logOperation('suspend_service', $serviceId, 'success');
            } catch (Exception $e) {
                $results['failed']++;
                $results['errors'][] = "Service #{$serviceId}: " . $e->getMessage();
            }
        }
        
        return $results;
    }
    
    /**
     * Bulk unsuspend services
     */
    public static function unsuspendServices($serviceIds)
    {
        $results = ['success' => 0, 'failed' => 0, 'errors' => []];
        
        foreach ($serviceIds as $serviceId) {
            try {
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblhosting
                    SET domainstatus = 'Active'
                    WHERE id = " . (int)$serviceId
                ");
                
                logActivity("Bulk unsuspension: Service #{$serviceId} unsuspended");
                $results['success']++;
                
                self::logOperation('unsuspend_service', $serviceId, 'success');
            } catch (Exception $e) {
                $results['failed']++;
                $results['errors'][] = "Service #{$serviceId}: " . $e->getMessage();
            }
        }
        
        return $results;
    }
    
    /**
     * Bulk change product
     */
    public static function changeProducts($serviceIds, $newProductId)
    {
        $results = ['success' => 0, 'failed' => 0, 'errors' => []];
        
        foreach ($serviceIds as $serviceId) {
            try {
                $current = full_query("SELECT * FROM " . TABLE_PREFIX . "tblhosting WHERE id = " . (int)$serviceId);
                $service = mysql_fetch_array($current);
                
                $newProduct = full_query("SELECT * FROM " . TABLE_PREFIX . "tblproducts WHERE id = " . (int)$newProductId);
                $product = mysql_fetch_array($newProduct);
                
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblhosting
                    SET packageid = " . (int)$newProductId . ",
                        server = " . (int)($product['server'] ?? 0) . "
                    WHERE id = " . (int)$serviceId
                ");
                
                logActivity("Bulk product change: Service #{$serviceId} changed to product #{$newProductId}");
                $results['success']++;
                
                self::logOperation('change_product', $serviceId, 'success');
            } catch (Exception $e) {
                $results['failed']++;
                $results['errors'][] = "Service #{$serviceId}: " . $e->getMessage();
            }
        }
        
        return $results;
    }
    
    /**
     * Bulk update renewal dates
     */
    public static function updateRenewalDates($domainIds, $newRenewalDate)
    {
        $results = ['success' => 0, 'failed' => 0];
        
        $idList = implode(',', array_map('intval', $domainIds));
        
        full_query("
            UPDATE " . TABLE_PREFIX . "tbldomains
            SET nextduedate = '" . db_escape_string($newRenewalDate) . "'
            WHERE id IN (" . $idList . ")
        ");
        
        $results['success'] = count($domainIds);
        
        return $results;
    }
    
    /**
     * Bulk send invoices
     */
    public static function sendInvoices($invoiceIds)
    {
        $results = ['success' => 0, 'failed' => 0];
        
        foreach ($invoiceIds as $invoiceId) {
            try {
                send_message(
                    $invoiceId,
                    'invoice_reminder',
                    ['invoice_id' => $invoiceId]
                );
                $results['success']++;
            } catch (Exception $e) {
                $results['failed']++;
            }
        }
        
        return $results;
    }
    
    /**
     * Bulk add credits
     */
    public static function addCredits($clientIds, $amount, $description = 'Bulk credit adjustment')
    {
        $results = ['success' => 0, 'failed' => 0];
        
        foreach ($clientIds as $clientId) {
            try {
                full_query("
                    INSERT INTO " . TABLE_PREFIX . "tblcredit (clientid, date, description, amount)
                    VALUES (" . (int)$clientId . ", NOW(), '" . db_escape_string($description) . "', " . (float)$amount . ")
                ");
                
                full_query("
                    UPDATE " . TABLE_PREFIX . "tblclients
                    SET credit = credit + " . (float)$amount . "
                    WHERE id = " . (int)$clientId
                ");
                
                $results['success']++;
            } catch (Exception $e) {
                $results['failed']++;
            }
        }
        
        return $results;
    }
    
    /**
     * Get pending operations
     */
    public static function getPendingOperations($limit = 100)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE status = 'pending'
            ORDER BY created_at DESC
            LIMIT " . (int)$limit
        );
        
        $operations = [];
        while ($row = mysql_fetch_array($result)) {
            $operations[] = $row;
        }
        
        return $operations;
    }
    
    private static function logOperation($type, $entityId, $status, $error = null)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (operation_type, entity_id, status, error_message, created_at, admin_id)
            VALUES (
                '" . db_escape_string($type) . "',
                " . (int)$entityId . ",
                '" . db_escape_string($status) . "',
                " . ($error ? "'" . db_escape_string($error) . "'" : "NULL") . ",
                NOW(),
                " . (int)($_SESSION['adminid'] ?? 0) . "
            )
        ");
    }
}

function bulk_operations_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_bulk_operations_log (
            id INT AUTO_INCREMENT PRIMARY KEY,
            operation_type VARCHAR(50) NOT NULL,
            entity_id INT NOT NULL,
            status VARCHAR(20) DEFAULT 'pending',
            error_message TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            admin_id INT DEFAULT 0
        )
    ");
    
    return ['status' => 'success', 'description' => 'Bulk Operations activated'];
}

function bulk_operations_deactivate()
{
    return ['status' => 'success', 'description' => 'Bulk Operations deactivated'];
}

function bulk_operations_config()
{
    return [
        'name' => 'Bulk Operations',
        'description' => 'Perform bulk service, domain, and invoice operations',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'batch_size' => [
                'Type' => 'text',
                'FriendlyName' => 'Batch Size',
                'Default' => '50'
            ],
            'require_confirmation' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Require Confirmation',
                'Default' => '1'
            ]
        ]
    ];
}
```