# WHMCS Cloud Storage Integration

Complete guide for integrating WHMCS with cloud storage services.

## Overview

Connect WHMCS to cloud storage for backups, file management, and document storage.

## AWS S3 Integration

### S3 Client Setup

```php
<?php
/**
 * AWS S3 storage integration
 */
class S3StorageClient
{
    private string $bucket;
    private string $region;
    private string $accessKey;
    private string $secretKey;
    
    public function __construct(array $config)
    {
        $this->bucket = $config['bucket'];
        $this->region = $config['region'];
        $this->accessKey = $config['access_key'];
        $this->secretKey = $config['secret_key'];
    }
    
    /**
     * Generate presigned URL for upload
     */
    public function generateUploadUrl(string $key, int $expires = 3600): string
    {
        $datetime = gmdate('Ymd\THis\Z');
        $datestamp = gmdate('Ymd');
        
        $payload = "AWS4-HMAC-SHA256\n{$datetime}\n{$datestamp}/{$this->region}/s3/aws4_request";
        
        $signingKey = $this->getSignatureKey(
            $this->secretKey,
            $datestamp,
            $this->region,
            's3'
        );
        
        $signature = bin2hex(hash_hmac('sha256', $payload, $signingKey, true));
        
        $params = [
            'X-Amz-Algorithm' => 'AWS4-HMAC-SHA256',
            'X-Amz-Credential' => $this->accessKey . '/' . 
                "{$datestamp}/{$this->region}/s3/aws4_request",
            'X-Amz-Date' => $datetime,
            'X-Amz-Expires' => $expires,
            'X-Amz-SignedHeaders' => 'host',
            'X-Amz-Signature' => $signature,
        ];
        
        return "https://{$this->bucket}.s3.{$this->region}.amazonaws.com/{$key}?" . 
               http_build_query($params);
    }
    
    /**
     * Get signature key
     */
    private function getSignatureKey(string $key, string $date, string $region, string $service): string
    {
        $kDate = hash_hmac('sha256', "AWS4{$key}", $date, true);
        $kRegion = hash_hmac('sha256', $kDate, $region, true);
        $kService = hash_hmac('sha256', $kRegion, $service, true);
        $kSigning = hash_hmac('sha256', $kService, 'aws4_request', true);
        
        return $kSigning;
    }
    
    /**
     * Upload file directly
     */
    public function uploadFile(string $localPath, string $s3Key, string $contentType = 'application/octet-stream'): bool
    {
        $content = file_get_contents($localPath);
        return $this->uploadContent($content, $s3Key, $contentType);
    }
    
    /**
     * Upload content directly
     */
    public function uploadContent(string $content, string $s3Key, string $contentType): bool
    {
        $datetime = gmdate('Ymd\THis\Z');
        $datestamp = gmdate('Ymd');
        
        $hash = hash('sha256', $content);
        
        $headers = [
            'Content-Type' => $contentType,
            'X-Amz-Date' => $datetime,
            'x-amz-content-sha256' => $hash,
        ];
        
        $canonical = $this->buildCanonicalRequest(
            'PUT',
            '/' . ltrim($s3Key, '/'),
            '',
            $headers,
            $hash
        );
        
        $credential = $this->accessKey . '/' . "{$datestamp}/{$this->region}/s3/aws4_request";
        
        $ch = curl_init("https://{$this->bucket}.s3.{$this->region}.amazonaws.com/{$s3Key}");
        curl_setopt_array($ch, [
            CURLOPT_PUT => true,
            CURLOPT_INFILE => fopen('data://text/plain;base64,' . base64_encode($content), 'r'),
            CURLOPT_INFILESIZE => strlen($content),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => array_map(
                fn($k, $v) => "{$k}: {$v}",
                array_keys($headers),
                $headers
            ),
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode >= 200 && $httpCode < 300;
    }
    
    /**
     * Build canonical request
     */
    private function buildCanonicalRequest(string $method, string $uri, string $query, array $headers, string $payloadHash): string
    {
        return implode("\n", [
            $method,
            $uri,
            $query,
            implode("\n", array_map(fn($k, $v) => "{$k}:{$v}", array_keys($headers), $headers)),
            implode(';', array_keys($headers)),
            $payloadHash,
        ]);
    }
    
    /**
     * Download file
     */
    public function downloadFile(string $s3Key, string $localPath): bool
    {
        $ch = curl_init("https://{$this->bucket}.s3.{$this->region}.amazonaws.com/{$s3Key}");
        $fp = fopen($localPath, 'wb');
        
        curl_setopt_array($ch, [
            CURLOPT_FILE => $fp,
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        fclose($fp);
        
        return $httpCode === 200;
    }
}
```

