# Client Data Privacy Guide

Protecting client data is a legal and ethical obligation. This guide covers privacy best practices for WHMCS module development.

## Data Classification

### Data Sensitivity Levels

```php
<?php
/**
 * Data sensitivity classification
 */
class DataClassification
{
    const PUBLIC = 'public';
    const INTERNAL = 'internal';
    const CONFIDENTIAL = 'confidential';
    const RESTRICTED = 'restricted';

    private static $classificationMap = [
        // Client basic info
        'client_id' => self::INTERNAL,
        'firstname' => self::INTERNAL,
        'lastname' => self::INTERNAL,
        'email' => self::INTERNAL,
        'phone' => self::INTERNAL,
        'address1' => self::CONFIDENTIAL,
        'address2' => self::CONFIDENTIAL,
        'city' => self::CONFIDENTIAL,
        'state' => self::CONFIDENTIAL,
        'postcode' => self::CONFIDENTIAL,
        'country' => self::INTERNAL,
        'companyname' => self::INTERNAL,

        // Financial data
        'credit_card' => self::RESTRICTED,
        'bank_account' => self::RESTRICTED,
        'tax_id' => self::RESTRICTED,
        'ssn' => self::RESTRICTED,

        // Authentication
        'password_hash' => self::RESTRICTED,
        'api_key' => self::RESTRICTED,

        // Usage data
        'login_history' => self::CONFIDENTIAL,
        'activity_logs' => self::INTERNAL,
        'ip_address' => self::INTERNAL,
    ];

    /**
     * Get classification for field
     */
    public static function get(string $field): string
    {
        return self::$classificationMap[$field] ?? self::INTERNAL;
    }

    /**
     * Check if field requires encryption
     */
    public static function requiresEncryption(string $field): bool
    {
        return in_array(self::get($field), [self::CONFIDENTIAL, self::RESTRICTED]);
    }

    /**
     * Check if field can be logged
     */
    public static function canBeLogged(string $field): bool
    {
        return self::get($field) !== self::RESTRICTED;
    }
}
```

## Personal Data Handling

### Personal Data Processor

```php
<?php
/**
 * Personal data processing handler
 */
class PersonalDataProcessor
{
    private $encryption;

    public function __construct(EncryptionService $encryption)
    {
        $this->encryption = $encryption;
    }

    /**
     * Store personal data securely
     */
    public function store(int $clientId, string $dataType, $data): void
    {
        $classification = DataClassification::get($dataType);

        // Encrypt sensitive data
        if (DataClassification::requiresEncryption($dataType)) {
            $data = $this->encryption->encrypt(json_encode($data));
        }

        Capsule::table('mod_secure_data')->insert([
            'client_id' => $clientId,
            'data_type' => $dataType,
            'data_value' => $data,
            'classification' => $classification,
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        // Log the storage action (without sensitive values)
        $this->logDataAccess($clientId, 'store', $dataType);
    }

    /**
     * Retrieve personal data
     */
    public function retrieve(int $clientId, string $dataType): ?array
    {
        $record = Capsule::table('mod_secure_data')
            ->where('client_id', $clientId)
            ->where('data_type', $dataType)
            ->first();

        if (!$record) {
            return null;
        }

        $this->logDataAccess($clientId, 'retrieve', $dataType);

        // Decrypt if needed
        if (DataClassification::requiresEncryption($dataType)) {
            return json_decode($this->encryption->decrypt($record->data_value), true);
        }

        return $record->data_value;
    }

    /**
     * Anonymize personal data
     */
    public function anonymize(int $clientId, string $dataType): bool
    {
        $count = Capsule::table('mod_secure_data')
            ->where('client_id', $clientId)
            ->where('data_type', $dataType)
            ->update([
                'data_value' => '[ANONYMIZED]',
                'anonymized_at' => date('Y-m-d H:i:s'),
            ]);

        $this->logDataAccess($clientId, 'anonymize', $dataType);

        return $count > 0;
    }

    /**
     * Delete personal data (GDPR Right to Erasure)
     */
    public function erase(int $clientId, ?string $dataType = null): int
    {
        $query = Capsule::table('mod_secure_data')
            ->where('client_id', $clientId);

        if ($dataType) {
            $query->where('data_type', $dataType);
        }

        $count = $query->delete();

        $this->logDataAccess($clientId, 'erase', $dataType ?? 'all');

        return $count;
    }

    private function logDataAccess(int $clientId, string $action, string $dataType): void
    {
        Capsule::table('mod_data_access_log')->insert([
            'client_id' => $clientId,
            'action' => $action,
            'data_type' => $dataType,
            'admin_id' => $_SESSION['adminid'] ?? null,
            'ip_address' => $this->getClientIp(),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    private function getClientIp(): string
    {
        return $_SERVER['HTTP_X_FORWARDED_FOR'] ?? $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    }
}
```

