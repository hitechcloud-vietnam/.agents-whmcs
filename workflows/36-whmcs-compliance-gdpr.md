# WHMCS GDPR Compliance Workflow

## Overview
This workflow ensures WHMCS installations comply with GDPR requirements.

## Step 1: GDPR Compliance Service

```php
<?php
// src/Service/GdprComplianceService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class GdprComplianceService
{
    private $auditLog;

    public function __construct()
    {
        $this->auditLog = new AuditLogService();
    }

    /**
     * Right to Access - Provide all data about a client
     */
    public function exportClientData(int $clientId): array
    {
        $client = Capsule::table('tblclients')->where('id', $clientId)->first();

        if (!$client) {
            throw new \Exception("Client not found");
        }

        $data = [
            'profile' => (array)$client,
            'contacts' => Capsule::table('tblcontacts')->where('userid', $clientId)->get()->toArray(),
            'services' => Capsule::table('tblhosting')->where('userid', $clientId)->get()->toArray(),
            'domains' => Capsule::table('tbldomains')->where('userid', $clientId)->get()->toArray(),
            'invoices' => Capsule::table('tblinvoices')->where('userid', $clientId)->get()->toArray(),
            'tickets' => Capsule::table('tbltickets')->where('userid', $clientId)->get()->toArray(),
            'notes' => Capsule::table('tblnotes')->where('userid', $clientId)->get()->toArray(),
            'activity_log' => Capsule::table('tblactivitylog')->where('userid', $clientId)->get()->toArray()
        ];

        $this->auditLog->log('export', 'gdpr', [
            'client_id' => $clientId,
            'data_types' => array_keys($data)
        ]);

        return $data;
    }

    /**
     * Right to Erasure - Delete client data
     */
    public function anonymizeClient(int $clientId): void
    {
        $this->auditLog->log('anonymize', 'gdpr', ['client_id' => $clientId]);

        // Anonymize main client record
        Capsule::table('tblclients')
            ->where('id', $clientId)
            ->update([
                'firstname' => 'REDACTED',
                'lastname' => 'USER',
                'email' => 'deleted_' . $clientId . '@redacted.local',
                'companyname' => '',
                'address1' => '',
                'address2' => '',
                'city' => '',
                'state' => '',
                'postcode' => '',
                'country' => '',
                'phonenumber' => '',
                'tax_id' => '',
                'password' => hash('sha256', random_bytes(32)),
                'email_verified' => 0,
                'status' => 'Closed'
            ]);

        // Anonymize contacts
        Capsule::table('tblcontacts')
            ->where('userid', $clientId)
            ->update([
                'firstname' => 'REDACTED',
                'lastname' => 'CONTACT',
                'email' => 'deleted_contact_' . $clientId . '@redacted.local'
            ]);

        // Anonymize notes
        Capsule::table('tblnotes')
            ->where('userid', $clientId)
            ->update([
                'note' => '[Content redacted for GDPR compliance]',
                'adminid' => null
            ]);
    }

    /**
     * Data portability - Export in standard format
     */
    public function exportClientDataPortable(int $clientId): array
    {
        $data = $this->exportClientData($clientId);

        return [
            'format_version' => '1.0',
            'exported_at' => date('Y-m-d H:i:s'),
            'client_id' => $clientId,
            'data' => $data
        ];
    }

    /**
     * Consent management
     */
    public function recordConsent(int $clientId, string $purpose, string $consentType, bool $granted): void
    {
        Capsule::table('mod_consent_log')->insert([
            'client_id' => $clientId,
            'purpose' => $purpose,
            'consent_type' => $consentType,
            'granted' => $granted,
            'ip_address' => $_SERVER['REMOTE_ADDR'] ?? null,
            'user_agent' => substr($_SERVER['HTTP_USER_AGENT'] ?? '', 0, 500),
            'recorded_at' => date('Y-m-d H:i:s')
        ]);
    }

    /**
     * Check if client has given consent for purpose
     */
    public function hasConsent(int $clientId, string $purpose): bool
    {
        $consent = Capsule::table('mod_consent_log')
            ->where('client_id', $clientId)
            ->where('purpose', $purpose)
            ->where('granted', 1)
            ->orderBy('recorded_at', 'desc')
            ->first();

        if (!$consent) {
            return false;
        }

        // Check if consent is still valid (e.g., within 1 year)
        $consentDate = new \DateTime($consent->recorded_at);
        $now = new \DateTime();
        $diff = $consentDate->diff($now);

        return $diff->y < 1;
    }

    /**
     * Data breach notification
     */
    public function logDataBreach(int $affectedClientId, string $description, string $severity): void
    {
        Capsule::table('mod_data_breaches')->insert([
            'affected_client_id' => $affectedClientId,
            'description' => $description,
            'severity' => $severity,
            'reported_at' => date('Y-m-d H:i:s'),
            'notified_at' => null,
            'resolved_at' => null
        ]);
    }

    /**
     * Generate data retention report
     */
    public function generateRetentionReport(): array
    {
        $closedClients = Capsule::table('tblclients')
            ->where('status', 'Closed')
            ->where('dateclosed', '<', date('Y-m-d', strtotime('-2 years')))
            ->count();

        $pendingDeletion = Capsule::table('mod_gdpr_deletion_queue')
            ->where('status', 'pending')
            ->count();

        return [
            'clients_pending_deletion' => $pendingDeletion,
            'clients_eligible_for_deletion' => $closedClients,
            'last_review_date' => date('Y-m-d'),
            'retention_policy' => '2 years after account closure'
        ];
    }
}
```

## Verification Checklist

- [ ] Data export working
- [ ] Data anonymization working
- [ ] Consent tracking implemented
- [ ] Breach logging configured
- [ ] Retention report working
- [ ] GDPR rights implemented
