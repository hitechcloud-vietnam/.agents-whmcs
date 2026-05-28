# WHMCS Email Archive Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build email archiving modules for compliance and retention requirements.

## Email Archive Module Structure

```php
<?php
/**
 * Email Archive Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Email Archiving',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'storage_endpoint'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'RetentionPeriod' => [
            'Type' => 'dropdown',
            'Options' => '1year,3years,5years,7years,10years',
            'Default' => '7years',
        ],
        'ArchiveMode' => [
            'Type' => 'dropdown',
            'Options' => 'journal,mirror,live',
            'Default' => 'journal',
        ],
        'Encryption' => [
            'Type' => 'yesno',
            'Description' => 'Enable encryption at rest',
        ],
        'SearchEnabled' => [
            'Type' => 'yesno',
            'Description' => 'Enable full-text search',
        ],
    ];
}
```

## Email Archiving

```php
function {module}_CreateAccount(array $params): string {
    $retention = str_replace('years', '', $params['configoption1']);

    $archive = $this->api->createArchive([
        'account_id' => $params['userid'],
        'domain' => $params['domain'],
        'retention_years' => (int) $retention,
        'mode' => $params['configoption2'],
        'encryption' => $params['configoption3'] === 'on',
        'full_text_search' => $params['configoption4'] === 'on',
    ]);

    Capsule::table('mod_email_archive')->insert([
        'service_id' => $params['serviceid'],
        'archive_id' => $archive['id'],
        'user_id' => $params['userid'],
        'domain' => $params['domain'],
        'retention_years' => (int) $retention,
        'mode' => $params['configoption2'],
        'email_count' => 0,
        'storage_used' => 0,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $archive = Capsule::table('mod_email_archive')
        ->where('service_id', $params['serviceid'])
        ->first();

    if ($archive) {
        $this->api->closeArchive($archive->archive_id);
        Capsule::table('mod_email_archive')
            ->where('id', $archive->id)
            ->update(['status' => 'closed']);
    }

    return 'success';
}
```

## Search & Retrieval

```php
public function searchEmails(int $serviceId, array $criteria): array {
    $archive = Capsule::table('mod_email_archive')
        ->where('service_id', $serviceId)
        ->first();

    $results = $this->api->search([
        'archive_id' => $archive->archive_id,
        'query' => $criteria['query'] ?? null,
        'from' => $criteria['from'] ?? null,
        'to' => $criteria['to'] ?? null,
        'date_from' => $criteria['date_from'] ?? null,
        'date_to' => $criteria['date_to'] ?? null,
        'has_attachments' => $criteria['has_attachments'] ?? null,
        'limit' => $criteria['limit'] ?? 50,
        'offset' => $criteria['offset'] ?? 0,
    ]);

    return $results;
}

public function exportEmail(int $serviceId, string $emailId, string $format = 'eml'): string {
    $archive = Capsule::table('mod_email_archive')
        ->where('service_id', $serviceId)
        ->first();

    $email = $this->api->getEmail($archive->archive_id, $emailId, $format);

    $filename = "archive_export_{$emailId}." . ($format === 'eml' ? 'eml' : 'msg');

    header('Content-Type: application/octet-stream');
    header('Content-Disposition: attachment; filename="' . $filename . '"');
    header('Content-Length: ' . strlen($email));

    echo $email;
    exit;
}
```

## Statistics

```php
public function getArchiveStats(int $serviceId): array {
    $archive = Capsule::table('mod_email_archive')
        ->where('service_id', $serviceId)
        ->first();

    $stats = $this->api->getArchiveStats($archive->archive_id);

    Capsule::table('mod_email_archive')
        ->where('service_id', $serviceId)
        ->update([
            'email_count' => $stats['total_emails'],
            'storage_used' => $stats['storage_used'],
            'last_sync' => date('Y-m-d H:i:s'),
        ]);

    return [
        'total_emails' => $stats['total_emails'],
        'storage_used' => $this->formatBytes($stats['storage_used']),
        'retention_until' => date('Y-m-d', strtotime("+{$archive->retention_years} years")),
        'mode' => $archive->mode,
    ];
}

private function formatBytes(int $bytes): string {
    $units = ['B', 'KB', 'MB', 'GB', 'TB'];
    $i = 0;
    while ($bytes >= 1024 && $i < 4) {
        $bytes /= 1024;
        $i++;
    }
    return round($bytes, 2) . ' ' . $units[$i];
}
```

## Client Area

```php
function {module}_ClientArea(array $params): array {
    $archive = Capsule::table('mod_email_archive')
        ->where('service_id', $params['serviceid'])
        ->first();

    $stats = $this->getArchiveStats($params['serviceid']);

    return [
        'pagetitle' => 'Email Archive',
        'templatefile' => 'templates/email_archive_clientarea',
        'vars' => [
            'archive' => $archive,
            'stats' => $stats,
            'search_enabled' => $archive->mode !== 'live',
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-gdpr-compliance
- whmcs-reporting