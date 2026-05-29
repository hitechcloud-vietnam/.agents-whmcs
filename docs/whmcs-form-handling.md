# WHMCS Form Handling

## Overview

Proper form handling ensures data integrity and security in WHMCS.

## Basic Form Structure

```smarty
<form method="post" action="{$smarty.server.PHP_SELF}" class="form-horizontal">
    <input type="hidden" name="token" value="{$token}">
    <input type="hidden" name="action" value="submit">
    
    <div class="form-group">
        <label class="col-md-3 control-label" for="email">Email Address</label>
        <div class="col-md-6">
            <input type="email" name="email" id="email" 
                   class="form-control" required
                   value="{$data.email|escape}">
        </div>
    </div>
    
    <div class="form-group">
        <div class="col-md-6 col-md-offset-3">
            <button type="submit" class="btn btn-primary">
                Submit
            </button>
            <a href="cancel.php" class="btn btn-default">
                Cancel
            </a>
        </div>
    </div>
</form>
```

## Form Processing

```php
<?php
function processForm(array $post): array
{
    $errors = [];
    
    // Validate CSRF token
    if (!check_token('WHMCS.default')) {
        $errors[] = 'Invalid security token. Please refresh and try again.';
        return ['success' => false, 'errors' => $errors];
    }
    
    // Get and sanitize input
    $email = trim($post['email'] ?? '');
    $name = trim($post['name'] ?? '');
    $phone = trim($post['phone'] ?? '');
    
    // Validate email
    if (empty($email)) {
        $errors['email'] = 'Email is required';
    } elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $errors['email'] = 'Invalid email format';
    }
    
    // Validate name
    if (empty($name)) {
        $errors['name'] = 'Name is required';
    } elseif (strlen($name) < 2) {
        $errors['name'] = 'Name must be at least 2 characters';
    }
    
    // Validate phone
    if (!empty($phone) && !preg_match('/^[0-9+\-\s]+$/', $phone)) {
        $errors['phone'] = 'Invalid phone number';
    }
    
    if (empty($errors)) {
        // Process valid data
        saveToDatabase($email, $name, $phone);
        
        return [
            'success' => true,
            'message' => 'Form submitted successfully',
        ];
    }
    
    return [
        'success' => false,
        'errors' => $errors,
    ];
}
```

## Form Display with Errors

```php
<?php
function renderForm(array $data = [], array $errors = []): string
{
    $html = '<form method="post" class="form-horizontal">';
    $html .= '<input type="hidden" name="token" value="' . generate_token() . '">';
    
    // Email field
    $html .= '<div class="form-group ' . (isset($errors['email']) ? 'has-error' : '') . '">';
    $html .= '<label class="col-md-3 control-label" for="email">Email</label>';
    $html .= '<div class="col-md-6">';
    $html .= '<input type="email" name="email" id="email" class="form-control" value="' . htmlspecialchars($data['email'] ?? '') . '">';
    if (isset($errors['email'])) {
        $html .= '<span class="help-block text-danger">' . htmlspecialchars($errors['email']) . '</span>';
    }
    $html .= '</div></div>';
    
    // Submit button
    $html .= '<div class="form-group">';
    $html .= '<div class="col-md-6 col-md-offset-3">';
    $html .= '<button type="submit" class="btn btn-primary">Submit</button>';
    $html .= '</div></div>';
    
    $html .= '</form>';
    
    return $html;
}
```

## Smarty Form Template

```smarty
<form method="post" action="{$smarty.server.PHP_SELF}">
    <input type="hidden" name="token" value="{$token}">
    
    <div class="form-group {if $errors.email}has-error{/if}">
        <label for="email" class="control-label">Email Address</label>
        <input type="email" name="email" id="email" class="form-control" 
               value="{$formData.email|escape}" required>
        {if $errors.email}
            <span class="help-block text-danger">{$errors.email}</span>
        {/if}
    </div>
    
    <div class="form-group {if $errors.name}has-error{/if}">
        <label for="name" class="control-label">Full Name</label>
        <input type="text" name="name" id="name" class="form-control" 
               value="{$formData.name|escape}" required>
        {if $errors.name}
            <span class="help-block text-danger">{$errors.name}</span>
        {/if}
    </div>
    
    <div class="form-group {if $errors.message}has-error{/if}">
        <label for="message" class="control-label">Message</label>
        <textarea name="message" id="message" class="form-control" 
                  rows="5">{$formData.message|escape}</textarea>
        {if $errors.message}
            <span class="help-block text-danger">{$errors.message}</span>
        {/if}
    </div>
    
    <div class="form-group">
        <button type="submit" class="btn btn-primary">Submit</button>
    </div>
</form>
```

## Multi-Step Forms

