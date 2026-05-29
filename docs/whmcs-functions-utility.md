# WHMCS Utility Functions

Complete reference for WHMCS utility and helper functions.

## Overview

WHMCS provides various utility functions for common operations like ID generation, array handling, and string manipulation.

## ID Generation Functions

### generateTicketMask()

Generates a unique ticket mask ID.

```php
/**
 * Generate a unique ticket mask
 * 
 * @return string Ticket mask (e.g., ABC-123-45678)
 */
function generateTicketMask(): string
{
    $chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ';
    $numbers = '23456789';
    
    $prefix = '';
    for ($i = 0; $i < 3; $i++) {
        $prefix .= $chars[random_int(0, strlen($chars) - 1)];
    }
    
    $middle = '';
    for ($i = 0; $i < 3; $i++) {
        $middle .= $numbers[random_int(0, strlen($numbers) - 1)];
    }
    
    $suffix = '';
    for ($i = 0; $i < 5; $i++) {
        $suffix .= $numbers[random_int(0, strlen($numbers) - 1)];
    }
    
    return $prefix . '-' . $middle . '-' . $suffix;
}
```

### generateOrderNumber()

Generates a unique order number.

```php
/**
 * Generate a unique order number
 * 
 * @return string Order number
 */
function generateOrderNumber(): string
{
    $prefix = Config\Setting::getValue('OrderNumberPrefix') ?: 'ORD';
    $nextNumber = (int) Config\Setting::getValue('OrderNumberCounter') ?: 1;
    
    // Increment counter atomically
    Capsule::table('tblconfiguration')
        ->where('setting', 'OrderNumberCounter')
        ->update(['value' => $nextNumber + 1]);
    
    return $prefix . '-' . date('Y') . '-' . str_pad($nextNumber, 6, '0', STR_PAD_LEFT);
}
```

### generateUniqueId()

Generates a unique identifier.

```php
/**
 * Generate unique ID
 * 
 * @param int $length ID length
 * @param string $prefix Prefix for the ID
 * @return string Unique ID
 */
function generateUniqueId(int $length = 16, string $prefix = ''): string
{
    $id = bin2hex(random_bytes($length / 2));
    return $prefix . $id;
}
```

## Array Utility Functions

### arrayFilterRecursive()

Recursively filters an array.

```php
/**
 * Recursively filter array values
 * 
 * @param array $array Array to filter
 * @param callable $callback Filter callback
 * @return array Filtered array
 */
function arrayFilterRecursive(array $array, callable $callback): array
{
    foreach ($array as $key => $value) {
        if (is_array($value)) {
            $array[$key] = arrayFilterRecursive($value, $callback);
        } else {
            if (!$callback($value, $key)) {
                unset($array[$key]);
            }
        }
    }
    
    return $array;
}
```

**Example:**
```php
$result = arrayFilterRecursive($data, function($value, $key) {
    return !empty($value) && $value !== null;
});
```

### arrayColumnRecursive()

Gets values from multi-dimensional arrays.

```php
/**
 * Get column values from nested arrays
 * 
 * @param array $array Source array
 * @param string $column Key to extract
 * @return array Extracted values
 */
function arrayColumnRecursive(array $array, string $column): array
{
    $result = [];
    
    foreach ($array as $item) {
        if (is_array($item) && isset($item[$column])) {
            $result[] = $item[$column];
        }
    }
    
    return $result;
}
```

### buildMultiDropdownArray()

Builds multi-select dropdown options.

```php
/**
 * Build multi-select options array
 * 
 * @param array $options Available options
 * @param array $selected Currently selected
 * @return array Options with selected flag
 */
function buildMultiDropdownArray(array $options, array $selected = []): array
{
    return array_map(function($option) use ($selected) {
        return [
            'value' => $option,
            'label' => $option,
            'selected' => in_array($option, $selected)
        ];
    }, $options);
}
```

## String Utility Functions

### sanitize()

Sanitizes a string for safe output.

```php
/**
 * Sanitize string for display
 * 
 * @param string $string String to sanitize
 * @param bool $html Allow HTML
 * @return string Sanitized string
 */
function sanitize(string $string, bool $html = false): string
{
    if ($html) {
        return htmlspecialchars($string, ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }
    
    return htmlspecialchars(strip_tags($string), ENT_QUOTES | ENT_HTML5, 'UTF-8');
}
```

### truncate()

Truncates a string to specified length.

```php
/**
 * Truncate string with ellipsis
 * 
 * @param string $string String to truncate
 * @param int $length Maximum length
 * @param string $ellipsis Ellipsis string
 * @return string Truncated string
 */
function truncate(string $string, int $length = 100, string $ellipsis = '...'): string
{
    if (strlen($string) <= $length) {
        return $string;
    }
    
    return substr($string, 0, $length - strlen($ellipsis)) . $ellipsis;
}
```

