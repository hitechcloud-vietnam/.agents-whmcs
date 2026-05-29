# WHMCS Email Functions

Complete reference for email management and sending functions in WHMCS.

## Overview

WHMCS provides comprehensive email management including sending emails, managing templates, and handling email queues.

## Core Email Functions

### sendEmail()

Sends an email using WHMCS mail system.

```php
/**
 * Send an email
 * 
 * @param string $template Email template name
 * @param array $variables Template variables
 * @param int $clientId Client ID
 * @param int $languageId Language ID
 * @return bool Success status
 */
function sendEmail(
    string $template,
    array $variables,
    int $clientId = 0,
    int $languageId = 0
): bool {
    $mail = new WHMCS\Mail\Mail();
    
    $mail->setTemplate($template);
    $mail->addRecipient('client', $clientId);
    $mail->addVars($variables);
    
    if ($languageId) {
        $mail->setLanguage($languageId);
    }
    
    return $mail->send();
}
```

**Example:**
```php
sendEmail('Service Welcome Email', [
    'service_id' => 456,
    'domain' => 'example.com',
    'username' => 'user123',
    'password' => 'TempPass123'
], $clientId);
```

### sendRawEmail()

Sends a raw email without template.

```php
/**
 * Send raw email
 * 
 * @param array $data Email data
 * @return bool Success status
 */
function sendRawEmail(array $data): bool
{
    $mail = new WHMCS\Mail\Mail();
    
    $mail->Subject = $data['subject'];
    $mail->Body = $data['body'];
    $mail->isHTML($data['is_html'] ?? false);
    
    if (!empty($data['to_email'])) {
        $mail->addAddress($data['to_email'], $data['to_name'] ?? '');
    }
    
    if (!empty($data['from_email'])) {
        $mail->setFrom($data['from_email'], $data['from_name'] ?? '');
    }
    
    if (!empty($data['cc'])) {
        foreach ($data['cc'] as $cc) {
            $mail->addCC($cc['email'], $cc['name'] ?? '');
        }
    }
    
    if (!empty($data['bcc'])) {
        foreach ($data['bcc'] as $bcc) {
            $mail->addBCC($bcc['email'], $bcc['name'] ?? '');
        }
    }
    
    return $mail->send();
}
```

**Example:**
```php
sendRawEmail([
    'to_email' => 'client@example.com',
    'to_name' => 'John Doe',
    'subject' => 'Custom Subject',
    'body' => '<h1>Hello!</h1><p>Your message here.</p>',
    'is_html' => true,
    'from_email' => 'support@example.com',
    'from_name' => 'Support Team'
]);
```

## Email Template Functions

### getEmailTemplate()

Retrieves an email template.

```php
/**
 * Get email template by name
 * 
 * @param string $templateName Template name
 * @param int $languageId Language ID
 * @return array|null Template data
 */
function getEmailTemplate(string $templateName, int $languageId = 0): ?array
{
    $query = Capsule::table('tblemailtemplates')
        ->where('name', $templateName);
    
    if ($languageId > 0) {
        $query->where('language', $languageId);
    } else {
        $query->where('language', '');
    }
    
    $result = $query->first();
    
    return $result ? (array) $result : null;
}
```

### createEmailTemplate()

Creates a new email template.

```php
/**
 * Create email template
 * 
 * @param array $data Template data
 * @return int Template ID
 */
function createEmailTemplate(array $data): int
{
    return Capsule::table('tblemailtemplates')->insertGetId([
        'type' => $data['type'] ?? 'product',
        'name' => $data['name'],
        'subject' => $data['subject'],
        'message' => $data['message'],
        'attachments' => $data['attachments'] ?? '',
        'disabled' => $data['disabled'] ?? 0,
        'language' => $data['language'] ?? '',
        'custom' => 1,
    ]);
}
```

