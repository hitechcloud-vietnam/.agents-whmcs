# WHMCS Email Hosting Provisioning Module DevKit
# Version: 1.0 | Updated: 2026-05-29

## Module Overview

Provisions email hosting accounts including mailboxes, distribution lists, aliases, and email routing with popular email providers.

## DevKit Structure

```
devkits/whmcs-email-hosting/
├── email_hosting.php         # Main provisioning module
├── lib/
│   └── ApiClient.php        # Email provider API client
├── templates/
│   └── clientarea.tpl        # Client area template
└── DEVKIT.md                 # This file
```

## Module Code

```php
<?php
/**
 * WHMCS Email Hosting Provisioning Module
 */

if (!defined("WHMCS")) {
    die("Direct access denied");
}

function email_hosting_MetaData(): array {
    return [
        'DisplayName' => 'Email Hosting',
        'APIVersion' => '1.1',
        'RequiresServer' => true,
        'DefaultNonSSLPort' => 443,
        'Parameters' => ['server_username', 'server_password', 'server_access_hash'],
    ];
}

function email_hosting_ConfigOptions(array $params): array {
    return [
        'mailbox_quota' => [
            'Type' => 'dropdown',
            'Options' => '1gb,5gb,10gb,25gb,50gb,100gb',
            'Default' => '5gb',
            'Description' => 'Mailbox storage limit',
        ],
        'max_mailboxes' => [
            'Type' => 'text',
            'Default' => '5',
            'Description' => 'Maximum mailboxes',
        ],
        'enable_aliases' => [
            'Type' => 'yesno',
            'Description' => 'Enable aliases',
        ],
        'enable_lists' => [
            'Type' => 'yesno',
            'Description' => 'Enable distribution lists',
        ],
        'spam_filter' => [
            'Type' => 'dropdown',
            'Options' => 'none,basic,advanced,enterprise',
            'Default' => 'basic',
            'Description' => 'Spam filtering level',
        ],
        'enable_archive' => [
            'Type' => 'yesno',
            'Description' => 'Enable email archiving',
        ],
    ];
}

function email_hosting_CreateAccount(array $params): string {
    try {
        $api = new EmailHosting\ApiClient($params);
        
        $domain = $params['domain'] ?? preg_replace('/^[^@]+@/', '', $params['email']);
        
        // Create domain if not exists
        $api->createDomain($domain, [
            'max_mailboxes' => (int) $params['configoption2'],
            'enable_aliases' => ($params['configoption3'] === 'on'),
            'enable_lists' => ($params['configoption4'] === 'on'),
        ]);

        // Create primary mailbox
        $primaryEmail = $params['customfields']['Primary Email'] ?? 'postmaster@' . $domain;
        $mailboxData = [
            'email' => $primaryEmail,
            'quota' => convertToBytes($params['configoption1']),
            'password' => $params['password'] ?? generateSecurePassword(),
            'spam_filter' => $params['configoption5'],
            'enable_archive' => ($params['configoption6'] === 'on'),
        ];

        $result = $api->createMailbox($domain, $mailboxData);

        saveCustomFieldValue($params['serviceid'], 'Domain', $domain);
        saveCustomFieldValue($params['serviceid'], 'Primary Email', $primaryEmail);
        saveCustomFieldValue($params['serviceid'], 'IMAP Host', $api->getImapHost());
        saveCustomFieldValue($params['serviceid'], 'SMTP Host', $api->getSmtpHost());
        saveCustomFieldValue($params['serviceid'], 'Quota', $params['configoption1']);

        logActivity("EmailHosting: Created email hosting for domain {$domain}");
        return 'success';
        
    } catch (\Exception $e) {
        logActivity("EmailHosting CreateAccount Error: " . $e->getMessage());
        return 'Error: ' . $e->getMessage();
    }
}

function email_hosting_SuspendAccount(array $params): string {
    try {
        $domain = getCustomFieldValue($params['serviceid'], 'Domain');
        $api = new EmailHosting\ApiClient($params);
        $api->suspendDomain($domain);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function email_hosting_UnsuspendAccount(array $params): string {
    try {
        $domain = getCustomFieldValue($params['serviceid'], 'Domain');
        $api = new EmailHosting\ApiClient($params);
        $api->activateDomain($domain);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function email_hosting_TerminateAccount(array $params): string {
    try {
        $domain = getCustomFieldValue($params['serviceid'], 'Domain');
        $api = new EmailHosting\ApiClient($params);
        $api->deleteDomain($domain);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function email_hosting_ChangePassword(array $params): string {
    try {
        $primaryEmail = getCustomFieldValue($params['serviceid'], 'Primary Email');
        $api = new EmailHosting\ApiClient($params);
        $api->updateMailboxPassword($primaryEmail, $params['password']);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function email_hosting_ChangePackage(array $params): string {
    try {
        $domain = getCustomFieldValue($params['serviceid'], 'Domain');
        $api = new EmailHosting\ApiClient($params);
        $api->updateDomainSettings($domain, [
            'max_mailboxes' => (int) $params['configoption2'],
            'quota' => convertToBytes($params['configoption1']),
        ]);
        return 'success';
    } catch (\Exception $e) {
        return 'Error: ' . $e->getMessage();
    }
}

function email_hosting_TestConnection(array $params): array {
    try {
        $api = new EmailHosting\ApiClient($params);
        $api->listDomains();
        return ['success' => true, 'error' => ''];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function email_hosting_AdminServices(array $params): array {
    return [
        'Domain' => getCustomFieldValue($params['serviceid'], 'Domain') ?: 'N/A',
        'Primary Email' => getCustomFieldValue($params['serviceid'], 'Primary Email') ?: 'N/A',
        'Quota' => getCustomFieldValue($params['serviceid'], 'Quota') ?: 'N/A',
        'Spam Filter' => $params['configoption5'],
    ];
}

function email_hosting_ClientArea(array $params): array {
    $domain = getCustomFieldValue($params['serviceid'], 'Domain');
    $primaryEmail = getCustomFieldValue($params['serviceid'], 'Primary Email');
    $imapHost = getCustomFieldValue($params['serviceid'], 'IMAP Host');
    $smtpHost = getCustomFieldValue($params['serviceid'], 'SMTP Host');
    $quota = getCustomFieldValue($params['serviceid'], 'Quota');

    $usage = [];
    try {
        $api = new EmailHosting\ApiClient($params);
        $usage = $api->getMailboxUsage($primaryEmail);
    } catch (\Exception $e) {}

    return [
        'pagetitle' => 'Email Hosting - ' . $domain,
        'templatefile' => 'templates/clientarea',
        'vars' => [
            'domain' => $domain,
            'primary_email' => $primaryEmail,
            'imap_host' => $imapHost,
            'smtp_host' => $smtpHost,
            'quota' => $quota,
            'usage' => $usage,
            'status' => $params['status'],
        ],
    ];
}

function email_hosting_ClientAreaAllowedFunctions(): array {
    return [
        'CreateMailbox' => 'Create Mailbox',
        'ListAliases' => 'Manage Aliases',
        'CreateList' => 'Create Distribution List',
    ];
}

// Helper Functions
function convertToBytes(string $size): int {
    $unit = strtolower(substr($size, -2));
    $value = (int) $size;
    
    return match($unit) {
        'gb' => $value * 1024 * 1024 * 1024,
        'mb' => $value * 1024 * 1024,
        'kb' => $value * 1024,
        default => $value,
    };
}

function generateSecurePassword(int $length = 16): string {
    $chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*';
    return substr(str_shuffle(str_repeat($chars, ceil($length / strlen($chars)))), 0, $length);
}
```

