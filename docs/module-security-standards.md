# WHMCS Module Security Standards

**Version:** 8.0 | **Updated:** 2026-05-28

## Overview

This guide establishes security standards for all WHMCS modules, including payment gateways, registrars, addons, and notification providers. Following these standards ensures module security and protects sensitive customer data.

---

## Input Validation

### Rule 1: Validate All User Input

```php
/**
 * Validate and sanitize input
 * 
 * @param mixed $input Raw input
 * @param string $type Expected type
 * @param array $options Validation options
 * @return mixed Sanitized input or false
 */
function validateModuleInput($input, $type = 'string', $options = [])
{
    switch ($type) {
        case 'string':
            $sanitized = filter_var($input, FILTER_SANITIZE_STRING);
            if (isset($options['max_length'])) {
                $sanitized = substr($sanitized, 0, $options['max_length']);
            }
            return $sanitized;
            
        case 'integer':
            return filter_var($input, FILTER_VALIDATE_INT);
            
        case 'email':
            return filter_var($input, FILTER_VALIDATE_EMAIL) ? $input : false;
            
        case 'url':
            $sanitized = filter_var($input, FILTER_SANITIZE_URL);
            if (!filter_var($sanitized, FILTER_VALIDATE_URL)) {
                return false;
            }
            // Verify URL is safe
            if (!preg_match('/^https?:\/\//', $sanitized)) {
                return false;
            }
            return $sanitized;
            
        case 'phone':
            $sanitized = preg_replace('/[^0-9+]/', '', $input);
            if (strlen($sanitized) < 7 || strlen($sanitized) > 20) {
                return false;
            }
            return $sanitized;
            
        case 'array':
            if (!is_array($input)) {
                return false;
            }
            if (isset($options['schema'])) {
                return validateArraySchema($input, $options['schema']);
            }
            return $input;
            
        case 'boolean':
            return filter_var($input, FILTER_VALIDATE_BOOLEAN, FILTER_NULL_ON_FAILURE);
            
        default:
            return false;
    }
}

/**
 * Validate array structure
 * 
 * @param array $input Input array
 * @param array $schema Schema definition
 * @return array|false Validated array or false
 */
function validateArraySchema($input, $schema)
{
    $validated = [];
    
    foreach ($schema as $key => $type) {
        if (!isset($input[$key])) {
            if (!empty($schema['required'])) {
                return false;
            }
            continue;
        }
        
        $result = validateModuleInput($input[$key], $type);
        if ($result === false) {
            return false;
        }
        
        $validated[$key] = $result;
    }
    
    return $validated;
}
```

### Rule 2: Use WHMCS Security Functions

```php
/**
 * Use WHMCS security functions
 */

// Escape output for HTML
$escapedOutput = WHMCS\Common\Utility\Environment\WebHelper::decode($dirtyHtml);
$safeOutput    = htmlspecialchars($escapedOutput, ENT_QUOTES, 'UTF-8');

// Use prepared statements
$stmt =WHMCS\Database\Capsule::connection()->prepare(
    "SELECT * FROM {tbl} WHERE id = ? AND status = ?"
);
$stmt->execute([$invoiceId, 'active']);
$results = $stmt->fetchAll();

// Fetch safe client data
$client = WHMCS\User\Client::find($clientId);
```

## Authentication and Authorization

### Rule 3: Implement Proper Access Control

```php
/**
 * Admin permission check
 * 
 * @param string $permission Required permission
 * @param int $adminId Admin ID
 * @return bool Has permission
 */
function checkModulePermission($permission, $adminId = 0)
{
    if (!function_exists('checkPermission')) {
        require_once ROOTDIR . '/includes/adminfunctions.php';
    }
    
    if ($adminId === 0) {
        $adminId = $_SESSION['adminid'] ?? 0;
    }
    
    return checkPermission($permission, true, $adminId);
}

/**
 * Client ownership verification
 * 
 * @param int $clientId Client ID
 * @param int $resourceOwnerId Resource owner ID
 * @return bool Is owner
 */
function verifyClientOwnership($clientId, $resourceOwnerId)
{
    // Get user's clients
    $clients = WHMCS\Session::get('clients');
    
    if (is_array($clients)) {
        return in_array($resourceOwnerId, $clients);
    }
    
    return $clientId === $resourceOwnerId;
}

/**
 * Validate admin session
 * 
 * @return bool Valid session
 */
function validateAdminSession()
{
    if (!$_SESSION['adminid'] ?? null) {
        return false;
    }
    
    if (!($_SESSION['adminlogin'] ?? false)) {
        return false;
    }
    
    return true;
}
```

