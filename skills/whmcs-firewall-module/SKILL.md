# WHMCS Firewall Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build network firewall management modules for server protection.

## Firewall Module Structure

```php
<?php
/**
 * Firewall Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Network Firewall',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'firewall_zone'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'DefaultPolicy' => [
            'Type' => 'dropdown',
            'Options' => 'allow,deny',
            'Default' => 'deny',
        ],
        'ManagedRules' => [
            'Type' => 'yesno',
            'Description' => 'Enable managed security rules',
        ],
        'Logging' => [
            'Type' => 'yesno',
            'Description' => 'Enable rule logging',
        ],
    ];
}
```

## Firewall Management

```php
function {module}_CreateAccount(array $params): string {
    $this->api->createFirewallGroup([
        'name' => 'service_' . $params['serviceid'],
        'default_policy' => $params['configoption1'],
        'rules' => $this->getDefaultRules(),
    ]);

    Capsule::table('mod_firewall_groups')->insert([
        'service_id' => $params['serviceid'],
        'group_id' => $params['serviceid'],
        'default_policy' => $params['configoption1'],
        'managed_rules' => $params['configoption2'] === 'on',
        'logging' => $params['configoption3'] === 'on',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $group = Capsule::table('mod_firewall_groups')
        ->where('service_id', $params['serviceid'])
        ->first();

    if ($group) {
        $this->api->deleteFirewallGroup($group->group_id);
        Capsule::table('mod_firewall_groups')
            ->where('id', $group->id)
            ->delete();
    }

    return 'success';
}
```

## Rule Management

```php
public function addRule(int $serviceId, array $rule): int {
    $group = $this->getFirewallGroup($serviceId);

    $firewallRule = $this->api->addRule($group->group_id, [
        'direction' => $rule['direction'] ?? 'ingress',
        'protocol' => $rule['protocol'],
        'port' => $rule['port'] ?? null,
        'source' => $rule['source'] ?? '0.0.0.0/0',
        'action' => $rule['action'] ?? 'allow',
        'description' => $rule['description'] ?? '',
    ]);

    Capsule::table('mod_firewall_rules')->insert([
        'group_id' => $group->id,
        'remote_rule_id' => $firewallRule['id'],
        'direction' => $firewallRule['direction'],
        'protocol' => $firewallRule['protocol'],
        'port' => $firewallRule['port'],
        'source' => $firewallRule['source'],
        'action' => $firewallRule['action'],
        'description' => $firewallRule['description'],
        'enabled' => true,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    if ($group->logging) {
        $this->logRuleChange($group->id, 'added', $firewallRule);
    }

    return $firewallRule['id'];
}

public function removeRule(int $serviceId, int $ruleId): bool {
    $rule = Capsule::table('mod_firewall_rules')->where('id', $ruleId)->first();

    if (!$rule) {
        return false;
    }

    $group = $this->getFirewallGroup($serviceId);
    $this->api->removeRule($group->group_id, $rule->remote_rule_id);

    Capsule::table('mod_firewall_rules')->where('id', $ruleId)->delete();

    return true;
}

public function toggleRule(int $ruleId, bool $enabled): bool {
    $rule = Capsule::table('mod_firewall_rules')->where('id', $ruleId)->first();

    $this->api->updateRule($rule->remote_rule_id, ['enabled' => $enabled]);

    Capsule::table('mod_firewall_rules')
        ->where('id', $ruleId)
        ->update(['enabled' => $enabled]);

    return true;
}
```

## Preset Rules

```php
public function getPresetRules(): array {
    return [
        'http' => [
            'protocol' => 'tcp',
            'port' => '80',
            'action' => 'allow',
            'description' => 'HTTP Traffic',
        ],
        'https' => [
            'protocol' => 'tcp',
            'port' => '443',
            'action' => 'allow',
            'description' => 'HTTPS Traffic',
        ],
        'ssh' => [
            'protocol' => 'tcp',
            'port' => '22',
            'action' => 'allow',
            'description' => 'SSH Access',
        ],
        'mysql' => [
            'protocol' => 'tcp',
            'port' => '3306',
            'action' => 'allow',
            'description' => 'MySQL Access',
        ],
        'dns_udp' => [
            'protocol' => 'udp',
            'port' => '53',
            'action' => 'allow',
            'description' => 'DNS (UDP)',
        ],
        'dns_tcp' => [
            'protocol' => 'tcp',
            'port' => '53',
            'action' => 'allow',
            'description' => 'DNS (TCP)',
        ],
    ];
}
```

## Statistics

```php
public function getRuleStats(int $serviceId): array {
    $group = $this->getFirewallGroup($serviceId);

    $stats = $this->api->getStats($group->group_id);

    Capsule::table('mod_firewall_stats')->insert([
        'group_id' => $group->id,
        'packets_allowed' => $stats['packets_allowed'],
        'packets_denied' => $stats['packets_denied'],
        'bytes_allowed' => $stats['bytes_allowed'],
        'bytes_denied' => $stats['bytes_denied'],
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);

    return [
        'packets_allowed' => $stats['packets_allowed'],
        'packets_denied' => $stats['packets_denied'],
        'bytes_allowed' => $this->formatBytes($stats['bytes_allowed']),
        'bytes_denied' => $this->formatBytes($stats['bytes_denied']),
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-waf-module
- whmcs-security-hardening