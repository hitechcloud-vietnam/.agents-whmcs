# WHMCS Email Template Reference

## Overview

Email templates in WHMCS control transactional emails sent to clients. They use Smarty syntax and can be customized for branding and content.

## Template Structure

### Directory Layout

```
templates/email/
    index.html
    notification-body.html
    notification-footer.html
    notification-header.html
    ...
```

### Admin Email Templates

Email templates are managed through the WHMCS admin area under Configuration > System Settings > Email Templates.

## Email Variables

### Common Variables

```smarty
{$client_name}           {* Full name *}
{$client_first_name}     {* First name *}
{$client_last_name}       {* Last name *}
{$client_email}          {* Email address *}
{$client_company_name}   {* Company name *}
{$system_url}            {* WHMCS URL *}
{$company_name}          {* Company name *}
{$date}                  {* Current date *}
{$time}                  {* Current time *}
```

### Invoice Variables

```smarty
{$invoice_id}            {* Invoice number *}
{$invoice_num}           {* Invoice number *}
{$invoice_date}          {* Invoice date *}
{$invoice_due_date}      {* Due date *}
{$invoice_total}         {* Total amount *}
{$invoice_balance}       {* Balance due *}
{$invoice_url}           {* Invoice link *}
```

### Order Variables

```smarty
{$order_id}              {* Order ID *}
{$order_num}             {* Order number *}
{$order_date}            {* Order date *}
{$order_total}           {* Total amount *}
{$order_products}        {* Products table *}
```

### Service Variables

```smarty
{$service_product}       {* Product name *}
{$service_domain}        {* Domain *}
{$service_username}     {* Username *}
{$service_password}      {* Password *}
{$service_reg_date}      {* Registration date *}
{$service_next_due}      {* Next due date *}
```

### Ticket Variables

```smarty
{$ticket_id}              {* Ticket ID *}
{$ticket_tid}            {* Tracking ID *}
{$ticket_subject}        {* Subject *}
{$ticket_message}        {* Message *}
{$ticket_status}         {* Status *}
{$ticket_priority}       {* Priority *}
{$ticket_url}            {* Ticket link *}
```

## Email Template Types

### Client Email Templates

| Template Name | Purpose |
|----------------|---------|
| General - Client Login Details | Login credentials email |
| Account Payment Confirmation | Payment receipt |
| Automated Invoice | New invoice notification |
| Credit Card Payment Due | Payment reminder |
| Domain Transfer Complete | Domain transfer confirmation |
| New Password | Password reset email |
| Support Ticket Notification | Ticket update email |
| Welcome Email | New account welcome |

### Admin Email Templates

| Template Name | Purpose |
|----------------|---------|
| Admin Login Notification | Admin login alert |
| Client Signup Notification | New client signup |
| Invoice Created | New invoice alert |
| Order Placed | New order notification |
| Ticket Opened | New ticket alert |
| Suspension Notice | Service suspension warning |

## Template Syntax