## GDPR Compliance

### Data Subject Request Handler

```php
<?php
/**
 * GDPR Data Subject Request Handler
 */
class DataSubjectRequestHandler
{
    /**
     * Handle data export request (Right to Portability)
     */
    public function export(int $clientId): array
    {
        $data = [
            'exported_at' => date('c'),
            'client_id' => $clientId,
            'personal_data' => $this->collectPersonalData($clientId),
            'service_data' => $this->collectServiceData($clientId),
            'billing_data' => $this->collectBillingData($clientId),
        ];

        // Log export request
        $this->logRequest($clientId, 'export', 'completed');

        return $data;
    }

    /**
     * Handle data deletion request (Right to Erasure)
     */
    public function requestDeletion(int $clientId): DeletionRequest
    {
        $request = Capsule::table('mod_deletion_requests')->insertGetId([
            'client_id' => $clientId,
            'status' => 'pending',
            'requested_at' => date('Y-m-d H:i:s'),
            'confirmation_token' => bin2hex(random_bytes(32)),
        ]);

        // Send confirmation email
        $this->sendDeletionConfirmationEmail($clientId);

        return new DeletionRequest($request);
    }

    /**
     * Confirm and process deletion
     */
    public function confirmDeletion(string $token): bool
    {
        $request = Capsule::table('mod_deletion_requests')
            ->where('confirmation_token', $token)
            ->where('status', 'pending')
            ->first();

        if (!$request) {
            return false;
        }

        // Check if within grace period (30 days for backups)
        Capsule::table('mod_deletion_requests')
            ->where('id', $request->id)
            ->update(['status' => 'processing']);

        // Process deletion in background
        $this->scheduleDeletion($request->client_id);

        return true;
    }

    private function collectPersonalData(int $clientId): array
    {
        $client = Capsule::table('tblclients')
            ->where('id', $clientId)
            ->first();

        return [
            'first_name' => $client->firstname ?? null,
            'last_name' => $client->lastname ?? null,
            'email' => $client->email ?? null,
            'phone' => $client->phonenumber ?? null,
            'company' => $client->companyname ?? null,
            'address' => [
                'address1' => $client->address1 ?? null,
                'address2' => $client->address2 ?? null,
                'city' => $client->city ?? null,
                'state' => $client->state ?? null,
                'postcode' => $client->postcode ?? null,
                'country' => $client->country ?? null,
            ],
            'created_at' => $client->datecreated ?? null,
        ];
    }

    private function collectServiceData(int $clientId): array
    {
        $services = Capsule::table('tblhosting')
            ->where('userid', $clientId)
            ->get();

        return array_map(function ($service) {
            return [
                'domain' => $service->domain ?? null,
                'product' => $service->packageid ?? null,
                'status' => $service->domainstatus ?? null,
                'created_at' => $service->regdate ?? null,
            ];
        }, $services);
    }

    private function collectBillingData(int $clientId): array
    {
        $invoices = Capsule::table('tblinvoices')
            ->where('userid', $clientId)
            ->get();

        return [
            'invoices' => array_map(function ($invoice) {
                return [
                    'id' => $invoice->id ?? null,
                    'amount' => $invoice->total ?? null,
                    'status' => $invoice->status ?? null,
                    'date' => $invoice->date ?? null,
                ];
            }, $invoices),
            'balance' => Capsule::table('tblclients')->where('id', $clientId)->value('credit'),
        ];
    }

    private function logRequest(int $clientId, string $type, string $status): void
    {
        Capsule::table('mod_gdpr_requests')->insert([
            'client_id' => $clientId,
            'request_type' => $type,
            'status' => $status,
            'ip_address' => $this->getClientIp(),
            'created_at' => date('Y-m-d H:i:s'),
        ]);
    }

    private function sendDeletionConfirmationEmail(int $clientId): void
    {
        // Email sending implementation
    }

    private function scheduleDeletion(int $clientId): void
    {
        // Queue deletion for background processing
    }
}
```

