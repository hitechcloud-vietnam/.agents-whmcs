# WHMCS GDPR Compliance Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for making WHMCS modules GDPR compliant.

## When to Use

- Processing EU customer data
- Building data export features
- Implementing privacy controls

## GDPR Patterns

### Data Export
```php
function exportUserData(int $userId): array {
    $client = Capsule::table('tblclients')->where('id', $userId)->first();

    return [
        'personal_info' => [
            'firstname' => $client->firstname,
            'lastname' => $client->lastname,
            'email' => $client->email,
            'address' => $client->address1,
            'city' => $client->city,
            'country' => $client->country,
        ],
        'services' => Capsule::table('tblhosting')
            ->where('userid', $userId)
            ->get(),
        'invoices' => Capsule::table('tblinvoices')
            ->where('userid', $userId)
            ->get(),
        'tickets' => Capsule::table('tbltickets')
            ->where('userid', $userId)
            ->get(),
        'exported_at' => date('Y-m-d H:i:s'),
    ];
}
```

### Data Deletion
```php
function deleteUserData(int $userId): void {
    // Anonymize client data
    Capsule::table('tblclients')
        ->where('id', $userId)
        ->update([
            'firstname' => 'Deleted',
            'lastname' => 'User',
            'email' => 'deleted_' . $userId . '@anonymized.local',
            'phonenumber' => '',
            'address1' => '',
            'address2' => '',
            'city' => '',
            'postcode' => '',
        ]);

    // Delete module-specific data
    Capsule::table('mod_{module}_logs')->where('user_id', $userId)->delete();
}
```

### Consent Management
```php
function recordConsent(int $userId, string $type, bool $granted): void {
    Capsule::table('mod_consent_log')->insert([
        'user_id' => $userId,
        'consent_type' => $type,
        'granted' => $granted,
        'ip_address' => $_SERVER['REMOTE_ADDR'],
        'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);
}
```

---

**Related Skills:**
- whmcs-security-hardening
- whmcs-database-design
- whmcs-clientarea-builder
