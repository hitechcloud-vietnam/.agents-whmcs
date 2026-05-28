# WHMCS Service Provisioning Automation Workflow
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Automate service provisioning workflows.

## Auto-Provisioning Pipeline

```php
<?php
class ProvisioningPipeline {
    private array $stages = [
        'validate',
        'provision',
        'configure',
        'notify',
        'complete',
    ];

    public function execute(int $serviceId): array {
        $results = [];

        foreach ($this->stages as $stage) {
            $method = 'stage' . ucfirst($stage);

            $result = $this->$method($serviceId);
            $results[$stage] = $result;

            if (!$result['success']) {
                $this->rollback($serviceId, $results);
                break;
            }
        }

        return $results;
    }

    private function stageProvision(int $serviceId): array {
        $api = new HostingApi();

        try {
            $instance = $api->createInstance($this->getInstanceParams($serviceId));

            Capsule::table('mod_provision_log')->insert([
                'service_id' => $serviceId,
                'stage' => 'provision',
                'result' => json_encode($instance),
                'created_at' => date('Y-m-d H:i:s'),
            ]);

            return ['success' => true, 'instance' => $instance];
        } catch (\Exception $e) {
            return ['success' => false, 'error' => $e->getMessage()];
        }
    }

    private function stageConfigure(int $serviceId): array {
        // Apply configurations
        $api = new HostingApi();
        $api->applyDNS($serviceId);
        $api->setupSSL($serviceId);
        $api->configureBackups($serviceId);

        return ['success' => true];
    }

    private function stageNotify(int $serviceId): array {
        // Send welcome email
        send_email([
            'type' => 'Product',
            'id' => $this->getProductId($serviceId),
            'customvars' => [
                'service_id' => $serviceId,
                'hostname' => $this->getHostname($serviceId),
            ],
        ]);

        return ['success' => true];
    }

    private function rollback(int $serviceId, array $results): void {
        foreach (array_reverse($results) as $stage => $result) {
            if ($result['success']) {
                $this->{'rollback'.' . ucfirst($stage)}($serviceId);
            }
        }
    }
}
```

## Hook-Based Automation
```php
add_hook('AfterModuleCreate', 1, function($vars) {
    $pipeline = new ProvisioningPipeline();
    $results = $pipeline->execute($vars['serviceid']);

    foreach ($results as $stage => $result) {
        if (!$result['success']) {
            logActivity("Provisioning failed at stage: $stage");
            // Trigger alert
        }
    }
});
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-cron-automation