## Data Retention

### Retention Policy Manager

```php
<?php
/**
 * Data retention policy manager
 */
class DataRetentionManager
{
    private array $policies = [
        'activity_logs' => 90,      // 90 days
        'login_history' => 180,    // 180 days
        'audit_logs' => 365,      // 1 year
        'session_data' => 30,     // 30 days
        'cache_data' => 7,         // 7 days
        'temp_files' => 1,         // 1 day
        'failed_login_attempts' => 30,
        'email_logs' => 90,
    ];

    /**
     * Check if data can be deleted based on retention policy
     */
    public function canDelete(string $dataType, DateTime $createdAt): bool
    {
        $retentionDays = $this->policies[$dataType] ?? 365;

        $retentionDate = (new DateTime())->modify("-{$retentionDays} days");

        return $createdAt < $retentionDate;
    }

    /**
     * Clean up data based on retention policies
     */
    public function enforceRetention(): CleanupResult
    {
        $deleted = [];
        $errors = [];

        foreach ($this->policies as $dataType => $retentionDays) {
            try {
                $cutoffDate = (new DateTime())->modify("-{$retentionDays} days");

                $count = $this->deleteExpiredData($dataType, $cutoffDate);
                $deleted[$dataType] = $count;
            } catch (Exception $e) {
                $errors[$dataType] = $e->getMessage();
            }
        }

        return new CleanupResult($deleted, $errors);
    }

    private function deleteExpiredData(string $dataType, DateTime $cutoff): int
    {
        $tableMap = [
            'activity_logs' => 'mod_activity_log',
            'login_history' => 'mod_login_history',
            'audit_logs' => 'mod_security_audit_log',
            'session_data' => 'mod_sessions',
        ];

        $table = $tableMap[$dataType] ?? null;

        if (!$table) {
            return 0;
        }

        return Capsule::table($table)
            ->where('created_at', '<', $cutoff->format('Y-m-d H:i:s'))
            ->delete();
    }

    /**
     * Get retention policy for a data type
     */
    public function getRetentionPeriod(string $dataType): int
    {
        return $this->policies[$dataType] ?? 365;
    }

    /**
     * Set retention policy
     */
    public function setRetentionPeriod(string $dataType, int $days): void
    {
        if ($days < 1) {
            throw new InvalidArgumentException('Retention period must be at least 1 day');
        }

        $this->policies[$dataType] = $days;
    }
}
```

## Consent Management

### Consent Manager

```php
<?php
/**
 * Consent management system
 */
class ConsentManager
{
    private array $consentTypes = [
        'marketing' => 'Marketing Communications',
        'analytics' => 'Analytics & Performance',
        'third_party' => 'Third-Party Integrations',
        'data_processing' => 'Data Processing Agreement',
    ];

    /**
     * Record user consent
     */
    public function recordConsent(
        int $clientId,
        string $consentType,
        bool $granted,
        ?string $version = null
    ): void {
        Capsule::table('mod_consent_records')->insert([
            'client_id' => $clientId,
            'consent_type' => $consentType,
            'granted' => $granted,
            'version' => $version ?? $this->getCurrentVersion($consentType),
            'ip_address' => $this->getClientIp(),
            'user_agent' => $_SERVER['HTTP_USER_AGENT'] ?? '',
            'recorded_at' => date('Y-m-d H:i:s'),
        ]);
    }

    /**
     * Check if consent is granted
     */
    public function hasConsent(int $clientId, string $consentType): bool
    {
        $record = Capsule::table('mod_consent_records')
            ->where('client_id', $clientId)
            ->where('consent_type', $consentType)
            ->where('granted', true)
            ->orderBy('recorded_at', 'desc')
            ->first();

        if (!$record) {
            return false;
        }

        // Check if consent is still valid (version matches current)
        $currentVersion = $this->getCurrentVersion($consentType);

        return $record->version === $currentVersion;
    }

    /**
     * Revoke consent
     */
    public function revokeConsent(int $clientId, string $consentType): void
    {
        $this->recordConsent($clientId, $consentType, false);
    }

    /**
     * Get all consent records for client
     */
    public function getClientConsents(int $clientId): array
    {
        $records = Capsule::table('mod_consent_records')
            ->where('client_id', $clientId)
            ->orderBy('recorded_at', 'desc')
            ->get();

        $consents = [];
        foreach ($records as $record) {
            if (!isset($consents[$record->consent_type])) {
                $consents[$record->consent_type] = [
                    'type' => $record->consent_type,
                    'granted' => false,
                    'version' => null,
                    'recorded_at' => null,
                ];
            }

            if ($record->granted && $consents[$record->consent_type]['recorded_at'] < $record->recorded_at) {
                $consents[$record->consent_type] = [
                    'type' => $record->consent_type,
                    'granted' => true,
                    'version' => $record->version,
                    'recorded_at' => $record->recorded_at,
                ];
            }
        }

        return array_values($consents);
    }

    private function getCurrentVersion(string $consentType): string
    {
        return '1.0'; // Would typically come from config
    }

    private function getClientIp(): string
    {
        return $_SERVER['HTTP_X_FORWARDED_FOR'] ?? $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    }
}
```

