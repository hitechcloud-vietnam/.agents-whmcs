# WHMCS Admin Validation

## Overview
Guide for implementing form validation in WHMCS admin area. Covers client-side validation, server-side validation, and custom validators.

## Validation System

### Validator Class

```php
<?php
// /includes/hooks/admin_validation.php

class Validator {
    private array $data;
    private array $rules;
    private array $errors = [];
    private array $messages = [];
    
    public function __construct(array $data, array $rules) {
        $this->data = $data;
        $this->rules = $rules;
    }
    
    public function validate(): bool {
        foreach ($this->rules as $field => $ruleString) {
            $rules = explode("|", $ruleString);
            $value = $this->data[$field] ?? null;
            
            foreach ($rules as $rule) {
                $params = [];
                
                if (strpos($rule, ":") !== false) {
                    [$rule, $paramString] = explode(":", $rule);
                    $params = explode(",", $paramString);
                }
                
                $error = $this->validateRule($field, $value, $rule, $params);
                
                if ($error !== true) {
                    $this->addError($field, $error);
                    break;
                }
            }
        }
        
        return empty($this->errors);
    }
    
    private function validateRule(string $field, $value, string $rule, array $params = []): bool|string {
        switch ($rule) {
            case "required":
                if (empty($value) && $value !== "0") {
                    return $this->getMessage($field, "required");
                }
                return true;
                
            case "email":
                if ($value && !filter_var($value, FILTER_VALIDATE_EMAIL)) {
                    return $this->getMessage($field, "email");
                }
                return true;
                
            case "min":
                if (strlen($value) < $params[0]) {
                    return sprintf($this->getMessage($field, "min"), $params[0]);
                }
                return true;
                
            case "max":
                if (strlen($value) > $params[0]) {
                    return sprintf($this->getMessage($field, "max"), $params[0]);
                }
                return true;
                
            case "numeric":
                if ($value && !is_numeric($value)) {
                    return $this->getMessage($field, "numeric");
                }
                return true;
                
            case "unique":
                $table = $params[0];
                $column = $params[1] ?? $field;
                $exceptId = $params[2] ?? null;
                
                $query = Capsule::table($table)->where($column, $value);
                if ($exceptId) {
                    $query->where("id", "!=", $exceptId);
                }
                
                if ($query->exists()) {
                    return $this->getMessage($field, "unique");
                }
                return true;
                
            case "exists":
                $table = $params[0];
                $column = $params[1] ?? $field;
                
                if (!Capsule::table($table)->where($column, $value)->exists()) {
                    return sprintf($this->getMessage($field, "exists"), $value);
                }
                return true;
                
            case "date":
                if ($value && !strtotime($value)) {
                    return $this->getMessage($field, "date");
                }
                return true;
                
            case "regex":
                if ($value && !preg_match($params[0], $value)) {
                    return $this->getMessage($field, "regex");
                }
                return true;
        }
        
        return true;
    }
    
    private function addError(string $field, string $message): void {
        if (!isset($this->errors[$field])) {
            $this->errors[$field] = [];
        }
        $this->errors[$field][] = $message;
    }
    
    private function getMessage(string $field, string $rule): string {
        if (isset($this->messages[$field . "." . $rule])) {
            return $this->messages[$field . "." . $rule];
        }
        
        $defaultMessages = [
            "required" => "This field is required",
            "email" => "Please enter a valid email address",
            "min" => "Field must be at least %d characters",
            "max" => "Field must not exceed %d characters",
            "numeric" => "Please enter a valid number",
            "unique" => "This value already exists",
            "exists" => "Selected value (%s) does not exist",
            "date" => "Please enter a valid date",
            "regex" => "Field format is invalid"
        ];
        
        return $defaultMessages[$rule] ?? "Validation failed";
    }
    
    public function setMessage(string $field, string $rule, string $message): self {
        $this->messages[$field . "." . $rule] = $message;
        return $this;
    }
    
    public function errors(): array {
        return $this->errors;
    }
    
    public function fails(): bool {
        return !empty($this->errors);
    }
}
```

### Usage Example

