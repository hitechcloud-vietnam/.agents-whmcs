# WHMCS Email Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## Purpose
Create a custom email module for WHMCS that provides advanced SMTP configuration, email tracking, analytics, and template management.

## Module Type
Notification Provider Module

## Use Case
- Custom SMTP configuration for outgoing emails
- Email delivery tracking and analytics
- Email queue management
- Template customization
- Bounce handling
- SPF/DKIM verification

## DevKit Structure

```
devkits/whmcs-email-module/
├── provider.php             # Notification provider class
├── lib/
│   ├── SmtpConnector.php    # SMTP connection handler
│   ├── EmailTracker.php     # Email tracking analytics
│   ├── TemplateManager.php  # Email template management
│   └── BounceHandler.php    # Bounce email processing
├── templates/               # Email templates
│   └── notification.tpl    # Notification template
└── DEVKIT.md              # This file
```

## Main Provider Template

```php
<?php
namespace WHMCS\Module\Notification\{Email};

use WHMCS\Module\Contracts\NotificationModuleInterface;
use WHMCS\Notification\Contracts\NotificationInterface;
use WHMCS\Module\Notification\DescriptionTrait;

if (!defined("WHMCS")) {
    die("Direct access denied");
}

class Provider implements NotificationModuleInterface {
    use DescriptionTrait;

    protected $smtpConnector;
    protected $emailTracker;
    protected $templateManager;

    public function __construct() {
        $this->smtpConnector = new SmtpConnector();
        $this->emailTracker = new EmailTracker();
        $this->templateManager = new TemplateManager();
    }

    public static function moduleConfiguration(): array {
        return [
            [
                'Name' => 'smtp_host',
                'Type' => 'text',
                'FriendlyName' => 'SMTP Host',
                'Description' => 'SMTP server hostname',
            ],
            [
                'Name' => 'smtp_port',
                'Type' => 'text',
                'FriendlyName' => 'SMTP Port',
                'Default' => '587',
            ],
            [
                'Name' => 'smtp_username',
                'Type' => 'text',
                'FriendlyName' => 'SMTP Username',
            ],
            [
                'Name' => 'smtp_password',
                'Type' => 'password',
                'FriendlyName' => 'SMTP Password',
            ],
            [
                'Name' => 'smtp_encryption',
                'Type' => 'dropdown',
                'Options' => 'none,tls,ssl',
                'FriendlyName' => 'Encryption',
                'Default' => 'tls',
            ],
            [
                'Name' => 'from_email',
                'Type' => 'text',
                'FriendlyName' => 'From Email Address',
            ],
            [
                'Name' => 'from_name',
                'Type' => 'text',
                'FriendlyName' => 'From Name',
            ],
            [
                'Name' => 'enable_tracking',
                'Type' => 'yesno',
                'FriendlyName' => 'Enable Email Tracking',
            ],
            [
                'Name' => 'tracking_domain',
                'Type' => 'text',
                'FriendlyName' => 'Tracking Domain',
                'Description' => 'Domain for tracking pixels and links',
            ],
        ];
    }

    public function testConnection(): void {
        $settings = $this->getSettings();

        try {
            $this->smtpConnector->connect([
                'host' => $settings['smtp_host'],
                'port' => (int)$settings['smtp_port'],
                'username' => $settings['smtp_username'],
                'password' => $settings['smtp_password'],
                'encryption' => $settings['smtp_encryption'] ?? 'tls',
            ]);

            $this->smtpConnector->disconnect();
        } catch (\Exception $e) {
            throw new \Exception('SMTP Connection Failed: ' . $e->getMessage());
        }
    }

    public function notificationSettings(): array {
        return [
            [
                'Name' => 'default_template',
                'Type' => 'dropdown',
                'Options' => 'default,minimal,detailed',
                'FriendlyName' => 'Default Template',
            ],
            [
                'Name' => 'include_logo',
                'Type' => 'yesno',
                'FriendlyName' => 'Include Company Logo',
            ],
            [
                'Name' => 'primary_color',
                'Type' => 'text',
                'FriendlyName' => 'Primary Color',
                'Default' => '#007bff',
            ],
        ];
    }

    public function send(NotificationInterface $notification, array $settings): void {
        $moduleSettings = $this->getSettings();

        $emailData = [
            'to' => $this->getRecipientEmail($notification),
            'to_name' => $this->getRecipientName($notification),
            'from_email' => $moduleSettings['from_email'] ?? 'noreply@example.com',
            'from_name' => $moduleSettings['from_name'] ?? 'WHMCS',
            'subject' => $notification->getSubject(),
            'body' => $this->buildEmailBody($notification, $settings),
            'headers' => $this->buildHeaders($notification, $moduleSettings),
        ];

        try {
            $this->smtpConnector->connect([
                'host' => $moduleSettings['smtp_host'],
                'port' => (int)$moduleSettings['smtp_port'],
                'username' => $moduleSettings['smtp_username'],
                'password' => $moduleSettings['smtp_password'],
                'encryption' => $moduleSettings['smtp_encryption'] ?? 'tls',
            ]);

            $result = $this->smtpConnector->send($emailData);
            $this->smtpConnector->disconnect();

            if ($moduleSettings['enable_tracking'] ?? false) {
                $this->emailTracker->logEmail([
                    'message_id' => $result['message_id'] ?? uniqid('email_'),
                    'to_email' => $emailData['to'],
                    'subject' => $emailData['subject'],
                    'status' => 'sent',
                    'sent_at' => date('Y-m-d H:i:s'),
                ]);
            }
        } catch (\Exception $e) {
            if ($moduleSettings['enable_tracking'] ?? false) {
                $this->emailTracker->logEmail([
                    'message_id' => uniqid('email_'),
                    'to_email' => $emailData['to'],
                    'subject' => $emailData['subject'],
                    'status' => 'failed',
                    'error_message' => $e->getMessage(),
                    'sent_at' => date('Y-m-d H:i:s'),
                ]);
            }
            throw new \Exception('Failed to send email: ' . $e->getMessage());
        }
    }

    protected function getRecipientEmail(NotificationInterface $notification): string {
        $channels = $notification->getOutputChannels();
        return $channels['email'] ?? '';
    }

    protected function getRecipientName(NotificationInterface $notification): string {
        $name = $notification->getName();
        return $name ?: '';
    }

    protected function buildEmailBody(NotificationInterface $notification, array $settings): string {
        $template = $settings['default_template'] ?? 'default';
        $includeLogo = ($settings['include_logo'] ?? false) === 'on';
        $primaryColor = $settings['primary_color'] ?? '#007bff';

        $fields = $notification->getFields();
        $message = $notification->getMessage();
        $actionText = $notification->getActionText();
        $actionUrl = $notification->getActionUrl();

        ob_start();
        include __DIR__ . '/templates/notification.tpl';
        return ob_get_clean();
    }

    protected function buildHeaders(NotificationInterface $notification, array $settings): array {
        $messageId = uniqid('email_') . '@' . ($settings['tracking_domain'] ?? 'whmcs.local');

        return [
            'Message-ID' => '<' . $messageId . '>',
            'X-Mailer' => 'WHMCS-EmailModule/1.0',
            'MIME-Version' => '1.0',
            'Content-Type' => 'text/html; charset=UTF-8',
        ];
    }

    protected function getSettings(): array {
        $module = \WHMCS\Database\Capsule::table('tbladdon_modules')
            ->where('module', '{email}')
            ->first();

        return $module ? json_decode($module->value, true) : [];
    }
}
```