## API Client: lib/ApiClient.php

```php
<?php
namespace EmailHosting;

class ApiClient {
    private string $baseUrl;
    private string $apiKey;
    private int $timeout = 30;

    public function __construct(array $params) {
        $this->baseUrl = rtrim($params['serverhost'] ?? '', '/');
        $this->apiKey = $params['serveraccesshash'] ?? '';
    }

    public function request(string $method, string $endpoint, array $data = []): array {
        $ch = curl_init();
        curl_setopt_array($ch, [
            CURLOPT_URL => $this->baseUrl . $endpoint,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT => $this->timeout,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($data));
        }

        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        $result = json_decode($response, true) ?? [];
        
        if ($httpCode >= 400) {
            throw new \Exception($result['message'] ?? "HTTP {$httpCode}");
        }

        return $result['data'] ?? $result;
    }

    public function getImapHost(): string {
        return 'imap.' . parse_url($this->baseUrl, PHP_URL_HOST);
    }

    public function getSmtpHost(): string {
        return 'smtp.' . parse_url($this->baseUrl, PHP_URL_HOST);
    }

    public function createDomain(string $domain, array $settings): array {
        return $this->request('POST', '/api/v1/domains', array_merge(['domain' => $domain], $settings));
    }

    public function listDomains(): array {
        return $this->request('GET', '/api/v1/domains');
    }

    public function suspendDomain(string $domain): array {
        return $this->request('POST', "/api/v1/domains/{$domain}/suspend");
    }

    public function activateDomain(string $domain): array {
        return $this->request('POST', "/api/v1/domains/{$domain}/activate");
    }

    public function deleteDomain(string $domain): array {
        return $this->request('DELETE', "/api/v1/domains/{$domain}");
    }

    public function createMailbox(string $domain, array $data): array {
        return $this->request('POST', "/api/v1/domains/{$domain}/mailboxes", $data);
    }

    public function getMailboxUsage(string $email): array {
        return $this->request('GET', "/api/v1/mailboxes/{$email}/usage");
    }

    public function updateMailboxPassword(string $email, string $password): array {
        return $this->request('PUT', "/api/v1/mailboxes/{$email}/password", ['password' => $password]);
    }

    public function updateDomainSettings(string $domain, array $settings): array {
        return $this->request('PUT', "/api/v1/domains/{$domain}", $settings);
    }
}
```