### slugify()

Converts string to URL-friendly slug.

```php
/**
 * Convert string to slug
 * 
 * @param string $string String to slugify
 * @param string $separator Separator character
 * @return string Slug
 */
function slugify(string $string, string $separator = '-'): string
{
    $string = strtolower(trim($string));
    $string = preg_replace('/[^a-z0-9]+/', $separator, $string);
    return trim($string, $separator);
}
```

**Example:**
```php
echo slugify('Hello World!'); // "hello-world"
echo slugify('Product & Service'); // "product-service"
```

### randomString()

Generates a random string.

```php
/**
 * Generate random string
 * 
 * @param int $length String length
 * @param string $charset Character set
 * @return string Random string
 */
function randomString(int $length = 16, string $charset = 'alphanumeric'): string
{
    $charsets = [
        'alphanumeric' => 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789',
        'alpha' => 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ',
        'numeric' => '0123456789',
        'hex' => '0123456789abcdef'
    ];
    
    $chars = $charsets[$charset] ?? $charsets['alphanumeric'];
    $result = '';
    $max = strlen($chars) - 1;
    
    for ($i = 0; $i < $length; $i++) {
        $result .= $chars[random_int(0, $max)];
    }
    
    return $result;
}
```

## Date/Time Utilities

### now()

Returns current timestamp.

```php
/**
 * Get current timestamp
 * 
 * @param bool $asDateTime Return as DateTime object
 * @return string|DateTime
 */
function now(bool $asDateTime = false)
{
    if ($asDateTime) {
        return new DateTime();
    }
    return date('Y-m-d H:i:s');
}
```

### formatDate()

Formats a date for display.

```php
/**
 * Format date for display
 * 
 * @param string $date Date string
 * @param string $format Output format
 * @param string $timezone Timezone
 * @return string Formatted date
 */
function formatDate(string $date, string $format = 'Y-m-d', string $timezone = ''): string
{
    $dt = new DateTime($date);
    
    if ($timezone) {
        $dt->setTimezone(new DateTimeZone($timezone));
    }
    
    return $dt->format($format);
}
```

### relativeTime()

Returns relative time string.

```php
/**
 * Get relative time string
 * 
 * @param string $date Date to compare
 * @param string|null $referenceDate Reference date (null = now)
 * @return string Relative time
 */
function relativeTime(string $date, ?string $referenceDate = null): string
{
    $reference = $referenceDate ? new DateTime($referenceDate) : new DateTime();
    $target = new DateTime($date);
    
    $diff = $reference->diff($target);
    
    if ($diff->y > 0) return $diff->y . ' year' . ($diff->y > 1 ? 's' : '') . ' ago';
    if ($diff->m > 0) return $diff->m . ' month' . ($diff->m > 1 ? 's' : '') . ' ago';
    if ($diff->d > 0) return $diff->d . ' day' . ($diff->d > 1 ? 's' : '') . ' ago';
    if ($diff->h > 0) return $diff->h . ' hour' . ($diff->h > 1 ? 's' : '') . ' ago';
    if ($diff->i > 0) return $diff->i . ' minute' . ($diff->i > 1 ? 's' : '') . ' ago';
    return 'just now';
}
```

**Example:**
```php
echo relativeTime('2024-01-15'); // "2 months ago"
```

## Validation Utilities

### isValidEmail()

Validates email address.

```php
/**
 * Validate email address
 * 
 * @param string $email Email to validate
 * @return bool Valid status
 */
function isValidEmail(string $email): bool
{
    return filter_var($email, FILTER_VALIDATE_EMAIL) !== false;
}
```

### isValidDomain()

Validates domain name.

```php
/**
 * Validate domain name
 * 
 * @param string $domain Domain to validate
 * @return bool Valid status
 */
function isValidDomain(string $domain): bool
{
    return preg_match('/^[a-zA-Z0-9][a-zA-Z0-9-]{0,61}[a-zA-Z0-9]?(?:\.[a-zA-Z]{2,})+$/', $domain) === 1;
}
```

### isValidUrl()

Validates URL.

```php
/**
 * Validate URL
 * 
 * @param string $url URL to validate
 * @return bool Valid status
 */
function isValidUrl(string $url): bool
{
    return filter_var($url, FILTER_VALIDATE_URL) !== false;
}
```

## Encryption Utilities

### encryptString()

Encrypts a string.

