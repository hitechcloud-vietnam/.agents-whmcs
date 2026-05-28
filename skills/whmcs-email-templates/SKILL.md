# WHMCS Email Templates Skill
# Version: 1.0 | Updated: 2026-05-29

## Purpose

Guide for creating, customizing, and managing WHMCS email templates with Smarty templating, variable substitution, and responsive HTML design.

## When to Use

- Creating custom email templates
- Modifying existing WHMCS email templates
- Building HTML email campaigns
- Implementing multi-language emails
- Creating automated email workflows

## Email Template System Overview

### 1. WHMCS Email Template Structure

```php
<?php
// Template variables available in WHMCS emails
// Reference: https://developers.whmcs.com/advanced/email-templating/

$templateVariables = [
    // Client variables
    'client_name' => 'Full Name',
    'client_first_name' => 'First Name',
    'client_last_name' => 'Last Name',
    'client_email' => 'email@example.com',
    'client_id' => '123',

    // Service variables
    'service_id' => '789',
    'service_domain' => 'example.com',
    'service_hosting' => 'Hosting Account',
    'service_username' => 'username',

    // Invoice variables
    'invoice_id' => '456',
    'invoice_amount' => '29.99',
    'invoice_total' => '29.99',
    'invoice_balance' => '29.99',
    'invoice_date_due' => '2026-06-15',

    // Order variables
    'order_id' => '1001',
    'order_number' => 'ORD-1001',

    // System variables
    'whmcs_url' => 'https://whmcs.example.com',
    'whmcs_system_url' => 'https://whmcs.example.com',
    'company_name' => 'Your Company',
    'company_logo' => '<img src="...">',
    'date' => '2026-05-29',
];
```

## Custom Email Template Development

### 1. Creating Custom Email Template File

```php
<?php
// templates/email/custom_welcome.html

{*
Template Name: Custom Welcome Email
Description: Enhanced welcome email for new clients
Subject: Welcome to {$company_name}! Your account is ready
*}

{include file='email-header.tpl'}

<div style="padding: 30px 0;">
    <h1 style="color: #2c3e50; font-size: 24px; margin: 0 0 20px;">
        Welcome to {$company_name}, {$client_first_name}!
    </h1>

    <p style="color: #555; font-size: 16px; line-height: 1.6;">
        Thank you for joining us. Your account has been successfully created and
        is now ready to use.
    </p>

    <div style="background: #f8f9fa; border-left: 4px solid #3498db;
                padding: 20px; margin: 25px 0;">
        <h3 style="color: #2c3e50; margin: 0 0 15px;">Account Details</h3>
        <table style="width: 100%; border-collapse: collapse;">
            <tr>
                <td style="padding: 8px 0; color: #666;">Email:</td>
                <td style="padding: 8px 0; font-weight: bold;">{$client_email}</td>
            </tr>
            <tr>
                <td style="padding: 8px 0; color: #666;">Client ID:</td>
                <td style="padding: 8px 0; font-weight: bold;">{$client_id}</td>
            </tr>
            <tr>
                <td style="padding: 8px 0; color: #666;">Member Since:</td>
                <td style="padding: 8px 0; font-weight: bold;">{$date}</td>
            </tr>
        </table>
    </div>

    <p style="color: #555; font-size: 16px; line-height: 1.6;">
        To get started, log in to your client area using the link below:
    </p>

    <div style="text-align: center; margin: 30px 0;">
        <a href="{$whmcs_url}/clientarea.php"
           style="background: #3498db; color: #fff; padding: 15px 30px;
                  text-decoration: none; border-radius: 5px; font-weight: bold;">
            Access Your Account
        </a>
    </div>

    <p style="color: #555; font-size: 14px; line-height: 1.6;">
        If you have any questions, our support team is here to help.
    </p>
</div>

{include file='email-footer.tpl'}
```

### 2. Programmatic Email Sending

