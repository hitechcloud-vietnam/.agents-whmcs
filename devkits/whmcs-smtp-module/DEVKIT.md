# WHMCS SMTP Module DevKit
# Version: 1.0 | Updated: 2026-05-28

## DevKit Structure

```
devkits/whmcs-smtp-module/
├── smtp.php              # SMTP mailer class
├── lib/
│   ├── SmtpClient.php    # SMTP client
│   └── SmtpConfig.php    # Configuration handler
└── templates/
    └── admin-config.tpl  # Admin configuration
```

## SMTP Mailer Template

```php
<?php
/**
 * WHMCS SMTP Module: {Module}
 * DevKit Template
 * 
 * Installation: Configure in WHMCS Settings > Mail
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

/**
 * SMTP Configuration Class
 */
class {Module}SmtpConfig {
    
    private array $config;
    
    public function __construct() {
        $this->loadConfig();
    }
    
    private function loadConfig(): void {
        $settings = get_query_vals(
            'tbladdon_modules',
            'value',
            ['module' => '{module}_smtp']
        );
        
        $this->config = $settings ? json_decode($settings['value'], true) : [
            'smtp_host' => '',
            'smtp_port' => 587,
            'smtp_username' => '',
            'smtp_password' => '',
            'smtp_encryption' => 'tls',
            'from_name' => '',
            'from_email' => '',
        ];
    }
    
    public function getHost(): string {
        return $this->config['smtp_host'] ?? '';
    }
    
    public function getPort(): int {
        return (int) ($this->config['smtp_port'] ?? 587);
    }
    
    public function getUsername(): string {
        return $this->config['smtp_username'] ?? '';
    }
    
    public function getPassword(): string {
        return $this->config['smtp_password'] ?? '';
    }
    
    public function getEncryption(): string {
        return $this->config['smtp_encryption'] ?? 'tls';
    }
    
    public function getFromName(): string {
        return $this->config['from_name'] ?? '';
    }
    
    public function getFromEmail(): string {
        return $this->config['from_email'] ?? '';
    }
    
    public function isValid(): bool {
        return !empty($this->getHost()) && !empty($this->getUsername());
    }
}

/**
 * SMTP Client Class
 */
class {Module}SmtpClient {
    
    private string $host;
    private int $port;
    private string $username;
    private string $password;
    private string $encryption;
    private $socket;
    private array $responses = [];
    private bool $connected = false;
    
    public function __construct({Module}SmtpConfig $config) {
        $this->host = $config->getHost();
        $this->port = $config->getPort();
        $this->username = $config->getUsername();
        $this->password = $config->getPassword();
        $this->encryption = $config->getEncryption();
    }
    
    public function connect(): bool {
        $protocol = ($this->encryption === 'ssl') ? 'ssl' : 'tcp';
        $host = ($this->encryption === 'ssl') ? "ssl://{$this->host}" : $this->host;
        
        $this->socket = @fsockopen($host, $this->port, $errno, $errstr, 30);
        
        if (!$this->socket) {
            throw new \Exception("Connection failed: {$errstr} ({$errno})");
        }
        
        stream_set_timeout($this->socket, 30);
        
        $this->readResponse();
        
        if (!$this->sendCommand("EHLO " . gethostname())) {
            // Fallback to HELO
            $this->sendCommand("HELO " . gethostname());
        }
        
        // Handle encryption upgrade for TLS
        if ($this->encryption === 'tls') {
            $this->sendCommand("STARTTLS");
            if (!stream_socket_enable_crypto(
                $this->socket,
                true,
                STREAM_CRYPTO_METHOD_TLS_CLIENT
            )) {
                throw new \Exception("TLS negotiation failed");
            }
            
            // Re-send EHLO after STARTTLS
            $this->sendCommand("EHLO " . gethostname());
        }
        
        // Authenticate
        $this->authenticate();
        
        $this->connected = true;
        return true;
    }
    
    private function authenticate(): void {
        $this->sendCommand("AUTH LOGIN");
        $this->sendCommand(base64_encode($this->username));
        $this->sendCommand(base64_encode($this->password));
    }
    
    public function send(array $mail): bool {
        if (!$this->connected) {
            $this->connect();
        }
        
        try {
            // MAIL FROM
            $this->sendCommand("MAIL FROM: <{$mail['from_email']}>");
            
            // RCPT TO
            foreach ($mail['to'] as $recipient) {
                $this->sendCommand("RCPT TO: <{$recipient['email']}>");
            }
            
            // DATA
            $this->sendCommand("DATA");
            
            // Build message
            $message = $this->buildHeaders($mail);
            $message .= "\r\n{$mail['body']}\r\n";
            $message .= "\r\n.\r\n";
            
            fwrite($this->socket, $message);
            $this->readResponse();
            
            return true;
        } catch (\Exception $e) {
            logActivity("{Module} SMTP Error: " . $e->getMessage());
            return false;
        }
    }
    
    private function buildHeaders(array $mail): string {
        $headers = [];
        $headers[] = "From: {$mail['from_name']} <{$mail['from_email']}>";
        $headers[] = "MIME-Version: 1.0";
        $headers[] = "Content-Type: {$mail['content_type']}"; // text/html or text/plain
        $headers[] = "Date: " . date('r');
        $headers[] = "Message-ID: <" . md5(uniqid(time())) . "@" . gethostname() . ">";
        $headers[] = "Subject: {$mail['subject']}";
        
        if (!empty($mail['cc'])) {
            $headers[] = "Cc: " . implode(', ', array_map(function($c) {
                return $c['email'];
            }, $mail['cc']));
        }
        
        return implode("\r\n", $headers) . "\r\n\r\n";
    }
    
    private function sendCommand(string $command): bool {
        fwrite($this->socket, $command . "\r\n");
        $this->readResponse();
        
        $code = substr($this->responses[count($this->responses) - 1], 0, 3);
        return in_array($code, ['250', '220', '354', '334']);
    }
    
    private function readResponse(): void {
        do {
            $line = fgets($this->socket, 512);
            $this->responses[] = $line;
        } while ($line[3] !== ' ' && !feof($this->socket));
    }
    
    public function disconnect(): void {
        if ($this->socket) {
            $this->sendCommand("QUIT");
            fclose($this->socket);
            $this->connected = false;
        }
    }
    
    public function __destruct() {
        $this->disconnect();
    }
    
    public function testConnection(): array {
        try {
            $this->connect();
            $this->disconnect();
            return ['success' => true, 'message' => 'Connection successful'];
        } catch (\Exception $e) {
            return ['success' => false, 'message' => $e->getMessage()];
        }
    }
}

/**
 * WHMCS Mail Hook
 */
add_hook('MailSend', 1, function(array $vars) {
    $config = new {Module}SmtpConfig();
    
    if (!$config->isValid()) {
        return; // Let default handler take over
    }
    
    $mail = [
        'from_name' => $vars['fromname'] ?? $config->getFromName(),
        'from_email' => $vars['fromemail'] ?? $config->getFromEmail(),
        'to' => [['email' => $vars['email']]],
        'subject' => $vars['subject'],
        'body' => $vars['message'],
        'content_type' => $vars['html'] ? 'text/html' : 'text/plain',
    ];
    
    try {
        $client = new {Module}SmtpClient($config);
        $result = $client->send($mail);
        
        if ($result) {
            return ['abort' => true]; // Prevent default sending
        }
    } catch (\Exception $e) {
        logActivity("{Module} SMTP Error: " . $e->getMessage());
    }
});
```