### S3 Backup Integration

```php
<?php
/**
 * WHMCS database backup to S3
 */
function backupDatabaseToS3(): array
{
    $s3 = new S3StorageClient([
        'bucket' => S3_BUCKET,
        'region' => S3_REGION,
        'access_key' => S3_ACCESS_KEY,
        'secret_key' => S3_SECRET_KEY,
    ]);
    
    // Create temporary backup file
    $backupFile = '/tmp/whmcs_backup_' . date('Y-m-d_H-i-s') . '.sql.gz';
    
    // Run mysqldump
    $host = Capsule::connection()->getConfig('host');
    $dbname = Capsule::connection()->getConfig('database');
    $username = Capsule::connection()->getConfig('username');
    $password = Capsule::connection()->getConfig('password');
    
    exec("mysqldump -h {$host} -u {$username} -p'{$password}' {$dbname} | gzip > {$backupFile}");
    
    // Upload to S3
    $s3Key = 'backups/whmcs/' . date('Y/m/d/H-i-s') . '_backup.sql.gz';
    $success = $s3->uploadFile($backupFile, $s3Key, 'application/gzip');
    
    // Clean up
    unlink($backupFile);
    
    return [
        'success' => $success,
        's3_key' => $s3Key,
    ];
}
```

## Google Cloud Storage

### GCS Integration

```php
<?php
/**
 * Google Cloud Storage client
 */
class GCSClient
{
    private string $bucket;
    private string $projectId;
    private string $serviceAccountKey;
    
    public function __construct(array $config)
    {
        $this->bucket = $config['bucket'];
        $this->projectId = $config['project_id'];
        $this->serviceAccountKey = $config['service_account_json'];
    }
    
    /**
     * Get access token
     */
    private function getAccessToken(): string
    {
        $key = json_decode($this->serviceAccountKey, true);
        
        $header = base64_encode(json_encode(['alg' => 'RS256', 'typ' => 'JWT']));
        $claimSet = base64_encode(json_encode([
            'iss' => $key['client_email'],
            'scope' => 'https://www.googleapis.com/auth/devstorage.read_write',
            'aud' => 'https://oauth2.googleapis.com/token',
            'exp' => time() + 3600,
            'iat' => time(),
        ]));
        
        $signature = '';
        openssl_sign(
            "{$header}.{$claimSet}",
            $signature,
            $key['private_key'],
            OPENSSL_ALGO_SHA256
        );
        
        $jwt = "{$header}.{$claimSet}." . base64_encode($signature);
        
        $ch = curl_init('https://oauth2.googleapis.com/token');
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => http_build_query([
                'grant_type' => 'urn:ietf:params:oauth:grant-type:jwt-bearer',
                'assertion' => $jwt,
            ]),
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        curl_close($ch);
        
        return $response['access_token'];
    }
    
    /**
     * Upload file
     */
    public function uploadFile(string $localPath, string $gcsKey): bool
    {
        $token = $this->getAccessToken();
        $content = file_get_contents($localPath);
        
        $ch = curl_init("https://storage.googleapis.com/upload/storage/v1/b/{$this->bucket}/o?uploadType=media&name=" . urlencode($gcsKey));
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $content,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Authorization: Bearer ' . $token,
                'Content-Type: application/octet-stream',
            ],
        ]);
        
        $response = json_decode(curl_exec($ch), true);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode >= 200 && $httpCode < 300;
    }
    
    /**
     * Download file
     */
    public function downloadFile(string $gcsKey, string $localPath): bool
    {
        $token = $this->getAccessToken();
        
        $ch = curl_init("https://storage.googleapis.com/storage/v1/b/{$this->bucket}/o/" . urlencode($gcsKey) . '?alt=media');
        $fp = fopen($localPath, 'wb');
        
        curl_setopt_array($ch, [
            CURLOPT_FILE => $fp,
            CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $token],
        ]);
        
        curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        fclose($fp);
        
        return $httpCode === 200;
    }
    
    /**
     * Generate signed URL
     */
    public function generateSignedUrl(string $gcsKey, int $expires = 3600): string
    {
        $token = $this->getAccessToken();
        
        $url = "https://storage.googleapis.com/{$this->bucket}/{$gcsKey}";
        
        return $url; // In production, add signature
    }
}
```