## SMTP Connector Class

```php
<?php
namespace WHMCS\Module\Notification\{Email};

class SmtpConnector {

    protected $socket;
    protected $host;
    protected $port;
    protected $username;
    protected $password;
    protected $encryption;

    public function connect(array $config): void {
        $this->host = $config['host'];
        $this->port = $config['port'];
        $this->username = $config['username'];
        $this->password = $config['password'];
        $this->encryption = $config['encryption'] ?? 'tls';

        $protocol = ($this->encryption === 'ssl') ? 'ssl' : 'tcp';
        $this->socket = @fsockopen(
            $protocol . '://' . $this->host,
            $this->port,
            $errno,
            $errstr,
            30
        );

        if (!$this->socket) {
            throw new \Exception("Cannot connect to SMTP: $errstr ($errno)");
        }

        $this->readResponse();
        $this->sendCommand("EHLO " . gethostname());
        $this->readResponse();

        if ($this->encryption === 'tls') {
            $this->sendCommand("STARTTLS");
            $this->readResponse();
            stream_socket_enable_crypto($this->socket, true, STREAM_CRYPTO_METHOD_TLS_CLIENT);

            $this->sendCommand("EHLO " . gethostname());
            $this->readResponse();
        }

        $this->sendCommand("AUTH LOGIN");
        $this->readResponse();
        $this->sendCommand(base64_encode($this->username));
        $this->readResponse();
        $this->sendCommand(base64_encode($this->password));
        $this->readResponse();
    }

    public function send(array $emailData): array {
        $messageId = $emailData['headers']['Message-ID'] ?? uniqid('email_') . '@local';

        $this->sendCommand("MAIL FROM:<" . $emailData['from_email'] . ">");
        $this->readResponse();

        $this->sendCommand("RCPT TO:<" . $emailData['to'] . ">");
        $this->readResponse();

        $this->sendCommand("DATA");
        $this->readResponse();

        $headers = [];
        foreach ($emailData['headers'] as $name => $value) {
            $headers[] = "$name: $value";
        }
        $headers[] = 'From: ' . $emailData['from_name'] . ' <' . $emailData['from_email'] . '>';
        $headers[] = 'To: ' . $emailData['to_name'] . ' <' . $emailData['to'] . '>';
        $headers[] = 'Subject: ' . $emailData['subject'];
        $headers[] = '';
        $headers[] = $emailData['body'];
        $headers[] = '.';
        $headers[] = '';

        fwrite($this->socket, implode("\r\n", $headers));

        $this->readResponse();

        return ['message_id' => $messageId, 'status' => 'sent'];
    }

    public function disconnect(): void {
        if ($this->socket) {
            $this->sendCommand("QUIT");
            fclose($this->socket);
            $this->socket = null;
        }
    }

    protected function sendCommand(string $command): void {
        fwrite($this->socket, $command . "\r\n");
    }

    protected function readResponse(): string {
        $response = '';
        while ($line = fgets($this->socket, 515)) {
            $response .= $line;
            if (substr($line, 3, 1) === ' ') break;
        }
        return $response;
    }
}
```

