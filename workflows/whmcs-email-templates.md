# WHMCS Email Template Workflow

## Purpose

Guide to creating, customizing, and managing email templates in WHMCS for professional client communications, automation, and branding consistency.

## Prerequisites

- WHMCS installation with email access
- HTML/CSS knowledge for template design
- Understanding of WHMCS smarty variables
- SMTP configuration

## Workflow Steps

### Step 1: Email Template Structure

Understand WHMCS email template architecture:

```php
// Template directory structure
/*
templates/
├── email/
│   ├── header.html
│   ├── footer.html
│   ├── body.html
│   └── styles.css
├── email_mixed/
│   └── ... (alternative format)
└── ...
*/

// Template variables available in WHMCS
$templateVariables = [
    // Client variables
    'client_first_name' => 'John',
    'client_last_name' => 'Doe',
    'client_full_name' => 'John Doe',
    'client_email' => 'john@example.com',
    'client_company_name' => 'Acme Corp',
    'client_phone' => '+1-555-1234',
    
    // Service variables
    'service_id' => '123',
    'service_domain' => 'example.com',
    'service_username' => 'johndoe',
    'service_first_payment' => '9.99',
    'service_recurring_amount' => '9.99',
    'service_billing_cycle' => 'Monthly',
    'service_next_due_date' => '2026-06-15',
    
    // Domain variables
    'domain_id' => '456',
    'domain_name' => 'example.com',
    'domain_registration_date' => '2025-06-15',
    'domain_expiry_date' => '2026-06-15',
    'domain_registrar' => 'GoDaddy',
    'domain_transfer_secret' => 'xxxx',
    
    // Invoice variables
    'invoice_id' => '789',
    'invoice_num' => 'INV-2026-00789',
    'invoice_amount' => '99.99',
    'invoice_balance' => '99.99',
    'invoice_date' => '2026-05-15',
    'invoice_due_date' => '2026-05-30',
    'invoice_items' => [], // Array of line items
    
    // Ticket variables
    'ticket_id' => 'TKT-12345',
    'ticket_subject' => 'Support Request',
    'ticket_status' => 'Open',
    'ticket_priority' => 'Medium',
    'ticket_response' => '',
    
    // Order variables
    'order_id' => '1001',
    'order_date' => '2026-05-15',
    'order_total' => '149.97',
    'order_products' => [],
    
    // System variables
    'whmcs_url' => 'https://billing.example.com',
    'whmcs_name' => 'Company Billing',
    'whmcs_logo' => '/assets/img/logo.png',
    'date' => '2026-05-28',
    'time' => '14:30:00',
];
```

### Step 2: Custom Email Template Creation

Create branded email templates:

```twig
{# templates/email/header.html #}

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$subject}</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            font-size: 14px;
            line-height: 1.6;
            color: #1E293B;
            background-color: #F8FAFC;
        }
        
        .email-wrapper {
            max-width: 600px;
            margin: 0 auto;
            background-color: #FFFFFF;
        }
        
        .email-header {
            background: linear-gradient(135deg, #2563EB 0%, #1E40AF 100%);
            padding: 30px;
            text-align: center;
        }
        
        .email-header img {
            max-height: 50px;
            margin-bottom: 15px;
        }
        
        .email-header h1 {
            color: #FFFFFF;
            font-size: 24px;
            font-weight: 600;
        }
        
        .email-body {
            padding: 30px;
        }
        
        .email-footer {
            background-color: #F1F5F9;
            padding: 20px 30px;
            text-align: center;
            font-size: 12px;
            color: #64748B;
            border-top: 3px solid #2563EB;
        }
        
        .btn {
            display: inline-block;
            padding: 12px 24px;
            background-color: #2563EB;
            color: #FFFFFF;
            text-decoration: none;
            border-radius: 6px;
            font-weight: 600;
        }
        
        .btn:hover {
            background-color: #1E40AF;
        }
        
        .alert {
            padding: 15px;
            border-radius: 6px;
            margin: 20px 0;
        }
        
        .alert-success {
            background-color: #D1FAE5;
            border: 1px solid #10B981;
            color: #065F46;
        }
        
        .alert-warning {
            background-color: #FEF3C7;
            border: 1px solid #F59E0B;
            color: #92400E;
        }
        
        .invoice-table {
            width: 100%;
            border-collapse: collapse;
            margin: 20px 0;
        }
        
        .invoice-table th,
        .invoice-table td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #E2E8F0;
        }
        
        .invoice-table th {
            background-color: #F8FAFC;
            font-weight: 600;
        }
        
        .invoice-total {
            text-align: right;
            font-size: 18px;
            font-weight: 600;
            color: #2563EB;
            margin-top: 15px;
        }
        
        @media only screen and (max-width: 480px) {
            .email-body {
                padding: 20px;
            }
            
            .btn {
                display: block;
                text-align: center;
            }
        }
    </style>
</head>
<body>
    <div class="email-wrapper">
        <div class="email-header">
            <img src="{$whmcs_logo}" alt="{$whmcs_name}">
            <h1>{$email_heading}</h1>
        </div>
        <div class="email-body">
```