```php
add_hook("AdminValidateForm", 1, function(array $params) {
    $data = $_POST;
    $type = $params["type"];
    
    $rules = [];
    
    switch ($type) {
        case "client":
            $rules = [
                "firstname" => "required|min:2|max:50",
                "lastname" => "required|min:2|max:50",
                "email" => "required|email|unique:tblclients,email",
                "password" => "min:8",
                "country" => "required|exists:tblcountries,code"
            ];
            break;
            
        case "service":
            $rules = [
                "client_id" => "required|exists:tblclients,id",
                "product_id" => "required|exists:tblproducts,id",
                "domain" => "required|domain_unique",
                "billingcycle" => "required|in:monthly,quarterly,annually"
            ];
            break;
    }
    
    $validator = new Validator($data, $rules);
    
    $validator->setMessage("email", "unique", "A client with this email already exists.");
    
    if (!$validator->validate()) {
        return [
            "success" => false,
            "errors" => $validator->errors()
        ];
    }
    
    return ["success" => true];
});
```

## Client-Side Validation

```php
add_hook("AdminFormAssets", 1, function(array $params) {
    return [
        "js" => [
            "/assets/js/jquery.validate.min.js",
            "/assets/js/additional-methods.min.js"
        ]
    ];
});
```

```smarty
<!-- /admin/templates/validated_form.tpl -->
<form method="post" action="{$form_action}" id="validated-form" novalidate>
    <input type="hidden" name="token" value="{$token}">
    
    <div class="form-group">
        <label for="firstname">First Name *</label>
        <input type="text" name="firstname" id="firstname" 
               class="form-control"
               required minlength="2" maxlength="50">
        <label for="firstname" class="error"></label>
    </div>
    
    <div class="form-group">
        <label for="email">Email *</label>
        <input type="email" name="email" id="email" 
               class="form-control"
               required>
        <label for="email" class="error"></label>
    </div>
    
    <div class="form-group">
        <label for="password">Password</label>
        <input type="password" name="password" id="password" 
               class="form-control"
               minlength="8">
        <label for="password" class="error"></label>
    </div>
    
    <button type="submit" class="btn btn-primary">Submit</button>
</form>

<script>
$(function() {
    $("#validated-form").validate({
        rules: {
            firstname: {
                required: true,
                minlength: 2,
                maxlength: 50
            },
            email: {
                required: true,
                email: true,
                remote: {
                    url: "ajax.php?action=check_email",
                    type: "post"
                }
            },
            password: {
                minlength: 8
            }
        },
        messages: {
            firstname: {
                required: "First name is required",
                minlength: "First name must be at least 2 characters",
                maxlength: "First name cannot exceed 50 characters"
            },
            email: {
                required: "Email is required",
                email: "Please enter a valid email address",
                remote: "This email is already in use"
            },
            password: {
                minlength: "Password must be at least 8 characters"
            }
        },
        highlight: function(element) {
            $(element).closest('.form-group').addClass('has-error');
        },
        unhighlight: function(element) {
            $(element).closest('.form-group').removeClass('has-error');
        },
        errorPlacement: function(error, element) {
            error.addClass('text-danger');
            error.appendTo(element.parent());
        },
        submitHandler: function(form) {
            $.post($(form).attr('action'), $(form).serialize(), function(response) {
                if (response.success) {
                    window.location.href = response.redirect || 'index.php';
                } else {
                    // Handle server-side errors
                    $.each(response.errors, function(field, messages) {
                        var input = $('[name="' + field + '"]');
                        $.each(messages, function(i, msg) {
                            input.parent().append('<label class="error">' + msg + '</label>');
                        });
                    });
                }
            });
            return false;
        }
    });
});
</script>
```

## Best Practices

1. **Server-Side**: Always validate server-side (client-side is convenience)
2. **Clear Messages**: Provide clear, user-friendly error messages
3. **Real-Time**: Use real-time validation for better UX
4. **Sanitization**: Sanitize input after validation
5. **CSRF**: Include CSRF token validation
6. **Async Validation**: Support async validation for unique checks
7. **Field Highlighting**: Visually indicate validation errors
8. **Accessibility**: Ensure errors are accessible to screen readers
