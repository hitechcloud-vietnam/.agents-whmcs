# WHMCS Provisioning Automation Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for automating provisioning workflows with custom hooks and triggers.

## When to Use

- Building automated provisioning chains
- Creating custom workflow automation
- Implementing ITSM integrations

## Automation Patterns

### Provisioning Pipeline

```php
// Automated provisioning flow
add_hook('AfterModuleCreate', 1, function($vars) {
    $serviceId = $vars['serviceid'];
    $userId = $vars['userid'];
    $params = $vars['params'];

    // Pipeline stages
    $stages = [
        'provision' => 'stageProvision',
        'configure' => 'stageConfigure',
        'notification' => 'stageNotification',
        'provisioning_complete' => 'stageComplete',
    ];

    foreach ($stages as $stageName => $handler) {
        $result = call_user_func($handler, $serviceId, $params);

        logProvisionStage($serviceId, $stageName, $result);

        if (!$result['success']) {
            handleProvisionFailure($serviceId, $stageName, $result['error']);
            break;
        }
    }
});

function stageProvision($serviceId, $params): array {
    try {
        $api = new \Provisioning\ApiClient($params);
        $result = $api->createInstance($params);

        // Store instance info
        Capsule::table('mod_provision_instances')->insert([
            'service_id' => $serviceId,
            'instance_id' => $result['instance_id'],
            'ip_address' => $result['ip'],
            'status' => 'provisioned',
            'created_at' => date('Y-m-d H:i:s'),
        ]);

        return ['success' => true, 'data' => $result];

    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}

function stageConfigure($serviceId, $params): array {
    try {
        // Configure security groups
        // Setup networking
        // Install base applications
        return ['success' => true];
    } catch (\Exception $e) {
        return ['success' => false, 'error' => $e->getMessage()];
    }
}
```

### Automated Workflow Triggers

```php
// Trigger-based automation
class WorkflowEngine {
    private array $triggers = [];

    public function registerTrigger(string $event, callable $handler, int $priority = 100): void {
        $this->triggers[$event][] = [
            'handler' => $handler,
            'priority' => $priority,
        ];

        usort($this->triggers[$event], fn($a, $b) => $a['priority'] <=> $b['priority']);
    }

    public function trigger(string $event, array $context): array {
        $results = [];

        if (empty($this->triggers[$event])) {
            return $results;
        }

        foreach ($this->triggers[$event] as $trigger) {
            try {
                $result = call_user_func($trigger['handler'], $context);
                $results[] = [
                    'success' => true,
                    'result' => $result,
                ];
            } catch (\Exception $e) {
                $results[] = [
                    'success' => false,
                    'error' => $e->getMessage(),
                ];
            }
        }

        return $results;
    }
}

// Automation rules
function defineAutomationRules(): void {
    global $workflowEngine;

    // Rule: Auto-scale on high usage
    $workflowEngine->registerTrigger('usage_threshold_exceeded', function($vars) {
        $serviceId = $vars['serviceid'];
        $currentSpec = getServiceSpec($serviceId);

        $nextSpec = getNextSpecTier($currentSpec);
        if ($nextSpec) {
            Capsule::table('mod_automation_suggestions')->insert([
                'service_id' => $serviceId,
                'suggestion_type' => 'upgrade',
                'suggested_spec' => $nextSpec['name'],
                'reason' => 'High resource usage detected',
                'created_at' => date('Y-m-d H:i:s'),
            ]);

            sendUpgradeSuggestionEmail($serviceId);
        }
    });

    // Rule: Auto backup on schedule
    $workflowEngine->registerTrigger('backup_schedule', function($vars) {
        $services = getServicesDueForBackup();

        foreach ($services as $service) {
            $api = new \Provisioning\ApiClient(getServerConfig($service->server));
            $api->createBackup($service->instance_id);
        }
    });

    // Rule: Auto renewal notification
    $workflowEngine->registerTrigger('renewal_warning', function($vars) {
        $expiredSoon = getServicesExpiringSoon(7);

        foreach ($expiredSoon as $service) {
            sendRenewalReminder($service);
        }
    });
}
```

### ITSM Integration

```php
// ServiceNow integration for provisioning
class ITSMIntegration {
    private string $instanceUrl;
    private string $apiKey;

    public function createTicket(array $data): string {
        $ticketData = [
            'short_description' => $data['title'],
            'description' => $data['description'],
            'category' => 'Provisioning',
            'priority' => $this->mapPriority($data['priority']),
            'affected_service' => $data['service_id'],
            'requested_for' => $data['user_email'],
        ];

        return $this->apiCall('POST', '/table/incident', $ticketData);
    }

    public function updateTicket(string $ticketId, array $updates): void {
        $this->apiCall('PATCH', "/table/incident/$ticketId", $updates);
    }

    public function getTicketStatus(string $ticketId): array {
        return $this->apiCall('GET', "/api/v1/tickets/$ticketId/status");
    }

    private function mapPriority(string $priority): int {
        return match ($priority) {
            'critical' => 1,
            'high' => 2,
            'medium' => 3,
            'low' => 4,
            default => 5,
        };
    }
}

// Hook-based ITSM integration
add_hook('AfterModuleCreate', 1, function($vars) {
    $itsm = new ITSMIntegration();

    $ticketId = $itsm->createTicket([
        'title' => 'New service provisioned',
        'description' => "Service ID: {$vars['serviceid']} has been provisioned",
        'priority' => 'medium',
        'service_id' => $vars['serviceid'],
        'user_email' => getClientEmail($vars['userid']),
    ]);

    Capsule::table('mod_automation_tickets')->insert([
        'service_id' => $vars['serviceid'],
        'itsm_ticket_id' => $ticketId,
        'type' => 'provisioning',
        'created_at' => date('Y-m-d H:i:s'),
    ]);
});
```

### Scheduled Automation

```yaml
# automation-schedule.yaml
schedules:
  - name: "Hourly Usage Check"
    cron: "0 * * * *"
    action: "check_service_usage"
    conditions:
      - type: "service_active"

  - name: "Daily Backup"
    cron: "0 2 * * *"
    action: "backup_services"
    conditions:
      - type: "backup_enabled"

  - name: "Weekly Reports"
    cron: "0 6 * * 0"
    action: "generate_reports"
    conditions:
      - type: "always"

  - name: "Monthly Cleaning"
    cron: "0 3 1 * *"
    action: "cleanup_old_data"
    conditions:
      - type: "retention_expired"
```

---

**Related Skills:**
- whmcs-hooks-development
- whmcs-cron-automation
- whmcs-server-builder
