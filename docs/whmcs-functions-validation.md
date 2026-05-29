# WHMCS Validation Functions

Complete reference for data validation functions in WHMCS.

## Overview

WHMCS provides comprehensive validation functions for ensuring data integrity across the system.

## Core Validation Functions

### validate()

Generic validation function.

```php
/**
 * Validate data against rules
 * 
 * @param array $data Data to validate
 * @param array $rules Validation rules
 * @return array Validation result
 */
function validate(array $data, array $rules): array
{
    $errors = [];
    
    foreach ($rules as $field => $ruleSet) {
        $value = $data[$field] ?? null;
        $fieldRules = explode('|', $ruleSet);
        
        foreach ($fieldRules as $rule) {
            $result = validateField($field, $value, $rule, $data);
            
            if ($result !== true) {
                $errors[$field][] = $result;
            }
        }
    }
    
    return [
        'valid' => empty($errors),
        'errors' => $errors
    ];
}

/**
 * Validate a single field
 * 
 * @param string $field Field name
 * @param mixed $value Field value
 * @param string $rule Validation rule
 * @param array $data All data
 * @return bool|string True or error message
 */
function validateField(string $field, $value, string $rule, array $data)
{
    // Parse rule with parameters
    $parts = explode(':', $rule);
    $ruleName = $parts[0];
    $params = isset($parts[1]) ? explode(',', $parts[1]) : [];
    
    switch ($ruleName) {
        case 'required':
            if (empty($value) && $value !== '0') {
                return "The {$field} field is required";
            }
            break;
            
        case 'email':
            if (!empty($value) && !filter_var($value, FILTER_VALIDATE_EMAIL)) {
                return "The {$field} must be a valid email address";
            }
            break;
            
        case 'min':
            if (is_string($value) && strlen($value) < $params[0]) {
                return "The {$field} must be at least {$params[0]} characters";
            }
            if (is_numeric($value) && $value < $params[0]) {
                return "The {$field} must be at least {$params[0]}";
            }
            break;
            
        case 'max':
            if (is_string($value) && strlen($value) > $params[0]) {
                return "The {$field} must not exceed {$params[0]} characters";
            }
            if (is_numeric($value) && $value > $params[0]) {
                return "The {$field} must not exceed {$params[0]}";
            }
            break;
            
        case 'numeric':
            if (!empty($value) && !is_numeric($value)) {
                return "The {$field} must be a number";
            }
            break;
            
        case 'integer':
            if (!empty($value) && !ctype_digit((string)$value)) {
                return "The {$field} must be an integer";
            }
            break;
            
        case 'alpha':
            if (!empty($value) && !ctype_alpha($value)) {
                return "The {$field} must contain only letters";
            }
            break;
            
        case 'alphanumeric':
            if (!empty($value) && !ctype_alnum($value)) {
                return "The {$field} must contain only letters and numbers";
            }
            break;
            
        case 'in':
            if (!empty($value) && !in_array($value, $params)) {
                return "The {$field} must be one of: " . implode(', ', $params);
            }
            break;
            
        case 'regex':
            if (!empty($value) && !preg_match($params[0], $value)) {
                return "The {$field} format is invalid";
            }
            break;
    }
    
    return true;
}
```

**Example:**
```php
$result = validate($_POST, [
    'email' => 'required|email',
    'firstname' => 'required|min:2|max:50',
    'password' => 'required|min:8',
    'age' => 'required|numeric|min:18'
]);

if (!$result['valid']) {
    foreach ($result['errors'] as $field => $errors) {
        echo "{$field}: " . implode(', ', $errors) . "\n";
    }
}
```

## Client Validation

### validateClientData()

Validates client data.

```php
/**
 * Validate client data
 * 
 * @param array $data Client data
 * @param bool $isUpdate Is update operation
 * @return array Validation result
 */
function validateClientData(array $data, bool $isUpdate = false): array
{
    $rules = [];
    
    if (!$isUpdate || isset($data['email'])) {
        $rules['email'] = 'required|email|max:255';
    }
    
    if (!$isUpdate || isset($data['password'])) {
        $rules['password'] = 'required|min:8';
    }
    
    $rules['firstname'] = 'required|min:2|max:50';
    $rules['lastname'] = 'required|min:2|max:50';
    $rules['phonenumber'] = 'max:30';
    $rules['country'] = 'required|in:' . implode(',', getValidCountries());
    
    return validate($data, $rules);
}
```

**Example:**
```php
$result = validateClientData($_POST);

if (!$result['valid']) {
    // Handle validation errors
}
```

### validateClientEmail()

Validates client email.

```php
/**
 * Validate client email
 * 
 * @param string $email Email to validate
 * @param int|null $excludeClientId Client ID to exclude
 * @return array Validation result
 */
function validateClientEmail(string $email, ?int $excludeClientId = null): array
{
    // Check email format
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        return ['valid' => false, 'error' => 'Invalid email format'];
    }
    
    // Check for duplicate
    $query = Capsule::table('tblclients')
        ->where('email', $email);
    
    if ($excludeClientId) {
        $query->where('id', '!=', $excludeClientId);
    }
    
    if ($query->first()) {
        return ['valid' => false, 'error' => 'Email already in use'];
    }
    
    return ['valid' => true];
}
```

