# WHMCS Mail Class Reference

**Version:** 8.x | **Updated:** 2026-05-29
**Related Skills:** `whmcs-email-template-builder`, `whmcs-multilanguage-support`

---

## Overview

WHMCS provides a comprehensive mail system for sending emails to clients and admins. This includes built-in email templates, custom email sending, and support for SMTP and other mail methods.

---

## Email Class (`WHMCS\Mail`)

The main WHMCS Mail class provides a fluent interface for composing and sending emails.

### Basic Usage

```php
<?php
use WHMCS\Mail;
use WHMCS\Contracts\Models\Mail\EmailerInterface;

/**
 * Create and send email
 */
$mail = new Mail();
$mail->setRecipient('client@example.com', 'John Doe');
$mail->setSubject('Your Service is Active');
$mail->setBody('Thank you for your order. Your service is now active.');
$mail->send();
```

### With Template

```php
use WHMCS\Mail;

/**
 * Send email using a template
 */
$mail = new Mail();
$mail->setRecipient('client@example.com', 'John Doe');
$mail->useTemplate('ProductWelcome')
    ->addVariable('client_name', 'John Doe')
    ->addVariable('service_name', 'Cloud VPS')
    ->addVariable('login_url', 'https://example.com/clientarea.php');
$mail->send();
```

---

## Email Sending Function (`send_email`)

The primary function for sending emails from hooks and modules.

### Function Signature

```php
/**
 * Send email to a client
 *
 * @param array $variables Email template variables
 * @param int|string $clientId Client ID
 * @return bool
 */
send_email(array $variables, $clientId);
```

### Parameters

```php
// $variables array structure
$variables = [
    'id' => 123,                    // Template ID (int) or template name (string)
    'type' => 'product',            // Template type
    'customvars' => [               // Custom template variables
        'custom_variable_name' => 'value',
    ],
    'custom_subject' => 'Subject',  // Override template subject
    'custom_message' => 'Message',  // Override template body
    'attachments' => [              // File attachments
        '/path/to/file.pdf',
    ],
];
```

### Examples

```php
<?php
// Send product welcome email
send_email([
    'type' => 'product',
    'id' => $service->packageid,
    'customvars' => [
        'service_id' => $service->id,
        'service_domain' => $service->domain,
        'server_ip' => $service->dedicatedip,
    ],
], $clientId);

// Send invoice email
send_email([
    'type' => 'invoice',
    'id' => $invoiceId,
    'customvars' => [
        'invoice_number' => $invoice->invoicenum,
        'amount_due' => formatCurrency($invoice->total, $client->currency),
    ],
], $clientId);

// Send custom email
send_email([
    'type' => 'general',
    'custom_subject' => 'Important Notice',
    'custom_message' => 'This is a custom message.',
], $clientId);
```

---

## WHMCS Mailer Class

Advanced email sending with the Mailer factory.

### Mailer Factory

```php
<?php
use WHMCS\Mail\Mailer;

// Get mailer instance
$mailer = Mailer::factory('php');        // PHP mail
$mailer = Mailer::factory('smtp');       // SMTP
$mailer = Mailer::factory('sendmail');   // Sendmail

// Configure SMTP
$mailer = Mailer::factory('smtp', [
    'host' => 'smtp.example.com',
    'port' => 587,
    'username' => 'smtp@example.com',
    'password' => 'smtp_password',
    'encryption' => 'tls',
]);
```

### Sending with Mailer

```php
<?php
use WHMCS\Mail\Mailer;
use WHMCS\Mail\Email\EmailTemplatableInterface;
use WHMCS\Mail\Email\Traits\EmailTemplateTrait;

class CustomEmailer implements EmailTemplatableInterface {
    use EmailTemplateTrait;

    protected string $recipient;
    protected string $subject;
    protected string $body;
    protected array $attachments = [];

    public function __construct(string $recipient, string $subject, string $body) {
        $this->recipient = $recipient;
        $this->subject = $subject;
        $this->body = $body;
    }

    public function getEmailRecipient(): string {
        return $this->recipient;
    }

    public function getEmailSubject(): string {
        return $this->subject;
    }

    public function getEmailBody(): string {
        return $this->body;
    }

    public function getEmailAttachments(): array {
        return $this->attachments;
    }

    public function addAttachments(string $path): self {
        $this->attachments[] = $path;
        return $this;
    }
}

// Send using mailer
$emailer = new CustomEmailer(
    'client@example.com',
    'Custom Subject',
    '<html><body><p>Your custom message here.</p></body></html>'
);

$mailer = Mailer::factory('smtp');
$mailer->send($emailer);
```

