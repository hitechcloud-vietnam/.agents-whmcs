# WHMCS Data Validation

## Overview

Data validation ensures data integrity and security in WHMCS modules and customizations.

## Validation Class

```php
<?php
use WHMCS\Validation\Validator;

class CustomValidator extends Validator
{
    public static function rules(): array
    {
        return [
            'email' => 'required|email|max:255',
            'firstname' => 'required|string|max:100',
            'lastname' => 'required|string|max:100',
            'phone' => 'sometimes|phone',
            'domain' => 'sometimes|domain',
            'amount' => 'required|numeric|min:0',
            'date' => 'sometimes|date',
        ];
    }
}
```

## Basic Validation

### Client Data Validation

```php
<?php
function validateClientData(array $data): array
{
    $errors = [];
    
    // Email validation
    if (empty($data['email'])) {
        $errors['email'] = 'Email is required';
    } elseif (!filter_var($data['email'], FILTER_VALIDATE_EMAIL)) {
        $errors['email'] = 'Invalid email format';
    } elseif (emailExists($data['email'], $data['exclude_id'] ?? null)) {
        $errors['email'] = 'Email already registered';
    }
    
    // Name validation
    if (empty($data['firstname'])) {
        $errors['firstname'] = 'First name is required';
    } elseif (strlen($data['firstname']) < 2) {
        $errors['firstname'] = 'First name too short';
    }
    
    if (empty($data['lastname'])) {
        $errors['lastname'] = 'Last name is required';
    }
    
    // Phone validation
    if (!empty($data['phonenumber'])) {
        $phone = preg_replace('/[^0-9+]/', '', $data['phonenumber']);
        if (strlen($phone) < 10 || strlen($phone) > 15) {
            $errors['phonenumber'] = 'Invalid phone number';
        }
    }
    
    // Country validation
    $validCountries = getValidCountryCodes();
    if (!in_array($data['country'] ?? '', $validCountries)) {
        $errors['country'] = 'Invalid country';
    }
    
    return $errors;
}
```

### Service Validation

```php
<?php
function validateServiceData(array $data): array
{
    $errors = [];
    
    // Domain validation
    if (!empty($data['domain'])) {
        if (!filter_var($data['domain'], FILTER_VALIDATE_DOMAIN, FILTER_FLAG_HOSTNAME)) {
            $errors['domain'] = 'Invalid domain format';
        }
        
        // Check domain availability if needed
        if (!isDomainAvailable($data['domain'])) {
            $errors['domain'] = 'Domain already registered';
        }
    }
    
    // Username validation
    if (!empty($data['username'])) {
        if (!preg_match('/^[a-zA-Z0-9_-]+$/', $data['username'])) {
            $errors['username'] = 'Invalid username format';
        }
        
        if (strlen($data['username']) < 3 || strlen($data['username']) > 32) {
            $errors['username'] = 'Username must be 3-32 characters';
        }
    }
    
    // Password validation
    if (!empty($data['password'])) {
        $passwordErrors = validatePassword($data['password']);
        if (!empty($passwordErrors)) {
            $errors['password'] = $passwordErrors;
        }
    }
    
    return $errors;
}

function validatePassword(string $password): array
{
    $errors = [];
    
    if (strlen($password) < 8) {
        $errors[] = 'Password must be at least 8 characters';
    }
    
    if (!preg_match('/[A-Z]/', $password)) {
        $errors[] = 'Password must contain an uppercase letter';
    }
    
    if (!preg_match('/[a-z]/', $password)) {
        $errors[] = 'Password must contain a lowercase letter';
    }
    
    if (!preg_match('/[0-9]/', $password)) {
        $errors[] = 'Password must contain a number';
    }
    
    return $errors;
}
```

## Form Validation Helper