## Order Validation

### validateOrderData()

Validates order data.

```php
/**
 * Validate order data
 * 
 * @param array $data Order data
 * @return array Validation result
 */
function validateOrderData(array $data): array
{
    $errors = [];
    
    // Validate client
    if (empty($data['clientid']) || !getClient($data['clientid'])) {
        $errors['clientid'][] = 'Invalid client';
    }
    
    // Validate payment method
    if (empty($data['paymentmethod'])) {
        $errors['paymentmethod'][] = 'Payment method is required';
    }
    
    // Validate items
    if (empty($data['items']) || !is_array($data['items'])) {
        $errors['items'][] = 'At least one item is required';
    } else {
        foreach ($data['items'] as $index => $item) {
            if (empty($item['productid']) && empty($item['domain'])) {
                $errors["items.{$index}"][] = 'Product or domain is required';
            }
        }
    }
    
    return [
        'valid' => empty($errors),
        'errors' => $errors
    ];
}
```

## Invoice Validation

### validateInvoiceData()

Validates invoice data.

```php
/**
 * Validate invoice data
 * 
 * @param array $data Invoice data
 * @return array Validation result
 */
function validateInvoiceData(array $data): array
{
    $errors = [];
    
    if (!empty($data['clientid']) && !getClient($data['clientid'])) {
        $errors['clientid'][] = 'Invalid client';
    }
    
    if (isset($data['total']) && $data['total'] < 0) {
        $errors['total'][] = 'Invoice total cannot be negative';
    }
    
    if (!empty($data['duedate']) && !strtotime($data['duedate'])) {
        $errors['duedate'][] = 'Invalid due date format';
    }
    
    return [
        'valid' => empty($errors),
        'errors' => $errors
    ];
}
```

## Domain Validation

### validateDomain()

Validates domain name.

```php
/**
 * Validate domain name
 * 
 * @param string $domain Domain to validate
 * @return array Validation result
 */
function validateDomain(string $domain): array
{
    // Remove www. if present
    $domain = preg_replace('/^www\./i', '', $domain);
    
    // Check length
    if (strlen($domain) < 3) {
        return ['valid' => false, 'error' => 'Domain too short'];
    }
    
    if (strlen($domain) > 63) {
        return ['valid' => false, 'error' => 'Domain too long'];
    }
    
    // Check format
    $pattern = '/^(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}$/';
    
    if (!preg_match($pattern, $domain)) {
        return ['valid' => false, 'error' => 'Invalid domain format'];
    }
    
    // Check for valid TLD
    $parts = explode('.', $domain);
    $tld = strtolower(end($parts));
    
    $validTlds = ['com', 'net', 'org', 'info', 'biz', 'co', 'us', 'uk', 'ca', 'au'];
    
    if (!in_array($tld, $validTlds)) {
        return ['valid' => false, 'error' => 'Unsupported TLD'];
    }
    
    return ['valid' => true];
}
```

### validateDomainTransfer()

Validates domain transfer.

```php
/**
 * Validate domain transfer
 * 
 * @param string $domain Domain name
 * @param string $authCode Transfer auth code
 * @return array Validation result
 */
function validateDomainTransfer(string $domain, string $authCode): array
{
    $domainValidation = validateDomain($domain);
    
    if (!$domainValidation['valid']) {
        return $domainValidation;
    }
    
    // Check auth code
    if (empty($authCode)) {
        return ['valid' => false, 'error' => 'Transfer authorization code is required'];
    }
    
    if (strlen($authCode) < 6) {
        return ['valid' => false, 'error' => 'Invalid authorization code'];
    }
    
    // Check domain is not already transferred
    $existing = Capsule::table('tbldomains')
        ->where('domain', $domain)
        ->first();
    
    if ($existing && $existing->domainstatus === 'Transferred') {
        return ['valid' => false, 'error' => 'Domain already transferred'];
    }
    
    return ['valid' => true];
}
```

## Payment Validation

### validatePaymentData()

Validates payment data.

```php
/**
 * Validate payment data
 * 
 * @param array $data Payment data
 * @return array Validation result
 */
function validatePaymentData(array $data): array
{
    $errors = [];
    
    // Validate amount
    if (!isset($data['amount']) || !is_numeric($data['amount'])) {
        $errors['amount'][] = 'Valid amount is required';
    } elseif ($data['amount'] <= 0) {
        $errors['amount'][] = 'Amount must be greater than zero';
    }
    
    // Validate gateway
    if (empty($data['gateway'])) {
        $errors['gateway'][] = 'Payment gateway is required';
    } else {
        $gateway = Capsule::table('tblpaymentgateways')
            ->where('gateway', $data['gateway'])
            ->where('setting', 'type')
            ->where('value', 'CC')
            ->first();
        
        if (!$gateway) {
            $errors['gateway'][] = 'Invalid payment gateway';
        }
    }
    
    return [
        'valid' => empty($errors),
        'errors' => $errors
    ];
}
```