---

## Email Templates

### WHMCS Built-in Templates

| Template Type | Description |
|--------------|-------------|
| `general` | General notifications |
| `product` | Product/service emails |
| `invoice` | Invoice emails |
| `support` | Ticket emails |
| `admin` | Admin notifications |
| `domain` | Domain emails |

### Custom Email Template Creation

```php
<?php
/**
 * Create custom email templates
 */
function createEmailTemplates(): void {
    $templates = [
        [
            'name' => 'Module Welcome Email',
            'subject' => 'Welcome to {{service_product_name}}',
            'body' => <<<'HTML'
Dear {{client_full_name}},

Your {{service_product_name}} service is now active!

Service Details:
- Hostname: {{service_domain}}
- Control Panel: {{module_link}}

Best regards,
{{company_name}}
HTML,
            'type' => 'product',
        ],
        [
            'name' => 'Module Usage Alert',
            'subject' => 'Usage Alert: {{service_domain}}',
            'body' => <<<'HTML'
Dear {{client_full_name}},

You are approaching your usage limit.

Current Usage: {{usage_percent}}%
Threshold: {{usage_threshold}}%

Please upgrade your plan to avoid service interruption.

Upgrade: {{service_link}}

Best regards,
{{company_name}}
HTML,
            'type' => 'product',
        ],
    ];

    foreach ($templates as $template) {
        // Check if template exists
        $existing = Capsule::table('tblemailtemplates')
            ->where('name', $template['name'])
            ->first();

        if (!$existing) {
            Capsule::table('tblemailtemplates')->insert([
                'name' => $template['name'],
                'subject' => $template['subject'],
                'body' => $template['body'],
                'type' => $template['type'],
                'active' => 1,
                'created_at' => date('Y-m-d H:i:s'),
            ]);
        }
    }
}
```

### Email Template Variables

```php
<?php
// Standard WHMCS template variables available:
// Client
$variables['client_first_name'];
$variables['client_last_name'];
$variables['client_full_name'];
$variables['client_email'];
$variables['client_id'];
$variables['client_company_name'];

// Service
$variables['service_id'];
$variables['service_product_name'];
$variables['service_domain'];
$variables['service_username'];
$variables['service_recurring_amount'];
$variables['service_next_due_date'];
$variables['module_link'];

// Invoice
$variables['invoice_number'];
$variables['invoice_amount'];
$variables['invoice_balance'];
$variables['invoice_due_date'];
$variables['invoice_url'];

// System
$variables['company_name'];
$variables['company_email'];
$variables['system_url'];
```

---

## HTML Email Sending

### Clean HTML Emails

```php
<?php
/**
 * Send HTML email
 */
function sendHtmlEmail(string $recipient, string $subject, string $htmlBody): bool {
    $html = <<<HTML
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$subject}</title>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: #667eea; color: white; padding: 20px; text-align: center; }
        .content { padding: 20px; background: #fff; }
        .footer { padding: 15px; text-align: center; color: #666; font-size: 12px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>{$subject}</h1>
        </div>
        <div class="content">
            {$htmlBody}
        </div>
        <div class="footer">
            <p>{$companyName}</p>
        </div>
    </div>
</body>
</html>
HTML;

    $mail = new Mail();
    $mail->setRecipient($recipient);
    $mail->setSubject($subject);
    $mail->setFromName('Support Team');
    $mail->setBody($html);
    $mail->setBodyType('html');
    $mail->send();

    return true;
}
```

---

## Email Attachments

### Adding Attachments

```php
<?php
use WHMCS\Mail\Mailer;

$mailer = Mailer::factory('smtp');

$emailer = new class {
    public string $recipient = 'client@example.com';
    public string $subject = 'Invoice Attached';
    public string $body = '<p>Please find your invoice attached.</p>';

    public function getEmailRecipient(): string {
        return $this->recipient;
    }

    public function getEmailSubject(): string {
        return $this->subject;
    }

    public function getEmailBody(): string {
        return $this->body;
    }

    public function getEmailAttachments(): array {
        return [
            '/path/to/invoice.pdf',
            '/path/to/other-document.pdf',
        ];
    }
};

$mailer->send($emailer);
```

### Dynamic Attachments from Query

