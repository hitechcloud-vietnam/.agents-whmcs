# WHMCS CDN Integration Skill
# Version: 1.0 | Updated: 2026-05-28

## Purpose

Guide for integrating CDN services into WHMCS modules.

## When to Use

- Media storage modules
- Cache management
- Static asset delivery

## CDN Patterns

```php
<?php
class CDNManager {
    private string $apiKey;
    private string $endpoint = 'https://api.cdn.com/v1';

    public function upload(string $localPath, string $remotePath): array {
        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->endpoint . '/files/upload',
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => [
                'file' => curl_file_create($localPath),
                'path' => $remotePath,
            ],
            CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $this->apiKey],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $result = json_decode(curl_exec($ch), true);
        curl_close($ch);

        return $result;
    }

    public function purgeCache(string $url): bool {
        $ch = curl_init();

        curl_setopt_array($ch, [
            CURLOPT_URL => $this->endpoint . '/cache/purge',
            CURLOPT_CUSTOMREQUEST => 'DELETE',
            CURLOPT_POSTFIELDS => json_encode(['url' => $url]),
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $this->apiKey,
                'Content-Type: application/json',
            ],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        $result = json_decode(curl_exec($ch), true);
        curl_close($ch);

        return $result['success'] ?? false;
    }

    public function getStats(): array {
        $ch = curl_init($this->endpoint . '/stats');
        curl_setopt_array($ch, [
            CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $this->apiKey],
            CURLOPT_RETURNTRANSFER => true,
        ]);

        return json_decode(curl_exec($ch), true);
    }

    public function getSignedUrl(string $path, int $expires = 3600): string {
        $expiry = time() + $expires;
        $signature = hash_hmac('sha256', $path . $expiry, $this->apiKey);

        return $this->endpoint . $path . '?expires=' . $expiry . '&signature=' . $signature;
 }
}
```

---

**Related Skills:**
- whmcs-cloudflare-module
- whmcs-server-builder
- whmcs-performance-optimization