**Example:**
```php
createEmailTemplate([
    'type' => 'product',
    'name' => 'Custom Welcome Email',
    'subject' => 'Welcome to our service!',
    'message' => '<p>Hello {$client_name},</p><p>Welcome!</p>',
    'language' => 'english'
]);
```

### updateEmailTemplate()

Updates an email template.

```php
/**
 * Update email template
 * 
 * @param string $templateName Template name
 * @param array $data Updated data
 * @param int $languageId Language ID
 * @return bool Success status
 */
function updateEmailTemplate(
    string $templateName,
    array $data,
    int $languageId = 0
): bool {
    $query = Capsule::table('tblemailtemplates')
        ->where('name', $templateName);
    
    if ($languageId > 0) {
        $query->where('language', $languageId);
    }
    
    return $query->update($data) > 0;
}
```

### deleteEmailTemplate()

Deletes an email template.

```php
/**
 * Delete email template
 * 
 * @param string $templateName Template name
 * @param int $languageId Language ID
 * @return bool Success status
 */
function deleteEmailTemplate(string $templateName, int $languageId = 0): bool
{
    $query = Capsule::table('tblemailtemplates')
        ->where('name', $templateName)
        ->where('custom', 1); // Only delete custom templates
    
    if ($languageId > 0) {
        $query->where('language', $languageId);
    }
    
    return $query->delete() > 0;
}
```

### getEmailTemplates()

Retrieves all email templates.

```php
/**
 * Get all email templates
 * 
 * @param string $type Filter by type
 * @return array Templates
 */
function getEmailTemplates(string $type = ''): array
{
    $query = Capsule::table('tblemailtemplates')
        ->orderBy('type', 'asc')
        ->orderBy('name', 'asc');
    
    if ($type) {
        $query->where('type', $type);
    }
    
    return $query->get()->toArray();
}
```

## Email Queue Functions

### queueEmail()

Adds an email to the queue for later sending.

```php
/**
 * Queue an email for sending
 * 
 * @param array $data Email data
 * @param string $schedule Scheduled time
 * @return int Queue entry ID
 */
function queueEmail(array $data, string $schedule = ''): int
{
    return Capsule::table('tblmailqueue')->insertGetId([
        'user_id' => $data['userid'] ?? 0,
        'send_from' => $data['from_email'] ?? '',
        'send_from_name' => $data['from_name'] ?? '',
        'send_to' => $data['to_email'] ?? '',
        'send_to_name' => $data['to_name'] ?? '',
        'cc' => $data['cc'] ?? '',
        'subject' => $data['subject'],
        'message' => $data['message'],
        'html' => $data['is_html'] ?? 0,
        'attachment' => $data['attachments'] ?? '',
        'date' => date('Y-m-d H:i:s'),
        'scheduled_at' => $schedule ?: null,
        'priority' => $data['priority'] ?? 0,
    ]);
}
```

**Example:**
```php
// Schedule email for later
queueEmail([
    'userid' => 123,
    'to_email' => 'client@example.com',
    'subject' => 'Scheduled Maintenance Notice',
    'message' => 'Maintenance scheduled for...',
    'is_html' => 1
], date('Y-m-d H:i:s', strtotime('+1 hour')));
```

### processEmailQueue()

Processes queued emails.

```php
/**
 * Process email queue
 * 
 * @param int $limit Maximum emails to process
 * @return array Processing results
 */
function processEmailQueue(int $limit = 100): array
{
    $pending = Capsule::table('tblmailqueue')
        ->whereRaw('(scheduled_at IS NULL OR scheduled_at <= ?)', [date('Y-m-d H:i:s')])
        ->where('sent', 0)
        ->orderBy('priority', 'desc')
        ->orderBy('date', 'asc')
        ->limit($limit)
        ->get();
    
    $sent = 0;
    $failed = 0;
    
    foreach ($pending as $email) {
        $result = sendRawEmail([
            'to_email' => $email->send_to,
            'to_name' => $email->send_to_name,
            'from_email' => $email->send_from,
            'from_name' => $email->send_from_name,
            'subject' => $email->subject,
            'body' => $email->message,
            'is_html' => $email->html,
            'cc' => $email->cc ? explode(',', $email->cc) : [],
            'attachments' => $email->attachment
        ]);
        
        if ($result) {
            Capsule::table('tblmailqueue')
                ->where('id', $email->id)
                ->update(['sent' => 1, 'sent_at' => date('Y-m-d H:i:s')]);
            $sent++;
        } else {
            Capsule::table('tblmailqueue')
                ->where('id', $email->id)
                ->increment('attempts');
            $failed++;
        }
    }
    
    return ['sent' => $sent, 'failed' => $failed];
}
```

