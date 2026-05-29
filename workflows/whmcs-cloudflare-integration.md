# WHMCS Cloudflare Integration Workflow

## Overview
This workflow implements Cloudflare CDN/DNS integration for WHMCS.

## Prerequisites
- WHMCS with Cloudflare API access
- Cloudflare account
- Admin access

## Step-by-Step Process

### Step 1: Cloudflare Integration
```php
<?php
// /includes/integrations/CloudflareIntegration.php

class CloudflareIntegration {
    private $apiKey;
    private $email;

    public function __construct()
    {
        $this->apiKey = getConfig('cloudflare_api_key');
        $this->email = getConfig('cloudflare_email');
    }

    /**
     * Purge CDN cache
     */
    public function purgeCache(string $zoneId): array
    {
        $ch = curl_init("https://api.cloudflare.com/client/v4/zones/{$zoneId}/purge_cache");

        curl_setopt_array($ch, [
            CURLOPT_CUSTOMREQUEST => 'DELETE',
            CURLOPT_HTTPHEADER => [
                'X-Auth-Email: ' . $this->email,
                'X-Auth-Key: ' . $this->apiKey,
                'Content-Type: application/json'
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }

    /**
     * Get DNS records
     */
    public function getDNSRecords(string $zoneId): array
    {
        $ch = curl_init("https://api.cloudflare.com/client/v4/zones/{$zoneId}/dns_records");

        curl_setopt_array($ch, [
            CURLOPT_HTTPHEADER => [
                'X-Auth-Email: ' . $this->email,
                'X-Auth-Key: ' . $this->apiKey
            ],
            CURLOPT_RETURNTRANSFER => true
        ]);

        $response = curl_exec($ch);
        curl_close($ch);

        return json_decode($response, true);
    }
}
```

## Related Workflows
- [WHMCS CDN Integration](./whmcs-cdn-integration.md)
- [WHMCS DNS Configuration](./whmcs-dns-configuration.md)