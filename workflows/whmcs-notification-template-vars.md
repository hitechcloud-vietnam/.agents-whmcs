# WHMCS Notification Template Variables Workflow

## Purpose
Master the use of variables in notification templates.

## Variable Syntax

### Standard Variables
```
{$variable_name}
```

### Conditional Variables
```
{if $var}content{/if}
```

### Loop Variables
```
{foreach $items as $item}{$item.name}{/foreach}
```

## Available Variables

### Client Variables
```
{$client_id}
{$client_name}
{$client_first_name}
{$client_last_name}
{$client_email}
{$client_company_name}
{$client_phone}
{$client_group}
{$client_status}
{$client_language}
{$client_currency}
```

### Order Variables
```
{$order_id}
{$order_number}
{$order_date}
{$order_total}
{$order_status}
{$order_products}
{$order_ip}
{$order_payment_method}
```

### Invoice Variables
```
{$invoice_id}
{$invoice_number}
{$invoice_date}
{$invoice_due_date}
{$invoice_amount}
{$invoice_tax}
{$invoice_balance}
{$invoice_status}
{$invoice_url}
{$payment_url}
```

### Service Variables
```
{$service_id}
{$service_name}
{$product_name}
{$domain}
{$registration_date}
{$next_due_date}
{$suspend_reason}
{$termination_date}
{$dedicated_ip}
```

### Ticket Variables
```
{$ticket_id}
{$ticket_number}
{$ticket_subject}
{$ticket_priority}
{$ticket_status}
{$ticket_department}
{$ticket_flag}
{$ticket_message}
```

## Event-Specific Variables

### New Order Event
```
{$order_number}
{$order_products}
{$order_total}
{$payment_method}
{$client_name}
{$client_email}
```

### Invoice Paid Event
```
{$invoice_number}
{$invoice_amount}
{$payment_method}
{$payment_date}
{$client_name}
```

### Ticket Reply Event
```
{$ticket_number}
{$ticket_subject}
{$reply_author}
{$reply_message}
{$ticket_status}
```

### Service Activation Event
```
{$service_id}
{$product_name}
{$domain}
{$next_due_date}
{$login_url}
```

## System Variables

### Global
```
{$date}              - Current date
{$time}              - Current time
{$company_name}      - WHMCS company name
{$company_url}       - WHMCS URL
{$admin_url}         - Admin URL
```

### Admin
```
{$admin_name}
{$admin_email}
{$admin_first_name}
{$admin_last_name}
```

## Custom Variables

### Via Hook
```php
add_hook('NotificationData', 1, function($vars) {
    return [
        'custom_field' => 'Custom Value',
        'calculated' => $vars['order_total'] * 0.1
    ];
});
```

### Via Module
```php
return [
    'custom_variable' => 'value',
    'dynamic_data' => $dynamicValue
];
```

## Formatting Variables

### Date Formatting
```
{$date|date_format:"Y-m-d"}
{$next_due_date|date_format:"d M Y"}
```

### Currency Formatting
```
{$amount|currency_format}
{$invoice_total|money_format:"USD"}
```

### String Formatting
```
{$client_name|upper}
{$product_name|lower}
{$message|truncate:100}
```

## Testing Variables

### Preview with Test Data
1. Select notification
2. Click "Preview"
3. Choose test client
4. View rendered output

## Related Workflows
- whmcs-notification-templates
- whmcs-notification-create
- whmcs-email-variables