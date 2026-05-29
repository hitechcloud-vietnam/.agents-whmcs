# WHMCS Contract Manager Module - DEVKIT

## Module Information
- **Name**: Contract Manager
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Manage customer contracts, renewals, and SLA tracking

## Installation
1. Copy to `/modules/addons/contract_manager/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('ServiceCreated', 1, function($vars) {
    ContractManager::createContract($vars);
});

add_hook('ServiceTerminated', 1, function($vars) {
    ContractManager::terminateContract($vars);
});

add_hook('DailyCronJob', 1, function($vars) {
    ContractManager::checkExpiringContracts();
});

add_hook('InvoiceCreated', 1, function($vars) {
    ContractManager::processContractInvoicing($vars);
});
```

### includes/ContractManager.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class ContractManager
{
    private static $table = 'mod_contracts';
    
    public static function createContract($vars)
    {
        $contractData = [
            'service_id' => $vars['serviceid'] ?? 0,
            'client_id' => $vars['userid'] ?? 0,
            'contract_type' => $vars['contract_type'] ?? 'standard',
            'start_date' => date('Y-m-d'),
            'end_date' => date('Y-m-d', strtotime('+1 year')),
            'auto_renew' => $vars['auto_renew'] ?? true,
            'sla_tier' => $vars['sla_tier'] ?? 'standard',
            'status' => 'active'
        ];
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$table . "
            (service_id, client_id, contract_type, start_date, end_date, auto_renew, sla_tier, status, created_at)
            VALUES (
                " . (int)$contractData['service_id'] . ",
                " . (int)$contractData['client_id'] . ",
                '" . db_escape_string($contractData['contract_type']) . "',
                '" . db_escape_string($contractData['start_date']) . "',
                '" . db_escape_string($contractData['end_date']) . "',
                " . ($contractData['auto_renew'] ? 1 : 0) . ",
                '" . db_escape_string($contractData['sla_tier']) . "',
                'active',
                NOW()
            )
        ");
        
        return mysql_insert_id();
    }
    
    public static function terminateContract($vars)
    {
        full_query("
            UPDATE " . TABLE_PREFIX . self::$table . "
            SET status = 'terminated', terminated_at = NOW()
            WHERE service_id = " . (int)($vars['serviceid'] ?? 0)
        );
    }
    
    public static function checkExpiringContracts()
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$table . "
            WHERE status = 'active'
            AND end_date BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)
            AND notified = 0
        ");
        
        while ($contract = mysql_fetch_array($result)) {
            self::sendExpiryNotification($contract);
            
            full_query("
                UPDATE " . TABLE_PREFIX . self::$table . "
                SET notified = 1, last_notification = NOW()
                WHERE id = " . (int)$contract['id']
            );
        }
    }
    
    private static function sendExpiryNotification($contract)
    {
        $client = get_query_val("tblclients", "email", "id = " . (int)$contract['client_id']);
        
        send_email(
            $client,
            'contract_expiry_reminder',
            [
                'contract_id' => $contract['id'],
                'end_date' => $contract['end_date'],
                'auto_renew' => $contract['auto_renew']
            ]
        );
    }
    
    public static function getContractDetails($contractId)
    {
        $result = full_query("
            SELECT c.*, s.domain, s.username, cl.firstname, cl.lastname, cl.email
            FROM " . TABLE_PREFIX . self::$table . " c
            JOIN " . TABLE_PREFIX . "tblhosting s ON c.service_id = s.id
            JOIN " . TABLE_PREFIX . "tblclients cl ON c.client_id = cl.id
            WHERE c.id = " . (int)$contractId
        );
        
        return mysql_fetch_array($result);
    }
    
    public static function renewContract($contractId, $period = '1 year')
    {
        $newEndDate = date('Y-m-d', strtotime('+' . $period, strtotime(date('Y-m-d'))));
        
        full_query("
            UPDATE " . TABLE_PREFIX . self::$table . "
            SET end_date = '" . db_escape_string($newEndDate) . "',
                renewal_count = renewal_count + 1,
                last_renewed = NOW()
            WHERE id = " . (int)$contractId
        );
    }
    
    public static function getClientContracts($clientId)
    {
        $result = full_query("
            SELECT c.*, s.domain, p.name as product_name
            FROM " . TABLE_PREFIX . self::$table . " c
            JOIN " . TABLE_PREFIX . "tblhosting s ON c.service_id = s.id
            JOIN " . TABLE_PREFIX . "tblproducts p ON s.packageid = p.id
            WHERE c.client_id = " . (int)$clientId . "
            ORDER BY c.end_date ASC
        ");
        
        $contracts = [];
        while ($row = mysql_fetch_array($result)) {
            $contracts[] = $row;
        }
        
        return $contracts;
    }
    
    public static function getSLAMetrics($contractId)
    {
        $contract = self::getContractDetails($contractId);
        
        $result = full_query("
            SELECT 
                COUNT(*) as total_tickets,
                AVG(TIMESTAMPDIFF(HOUR, created_at, last_reply)) as avg_response_time,
                SUM(CASE WHEN status = 'Closed' THEN 1 ELSE 0 END) as resolved,
                SUM(CASE WHEN TIMESTAMPDIFF(HOUR, created_at, last_reply) > 24 THEN 1 ELSE 0 END) as breached
            FROM " . TABLE_PREFIX . "tbltickets
            WHERE userid = " . (int)$contract['client_id'] . "
            AND created_at BETWEEN '" . db_escape_string($contract['start_date']) . "' AND '" . db_escape_string($contract['end_date']) . "'
        ");
        
        return mysql_fetch_array($result);
    }
}

function contract_manager_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_contracts (
            id INT AUTO_INCREMENT PRIMARY KEY,
            service_id INT NOT NULL,
            client_id INT NOT NULL,
            contract_type VARCHAR(50) DEFAULT 'standard',
            start_date DATE NOT NULL,
            end_date DATE NOT NULL,
            auto_renew TINYINT(1) DEFAULT 1,
            sla_tier VARCHAR(50) DEFAULT 'standard',
            status VARCHAR(20) DEFAULT 'active',
            renewal_count INT DEFAULT 0,
            notified TINYINT(1) DEFAULT 0,
            last_notification DATETIME,
            terminated_at DATETIME,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("CREATE INDEX idx_service ON " . TABLE_PREFIX . "mod_contracts(service_id)");
    full_query("CREATE INDEX idx_client ON " . TABLE_PREFIX . "mod_contracts(client_id)");
    full_query("CREATE INDEX idx_end_date ON " . TABLE_PREFIX . "mod_contracts(end_date)");
    
    return ['status' => 'success', 'description' => 'Contract Manager activated'];
}

function contract_manager_deactivate()
{
    return ['status' => 'success', 'description' => 'Contract Manager deactivated'];
}

function contract_manager_config()
{
    return [
        'name' => 'Contract Manager',
        'description' => 'Manage customer contracts, renewals, and SLA tracking',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'notification_days' => [
                'Type' => 'text',
                'FriendlyName' => 'Notify Before Expiry (Days)',
                'Default' => '30'
            ],
            'auto_renew_default' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Auto Renew by Default',
                'Default' => '1'
            ]
        ]
    ];
}
```