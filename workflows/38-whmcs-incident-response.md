# WHMCS Incident Response Workflow

## Overview
This workflow provides a structured approach to handling security incidents in WHMCS.

## Step 1: Incident Response Service

```php
<?php
// src/Service/IncidentResponseService.php

namespace WHMCS\Module\Addon\YourModule\Service;

use WHMCS\Database\Capsule;

class IncidentResponseService
{
    private $severityLevels = [
        'critical' => 1,
        'high' => 2,
        'medium' => 3,
        'low' => 4
    ];

    public function createIncident(string $title, string $description, string $severity, array $affectedEntities = []): int
    {
        $incidentId = Capsule::table('mod_incidents')->insertGetId([
            'title' => $title,
            'description' => $description,
            'severity' => $severity,
            'status' => 'open',
            'detected_at' => date('Y-m-d H:i:s'),
            'reported_by' => $_SESSION['adminid'] ?? null
        ]);

        foreach ($affectedEntities as $entity) {
            Capsule::table('mod_incident_affected')->insert([
                'incident_id' => $incidentId,
                'entity_type' => $entity['type'],
                'entity_id' => $entity['id'],
                'created_at' => date('Y-m-d H:i:s')
            ]);
        }

        // Send alert for critical/high incidents
        if (in_array($severity, ['critical', 'high'])) {
            $this->sendIncidentAlert($incidentId, $severity);
        }

        return $incidentId;
    }

    public function updateIncidentStatus(int $incidentId, string $status, string $notes = ''): void
    {
        $update = ['status' => $status];

        if ($status === 'resolved') {
            $update['resolved_at'] = date('Y-m-d H:i:s');
            $update['resolved_by'] = $_SESSION['adminid'] ?? null;
        }

        Capsule::table('mod_incidents')
            ->where('id', $incidentId)
            ->update($update);

        Capsule::table('mod_incident_notes')->insert([
            'incident_id' => $incidentId,
            'note' => $notes,
            'created_by' => $_SESSION['adminid'] ?? null,
            'created_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function addContainmentAction(int $incidentId, string $action, string $performedBy): void
    {
        Capsule::table('mod_incident_actions')->insert([
            'incident_id' => $incidentId,
            'action_type' => 'containment',
            'action' => $action,
            'performed_by' => $performedBy,
            'performed_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function addEradicationAction(int $incidentId, string $action, string $performedBy): void
    {
        Capsule::table('mod_incident_actions')->insert([
            'incident_id' => $incidentId,
            'action_type' => 'eradication',
            'action' => $action,
            'performed_by' => $performedBy,
            'performed_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function addRecoveryAction(int $incidentId, string $action, string $performedBy): void
    {
        Capsule::table('mod_incident_actions')->insert([
            'incident_id' => $incidentId,
            'action_type' => 'recovery',
            'action' => $action,
            'performed_by' => $performedBy,
            'performed_at' => date('Y-m-d H:i:s')
        ]);
    }

    public function getActiveIncidents(): array
    {
        return Capsule::table('mod_incidents')
            ->whereIn('status', ['open', 'investigating', 'contained'])
            ->orderByRaw("FIELD(severity, 'critical', 'high', 'medium', 'low')")
            ->orderBy('detected_at', 'desc')
            ->get()
            ->toArray();
    }

    public function generateIncidentReport(int $incidentId): array
    {
        $incident = Capsule::table('mod_incidents')->where('id', $incidentId)->first();
        $actions = Capsule::table('mod_incident_actions')->where('incident_id', $incidentId)->get();
        $notes = Capsule::table('mod_incident_notes')->where('incident_id', $incidentId)->get();
        $affected = Capsule::table('mod_incident_affected')->where('incident_id', $incidentId)->get();

        $timeline = [];
        foreach ($actions as $action) {
            $timeline[] = [
                'type' => $action->action_type,
                'action' => $action->action,
                'by' => $action->performed_by,
                'at' => $action->performed_at
            ];
        }

        return [
            'incident' => $incident,
            'affected_entities' => $affected,
            'actions_taken' => $timeline,
            'notes' => $notes,
            'duration_minutes' => $this->calculateDuration($incident)
        ];
    }

    private function sendIncidentAlert(int $incidentId, string $severity): void
    {
        $incident = Capsule::table('mod_incidents')->where('id', $incidentId)->first();

        // Send to security team
        $admins = Capsule::table('tbladmins')
            ->where('roleid', 1) // Admin role
            ->get();

        foreach ($admins as $admin) {
            send_email('SecurityIncidentAlert', $admin->email, [
                'incident_id' => $incidentId,
                'severity' => $severity,
                'title' => $incident->title,
                'detected_at' => $incident->detected_at
            ]);
        }
    }

    private function calculateDuration($incident): int
    {
        $start = new \DateTime($incident->detected_at);
        $end = $incident->resolved_at
            ? new \DateTime($incident->resolved_at)
            : new \DateTime();

        return $start->diff($end)->i;
    }
}
```

## Step 2: Emergency Response Procedures

```bash
#!/bin/bash
# emergency-containment.sh

# 1. Isolate affected systems
echo "Isolating affected systems..."

# 2. Block suspicious IPs
iptables -A INPUT -s $SUSPICIOUS_IP -j DROP

# 3. Disable compromised accounts
php artisan tinker --execute="App\Models\User::where('email', '$EMAIL')->update(['active' => false])"

# 4. Revoke compromised API keys
php artisan tinker --execute="ApiKey::where('key', '$KEY')->delete()"

# 5. Clear sessions
php artisan session:clear

# 6. Notify team
echo "Emergency containment complete. Check incident dashboard."
```

## Verification Checklist

- [ ] Incident tracking service implemented
- [ ] Incident creation working
- [ ] Status updates working
- [ ] Containment actions logging
- [ ] Alert system configured
- [ ] Report generation working
- [ ] Emergency procedures documented
