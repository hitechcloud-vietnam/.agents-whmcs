# WHMCS Email Conditional Content Workflow

## Purpose
Create dynamic email content that changes based on client data or conditions.

## Syntax Overview

### Basic If/Then/Else
```
{if $variable == "value"}Content{/if}
{if $variable != "value"}Content{/if}
```

### Multiple Conditions
```
{if $variable1 == "A" && $variable2 == "B"}Content{/if}
{if $variable1 == "A" || $variable2 == "B"}Content{/if}
```

### Nested Conditions
```
{if $variable1 == "A"}
   {if $variable2 == "B"}
      Content A and B
   {/if}
{/if}
```

## Common Use Cases

### Client Status
```
{if $client_status == "Active"}
   Thank you for your continued business!
{else}
   We look forward to serving you!
{/if}
```

### Product Type
```
{if $product_name|lower contains "vps"}
   Your VPS management panel: {$vps_url}
{elseif $product_name|lower contains "shared"}
   Access your cPanel: {$cpanel_url}
{else}
   Account dashboard: {$account_url}
{/if}
```

### Payment Status
```
{if $invoice_balance > 0}
   Please complete payment of {$invoice_balance}
{else}
   No payment required - thank you!
{/if}
```

### Country-Based
```
{if $client_country == "US"}
   US customers: 1-800-XXX-XXXX
{elseif $client_country == "UK"}
   UK customers: 0800-XXX-XXXX
{else}
   International: +1-XXX-XXX-XXXX
{/if}
```

### Date-Based
```
{if $days_until_expiry <= 7}
   Your service expires in {$days_until_expiry} days!
{else}
   Service renewal date: {$next_due_date}
{/if}
```

## Real-World Examples

### Renewal Notice
```
{if $days_until_expiry <= 7}
   <strong>URGENT: {$product_name} expires in {$days_until_expiry} days!</strong>
   Renew now to avoid service interruption.
{elseif $days_until_expiry <= 30}
   Your {$product_name} will expire on {$next_due_date}.
   Early renewal discount available!
{/if}
```

### Welcome with Add-ons
```
{if $has_addons}
   Your package includes:
   {foreach $addons as $addon}
      - {$addon.name}
   {/foreach}
{/if}
```

## Debugging Tips
- Preview template with sample data
- Use test email feature
- Check for typos in variable names
- Ensure closing tags {/if} are present

## Related Workflows
- whmcs-email-variables
- whmcs-email-template-create