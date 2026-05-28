# WHMCS Spam Filter Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build spam filtering modules for email security.

## Spam Filter Module Structure

```php
<?php
/**
 * Spam Filter Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Spam Filter',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'filter_level'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'FilterLevel' => [
            'Type' => 'dropdown',
            'Options' => 'relaxed,standard,aggressive',
            'Default' => 'standard',
        ],
        'Quarantine' => [
            'Type' => 'yesno',
            'Description' => 'Quarantine spam instead of deleting',
        ],
        'BayesianFilter' => [
            'Type' => 'yesno',
            'Description' => 'Enable Bayesian learning',
        ],
        'SPFCheck' => [
            'Type' => 'yesno',
            'Description' => 'Enable SPF verification',
        ],
        'DKIMCheck' => [
            'Type' => 'yesno',
            'Description' => 'Enable DKIM verification',
        ],
    ];
}
```

## Spam Filter Operations

```php
function {module}_CreateAccount(array $params): string {
    $domain = $params['domain'];

    $this->api->createDomainFilter([
        'domain' => $domain,
        'level' => $params['configoption1'],
        'quarantine' => $params['configoption2'] === 'on',
        'bayesian' => $params['configoption3'] === 'on',
        'spf' => $params['configoption4'] === 'on',
        'dkim' => $params['configoption5'] === 'on',
    ]);

    Capsule::table('mod_spam_filter')->insert([
        'service_id' => $params['serviceid'],
        'domain' => $domain,
        'filter_level' => $params['configoption1'],
        'quarantine_enabled' => $params['configoption2'] === 'on',
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $filter = Capsule::table('mod_spam_filter')
        ->where('service_id', $params['serviceid'])
        ->first();

    if ($filter) {
        $this->api->deleteDomainFilter($filter->domain);
        Capsule::table('mod_spam_filter')
            ->where('id', $filter->id)
            ->delete();
    }

    return 'success';
}
```

## User Preferences

```php
public function getUserPreferences(int $serviceId): array {
    return Capsule::table('mod_spam_filter_prefs')
        ->where('service_id', $serviceId)
        ->first() ?? [
            'filter_level' => 'standard',
            'quarantine_days' => 14,
            'whitelist' => [],
            'blacklist' => [],
            'notify_daily' => true,
        ];
}

public function updateUserPreferences(int $serviceId, array $prefs): bool {
    $existing = Capsule::table('mod_spam_filter_prefs')
        ->where('service_id', $serviceId)
        ->first();

    $data = [
        'filter_level' => $prefs['filter_level'] ?? 'standard',
        'quarantine_days' => (int) ($prefs['quarantine_days'] ?? 14),
        'whitelist' => json_encode($prefs['whitelist'] ?? []),
        'blacklist' => json_encode($prefs['blacklist'] ?? []),
        'notify_daily' => $prefs['notify_daily'] ?? true,
        'updated_at' => date('Y-m-d H:i:s'),
    ];

    if ($existing) {
        Capsule::table('mod_spam_filter_prefs')
            ->where('service_id', $serviceId)
            ->update($data);
    } else {
        $data['service_id'] = $serviceId;
        Capsule::table('mod_spam_filter_prefs')->insert($data);
    }

    return true;
}
```

## Whitelist/Blacklist Management

```php
public function addToWhitelist(int $serviceId, string $email): bool {
    $domain = Capsule::table('mod_spam_filter')
        ->where('service_id', $serviceId)
        ->first();

    $this->api->addWhitelist($domain->domain, $email);

    Capsule::table('mod_spam_filter_entries')->insert([
        'service_id' => $serviceId,
        'type' => 'whitelist',
        'email' => $email,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return true;
}

public function addToBlacklist(int $serviceId, string $email): bool {
    $domain = Capsule::table('mod_spam_filter')
        ->where('service_id', $serviceId)
        ->first();

    $this->api->addBlacklist($domain->domain, $email);

    Capsule::table('mod_spam_filter_entries')->insert([
        'service_id' => $serviceId,
        'type' => 'blacklist',
        'email' => $email,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return true;
}
```

## Quarantine Management

```php
public function getQuarantinedMessages(int $serviceId, int $limit = 50): array {
    $domain = Capsule::table('mod_spam_filter')
        ->where('service_id', $serviceId)
        ->first();

    $messages = $this->api->getQuarantine($domain->domain, [
        'limit' => $limit,
    ]);

    $result = [];
    foreach ($messages as $msg) {
        $result[] = [
            'id' => $msg['id'],
            'subject' => $msg['subject'],
            'from' => $msg['from'],
            'received' => $msg['received_at'],
            'score' => $msg['spam_score'],
            'size' => $msg['size'],
        ];
    }

    return $result;
}

public function releaseMessage(int $serviceId, string $messageId): bool {
    $domain = Capsule::table('mod_spam_filter')
        ->where('service_id', $serviceId)
        ->first();

    $this->api->releaseMessage($messageId);

    Capsule::table('mod_spam_filter_log')->insert([
        'service_id' => $serviceId,
        'action' => 'release',
        'message_id' => $messageId,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return true;
}

public function deleteQuarantinedMessage(int $serviceId, string $messageId): bool {
    $domain = Capsule::table('mod_spam_filter')
        ->where('service_id', $serviceId)
        ->first();

    $this->api->deleteQuarantinedMessage($messageId);

    return true;
}
```

## Statistics

```php
public function getSpamStats(int $serviceId): array {
    $domain = Capsule::table('mod_spam_filter')
        ->where('service_id', $serviceId)
        ->first();

    $stats = $this->api->getStats($domain->domain);

    Capsule::table('mod_spam_filter_stats')->insert([
        'service_id' => $serviceId,
        'emails_received' => $stats['received'],
        'spam_blocked' => $stats['spam_blocked'],
        'spam_quarantined' => $stats['spam_quarantined'],
        'virus_blocked' => $stats['virus_blocked'],
        'false_positives' => $stats['false_positives'],
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);

    return [
        'received' => $stats['received'],
        'spam_blocked' => $stats['spam_blocked'],
        'spam_quarantined' => $stats['spam_quarantined'],
        'blocked_rate' => round(($stats['spam_blocked'] / $stats['received']) * 100, 2),
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-security-hardening
- whmcs-email-template-builder