## Client Area Template

```smarty
<div class="email-hosting-client">
    <div class="email-header">
        <h2><i class="fa fa-envelope"></i> Email Hosting</h2>
        <span class="badge badge-{$status|lower}">{$status}</span>
    </div>

    <div class="email-config">
        <h4><i class="fa fa-cog"></i> Configuration</h4>
        <table class="config-table">
            <tr>
                <td><strong>Domain:</strong></td>
                <td>{$domain}</td>
            </tr>
            <tr>
                <td><strong>Primary Email:</strong></td>
                <td>{$primary_email}</td>
            </tr>
            <tr>
                <td><strong>IMAP Host:</strong></td>
                <td><code>{$imap_host}</code></td>
            </tr>
            <tr>
                <td><strong>SMTP Host:</strong></td>
                <td><code>{$smtp_host}</code></td>
            </tr>
            <tr>
                <td><strong>Port (IMAP):</strong></td>
                <td>993 (SSL)</td>
            </tr>
            <tr>
                <td><strong>Port (SMTP):</strong></td>
                <td>587 (STARTTLS)</td>
            </tr>
        </table>
    </div>

    {if $usage}
    <div class="email-usage">
        <h4><i class="fa fa-chart-pie"></i> Mailbox Usage</h4>
        <div class="usage-bar">
            <div class="usage-fill" style="width: {min(100, ($usage.used / $usage.quota) * 100)}%"></div>
        </div>
        <div class="usage-text">
            {$usage.used|bytes} of {$usage.quota|bytes} used
        </div>
    </div>
    {/if}
</div>

<style>
.email-hosting-client { padding: 20px; }
.email-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
.email-config { background: #f8f9fa; border-radius: 8px; padding: 15px; margin-bottom: 20px; }
.config-table { width: 100%; }
.config-table td { padding: 8px 0; }
.email-usage { background: #f8f9fa; border-radius: 8px; padding: 15px; }
.usage-bar { height: 20px; background: #e0e0e0; border-radius: 10px; overflow: hidden; margin-bottom: 10px; }
.usage-fill { height: 100%; background: linear-gradient(90deg, #28a745, #ffc107); transition: width 0.3s; }
</style>
```

## Required Custom Fields

| Field Name | Type | Description |
|------------|------|-------------|
| Domain | Text | Email domain |
| Primary Email | Text | Primary mailbox address |
| IMAP Host | Text | IMAP server hostname |
| SMTP Host | Text | SMTP server hostname |
| Quota | Text | Mailbox quota |
