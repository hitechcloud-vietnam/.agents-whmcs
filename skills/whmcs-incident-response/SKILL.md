---
name: whmcs-incident-response
description: Incident response for WHMCS
category: Troubleshooting & Debugging
version: 1.0.0
---

# WHMCS Incident Response Skill

## Overview
This skill provides patterns and implementations for managing incidents in WHMCS, including detection, response, and resolution workflows.

## Implementation Patterns

### Incident Response Manager
```php
<?php
/**
 * WHMCS Incident Response
 * Manages incident lifecycle
 */

namespace WHMCS\Module\Diagnostics\Incidents;

class IncidentResponseManager {
    private $db;

    public function __construct() {
        $this->db = \WHMCS\Database\Capsule::connection();
    }

    /**
     * Create incident
     */
    public function createIncident(array $params): array {
        $incidentId = 'inc_' . bin2hex(random_bytes(8));

        $incident = [
            'id' => $incidentId,
            'service_id' => $params['service_id'],
            'title' => $params['title'],
            'severity' => $params['severity'] ?? 'medium', // critical, high, medium, low
            'description' => $params['description'],
            'detected_at' => date('Y-m-d H:i:s'),
            'status' => 'investigating',
            'created_at' => date('Y-m-d H:i:s')
        ];

        $this->db->insert('mod_incidents', $incident);

        // Notify response team
        $this->notifyResponseTeam($incident);

        return [
            'success' => true,
            'incident_id' => $incidentId,
            'severity' => $params['severity']
        ];
    }

    /**
     * Update incident status
     */
    public function updateStatus(string $incidentId, string $status): array {
        $updates = [
            'status' => $status,
            'updated_at' => date('Y-m-d H:i:s')
        ];

        if ($status === 'resolved') {
            $updates['resolved_at'] = date('Y-m-d H:i:s');
        }

        $this->db->update('mod_incidents', $updates, ['id' => $incidentId]);

        return [
            'success' => true,
            'incident_id' => $incidentId,
            'status' => $status
        ];
    }

    /**
     * Add incident timeline entry
     */
    public function addTimelineEntry(string $incidentId, array $entry): array {
        $this->db->insert('mod_incident_timeline', [
            'incident_id' => $incidentId,
            'action' => $entry['action'],
            'details' => $entry['details'],
            'created_by' => $entry['created_by'] ?? 'system',
            'created_at' => date('Y-m-d H:i:s')
        ]);

        return ['success' => true];
    }

    /**
     * Get active incidents
     */
    public function getActiveIncidents(): array {
        return $this->db->select(
            "SELECT * FROM mod_incidents
             WHERE status NOT IN ('resolved', 'closed')
             ORDER BY severity ASC, created_at DESC"
        );
    }
}
```

## Database Schema
```sql
CREATE TABLE `mod_incidents` (
  `id` VARCHAR(50) PRIMARY KEY,
  `service_id` INT NOT NULL,
  `title` VARCHAR(255) NOT NULL,
  `severity` ENUM('critical', 'high', 'medium', 'low') DEFAULT 'medium',
  `description' TEXT,
  `detected_at' DATETIME NOT NULL,
  `resolved_at' DATETIME,
  `status' ENUM('detected', 'investigating', 'identified', 'mitigating', 'resolved', 'closed') DEFAULT 'detected',
  `created_at' DATETIME NOT NULL,
  `updated_at' DATETIME
);

CREATE TABLE `mod_incident_timeline` (
  `id` INT AUTO_INCREMENT PRIMARY KEY,
  `incident_id` VARCHAR(50) NOT NULL,
  `action` VARCHAR(255) NOT NULL,
  `details' TEXT,
  `created_by' VARCHAR(100),
  `created_at' DATETIME NOT NULL,
  FOREIGN KEY (`incident_id`) REFERENCES `mod_incidents`(`id`)
);
```

## Incident Response Workflow

1. **Detect**: Monitoring triggers incident
2. **Triage**: Assess severity and impact
3. **Investigate**: Find root cause
4. **Mitigate**: Take immediate action
5. **Resolve**: Implement fix
6. **Review**: Post-mortem analysis

## Best Practices

1. **Clear Communication**: Keep stakeholders updated
2. **Prioritization**: Focus on critical issues first
3. **Documentation**: Track all actions taken
4. **Root Cause**: Always find root cause
5. **Prevention**: Implement fixes to prevent recurrence

## Related Skills

- whmcs-monitoring-agent
- whmcs-debug-mode
- whmcs-log-analysis
- whmcs-crash-analysis