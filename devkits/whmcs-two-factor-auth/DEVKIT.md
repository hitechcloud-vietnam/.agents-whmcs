# WHMCS Two-Factor Authentication Module

```php
<?php
/**
 * WHMCS Two-Factor Authentication Module
 * 
 * Provides multi-factor authentication with TOTP, SMS, Email,
 * and backup codes support.
 * 
 * @Author: HiTech Cloud DevKit
 * @Version: 1.0.0
 */

// Prevent direct access
if (!defined("WHMCS")) {
    die("Direct access prohibited");
}

function twofactorauth_MetaData()
{
    return array(
        'DisplayName' => 'Two-Factor Authentication',
        'APIVersion' => '1.1',
        'RequiresServer' => false,
    );
}

function twofactorauth_ConfigArray()
{
    return array(
        'FriendlyName' => array(
            'Type' => 'System',
            'Value' => 'Two-Factor Authentication',
        ),
        'EnableTOTP' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable TOTP authenticator',
        ),
        'EnableEmail' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable email verification',
        ),
        'EnableSMS' => array(
            'Type' => 'yesno',
            'Default' => 'no',
            'Description' => 'Enable SMS verification (requires SMS provider)',
        ),
        'EnableBackupCodes' => array(
            'Type' => 'yesno',
            'Default' => 'yes',
            'Description' => 'Enable backup codes',
        ),
        'BackupCodeCount' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '10',
            'Description' => 'Number of backup codes to generate',
        ),
        'CodeExpiry' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '300',
            'Description' => 'Code expiry time in seconds',
        ),
        'MaxAttempts' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '5',
            'Description' => 'Maximum verification attempts before lockout',
        ),
        'LockoutDuration' => array(
            'Type' => 'text',
            'Size' => '10',
            'Default' => '900',
            'Description' => 'Lockout duration in seconds',
        ),
        'RequireForAdmin' => array(
            'Type' => 'yesno',
            'Default' => 'no',
            'Description' => 'Require 2FA for admin accounts',
        ),
        'RequireForClients' => array(
            'Type' => 'yesno',
            'Default' => 'no',
            'Description' => 'Require 2FA for client accounts',
        ),
    );
}

function twofactorauth_activate()
{
    try {
        if (!function_exists('createTable')) {
            require_once dirname(__FILE__) . '/../../includes/modulefunctions.php';
        }
        
        // User 2FA settings table
        $userSettingsTable = 'mod_twofactorauth_user_settings';
        $userSettingsSchema = "
            CREATE TABLE `{$userSettingsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `user_type` ENUM('admin', 'client') NOT NULL,
                `method` VARCHAR(50) NOT NULL,
                `is_enabled` TINYINT(1) DEFAULT 1,
                `totp_secret` VARCHAR(255) NULL,
                `email_confirmed` TINYINT(1) DEFAULT 0,
                `phone_number` VARCHAR(50) NULL,
                `backup_codes` JSON NULL,
                `settings` JSON NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                `updated_at` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                UNIQUE KEY `unique_user` (`user_id`, `user_type`),
                INDEX `idx_method` (`method`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($userSettingsTable, $userSettingsSchema);
        
        // Verification codes table
        $codesTable = 'mod_twofactorauth_codes';
        $codesSchema = "
            CREATE TABLE `{$codesTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `user_type` ENUM('admin', 'client') NOT NULL,
                `code` VARCHAR(255) NOT NULL,
                `method` VARCHAR(50) NOT NULL,
                `attempts` INT DEFAULT 0,
                `expires_at` DATETIME NOT NULL,
                `used_at` DATETIME NULL,
                `ip_address` VARCHAR(45) NULL,
                `created_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_code` (`user_id`, `user_type`, `code`),
                INDEX `idx_expires` (`expires_at`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($codesTable, $codesSchema);
        
        // Login attempts table
        $attemptsTable = 'mod_twofactorauth_attempts';
        $attemptsSchema = "
            CREATE TABLE `{$attemptsTable}` (
                `id` INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
                `user_id` INT NOT NULL,
                `user_type` ENUM('admin', 'client') NOT NULL,
                `ip_address` VARCHAR(45) NOT NULL,
                `method` VARCHAR(50) NOT NULL,
                `success` TINYINT(1) DEFAULT 0,
                `attempted_at` DATETIME DEFAULT CURRENT_TIMESTAMP,
                INDEX `idx_user_attempts` (`user_id`, `user_type`),
                INDEX `idx_ip_attempts` (`ip_address`)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
        ";
        createTable($attemptsTable, $attemptsSchema);
        
        return array(
            'status' => 'success',
            'description' => 'Two-Factor Authentication module activated.',
        );
    } catch (\Exception $e) {
        return array(
            'status' => 'error',
            'description' => 'Failed to activate module: ' . $e->getMessage(),
        );
    }
}

function twofactorauth_deactivate()
{
    return array('status' => 'success', 'description' => 'Module deactivated.');
}

/**
 * Generate TOTP secret
 */
function twofactorauth_GenerateTOTPSecret($length = 32)
{
    $secret = '';
    $chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
    
    for ($i = 0; $i < $length; $i++) {
        $secret .= $chars[random_int(0, strlen($chars) - 1)];
    }
    
    return $secret;
}

/**
 * Generate TOTP code
 */
function twofactorauth_GenerateTOTPCode($secret, $time = null)
{
    $time = $time ?? time();
    $timeSlice = floor($time / 30);
    
    $secretKey = base32_decode($secret);
    $timeBytes = pack('N*', 0) . pack('N*', $timeSlice);
    
    $hash = hash_hmac('sha1', $timeBytes, $secretKey, true);
    
    $offset = ord($hash[19]) & 0xf;
    $binary = (
        ((ord($hash[$offset]) & 0x7f) << 24) |
        ((ord($hash[$offset + 1]) & 0xff) << 16) |
        ((ord($hash[$offset + 2]) & 0xff) << 8) |
        (ord($hash[$offset + 3]) & 0xff)
    );
    
    $otp = $binary % 1000000;
    return str_pad((string)$otp, 6, '0', STR_PAD_LEFT);
}

/**
 * Verify TOTP code
 */
function twofactorauth_VerifyTOTP($secret, $code, $window = 1)
{
    $currentTime = time();
    
    // Allow codes within the window (before and after)
    for ($i = -$window; $i <= $window; $i++) {
        $testTime = $currentTime + ($i * 30);
        $testCode = twofactorauth_GenerateTOTPCode($secret, $testTime);
        
        if (hash_equals($testCode, $code)) {
            return true;
        }
    }
    
    return false;
}

/**
 * Get TOTP QR code URL
 */
function twofactorauth_GetTOTPQRUrl($secret, $email, $issuer = 'WHMCS')
{
    $otpauth = 'otpauth://totp/' . rawurlencode($issuer . ':' . $email);
    $otpauth .= '?secret=' . $secret;
    $otpauth .= '&issuer=' . rawurlencode($issuer);
    $otpauth .= '&algorithm=SHA1';
    $otpauth .= '&digits=6';
    $otpauth .= '&period=30';
    
    return 'https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=' . rawurlencode($otpauth);
}

/**
 * Generate backup codes
 */
function twofactorauth_GenerateBackupCodes($count = 10)
{
    $codes = array();
    
    for ($i = 0; $i < $count; $i++) {
        // Generate 8-character alphanumeric code
        $code = '';
        $chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
        
        for ($j = 0; $j < 8; $j++) {
            $code .= $chars[random_int(0, strlen($chars) - 1)];
        }
        
        // Format as XXXX-XXXX
        $formatted = substr($code, 0, 4) . '-' . substr($code, 4, 4);
        $codes[] = array(
            'code' => strtoupper($formatted),
            'used' => false,
            'hash' => password_hash($code, PASSWORD_DEFAULT),
        );
    }
    
    return $codes;
}

/**
 * Setup user 2FA
 */
function twofactorauth_SetupUser($userId, $userType, $method, $config = array())
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $data = array(
            'user_id' => $userId,
            'user_type' => $userType,
            'method' => $method,
            'is_enabled' => 1,
        );
        
        switch ($method) {
            case 'totp':
                $data['totp_secret'] = $config['secret'] ?? twofactorauth_GenerateTOTPSecret();
                break;
            
            case 'email':
                $data['email_confirmed'] = 1;
                break;
            
            case 'sms':
                $data['phone_number'] = $config['phone_number'];
                break;
        }
        
        if ($config['generate_backup_codes'] ?? true) {
            $data['backup_codes'] = json_encode(twofactorauth_GenerateBackupCodes(10));
        }
        
        Capsule::table('mod_twofactorauth_user_settings')->updateOrInsert(
            array('user_id' => $userId, 'user_type' => $userType),
            $data
        );
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get user 2FA settings
 */
function twofactorauth_GetUserSettings($userId, $userType)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $settings = Capsule::table('mod_twofactorauth_user_settings')
        ->where('user_id', $userId)
        ->where('user_type', $userType)
        ->first();
    
    if ($settings && $settings->backup_codes) {
        $settings->backup_codes = json_decode($settings->backup_codes, true);
    }
    
    if ($settings && $settings->settings) {
        $settings->settings = json_decode($settings->settings, true);
    }
    
    return $settings;
}

/**
 * Check if user has 2FA enabled
 */
function twofactorauth_IsEnabled($userId, $userType)
{
    $settings = twofactorauth_GetUserSettings($userId, $userType);
    return $settings && $settings->is_enabled;
}

/**
 * Disable user 2FA
 */
function twofactorauth_DisableUser($userId, $userType)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        Capsule::table('mod_twofactorauth_user_settings')
            ->where('user_id', $userId)
            ->where('user_type', $userType)
            ->update(array('is_enabled' => 0));
        
        return array('success' => true);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Send verification code
 */
function twofactorauth_SendCode($userId, $userType, $method)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    try {
        $user = twofactorauth_GetUserById($userId, $userType);
        
        if (!$user) {
            return array('success' => false, 'error' => 'User not found');
        }
        
        $code = str_pad((string)random_int(0, 999999), 6, '0', STR_PAD_LEFT);
        $hash = password_hash($code, PASSWORD_DEFAULT);
        
        // Store code
        Capsule::table('mod_twofactorauth_codes')->insert(array(
            'user_id' => $userId,
            'user_type' => $userType,
            'code' => $hash,
            'method' => $method,
            'expires_at' => date('Y-m-d H:i:s', time() + 300),
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
        ));
        
        switch ($method) {
            case 'email':
                twofactorauth_SendEmailCode($user->email, $code);
                break;
            
            case 'sms':
                $settings = twofactorauth_GetUserSettings($userId, $userType);
                if ($settings && $settings->phone_number) {
                    twofactorauth_SendSMSCode($settings->phone_number, $code);
                }
                break;
        }
        
        return array('success' => true, 'expires_in' => 300);
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Send email verification code
 */
function twofactorauth_SendEmailCode($email, $code)
{
    $subject = 'Your Verification Code';
    $body = "Your verification code is: {$code}\n\n";
    $body .= "This code will expire in 5 minutes.\n";
    $body .= "If you didn't request this code, please ignore this email.";
    
    sendEmail($email, $subject, $body);
}

/**
 * Send SMS verification code (placeholder - implement SMS provider)
 */
function twofactorauth_SendSMSCode($phoneNumber, $code)
{
    // Implement SMS provider integration here
    // Example: Twilio, Nexmo, etc.
    
    $message = "Your verification code is: {$code}";
    // SMS sending logic would go here
    
    return true;
}

/**
 * Verify code
 */
function twofactorauth_VerifyCode($userId, $userType, $code, $method = 'totp')
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    try {
        // Check for lockout
        if (twofactorauth_IsLockedOut($userId, $userType)) {
            return array('success' => false, 'error' => 'Account is locked out', 'locked' => true);
        }
        
        $valid = false;
        
        switch ($method) {
            case 'totp':
                $settings = twofactorauth_GetUserSettings($userId, $userType);
                if ($settings && $settings->totp_secret) {
                    $valid = twofactorauth_VerifyTOTP($settings->totp_secret, $code);
                }
                break;
            
            case 'email':
            case 'sms':
                $valid = twofactorauth_VerifyStoredCode($userId, $userType, $code, $method);
                break;
            
            case 'backup':
                $valid = twofactorauth_VerifyBackupCode($userId, $userType, $code);
                break;
        }
        
        // Record attempt
        twofactorauth_RecordAttempt($userId, $userType, $method, $valid);
        
        if ($valid) {
            return array('success' => true);
        } else {
            // Check if max attempts exceeded
            $attempts = twofactorauth_GetFailedAttempts($userId, $userType);
            $maxAttempts = Capsule::table('mod_twofactorauth_config')
                ->where('setting', 'MaxAttempts')
                ->first();
            
            $remaining = ($maxAttempts ? (int)$maxAttempts->value : 5) - $attempts;
            
            return array(
                'success' => false,
                'error' => 'Invalid code',
                'remaining_attempts' => max(0, $remaining),
            );
        }
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Verify stored code (email/SMS)
 */
function twofactorauth_VerifyStoredCode($userId, $userType, $code, $method)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $now = date('Y-m-d H:i:s');
    
    $storedCode = Capsule::table('mod_twofactorauth_codes')
        ->where('user_id', $userId)
        ->where('user_type', $userType)
        ->where('method', $method)
        ->where('expires_at', '>', $now)
        ->whereNull('used_at')
        ->orderBy('created_at', 'desc')
        ->first();
    
    if (!$storedCode) {
        return false;
    }
    
    // Update attempt count
    Capsule::table('mod_twofactorauth_codes')
        ->where('id', $storedCode->id)
        ->update(array('attempts' => $storedCode->attempts + 1));
    
    // Verify code
    if (password_verify($code, $storedCode->code)) {
        // Mark as used
        Capsule::table('mod_twofactorauth_codes')
            ->where('id', $storedCode->id)
            ->update(array('used_at' => $now));
        
        return true;
    }
    
    return false;
}

/**
 * Verify backup code
 */
function twofactorauth_VerifyBackupCode($userId, $userType, $code)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    // Normalize code (remove dashes)
    $code = str_replace('-', '', strtoupper($code));
    
    $settings = twofactorauth_GetUserSettings($userId, $userType);
    
    if (!$settings || !$settings->backup_codes) {
        return false;
    }
    
    foreach ($settings->backup_codes as &$backupCode) {
        if (!$backupCode['used']) {
            // Verify against stored hash
            if (password_verify($code, $backupCode['hash'])) {
                $backupCode['used'] = true;
                $backupCode['used_at'] = date('Y-m-d H:i:s');
                
                // Update stored codes
                Capsule::table('mod_twofactorauth_user_settings')
                    ->where('user_id', $userId)
                    ->where('user_type', $userType)
                    ->update(array('backup_codes' => json_encode($settings->backup_codes)));
                
                return true;
            }
        }
    }
    
    return false;
}

/**
 * Get user by ID
 */
function twofactorauth_GetUserById($userId, $userType)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    if ($userType === 'admin') {
        return Capsule::table('tbladmins')
            ->where('id', $userId)
            ->first();
    } else {
        return Capsule::table('tblclients')
            ->where('id', $userId)
            ->first();
    }
}

/**
 * Record verification attempt
 */
function twofactorauth_RecordAttempt($userId, $userType, $method, $success)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    Capsule::table('mod_twofactorauth_attempts')->insert(array(
        'user_id' => $userId,
        'user_type' => $userType,
        'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
        'method' => $method,
        'success' => $success ? 1 : 0,
    ));
}

/**
 * Get failed attempts count
 */
function twofactorauth_GetFailedAttempts($userId, $userType)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    $since = date('Y-m-d H:i:s', time() - 900); // 15 minutes
    
    return Capsule::table('mod_twofactorauth_attempts')
        ->where('user_id', $userId)
        ->where('user_type', $userType)
        ->where('success', 0)
        ->where('attempted_at', '>=', $since)
        ->count();
}

/**
 * Check if user is locked out
 */
function twofactorauth_IsLockedOut($userId, $userType)
{
    $maxAttempts = 5;
    $attempts = twofactorauth_GetFailedAttempts($userId, $userType);
    
    return $attempts >= $maxAttempts;
}

/**
 * Regenerate backup codes
 */
function twofactorauth_RegenerateBackupCodes($userId, $userType)
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) -> '/../../includesWHMCS.php';
    }
    
    try {
        $codes = twofactorauth_GenerateBackupCodes(10);
        
        Capsule::table('mod_twofactorauth_user_settings')
            ->where('user_id', $userId)
            ->where('user_type', $userType)
            ->update(array('backup_codes' => json_encode($codes)));
        
        return array('success' => true, 'codes' => array_column($codes, 'code'));
    } catch (\Exception $e) {
        return array('success' => false, 'error' => $e->getMessage());
    }
}

/**
 * Get remaining backup codes
 */
function twofactorauth_GetRemainingBackupCodes($userId, $userType)
{
    $settings = twofactorauth_GetUserSettings($userId, $userType);
    
    if (!$settings || !$settings->backup_codes) {
        return 0;
    }
    
    return count(array_filter($settings->backup_codes, function($code) {
        return !$code['used'];
    }));
}

/**
 * Validate setup (verify TOTP code during setup)
 */
function twofactorauth_ValidateSetup($userId, $userType, $secret, $code)
{
    $valid = twofactorauth_VerifyTOTP($secret, $code);
    
    if ($valid) {
        twofactorauth_SetupUser($userId, $userType, 'totp', array('secret' => $secret));
    }
    
    return $valid;
}

/**
 * Get available methods
 */
function twofactorauth_GetAvailableMethods()
{
    if (!function_exists('Capsule')) {
        require_once dirname(__FILE__) . '/../../includesWHMCS.php';
    }
    
    $methods = array();
    
    $config = Capsule::table('mod_twofactorauth_config')
        ->whereIn('setting', array('EnableTOTP', 'EnableEmail', 'EnableSMS', 'EnableBackupCodes'))
        ->get();
    
    foreach ($config as $item) {
        switch ($item->setting) {
            case 'EnableTOTP':
                $methods['totp'] = (bool) $item->value;
                break;
            case 'EnableEmail':
                $methods['email'] = (bool) $item->value;
                break;
            case 'EnableSMS':
                $methods['sms'] = (bool) $item->value;
                break;
            case 'EnableBackupCodes':
                $methods['backup'] = (bool) $item->value;
                break;
        }
    }
    
    return $methods;
}

// Base32 encoding helper
function base32_decode($encoded)
{
    $base32Chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ234567';
    $encoded = strtoupper($encoded);
    $encoded = str_replace(array('=', '-'), '', $encoded);
    
    $output = '';
    $buffer = 0;
    $bitsLeft = 0;
    
    foreach (str_split($encoded) as $char) {
        $value = strpos($base32Chars, $char);
        if ($value === false) continue;
        
        $buffer = ($buffer << 5) | $value;
        $bitsLeft += 5;
        
        if ($bitsLeft >= 8) {
            $bitsLeft -= 8;
            $output .= chr(($buffer >> $bitsLeft) & 0xFF);
        }
    }
    
    return $output;
}
