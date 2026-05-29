# WHMCS DNS Management Integration

Complete guide for integrating WHMCS with DNS management systems.

## Overview

Connect WHMCS with DNS providers for automatic DNS management and domain configuration.

## Cloudflare DNS Integration

### Cloudflare API Client

```php
<?php
/**
 * Cloudflare DNS management
 */
class CloudflareDNSClient
{
    private string $apiKey;
    private string $email;
    
    public function __construct(string $apiKey, string $email)
    {
        $this->apiKey = $apiKey;
        $this->email = $email;
    }
    
    /**
     * List DNS records for zone
     */
    public function listRecords(string $zoneId): array
    {
        $ch = curl_init("https://api.cloudflare.com/client/v4/zones/{$zoneId}/dns_records");
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response['result'] ?? [];
    }
    
    /**
     * Add DNS record
     */
    public function addRecord(string $zoneId, array $record): array
    {
        $ch = curl_init("https://api.cloudflare.com/client/v4/zones/{$zoneId}/dns_records");
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode($record),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response;
    }
    
    /**
     * Update DNS record
     */
    public function updateRecord(string $zoneId, string $recordId, array $record): array
    {
        $ch = curl_init("https://api.cloudflare.com/client/v4/zones/{$zoneId}/dns_records/{$recordId}");
        curl_setopt_array($ch, [
            CURLOPT_PUT => true,
            CURLOPT_POSTFIELDS => json_encode($record),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response;
    }
    
    /**
     * Delete DNS record
     */
    public function deleteRecord(string $zoneId, string $recordId): array
    {
        $ch = curl_init("https://api.cloudflare.com/client/v4/zones/{$zoneId}/dns_records/{$recordId}");
        curl_setopt_array($ch, [
            CURLOPT_CUSTOMREQUEST => 'DELETE',
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
            ],
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response;
    }
}
```

### Cloudflare Zone Management

```php
<?php
/**
 * Cloudflare zone operations
 */
class CloudflareZoneManager
{
    private CloudflareDNSClient $dns;
    
    public function __construct(CloudflareDNSClient $dns)
    {
        $this->dns = $dns;
    }
    
    /**
     * Create zone for domain
     */
    public function createZone(string $domain): array
    {
        $ch = curl_init('https://api.cloudflare.com/client/v4/zones');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => json_encode([
                'name' => $domain,
                'account' => ['name' => CLOUDFLARE_ACCOUNT_NAME],
                'jump_start' => true,
                'zone_type' => 'full',
            ]),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->dns->getApiKey(),
                'Content-Type: application/json',
            ],
        ]);
        
        return json_decode(curl_exec($ch), true);
    }
    
    /**
     * Get zone by domain
     */
    public function getZoneByDomain(string $domain): ?array
    {
        $ch = curl_init("https://api.cloudflare.com/client/v4/zones?name={$domain}");
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->dns->getApiKey(),
            ],
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        $results = $response['result'] ?? [];
        return $results[0] ?? null;
    }
    
    /**
     * Enable proxy for record
     */
    public function enableProxy(string $zoneId, string $recordId): array
    {
        return $this->dns->updateRecord($zoneId, $recordId, [
            'proxied' => true,
        ]);
    }
    
    /**
     * Disable proxy for record
     */
    public function disableProxy(string $zoneId, string $recordId): array
    {
        return $this->dns->updateRecord($zoneId, $recordId, [
            'proxied' => false,
        ]);
    }
}
```

## Route53 Integration