## API Security

### Rule 4: Secure API Communication

```php
/**
 * Make secure API call
 * 
 * @param string $method HTTP method
 * @param string $url API endpoint
 * @param array $data Request data
 * @param array $credentials API credentials
 * @return array API response
 */
function secureApiCall($method, $url, $data = [], $credentials = [])
{
    // Validate URL is HTTPS
    $parsedUrl = parse_url($url);
    if (($parsedUrl['scheme'] ?? '') !== 'https') {
        throw new \Exception('HTTPS required for API calls');
    }
    
    $ch = curl_init();
    
    $headers = [
        'Content-Type: application/json',
        'Accept: application/json',
        'User-Agent: WHMCS/' . WHMCS_VERSION,
    ];
    
    // Add authentication
    if (!empty($credentials['api_key'])) {
        $timestamp = time();
        $signature = hash_hmac('sha256', $timestamp . $credentials['api_key'], $credentials['api_secret'] ?? '');
        
        $headers[] = 'X-API-Key: ' . $credentials['api_key'];
        $headers[] = 'X-Timestamp: ' . $timestamp;
        $headers[] = 'X-Signature: ' . $signature;
    }
    
    curl_setopt_array($ch, [
        CURLOPT_URL            => $url,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 30,
        CURLOPT_HTTPHEADER     => $headers,
        CURLOPT_SSL_VERIFYPEER => true,
        CURLOPT_SSL_VERIFYHOST => 2,
        CURLOPT_FOLLOWLOCATION => false,
    ]);
    
    // Set request body
    if (in_array($method, ['POST', 'PUT', 'PATCH'])) {
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, $method);
        if (!empty($data)) {
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }
    }
    
    // Set timeouts
    curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 10);
    curl_setopt($ch, CURLOPT_TIMEOUT, 30);
    
    $response = curl_exec($ch);
    
    if ($response === false) {
        $error = curl_error($ch);
        curl_close($ch);
        throw new \Exception('API call failed: ' . $error);
    }
    
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    return [
        'body'      => json_decode($response, true) ?? $response,
        'http_code' => $httpCode,
    ];
}
```

## Signature Verification

### Rule 5: Verify All Webhook/Callback Signatures

```php
/**
 * Verify webhook signature using HMAC
 * 
 * @param string $payload Raw payload
 * @param string $signature Provided signature
 * @param string $secret Shared secret
 * @param string $algorithm Hash algorithm
 * @return bool Valid signature
 */
function verifyWebhookSignature($payload, $signature, $secret, $algorithm = 'sha256')
{
    // Constant-time comparison to prevent timing attacks
    $expectedSignature = hash_hmac($algorithm, $payload, $secret);
    
    // Use hash_equals for constant-time comparison
    if (function_exists('hash_equals')) {
        return hash_equals($expectedSignature, $signature);
    }
    
    // Fallback for older PHP versions
    $expectedSigLen = strlen($expectedSignature);
    $sigLen = strlen($signature);
    
    if ($expectedSigLen !== $sigLen) {
        return false;
    }
    
    $result = 0;
    for ($i = 0; $i < $expectedSigLen; $i++) {
        $result |= ord($expectedSignature[$i]) ^ ord($signature[$i]);
    }
    
    return $result === 0;
}

/**
 * Verify PayPal-style IPN signature
 * 
 * @param array $data POST data
 * @param string $endpoint Verification URL
 * @param string $apiUsername API username
 * @return bool Valid signature
 */
function verifyPayPalSignature($data, $endpoint, $apiUsername)
{
    $postData = http_build_query($data);
    $postData .= '&cmd=_notify-validate';
    
    $ch = curl_init($endpoint);
    curl_setopt_array($ch, [
        CURLOPT_POST          => true,
        CURLOPT_POSTFIELDS    => $postData,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT       => 30,
    ]);
    
    $response = curl_exec($ch);
    curl_close($ch);
    
    return trim($response) === 'VERIFIED';
}
```