### HTML Email Structure

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$subject}</title>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: #007bff; color: white; padding: 20px; text-align: center; }
        .content { padding: 20px; background: #f9f9f9; }
        .footer { padding: 20px; text-align: center; font-size: 12px; color: #666; }
        .button { display: inline-block; padding: 10px 20px; background: #007bff; color: white; text-decoration: none; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>{$company_name}</h1>
        </div>
        <div class="content">
            <p>Hello {$client_first_name},</p>
            
            <p>{$message}</p>
            
            <p><a href="{$invoice_url}" class="button">View Invoice</a></p>
        </div>
        <div class="footer">
            <p>&copy; {date('Y')} {$company_name}. All rights reserved.</p>
            <p>{$system_url}</p>
        </div>
    </div>
</body>
</html>
```

### Plain Text Emails

```text
{$company_name}
{$system_url}

Hello {$client_first_name},

{$message}

{if $invoice_url}
View your invoice: {$invoice_url}
{/if}

--
{$company_name}
{$system_url}
```

## Invoice Email Template

### Example

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Invoice #{$invoice_num}</title>
</head>
<body style="font-family: Arial, sans-serif; margin: 0; padding: 20px;">
    
    <div style="max-width: 600px; margin: 0 auto;">
        <div style="text-align: center; padding: 20px; background: #4a90d9; color: white;">
            <h1>{$company_name}</h1>
            <h2>Invoice #{$invoice_num}</h2>
        </div>
        
        <div style="padding: 20px; background: #f5f5f5;">
            <table style="width: 100%;">
                <tr>
                    <td><strong>Invoice Date:</strong></td>
                    <td>{$invoice_date}</td>
                </tr>
                <tr>
                    <td><strong>Due Date:</strong></td>
                    <td>{$invoice_due_date}</td>
                </tr>
                <tr>
                    <td><strong>Amount Due:</strong></td>
                    <td style="font-size: 1.2em;"><strong>{$invoice_total}</strong></td>
                </tr>
            </table>
        </div>
        
        <div style="padding: 20px;">
            <p>Dear {$client_first_name},</p>
            
            <p>Thank you for your business. Please find your invoice details below.</p>
            
            <p style="text-align: center; margin: 30px 0;">
                <a href="{$invoice_url}" style="background: #4a90d9; color: white; padding: 15px 30px; text-decoration: none; border-radius: 5px;">
                    Pay Invoice Now
                </a>
            </p>
            
            <p>If you have any questions, please don't hesitate to contact us.</p>
            
            <p>Best regards,<br>{$company_name} Team</p>
        </div>
        
        <div style="text-align: center; padding: 20px; font-size: 12px; color: #666;">
            <p>&copy; {date('Y')} {$company_name}</p>
            <p>{$system_url}</p>
        </div>
    </div>
    
</body>
</html>
```

## Ticket Notification Template

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Support Ticket - {$ticket_subject}</title>
</head>
<body style="font-family: Arial, sans-serif; line-height: 1.6;">
    
    <div style="max-width: 600px; margin: 0 auto; border: 1px solid #ddd;">
        
        <div style="background: #333; color: white; padding: 20px;">
            <h2>{$company_name} Support</h2>
        </div>
        
        <div style="padding: 20px;">
            <p><strong>Ticket ID:</strong> {$ticket_tid}</p>
            <p><strong>Subject:</strong> {$ticket_subject}</p>
            <p><strong>Status:</strong> {$ticket_status}</p>
            
            <hr>
            
            <div style="background: #f9f9f9; padding: 15px; margin: 20px 0;">
                {$ticket_message|nl2br}
            </div>
            
            <p>
                <a href="{$ticket_url}" style="background: #007bff; color: white; padding: 10px 20px; text-decoration: none;">
                    View Ticket
                </a>
            </p>
        </div>
        
        <div style="background: #f5f5f5; padding: 15px; font-size: 12px; color: #666;">
            <p>Please do not reply directly to this email. Use the link above to respond.</p>
        </div>
        
    </div>
    
</body>
</html>
```

## Dynamic Content

### Conditional Content

```smarty
{if $client_company_name}
    <p>Company: {$client_company_name}</p>
{/if}

{if $service_domain}
    <p>Domain: {$service_domain}</p>
{/if}
```

### Loop Through Items

```smarty
{foreach $order_items as $item}
    <tr>
        <td>{$item.name}</td>
        <td>{$item.quantity}</td>
        <td>{$item.price}</td>
    </tr>
{/foreach}
```

## Email Settings

### Global Settings

| Setting | Location |
|---------|----------|
| Email Subject Prefix | Configuration > General Settings |
| Email Branding | Configuration > General Settings |
| SMTP Settings | Configuration > System > Mail |
| Email Logging | Configuration > System > Logs |

### Template Variables Available

```smarty
{$custom_subject}        {* Customizable subject *}
{$custom_message}        {* Customizable message *}
{$template_type}         {* html or plain *}
{$send_date}            {* Send timestamp *}
```

## Best Practices

1. **Use inline CSS** for email compatibility
2. **Keep subject lines clear** and descriptive
3. **Include plain text version** when possible
4. **Test across email clients** before sending
5. **Use responsive design** for mobile
6. **Avoid spam keywords** in content

## See Also

- [Email Templates Configuration](../whmcs-email-setup.md)
- [Template Variables](../whmcs-template-variables.md)
- [Invoice Template](../whmcs-invoice-template.md)