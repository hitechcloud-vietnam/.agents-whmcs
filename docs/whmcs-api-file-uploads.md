# WHMCS API File Uploads

## Overview

The WHMCS API supports file uploads for various operations including support tickets, document management, and module attachments.

## Upload Methods

### Base64 Encoding Method

```php
<?php
function uploadFileToWhmcs(
    string $apiUrl,
    string $apiIdentifier,
    string $apiSecret,
    string $filePath,
    int $relatedId,
    string $uploadType
): array {
    if (!file_exists($filePath)) {
        throw new RuntimeException("File not found: {$filePath}");
    }
    
    $fileContent = file_get_contents($filePath);
    $fileName = basename($filePath);
    $fileType = mime_content_type($filePath);
    $base64Content = base64_encode($fileContent);
    
    $postData = [
        'action' => 'AddTicketAttachment',
        'identifier' => $apiIdentifier,
        'secret' => $apiSecret,
        'responsetype' => 'json',
        'file_data' => $base64Content,
        'filename' => $fileName,
        'filetype' => $fileType,
    ];
    
    if ($uploadType === 'ticket') {
        $postData['ticketid'] = $relatedId;
    }
    
    $ch = curl_init($apiUrl);
    curl_setopt_array($ch, [
        CURLOPT_POST => true,
        CURLOPT_POSTFIELDS => $postData,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT => 60,
    ]);
    
    $response = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    
    return json_decode($response, true);
}
```

### Multipart Form Upload

```php
<?php
class WhmcsFileUploader {
    private string $apiUrl;
    private string $apiIdentifier;
    private string $apiSecret;
    private int $maxFileSize = 10485760; // 10MB default
    
    public function __construct(string $apiUrl, string $apiIdentifier, string $apiSecret)
    {
        $this->apiUrl = $apiUrl;
        $this->apiIdentifier = $apiIdentifier;
        $this->apiSecret = $apiSecret;
    }
    
    public function setMaxFileSize(int $bytes): self
    {
        $this->maxFileSize = $bytes;
        return $this;
    }
    
    public function upload(string $action, string $filePath, array $additionalParams = []): array
    {
        $this->validateFile($filePath);
        
        $fileName = basename($filePath);
        $fileType = $this->getMimeType($filePath);
        
        $postData = [
            'action' => $action,
            'identifier' => $this->apiIdentifier,
            'secret' => $this->apiSecret,
            'responsetype' => 'json',
        ];
        
        foreach ($additionalParams as $key => $value) {
            $postData[$key] = $value;
        }
        
        $ch = curl_init($this->apiUrl);
        
        // Build multipart form data
        $boundary = '----WebKitFormBoundary' . bin2hex(random_bytes(16));
        
        $body = $this->buildMultipartBody($postData, $filePath, $fileName, $fileType, $boundary);
        
        curl_setopt_array($ch, [
            CURLOPT_POST => true,
            CURLOPT_POSTFIELDS => $body,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HTTPHEADER => [
                'Content-Type: multipart/form-data; boundary=' . $boundary,
            ],
            CURLOPT_TIMEOUT => 120,
        ]);
        
        $response = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);
        
        return json_decode($response, true);
    }
    
    private function validateFile(string $filePath): void
    {
        if (!file_exists($filePath)) {
            throw new InvalidArgumentException("File does not exist: {$filePath}");
        }
        
        $fileSize = filesize($filePath);
        if ($fileSize > $this->maxFileSize) {
            throw new InvalidArgumentException(sprintf(
                'File size %d exceeds maximum allowed %d bytes',
                $fileSize,
                $this->maxFileSize
            ));
        }
    }
    
    private function getMimeType(string $filePath): string
    {
        $finfo = finfo_open(FILEINFO_MIME_TYPE);
        $mimeType = finfo_file($finfo, $filePath);
        finfo_close($finfo);
        return $mimeType;
    }
    
    private function buildMultipartBody(
        array $params,
        string $filePath,
        string $fileName,
        string $fileType,
        string $boundary
    ): string {
        $body = '';
        
        // Add form fields
        foreach ($params as $key => $value) {
            $body .= "--{$boundary}\r\n";
            $body .= "Content-Disposition: form-data; name=\"{$key}\"\r\n\r\n";
            $body .= "{$value}\r\n";
        }
        
        // Add file
        $body .= "--{$boundary}\r\n";
        $body .= "Content-Disposition: form-data; name=\"attachment\"; filename=\"{$fileName}\"\r\n";
        $body .= "Content-Type: {$fileType}\r\n\r\n";
        $body .= file_get_contents($filePath) . "\r\n";
        
        $body .= "--{$boundary}--\r\n";
        
        return $body;
    }
}

// Usage
$uploader = new WhmcsFileUploader(
    'https://whmcs.example.com/includes/api.php',
    'api_identifier',
    'api_secret'
);

$result = $uploader->upload(
    'AddTicketAttachment',
    '/path/to/document.pdf',
    ['ticketid' => 12345]
);
```