```php
/**
 * Encrypt string
 * 
 * @param string $string String to encrypt
 * @param string $key Encryption key
 * @return string Encrypted string
 */
function encryptString(string $string, string $key = ''): string
{
    $key = $key ?: Config\Setting::getValue('EncryptionKey');
    $iv = random_bytes(16);
    
    $encrypted = openssl_encrypt($string, 'AES-256-CBC', $key, 0, $iv);
    
    return base64_encode($iv . $encrypted);
}
```

### decryptString()

Decrypts a string.

```php
/**
 * Decrypt string
 * 
 * @param string $encrypted Encrypted string
 * @param string $key Encryption key
 * @return string Decrypted string
 */
function decryptString(string $encrypted, string $key = ''): string
{
    $key = $key ?: Config\Setting::getValue('EncryptionKey');
    $data = base64_decode($encrypted);
    
    $iv = substr($data, 0, 16);
    $encrypted = substr($data, 16);
    
    return openssl_decrypt($encrypted, 'AES-256-CBC', $key, 0, $iv);
}
```

## File Utilities

### ensureDirectoryExists()

Ensures a directory exists, creates if not.

```php
/**
 * Ensure directory exists
 * 
 * @param string $path Directory path
 * @param int $mode Directory permissions
 * @return bool Success status
 */
function ensureDirectoryExists(string $path, int $mode = 0755): bool
{
    if (!is_dir($path)) {
        return mkdir($path, $mode, true);
    }
    return true;
}
```

### getFileExtension()

Gets file extension.

```php
/**
 * Get file extension
 * 
 * @param string $filename Filename
 * @return string Extension
 */
function getFileExtension(string $filename): string
{
    return strtolower(pathinfo($filename, PATHINFO_EXTENSION));
}
```

### isAllowedFileExtension()

Checks if file extension is allowed.

```php
/**
 * Check if file extension is allowed
 * 
 * @param string $filename Filename
 * @param array $allowedExtensions Allowed extensions
 * @return bool Allowed status
 */
function isAllowedFileExtension(string $filename, array $allowedExtensions = []): bool
{
    if (empty($allowedExtensions)) {
        $allowedExtensions = ['jpg', 'jpeg', 'png', 'gif', 'pdf', 'doc', 'docx'];
    }
    
    return in_array(getFileExtension($filename), $allowedExtensions);
}
```

## IP Utilities

### getClientIp()

Gets client IP address.

```php
/**
 * Get client IP address
 * 
 * @return string Client IP
 */
function getClientIp(): string
{
    $headers = [
        'HTTP_CF_CONNECTING_IP', // Cloudflare
        'HTTP_X_FORWARDED_FOR',
        'HTTP_X_REAL_IP',
        'REMOTE_ADDR'
    ];
    
    foreach ($headers as $header) {
        if (!empty($_SERVER[$header])) {
            $ip = $_SERVER[$header];
            
            // Handle comma-separated IPs
            if (strpos($ip, ',') !== false) {
                $ip = trim(explode(',', $ip)[0]);
            }
            
            return filter_var($ip, FILTER_VALIDATE_IP) ? $ip : '';
        }
    }
    
    return '';
}
```

### isIpBlacklisted()

Checks if IP is blacklisted.

```php
/**
 * Check if IP is blacklisted
 * 
 * @param string $ip IP address
 * @return bool Blacklisted status
 */
function isIpBlacklisted(string $ip): bool
{
    $blacklist = Capsule::table('tblipfilter')
        ->where('ip', $ip)
        ->orWhereRaw("? LIKE REPLACE(ip, '*', '%')", [$ip])
        ->first();
    
    return $blacklist !== null;
}
```

## Debug Utilities

### dump()

Dumps variable for debugging.

```php
/**
 * Dump variable for debugging
 * 
 * @param mixed $var Variable to dump
 * @param bool $exit Exit after dump
 * @return void
 */
function dump($var, bool $exit = false): void
{
    echo '<pre>';
    print_r($var);
    echo '</pre>';
    
    if ($exit) {
        exit;
    }
}
```

### logDebug()

Logs debug message.

```php
/**
 * Log debug message
 * 
 * @param string $message Debug message
 * @param array $context Context data
 * @return void
 */
function logDebug(string $message, array $context = []): void
{
    if (!Config\Setting::getValue('debug')) {
        return;
    }
    
    $logEntry = [
        'timestamp' => date('Y-m-d H:i:s'),
        'message' => $message,
        'context' => $context
    ];
    
    error_log(print_r($logEntry, true), 3, '/path/to/debug.log');
}
```

## Related Functions

- [whmcs-functions-sanitization.md](whmcs-functions-sanitization.md) - Input sanitization
- [whmcs-functions-validation.md](whmcs-functions-validation.md) - Data validation
- [whmcs-functions-date-time.md](whmcs-functions-date-time.md) - Date/time functions