## Email Tracker Class

```php
<?php
namespace WHMCS\Module\Notification\{Email};

use WHMCS\Database\Capsule;

class EmailTracker {

    public function logEmail(array $data): void {
        Capsule::table('mod_{email}_email_logs')->insert([
            'message_id' => $data['message_id'],
            'to_email' => $data['to_email'],
            'subject' => $data['subject'],
            'status' => $data['status'],
            'error_message' => $data['error_message'] ?? null,
            'sent_at' => $data['sent_at'],
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    public function markOpened(string $messageId): void {
        Capsule::table('mod_{email}_email_logs')
            ->where('message_id', $messageId)
            ->update([
                'opened' => true,
                'opened_at' => date('Y-m-d H:i:s'),
            ]);
    }

    public function markClicked(string $messageId, string $url): void {
        Capsule::table('mod_{email}_click_logs')->insert([
            'message_id' => $messageId,
            'url' => $url,
            'clicked_at' => date('Y-m-d H:i:s'),
        ]);

        Capsule::table('mod_{email}_email_logs')
            ->where('message_id', $messageId)
            ->increment('click_count');
    }

    public function markBounced(string $messageId, string $reason): void {
        Capsule::table('mod_{email}_email_logs')
            ->where('message_id', $messageId)
            ->update([
                'bounced' => true,
                'bounce_reason' => $reason,
                'bounced_at' => date('Y-m-d H:i:s'),
            ]);
    }

    public function markComplained(string $messageId): void {
        Capsule::table('mod_{email}_email_logs')
            ->where('message_id', $messageId)
            ->update([
                'complained' => true,
                'complained_at' => date('Y-m-d H:i:s'),
            ]);
    }

    public function getStats(): array {
        $total = Capsule::table('mod_{email}_email_logs')->count();
        $sent = Capsule::table('mod_{email}_email_logs')->where('status', 'sent')->count();
        $failed = Capsule::table('mod_{email}_email_logs')->where('status', 'failed')->count();
        $opened = Capsule::table('mod_{email}_email_logs')->where('opened', true)->count();
        $bounced = Capsule::table('mod_{email}_email_logs')->where('bounced', true)->count();

        return [
            'total' => $total,
            'sent' => $sent,
            'failed' => $failed,
            'opened' => $opened,
            'open_rate' => $sent > 0 ? round(($opened / $sent) * 100, 2) : 0,
            'bounced' => $bounced,
            'bounce_rate' => $sent > 0 ? round(($bounced / $sent) * 100, 2) : 0,
        ];
    }

    public function getEmailLogs(int $limit = 100, int $offset = 0): array {
        return Capsule::table('mod_{email}_email_logs')
            ->orderBy('created_at', 'desc')
            ->limit($limit)
            ->offset($offset)
            ->get()
            ->toArray();
    }
}
```

