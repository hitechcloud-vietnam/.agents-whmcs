# WHMCS Validation

Complete guide to input validation.

## Overview

Implement robust validation for WHMCS.

## Validator

```php
<?php
/**
 * Input validator
 */
class Validator
{
    private array $rules = [];
    private array $errors = [];
    
    /**
     * Add rule
     */
    public function rule(string $field, string $rule, ...$params): self
    {
        $this->rules[$field][] = [
            'rule' => $rule,
            'params' => $params,
        ];
        
        return $this;
    }
    
    /**
     * Validate data
     */
    public function validate(array $data): bool
    {
        $this->errors = [];
        
        foreach ($this->rules as $field => $rules) {
            foreach ($rules as $rule) {
                $value = $data[$field] ?? null;
                
                if (!$this->checkRule($rule['rule'], $value, $rule['params'])) {
                    $this->errors[$field][] = $this->getMessage($field, $rule['rule']);
                }
            }
        }
        
        return empty($this->errors);
    }
    
    /**
     * Check rule
     */
    private function checkRule(string $rule, $value, array $params): bool
    {
        return match($rule) {
            'required' => !empty($value),
            'email' => filter_var($value, FILTER_VALIDATE_EMAIL),
            'min' => strlen($value) >= $params[0],
            'max' => strlen($value) <= $params[0],
            'in' => in_array($value, $params),
            'regex' => preg_match($params[0], $value),
            default => true,
        };
    }
    
    /**
     * Get errors
     */
    public function errors(): array
    {
        return $this->errors;
    }
}
```

## Best Practices

1. **Validate early** - Check input at entry points
2. **Whitelist** - Allow only known good values
3. **Sanitize** - Clean data before validation
4. **Clear messages** - Provide helpful error messages
5. **Chain rules** - Combine multiple validations
6. **Test cases** - Cover edge cases

## Related Documentation

- [whmcs-advanced-security.md](whmcs-advanced-security.md)
- [whmcs-module-best-practices.md](whmcs-module-best-practices.md)
