# WHMCS Email Variables Workflow

## Purpose
Master the use of template variables in WHMCS email templates.

## Common Variable Categories

### Client Variables
```
{$client_name}           - Full name
{$client_first_name}     - First name only
{$client_last_name}      - Last name only
{$client_email}          - Email address
{$client_company_name}    - Company name
{$client_phone}          - Phone number
{$client_address1}       - Address line 1
{$client_address2}       - Address line 2
{$client_city}           - City
{$client_state}          - State/Region
{$client_postcode}       - Postal code
{$client_country}       - Country
{$client_language}       - Language code
{$client_status}         - Active/Inactive
```

### Service/Product Variables
```
{$service_id}            - Service ID
{$product_name}          - Product name
{$product_description}   - Product description
{$domain}                - Domain name
{$registration_date}    - Registration date
{$next_due_date}        - Next due date
{$termination_date}     - Termination date
{$dedicated_ip}         - Dedicated IP
{$assigned_ip}          - Assigned IP
```

### Invoice Variables
```
{$invoice_num}          - Invoice number
{$invoice_date}         - Invoice date
{$due_date}             - Due date
{$amount}               - Total amount
{$tax}                  - Tax amount
{$subtotal}             - Subtotal
{$total}                - Total
{$balance}              - Balance due
```

### Order Variables
```
{$order_num}            - Order number
{$order_date}           - Order date
{$order_total}          - Order total
{$items}                - Order items
```

### Support Ticket Variables
```
{$ticket_id}            - Ticket ID
{$ticket_subject}       - Ticket subject
{$ticket_priority}      - Priority level
{$ticket_status}       - Status
{$ticket_department}    - Department
{$ticket_created}      - Creation date
```

## Usage Examples

### Welcome Email
```
Dear {$client_first_name},

Welcome to {$company_name}!

Your account has been created with email: {$client_email}

Login at: {$login_url}
```

### Invoice Email
```
Dear {$client_name},

Invoice #{$invoice_num} for {$amount} is due on {$due_date}.

View invoice: {$invoice_url}
```

## Advanced: Conditional Variables
```
{if $client_status eq "Active"}Thank you for being an active client!{/if}
```

## Related Workflows
- whmcs-email-template-create
- whmcs-email-conditional