## Template Manager Class

```php
<?php
namespace WHMCS\Module\Notification\{Email};

use WHMCS\Database\Capsule;

class TemplateManager {

    public function getTemplate(string $name, string $type = 'default'): ?array {
        return Capsule::table('mod_{email}_templates')
            ->where('name', $name)
            ->where('type', $type)
            ->first();
    }

    public function saveTemplate(string $name, string $type, string $subject, string $body): void {
        Capsule::table('mod_{email}_templates')->updateOrInsert(
            ['name' => $name, 'type' => $type],
            [
                'subject' => $subject,
                'body' => $body,
                'updated_at' => date('Y-m-d H:i:s'),
            ]
        );
    }

    public function getAllTemplates(): array {
        return Capsule::table('mod_{email}_templates')
            ->orderBy('name')
            ->get()
            ->toArray();
    }

    public function renderTemplate(string $template, array $variables): string {
        $body = $template;

        foreach ($variables as $key => $value) {
            $body = str_replace('{{' . $key . '}}', $value, $body);
        }

        return $body;
    }

    public function createDefaultTemplates(): void {
        $defaultTemplates = [
            [
                'name' => 'default',
                'type' => 'default',
                'subject' => '{{subject}}',
                'body' => file_get_contents(__DIR__ . '/../templates/notification.tpl'),
            ],
        ];

        foreach ($defaultTemplates as $template) {
            $this->saveTemplate(
                $template['name'],
                $template['type'],
                $template['subject'],
                $template['body']
            );
        }
    }
}
```

## Bounce Handler Class

```php
<?php
namespace WHMCS\Module\Notification\{Email};

use WHMCS\Database\Capsule;

class BounceHandler {

    protected $emailTracker;

    public function __construct() {
        $this->emailTracker = new EmailTracker();
    }

    public function processBounce(string $rawEmail): array {
        $bounceData = $this->parseBounceEmail($rawEmail);

        if (!isset($bounceData['original_message_id'])) {
            return ['success' => false, 'reason' => 'No message ID found'];
        }

        $messageId = $this->extractMessageId($bounceData['original_message_id']);
        $reason = $this->classifyBounce($bounceData);

        $this->emailTracker->markBounced($messageId, $reason);

        $this->logBounceEvent($bounceData, $reason);

        return [
            'success' => true,
            'message_id' => $messageId,
            'reason' => $reason,
            'bounce_type' => $bounceData['bounce_type'] ?? 'unknown',
        ];
    }

    protected function parseBounceEmail(string $rawEmail): array {
        $data = [];

        if (preg_match('/X-Failed-Recipients:\s*(.+)/i', $rawEmail, $matches)) {
            $data['failed_recipient'] = trim($matches[1]);
        }

        if (preg_match('/Original-Recipient:\s*(.+)/i', $rawEmail, $matches)) {
            $data['original_recipient'] = trim($matches[1]);
        }

        if (preg_match('/Message-ID:\s*<(.+)>/i', $rawEmail, $matches)) {
            $data['original_message_id'] = trim($matches[1]);
        }

        if (preg_match('/Diagnostic-Code:\s*(.+)/i', $rawEmail, $matches)) {
            $data['diagnostic_code'] = trim($matches[1]);
        }

        $data['bounce_type'] = $this->detectBounceType($rawEmail);

        return $data;
    }

    protected function classifyBounce(array $bounceData): string {
        $diagnostic = $bounceData['diagnostic_code'] ?? '';

        $hardBouncePatterns = [
            'User unknown',
            'No such user',
            'Invalid recipient',
            'Mailbox not found',
            'does not exist',
        ];

        foreach ($hardBouncePatterns as $pattern) {
            if (stripos($diagnostic, $pattern) !== false) {
                return 'hard_bounce';
            }
        }

        $softBouncePatterns = [
            'Mailbox full',
            'quota exceeded',
            'temporary failure',
            'try again later',
            'Connection timed out',
        ];

        foreach ($softBouncePatterns as $pattern) {
            if (stripos($diagnostic, $pattern) !== false) {
                return 'soft_bounce';
            }
        }

        return $bounceData['bounce_type'] ?? 'unknown';
    }

    protected function detectBounceType(string $rawEmail): string {
        if (stripos($rawEmail, 'Message-ID') === false &&
            stripos($rawEmail, 'feedback-type') !== false) {
            return 'complaint';
        }

        return 'bounce';
    }

    protected function extractMessageId(string $reference): string {
        if (preg_match('/<(.+@.+)>/', $reference, $matches)) {
            return $matches[1];
        }
        return $reference;
    }

    protected function logBounceEvent(array $bounceData, string $reason): void {
        Capsule::table('mod_{email}_bounce_events')->insert([
            'message_id' => $bounceData['original_message_id'] ?? null,
            'recipient' => $bounceData['failed_recipient'] ?? null,
            'reason' => $reason,
            'diagnostic_code' => $bounceData['diagnostic_code'] ?? null,
            'bounce_type' => $bounceData['bounce_type'] ?? 'unknown',
            'raw_data' => json_encode($bounceData),
            'processed_at' => date('Y-m-d H:i:s'),
        ]);
    }
}
```