```php
<?php
// Send custom email using WHMCS mail class

use WHMCS\Mail\Message;
use WHMCS\Smarty\Mail\Manager as MailManager;

class CustomEmailService {
    /**
     * Send custom email with template
     */
    public function send(string $to, string $templateName, array $variables, array $options = []): bool {
        try {
            $mailer = \WHMCS\Mail::factory();

            // Set recipients
            $mailer->addRecipient($to);

            // Load and parse template
            $template = $this->getTemplate($templateName);
            $html = $this->parseTemplate($template, $variables);

            // Configure mail
            $mailer->Subject = $this->parseVariables($template['subject'], $variables);
            $mailer->Body = $html;
            $mailer->From = $options['from'] ?? \App::Config()->get('EmailFrom');
            $mailer->FromName = $options['from_name'] ?? \App::Config()->get('CompanyName');

            // Add attachments
            if (!empty($options['attachments'])) {
                foreach ($options['attachments'] as $attachment) {
                    $mailer->addAttachment($attachment['path'], $attachment['name'] ?? '');
                }
            }

            // Send
            $result = $mailer->send();

            logActivity("Custom email sent: {$templateName} to {$to}");
            return true;

        } catch (\Exception $e) {
            logActivity("Email send failed: " . $e->getMessage());
            return false;
        }
    }

    /**
     * Send email to client
     */
    public function sendToClient(int $clientId, string $templateName, array $variables = []): bool {
        $client = \WHMCS\Database\Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        if (!$client) {
            return false;
        }

        // Add client variables
        $variables = array_merge($variables, [
            'client_id' => $client->id,
            'client_first_name' => $client->firstname,
            'client_last_name' => $client->lastname,
            'client_name' => $client->firstname . ' ' . $client->lastname,
            'client_email' => $client->email,
        ]);

        return $this->send($client->email, $templateName, $variables);
    }

    /**
     * Send bulk email
     */
    public function sendBulk(array $recipients, string $templateName, array $variables = []): array {
        $results = [
            'sent' => 0,
            'failed' => 0,
            'errors' => [],
        ];

        foreach ($recipients as $recipient) {
            $recipientVariables = array_merge($variables, [
                'client_id' => $recipient['id'],
                'client_first_name' => $recipient['firstname'],
                'client_last_name' => $recipient['lastname'],
                'client_email' => $recipient['email'],
            ]);

            if ($this->send($recipient['email'], $templateName, $recipientVariables)) {
                $results['sent']++;
            } else {
                $results['failed']++;
                $results['errors'][] = $recipient['email'];
            }

            // Rate limiting
            usleep(100000); // 0.1 second delay
        }

        return $results;
    }

    private function getTemplate(string $templateName): ?array {
        return \WHMCS\Database\Capsule::table('tblemailtemplates')
            ->where('name', $templateName)
            ->first();
    }

    private function parseTemplate(string $template, array $variables): string {
        $smarty = \WHMCS\Utility\Smarty::factory();

        foreach ($variables as $key => $value) {
            $smarty->assign($key, $value);
        }

        // Add common variables
        $smarty->assign('whmcs_url', \App::getSystemURL());
        $smarty->assign('company_name', \App::Config()->get('CompanyName'));
        $smarty->assign('date', date('Y-m-d'));

        return $smarty->view($template);
    }

    private function parseVariables(string $text, array $variables): string {
        foreach ($variables as $key => $value) {
            $text = str_replace('{$' . $key . '}', (string) $value, $text);
        }
        return $text;
    }
}
```

### 3. Email Template Variables System

```php
<?php
// Custom template variables hook

add_hook('EmailPreSend', 1, function($vars) {
    // Add custom variables to email
    $customVars = [
        'custom_greeting' => getCustomGreeting($vars),
        'unsubscribe_url' => generateUnsubscribeUrl($vars),
        'email_preferences_url' => generatePreferencesUrl($vars),
        'client_loyalty_tier' => getClientLoyaltyTier($vars),
        'client_total_spent' => getClientTotalSpent($vars),
    ];

    // Merge with existing variables
    if (isset($vars['templateVariables'])) {
        $vars['templateVariables'] = array_merge($vars['templateVariables'], $customVars);
    }

    return $vars;
});

// Custom template function for Smarty
function smarty_function_custom_date_format($params, &$smarty) {
    $timestamp = $params['timestamp'] ?? time();
    $format = $params['format'] ?? 'Y-m-d';

    return date($format, is_numeric($timestamp) ? $timestamp : strtotime($timestamp));
}

function smarty_function_custom_money($params, &$smarty) {
    $amount = $params['amount'] ?? 0;
    $currency = $params['currency'] ?? 'USD';

    return formatMoney($amount, $currency);
}

function smarty_function_custom_truncate($params, &$smarty) {
    $text = $params['text'] ?? '';
    $length = $params['length'] ?? 100;
    $suffix = $params['suffix'] ?? '...';

    if (strlen($text) <= $length) {
        return $text;
    }

    return substr($text, 0, $length) . $suffix;
}
```

