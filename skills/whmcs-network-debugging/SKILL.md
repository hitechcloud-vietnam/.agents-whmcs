---
name: whmcs-network-debugging
description: Network diagnostics for WHMCS
category: Troubleshooting & Debugging
version: 1.0.0
---

# WHMCS Network Diagnostics Skill

## Overview
This skill provides patterns for network debugging in WHMCS.

## Implementation Patterns

### Network Diagnostics
```php
<?php
/**
 * WHMCS Network Diagnostics
 * Diagnoses network issues
 */

namespace WHMCS\Module\Diagnostics\Network;

class NetworkDiagnostics {
    /**
     * Test connectivity
     */
    public function testConnectivity(string $host, int $port = 80): array {
        $start = microtime(true);
        $socket = @fsockopen($host, $port, $errno, $errstr, 5);
        $latency = (microtime(true) - $start) * 1000;

        return [
            'host' => $host,
            'port' => $port,
            'reachable' => $socket !== false,
            'latency_ms' => round($latency, 2),
            'error' => $errstr ?? null
        ];
    }

    /**
     * DNS lookup test
     */
    public function testDNS(string $domain): array {
        $start = microtime(true);
        $ips = gethostbynamel($domain);
        $duration = (microtime(true) - $start) * 1000;

        return [
            'domain' => $domain,
            'resolved' => $ips !== false,
            'ips' => $ips ?? [],
            'lookup_time_ms' => round($duration, 2)
        ];
    }

    /**
     * Trace route
     */
    public function traceRoute(string $host): array {
        $output = shell_exec("traceroute -m 10 {$host} 2>&1");
        $hops = array_filter(explode("\n", $output));

        return [
            'host' => $host,
            'hops' => array_map('trim', $hops),
            'reachable' => count($hops) > 0
        ];
    }
}
```

## Best Practices

1. **Timeout Settings**: Use appropriate timeouts
2. **Port Verification**: Check specific ports
3. **DNS Caching**: Monitor DNS resolution
4. **Latency Monitoring**: Track network latency
5. **Connection Failures**: Log and analyze failures

## Related Skills

- whmcs-debug-mode
- whmcs-monitoring-agent
- whmcs-incident-response
- whmcs-log-analysis