## Email Template (notification.tpl)

```php
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><?php echo htmlspecialchars($emailData['subject'] ?? ''); ?></title>
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; color: #333; max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background: <?php echo htmlspecialchars($primaryColor); ?>; color: white; padding: 20px; text-align: center; }
        .header img { max-height: 50px; }
        .content { padding: 20px; background: #f9f9f9; }
        .message { background: white; padding: 20px; border-radius: 5px; margin: 15px 0; }
        .action-button { display: inline-block; background: <?php echo htmlspecialchars($primaryColor); ?>; color: white; padding: 12px 30px; text-decoration: none; border-radius: 5px; margin: 15px 0; }
        .fields { background: white; padding: 15px; margin: 10px 0; border-left: 3px solid <?php echo htmlspecialchars($primaryColor); ?>; }
        .footer { text-align: center; padding: 20px; color: #666; font-size: 12px; }
        .tracking-pixel { width: 1px; height: 1px; opacity: 0; position: absolute; }
    </style>
</head>
<body>
    <div class="header">
        <?php if ($includeLogo): ?>
            <img src="<?php echo $trackingDomain; ?>/logo.png" alt="Logo">
        <?php endif; ?>
        <h1><?php echo htmlspecialchars($emailData['subject'] ?? ''); ?></h1>
    </div>

    <div class="content">
        <?php if (!empty($fields)): ?>
            <div class="fields">
                <?php foreach ($fields as $field): ?>
                    <p><strong><?php echo htmlspecialchars($field['label'] ?? ''); ?>:</strong> <?php echo htmlspecialchars($field['value'] ?? ''); ?></p>
                <?php endforeach; ?>
            </div>
        <?php endif; ?>

        <?php if (!empty($message)): ?>
            <div class="message">
                <?php echo nl2br(htmlspecialchars($message)); ?>
            </div>
        <?php endif; ?>

        <?php if (!empty($actionText) && !empty($actionUrl)): ?>
            <div style="text-align: center;">
                <a href="<?php echo htmlspecialchars($actionUrl); ?>" class="action-button"><?php echo htmlspecialchars($actionText); ?></a>
            </div>
        <?php endif; ?>
    </div>

    <div class="footer">
        <p>You received this email because you have an account with us.</p>
        <p>&copy; <?php echo date('Y'); ?> Your Company Name. All rights reserved.</p>
    </div>

    <?php if (!empty($trackingPixel)): ?>
        <img src="<?php echo $trackingPixel; ?>" class="tracking-pixel" alt="">
    <?php endif; ?>
</body>
</html>
```

## Database Schema

### mod_{email}_email_logs
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| message_id | VARCHAR(255) | Unique message identifier |
| to_email | VARCHAR(255) | Recipient email |
| subject | VARCHAR(500) | Email subject |
| status | VARCHAR(20) | sent/failed |
| error_message | TEXT | Error details if failed |
| opened | BOOLEAN | Has been opened |
| opened_at | TIMESTAMP | When opened |
| clicked | BOOLEAN | Has been clicked |
| bounced | BOOLEAN | Has bounced |
| bounce_reason | VARCHAR(255) | Bounce reason |
| complained | BOOLEAN | Spam complaint |
| sent_at | TIMESTAMP | When sent |
| created_at | TIMESTAMP | Record creation |

### mod_{email}_click_logs
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| message_id | VARCHAR(255) | Parent message ID |
| url | TEXT | Clicked URL |
| clicked_at | TIMESTAMP | Click time |

