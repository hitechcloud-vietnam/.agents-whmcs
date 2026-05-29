# WHMCS Sanitization Functions

Complete reference for data sanitization and input cleaning in WHMCS.

## Overview

WHMCS provides comprehensive sanitization functions to ensure data security and prevent injection attacks.

## Core Sanitization Functions

### sanitize()

Generic sanitization function.

```php
/**
 * Sanitize input data
 * 
 * @param mixed $data Data to sanitize
 * @param string $type Sanitization type
 * @return mixed Sanitized data
 */
function sanitize(mixed $data, string $type = 'general')
{
    switch ($type) {
        case 'general':
            return sanitizeGeneral($data);
        case 'html':
            return sanitizeHtml($data);
        case 'sql':
            return sanitizeSql($data);
        case 'email':
            return sanitizeEmail($data);
        case 'url':
            return sanitizeUrl($data);
        case 'int':
            return (int) $data;
        case 'float':
            return (float) $data;
        case 'alphanumeric':
            return preg_replace('/[^a-zA-Z0-9]/', '', $data);
        case 'filename':
            return sanitizeFilename($data);
        default:
            return $data;
    }
}
```

### sanitizeGeneral()

General sanitization.

```php
/**
 * General sanitization
 * 
 * @param string $input Input string
 * @return string Sanitized string
 */
function sanitizeGeneral(string $input): string
{
    // Remove null bytes
    $input = str_replace("\0", '', $input);
    
    // Trim whitespace
    $input = trim($input);
    
    // Remove control characters
    $input = preg_replace('/[\x00-\x1F\x7F]/', '', $input);
    
    return $input;
}
```

### sanitizeHtml()

Sanitizes HTML content.

```php
/**
 * Sanitize HTML content
 * 
 * @param string $html HTML content
 * @param array $allowedTags Allowed HTML tags
 * @return string Sanitized HTML
 */
function sanitizeHtml(string $html, array $allowedTags = []): string
{
    // If no allowed tags, strip all HTML
    if (empty($allowedTags)) {
        return strip_tags($html);
    }
    
    // Use allowed tags
    $allowed = '<' . implode('><', $allowedTags) . '>';
    
    // Strip disallowed tags
    return strip_tags($html, $allowed);
}
```

**Example:**
```php
$content = sanitizeHtml('<p>Hello <b>world</b>!</p><script>alert("xss")</script>');
// Returns: <p>Hello <b>world</b>!</p>

$content = sanitizeHtml('<p>Hello</p><b>bold</b>', ['p', 'b']);
// Returns: <p>Hello</p><b>bold</b>
```

### sanitizeSql()

Sanitizes for SQL.

```php
/**
 * Sanitize for SQL injection prevention
 * Note: Use prepared statements instead
 * 
 * @param string $input Input string
 * @return string Sanitized string
 */
function sanitizeSql(string $input): string
{
    // Escape special characters
    $search = ["\\", "\0", "\n", "\r", "\x1a", "'", '"'];
    $replace = ["\\\\", "\\0", "\\n", "\\r", "\\Z", "\\'", "\\\""];
    
    return str_replace($search, $replace, $input);
}
```

## Input Sanitization

### sanitizeInput()

Sanitizes user input.

```php
/**
 * Sanitize user input
 * 
 * @param mixed $input Input data
 * @param string $type Expected data type
 * @return mixed Sanitized input
 */
function sanitizeInput(mixed $input, string $type = 'string')
{
    if (is_array($input)) {
        return array_map(function($item) use ($type) {
            return sanitizeInput($item, $type);
        }, $input);
    }
    
    switch ($type) {
        case 'string':
            return filter_var($input, FILTER_SANITIZE_STRING);
        
        case 'email':
            return filter_var($input, FILTER_SANITIZE_EMAIL);
        
        case 'url':
            return filter_var($input, FILTER_SANITIZE_URL);
        
        case 'int':
            return (int) filter_var($input, FILTER_SANITIZE_NUMBER_INT);
        
        case 'float':
            return (float) filter_var($input, FILTER_SANITIZE_NUMBER_FLOAT, FILTER_FLAG_ALLOW_FRACTION);
        
        case 'encoded':
            return filter_var($input, FILTER_SANITIZE_ENCODED);
        
        case 'special':
            return filter_var($input, FILTER_SANITIZE_SPECIAL_CHARS);
        
        default:
            return sanitizeGeneral($input);
    }
}
```

**Example:**
```php
$name = sanitizeInput($_POST['name'], 'string');
$email = sanitizeInput($_POST['email'], 'email');
$age = sanitizeInput($_POST['age'], 'int');
```

### sanitizeArray()

Sanitizes array input.

```php
/**
 * Sanitize array of inputs
 * 
 * @param array $data Input data
 * @param array $types Field types
 * @return array Sanitized data
 */
function sanitizeArray(array $data, array $types): array
{
    $sanitized = [];
    
    foreach ($types as $field => $type) {
        if (isset($data[$field])) {
            $sanitized[$field] = sanitizeInput($data[$field], $type);
        }
    }
    
    return $sanitized;
}
```