## Admin Configuration Template

```smarty
<div class="smtp-config">
    <div class="alert alert-info">
        <i class="fa fa-info-circle"></i>
        Configure your SMTP server settings below. These settings will be used for all outgoing emails.
    </div>
    
    <form method="post" action="{$smarty.server.PHP_SELF}">
        <input type="hidden" name="module" value="{module}_smtp">
        <input type="hidden" name="action" value="save">
        <input type="hidden" name="csrf_token" value="{$csrf_token}">
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">SMTP Server</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label for="smtp_host">SMTP Host</label>
                    <input type="text" name="smtp_host" id="smtp_host" 
                           class="form-control" value="{$smtp_host}"
                           placeholder="smtp.example.com" required>
                </div>
                
                <div class="form-group">
                    <label for="smtp_port">SMTP Port</label>
                    <select name="smtp_port" id="smtp_port" class="form-control">
                        <option value="25" {if $smtp_port == 25}selected{/if}>25 (Standard)</option>
                        <option value="465" {if $smtp_port == 465}selected{/if}>465 (SSL)</option>
                        <option value="587" {if $smtp_port == 587}selected{/if}>587 (TLS)</option>
                        <option value="2525" {if $smtp_port == 2525}selected{/if}>2525 (Alternative)</option>
                    </select>
                </div>
                
                <div class="form-group">
                    <label for="smtp_encryption">Encryption</label>
                    <select name="smtp_encryption" id="smtp_encryption" class="form-control">
                        <option value="none" {if $smtp_encryption == 'none'}selected{/if}>None</option>
                        <option value="ssl" {if $smtp_encryption == 'ssl'}selected{/if}>SSL</option>
                        <option value="tls" {if $smtp_encryption == 'tls'}selected{/if}>TLS</option>
                    </select>
                </div>
            </div>
        </div>
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Authentication</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label for="smtp_username">Username</label>
                    <input type="text" name="smtp_username" id="smtp_username" 
                           class="form-control" value="{$smtp_username}"
                           placeholder="your@email.com" required>
                </div>
                
                <div class="form-group">
                    <label for="smtp_password">Password</label>
                    <input type="password" name="smtp_password" id="smtp_password" 
                           class="form-control" value="{$smtp_password}"
                           placeholder="Your password">
                </div>
            </div>
        </div>
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Sender Identity</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label for="from_name">From Name</label>
                    <input type="text" name="from_name" id="from_name" 
                           class="form-control" value="{$from_name}"
                           placeholder="Your Company Name">
                </div>
                
                <div class="form-group">
                    <label for="from_email">From Email</label>
                    <input type="email" name="from_email" id="from_email" 
                           class="form-control" value="{$from_email}"
                           placeholder="noreply@example.com">
                </div>
            </div>
        </div>
        
        <div class="panel panel-default">
            <div class="panel-heading">
                <h3 class="panel-title">Test Connection</h3>
            </div>
            <div class="panel-body">
                <div class="form-group">
                    <label for="test_email">Test Email Address</label>
                    <div class="input-group">
                        <input type="email" name="test_email" id="test_email" 
                               class="form-control" placeholder="test@example.com">
                        <span class="input-group-btn">
                            <button type="button" class="btn btn-info" id="test-connection">
                                <i class="fa fa-paper-plane"></i> Send Test
                            </button>
                        </span>
                    </div>
                </div>
                <div id="test-result" class="alert" style="display:none;"></div>
            </div>
        </div>
        
        <button type="submit" class="btn btn-primary">
            <i class="fa fa-save"></i> Save Configuration
        </button>
        <button type="button" class="btn btn-success" id="test-smtp">
            <i class="fa fa-check"></i> Test & Save
        </button>
    </form>
</div>

<script>
    $(document).ready(function() {
        $('#test-connection, #test-smtp').on('click', function() {
            var smtpHost = $('#smtp_host').val();
            var smtpPort = $('#smtp_port').val();
            var smtpUsername = $('#smtp_username').val();
            var smtpPassword = $('#smtp_password').val();
            var smtpEncryption = $('#smtp_encryption').val();
            var testEmail = $('#test_email').val();
            var saveAfterTest = $(this).attr('id') === 'test-smtp';
            
            $.post('{$base_url}ajax.php', {
                module: '{module}_smtp',
                action: 'test_connection',
                smtp_host: smtpHost,
                smtp_port: smtpPort,
                smtp_username: smtpUsername,
                smtp_password: smtpPassword,
                smtp_encryption: smtpEncryption,
                test_email: testEmail
            }, function(response) {
                var resultDiv = $('#test-result');
                resultDiv.show();
                
                if (response.success) {
                    resultDiv.removeClass('alert-danger').addClass('alert-success');
                    resultDiv.html('<i class="fa fa-check-circle"></i> ' + response.message);
                    
                    if (saveAfterTest) {
                        $('form').submit();
                    }
                } else {
                    resultDiv.removeClass('alert-success').addClass('alert-danger');
                    resultDiv.html('<i class="fa fa-exclamation-circle"></i> ' + response.message);
                }
            }, 'json');
        });
    });
</script>
```

## Checklist

```
Pre-Dev:
□ Get SMTP server details from provider
□ Determine encryption method (SSL/TLS/None)
□ Identify authentication requirements
□ Plan error handling
□ Design configuration storage

Development:
□ Create SmtpConfig class
□ Implement SmtpClient class
□ Add connect() method with socket
□ Implement SMTP authentication (AUTH LOGIN)
□ Add send() method for sending emails
□ Build MIME email headers
□ Add STARTTLS support
□ Add SSL/TLS support
□ Create MailSend hook
□ Create admin configuration template
□ Add test connection feature

Testing:
□ Test connection to SMTP server
□ Test authentication
□ Test sending plain text email
□ Test sending HTML email
□ Test with attachments
□ Test error handling
□ Verify emails arrive correctly
□ Test with different SMTP providers
```