### 4. Multi-Language Email Templates

```php
<?php
// templates/email/localized_welcome.html

{*
Template Name: Localized Welcome Email
Description: Multi-language welcome email
Subject: {$subject}
*}

{if $language eq 'vi'}
    Xin chào {$client_first_name},

    Cảm ơn bạn đã đăng ký tại {$company_name}!

    Tài khoản của bạn đã được tạo thành công.

{elseif $language eq 'de'}
    Hallo {$client_first_name},

    Willkommen bei {$company_name}!

    Ihr Konto wurde erfolgreich erstellt.

{elseif $language eq 'fr'}
    Bonjour {$client_first_name},

    Bienvenue chez {$company_name}!

    Votre compte a ete cree avec succes.

{else}
    Hello {$client_first_name},

    Welcome to {$company_name}!

    Your account has been successfully created.
{/if}
```

### 5. Email Template Management Class

```php
<?php
// includes/classes/EmailTemplateManager.php

namespace MyModule\Email;

use WHMCS\Database\Capsule;

class EmailTemplateManager {
    /**
     * Create or update email template
     */
    public function createOrUpdate(string $name, array $data): void {
        $existing = Capsule::table('tblemailtemplates')
            ->where('name', $name)
            ->first();

        if ($existing) {
            Capsule::table('tblemailtemplates')
                ->where('name', $name)
                ->update([
                    'subject' => $data['subject'],
                    'message' => $data['message'],
                    'fromname' => $data['from_name'] ?? '',
                    'fromemail' => $data['from_email'] ?? '',
                    'disabled' => $data['disabled'] ?? 0,
                ]);
        } else {
            Capsule::table('tblemailtemplates')->insert([
                'name' => $name,
                'subject' => $data['subject'],
                'message' => $data['message'],
                'fromname' => $data['from_name'] ?? '',
                'fromemail' => $data['from_email'] ?? '',
                'disabled' => $data['disabled'] ?? 0,
                'language' => $data['language'] ?? '',
                'type' => $data['type'] ?? 'product',
                'custom' => 1,
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
    }

    /**
     * Install default templates for module
     */
    public function installModuleTemplates(string $moduleName): void {
        $templates = $this->getModuleTemplates($moduleName);

        foreach ($templates as $name => $data) {
            $this->createOrUpdate($name, $data);
        }
    }

    /**
     * Export templates
     */
    public function exportTemplates(array $names): array {
        $templates = Capsule::table('tblemailtemplates')
            ->whereIn('name', $names)
            ->get();

        $export = [];
        foreach ($templates as $template) {
            $export[$template->name] = [
                'subject' => $template->subject,
                'message' => $template->message,
                'from_name' => $template->fromname,
                'from_email' => $template->fromemail,
                'language' => $template->language,
            ];
        }

        return $export;
    }

    /**
     * Get available template variables
     */
    public function getTemplateVariables(string $type = 'all'): array {
        $variables = [
            // Client variables
            'client_id', 'client_first_name', 'client_last_name', 'client_name',
            'client_email', 'client_company_name', 'client_address1', 'client_address2',
            'client_city', 'client_state', 'client_postcode', 'client_country',
            'client_phone', 'client_password',

            // Service variables
            'service_id', 'service_domain', 'service_hosting', 'service_username',
            'service_password', 'service_regdate', 'service_nextduedate',
            'service_first_payment_amount', 'service_recurring_amount',

            // Invoice variables
            'invoice_id', 'invoice_number', 'invoice_amount', 'invoice_total',
            'invoice_balance', 'invoice_date', 'invoice_date_due', 'invoice_items',

            // Order variables
            'order_id', 'order_number', 'order_date', 'order_amount',

            // Domain variables
            'domain_id', 'domain_name', 'domain_registrar', 'domain_expiry_date',
            'domain_registration_date', 'domain_transfer_secret',

            // Ticket variables
            'ticket_id', 'ticket_ticket', 'ticket_subject', 'ticket_priority',
            'ticket_message', 'ticket_status',

            // System variables
            'whmcs_url', 'company_name', 'company_logo', 'date', 'time',
        ];

        return $variables;
    }

    private function getModuleTemplates(string $moduleName): array {
        // Return templates specific to module
        return [
            "{$moduleName}_welcome" => [
                'subject' => 'Welcome to ' . $moduleName,
                'message' => $this->getWelcomeTemplate(),
            ],
            "{$moduleName}_setup_complete" => [
                'subject' => 'Your ' . $moduleName . ' is ready',
                'message' => $this->getSetupCompleteTemplate(),
            ],
            "{$moduleName}_usage_warning" => [
                'subject' => 'Usage warning for your ' . $moduleName . ' service',
                'message' => $this->getUsageWarningTemplate(),
            ],
        ];
    }

    private function getWelcomeTemplate(): string {
        return '<p>Welcome to our service!</p>';
    }

    private function getSetupCompleteTemplate(): string {
        return '<p>Your setup is complete!</p>';
    }

    private function getUsageWarningTemplate(): string {
        return '<p>You are approaching your usage limit.</p>';
    }
}
```