**Example:**
```php
$sanitized = sanitizeArray($_POST, [
    'firstname' => 'string',
    'email' => 'email',
    'age' => 'int',
    'website' => 'url'
]);
```

## HTML Sanitization

### sanitizeHtmlInput()

Comprehensive HTML sanitization.

```php
/**
 * Sanitize HTML input
 * 
 * @param string $html HTML content
 * @param array $options Sanitization options
 * @return string Sanitized HTML
 */
function sanitizeHtmlInput(string $html, array $options = []): string
{
    // Remove script tags
    $html = preg_replace('/<script\b[^>]*>(.*?)<\/script>/is', '', $html);
    
    // Remove on* event handlers
    $html = preg_replace('/\s*on\w+\s*=\s*["\'][^"\']*["\']/i', '', $html);
    
    // Remove javascript: URLs
    $html = preg_replace('/javascript:/i', '', $html);
    
    // Remove data: URLs (can be used for XSS)
    $html = preg_replace('/data:/i', '', $html);
    
    // If strict mode, remove all tags
    if ($options['strict'] ?? false) {
        return strip_tags($html);
    }
    
    // Remove dangerous tags
    $dangerous = ['iframe', 'object', 'embed', 'form', 'input', 'button'];
    
    foreach ($dangerous as $tag) {
        $html = preg_replace('/<\/?' . $tag . '[^>]*>/i', '', $html);
    }
    
    // Keep only allowed attributes
    if (!empty($options['allowed_attrs'])) {
        $html = sanitizeHtmlAttributes($html, $options['allowed_attrs']);
    }
    
    return $html;
}

/**
 * Sanitize HTML attributes
 * 
 * @param string $html HTML content
 * @param array $allowedAttrs Allowed attributes
 * @return string Sanitized HTML
 */
function sanitizeHtmlAttributes(string $html, array $allowedAttrs): string
{
    return preg_replace_callback(
        '/<([a-z]+)([^>]*)>/i',
        function($matches) use ($allowedAttrs) {
            $tag = $matches[1];
            $attrs = $matches[2];
            
            // Parse attributes
            preg_match_all('/([a-z-]+)\s*=\s*["\']([^"\']*)["\']/i', $attrs, $attrMatches);
            
            $cleanAttrs = [];
            if (isset($attrMatches[1])) {
                foreach ($attrMatches[1] as $i => $attr) {
                    if (in_array($attr, $allowedAttrs)) {
                        $cleanAttrs[] = $attr . '="' . htmlspecialchars($attrMatches[2][$i], ENT_QUOTES) . '"';
                    }
                }
            }
            
            $attrStr = !empty($cleanAttrs) ? ' ' . implode(' ', $cleanAttrs) : '';
            
            return '<' . $tag . $attrStr . '>';
        },
        $html
    );
}
```

## Email Sanitization

### sanitizeEmail()

Sanitizes email address.

```php
/**
 * Sanitize email address
 * 
 * @param string $email Email address
 * @return string Sanitized email
 */
function sanitizeEmail(string $email): string
{
    // Trim and lowercase
    $email = strtolower(trim($email));
    
    // Remove dangerous characters
    $email = preg_replace('/[\x00-\x1F\x7F<>]/', '', $email);
    
    // Validate format
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        return '';
    }
    
    return $email;
}
```

**Example:**
```php
$email = sanitizeEmail('  John.Doe+tag@example.com  ');
// Returns: john.doe+tag@example.com
```

## URL Sanitization

### sanitizeUrl()

Sanitizes URL.

```php
/**
 * Sanitize URL
 * 
 * @param string $url URL to sanitize
 * @param array $allowedProtocols Allowed protocols
 * @return string Sanitized URL
 */
function sanitizeUrl(string $url, array $allowedProtocols = ['http', 'https']): string
{
    // Trim
    $url = trim($url);
    
    // Decode entities
    $url = html_entity_decode($url);
    
    // Check protocol
    $protocol = parse_url($url, PHP_URL_SCHEME);
    $protocol = strtolower($protocol ?? '');
    
    if (!in_array($protocol, $allowedProtocols)) {
        return '';
    }
    
    // Use filter
    $sanitized = filter_var($url, FILTER_SANITIZE_URL);
    
    // Remove dangerous patterns
    $sanitized = preg_replace('/javascript:/i', '', $sanitized);
    $sanitized = preg_replace('/data:/i', '', $sanitized);
    
    return $sanitized;
}
```

## Filename Sanitization

### sanitizeFilename()

Sanitizes filename.