```php
<?php
class FormValidator
{
    private array $data;
    private array $rules;
    private array $errors = [];
    
    public function __construct(array $data, array $rules)
    {
        $this->data = $data;
        $this->rules = $rules;
    }
    
    public function validate(): bool
    {
        foreach ($this->rules as $field => $ruleString) {
            $rules = explode('|', $ruleString);
            
            foreach ($rules as $rule) {
                $this->applyRule($field, $rule);
            }
        }
        
        return empty($this->errors);
    }
    
    private function applyRule(string $field, string $rule): void
    {
        $value = $this->data[$field] ?? null;
        $params = [];
        
        if (strpos($rule, ':') !== false) {
            [$rule, $paramString] = explode(':', $rule, 2);
            $params = explode(',', $paramString);
        }
        
        $method = 'validate' . ucfirst($rule);
        
        if (method_exists($this, $method)) {
            if (!$this->$method($field, $value, $params)) {
                // Rule failed
            }
        }
    }
    
    private function validateRequired(string $field, $value, array $params): bool
    {
        if (empty($value) && $value !== '0') {
            $this->errors[$field][] = ucfirst($field) . ' is required';
            return false;
        }
        return true;
    }
    
    private function validateEmail(string $field, $value, array $params): bool
    {
        if (!empty($value) && !filter_var($value, FILTER_VALIDATE_EMAIL)) {
            $this->errors[$field][] = 'Invalid email address';
            return false;
        }
        return true;
    }
    
    private function validateMin(string $field, $value, array $params): bool
    {
        $min = (int) $params[0];
        
        if (is_numeric($value) && $value < $min) {
            $this->errors[$field][] = ucfirst($field) . ' must be at least ' . $min;
            return false;
        }
        
        if (is_string($value) && strlen($value) < $min) {
            $this->errors[$field][] = ucfirst($field) . ' must be at least ' . $min . ' characters';
            return false;
        }
        
        return true;
    }
    
    private function validateMax(string $field, $value, array $params): bool
    {
        $max = (int) $params[0];
        
        if (is_numeric($value) && $value > $max) {
            $this->errors[$field][] = ucfirst($field) . ' must not exceed ' . $max;
            return false;
        }
        
        if (is_string($value) && strlen($value) > $max) {
            $this->errors[$field][] = ucfirst($field) . ' must not exceed ' . $max . ' characters';
            return false;
        }
        
        return true;
    }
    
    public function getErrors(): array
    {
        return $this->errors;
    }
    
    public function getFirstError(string $field): ?string
    {
        return $this->errors[$field][0] ?? null;
    }
}

// Usage
$validator = new FormValidator($_POST, [
    'email' => 'required|email|max:255',
    'firstname' => 'required|min:2|max:100',
    'lastname' => 'required|min:2|max:100',
    'amount' => 'required|numeric|min:0',
]);

if (!$validator->validate()) {
    $errors = $validator->getErrors();
    // Handle errors
}
```

## Custom Validation Rules

```php
<?php
class CustomValidationRules
{
    public static function validateDomain(string $domain): bool
    {
        // Check domain format
        if (!preg_match('/^(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}$/', $domain)) {
            return false;
        }
        
        // Check TLD length
        $parts = explode('.', $domain);
        $tld = end($parts);
        
        if (strlen($tld) < 2 || strlen($tld) > 10) {
            return false;
        }
        
        return true;
    }
    
    public static function validatePhone(string $phone, string $country = 'US'): bool
    {
        $phone = preg_replace('/[^0-9+]/', '', $phone);
        
        // Basic length check
        if (strlen($phone) < 10 || strlen($phone) > 15) {
            return false;
        }
        
        // Country-specific validation could go here
        return true;
    }
    
    public static function validateUsername(string $username): bool
    {
        // Allow alphanumeric, underscore, hyphen
        if (!preg_match('/^[a-zA-Z0-9_-]+$/', $username)) {
            return false;
        }
        
        // Length check
        if (strlen($username) < 3 || strlen($username) > 32) {
            return false;
        }
        
        // Reserved names
        $reserved = ['admin', 'root', 'system', 'support'];
        if (in_array(strtolower($username), $reserved)) {
            return false;
        }
        
        return true;
    }
    
    public static function validateDateRange(string $startDate, string $endDate): bool
    {
        $start = strtotime($startDate);
        $end = strtotime($endDate);
        
        if ($start === false || $end === false) {
            return false;
        }
        
        return $start <= $end;
    }
}
```

## Sanitization

```php
<?php
class InputSanitizer
{
    public static function sanitizeString(string $input, int $maxLength = 255): string
    {
        $input = trim($input);
        $input = strip_tags($input);
        $input = htmlspecialchars($input, ENT_QUOTES, 'UTF-8');
        
        return substr($input, 0, $maxLength);
    }
    
    public static function sanitizeEmail(string $email): string
    {
        $email = trim(strtolower($email));
        return filter_var($email, FILTER_SANITIZE_EMAIL);
    }
    
    public static function sanitizeInteger($input): int
    {
        return (int) filter_var($input, FILTER_SANITIZE_NUMBER_INT);
    }
    
    public static function sanitizeFloat($input): float
    {
        return (float) filter_var($input, FILTER_SANITIZE_NUMBER_FLOAT, FILTER_FLAG_ALLOW_FRACTION);
    }
    
    public static function sanitizeUrl(string $url): string
    {
        return filter_var($url, FILTER_SANITIZE_URL);
    }
    
    public static function sanitizeArray(array $array, array $rules): array
    {
        $sanitized = [];
        
        foreach ($rules as $field => $type) {
            if (isset($array[$field])) {
                $method = 'sanitize' . ucfirst($type);
                $sanitized[$field] = self::$method($array[$field]);
            }
        }
        
        return $sanitized;
    }
}
```

## Best Practices

1. **Validate server-side** - Never rely on client-side validation
2. **Sanitize all input** - Clean data before storage
3. **Use whitelists** - Define allowed values explicitly
4. **Return detailed errors** - Help users fix input
5. **Log validation failures** - Track attack attempts

## Related Documentation

- [WHMCS Module Security](/docs/whmcs-module-security.md)
- [WHMCS Form Handling](/docs/whmcs-form-handling.md)