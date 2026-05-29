# WHMCS Domain Broker Module - DEVKIT

## Module Information
- **Name**: Domain Broker
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Domain backordering, monitoring, and brokerage services

## Installation
1. Copy to `/modules/addons/domain_broker/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('DailyCronJob', 1, function($vars) {
    DomainBroker::checkExpiringDomains();
    DomainBroker::processBackorders();
});

add_hook('DomainTransferCompleted', 1, function($vars) {
    DomainBroker::notifyBackorderCustomers($vars['domain']);
});
```

### includes/DomainBroker.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class DomainBroker
{
    private static $backordersTable = 'mod_domain_broker_backorders';
    private static $monitoringTable = 'mod_domain_broker_monitoring';
    
    public static function createBackorder($domain, $userId, $maxBid)
    {
        $tld = self::extractTld($domain);
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$backordersTable . "
            (domain, user_id, max_bid, tld, status, created_at)
            VALUES (
                '" . db_escape_string(strtolower($domain)) . "',
                " . (int)$userId . ",
                " . (float)$maxBid . ",
                '" . db_escape_string($tld) . "',
                'pending',
                NOW()
            )
        ");
        
        return mysql_insert_id();
    }
    
    public static function addToMonitoring($domain, $userId, $email = true, $sms = false)
    {
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$monitoringTable . "
            (domain, user_id, notify_email, notify_sms, created_at)
            VALUES (
                '" . db_escape_string(strtolower($domain)) . "',
                " . (int)$userId . ",
                " . ($email ? 1 : 0) . ",
                " . ($sms ? 1 : 0) . ",
                NOW()
            )
            ON DUPLICATE KEY UPDATE notify_email = VALUES(notify_email), notify_sms = VALUES(notify_sms)
        ");
    }
    
    public static function checkExpiringDomains()
    {
        // Check for expiring domains in next 30 days
        $result = full_query("
            SELECT d.*, m.user_id, m.notify_email
            FROM " . TABLE_PREFIX . "tbldomains d
            JOIN " . TABLE_PREFIX . self::$monitoringTable . " m ON d.domain = m.domain
            WHERE d.expirydate BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)
            AND d.domainstatus = 'Active'
            AND m.notified = 0
        ");
        
        while ($domain = mysql_fetch_array($result)) {
            self::sendExpiryNotification($domain);
            
            full_query("UPDATE " . TABLE_PREFIX . self::$monitoringTable . " SET notified = 1 WHERE domain = '" . db_escape_string($domain['domain']) . "'");
        }
    }
    
    private static function sendExpiryNotification($domain)
    {
        $client = full_query("SELECT email FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$domain['user_id']);
        $clientData = mysql_fetch_array($client);
        
        if ($clientData && $domain['notify_email']) {
            send_email(
                $clientData['email'],
                'domain_expiry_reminder',
                [
                    'domain' => $domain['domain'],
                    'expiry_date' => $domain['expirydate'],
                    'days_until_expiry' => floor((strtotime($domain['expirydate']) - time()) / 86400)
                ]
            );
        }
    }
    
    public static function processBackorders()
    {
        // Check for newly available domains
        $pending = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$backordersTable . "
            WHERE status = 'pending'
        ");
        
        while ($backorder = mysql_fetch_array($pending)) {
            if (self::isDomainAvailable($backorder['domain'])) {
                self::attemptRegistration($backorder);
            }
        }
    }
    
    private static function isDomainAvailable($domain)
    {
        // Use WHOIS lookup or registrar API to check availability
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, 'https://api.example.com/whois/' . urlencode($domain));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 10);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        $result = json_decode($response, true);
        
        return $result['available'] ?? false;
    }
    
    private static function attemptRegistration($backorder)
    {
        $client = full_query("SELECT * FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$backorder['user_id']);
        $clientData = mysql_fetch_array($client);
        
        if (!$clientData) {
            return false;
        }
        
        // Create order for domain registration
        $orderId = create_order([
            'client_id' => $backorder['user_id'],
            'pid' => 0, // Domain only
            'domain' => $backorder['domain'],
            'billingcycle' => 'annually',
            'paymentmethod' => 'paypal'
        ]);
        
        // Process payment up to max bid
        full_query("
            UPDATE " . TABLE_PREFIX . "mod_domain_broker_backorders
            SET status = 'processing', order_id = " . (int)$orderId . "
            WHERE id = " . (int)$backorder['id']
        );
    }
    
    public static function notifyBackorderCustomers($domain)
    {
        $backorders = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$backordersTable . "
            WHERE domain = '" . db_escape_string($domain) . "'
            AND status = 'pending'
        ");
        
        while ($backorder = mysql_fetch_array($backorders)) {
            $client = full_query("SELECT email, firstname, lastname FROM " . TABLE_PREFIX . "tblclients WHERE id = " . (int)$backorder['user_id']);
            $clientData = mysql_fetch_array($client);
            
            if ($clientData) {
                send_email(
                    $clientData['email'],
                    'backorder_acquired',
                    [
                        'domain' => $domain,
                        'max_bid' => $backorder['max_bid']
                    ]
                );
                
                full_query("
                    UPDATE " . TABLE_PREFIX . "mod_domain_broker_backorders
                    SET status = 'notified', notified_at = NOW()
                    WHERE id = " . (int)$backorder['id']
                );
            }
        }
    }
    
    public static function getBackorders($userId = null)
    {
        $where = $userId ? "WHERE user_id = " . (int)$userId : "";
        
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$backordersTable . "
            " . $where . "
            ORDER BY created_at DESC
        ");
        
        $backorders = [];
        while ($row = mysql_fetch_array($result)) {
            $backorders[] = $row;
        }
        
        return $backorders;
    }
    
    private static function extractTld($domain)
    {
        $parts = explode('.', $domain);
        return '.' . end($parts);
    }
    
    public static function getPricing($tld)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . "mod_domain_broker_pricing
            WHERE tld = '" . db_escape_string($tld) . "'
        ");
        
        return mysql_fetch_array($result);
    }
}

