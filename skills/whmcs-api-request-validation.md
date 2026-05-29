# WHMCS API Request Validation

## Skill Description
Implement comprehensive input validation for WHMCS API requests including field validation, type checking, sanitization, and consistent error responses.

## Prerequisites
- WHMCS 7.0+ installation
- PHP 7.4+
- Understanding of validation patterns
- Basic knowledge of security best practices

## Step-by-Step Implementation

### 1. Validation Rule Classes
```php
<?php
// includes/validation/Rule.php

namespace WHMCS\Module\YourModule\Validation;

abstract class Rule
{
    public string $message = '';
    public string $field = '';

    abstract public function validate(mixed $value): bool;

    public function setField(string $field): self
    {
        $this->field = $field;
        return $this;
    }

    public function setMessage(string $message): self
    {
        $this->message = $message;
        return $this;
    }

    public function getMessage(): string
    {
        return $this->message ?: "The {$this->field} field is invalid";
    }
}
```

```php
<?php
// includes/validation/Rules/RequiredRule.php

namespace WHMCS\Module\YourModule\Validation\Rules;

use WHMCS\Module\YourModule\Validation\Rule;

class RequiredRule extends Rule
{
    public function validate(mixed $value): bool
    {
        if ($value === null) {
            return false;
        }

        if (is_string($value) && trim($value) === '') {
            return false;
        }

        if (is_array($value) && count($value) === 0) {
            return false;
        }

        return true;
    }

    public function getMessage(): string
    {
        return "The {$this->field} field is required";
    }
}
```

```php
<?php
// includes/validation/Rules/EmailRule.php

namespace WHMCS\Module\YourModule\Validation\Rules;

use WHMCS\Module\YourModule\Validation\Rule;

class EmailRule extends Rule
{
    public function validate(mixed $value): bool
    {
        if ($value === null || $value === '') {
            return true; // Use required rule for empty check
        }

        return filter_var($value, FILTER_VALIDATE_EMAIL) !== false;
    }

    public function getMessage(): string
    {
        return "The {$this->field} must be a valid email address";
    }
}
```

```php
<?php
// includes/validation/Rules/MinRule.php

namespace WHMCS\Module\YourModule\Validation\Rules;

use WHMCS\Module\YourModule\Validation\Rule;

class MinRule extends Rule
{
    private int $min;

    public function __construct(int $min)
    {
        $this->min = $min;
    }

    public function validate(mixed $value): bool
    {
        if ($value === null || $value === '') {
            return true;
        }

        if (is_numeric($value)) {
            return $value >= $this->min;
        }

        if (is_string($value)) {
            return strlen($value) >= $this->min;
        }

        if (is_array($value)) {
            return count($value) >= $this->min;
        }

        return false;
    }

    public function getMessage(): string
    {
        return "The {$this->field} must be at least {$this->min}";
    }
}
```

```php
<?php
// includes/validation/Rules/MaxRule.php

namespace WHMCS\Module\YourModule\Validation\Rules;

use WHMCS\Module\YourModule\Validation\Rule;

class MaxRule extends Rule
{
    private int $max;

    public function __construct(int $max)
    {
        $this->max = $max;
    }

    public function validate(mixed $value): bool
    {
        if ($value === null || $value === '') {
            return true;
        }

        if (is_numeric($value)) {
            return $value <= $this->max;
        }

        if (is_string($value)) {
            return strlen($value) <= $this->max;
        }

        if (is_array($value)) {
            return count($value) <= $this->max;
        }

        return false;
    }

    public function getMessage(): string
    {
        return "The {$this->field} must not exceed {$this->max}";
    }
}
```

```php
<?php
// includes/validation/Rules/InRule.php

namespace WHMCS\Module\YourModule\Validation\Rules;

use WHMCS\Module\YourModule\Validation\Rule;

class InRule extends Rule
{
    private array $values;

    public function __construct(array $values)
    {
        $this->values = $values;
    }

    public function validate(mixed $value): bool
    {
        if ($value === null || $value === '') {
            return true;
        }

        return in_array($value, $this->values, true);
    }

    public function getMessage(): string
    {
        $allowed = implode(', ', $this->values);
        return "The {$this->field} must be one of: {$allowed}";
    }
}
```