## Azure Blob Storage

```php
<?php
/**
 * Azure Blob Storage client
 */
class AzureBlobClient
{
    private string $accountName;
    private string $accountKey;
    private string $container;
    
    public function __construct(array $config)
    {
        $this->accountName = $config['account_name'];
        $this->accountKey = $config['account_key'];
        $this->container = $config['container'];
    }
    
    /**
     * Generate SAS token
     */
    public function generateSasToken(string $blobName, int $expires = 3600): string
    {
        $start = time();
        $expiry = $start + $expires;
        
        $stringToSign = implode("\n", [
            'GET',
            '', // Content-Encoding
            '', // Content-Language
            '', // Content-Length
            '', // Content-MD5
            '', // Content-Type
            '', // Date
            '', // If-Modified-Singe
            '', // If-Match
            '', // If-None-Match
            '', // If-Unmodified-Singe
            '', // Range
            $this->container . '/' . $blobName,
            '', // Snapshot
            'sp=r&st=' . gmdate('Y-m-d\TH:i:s\Z', $start) . '&se=' . gmdate('Y-m-d\TH:i:s\Z', $expiry) . '&spr=https&sv=2020-08-04&ss=b&srt=sco',
        ]);
        
        $signature = base64_encode(hash_hmac('sha256', base64_encode($stringToSign), base64_decode($this->accountKey), true));
        
        $params = [
            'sv' => '2020-08-04',
            'ss' => 'b',
            'srt' => 'sco',
            'sp' => 'r',
            'st' => gmdate('Y-m-d\TH:i:s\Z', $start),
            'se' => gmdate('Y-m-d\TH:i:s\Z', $expiry),
            'sig' => $signature,
        ];
        
        return '?' . http_build_query($params);
    }
    
    /**
     * Upload blob
     */
    public function uploadBlob(string $localPath, string $blobName): bool
    {
        $content = file_get_contents($localPath);
        $md5 = base64_encode(md5($content, true));
        
        $url = "https://{$this->accountName}.blob.core.windows.net/{$this->container}/{$blobName}";
        
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_PUT => true,
            CURLOPT_INFILE => fopen($localPath, 'r'),
            CURLOPT_INFILESIZE => filesize($localPath),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'x-ms-date: ' . gmdate('D, d M Y H:i:s T'),
                'x-ms-version: 2020-08-04',
                'Content-Type: application/octet-stream',
                'Content-MD5: ' . $md5,
            ],
        ]);
        
        curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return $httpCode >= 200 && $httpCode < 300;
    }
    
    /**
     * Download blob
     */
    public function downloadBlob(string $blobName, string $localPath): bool
    {
        $sasToken = $this->generateSasToken($blobName);
        $url = "https://{$this->accountName}.blob.core.windows.net/{$this->container}/{$blobName}{$sasToken}";
        
        $ch = curl_init($url);
        $fp = fopen($localPath, 'wb');
        
        curl_setopt_array($ch, [
            CURLOPT_FILE => $fp,
            CURLOPT_RETURNTRANSFER => true,
        ]);
        
        curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        fclose($fp);
        
        return $httpCode === 200;
    }
}
```

## Best Practices

1. **Use signed URLs** - For secure temporary access
2. **Enable encryption** - Use server-side encryption
3. **Set lifecycle policies** - Auto-delete old backups
4. **Monitor usage** - Track storage costs
5. **Implement retries** - Handle transient failures
6. **Compress uploads** - Reduce bandwidth and costs

## Related Documentation

- [whmcs-integration-api.md](whmcs-integration-api.md)
- [whmcs-integration-backup.md](whmcs-integration-backup.md)
