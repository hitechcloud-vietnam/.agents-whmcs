# WHMCS Email Template Builder Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for creating custom email templates and managing email communications.

## When to Use

- Building custom email templates
- Creating module-specific emails
- Managing transactional emails

## Email Template Patterns

### Custom Email Template Definition

```php
function {module}_createEmailTemplates(): void {
    $templates = [
        [
            'name' => '{Module} Service Activated',
            'subject' => 'Your {service_product_name} is now active',
            'body' => <<<'HTML'
Dear {$client_full_name},

Your {$service_product_name} service has been activated!

Service Details:
- Hostname: {$service_domain}
- Username: {$service_username}
- Control Panel: {$module_link}

You can access your control panel at: {$module_link}

Best regards,
{$company_name}
HTML,
            'type' => 'product',
        ],
        [
            'name' => '{Module} Usage Alert',
            'subject' => 'Usage Alert: {$service_domain}',
            'body' => <<<'HTML'
Dear {$client_full_name},

You are approaching your usage limit for {$service_product_name}.

Current Usage: {$usage_percent}%
Threshold: {$usage_threshold}%

Please upgrade your plan or reduce usage to avoid service interruption.

Upgrade here: {$service_link}

Best regards,
{$company_name}
HTML,
            'type' => 'product',
        ],
        [
            'name' => '{Module} Renewal Reminder',
            'subject' => 'Your {$service_product_name} renews in {$service_days_until_expiry} days',
            'body' => <<<'HTML'
Dear {$client_full_name},

Your {$service_product_name} service will renew in {$service_days_until_expiry} days.

Renewal Amount: {$service_recurring_amount}

To ensure uninterrupted service, please update your payment method if needed.

Renew now: {$service_link}

Best regards,
{$company_name}
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

### Send Custom Email

```php
private function sendEmail(int $userId, string $templateName, array $vars = []): bool {
    // Get client info
    $client = Capsule::table('tblclients')
        ->where('id', $userId)
        ->first();

    // Get email template
    $template = Capsule::table('tblemailtemplates')
        ->where('name', $templateName)
        ->first();

    if (!$template) {
        return false;
    }

    // Replace template variables
    $subject = $template->subject;
    $body = $template->body;

    // Standard WHMCS variables
    $replacements = [
        '{$client_full_name}' => $client->firstname . ' ' . $client->lastname,
        '{$client_first_name}' => $client->firstname,
        '{$client_email}' => $client->email,
        '{$company_name}' => \WHMCS\Config\Setting::getValue('CompanyName'),
    ];

    foreach ($vars as $key => $value) {
        $replacements['{$' . $key . '}'] = $value;
    }

    $subject = str_replace(array_keys($replacements), array_values($replacements), $subject);
    $body = str_replace(array_keys($replacements), array_values($replacements), $body);

    // Send email via WHMCS
    $mail = new \WHMCS\Mail();
    $mail->setRecipient($client->email, $client->firstname . ' ' . $client->lastname);
    $mail->setSubject($subject);
    $mail->setBody($body);
    $mail->send();

    return true;
}
```

### HTML Email Template

```smarty
<!---------------
{
    "name": "{Module} Welcome Email",
    "subject": "Welcome to {Module} - Get Started",
    "type": "product"
}
--------------->
<html>
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{$subject}</title>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; padding: 30px; text-align: center; }
        .content { padding: 30px; background: #fff; }
        .button { display: inline-block; padding: 12px 30px; background: #667eea; color: white; text-decoration: none; border-radius: 5px; }
        .footer { padding: 20px; text-align: center; color: #666; font-size: 12px; }
        .features { list-style: none; padding: 0; }
        .features li { padding: 10px 0; border-bottom: 1px solid #eee; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Welcome to {$service_product_name}</h1>
        </div>
        <div class="content">
            <p>Dear {$client_first_name},</p>
            <p>Your service is now ready! Here are your details:</p>
            <ul class="features">
                <li><strong>Hostname:</strong> {$service_domain}</li>
                <li><strong>Username:</strong> {$service_username}</li>
                <li><strong>Password:</strong> {$service_password}</li>
            </ul>
            <p style="text-align: center; margin: 30px 0;">
                <a href="{$module_link}" class="button">Access Your Panel</a>
            </p>
        </div>
        <div class="footer">
            <p>{$company_name}</p>
            <p>This email was sent to {$client_email}</p>
        </div>
    </div>
</body>
</html>
```

### Email with Attachment

```php
private function sendEmailWithAttachment(int $userId, string $templateName, string $filePath): bool {
    $client = Capsule::table('tblclients')->where('id', $userId)->first();
    $template = Capsule::table('tblemailtemplates')->where('name', $templateName)->first();

    if (!$template || !$client) {
        return false;
    }

    $mailer = \WHMCS\Mail\Mailer::factory('smtp');
    $mailer->setRecipient($client->email);
    $mailer->setSubject($template->subject);
    $mailer->setBody($this->parseTemplate($template->body, $client));
    $mailer->addAttachment($filePath);
    $mailer->send();

    return true;
}
```

### Email Queue for Bulk Sending

```php
function {module}_queueEmail(int $userId, string $templateName, array $vars = []): void {
    Capsule::table('mod_email_queue')->insert([
        'user_id' => $userId,
        'template_name' => $templateName,
        'vars' => json_encode($vars),
        'status' => 'pending',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
}

function {module}_processEmailQueue(): void {
    $pending = Capsule::table('mod_email_queue')
        ->where('status', 'pending')
        ->limit(100)
        ->get();

    foreach ($pending as $item) {
        try {
            $this->sendEmail($item->user_id, $item->template_name, json_decode($item->vars, true));

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
                ]);
        }
    }
}
```

---

**Related Skills:**
- whmcs-cron-automation
- whmcs-addon-builder
- whmcs-multilanguage-support