```php
/**
 * Sanitize filename
 * 
 * @param string $filename Filename
 * @param bool $allowExtensions Allow file extensions
 * @return string Sanitized filename
 */
function sanitizeFilename(string $filename, bool $allowExtensions = true): string
{
    // Remove path info
    $filename = basename($filename);
    
    // Replace spaces with underscores
    $filename = preg_replace('/\s+/', '_', $filename);
    
    // Remove special characters (keep alphanumeric, dot, dash, underscore)
    $filename = preg_replace('/[^a-zA-Z0-9._-]/', '', $filename);
    
    // Limit length
    if (strlen($filename) > 255) {
        $ext = pathinfo($filename, PATHINFO_EXTENSION);
        $name = pathinfo($filename, PATHINFO_FILENAME);
        $filename = substr($name, 0, 255 - strlen($ext) - 1) . '.' . $ext;
    }
    
    // Remove double extensions if not allowed
    if (!$allowExtensions) {
        $filename = preg_replace('/\.[^.]+$/', '', $filename);
    }
    
    return $filename;
}
```

**Example:**
```php
$filename = sanitizeFilename('../../../etc/passwd');
// Returns: etcpasswd (on Linux) or the basename only

$filename = sanitizeFilename('my file.pdf');
// Returns: my_file.pdf
```

## Database Sanitization

### escapeString()

Escapes string for database.

```php
/**
 * Escape string for database
 * 
 * @param string $string String to escape
 * @return string Escaped string
 */
function escapeString(string $string): string
{
    return addslashes($string);
}

/**
 * Escape array for database
 * 
 * @param array $array Array to escape
 * @return array Escaped array
 */
function escapeArray(array $array): array
{
    return array_map(function($value) {
        if (is_array($value)) {
            return escapeArray($value);
        }
        if (is_string($value)) {
            return escapeString($value);
        }
        return $value;
    }, $array);
}
```

## XSS Prevention

### preventXSS()

Prevents XSS attacks.

```php
/**
 * Prevent XSS attacks
 * 
 * @param string $input Input string
 * @param bool $allowHtml Allow HTML
 * @return string Safe string
 */
function preventXSS(string $input, bool $allowHtml = false): string
{
    if ($allowHtml) {
        return sanitizeHtmlInput($input, ['strict' => false]);
    }
    
    return htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}

/**
 * Clean for JavaScript
 * 
 * @param string $input Input string
 * @return string Cleaned string
 */
function cleanForJavaScript(string $input): string
{
    return json_encode($input, JSON_HEX_TAG | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_HEX_AMP);
}
```

## Path Sanitization

### sanitizePath()

Sanitizes file path.

```php
/**
 * Sanitize file path
 * 
 * @param string $path Path to sanitize
 * @param string $basePath Base path for validation
 * @return string Sanitized path
 */
function sanitizePath(string $path, string $basePath = ''): string
{
    // Normalize path
    $path = str_replace(['../', '..\\'], '', $path);
    $path = preg_replace('/[\/\\]+/', '/', $path);
    
    // Remove null bytes
    $path = str_replace("\0", '', $path);
    
    // Resolve to absolute path if base provided
    if ($basePath && !is_absolute($path)) {
        $path = realpath($basePath . '/' . $path);
        
        // Ensure path is within base
        if (strpos($path, realpath($basePath)) !== 0) {
            return '';
        }
    }
    
    return $path;
}
```

## Phone Number Sanitization

### sanitizePhone()

Sanitizes phone number.

```php
/**
 * Sanitize phone number
 * 
 * @param string $phone Phone number
 * @param string $country Country code
 * @return string Sanitized phone
 */
function sanitizePhone(string $phone, string $country = 'US'): string
{
    // Remove all non-digits
    $phone = preg_replace('/[^0-9+]/', '', $phone);
    
    // Handle country-specific formatting
    switch ($country) {
        case 'US':
        case 'CA':
            // Format: +1-XXX-XXX-XXXX
            if (strlen($phone) === 11 && $phone[0] === '1') {
                $phone = '+1-' . substr($phone, 1, 3) . '-' . substr($phone, 4, 3) . '-' . substr($phone, 7);
            }
            break;
    }
    
    return $phone;
}
```

## Credit Card Sanitization

### sanitizeCreditCard()

Sanitizes credit card number (for display).

```php
/**
 * Sanitize credit card number
 * 
 * @param string $cardNumber Card number
 * @param string $mask Mask character
 * @return string Masked card number
 */
function sanitizeCreditCard(string $cardNumber, string $mask = 'X'): string
{
    $length = strlen($cardNumber);
    
    // Show only last 4 digits
    if ($length > 4) {
        return str_repeat($mask, $length - 4) . substr($cardNumber, -4);
    }
    
    return str_repeat($mask, $length);
}
```

## Best Practices

1. **Always sanitize user input** - Never trust user data
2. **Use allowlists** - Prefer allowed values over blocklists
3. **Escape output** - Escape when displaying, not when storing
4. **Validate types** - Check data types before sanitizing
5. **Use prepared statements** - For database queries
6. **Keep sanitization focused** - Different data needs different treatment

## Related Functions

- [whmcs-functions-validation.md](whmcs-functions-validation.md) - Data validation
- [whmcs-functions-utility.md](whmcs-functions-utility.md) - Utility functions