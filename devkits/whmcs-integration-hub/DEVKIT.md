# WHMCS Integration Hub Module - DEVKIT

## Module Information
- **Name**: Integration Hub
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Central hub for third-party integrations and API management

## Installation
1. Copy to `/modules/addons/integration_hub/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('ClientCreated', 1, function($vars) {
    IntegrationHub::syncClient($vars);
});

add_hook('OrderPlaced', 1, function($vars) {
    IntegrationHub::notifyExternal('order_created', $vars);
});

add_hook('InvoicePaid', 1, function($vars) {
    IntegrationHub::syncPayment($vars);
});

add_hook('ServiceCreated', 1, function($vars) {
    IntegrationHub::provisionExternal($vars);
});
```

### includes/IntegrationHub.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class IntegrationHub
{
    private static $integrationsTable = 'mod_integration_connections';
    private static $logTable = 'mod_integration_logs';
    
    public static function registerIntegration($name, $config)
    {
        $data = [
            'name' => $name,
            'type' => $config['type'] ?? 'api',
            'endpoint' => $config['endpoint'] ?? '',
            'api_key' => $config['api_key'] ?? '',
            'webhook_secret' => $config['webhook_secret'] ?? '',
            'enabled' => 1,
            'settings' => json_encode($config['settings'] ?? []),
            'created_at' => date('Y-m-d H:i:s')
        ];
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$integrationsTable . "
            (name, type, endpoint, api_key, webhook_secret, enabled, settings, created_at)
            VALUES (
                '" . db_escape_string($data['name']) . "',
                '" . db_escape_string($data['type']) . "',
                '" . db_escape_string($data['endpoint']) . "',
                '" . db_escape_string($data['api_key']) . "',
                '" . db_escape_string($data['webhook_secret']) . "',
                1,
                '" . db_escape_string($data['settings']) . "',
                NOW()
            )
        ");
        
        return mysql_insert_id();
    }
    
    public static function getIntegration($name)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$integrationsTable . "
            WHERE name = '" . db_escape_string($name) . "'
            AND enabled = 1
        ");
        
        return mysql_fetch_array($result);
    }
    
    public static function getAllIntegrations($type = null)
    {
        $where = $type ? "WHERE type = '" . db_escape_string($type) . "'" : "";
        
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$integrationsTable . "
            " . $where . "
            ORDER BY name ASC
        ");
        
        $integrations = [];
        while ($row = mysql_fetch_array($result)) {
            $integrations[] = $row;
        }
        
        return $integrations;
    }
    
    public static function syncClient($vars)
    {
        $integrations = self::getAllIntegrations('crm');
        
        foreach ($integrations as $integration) {
            self::callApi($integration, 'POST', '/contacts', [
                'email' => $vars['email'] ?? '',
                'first_name' => $vars['firstname'] ?? '',
                'last_name' => $vars['lastname'] ?? '',
                'company' => $vars['companyname'] ?? '',
                'whmcs_client_id' => $vars['userid'] ?? 0
            ]);
        }
    }
    
    public static function notifyExternal($event, $data)
    {
        $integrations = self::getAllIntegrations();
        
        foreach ($integrations as $integration) {
            $settings = json_decode($integration['settings'], true);
            
            if (!empty($settings['events']) && in_array($event, $settings['events'])) {
                self::callApi($integration, 'POST', '/webhooks/' . $event, $data);
            }
        }
    }
    
    public static function syncPayment($vars)
    {
        $integrations = self::getAllIntegrations('accounting');
        
        foreach ($integrations as $integration) {
            self::callApi($integration, 'POST', '/payments', [
                'invoice_id' => $vars['invoiceid'] ?? 0,
                'amount' => $vars['amount'] ?? 0,
                'client_id' => $vars['userid'] ?? 0,
                'payment_method' => $vars['paymentmethod'] ?? 'unknown',
                'date' => date('Y-m-d H:i:s')
            ]);
        }
    }
    
    public static function provisionExternal($vars)
    {
        $integrations = self::getAllIntegrations('provisioning');
        
        foreach ($integrations as $integration) {
            $settings = json_decode($integration['settings'], true);
            
            if (!empty($settings['auto_provision'])) {
                self::callApi($integration, 'POST', '/services', [
                    'client_id' => $vars['userid'] ?? 0,
                    'product_id' => $vars['productid'] ?? 0,
                    'domain' => $vars['domain'] ?? '',
                    'username' => $vars['username'] ?? ''
                ]);
            }
        }
    }
    
    public static function callApi($integration, $method, $path, $data)
    {
        $url = rtrim($integration['endpoint'], '/') . $path;
        
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        
        $headers = [
            'Content-Type: application/json',
            'Authorization: Bearer ' . $integration['api_key']
        ];
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        
        if (!empty($data) && in_array($method, ['POST', 'PUT', 'PATCH'])) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        $error = curl_error($ch);
        curl_close($ch);
        
        self::logApiCall($integration['name'], $method, $path, $data, $response, $httpCode, $error);
        
        return [
            'success' => $httpCode >= 200 && $httpCode < 300,
            'status_code' => $httpCode,
            'response' => $response
        ];
    }
    
    private static function logApiCall($integrationName, $method, $path, $requestData, $response, $statusCode, $error)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$logTable . "
            (integration_name, method, path, request_data, response, status_code, error, created_at)
            VALUES (
                '" . db_escape_string($integrationName) . "',
                '" . db_escape_string($method) . "',
                '" . db_escape_string($path) . "',
                '" . db_escape_string(json_encode($requestData)) . "',
                '" . db_escape_string($response) . "',
                " . (int)$statusCode . ",
                " . ($error ? "'" . db_escape_string($error) . "'" : "NULL") . ",
                NOW()
            )
        ");
    }
    
    public static function handleWebhook($integrationName, $payload, $signature)
    {
        $integration = self::getIntegration($integrationName);
        
        if (!$integration) {
            return ['success' => false, 'error' => 'Integration not found'];
        }
        
        // Verify signature
        $expectedSignature = hash_hmac('sha256', $payload, $integration['webhook_secret']);
        if ($signature !== $expectedSignature) {
            return ['success' => false, 'error' => 'Invalid signature'];
        }
        
        $data = json_decode($payload, true);
        
        // Process webhook based on event type
        $eventType = $data['event'] ?? 'unknown';
        
        switch ($eventType) {
            case 'contact.updated':
                self::processContactUpdate($data);
                break;
                
            case 'payment.synced':
                self::processPaymentSync($data);
                break;
                
            case 'subscription.renewed':
                self::processSubscriptionRenewal($data);
                break;
        }
        
        return ['success' => true];
    }
    
    private function processContactUpdate($data)
    {
        $clientId = $data['whmcs_client_id'] ?? 0;
        
        if ($clientId) {
            full_query("
                UPDATE " . TABLE_PREFIX . "tblclients
                SET firstname = '" . db_escape_string($data['first_name'] ?? '') . "',
                    lastname = '" . db_escape_string($data['last_name'] ?? '') . "',
                    companyname = '" . db_escape_string($data['company'] ?? '') . "'
                WHERE id = " . (int)$clientId
            );
        }
    }
    
    private function processPaymentSync($data)
    {
        // Sync payment data from external source
    }
    
    private function processSubscriptionRenewal($data)
    {
        // Process subscription renewal
    }
    
    public static function getLogs($integrationName = null, $limit = 100)
    {
        $where = $integrationName ? "WHERE integration_name = '" . db_escape_string($integrationName) . "'" : "";
        
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$logTable . "
            " . $where . "
            ORDER BY created_at DESC
            LIMIT " . (int)$limit
        ");
        
        $logs = [];
        while ($row = mysql_fetch_array($result)) {
            $logs[] = $row;
        }
        
        return $logs;
    }
    
    public static function testConnection($integrationId)
    {
        $integration = full_query("SELECT * FROM " . TABLE_PREFIX . self::$integrationsTable . " WHERE id = " . (int)$integrationId);
        $data = mysql_fetch_array($integration);
        
        if (!$data) {
            return ['success' => false, 'error' => 'Integration not found'];
        }
        
        $result = self::callApi($data, 'GET', '/health', []);
        
        return $result;
    }
}

function integration_hub_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_integration_connections (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(100) NOT NULL,
            type VARCHAR(50),
            endpoint VARCHAR(500),
            api_key VARCHAR(255),
            webhook_secret VARCHAR(255),
            enabled TINYINT(1) DEFAULT 1,
            settings TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_integration_logs (
            id INT AUTO_INCREMENT PRIMARY KEY,
            integration_name VARCHAR(100),
            method VARCHAR(10),
            path VARCHAR(255),
            request_data TEXT,
            response TEXT,
            status_code INT,
            error TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("CREATE INDEX idx_integration ON " . TABLE_PREFIX . "mod_integration_logs(integration_name)");
    
    return ['status' => 'success', 'description' => 'Integration Hub activated'];
}

function integration_hub_deactivate()
{
    return ['status' => 'success', 'description' => 'Integration Hub deactivated'];
}

function integration_hub_config()
{
    return [
        'name' => 'Integration Hub',
        'description' => 'Central hub for third-party integrations and API management',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'log_requests' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Log API Requests',
                'Default' => '1'
            ],
            'timeout' => [
                'Type' => 'text',
                'FriendlyName' => 'Request Timeout (seconds)',
                'Default' => '30'
            ]
        ]
    ];
}
```