## Email Attachment Functions

### attachFile()

Adds attachment to email.

```php
/**
 * Add attachment to email
 * 
 * @param string $templateName Template name
 * @param string $filePath File path
 * @return bool Success status
 */
function attachFile(string $templateName, string $filePath): bool
{
    $template = getEmailTemplate($templateName);
    
    if (!$template) {
        return false;
    }
    
    $attachments = $template['attachments'] ?? '';
    
    if ($attachments) {
        $attachments .= "\n" . $filePath;
    } else {
        $attachments = $filePath;
    }
    
    return updateEmailTemplate($templateName, ['attachments' => $attachments]);
}
```

## Email Variable Functions

### getEmailVariables()

Gets available variables for email templates.

```php
/**
 * Get email template variables
 * 
 * @param string $type Template type
 * @return array Variables
 */
function getEmailVariables(string $type): array
{
    $commonVariables = [
        'client_name' => 'Client Full Name',
        'client_first_name' => 'Client First Name',
        'client_last_name' => 'Client Last Name',
        'client_email' => 'Client Email',
        'client_company_name' => 'Client Company Name',
        'signature' => 'Email Signature',
        'date' => 'Current Date',
        'time' => 'Current Time',
    ];
    
    $typeVariables = [
        'invoice' => [
            'invoice_id' => 'Invoice ID',
            'invoice_number' => 'Invoice Number',
            'invoice_amount' => 'Invoice Amount',
            'invoice_date' => 'Invoice Date',
            'invoice_due_date' => 'Invoice Due Date',
            'invoice_items' => 'Invoice Items',
        ],
        'support' => [
            'ticket_id' => 'Ticket ID',
            'ticket_mask' => 'Ticket Mask ID',
            'ticket_subject' => 'Ticket Subject',
            'ticket_priority' => 'Ticket Priority',
            'ticket_status' => 'Ticket Status',
        ],
        'product' => [
            'product_name' => 'Product Name',
            'domain' => 'Domain',
            'username' => 'Service Username',
            'password' => 'Service Password',
        ],
        'domain' => [
            'domain_name' => 'Domain Name',
            'domain_registrar' => 'Domain Registrar',
            'registration_date' => 'Registration Date',
            'expiry_date' => 'Expiry Date',
        ]
    ];
    
    return array_merge($commonVariables, $typeVariables[$type] ?? []);
}
```

### parseEmailVariables()

Parses variables in email content.

```php
/**
 * Parse variables in email content
 * 
 * @param string $content Email content
 * @param array $variables Variables to replace
 * @return string Parsed content
 */
function parseEmailVariables(string $content, array $variables): string
{
    foreach ($variables as $key => $value) {
        $content = str_replace('{$' . $key . '}', $value, $content);
    }
    
    return $content;
}
```

## Email Sending Examples

### Send Invoice Email