### Chunked Upload for Large Files

```php
<?php
class ChunkedFileUploader {
    private const CHUNK_SIZE = 1024 * 1024; // 1MB chunks
    
    private WhmcsFileUploader $baseUploader;
    private string $uploadId;
    
    public function __construct(WhmcsFileUploader $baseUploader)
    {
        $this->baseUploader = $baseUploader;
    }
    
    public function upload(string $action, string $filePath, array $params = []): array
    {
        $fileSize = filesize($filePath);
        
        // For small files, use regular upload
        if ($fileSize <= self::CHUNK_SIZE) {
            return $this->baseUploader->upload($action, $filePath, $params);
        }
        
        // Initiate chunked upload
        $this->uploadId = $this->initiateChunkedUpload($filePath, $params);
        
        // Upload chunks
        $handle = fopen($filePath, 'rb');
        $chunkNumber = 0;
        
        while (!feof($handle)) {
            $chunk = fread($handle, self::CHUNK_SIZE);
            $this->uploadChunk($chunkNumber, $chunk);
            $chunkNumber++;
            
            // Progress callback could be called here
        }
        
        fclose($handle);
        
        // Complete the upload
        return $this->completeChunkedUpload();
    }
    
    private function initiateChunkedUpload(string $filePath, array $params): string
    {
        $uploadId = bin2hex(random_bytes(16));
        
        // Store upload session info
        $_SESSION['uploads'][$uploadId] = [
            'file_name' => basename($filePath),
            'file_size' => filesize($filePath),
            'total_chunks' => ceil(filesize($filePath) / self::CHUNK_SIZE),
            'uploaded_chunks' => [],
            'params' => $params,
        ];
        
        return $uploadId;
    }
    
    private function uploadChunk(int $chunkNumber, string $data): array
    {
        // In production, this would call an internal API endpoint
        // that stores the chunk temporarily
        $base64Chunk = base64_encode($data);
        
        return [
            'result' => 'success',
            'chunk_number' => $chunkNumber,
        ];
    }
    
    private function completeChunkedUpload(): array
    {
        // In production, this would assemble chunks
        // and create the final file
        return [
            'result' => 'success',
            'upload_id' => $this->uploadId,
        ];
    }
}
```

## File Storage Integration