function domain_broker_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_domain_broker_backorders (
            id INT AUTO_INCREMENT PRIMARY KEY,
            domain VARCHAR(255) NOT NULL,
            user_id INT NOT NULL,
            max_bid DECIMAL(10,2),
            tld VARCHAR(50),
            status VARCHAR(20) DEFAULT 'pending',
            order_id INT,
            notified_at DATETIME,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_domain_broker_monitoring (
            id INT AUTO_INCREMENT PRIMARY KEY,
            domain VARCHAR(255) NOT NULL,
            user_id INT NOT NULL,
            notify_email TINYINT(1) DEFAULT 1,
            notify_sms TINYINT(1) DEFAULT 0,
            notified TINYINT(1) DEFAULT 0,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            UNIQUE KEY domain_user (domain, user_id)
        )
    ");
    
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_domain_broker_pricing (
            id INT AUTO_INCREMENT PRIMARY KEY,
            tld VARCHAR(50) NOT NULL,
            backorder_fee DECIMAL(10,2),
            transfer_fee DECIMAL(10,2),
            registration_fee DECIMAL(10,2),
            updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("CREATE INDEX idx_domain ON " . TABLE_PREFIX . "mod_domain_broker_backorders(domain)");
    
    return ['status' => 'success', 'description' => 'Domain Broker activated'];
}

function domain_broker_deactivate()
{
    return ['status' => 'success', 'description' => 'Domain Broker deactivated'];
}

function domain_broker_config()
{
    return [
        'name' => 'Domain Broker',
        'description' => 'Domain backordering, monitoring, and brokerage',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'auto_check' => [
                'Type' => 'yesno',
                'FriendlyName' => 'Auto Check Availability',
                'Default' => '1'
            ],
            'check_interval' => [
                'Type' => 'dropdown',
                'FriendlyName' => 'Check Interval',
                'Options' => ['1' => 'Daily', '7' => 'Weekly'],
                'Default' => '1'
            ]
        ]
    ];
}
```