```php
<?php
class MultiStepForm
{
    private int $currentStep = 1;
    private int $totalSteps = 3;
    private array $formData = [];
    
    public function __construct()
    {
        $this->loadFromSession();
    }
    
    private function loadFromSession(): void
    {
        $this->currentStep = $_SESSION['form_step'] ?? 1;
        $this->formData = $_SESSION['form_data'] ?? [];
    }
    
    public function processStep(array $data): array
    {
        $errors = $this->validateStep($this->currentStep, $data);
        
        if (empty($errors)) {
            // Save data
            $this->formData = array_merge($this->formData, $data);
            $_SESSION['form_data'] = $this->formData;
            
            // Move to next step
            if ($this->currentStep < $this->totalSteps) {
                $this->currentStep++;
                $_SESSION['form_step'] = $this->currentStep;
                return ['success' => true, 'redirect' => true];
            } else {
                // Submit form
                return $this->submitForm();
            }
        }
        
        return ['success' => false, 'errors' => $errors];
    }
    
    private function validateStep(int $step, array $data): array
    {
        $errors = [];
        
        switch ($step) {
            case 1:
                if (empty($data['email'])) {
                    $errors['email'] = 'Email is required';
                }
                break;
            case 2:
                if (empty($data['name'])) {
                    $errors['name'] = 'Name is required';
                }
                break;
            case 3:
                if (empty($data['message'])) {
                    $errors['message'] = 'Message is required';
                }
                break;
        }
        
        return $errors;
    }
    
    public function reset(): void
    {
        unset($_SESSION['form_step'], $_SESSION['form_data']);
        $this->currentStep = 1;
        $this->formData = [];
    }
}
```

```smarty
<!-- Step 1: Email -->
{if $step == 1}
    <form method="post">
        <input type="hidden" name="step" value="1">
        <input type="hidden" name="token" value="{$token}">
        
        <h3>Step 1: Contact Information</h3>
        
        <div class="form-group">
            <label>Email</label>
            <input type="email" name="email" class="form-control" required>
        </div>
        
        <button type="submit" class="btn btn-primary">Next</button>
    </form>
{/if}

<!-- Step 2: Details -->
{if $step == 2}
    <form method="post">
        <input type="hidden" name="step" value="2">
        <input type="hidden" name="token" value="{$token}">
        
        <h3>Step 2: Your Details</h3>
        
        <div class="form-group">
            <label>Full Name</label>
            <input type="text" name="name" class="form-control" required>
        </div>
        
        <button type="submit" class="btn btn-default">Back</button>
        <button type="submit" class="btn btn-primary">Next</button>
    </form>
{/if}

<!-- Step 3: Confirmation -->
{if $step == 3}
    <form method="post">
        <input type="hidden" name="step" value="3">
        <input type="hidden" name="token" value="{$token}">
        
        <h3>Step 3: Review & Submit</h3>
        
        <div class="review-box">
            <p><strong>Email:</strong> {$formData.email}</p>
            <p><strong>Name:</strong> {$formData.name}</p>
        </div>
        
        <button type="submit" class="btn btn-default">Back</button>
        <button type="submit" class="btn btn-success">Submit</button>
    </form>
{/if}
```

## AJAX Form Submission

```javascript
$(document).ready(function() {
    $('#ajax-form').on('submit', function(e) {
        e.preventDefault();
        
        const $form = $(this);
        const $btn = $form.find('button[type="submit"]');
        
        $.ajax({
            url: $form.attr('action'),
            type: 'POST',
            data: $form.serialize(),
            beforeSend: function() {
                $btn.prop('disabled', true).text('Processing...');
            },
            success: function(response) {
                if (response.success) {
                    showMessage('success', response.message);
                    $form[0].reset();
                } else {
                    showErrors(response.errors);
                }
            },
            error: function() {
                showMessage('error', 'An error occurred. Please try again.');
            },
            complete: function() {
                $btn.prop('disabled', false).text('Submit');
            }
        });
    });
});
```

## Form Validation JavaScript

```javascript
const FormValidator = {
    rules: {
        email: {
            required: true,
            email: true
        },
        password: {
            required: true,
            minLength: 8
        },
        name: {
            required: true,
            minLength: 2
        }
    },
    
    validate: function(form) {
        const errors = {};
        
        for (const field in this.rules) {
            const value = form[field].value;
            const rules = this.rules[field];
            
            if (rules.required && !value) {
                errors[field] = 'This field is required';
                continue;
            }
            
            if (rules.email && value && !this.isEmail(value)) {
                errors[field] = 'Invalid email format';
            }
            
            if (rules.minLength && value.length < rules.minLength) {
                errors[field] = `Minimum ${rules.minLength} characters required`;
            }
        }
        
        return errors;
    },
    
    isEmail: function(email) {
        return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
    }
};
```

## Best Practices

1. **Always validate server-side** - Never trust client validation
2. **Use CSRF tokens** - Protect against CSRF attacks
3. **Sanitize input** - Clean data before processing
4. **Show clear errors** - Help users fix issues
5. **Preserve form data** - Keep user input on errors

## Related Documentation

- [WHMCS Data Validation](/docs/whmcs-data-validation.md)
- [WHMCS JavaScript & AJAX](/docs/whmcs-javascript-ajax.md)