## HTML Email Best Practices

```html
<!-- DO: Use inline styles for email compatibility -->
<div style="font-family: Arial, sans-serif; font-size: 16px; color: #333;">
    <h1 style="color: #2c3e50; margin: 0 0 20px;">Heading</h1>
</div>

<!-- DO: Use tables for layout -->
<table cellpadding="0" cellspacing="0" border="0" style="width: 100%;">
    <tr>
        <td style="padding: 20px;">Content here</td>
    </tr>
</table>

<!-- DO: Use web-safe fonts -->
<div style="font-family: Arial, Helvetica, sans-serif;">Text</div>

<!-- DO: Set explicit widths -->
<img src="logo.png" width="200" height="100" alt="Logo">

<!-- DON'T: Use CSS float -->
<div style="float: left;">Bad for email</div>

<!-- DON'T: Use complex positioning -->
<div style="position: absolute;">Not reliable</div>

<!-- DON'T: Use JavaScript -->
<script>// Won't work in email</script>
```

## Email Template Checklist

```html
<!-- Responsive email template structure -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Email Subject</title>
    <!--[if mso]>
    <style type="text/css">
        table {border-collapse: collapse;}
        .button {padding: 10px 20px;}
    </style>
    <![endif]-->
</head>
<body style="margin: 0; padding: 0; background-color: #f4f4f4;">
    <table role="presentation" cellpadding="0" cellspacing="0" width="100%"
           style="background-color: #f4f4f4;">
        <tr>
            <td align="center" style="padding: 40px 0;">
                <!-- Email container -->
                <table role="presentation" cellpadding="0" cellspacing="0"
                       width="600" style="background-color: #ffffff; border-radius: 8px;">
                    <!-- Header -->
                    <tr>
                        <td style="padding: 30px; text-align: center;">
                            <img src="{$company_logo}" alt="{$company_name}"
                                 width="150" style="max-width: 150px;">
                        </td>
                    </tr>
                    <!-- Content -->
                    <tr>
                        <td style="padding: 0 30px 30px;">
                            {$message_content}
                        </td>
                    </tr>
                    <!-- Footer -->
                    <tr>
                        <td style="padding: 30px; background-color: #f8f9fa;
                                    text-align: center; font-size: 12px; color: #666;">
                            <p>{$company_name}<br>
                            <a href="{$whmcs_url}">{$whmcs_url}</a></p>
                            <p style="margin-top: 20px;">
                                <a href="{$unsubscribe_url}">Unsubscribe</a>
                            </p>
                        </td>
                    </tr>
                </table>
            </td>
        </tr>
    </table>
</body>
</html>
```

## Checklist

- [ ] HTML email template follows web-safe practices
- [ ] Inline styles for maximum email client compatibility
- [ ] Responsive design for mobile devices
- [ ] Plain text alternative provided
- [ ] Proper image URLs (absolute paths)
- [ ] Unsubscribe link included
- [ ] Template variables documented
- [ ] Multi-language support considered

---

**Related Skills:**
- whmcs-email-template-builder
- whmcs-multilanguage-support
- whmcs-notification-development
- whmcs-clientarea-builder

**Reference:**
- WHMCS Email Templates: https://developers.whmcs.com/advanced/email-templating/