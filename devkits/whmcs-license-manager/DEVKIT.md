# WHMCS License Manager Module - DEVKIT

## Module Information
- **Name**: License Manager
- **Version**: 1.0.0
- **Type**: Addon Module
- **Description**: Generate and manage software licenses for products

## Installation
1. Copy to `/modules/addons/license_manager/`
2. Activate via WHMCS Admin

## hooks.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

add_hook('ServiceCreated', 1, function($vars) {
    LicenseManager::generateLicense($vars);
});

add_hook('ServiceTerminated', 1, function($vars) {
    LicenseManager::revokeLicense($vars);
});

add_hook('DailyCronJob', 1, function($vars) {
    LicenseManager::checkExpiringLicenses();
});
```

### includes/LicenseManager.php
```php
<?php
if (!defined("WHMCS")) {
    die("This file cannot be accessed directly");
}

class LicenseManager
{
    private static $licensesTable = 'mod_license_manager';
    
    public static function generateLicense($vars)
    {
        $serviceId = $vars['serviceid'] ?? 0;
        $productId = $vars['productid'] ?? 0;
        
        // Check if product requires license generation
        $product = full_query("SELECT * FROM " . TABLE_PREFIX . "tblproducts WHERE id = " . (int)$productId);
        $productData = mysql_fetch_array($product);
        
        if (!$productData || !($productData['license_enabled'] ?? false)) {
            return null;
        }
        
        $licenseKey = self::generateKey($productData['license_type'] ?? 'standard');
        
        full_query("
            INSERT INTO " . TABLE_PREFIX . self::$licensesTable . "
            (service_id, product_id, license_key, license_type, status, valid_until, created_at)
            VALUES (
                " . (int)$serviceId . ",
                " . (int)$productId . ",
                '" . db_escape_string($licenseKey) . "',
                '" . db_escape_string($productData['license_type'] ?? 'standard') . "',
                'active',
                DATE_ADD(NOW(), INTERVAL " . (int)($productData['license_duration'] ?? 12) . " MONTH),
                NOW()
            )
        ");
        
        return $licenseKey;
    }
    
    private static function generateKey($type)
    {
        switch ($type) {
            case 'standard':
                return self::generateStandardKey();
            case 'extended':
                return self::generateExtendedKey();
            case 'enterprise':
                return self::generateEnterpriseKey();
            default:
                return self::generateStandardKey();
        }
    }
    
    private static function generateStandardKey()
    {
        $chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
        $segments = [];
        
        for ($i = 0; $i < 4; $i++) {
            $segment = '';
            for ($j = 0; $j < 5; $j++) {
                $segment .= $chars[rand(0, strlen($chars) - 1)];
            }
            $segments[] = $segment;
        }
        
        return implode('-', $segments);
    }
    
    private static function generateExtendedKey()
    {
        return strtoupper(md5(time() . rand(1000, 9999)));
    }
    
    private static function generateEnterpriseKey()
    {
        $prefix = 'ENT';
        $middle = strtoupper(substr(md5(rand()), 0, 16));
        $suffix = strtoupper(substr(md5(rand()), 0, 8));
        
        return $prefix . '-' . $middle . '-' . $suffix;
    }
    
    public static function revokeLicense($vars)
    {
        $serviceId = $vars['serviceid'] ?? 0;
        
        full_query("
            UPDATE " . TABLE_PREFIX . self::$licensesTable . "
            SET status = 'revoked', revoked_at = NOW()
            WHERE service_id = " . (int)$serviceId . "
            AND status = 'active'
        ");
    }
    
    public static function validateLicense($licenseKey)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$licensesTable . "
            WHERE license_key = '" . db_escape_string($licenseKey) . "'
        ");
        
        $license = mysql_fetch_array($result);
        
        if (!$license) {
            return ['valid' => false, 'error' => 'License not found'];
        }
        
        if ($license['status'] != 'active') {
            return ['valid' => false, 'error' => 'License is ' . $license['status']];
        }
        
        if (strtotime($license['valid_until']) < time()) {
            // Mark as expired
            full_query("
                UPDATE " . TABLE_PREFIX . self::$licensesTable . "
                SET status = 'expired'
                WHERE id = " . (int)$license['id']
            );
            
            return ['valid' => false, 'error' => 'License has expired'];
        }
        
        // Update last validation
        full_query("
            UPDATE " . TABLE_PREFIX . self::$licensesTable . "
            SET last_validated = NOW(), validation_count = validation_count + 1
            WHERE id = " . (int)$license['id']
        );
        
        return [
            'valid' => true,
            'license' => $license
        ];
    }
    
    public static function renewLicense($licenseKey, $months = 12)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$licensesTable . "
            WHERE license_key = '" . db_escape_string($licenseKey) . "'
        ");
        
        $license = mysql_fetch_array($result);
        
        if (!$license) {
            return false;
        }
        
        $newExpiry = date('Y-m-d H:i:s', strtotime('+' . $months . ' months', strtotime($license['valid_until'])));
        
        full_query("
            UPDATE " . TABLE_PREFIX . self::$licensesTable . "
            SET valid_until = '" . db_escape_string($newExpiry) . "',
                status = 'active',
                renewal_count = renewal_count + 1,
                last_renewed = NOW()
            WHERE id = " . (int)$license['id']
        ");
        
        return true;
    }
    
    public static function checkExpiringLicenses()
    {
        // Find licenses expiring in next 30 days
        $result = full_query("
            SELECT l.*, c.email, c.firstname, c.lastname, h.domain
            FROM " . TABLE_PREFIX . self::$licensesTable . " l
            JOIN " . TABLE_PREFIX . "tblhosting h ON l.service_id = h.id
            JOIN " . TABLE_PREFIX . "tblclients c ON h.userid = c.id
            WHERE l.status = 'active'
            AND l.valid_until BETWEEN CURDATE() AND DATE_ADD(CURDATE(), INTERVAL 30 DAY)
            AND l.notified = 0
        ");
        
        while ($license = mysql_fetch_array($result)) {
            self::sendExpiryNotification($license);
            
            full_query("
                UPDATE " . TABLE_PREFIX . self::$licensesTable . "
                SET notified = 1
                WHERE id = " . (int)$license['id']
            );
        }
    }
    
    private static function sendExpiryNotification($license)
    {
        send_email(
            $license['email'],
            'license_expiry_reminder',
            [
                'license_key' => $license['license_key'],
                'expiry_date' => $license['valid_until'],
                'domain' => $license['domain'],
                'days_remaining' => ceil((strtotime($license['valid_until']) - time()) / 86400)
            ]
        );
    }
    
    public static function getLicensesByService($serviceId)
    {
        $result = full_query("
            SELECT * FROM " . TABLE_PREFIX . self::$licensesTable . "
            WHERE service_id = " . (int)$serviceId . "
            ORDER BY created_at DESC
        ");
        
        $licenses = [];
        while ($row = mysql_fetch_array($result)) {
            $licenses[] = $row;
        }
        
        return $licenses;
    }
}

function license_manager_activate()
{
    full_query("
        CREATE TABLE IF NOT EXISTS " . TABLE_PREFIX . "mod_license_manager (
            id INT AUTO_INCREMENT PRIMARY KEY,
            service_id INT NOT NULL,
            product_id INT NOT NULL,
            license_key VARCHAR(100) NOT NULL,
            license_type VARCHAR(50) DEFAULT 'standard',
            status VARCHAR(20) DEFAULT 'active',
            valid_until DATETIME,
            last_validated DATETIME,
            validation_count INT DEFAULT 0,
            renewal_count INT DEFAULT 0,
            notified TINYINT(1) DEFAULT 0,
            revoked_at DATETIME,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ");
    
    full_query("CREATE UNIQUE INDEX idx_license_key ON " . TABLE_PREFIX . "mod_license_manager(license_key)");
    full_query("CREATE INDEX idx_service ON " . TABLE_PREFIX . "mod_license_manager(service_id)");
    
    return ['status' => 'success', 'description' => 'License Manager activated'];
}

function license_manager_deactivate()
{
    return ['status' => 'success', 'description' => 'License Manager deactivated'];
}

function license_manager_config()
{
    return [
        'name' => 'License Manager',
        'description' => 'Generate and manage software licenses',
        'version' => '1.0.0',
        'author' => 'Your Name',
        'fields' => [
            'default_duration' => [
                'Type' => 'text',
                'FriendlyName' => 'Default License Duration (Months)',
                'Default' => '12'
            ],
            'notify_before_days' => [
                'Type' => 'text',
                'FriendlyName' => 'Notify Before (Days)',
                'Default' => '30'
            ]
        ]
    ];
}
```