```twig
{# templates/email/footer.html #}

        </div>
        <div class="email-footer">
            <p style="margin-bottom: 10px;">
                <strong>{$company_name}</strong><br>
                {$company_address}
            </p>
            <p style="margin-bottom: 10px;">
                <a href="{$whmcs_url}" style="color: #2563EB;">Website</a> |
                <a href="{$whmcs_url}/privacy.php" style="color: #2563EB;">Privacy Policy</a> |
                <a href="{$whmcs_url}/terms.php" style="color: #2563EB;">Terms of Service</a>
            </p>
            <p>
                You received this email because you have an account with {$whmcs_name}.<br>
                {$date}
            </p>
        </div>
    </div>
</body>
</html>
```

### Step 3: Automated Email Hooks

Create custom email triggers:

```php
// modules/addons/custom_emails/custom_emails.php

/**
 * Trigger custom email on specific events
 */
add_hook('AfterModuleCreate', 1, function($vars) {
    $service = Capsule::table('tblhosting')
        ->where('id', $vars['service_id'])
        ->first();
    
    $client = Capsule::table('tblclients')
        ->where('id', $service->userid)
        ->first();
    
    send_email('service_welcome', [
        'id' => $service->id,
        'client_id' => $client->id,
    ], [
        'service_domain' => $service->domain,
        'service_username' => $service->username,
        'service_password' => $vars['password'] ?? '',
        'service_ip' => $vars['server_ip'] ?? '',
    ]);
});

/**
 * Send renewal reminder emails
 */
add_hook('DailyCronJob', 1, function($vars) {
    $daysBeforeExpiry = [30, 14, 7, 3, 1];
    
    foreach ($daysBeforeExpiry as $days) {
        $expiryDate = date('Y-m-d', strtotime("+{$days} days"));
        
        $domains = Capsule::table('tbldomains')
            ->whereRaw("DATE_FORMAT(expirydate, '%Y-%m-%d') = ?", [$expiryDate])
            ->where('domainstatus', 'Active')
            ->get();
        
        foreach ($domains as $domain) {
            $client = Capsule::table('tblclients')
                ->where('id', $domain->userid)
                ->first();
            
            send_email('domain_renewal_reminder', [
                'id' => $domain->id,
                'client_id' => $client->id,
            ], [
                'domain_name' => $domain->domain,
                'domain_expiry_date' => $domain->expirydate,
                'days_until_expiry' => $days,
            ]);
        }
    }
});

/**
 * Payment reminder sequence
 */
add_hook('InvoiceCreated', 1, function($vars) {
    $invoiceId = $vars['invoice_id'];
    
    // Schedule reminder emails
    $reminders = [
        ['days' => -3, 'template' => 'invoice_reminder_early'],
        ['days' => 0, 'template' => 'invoice_reminder_due'],
        ['days' => 3, 'template' => 'invoice_reminder_overdue'],
        ['days' => 7, 'template' => 'invoice_reminder_final'],
    ];
    
    foreach ($reminders as $reminder) {
        $sendDate = date('Y-m-d', strtotime($reminder['days'] . ' days'));
        
        Capsule::table('mod_email_queue')->insert([
            'invoice_id' => $invoiceId,
            'template' => $reminder['template'],
            'scheduled_date' => $sendDate,
            'status' => 'pending',
        ]);
    }
});
```