```php
/**
 * Send invoice email
 * 
 * @param int $invoiceId Invoice ID
 * @param string $template Template name
 * @return bool Success status
 */
function sendInvoiceEmail(int $invoiceId, string $template = 'Invoice Created'): bool
{
    $invoice = getInvoice($invoiceId);
    $client = getClient($invoice['userid']);
    
    $variables = [
        'client_name' => "{$client['firstname']} {$client['lastname']}",
        'client_first_name' => $client['firstname'],
        'client_email' => $client['email'],
        'invoice_id' => $invoiceId,
        'invoice_number' => $invoice['invoicenum'],
        'invoice_amount' => formatCurrency($invoice['total'], $client['currency']),
        'invoice_date' => fromMySQLDate($invoice['date']),
        'invoice_due_date' => fromMySQLDate($invoice['duedate']),
        'invoice_items' => formatInvoiceItems(getInvoiceItems($invoiceId)),
    ];
    
    return sendEmail($template, $variables, $client['id']);
}
```

### Send Ticket Reply Email

```php
/**
 * Send ticket reply notification
 * 
 * @param int $ticketId Ticket ID
 * @param string $message Reply message
 * @param bool $isAdmin Is admin reply
 * @return bool Success status
 */
function sendTicketReplyEmail(int $ticketId, string $message, bool $isAdmin = true): bool
{
    $ticket = getTicket($ticketId);
    $client = getClient($ticket['userid']);
    
    $variables = [
        'ticket_id' => $ticketId,
        'ticket_mask' => $ticket['tid'],
        'ticket_subject' => $ticket['subject'],
        'ticket_priority' => $ticket['priority'],
        'ticket_status' => $ticket['status'],
        'reply_message' => $message,
        'reply_date' => date('Y-m-d H:i:s'),
    ];
    
    $template = $isAdmin ? 'Support Ticket Reply' : 'Support Ticket Reply From Customer';
    
    return sendEmail($template, $variables, $client['id']);
}
```

### Send Service Welcome Email

```php
/**
 * Send service welcome email
 * 
 * @param int $serviceId Service ID
 * @return bool Success status
 */
function sendServiceWelcomeEmail(int $serviceId): bool
{
    $service = getService($serviceId);
    $client = getClient($service['userid']);
    $product = getProduct($service['packageid']);
    
    $variables = [
        'client_name' => "{$client['firstname']} {$client['lastname']}",
        'client_email' => $client['email'],
        'product_name' => $product['name'],
        'domain' => $service['domain'],
        'username' => $service['username'],
        'password' => decryptPassword($service['password']),
        'server_ip' => getServerIp($service['server']),
        'control_panel_url' => getServerUrl($service['server']),
    ];
    
    return sendEmail('Service Welcome Email', $variables, $client['id']);
}
```

## Email Statistics

```php
/**
 * Get email statistics
 * 
 * @param string $startDate Start date
 * @param string $endDate End date
 * @return array Statistics
 */
function getEmailStatistics(string $startDate, string $endDate): array
{
    $stats = Capsule::table('tblmailqueue')
        ->selectRaw("
            COUNT(*) as total,
            SUM(CASE WHEN sent = 1 THEN 1 ELSE 0 END) as sent,
            SUM(CASE WHEN sent = 0 AND attempts > 0 THEN 1 ELSE 0 END) as failed
        ")
        ->whereBetween('date', [$startDate, $endDate])
        ->first();
    
    return [
        'total' => $stats->total ?? 0,
        'sent' => $stats->sent ?? 0,
        'failed' => $stats->failed ?? 0,
        'success_rate' => $stats->total > 0 
            ? round(($stats->sent / $stats->total) * 100, 2) 
            : 0
    ];
}
```

## Best Practices

1. **Use templates** - Always use email templates for consistency
2. **Parse variables** - Use variable parsing for dynamic content
3. **Queue large batches** - Queue mass emails to avoid timeouts
4. **Track delivery** - Monitor email delivery rates
5. **Handle failures** - Implement retry logic for failed emails
6. **Sanitize content** - Always sanitize user-generated content

## Related Functions

- [whmcs-functions-utility.md](whmcs-functions-utility.md) - Utility functions
- [whmcs-integration-email.md](whmcs-integration-email.md) - Email integration
- [whmcs-module-mailprovider-api.md](whmcs-module-mailprovider-api.md) - Mail provider module API