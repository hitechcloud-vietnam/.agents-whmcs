# WHMCS Mail Server Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build mail server modules for hosted email services.

## Mail Server Module Structure

```php
<?php
/**
 * Mail Server Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Hosted Email',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'server_hostname'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'MailboxQuota' => [
            'Type' => 'dropdown',
            'Options' => '1GB,5GB,10GB,25GB,50GB,100GB',
            'Default' => '10GB',
        ],
        'MaxAliases' => [
            'Type' => 'dropdown',
            'Options' => '0,5,10,25,unlimited',
            'Default' => '10',
        ],
        'EnableCatchAll' => [
            'Type' => 'yesno',
            'Description' => 'Enable catch-all email',
        ],
        'EnableWebmail' => [
            'Type' => 'yesno',
            'Description' => 'Enable webmail access',
        ],
        'EnableActiveSync' => [
            'Type' => 'yesno',
            'Description' => 'Enable ActiveSync',
        ],
    ];
}
```

## Mailbox Management

```php
function {module}_CreateAccount(array $params): string {
    $domain = $params['domain'];

    $this->api->createDomain([
        'domain' => $domain,
        'quota' => $this->parseQuota($params['configoption1']),
    ]);

    Capsule::table('mod_mail_server')->insert([
        'service_id' => $params['serviceid'],
        'domain' => $domain,
        'quota' => $this->parseQuota($params['configoption1']),
        'max_aliases' => $params['configoption2'],
        'catch_all' => $params['configoption3'] === 'on',
        'webmail' => $params['configoption4'] === 'on',
        'active_sync' => $params['configoption5'] === 'on',
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_SuspendAccount(array $params): string {
    $domain = $this->getMailDomain($params['serviceid']);

    $this->api->suspendDomain($domain->domain);

    Capsule::table('mod_mail_server')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'suspended']);

    return 'success';
}

function {module}_UnsuspendAccount(array $params): string {
    $domain = $this->getMailDomain($params['serviceid']);

    $this->api->unsuspendDomain($domain->domain);

    Capsule::table('mod_mail_server')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'active']);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $domain = $this->getMailDomain($params['serviceid']);

    if ($domain) {
        $this->api->deleteDomain($domain->domain);
        Capsule::table('mod_mail_server')
            ->where('id', $domain->id)
            ->delete();
    }

    return 'success';
}
```

## Mailbox Operations

```php
public function createMailbox(int $serviceId, string $email, string $password): array {
    $domain = $this->getMailDomain($serviceId);
    $quota = $domain->quota;

    $mailbox = $this->api->createMailbox([
        'domain' => $domain->domain,
        'email' => $email,
        'password' => $password,
        'quota' => $quota,
    ]);

    Capsule::table('mod_mail_mailboxes')->insert([
        'domain_id' => $domain->id,
        'email' => $email,
        'remote_id' => $mailbox['id'],
        'quota' => $quota,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return $mailbox;
}

public function deleteMailbox(int $serviceId, string $email): bool {
    $mailbox = Capsule::table('mod_mail_mailboxes')
        ->where('domain_id', $serviceId)
        ->where('email', $email)
        ->first();

    if ($mailbox) {
        $this->api->deleteMailbox($mailbox->remote_id);
        Capsule::table('mod_mail_mailboxes')
            ->where('id', $mailbox->id)
            ->delete();
    }

    return true;
}

public function setMailboxPassword(int $serviceId, string $email, string $newPassword): bool {
    $mailbox = Capsule::table('mod_mail_mailboxes')
        ->where('domain_id', $serviceId)
        ->where('email', $email)
        ->first();

    $this->api->setPassword($mailbox->remote_id, $newPassword);

    return true;
}
```

## Alias Management

```php
public function createAlias(int $serviceId, string $source, string $destination): bool {
    $domain = $this->getMailDomain($serviceId);

    $alias = $this->api->createAlias([
        'domain' => $domain->domain,
        'source' => $source,
        'destination' => $destination,
    ]);

    Capsule::table('mod_mail_aliases')->insert([
        'domain_id' => $domain->id,
        'source' => $source,
        'destination' => $destination,
        'remote_id' => $alias['id'],
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return true;
}

public function setCatchAll(int $serviceId, string $destination): bool {
    $domain = $this->getMailDomain($serviceId);

    $this->api->setCatchAll($domain->domain, $destination);

    Capsule::table('mod_mail_server')
        ->where('id', $domain->id)
        ->update(['catch_all_destination' => $destination]);

    return true;
}
```

## Statistics

```php
public function getMailboxStats(int $serviceId, string $email): array {
    $mailbox = Capsule::table('mod_mail_mailboxes')
        ->where('domain_id', $serviceId)
        ->where('email', $email)
        ->first();

    $stats = $this->api->getMailboxStats($mailbox->remote_id);

    Capsule::table('mod_mail_stats')->insert([
        'mailbox_id' => $mailbox->id,
        'messages_total' => $stats['total'],
        'messages_unread' => $stats['unread'],
        'storage_used' => $stats['used'],
        'storage_quota' => $stats['quota'],
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);

    return $stats;
}

private function parseQuota(string $quota): int {
    $value = (int) $quota;
    $unit = str_replace($value, '', $quota);
    $multiplier = ['GB' => 1024 * 1024 * 1024, 'MB' => 1024 * 1024];
    return $value * ($multiplier[$unit] ?? 1);
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-spam-filter
- whmcs-clientarea-builder