## Privacy by Design

### Data Minimization Service

```php
<?php
/**
 * Data minimization service
 */
class DataMinimizer
{
    /**
     * Collect only necessary data
     */
    public function minimizePersonalData(array $data, string $purpose): array
    {
        $requiredFields = $this->getRequiredFields($purpose);

        return array_filter($data, function ($key) use ($requiredFields) {
            return in_array($key, $requiredFields);
        }, ARRAY_FILTER_USE_KEY);
    }

    /**
     * Get required fields for specific purpose
     */
    private function getRequiredFields(string $purpose): array
    {
        return match ($purpose) {
            'order_processing' => [
                'firstname', 'lastname', 'email', 'address1', 'city', 'country',
            ],
            'payment' => [
                'billing_firstname', 'billing_lastname', 'billing_address',
                'billing_city', 'billing_country',
            ],
            'support' => [
                'client_id', 'email', 'firstname', 'lastname',
            ],
            'marketing' => [
                'email', 'firstname', 'lastname', 'country',
            ],
            default => [],
        };
    }

    /**
     * Mask sensitive data for display
     */
    public function mask(string $dataType, $value): string
    {
        if (empty($value)) {
            return '';
        }

        return match ($dataType) {
            'email' => $this->maskEmail($value),
            'phone' => $this->maskPhone($value),
            'credit_card' => $this->maskCreditCard($value),
            'ssn' => $this->maskSsn($value),
            default => substr($value, 0, 3) . '***',
        };
    }

    private function maskEmail(string $email): string
    {
        $parts = explode('@', $email);
        $local = $parts[0] ?? '';
        $domain = $parts[1] ?? '';

        $maskedLocal = substr($local, 0, 2) . str_repeat('*', max(0, strlen($local) - 2));

        return $maskedLocal . '@' . $domain;
    }

    private function maskPhone(string $phone): string
    {
        return substr($phone, 0, 3) . '****' . substr($phone, -4);
    }

    private function maskCreditCard(string $card): string
    {
        return '****' . substr($card, -4);
    }

    private function maskSsn(string $ssn): string
    {
        return '***-**-' . substr($ssn, -4);
    }
}
```

## Privacy Compliance Checklist

| Requirement | Implementation | Status |
|-------------|----------------|--------|
| Lawful basis for processing | Document legal basis for each data type | [ ] |
| Data minimization | Collect only necessary data | [ ] |
| Purpose limitation | Use data only for stated purposes | [ ] |
| Storage limitation | Implement retention policies | [ ] |
| Accuracy | Keep personal data accurate | [ ] |
| Integrity and confidentiality | Apply appropriate security | [ ] |
| Accountability | Document all data processing | [ ] |
| Data subject rights | Implement access/export/delete requests | [ ] |
| Consent management | Track and honor user consent | [ ] |
| Breach notification | Define notification procedures | [ ] |

## Related Patterns

- [Admin Security](./admin-security.md) - Admin data access
- [Database Migrations](./database-migrations.md) - Data handling in migrations
- [Session Management](./session-management.md) - Session data handling