## Credential Storage

### Rule 6: Secure Credential Handling

```php
/**
 * Encrypt sensitive data
 * 
 * @param string $data Plain text data
 * @return string Encrypted data
 */
function encryptModuleData($data)
{
    $cipher = 'aes-256-cbc';
    $encryptionKey = \WHMCS\Utility\Environment\WebHelper::getSslCompatibilityHash();
    
    $iv = openssl_random_pseudo_bytes(openssl_cipher_iv_length($cipher));
    $encrypted = openssl_encrypt($data, $cipher, $encryptionKey, 0, $iv);
    
    return base64_encode($iv . $encrypted);
}

/**
 * Decrypt sensitive data
 * 
 * @param string $data Encrypted data
 * @return string Plain text data
 */
function decryptModuleData($data)
{
    $cipher = 'aes-256-cbc';
    $encryptionKey = \WHMCS\Utility\Environment\WebHelper::getSslCompatibilityHash();
    
    $data = base64_decode($data);
    $ivLength = openssl_cipher_iv_length($cipher);
    $iv = substr($data, 0, $ivLength);
    $encrypted = substr($data, $ivLength);
    
    return openssl_decrypt($encrypted, $cipher, $encryptionKey, 0, $iv);
}

/**
 * Mask sensitive values in logs
 * 
 * @param array $data Data to log
 * @return array Masked data
 */
function maskSensitiveData($data)
{
    $sensitiveKeys = [
        'password', 'secret', 'api_key', 'apiSecret', 'token', 
        'credit_card', 'cvv', 'pin', 'ssn',
    ];
    
    foreach ($data as $key => $value) {
        $lowerKey = strtolower($key);
        foreach ($sensitiveKeys as $sensitiveKey) {
            if (strpos($lowerKey, strtolower($sensitiveKey)) !== false) {
                $data[$key] = '******';
                break;
            }
        }
    }
    
    return $data;
}
```

## CSRF Protection

### Rule 7: Implement CSRF Tokens

```php
/**
 * Generate CSRF token
 * 
 * @param string $action Action identifier
 * @return string CSRF token
 */
function generateCsrfToken($action)
{
    $sessionKey = 'csrf_token_' . $action;
    
    if (empty($_SESSION[$sessionKey])) {
        $_SESSION[$sessionKey] = bin2hex(random_bytes(32));
    }
    
    return $_SESSION[$sessionKey];
}

/**
 * Validate CSRF token
 * 
 * @param string $action Action identifier
 * @param string $token Token to validate
 * @return bool Valid token
 */
function validateCsrfToken($action, $token)
{
    $sessionKey = 'csrf_token_' . $action;
    $storedToken = $_SESSION[$sessionKey] ?? '';
    
    // Use constant-time comparison
    if (function_exists('hash_equals')) {
        $valid = hash_equals($storedToken, $token);
    } else {
        $valid = $storedToken === $token;
    }
    
    // Regenerate token after validation
    if ($valid) {
        $_SESSION[$sessionKey] = bin2hex(random_bytes(32));
    }
    
    return $valid;
}

/**
 * Include CSRF token in form
 * 
 * @param string $action Action identifier
 * @return string Hidden input HTML
 */
function csrfInputField($action)
{
    $token = generateCsrfToken($action);
    return '<input type="hidden" name="csrf_token" value="' . htmlspecialchars($token) . '">';
}
```

## SQL Injection Prevention

### Rule 8: Use Parameterized Queries

