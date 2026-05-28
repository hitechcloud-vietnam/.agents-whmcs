# WHMCS WAF Module Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Build Web Application Firewall (WAF) modules for security protection.

## WAF Module Structure

```php
<?php
/**
 * WAF Module
 * Location: modules/servers/{module}/
 */

function {module}_MetaData(): array {
    return [
        'DisplayName' => 'Web Application Firewall',
        'APIVersion' => '1.0',
        'RequiresServer' => true,
        'Parameters' => ['api_key', 'waf_zone_id'],
    ];
}

function {module}_ConfigOptions(array $params): array {
    return [
        'ProtectionLevel' => [
            'Type' => 'dropdown',
            'Options' => 'low,medium,high,paranoid',
            'Default' => 'medium',
        ],
        'DDoSProtection' => [
            'Type' => 'yesno',
            'Description' => 'Enable DDoS mitigation',
        ],
        'RateLimiting' => [
            'Type' => 'yesno',
            'Description' => 'Enable rate limiting',
        ],
        'BotProtection' => [
            'Type' => 'yesno',
            'Description' => 'Enable bot detection',
        ],
    ];
}
```

## WAF Operations

```php
function {module}_CreateAccount(array $params): string {
    $waf = $this->api->createDomain([
        'domain' => $params['domain'],
        'protection_level' => $params['configoption1'],
        'ddos_protection' => $params['configoption2'] === 'on',
        'rate_limiting' => $params['configoption3'] === 'on',
        'bot_protection' => $params['configoption4'] === 'on',
    ]);

    Capsule::table('mod_waf_domains')->insert([
        'service_id' => $params['serviceid'],
        'domain_id' => $waf['id'],
        'domain' => $params['domain'],
        'protection_level' => $params['configoption1'],
        'ddos_protection' => $params['configoption2'] === 'on',
        'rate_limiting' => $params['configoption3'] === 'on',
        'bot_protection' => $params['configoption4'] === 'on',
        'status' => 'active',
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return 'success';
}

function {module}_SuspendAccount(array $params): string {
    $domain = $this->getWafDomain($params['serviceid']);

    $this->api->updateDomain($domain['domain_id'], [
        'status' => 'paused',
    ]);

    Capsule::table('mod_waf_domains')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'suspended']);

    return 'success';
}

function {module}_UnsuspendAccount(array $params): string {
    $domain = $this->getWafDomain($params['serviceid']);

    $this->api->updateDomain($domain['domain_id'], [
        'status' => 'active',
    ]);

    Capsule::table('mod_waf_domains')
        ->where('service_id', $params['serviceid'])
        ->update(['status' => 'active']);

    return 'success';
}

function {module}_TerminateAccount(array $params): string {
    $domain = $this->getWafDomain($params['serviceid']);

    if ($domain) {
        $this->api->deleteDomain($domain['domain_id']);
        Capsule::table('mod_waf_domains')
            ->where('service_id', $params['serviceid'])
            ->delete();
    }

    return 'success';
}
```

## Rule Management

```php
public function addIPRule(int $serviceId, string $ip, string $action = 'block'): bool {
    $domain = $this->getWafDomain($serviceId);

    $this->api->addRule($domain['domain_id'], [
        'type' => 'ip',
        'value' => $ip,
        'action' => $action,
        'expires' => time() + 86400,
    ]);

    Capsule::table('mod_waf_rules')->insert([
        'service_id' => $serviceId,
        'rule_type' => 'ip',
        'value' => $ip,
        'action' => $action,
        'expires_at' => date('Y-m-d H:i:s', time() + 86400),
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return true;
}

public function addURLCondition(int $serviceId, string $pattern, string $action = 'block'): bool {
    $domain = $this->getWafDomain($serviceId);

    $this->api->addRule($domain['domain_id'], [
        'type' => 'url',
        'pattern' => $pattern,
        'action' => $action,
    ]);

    Capsule::table('mod_waf_rules')->insert([
        'service_id' => $serviceId,
        'rule_type' => 'url',
        'value' => $pattern,
        'action' => $action,
        'created_at' => date('Y-m-d H:i:s'),
    ]);

    return true;
}
```

## Statistics & Analytics

```php
public function getStats(int $serviceId, string $period = '24h'): array {
    $domain = $this->getWafDomain($serviceId);

    $stats = $this->api->getStats($domain['domain_id'], $period);

    Capsule::table('mod_waf_stats')->insert([
        'service_id' => $serviceId,
        'requests_total' => $stats['total_requests'],
        'requests_blocked' => $stats['blocked_requests'],
        'requests_cached' => $stats['cached_requests'],
        'bandwidth_saved' => $stats['bandwidth_saved'],
        'recorded_at' => date('Y-m-d H:i:s'),
    ]);

    return [
        'total_requests' => $stats['total_requests'],
        'blocked_requests' => $stats['blocked_requests'],
        'block_rate' => round(($stats['blocked_requests'] / $stats['total_requests']) * 100, 2),
        'bandwidth_saved' => $this->formatBytes($stats['bandwidth_saved']),
        'top_threats' => $stats['top_threats'] ?? [],
    ];
}

private function formatBytes(int $bytes): string {
    $units = ['B', 'KB', 'MB', 'GB'];
    $i = 0;
    while ($bytes >= 1024 && $i < 3) {
        $bytes /= 1024;
        $i++;
    }
    return round($bytes, 2) . ' ' . $units[$i];
}
```

## Client Area

```php
function {module}_ClientArea(array $params): array {
    $domain = $this->getWafDomain($params['serviceid']);
    $stats = $this->getStats($params['serviceid']);

    $recentBlocks = Capsule::table('mod_waf_logs')
        ->where('service_id', $params['serviceid'])
        ->where('action', 'block')
        ->orderBy('created_at', 'desc')
        ->limit(10)
        ->get();

    return [
        'pagetitle' => 'WAF Dashboard',
        'templatefile' => 'templates/waf_clientarea',
        'vars' => [
            'domain' => $domain,
            'stats' => $stats,
            'recent_blocks' => $recentBlocks,
        ],
    ];
}
```

---

**Related Skills:**
- whmcs-server-builder
- whmcs-security-hardening
- whmcs-monitoring