```php
<?php
// includes/validation/Rules/RegexRule.php

namespace WHMCS\Module\YourModule\Validation\Rules;

use WHMCS\Module\YourModule\Validation\Rule;

class RegexRule extends Rule
{
    private string $pattern;

    public function __construct(string $pattern)
    {
        $this->pattern = $pattern;
    }

    public function validate(mixed $value): bool
    {
        if ($value === null || $value === '') {
            return true;
        }

        return preg_match($this->pattern, $value) === 1;
    }

    public function getMessage(): string
    {
        return "The {$this->field} format is invalid";
    }
}
```

```php
<?php
// includes/validation/Rules/DateRule.php

namespace WHMCS\Module\YourModule\Validation\Rules;

use WHMCS\Module\YourModule\Validation\Rule;

class DateRule extends Rule
{
    private string $format;

    public function __construct(string $format = 'Y-m-d')
    {
        $this->format = $format;
    }

    public function validate(mixed $value): bool
    {
        if ($value === null || $value === '') {
            return true;
        }

        $date = \DateTime::createFromFormat($this->format, $value);

        if ($date === false) {
            return false;
        }

        return $date->format($this->format) === $value;
    }

    public function getMessage(): string
    {
        return "The {$this->field} must be a valid date in format {$this->format}";
    }
}
```

### 2. Validator Class
```php
<?php
// includes/validation/Validator.php

namespace WHMCS\Module\YourModule\Validation;

use WHMCS\Module\YourModule\Validation\Rules\RequiredRule;
use WHMCS\Module\YourModule\Validation\Rules\EmailRule;
use WHMCS\Module\YourModule\Validation\Rules\MinRule;
use WHMCS\Module\YourModule\Validation\Rules\MaxRule;
use WHMCS\Module\YourModule\Validation\Rules\InRule;
use WHMCS\Module\YourModule\Validation\Rules\RegexRule;
use WHMCS\Module\YourModule\Validation\Rules\DateRule;

class Validator
{
    private array $data;
    private array $rules = [];
    private array $errors = [];
    private array $validated = [];

    public function __construct(array $data)
    {
        $this->data = $data;
    }

    public static function make(array $data, array $rules): self
    {
        $validator = new self($data);

        foreach ($rules as $field => $fieldRules) {
            $rulesArray = is_string($fieldRules) ? explode('|', $fieldRules) : $fieldRules;
            foreach ($rulesArray as $rule) {
                $validator->addRule($field, $rule);
            }
        }

        return $validator;
    }

    public function addRule(string $field, mixed $rule): self
    {
        if (is_string($rule)) {
            $rule = $this->parseRuleString($rule);
        }

        if ($rule instanceof Rule) {
            $rule->setField($field);
            $this->rules[$field][] = $rule;
        }

        return $this;
    }

    private function parseRuleString(string $rule): ?Rule
    {
        if (strpos($rule, ':') !== false) {
            [$name, $param] = explode(':', $rule, 2);
        } else {
            $name = $rule;
            $param = null;
        }

        return match ($name) {
            'required' => new RequiredRule(),
            'email' => new EmailRule(),
            'min' => new MinRule((int) $param),
            'max' => new MaxRule((int) $param),
            'in' => new InRule(explode(',', $param)),
            'regex' => new RegexRule($param),
            'date' => new DateRule($param ?? 'Y-m-d'),
            'numeric' => new RegexRule('/^\d+$/'),
            'alpha' => new RegexRule('/^[a-zA-Z]+$/'),
            'alphanumeric' => new RegexRule('/^[a-zA-Z0-9]+$/'),
            'url' => new RegexRule('/^https?:\/\/.+/'),
            'ip' => new RegexRule('/^[\d\.]+$/'),
            default => null
        };
    }

    public function validate(): bool
    {
        $this->errors = [];
        $this->validated = [];

        foreach ($this->rules as $field => $rules) {
            $value = $this->data[$field] ?? null;
            $fieldErrors = [];

            foreach ($rules as $rule) {
                if (!$rule->validate($value)) {
                    $fieldErrors[] = $rule->getMessage();
                }
            }

            if (empty($fieldErrors)) {
                $this->validated[$field] = $value;
            } else {
                $this->errors[$field] = $fieldErrors;
            }
        }

        return empty($this->errors);
    }

    public function fails(): bool
    {
        return !$this->validate();
    }

    public function errors(): array
    {
        if (empty($this->errors)) {
            $this->validate();
        }

        return $this->errors;
    }

    public function validated(): array
    {
        if (empty($this->validated)) {
            $this->validate();
        }

        return $this->validated;
    }

    public function failed(): array
    {
        return $this->errors();
    }
}
```