```php
/**
 * WRONG - SQL Injection vulnerable
 */
function vulnerableQuery($userId)
{
    $query = "SELECT * FROM clients WHERE id = {$userId}";
    $result = full_query($query); // DANGEROUS!
}

/**
 * RIGHT - Parameterized query
 */
function safeQuery($userId)
{
    $query = "SELECT * FROM clients WHERE id = ?";
    $stmt = WHMCS\Database\Capsule::connection()->prepare($query);
    $stmt->execute([$userId]);
    return $stmt->fetchAll();
}

/**
 * Parameterized query with multiple parameters
 */
function safeMultiParamQuery($status, $limit)
{
    $query = "SELECT * FROM invoices WHERE status = ? ORDER BY id DESC LIMIT ?";
    $stmt = WHMCS\Database\Capsule::connection()->prepare($query);
    $stmt->execute([$status, $limit]);
    return $stmt->fetchAll();
}

/**
 * Using WHMCS query builder
 */
function usingQueryBuilder($clientId)
{
    $results = WHMCS\Database\Capsule::table('mod_your_table')
        ->where('client_id', $clientId)
        ->where('status', 'active')
        ->get();
    
    return $results;
}
```

## File Upload Security

### Rule 9: Validate File Uploads

```php
/**
 * Validate file upload
 * 
 * @param array $file $_FILES entry
 * @param array $allowedTypes Mime types
 * @param int $maxSize Maximum size in bytes
 * @return array Validation result
 */
function validateFileUpload($file, $allowedTypes = [], $maxSize = 10485760)
{
    // Check for upload errors
    if ($file['error'] !== UPLOAD_ERR_OK) {
        return [
            'success' => false,
            'error'   => 'Upload error: ' . $file['error'],
        ];
    }
    
    // Check file size
    if ($file['size'] > $maxSize) {
        return [
            'success' => false,
            'error'   => 'File too large',
        ];
    }
    
    // Check MIME type
    $finfo = finfo_open(FILEINFO_MIME_TYPE);
    $mimeType = finfo_file($finfo, $file['tmp_name']);
    finfo_close($finfo);
    
    if (!empty($allowedTypes) && !in_array($mimeType, $allowedTypes)) {
        return [
            'success' => false,
            'error'   => 'Invalid file type',
        ];
    }
    
    // Verify file content
    $allowedExtensions = ['jpg', 'jpeg', 'png', 'gif', 'pdf'];
    $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
    
    if (!in_array($extension, $allowedExtensions)) {
        return [
            'success' => false,
            'error'   => 'Invalid extension',
        ];
    }
    
    return [
        'success'    => true,
        'mime_type'  => $mimeType,
        'size'       => $file['size'],
        'extension'  => $extension,
    ];
}

/**
 * Store uploaded file securely
 * 
 * @param array $file Validated file
 * @param string $uploadDir Upload directory
 * @return string Saved file path
 */
function storeUploadedFile($file, $uploadDir)
{
    // Generate random filename
    $extension = strtolower(pathinfo($file['name'], PATHINFO_EXTENSION));
    $newFilename = bin2hex(random_bytes(16)) . '.' . $extension;
    
    // Ensure directory exists and is secure
    if (!is_dir($uploadDir)) {
        mkdir($uploadDir, 0755, true);
    }
    
    $targetPath = rtrim($uploadDir, '/') . '/' . $newFilename;
    
    if (!move_uploaded_file($file['tmp_name'], $targetPath)) {
        throw new \Exception('Failed to save uploaded file');
    }
    
    return $targetPath;
}
```

## Security Checklist

### Pre-Deployment

- [ ] All user input validated and sanitized
- [ ] All database queries use parameterized statements
- [ ] API calls use HTTPS only
- [ ] Webhook signatures properly verified
- [ ] CSRF tokens implemented for all forms
- [ ] Sensitive data properly encrypted
- [ ] No sensitive data logged
- [ ] File uploads properly validated
- [ ] Access controls implemented
- [ ] Security review performed

### Runtime

- [ ] IP whitelist support for admin areas
- [ ] Rate limiting on public endpoints
- [ ] Session timeout enforcement
- [ ] Failed login tracking
- [ ] Audit logging enabled

---

## Related Skills and Workflows

- `security-best-practices` - General security guidelines
- `security-audit-checklist` - Pre-deployment audit
- `module-logging-guide` - Logging with masked sensitive data
- `module-error-handling-guide` - Error handling without data leakage
- `payment-gateway-developer-guide` - Gateway security
- `config-constants-reference` - Security-related configuration
