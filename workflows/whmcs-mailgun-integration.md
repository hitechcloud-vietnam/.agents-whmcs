# WHMCS Mailgun Integration Workflow

## Overview
This workflow implements Mailgun email integration for WHMCS.

## Prerequisites
- WHMCS with Mailgun API access
- Mailgun account
- Admin access

## Step-by-Step Process

### Step 1: Mailgun Integration
```php
<?php
// /includes/email/MailgunIntegration.php

class MailgunIntegration {
    private $apiKey;
    private $domain;

    public function __construct()
    {
        $this->apiKey = getConfig('mailgun_api_key');
        $this->domain = getConfig('mailgun_domain');
    }

    /**
     * Send email
     */
    public function sendEmail(string $to, string $subject, string $html): array
    {
        $ch = curl_init("https://api.mailgun.net/v3/{$this->domain}/messages");

        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => [
                'from' => getConfig('from_email'),
                'to' => $to,
                'subject' => $subject,
                'html' => $html
            ],
            CURLOPT_USERPWD => 'api:' . $this->apiKey,
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

### Step 2: Mailgun Hooks
```php
<?php
// /includes/hooks/mailgun_hooks.php

$mailgun = new MailgunIntegration();

add_hook('InvoiceCreated', 1, function($vars) use ($mailgun) {
    $invoice = getInvoice($vars['invoiceid']);
    $client = getClientsDetails($invoice['userid']);

    $mailgun->sendEmail(
        $client['email'],
        "Invoice #{$vars['invoiceid']} Created",
        "Please find attached invoice #{$vars['invoiceid']}"
    );
});
```

## Related Workflows
- [WHMCS SendGrid Integration](./whmcs-sendgrid-integration.md)
- [WHMCS Email Automation](./whmcs-email-automation.md)