### 3. Sanitizer Class
```php
<?php
// includes/validation/Sanitizer.php

namespace WHMCS\Module\YourModule\Validation;

class Sanitizer
{
    private array $data;

    public function __construct(array $data)
    {
        $this->data = $data;
    }

    public static function clean(array $data, array $rules): array
    {
        $sanitizer = new self($data);
        return $sanitizer->applyRules($rules);
    }

    private function applyRules(array $rules): array
    {
        $cleaned = [];

        foreach ($rules as $field => $fieldRules) {
            $value = $this->data[$field] ?? null;

            if ($value === null && !in_array('nullable', $fieldRules)) {
                continue;
            }

            $cleaned[$field] = $this->applyFieldRules($value, $fieldRules);
        }

        return $cleaned;
    }

    private function applyFieldRules(mixed $value, array $rules): mixed
    {
        if ($value === null) {
            return null;
        }

        foreach ($rules as $rule) {
            $value = $this->applyRule($value, $rule);
        }

        return $value;
    }

    private function applyRule(mixed $value, string $rule): mixed
    {
        if (strpos($rule, ':') !== false) {
            [$name, $param] = explode(':', $rule, 2);
        } else {
            $name = $rule;
            $param = null;
        }

        return match ($name) {
            'trim' => is_string($value) ? trim($value) : $value,
            'strip_tags' => is_string($value) ? strip_tags($value) : $value,
            'escape' => is_string($value) ? htmlspecialchars($value, ENT_QUOTES, 'UTF-8') : $value,
            'lower' => is_string($value) ? strtolower($value) : $value,
            'upper' => is_string($value) ? strtoupper($value) : $value,
            'int' => (int) $value,
            'float' => (float) $value,
            'bool' => (bool) $value,
            'string' => (string) $value,
            'abs' => abs((int) $value),
            'url' => filter_var($value, FILTER_SANITIZE_URL),
            'email' => filter_var($value, FILTER_SANITIZE_EMAIL),
            'remove_accents' => $this->removeAccents($value),
            default => $value
        };
    }

    private function removeAccents(string $value): string
    {
        $transliteration = [
            'Š' => 'S', 'š' => 's', 'Ž' => 'Z', 'ž' => 'z', 'Ð' => 'Dj', 'đ' => 'dj',
            'À' => 'A', 'Á' => 'A', 'Â' => 'A', 'Ã' => 'A', 'Ä' => 'A', 'Å' => 'A',
            'Æ' => 'Ae', 'Ç' => 'C', 'È' => 'E', 'É' => 'E', 'Ê' => 'E', 'Ë' => 'E',
            'Ì' => 'I', 'Í' => 'I', 'Î' => 'I', 'Ï' => 'I', 'Ñ' => 'N', 'Ò' => 'O',
            'Ó' => 'O', 'Ô' => 'O', 'Õ' => 'O', 'Ö' => 'O', 'Ø' => 'O', 'Ù' => 'U',
            'Ú' => 'U', 'Û' => 'U', 'Ü' => 'U', 'Ý' => 'Y', 'Þ' => 'B', 'ß' => 'Ss',
            'à' => 'a', 'á' => 'a', 'â' => 'a', 'ã' => 'a', 'ä' => 'a', 'å' => 'a',
            'æ' => 'ae', 'ç' => 'c', 'è' => 'e', 'é' => 'e', 'ê' => 'e', 'ë' => 'e',
            'ì' => 'i', 'í' => 'i', 'î' => 'i', 'ï' => 'i', 'ð' => 'o', 'ñ' => 'n',
            'ò' => 'o', 'ó' => 'o', 'ô' => 'o', 'õ' => 'o', 'ö' => 'o', 'ø' => 'o',
            'ù' => 'u', 'ú' => 'u', 'û' => 'u', 'ü' => 'u', 'ý' => 'y', 'þ' => 'b',
            'ÿ' => 'y', 'ƒ' => 'f'
        ];

        return strtr($value, $transliteration);
    }
}
```