### Step 4: Email Template Management

Programmatic template management:

```php
// includes/functions/email_templates.php

class EmailTemplateManager
{
    /**
     * Create or update email template
     */
    public static function createTemplate(string $type, string $name, array $data): int
    {
        $existing = Capsule::table('tblemailtemplates')
            ->where('type', $type)
            ->where('name', $name)
            ->first();
        
        if ($existing) {
            Capsule::table('tblemailtemplates')
                ->where('id', $existing->id)
                ->update([
                    'subject' => $data['subject'],
                    'message' => $data['message'],
                    'fromname' => $data['fromname'] ?? '',
                    'fromemail' => $data['fromemail'] ?? '',
                    'disabled' => $data['disabled'] ?? 0,
                    'custom' => 1,
                ]);
            return $existing->id;
        }
        
        return Capsule::table('tblemailtemplates')->insertGetId([
            'type' => $type,
            'name' => $name,
            'subject' => $data['subject'],
            'message' => $data['message'],
            'fromname' => $data['fromname'] ?? '',
            'fromemail' => $data['fromemail'] ?? '',
            'disabled' => $data['disabled'] ?? 0,
            'custom' => 1,
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }
    
    /**
     * Duplicate existing template
     */
    public static function duplicateTemplate(int $sourceId, string $newName): int
    {
        $source = Capsule::table('tblemailtemplates')
            ->where('id', $sourceId)
            ->first();
        
        return self::createTemplate($source->type, $newName, [
            'subject' => $source->subject,
            'message' => $source->message,
            'fromname' => $source->fromname,
            'fromemail' => $source->fromemail,
        ]);
    }
    
    /**
     * Get template by name
     */
    public static function getTemplate(string $name): ?object
    {
        return Capsule::table('tblemailtemplates')
            ->where('name', $name)
            ->where('language', '')
            ->first();
    }
    
    /**
     * Render template with variables
     */
    public static function renderTemplate(string $name, array $variables = []): array
    {
        $template = self::getTemplate($name);
        
        if (!$template) {
            return ['error' => 'Template not found'];
        }
        
        // Assign variables
        $smarty = new WHMCS\Smarty\Frontend();
        foreach ($variables as $key => $value) {
            $smarty->assign($key, $value);
        }
        
        return [
            'subject' => $smarty->fetch('string:' . $template->subject),
            'message' => $smarty->fetch('string:' . $template->message),
        ];
    }
}

/**
 * Create custom templates for new installation
 */
function installCustomEmailTemplates(): void
{
    $templates = [
        'welcome_new_client' => [
            'subject' => 'Welcome to {$whmcs_name}!',
            'message' => file_get_contents(__DIR__ . '/templates/email/welcome_client.html'),
        ],
        'service_suspended' => [
            'subject' => 'Service Suspended - {$service_domain}',
            'message' => file_get_contents(__DIR__ . '/templates/email/service_suspended.html'),
        ],
        'invoice_paid_thankyou' => [
            'subject' => 'Payment Received - Invoice #{$invoice_num}',
            'message' => file_get_contents(__DIR__ . '/templates/email/invoice_paid.html'),
        ],
    ];
    
    foreach ($templates as $name => $data) {
        EmailTemplateManager::createTemplate('custom', $name, $data);
    }
}
```

### Step 5: HTML Email Best Practices

Implement email-safe HTML:

```php
/**
 * Email-safe HTML generator
 */
class EmailHTMLGenerator
{
    /**
     * Generate inline-styled HTML for email clients
     */
    public static function generateInvoiceEmail(array $data): string
    {
        $html = <<<'HTML'
        <!DOCTYPE html>
        <html>
        <head>
            <meta charset="UTF-8">
            <title>Invoice</title>
        </head>
        <body style="margin: 0; padding: 0; font-family: Arial, sans-serif;">
            <table width="100%" cellpadding="0" cellspacing="0" style="background-color: #f4f4f4;">
                <tr>
                    <td align="center" style="padding: 20px;">
                        <table width="600" cellpadding="0" cellspacing="0" style="background-color: #ffffff; border-radius: 8px; overflow: hidden;">
                            <!-- Header -->
                            <tr>
                                <td style="background-color: #2563EB; padding: 30px; text-align: center;">
                                    <h1 style="color: #ffffff; margin: 0; font-size: 24px;">
                                        INVOICE #{$invoice_num}
                                    </h1>
                                </td>
                            </tr>
                            
                            <!-- Body -->
                            <tr>
                                <td style="padding: 30px;">
                                    <p style="margin: 0 0 15px;">Invoice Date: {$invoice_date}</p>
                                    <p style="margin: 0 0 15px;">Due Date: {$invoice_due_date}</p>
                                    <p style="margin: 0 0 30px;">Amount Due: <strong style="color: #2563EB; font-size: 20px;">{$invoice_amount}</strong></p>
                                    
                                    <!-- Line Items -->
                                    <table width="100%" cellpadding="10" cellspacing="0" style="border: 1px solid #e5e7eb;">
                                        <tr style="background-color: #f9fafb;">
                                            <th style="text-align: left; border-bottom: 1px solid #e5e7eb;">Description</th>
                                            <th style="text-align: right; border-bottom: 1px solid #e5e7eb;">Amount</th>
                                        </tr>
                                        {$invoice_items_html}
                                    </table>
                                    
                                    <!-- Total -->
                                    <table width="100%" cellpadding="10" style="margin-top: 20px;">
                                        <tr>
                                            <td style="text-align: right;">
                                                <strong style="font-size: 18px;">Total: {$invoice_amount}</strong>
                                            </td>
                                        </tr>
                                    </table>
                                    
                                    <!-- Pay Button -->
                                    <div style="text-align: center; margin-top: 30px;">
                                        <a href="{$invoice_url}" style="display: inline-block; background-color: #2563EB; color: #ffffff; padding: 15px 40px; text-decoration: none; border-radius: 5px; font-weight: bold;">
                                            Pay Invoice
                                        </a>
                                    </div>
                                </td>
                            </tr>
                            
                            <!-- Footer -->
                            <tr>
                                <td style="background-color: #f9fafb; padding: 20px; text-align: center; font-size: 12px; color: #6b7280;">
                                    <p style="margin: 0 0 5px;">{$company_name}</p>
                                    <p style="margin: 0;">{$company_address}</p>
                                </td>
                            </tr>
                        </table>
                    </td>
                </tr>
            </table>
        </body>
        </html>
        HTML;
        
        return $html;
    }
    
    /**
     * Generate plain text version
     */
    public static function generatePlainText(array $data): string
    {
        return <<<TEXT
{$company_name}
Invoice #{$invoice_num}

Invoice Date: {$invoice_date}
Due Date: {$invoice_due_date}
Amount Due: {$invoice_amount}

Items:
{$invoice_items_text}

Total: {$invoice_amount}

Pay online: {$invoice_url}

{$company_name}
{$company_address}
TEXT;
    }
}
```

## Best Practices

1. **Mobile-first design** - Most emails are read on mobile
2. **Consistent branding** - Match website and email styles
3. **Plain text fallback** - Always include text version
4. **Test on major clients** - Gmail, Outlook, Apple Mail
5. **Optimize images** - Keep email size small
6. **Clear CTAs** - Make actions obvious
7. **Personalize content** - Use client data variables
8. **Monitor deliverability** - Check spam scores

## Common Pitfalls to Avoid

1. **Complex CSS** - Many email clients don't support it
2. **Large images** - Slow loading, blocked by default
3. **Missing alt text** - Images may not load
4. **No plain text version** - Breaks for some clients
5. **Spam trigger words** - Avoid excessive "FREE", "ACT NOW"
6. **Non-responsive design** - Breaks on mobile
7. **Hardcoded text** - Use template variables
8. **Forgetting unsubscribe** - Legal requirement