```php
<?php
class WhmcsAttachmentStorage {
    private string $storagePath;
    private string $apiUrl;
    private string $apiIdentifier;
    private string $apiSecret;
    
    public function __construct(
        string $storagePath,
        string $apiUrl,
        string $apiIdentifier,
        string $apiSecret
    ) {
        $this->storagePath = rtrim($storagePath, '/');
        $this->apiUrl = $apiUrl;
        $this->apiIdentifier = $apiIdentifier;
        $this->apiSecret = $apiSecret;
    }
    
    public function uploadTicketAttachment(int $ticketId, string $filePath): array
    {
        $uploader = new WhmcsFileUploader(
            $this->apiUrl,
            $this->apiIdentifier,
            $this->apiSecret
        );
        
        $result = $uploader->upload(
            'AddTicketAttachment',
            $filePath,
            ['ticketid' => $ticketId]
        );
        
        if ($result['result'] === 'success') {
            $this->storeMetadata($result['file_id'], $filePath, $ticketId);
        }
        
        return $result;
    }
    
    public function uploadClientDocument(int $clientId, string $filePath, string $documentType): array
    {
        $uploader = new WhmcsFileUploader(
            $this->apiUrl,
            $this->apiIdentifier,
            $this->apiSecret
        );
        
        return $uploader->upload(
            'UploadClientDocument',
            $filePath,
            [
                'client_id' => $clientId,
                'document_type' => $documentType,
            ]
        );
    }
    
    private function storeMetadata(string $fileId, string $filePath, int $ticketId): void
    {
        $metadata = [
            'file_id' => $fileId,
            'original_path' => $filePath,
            'original_name' => basename($filePath),
            'ticket_id' => $ticketId,
            'uploaded_at' => date('Y-m-d H:i:s'),
        ];
        
        $metaPath = "{$this->storagePath}/{$fileId}.json";
        file_put_contents($metaPath, json_encode($metadata, JSON_PRETTY_PRINT));
    }
    
    public function getAttachmentInfo(string $fileId): ?array
    {
        $metaPath = "{$this->storagePath}/{$fileId}.json";
        
        if (!file_exists($metaPath)) {
            return null;
        }
        
        return json_decode(file_get_contents($metaPath), true);
    }
    
    public function deleteAttachment(string $fileId): bool
    {
        $metaPath = "{$this->storagePath}/{$fileId}.json";
        
        if (file_exists($metaPath)) {
            unlink($metaPath);
        }
        
        // Call WHMCS API to delete the attachment
        // ...
        
        return true;
    }
}
```

## Allowed File Types

```php
<?php
class FileTypeValidator {
    private array $allowedMimeTypes = [
        'image/jpeg',
        'image/png',
        'image/gif',
        'application/pdf',
        'text/plain',
        'application/zip',
        'application/x-zip-compressed',
        'application/msword',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    ];
    
    private array $allowedExtensions = [
        'jpg', 'jpeg', 'png', 'gif', 'pdf',
        'txt', 'zip', 'doc', 'docx',
    ];
    
    private array $dangerousExtensions = [
        'php', 'php3', 'php4', 'php5', 'phtml',
        'exe', 'sh', 'bat', 'cmd', 'scr',
        'js', 'vbs', 'ps1', 'sh',
    ];
    
    public function validate(string $filePath): ValidationResult
    {
        $extension = strtolower(pathinfo($filePath, PATHINFO_EXTENSION));
        
        // Check extension
        if (in_array($extension, $this->dangerousExtensions)) {
            return new ValidationResult(false, "File type not allowed: {$extension}");
        }
        
        if (!in_array($extension, $this->allowedExtensions)) {
            return new ValidationResult(false, "Unknown file extension: {$extension}");
        }
        
        // Check MIME type
        $finfo = finfo_open(FILEINFO_MIME_TYPE);
        $mimeType = finfo_file($finfo, $filePath);
        finfo_close($finfo);
        
        // Basic MIME validation (extension matches content)
        $expectedMimeType = $this->getMimeTypeForExtension($extension);
        if ($mimeType !== $expectedMimeType && !in_array($mimeType, $this->allowedMimeTypes)) {
            return new ValidationResult(false, "File content does not match extension");
        }
        
        return new ValidationResult(true, 'Valid file');
    }
    
    private function getMimeTypeForExtension(string $extension): string
    {
        $mimeTypes = [
            'jpg' => 'image/jpeg',
            'jpeg' => 'image/jpeg',
            'png' => 'image/png',
            'gif' => 'image/gif',
            'pdf' => 'application/pdf',
            'txt' => 'text/plain',
            'zip' => 'application/zip',
            'doc' => 'application/msword',
            'docx' => 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
        ];
        
        return $mimeTypes[$extension] ?? 'application/octet-stream';
    }
}

class ValidationResult {
    public function __construct(
        public readonly bool $valid,
        public readonly string $message
    ) {}
}
```

## Best Practices

1. **Validate file types** - Check both extension and MIME type
2. **Limit file sizes** - Enforce reasonable size limits
3. **Sanitize filenames** - Prevent path traversal attacks
4. **Use secure connections** - Always upload over HTTPS
5. **Store metadata** - Keep track of uploaded files
6. **Clean up orphaned files** - Remove unused attachments

## Related Documentation

- [WHMCS Ticket Hooks](/docs/whmcs-ticket-hooks.md)
- [WHMCS API Error Handling](/docs/whmcs-api-error-handling.md)