### 4. Usage Examples
```php
<?php
// Example validation in a controller

namespace WHMCS\Module\YourModule\Api\Controllers;

use WHMCS\Module\YourModule\Api\ApiController;
use WHMCS\Module\YourModule\Api\ApiResponse;
use WHMCS\Module\YourModule\Validation\Validator;
use WHMCS\Module\YourModule\Validation\Sanitizer;

class UserController extends ApiController
{
    public function store(): void
    {
        // Sanitize input
        $sanitized = Sanitizer::clean($this->requestData, [
            'first_name' => ['trim', 'strip_tags', 'string'],
            'last_name' => ['trim', 'strip_tags', 'string'],
            'email' => ['trim', 'lower', 'escape', 'email'],
            'age' => ['int'],
            'website' => ['trim', 'url']
        ]);

        // Validate input
        $validator = Validator::make($sanitized, [
            'first_name' => ['required', 'min:2', 'max:100'],
            'last_name' => ['required', 'min:2', 'max:100'],
            'email' => ['required', 'email', 'max:255'],
            'age' => ['required', 'min:18', 'max:150'],
            'status' => ['required', 'in:active,inactive,suspended']
        ]);

        if ($validator->fails()) {
            ApiResponse::validationError($validator->errors())->send();
            return;
        }

        $data = $validator->validated();

        // Proceed with creating the user...
        ApiResponse::created(['user_id' => 123])->send();
    }

    public function update(): void
    {
        $validator = Validator::make($this->requestData, [
            'id' => ['required', 'numeric'],
            'first_name' => ['min:2', 'max:100'],
            'last_name' => ['min:2', 'max:100'],
            'email' => ['email', 'max:255']
        ]);

        if ($validator->fails()) {
            ApiResponse::validationError($validator->errors())->send();
            return;
        }

        $data = $validator->validated();

        // Proceed with updating the user...
        ApiResponse::success(['message' => 'User updated'])->send();
    }
}
```

## Common Pitfalls and Solutions

| Pitfall | Solution |
|---------|----------|
| Missing required fields | Always define required rules explicitly |
| Type confusion attacks | Use strict type checking in rules |
| XSS through input fields | Always sanitize HTML input |
| SQL injection via input | Use parameterized queries, never user input directly |
| Overly permissive validation | Whitelist allowed values where possible |

## Security Considerations

1. **Always sanitize before validation** - Clean data first
2. **Use strict type checks** - Avoid loose comparisons
3. **Whitelist validation** - Prefer 'in' rules over blacklist
4. **Validate on server side** - Never trust client-side validation
5. **Log validation failures** - Track potential attack attempts
6. **Use prepared statements** - For database operations with user input

## Testing Checklist

- [ ] Test required field validation
- [ ] Test email validation with valid/invalid emails
- [ ] Test numeric ranges with boundary values
- [ ] Test regex patterns with matching/non-matching input
- [ ] Test 'in' rule with allowed/disallowed values
- [ ] Test date format validation
- [ ] Test sanitization removes dangerous content
- [ ] Test empty string vs null handling
- [ ] Test array input validation
- [ ] Test max length with very long strings

## Reference Links

- [OWASP Input Validation](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [PHP Filter Functions](https://www.php.net/manual/en/filter.filters.sanitize.php)
- [WHMCS Security Guidelines](https://developers.whmcs.com/security/)
