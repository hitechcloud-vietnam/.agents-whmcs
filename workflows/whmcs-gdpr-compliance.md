# WHMCS GDPR Compliance Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Make WHMCS modules GDPR compliant.

## Steps

### 1. Data Inventory
```
□ Identify all personal data collected
□ Map data flow
□ Document retention periods
□ Identify third-party data sharing
```

### 2. Data Minimization
```php
// Only collect necessary data
$data = [
    'id' => $userId,
    'email' => $email,  // Required
    // 'phone' => $phone, // Only if explicitly needed
    // 'address' => $address, // Only if shipping required
];
```

### 3. Consent Management
```php
Capsule::schema()->create('mod_consent_records', function($t) {
    $t->increments('id');
    $t->unsignedInteger('user_id');
    $t->string('consent_type', 100);
    $t->boolean('granted');
    $t->string('ip_address', 45);
    $t->text('user_agent');
    $t->timestamp('recorded_at');
});
```

### 4. Right to Access
```php
function getUserDataExport(int $userId): array {
    return [
        'profile' => getClientProfile($userId),
        'services' => getClientServices($userId),
        'invoices' => getClientInvoices($userId),
        'tickets' => getClientTickets($userId),
        'consent' => getConsentRecords($userId),
        'exported_at' => date('Y-m-d H:i:s'),
    ];
}
```

### 5. Right to Deletion
```php
function anonymizeUserData(int $userId): void {
    Capsule::table('tblclients')
        ->where('id', $userId)
        ->update([
            'firstname' => 'Deleted',
            'lastname' => 'User',
            'email' => 'deleted_' . $userId . '@anonymized.local',
            'phonenumber' => '',
        ]);

    Capsule::table('mod_log')->where('user_id', $userId)->delete();
}
```

### 6. Data Retention
```php
function cleanupOldData(int $retentionDays): void {
    $cutoff = date('Y-m-d H:i:s', strtotime("-{$retentionDays} days"));

    Capsule::table('mod_activity_logs')
        ->where('created_at', '<', $cutoff)
        ->where('anonymized', 1)
        ->delete();
}
```

## Output

GDPR-compliant module checklist:
- Consent management
- Data export capability
- Anonymization procedures
- Retention policies