```php
<?php
/**
 * Send email with attachments from database
 */
function sendEmailWithAttachments(int $userId, string $templateName, array $fileIds): bool {
    $client = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();

    $files = Capsule::table('mod_attachments')
        ->whereIn('id', $fileIds)
        ->get()
        ->pluck('file_path')
        ->toArray();

    $mailer = Mailer::factory('smtp');

    $emailer = new class($client->email, $templateName, $files) {
        private string $recipient;
        private string $subject;
        private array $files;

        public function __construct(string $recipient, string $subject, array $files) {
            $this->recipient = $recipient;
            $this->subject = $subject;
            $this->files = $files;
        }

        public function getEmailRecipient(): string {
            return $this->recipient;
        }

        public function getEmailSubject(): string {
            return $this->subject;
        }

        public function getEmailBody(): string {
            return '<p>Please find your file(s) attached.</p>';
        }

        public function getEmailAttachments(): array {
            return $this->files;
        }
    };

    return $mailer->send($emailer);
}
```

---

## Email Queue

### Bulk Email Queue

```php
<?php
/**
 * Queue email for async sending
 */
function queueEmail(int $userId, string $templateName, array $vars = [], array $attachments = []): void {
    Capsule::table('mod_email_queue')->insert([
        'user_id' => $userId,
        'template_name' => $templateName,
        'vars' => json_encode($vars),
        'attachments' => json_encode($attachments),
        'status' => 'pending',
        'priority' => 5,
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

/**
 * Process email queue
 */
function processEmailQueue(): void {
    $pending = Capsule::table('mod_email_queue')
        ->where('status', 'pending')
        ->orderBy('priority', 'desc')
        ->orderBy('created_at', 'asc')
        ->limit(50)
        ->get();

    foreach ($pending as $item) {
        try {
            send_email([
                'type' => 'general',
                'id' => $item->template_name,
                'customvars' => json_decode($item->vars, true),
                'attachments' => json_decode($item->attachments, true),
            ], $item->user_id);

            Capsule::table('mod_email_queue')
                ->where('id', $item->id)
                ->update([
                    'status' => 'sent',
                    'sent_at' => date('Y-m-d H:i:s'),
                ]);

        } catch (\Exception $e) {
            Capsule::table('mod_email_queue')
                ->where('id', $item->id)
                ->update([
                    'status' => 'failed',
                    'error' => $e->getMessage(),
                    'attempts' => $item->attempts + 1,
                ]);
        }
    }
}
```

### Cron Hook for Email Processing

```php
add_hook('HourlyCronJob', 1, function() {
    processEmailQueue();
});
```

---

## SMTP Configuration

### Custom SMTP Settings

```php
<?php
/**
 * Configure SMTP for module
 */
function configureModuleSmtp(): array {
    return [
        'host' => 'smtp.example.com',
        'port' => 587,
        'username' => get_config('module_smtp_user'),
        'password' => get_config('module_smtp_pass'),  // Should be encrypted
        'encryption' => 'tls',
        'from_address' => 'noreply@example.com',
        'from_name' => 'Module Name',
    ];
}

/**
 * Send via custom SMTP
 */
function sendViaCustomSmtp(string $to, string $subject, string $body): bool {
    $config = configureModuleSmtp();

    $mailer = Mailer::factory('smtp', [
        'host' => $config['host'],
        'port' => $config['port'],
        'username' => $config['username'],
        'password' => $config['password'],
        'encryption' => $config['encryption'],
    ]);

    $emailer = new class($to, $subject, $body, $config) {
        private string $recipient;
        private string $subject;
        private string $body;
        private array $config;

        public function __construct(string $recipient, string $subject, string $body, array $config) {
            $this->recipient = $recipient;
            $this->subject = $subject;
            $this->body = $body;
            $this->config = $config;
        }

        public function getEmailRecipient(): string {
            return $this->recipient;
        }

        public function getEmailSubject(): string {
            return $this->subject;
        }

        public function getEmailBody(): string {
            return $this->body;
        }

        public function getEmailAttachments(): array {
            return [];
        }

        public function getEmailSenderName(): string {
            return $this->config['from_name'];
        }
    };

    return $mailer->send($emailer);
}
```

---

## Best Practices

1. **Use templates** - Leverage WHMCS email templates for consistency
2. **Sanitize input** - Always validate email addresses
3. **Handle failures** - Implement retry logic for failed sends
4. **Track bounces** - Monitor bounced emails
5. **HTML + Plain text** - Send both formats for compatibility
6. **Unsubscribe links** - Include unsubscribe options for marketing
7. **Queue bulk emails** - Never send bulk emails synchronously
8. **Respect rate limits** - Implement rate limiting for SMTP

---

## Related Documentation

- [Email Template Builder Skill](../skills/whmcs-email-template-builder)
- [Cron Automation Guide](cron-job-reference.md)
- [Multilanguage Support](internationalization-guide.md)