## Password Validation

### validatePassword()

Validates password strength.

```php
/**
 * Validate password strength
 * 
 * @param string $password Password to validate
 * @param array $options Validation options
 * @return array Validation result
 */
function validatePassword(string $password, array $options = []): array
{
    $minLength = $options['min_length'] ?? 8;
    $requireUppercase = $options['require_uppercase'] ?? true;
    $requireLowercase = $options['require_lowercase'] ?? true;
    $requireNumber = $options['require_number'] ?? true;
    $requireSpecial = $options['require_special'] ?? false;
    
    $errors = [];
    
    if (strlen($password) < $minLength) {
        $errors[] = "Password must be at least {$minLength} characters";
    }
    
    if ($requireUppercase && !preg_match('/[A-Z]/', $password)) {
        $errors[] = 'Password must contain at least one uppercase letter';
    }
    
    if ($requireLowercase && !preg_match('/[a-z]/', $password)) {
        $errors[] = 'Password must contain at least one lowercase letter';
    }
    
    if ($requireNumber && !preg_match('/[0-9]/', $password)) {
        $errors[] = 'Password must contain at least one number';
    }
    
    if ($requireSpecial && !preg_match('/[!@#$%^&*(),.?":{}|<>]/', $password)) {
        $errors[] = 'Password must contain at least one special character';
    }
    
    return [
        'valid' => empty($errors),
        'errors' => $errors
    ];
}
```

**Example:**
```php
$result = validatePassword('MyP@ssw0rd!', [
    'min_length' => 10,
    'require_special' => true
]);

if (!$result['valid']) {
    echo implode("\n", $result['errors']);
}
```

## Custom Validation Rules

### addValidationRule()

Adds a custom validation rule.

```php
/**
 * Add custom validation rule
 * 
 * @param string $name Rule name
 * @param callable $callback Validation callback
 * @return void
 */
function addValidationRule(string $name, callable $callback): void
{
    global $customValidationRules;
    
    $customValidationRules[$name] = $callback;
}

/**
 * Run custom validation rule
 * 
 * @param string $field Field name
 * @param mixed $value Field value
 * @param array $params Rule parameters
 * @return bool|string
 */
function runCustomRule(string $field, $value, array $params)
{
    global $customValidationRules;
    
    $ruleName = $params[0] ?? '';
    
    if (isset($customValidationRules[$ruleName])) {
        return $customValidationRules[$ruleName]($field, $value, $params);
    }
    
    return true;
}
```

**Example:**
```php
addValidationRule('unique_email', function($field, $value, $params) {
    $exists = Capsule::table('tblclients')
        ->where('email', $value)
        ->first();
    
    if ($exists) {
        return "The email address is already in use";
    }
    
    return true;
});

// Use in validation
$result = validate($_POST, [
    'email' => 'required|email|unique_email'
]);
```

## Validation Helpers

### isValidAmount()

Validates monetary amount.

```php
/**
 * Validate amount
 * 
 * @param mixed $amount Amount to validate
 * @param int $decimals Decimal places
 * @return bool Valid status
 */
function isValidAmount($amount, int $decimals = 2): bool
{
    if (!is_numeric($amount)) {
        return false;
    }
    
    $pattern = '/^\d+(\.\d{1,' . $decimals . '})?$/';
    return preg_match($pattern, (string)$amount) === 1;
}
```

### isValidDate()

Validates date format.

```php
/**
 * Validate date
 * 
 * @param string $date Date string
 * @param string $format Date format
 * @return bool Valid status
 */
function isValidDate(string $date, string $format = 'Y-m-d'): bool
{
    $dt = DateTime::createFromFormat($format, $date);
    return $dt && $dt->format($format) === $date;
}
```

### isValidUrl()

Validates URL.

```php
/**
 * Validate URL
 * 
 * @param string $url URL to validate
 * @param array $protocols Allowed protocols
 * @return bool Valid status
 */
function isValidUrl(string $url, array $protocols = ['http', 'https']): bool
{
    $pattern = '/^(' . implode('|', $protocols) . '):\/\/.+/i';
    
    if (!preg_match($pattern, $url)) {
        return false;
    }
    
    return filter_var($url, FILTER_VALIDATE_URL) !== false;
}
```

## Best Practices

1. **Validate on both sides** - Client and server-side validation
2. **Use specific messages** - Clear error messages for users
3. **Sanitize before validating** - Clean data before validation
4. **Fail securely** - Default to rejecting invalid data
5. **Log validation failures** - Track attempted attacks
6. **Use constants** - Define validation rules as reusable constants

## Related Functions

- [whmcs-functions-sanitization.md](whmcs-functions-sanitization.md) - Data sanitization
- [whmcs-functions-utility.md](whmcs-functions-utility.md) - Utility functions