```php
<?php
/**
 * AWS Route53 DNS management
 */
class Route53DNSClient
{
    private string $accessKey;
    private string $secretKey;
    private string $region;
    
    public function __construct(array $config)
    {
        $this->accessKey = $config['access_key'];
        $this->secretKey = $config['secret_key'];
        $this->region = $config['region'] ?? 'us-east-1';
    }
    
    /**
     * Create hosted zone
     */
    public function createHostedZone(string $domain): string
    {
        $xml = <<<XML
        <?xml version="1.0" encoding="UTF-8"?>
        <CreateHostedZoneRequest xmlns="https://route53.amazonaws.com/doc/2013-04-01/">
            <Name>{$domain}</Name>
            <CallerReference>whmcs-" . time() . "</CallerReference>
        </CreateHostedZoneRequest>
        XML;
        
        $response = $this->request('POST', '/2013-04-01/hostedzone', $xml, 'application/xml');
        
        // Extract zone ID from response
        preg_match('/\/hostedzone\/(.+)\//', $response, $matches);
        return $matches[1] ?? '';
    }
    
    /**
     * Add record
     */
    public function addRecord(string $zoneId, array $record): bool
    {
        $changeBatch = [
            'Comment' => 'WHMCS DNS Sync',
            'Changes' => [
                [
                    'Action' => 'CREATE',
                    'ResourceRecordSet' => [
                        'Name' => $record['name'],
                        'Type' => $record['type'],
                        'TTL' => $record['ttl'] ?? 300,
                        'ResourceRecords' => [
                            ['Value' => $record['value']],
                        ],
                    ],
                ],
            ],
        ];
        
        $xml = $this->arrayToXml($changeBatch, '<ChangeBatch>');
        $response = $this->request('POST', "/2013-04-01/hostedzone/{$zoneId}/rrset", $xml, 'application/xml');
        
        return strpos($response, '<Status>INSYNC</Status>') !== false;
    }
    
    /**
     * List records
     */
    public function listRecords(string $zoneId): array
    {
        $response = $this->request('GET', "/2013-04-01/hostedzone/{$zoneId}/rrset");
        
        $records = [];
        preg_match_all('/<ResourceRecordSet>(.*?)<\/ResourceRecordSet>/s', $response, $matches);
        
        foreach ($matches[1] as $match) {
            $records[] = [
                'name' => $this->extractTag($match, 'Name'),
                'type' => $this->extractTag($match, 'Type'),
                'ttl' => $this->extractTag($match, 'TTL'),
                'value' => $this->extractTag($match, 'Value'),
            ];
        }
        
        return $records;
    }
    
    /**
     * Make signed request
     */
    private function request(string $method, string $path, string $body = '', string $contentType = 'application/xml'): string
    {
        $datetime = gmdate('Ymd\THis\Z');
        $date = gmdate('Ymd');
        
        $headers = [
            'Host' => 'route53.' . $this->region . '.amazonaws.com',
            'X-Amz-Date' => $datetime,
            'X-Amz-Target' => 'Route53_20130401',
            'Content-Type' => $contentType,
        ];
        
        $ch = curl_init('https://route53.' . $this->region . '.amazonaws.com' . $path);
        curl_setopt_array($ch, [
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CUSTOMREQUEST => $method,
            CURLOPT_POSTFIELDS => $body,
            CURLOPT_HTTPHEADER => array_values($headers),
        ]);
        
        $response = curl_exec($ch);
        curl_close($ch);
        
        return $response;
    }
    
    private function extractTag(string $xml, string $tag): string
    {
        preg_match("/<{$tag}>(.*?)<\\/{$tag}>/s", $xml, $matches);
        return $matches[1] ?? '';
    }
}
```

## WHMCS DNS Sync Hook

```php
<?php
/**
 * Sync WHMCS nameservers to DNS provider
 */
add_hook('DNSManagementSync', 1, function($vars) {
    $domain = $vars['domain'];
    $nameservers = $vars['nameservers'];
    
    $dns = new CloudflareDNSClient(CLOUDFLARE_API_KEY, CLOUDFLARE_EMAIL);
    $zoneManager = new CloudflareZoneManager($dns);
    
    // Get zone
    $zone = $zoneManager->getZoneByDomain($domain);
    
    if (!$zone) {
        // Create zone
        $zone = $zoneManager->createZone($domain);
    }
    
    // Update NS records
    $records = $dns->listRecords($zone['id']);
    
    foreach ($records as $record) {
        if ($record['type'] === 'NS') {
            $dns->deleteRecord($zone['id'], $record['id']);
        }
    }
    
    foreach ($nameservers as $ns) {
        $dns->addRecord($zone['id'], [
            'type' => 'NS',
            'name' => $domain,
            'content' => $ns,
            'ttl' => 86400,
        ]);
    }
    
    logActivity("DNS Sync: Updated nameservers for {$domain}");
});
```

## Bulk DNS Operations

```php
<?php
/**
 * Bulk DNS sync for all domains
 */
function syncAllDomainsDNS(): array
{
    $dns = new CloudflareDNSClient(CLOUDFLARE_API_KEY, CLOUDFLARE_EMAIL);
    $zoneManager = new CloudflareZoneManager($dns);
    
    $results = [
        'synced' => 0,
        'failed' => 0,
        'errors' => [],
    ];
    
    $domains = Capsule::table('tbldomains')
        ->where('status', 'Active')
        ->where('dnsmanagement', 1)
        ->get();
    
    foreach ($domains as $domain) {
        try {
            // Get current nameservers from WHMCS
            $whmcsNS = [
                $domain->ns1,
                $domain->ns2,
                $domain->ns3 ?? '',
                $domain->ns4 ?? '',
            ];
            $whmcsNS = array_filter($whmcsNS);
            
            // Sync to DNS provider
            $zone = $zoneManager->getZoneByDomain($domain->domain);
            
            if (!$zone) {
                $zone = $zoneManager->createZone($domain->domain);
            }
            
            $results['synced']++;
            
        } catch (Exception $e) {
            $results['failed']++;
            $results['errors'][] = "{$domain->domain}: " . $e->getMessage();
        }
    }
    
    return $results;
}
```

## Best Practices

1. **Cache zone data** - Reduce API calls
2. **Batch operations** - Group record changes
3. **Implement rate limiting** - Respect provider limits
4. **Handle propagation** - Allow time for DNS changes
5. **Verify changes** - Confirm record updates
6. **Log all operations** - Track DNS modifications

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-module-registrar-api.md](whmcs-module-registrar-api.md)
