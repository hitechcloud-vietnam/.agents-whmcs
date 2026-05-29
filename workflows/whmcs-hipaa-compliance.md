# WHMCS HIPAA Compliance Workflow

## Overview
This workflow implements HIPAA (Health Insurance Portability and Accountability Act) compliance for WHMCS.

## Prerequisites
- WHMCS with healthcare-related services
- Compliance officer access
- BAA with hosting provider (if applicable)

## Step-by-Step Process

### Step 1: HIPAA Compliance Framework
```
HIPAA SAFEGUARDS:

Administrative Safeguards:
□ Risk assessment and management
□ Workforce training
□ Contingency planning

Physical Safeguards:
□ Facility access controls
□ Workstation security
□ Device and media controls

Technical Safeguards:
□ Access controls (unique IDs)
□ Audit controls
□ Integrity controls
□ Transmission security
```

### Step 2: Protected Health Information (PHI) Handling
```php
<?php
// Identify and protect PHI

class PHIProtectionHandler {
    private $phiFields = [
        'health_info',
        'medical_records',
        'insurance_id',
        'diagnosis',
        'treatment_info'
    ];

    /**
     * Encrypt PHI at rest
     */
    public function storePHI(int $clientId, array $data): void
    {
        // Encrypt sensitive fields
        $encryptedData = [];
        foreach ($data as $key => $value) {
            if (in_array($key, $this->phiFields)) {
                $encryptedData[$key] = encryptData($value);
            } else {
                $encryptedData[$key] = $value;
            }
        }

        Capsule::table('mod_phi_records')->insert([
            'client_id' => $clientId,
            'data' => json_encode($encryptedData),
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    /**
     * Access logging for HIPAA audit trail
     */
    public function logPHIAccess(int $clientId, string $action): void
    {
        Capsule::table('mod_phi_audit_trail')->insert([
            'client_id' => $clientId,
            'admin_id' => adminId(),
            'action' => $action,
            'ip_address' => $_SERVER['REMOTE_ADDR'],
            'timestamp' => date('Y-m-d H:i:s')
        ]);
    }
}
```

### Step 3: Access Controls
```php
<?php
// Minimum necessary access

function requestPHIAccess(int $clientId, string $reason): bool
{
    // Verify admin has HIPAA training
    $hasTraining = Capsule::table('tbladminperms')
        ->join('tbladminpermissions', 'tbladminpermissions.id', '=', 'tbladminperms.permid')
        ->where('tbladminperms.adminid', adminId())
        ->where('tbladminpermissions.name', 'hipaa_trained')
        ->exists();

    if (!$hasTraining) {
        throw new Exception('Admin lacks required HIPAA training');
    }

    // Log access request
    Capsule::table('mod_phi_access_log')->insert([
        'admin_id' => adminId(),
        'client_id' => $clientId,
        'reason' => $reason,
        'approved_at' => date('Y-m-d H:i:s')
    ]);

    return true;
}
```

### Step 4: Breach Notification Procedures
```php
<?php
// HIPAA breach response workflow

function handlePHIBreach(array $breachDetails): void
{
    // Step 1: Document the breach
    $breachId = Capsule::table('mod_breach_log')->insertGetId([
        'type' => 'phi_breach',
        'description' => json_encode($breachDetails),
        'discovered_at' => date('Y-m-d H:i:s'),
        'status' => 'investigating'
    ]);

    // Step 2: Assess severity
    $affectedCount = assessBreachSeverity($breachDetails);

    // Step 3: Notify (if required)
    if ($affectedCount > 500) {
        notifyHHSImmediately();
        notifyAffectedIndividuals($breachId);
        notifyMedia(); // For large breaches
    } else {
        notifyAffectedIndividuals($breachId); // Annual notification OK
    }

    // Step 4: Document mitigation
    Capsule::table('mod_breach_log')
        ->where('id', $breachId)
        ->update(['mitigation_actions' => json_encode(getMitigationActions())]);
}
```

## HIPAA Compliance Checklist

- [ ] Business Associate Agreement (BAA) signed
- [ ] Risk assessment completed annually
- [ ] Workforce training documented
- [ ] PHI access logging enabled
- [ ] Encryption for PHI at rest and in transit
- [ ] Breach notification procedures documented
- [ ] Incident response plan tested

## Related Workflows
- [WHMCS GDPR Compliance](./whmcs-gdpr-compliance.md)
- [WHMCS Incident Response](./whmcs-incident-response.md)