### mod_{email}_templates
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| name | VARCHAR(100) | Template name |
| type | VARCHAR(50) | Template type |
| subject | VARCHAR(500) | Email subject template |
| body | TEXT | Email body template |
| updated_at | TIMESTAMP | Last update |

### mod_{email}_bounce_events
| Column | Type | Description |
|--------|------|-------------|
| id | INT AUTO_INCREMENT | Primary key |
| message_id | VARCHAR(255) | Related message |
| recipient | VARCHAR(255) | Failed recipient |
| reason | VARCHAR(100) | Bounce classification |
| diagnostic_code | TEXT | Raw bounce code |
| bounce_type | VARCHAR(50) | bounce/complaint |
| raw_data | TEXT | Full bounce data JSON |
| processed_at | TIMESTAMP | Processing time |

## Hooks Integration

```php
<?php
/**
 * WHMCS Email Module Hooks
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

// Track email opens via tracking pixel
add_hook('EmailPreSend', 1, function(array $vars) {
    $trackingId = bin2hex(random_bytes(16));
    return ['X-Tracking-ID' => $trackingId];
});

// Log all outgoing emails
add_hook('EmailSent', 1, function(array $vars) {
    Capsule::table('mod_{email}_email_logs')->insert([
        'message_id' => $vars['messageId'] ?? uniqid('email_'),
        'to_email' => $vars['mailto'] ?? '',
        'subject' => $vars['subject'] ?? '',
        'status' => 'sent',
        'sent_at' => date('Y-m-d H:i:s'),
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});

// Process bounce emails (for IMAP/POP3 inbox monitoring)
add_hook('DailyCronJob', 1, function(array $vars) {
    $bounceHandler = new \WHMCS\Module\Notification\{Email}\BounceHandler();

    $bounceEmails = fetchBounceEmailsFromInbox();
    foreach ($bounceEmails as $rawEmail) {
        $bounceHandler->processBounce($rawEmail);
    }
});

// Update stats on email activity
add_hook('EmailOpened', 1, function(array $vars) {
    Capsule::table('mod_{email}_email_logs')
        ->where('message_id', $vars['messageId'])
        ->update([
            'opened' => true,
            'opened_at' => date('Y-m-d H:i:s'),
        ]);
});
```

## Activation/Deactivation (Addon Module Wrapper)

If you need an admin interface, create an addon wrapper:

```php
<?php
/**
 * WHMCS Email Module Admin Wrapper
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function {email}_config(): array {
    return [
        'name' => '{Email Module}',
        'description' => 'Advanced email management with SMTP, tracking, and analytics',
        'version' => '1.0',
        'author' => '{Author Name}',
    ];
}

function {email}_activate(): array {
    try {
        // Email logs
        if (!Capsule::schema()->hasTable('mod_{email}_email_logs')) {
            Capsule::schema()->create('mod_{email}_email_logs', function($t) {
                $t->increments('id');
                $t->string('message_id', 255);
                $t->string('to_email', 255);
                $t->string('subject', 500);
                $t->string('status', 20)->default('sent');
                $t->text('error_message')->nullable();
                $t->boolean('opened')->default(false);
                $t->timestamp('opened_at')->nullable();
                $t->boolean('clicked')->default(false);
                $t->integer('click_count')->default(0);
                $t->boolean('bounced')->default(false);
                $t->string('bounce_reason', 255)->nullable();
                $t->boolean('complained')->default(false);
                $t->timestamp('sent_at');
                $t->timestamp('created_at');

                $t->index('message_id');
                $t->index('to_email');
                $t->index('status');
            });
        }

        // Click logs
        if (!Capsule::schema()->hasTable('mod_{email}_click_logs')) {
            Capsule::schema()->create('mod_{email}_click_logs', function($t) {
                $t->increments('id');
                $t->string('message_id', 255);
                $t->text('url');
                $t->timestamp('clicked_at');
            });
        }

        // Templates
        if (!Capsule::schema()->hasTable('mod_{email}_templates')) {
            Capsule::schema()->create('mod_{email}_templates', function($t) {
                $t->increments('id');
                $t->string('name', 100);
                $t->string('type', 50);
                $t->string('subject', 500);
                $t->text('body');
                $t->timestamp('updated_at');
            });
        }

        // Bounce events
        if (!Capsule::schema()->hasTable('mod_{email}_bounce_events')) {
            Capsule::schema()->create('mod_{email}_bounce_events', function($t) {
                $t->increments('id');
                $t->string('message_id', 255);
                $t->string('recipient', 255);
                $t->string('reason', 100);
                $t->text('diagnostic_code')->nullable();
                $t->string('bounce_type', 50);
                $t->text('raw_data')->nullable();
                $t->timestamp('processed_at');
            });
        }

        return ['status' => 'success', 'description' => '{Email Module} activated'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Activation failed: ' . $e->getMessage()];
    }
}

function {email}_deactivate(): array {
    try {
        Capsule::schema()->dropIfExists('mod_{email}_email_logs');
        Capsule::schema()->dropIfExists('mod_{email}_click_logs');
        Capsule::schema()->dropIfExists('mod_{email}_templates');
        Capsule::schema()->dropIfExists('mod_{email}_bounce_events');
        return ['status' => 'success'];
    } catch (\Exception $e) {
        return ['status' => 'error', 'description' => 'Deactivation failed'];
    }
}

function {email}_output(array $vars): void {
    $tab = $_REQUEST['tab'] ?? 'dashboard';

    echo '<div class="email-module">';
    echo '<h1><i class="fa fa-envelope"></i> Email Module</h1>';
    echo '<ul class="nav nav-tabs">';
    echo '<li class="' . ($tab === 'dashboard' ? 'active' : '') . '"><a href="?module={email}&tab=dashboard">Dashboard</a></li>';
    echo '<li class="' . ($tab === 'logs' ? 'active' : '') . '"><a href="?module={email}&tab=logs">Email Logs</a></li>';
    echo '<li class="' . ($tab === 'templates' ? 'active' : '') . '"><a href="?module={email}&tab=templates">Templates</a></li>';
    echo '<li class="' . ($tab === 'settings' ? 'active' : '') . '"><a href="?module={email}&tab=settings">Settings</a></li>';
    echo '</ul>';

    $tracker = new \WHMCS\Module\Notification\{Email}\EmailTracker();

    switch ($tab) {
        case 'dashboard':
            $stats = $tracker->getStats();
            echo '<div class="stats-grid">';
            echo '<div class="stat-box"><h3>' . $stats['sent'] . '</h3><p>Emails Sent</p></div>';
            echo '<div class="stat-box"><h3>' . $stats['open_rate'] . '%</h3><p>Open Rate</p></div>';
            echo '<div class="stat-box"><h3>' . $stats['bounce_rate'] . '%</h3><p>Bounce Rate</p></div>';
            echo '<div class="stat-box"><h3>' . $stats['failed'] . '</h3><p>Failed</p></div>';
            echo '</div>';
            break;

        case 'logs':
            $logs = $tracker->getEmailLogs(50);
            echo '<table class="data-table"><thead><tr><th>Date</th><th>To</th><th>Subject</th><th>Status</th><th>Opened</th></tr></thead><tbody>';
            foreach ($logs as $log) {
                echo '<tr>';
                echo '<td>' . $log->sent_at . '</td>';
                echo '<td>' . htmlspecialchars($log->to_email) . '</td>';
                echo '<td>' . htmlspecialchars($log->subject) . '</td>';
                echo '<td><span class="label label-' . ($log->status === 'sent' ? 'success' : 'danger') . '">' . $log->status . '</span></td>';
                echo '<td>' . ($log->opened ? '<i class="fa fa-check"></i>' : '<i class="fa fa-minus"></i>') . '</td>';
                echo '</tr>';
            }
            echo '</tbody></table>';
            break;
    }

    echo '</div>';
}
```

## Checklist

```
Pre-Dev:
□ Define SMTP providers to support
□ Plan tracking pixel implementation
□ Design bounce handling workflow
□ Choose template rendering approach

Development:
□ Implement Provider class with NotificationModuleInterface
□ Create SmtpConnector for email sending
□ Implement EmailTracker for analytics
□ Build TemplateManager for custom templates
□ Create BounceHandler for bounce processing
□ Design notification.tpl email template
□ Implement admin interface (addon wrapper)
□ Add hooks for email tracking

Security:
□ Validate SMTP credentials
□ Sanitize email content
□ Secure tracking pixels
□ Handle sensitive bounce data properly

Testing:
□ Test SMTP connection
□ Verify email delivery
□ Test tracking pixel
□ Test bounce handling
□ Verify email rendering
□ Test template customization
```
